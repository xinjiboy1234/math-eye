# MathEye Electron 方案评估：桌面端 · BYOK · 多设备同步

> 日期：2026-08-26
> 状态：评估阶段（非实施规范）
> 上游文档：[技术选型评估报告](./2026-08-25-math-eye-tech-evaluation.md)、[详细设计文档](./2026-08-25-math-eye-detailed-design.md)
> 议题：① 用 Electron 做 UI 替代/补充 Web 形态；② LLM API key 由用户自行管理（BYOK）；③ 预留多设备同步 API。

## 0. 结论先行

**可行且推荐，但真正的分叉不是 Electron vs Web，而是「本地优先」vs「云端集中」。** BYOK 和多设备同步这两个诉求本身就指向本地优先：

| 维度 | 云端 Web（现方案） | 本地优先桌面（本评估推荐） |
|---|---|---|
| LLM 密钥 | 服务端托管，你有合规与盗用责任 | 用户 key 存本机系统钥匙串，只发往 LLM 厂商 |
| 服务器成本 | 全额承担 | MVP 阶段≈0（同步服务后置且很薄） |
| 未成年人隐私 | 对话/事件数据过你的服务器（§7 合规负担重） | 数据不出设备，合规负担大幅消失 |
| 离线可用 | 无 | 图谱浏览、SymPy 计算、练习全部离线 |
| 多设备同步 | 天然集中，无需做 | 需要后建薄同步服务（本文 §5） |
| 使用门槛 | 打开网址即用 | 需安装 + 自配 API key |

**关键代价：Python 后端不再以服务形态存在。** 数学计算改由 Pyodide（WASM 版 Python，SymPy 可用）承担，答疑编排/mastery/图谱服务迁移到 Electron 主进程的 TypeScript。原设计中**算法层（Elo、高亮规则、重组流程）与数据模型几乎无损保留**，这是本次转向的最大收益。FastAPI 保留为未来同步服务的实现语言。

## 1. 桌面框架选型：Electron vs Tauri vs 纯 Web

| 维度 | **Electron（选定）** | Tauri 2 | 纯 Web/PWA |
|---|---|---|---|
| 安装包体积 | ~90–120 MB | ~10 MB | 0 |
| 内存占用 | 高（Chromium 常驻） | 低（系统 WebView） | 浏览器自身 |
| 生态成熟度 | 最高：auto-update、崩溃上报、签名工具链全现成 | 快速成长但需引入 Rust 工具链 | 最高 |
| 后台任务/多进程 | utilityProcess 成熟 | sidecar 支持较弱 | 受限 |
| 与 BYOK 契合度 | 好：主进程直连 LLM 绕开 CORS | 好（同样可绕开） | 差：浏览器 CORS 使直连厂商 API 困难 |
| 移动端扩展路线 | 无（需另起项目） | 有（iOS/Android 支持） | PWA 可覆盖部分 |

选 Electron 的理由：本项目需要在后台常驻 Pyodide 计算线程、本地 SQLite、以及未来的同步客户端——这些「多进程 + 本地资源」场景 Electron 的工程答案最齐全；Tauri 的体积优势对本产品（教育工具，非高频轻工具）价值有限，却引入 Rust 维护面。纯 Web 被 BYOK 直连的 CORS 问题卡死（除非用各家 `dangerous-direct-browser-access` 类开关，等于把用户 key 暴露在浏览器上下文）。

## 2. 目标架构

```
┌─ Electron 应用 ─────────────────────────────────────┐
│ Renderer（React + Vite，替代 Next.js）              │
│   KaTeX │ Cytoscape 迷雾地图 │ JSXGraph │ TutorChat │
│   Pyodide Worker：用户代码沙箱 + SymPy 计算复核      │
│        ▲ contextBridge IPC（白名单接口）             │
│ Main（TypeScript / Node）                           │
│   llm/gateway：BYOK 多厂商路由、重试、成本记账       │
│   services：tutor 编排 │ mastery │ graph 高亮        │
│   db：better-sqlite3（沿用原 DDL + 同步预留列）      │
│   safeStorage：API key 存取                          │
└───────────┬─────────────────────────────────────────┘
            │ 仅两类出站 HTTPS：
   ┌────────┴─────────┐      ┌─ 同步服务（S2 后置，FastAPI）─┐
   │ LLM 厂商 API      │      │ 事件日志中转 + 账号体系       │
   │ （用户自己的 key）│      │ （用户数据 E2E 加密可选）     │
```

要点：

1. **LLM 调用只在主进程发生**：渲染进程永不接触 key 明文，也顺带解决浏览器直连厂商的 CORS 问题。
2. **Next.js → React + Vite**：桌面内不需要 SSR/路由服务端能力，Vite 是 Electron 生态惯配（electron-vite）。原设计的四个页面与五个组件划分不变。
3. **Pyodide 跑在隐藏 Worker/utilityProcess**：不阻塞 UI；「所有数学结论必须经 SymPy 验证」这条不变量原样保留，只是执行位置从服务端移到本机。
4. **better-sqlite3** 同步 API 性能好，适合主进程直接持有连接。

## 3. BYOK 设计

### 3.1 密钥生命周期

- **录入**：设置页录入，UI 上掩码显示（`sk-ant-…7f2a`）；
- **存储**：Electron `safeStorage`（Windows DPAPI / macOS Keychain / Linux libsecret），落盘为密文；
- **使用**：主进程读取解密 → 注入请求头 → 只发往该 provider 的 baseURL；
- **删除**：一键擦除；卸载不残留。

### 3.2 多厂商抽象

```ts
interface ProviderConfig {
  name: string            // anthropic / openai / deepseek / custom…
  baseURL: string         // 兼容 OpenAI 协议的自定义端点即可接第三方
  apiKeyRef: string       // safeStorage 存储句柄，不存明文
  models: string[]        // 该 key 可用的模型
}
```

统一走 **OpenAI-compatible 协议 + Anthropic 原生协议**两个适配器即可覆盖绝大多数场景（DeepSeek、Kimi、GLM、OpenRouter、Ollama/LM Studio 本地模型都是 OpenAI-compatible）。这对国内用户尤其重要：BYOK + 自定义 baseURL 让用户可以接任何国产厂商乃至纯本地模型，实现「零云端依赖」。

### 3.3 对原设计的具体影响

| 原 §7 条目 | 云端方案 | BYOK 本地方案 |
|---|---|---|
| 每用户每日 token 配额 | 服务端强制 | 客户端记账 + 本机限额提醒（约束力弱，但对自有 key 场景足够） |
| LLM 网关成本记账 | 按 user/day 聚合报表 | 本机统计页（每 key 用量/费用估算） |
| 未成年人不留存对话原文 | 需服务端配合 | 默认满足：数据在本机，用户可整库删除 |
| LLM 失败降级 | 服务端控制 | 主进程控制，逻辑不变 |

### 3.4 一个必须正视的产品矛盾

**儿童没有（也不应有）API key。** 产品定位是全年龄段，而 BYOK 把门槛压在了最需要保护的群体上。对策分两步：

1. **近期**：「家长配置」模式——家长/教师录入 key，儿童档案下使用；应用内明确标注当前使用的计费主体。
2. **远期（可选商业项）**：提供托管 key 的订阅模式——这会把一个轻量服务端重新拉回来，但届时它是增值服务而非基础设施，不影响 MVP 的零服务器承诺。此矛盾是 BYOK 路线唯一的结构性代价，需产品侧确认接受。

## 4. 数学引擎与 embeddings 的去向

- **SymPy**：Pyodide 内置可用（solve/diff/integrate/simplify 均正常工作）。原 `/api/compute` 的五种操作全部改为 Worker 内调用；`plot_data` 采样同理，反而省掉一次网络往返，滑动条联动延迟更优。
- **embeddings（图谱重组 §3.3）**：三个选项——① 用户 key 调 embedding API（质量最好，多一笔开销）；② transformers.js 跑本地小模型（离线但占资源）；③ MVP 先不做自动提议，图谱种子人工构建。建议 ③→① 演进：重组本来就是后置功能（P4），不阻塞主线。
- **Python 服务进程（PyInstaller sidecar）方案被否决**：能保住 FastAPI 代码原样，但安装包 +60–80 MB、双平台打包矩阵、启动时序、localhost 端口管理，复杂度不成比例。TS 重写的三个服务里，mastery 和高亮是纯函数规则引擎（几十行），tutor 编排是 prompt 组装 + JSON 解析——重写成本低且有 pytest 用例可平移为 vitest 用例。

## 5. 多设备同步：现在定规范，以后写代码

### 5.1 核心洞察：原设计已经是事件溯源

详细设计 §2 已规定 `learning_event` 是**不可变日志、水平模型的唯一数据来源**，且不变量 2 要求「mastery 更新是事件的纯函数」。这意味着：

> **同步不需要合并状态，只需要同步事件日志，然后各设备确定性重放。** Elo 更新、高亮计算全部可离线重算，冲突面天然极小。

### 5.2 立即落地的五条规范（写入设计文档 v2）

1. **主键一律 UUIDv7**（时间有序）：禁用 autoincrement/BIGSERIAL 作跨设备标识；SQLite 端 `learning_event.id` 从 BIGSERIAL 改 TEXT UUID。
2. **每表增加四列**：`device_id`（来源设备）、`hlc`（混合逻辑时钟，抗时钟回拨与漂移）、`updated_at`、`deleted_at`（软删墓碑，同步删除意图）。
3. **mastery 表永不同步**：它是派生缓存，各设备从事件流重建；同步协议里显式排除。
4. **图谱编辑也是事件**：节点/边的增改、以及 LLM 提议的 confirm/reject 决策，都记为事件类型（如 `graph_edge_confirmed`）——人工审核不变量在多设备下依然成立。节点正文等大字段的并发编辑采用 last-writer-wins（按 hlc），教育内容场景可接受。
5. **content_item 种子内容随应用版本分发**，不入同步流；用户自建内容才同步。

### 5.3 阶段规划

| 阶段 | 内容 | 成本 |
|---|---|---|
| S0（现在） | 仅落地 §5.2 规范到 DDL 与代码约定 | ≈0，但事后补的代价是数倍 |
| S1 | 导出/导入 JSON 快照文件（手动跨设备迁移） | 数天级，先解「有无」 |
| S2 | 薄同步服务（FastAPI）：`GET /sync/pull?cursor=` / `POST /sync/push`（携带 HLC 的事件批次），账号用邮箱 magic link 或 OAuth；按 user 分区的 append-only 表，服务端无业务逻辑 | 1–2 周量级 |

**为什么不用现成同步引擎**（PowerSync / cr-sqlite / ElectricSQL / Zero）：它们解决的是「任意表双向并发编辑」的重难题，而本应用冲突面近乎为零（append-only + 派生缓存）。引入它们换来的是外部依赖、调试黑盒与许可成本。自研薄同步在这个数据形状下是更便宜且完全可控的选择。若未来出现实时协作编辑需求再重估。

### 5.4 隐私加分项

同步服务可以做**纯密文中转**：客户端以用户口令派生密钥在本地加密事件批次，服务端只见密文。因为同步的是结构化事件而非自由文本，端到端加密的实现成本很低。这把「未成年人的学习数据存在谁的服务器上」这个问题直接消解掉了。

## 6. 风险清单

1. **儿童/BYOK 矛盾**（§3.4）：结构性问题，需产品侧拍板「家长配置」是否足以作为近期答案。
2. **分发与签名成本**：Windows 代码签名证书约 $70–400/年（EV 更贵，影响 SmartScreen 信誉建立速度）；macOS 公证 $99/年 + 一台 Mac CI；electron-updater 需要一个静态文件源（GitHub Releases 即可）。合计首年数百美元级 + 运维心智。
3. **双运行时心智**：TS 主逻辑 + Pyodide Worker，两套测试栈（vitest + Pyodide 内跑 pytest 式断言）。缓解：数学函数收敛在一个窄接口（原 `/api/compute` 的 5 个 op）后面。
4. **Pyodide 首次加载 ~10 MB**（原风险 4 延续）：桌面端反而更好解——随应用打包 pyodide 资源，彻底消除冷启动网络加载。
5. **低端设备内存**：Chromium + Pyodide 常驻约 400–600 MB；学校老旧机房需实测。缓解：Pyodide Worker 懒启动、空闲卸载。
6. **自动更新的强制力**：桌面端无法像 Web 一样瞬间修复线上算法缺陷；mastery 算法若有 bug，坏数据已在各设备事件流上。缓解：正因如此更要坚持「mastery 是事件纯函数」——修算法 = 各设备重放，天然具备数据自愈能力。这也是事件溯源路线最大的隐性回报。

## 7. 对现有文档的修订清单（待批准后执行）

| 文档 | 修订点 |
|---|---|
| 技术选型评估 | 整体架构图换为本文件 §2；沙箱一节改为「Pyodide 随应用打包」；新增 BYOK 与同步章节引用本文件 |
| 详细设计 | 目录结构改为 Electron 工程（main/preload/renderer）；DDL 按 §5.2 加列并改主键策略；`/api/*` 接口表拆为「IPC 通道表」（形态相同，传输层改名）；测试策略中 pytest 用例平移说明；§8 P1 验收标准不变（闭环仍在，只是单机版） |
| 保持不变 | Elo 公式、高亮规则阈值、重组流程、KaTeX/Cytoscape/JSXGraph 选型、数据模型实体关系、游戏化设计 |

## 8. 建议决策

1. ✅ 采纳：Electron + 本地优先 + BYOK，按 §5.2 五条规范修订设计文档后再开工（规范改动必须在第一行代码之前冻结）。
2. ❓ 待确认：§3.4 儿童使用门槛的产品答案（家长配置模式是否足够）。
3. ❓ 待确认：签名证书预算与发布渠道（GitHub Releases 公开仓库 vs 自建静态源）。
