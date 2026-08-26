# MathEye 答疑智能体设计文档

> 日期：2026-08-26
> 状态：设计阶段（已确认）
> 产品方向：**数学答疑智能体**（替代此前「知识图谱学习平台」方向，技术架构从零重选，不沿用 Electron 本地优先方案）
> 上游参考：技术选型评估（2026-08-25，UI 生态选型部分）、详细设计（2026-08-25，算法与不变量部分）

## 1. 产品定义

### 1.1 背景与定位

面向**缺少专家指导**的用户（自学学生、无法获得老师答疑的人群等）的数学答疑智能体。工具本身承担"专家"角色：先摸清用户水平，按水平答疑，记住用户的学习轨迹，能画动态图，能洞察并纠正错误联想。

无数学专家参与内容构建：**内容由 LLM 生成 + 自动化验证兜底**（SymPy 验证计算类断言；验证器是保守方向的——正确内容可能被误拒，错误内容几乎不可能被误收）。

### 1.2 核心需求（用户确认）

1. **入口测试**：进入时先测——选年龄段/学历，出对应档位的标志性题/概念（"听过吗？""算得对吗？"），摸清解释的专业程度。
2. **知识点关联**：若之前问过相关知识点，回答末尾说明相关性；列出其他密切相关知识点（**只列密切相关，≤3 个**）。
3. **动态图**：可以画出函数图/图表，**动态图优先**（参数滑动条联动、缩放拖拽）。
4. **错误纠正**：问题中包含错误答案/错误假设时，洞察错误联想的可能原因，举例纠正错误联想。

### 1.3 补充需求（已确认）

- **记忆系统三层**：会话历史（跨会话检索）+ 知识点时间线（问过/困惑/答错/掌握）+ 用户画像（档位 + 三维系数）。不做向量语义记忆。
- **输入方式**：文本 + LaTeX 公式（含常用公式模板快捷输入）。不做拍照 OCR、不做语音。
- **数学结论验证**：所有数学断言经 SymPy 验证后再展示，带验证状态徽标（✅ 已验证 / ⚠️ 未验证）。
- **记忆管理**：记忆面板可见、可编辑、一键清空、可导出。
- **匿名可用**：无登录、无服务端、无同步、无账号体系。

### 1.4 明确不做（YAGNI）

Electron/双进程、Pyodide/WASM、知识图谱迷雾地图、Elo 掌握度、多设备同步、游戏化、OCR/语音输入、托管服务端、向量语义记忆、Next.js/SSR、ORM、实时协作。

## 2. 关键决策记录

| 决策点 | 结论 | 理由 |
|---|---|---|
| 形态 | **本地 Web 应用**：FastAPI + 浏览器（localhost） | 单一 Python 进程、原生 SymPy、web UI 生态现成、复杂度最低；可后置 PyInstaller 打包为桌面应用 |
| LLM 来源 | **BYOK 多厂商**（OpenAI-compatible + Anthropic 原生） | key 在本机、数据不出设备；DeepSeek/Kimi/GLM/Ollama 本地模型均可通过兼容端点接入 |
| 记忆 | 三层本地 SQLite | 跨会话关联 + 画像稳定 |
| 管线 | **两阶段**：先分析后回答 | 错误检测显式化、compute 类问题路由 SymPy 省 token、可分别测试 |
| 动态图 | 一元函数/参数曲线/极坐标/隐式方程 + 滑动条缩放 | 覆盖中学/大学主流题型 |
| 输入 | 文本 + LaTeX | 零额外依赖 |
| 数学引擎 | 原生 SymPy（本机进程） | 无 WASM 移植边界，计算可靠性更高 |
| 验证 | SymPy 断言验证 + 状态徽标 | 无专家场景的诚实底线 |

## 3. 架构

### 3.1 总览

```
┌─ 本机（单一 Python 进程 + 浏览器）────────────────────────┐
│  FastAPI (uvicorn, localhost:8765)                        │
│    api/      聊天 / 入门测试 / 绘图 / 记忆 / 设置          │
│    pipeline/ 阶段1 分析 + 阶段2 回答（SSE 流式）           │
│    llm/      BYOK 多厂商网关（重试 / 本机记账）            │
│    math/     原生 SymPy：计算、验证、绘图采样              │
│    memory/   三层记忆读写                                 │
│    db/       SQLite（sqlite3 标准库，不引 ORM）            │
│    config/   API key 存系统 keyring，降级为本地配置文件    │
│         │ localhost HTTP/JSON + SSE                        │
│  Browser（React + Vite 静态前端，由 FastAPI 托管，同源）    │
│    ChatPanel │ KaTeX │ PlotCanvas(JSXGraph) │ 记忆面板 │   │
│    设置 │ 入门测试向导                                     │
└────────────────────────────────────────────────────────────┘
        │ 唯一出站连接：LLM 厂商 API（用户自己的 key，Python 进程持有）
```

### 3.2 技术栈

- 后端：Python 3.11+ / FastAPI / uvicorn / SymPy / keyring / sqlite3 标准库 / httpx（LLM 调用）
- 前端：React + Vite（纯静态，由 FastAPI StaticFiles 托管，同源无 CORS）+ KaTeX + JSXGraph
- 流式：SSE（阶段 2 回答流式返回）

### 3.3 进程与安全模型

- 单进程；浏览器仅与 localhost 同源通信。
- **API key 永不出现在浏览器上下文**：Python 进程持有，从 keyring 读取后注入请求头，只发往该 provider 的 baseURL。
- 启动：`uv run start` 或双击 `start.bat` → 自动打开浏览器；`Ctrl+C` 退出。端口固定 8765，占用时提示并给出 `--port` 参数。
- **本地安全**：所有 API 校验 `Origin`/`Host` 头，允许列表在启动时按实际绑定 host:port 生成（localhost 与 127.0.0.1 两种形式，随 `--port` 变化），防止本地网页驱动本服务（如 `DELETE /api/memory`、消耗用户 key）；keyring 降级配置文件路径 `~/.matheye/config.json`（0600 权限）。
- 桌面打包（PyInstaller）列为后置选项，不改变架构。

## 4. 两阶段管线

### 4.1 阶段 1：分析（轻量 LLM 调用 + 本地规则兜底）

输入：用户消息 + 画像档位摘要。输出结构化 JSON：

```json
{
  "question_type": "compute | conceptual | proof | plot_request",
  "contains_error": true,
  "error_hypotheses": [
    {"kind": "misconception", "likely_cause": "分配律滥用", "related_topic": "binomial-expansion"}
  ],
  "needs_plot": true,
  "plot_spec": {"type": "parametric", "exprs": ["a*cos(t)", "a*sin(t)"],
                "params": {"t": [0, 6.2832]},
                "sliders": [{"symbol": "a", "min": 0.5, "max": 3, "default": 1}]},
  "compute_spec": {"op": "solve", "sympy_expr": "x**2 - 4"},
  "topics": ["binomial-expansion", "algebra"]
}
```

- `plot_spec.sliders`：自由符号参数成为前端滑动条（联动重算），默认范围由阶段 1 给出。
- `compute_spec`：compute 类问题由阶段 1 在此给出计算表达式（流式开始前已确定，见 §4.2）。

- 本地规则兜底：检测 `solve/diff/integrate/化简` 等命令词 → 直接判 `compute`，跳过部分 LLM 调用以省 token；阶段 1 的 LLM **调用失败**同样降级到本地规则（至少可识别 compute 命令词）；仅当网关重试一次后仍失败（§9 的"重试中"）才中断本请求。
- `plot_spec` 中的表达式在阶段 1 输出时即校验（SymPy `parse_expr` 解析 + 语法验证），校验失败重试一次；仍失败 → `needs_plot=false` 并提示无法绘图——校验发生在 `meta` 事件发出前。
- `compute_spec` 与 plot_spec 同等契约：阶段 1 输出时即校验（`parse_expr`），失败重试一次；仍失败或执行期异常（如 solve 无解）→ 降级按 conceptual 处理并附错误提示，不中断流。
- 结构化输出解析失败 → 重试一次（带上次输出提示修正）；两次仍失败 → 本地规则兜底（默认按 conceptual、无错误、无绘图处理），流程继续不中断。

### 4.2 阶段 2：回答（主调用，SSE 流式）

注入上下文：用户画像（档位 + 三维系数）+ 知识点记忆（该知识点历史状态 + 密切相关知识点 ≤3）+ 相关历史问答摘要。

分支：

- `question_type = compute` → 计算由阶段 1 的 `compute_spec` 给出、后端 SymPy 执行，**LLM 不判定数学正确性**；解释/步骤由 LLM 措辞，其中每个可验证断言经 SymPy 验证（验证通过才标 ✅；验证失败时该断言降级为 ❌/⚠️，不阻断回答）。
- `contains_error = true` → **纠错模式**（见 §4.3）。
- 其余 → 常规回答模式。

输出：`reply_markdown`（KaTeX 渲染）+ `topics` + 关联卡片数据 + 可选 `followup_exercise`（"再练一道"按钮触发，LLM 生成 + SymPy 验证答案）。**关联卡片机制**：基于知识点记忆共现统计 + LLM 判断生成，取 ≤3 条密切相关。

**验证流程（断言由生成方声明，验证由确定性引擎执行）**：
- 阶段 2 输出除 `reply_markdown` 外，附带 `assertions: [{claim, sympy_expr, expected}]`——LLM 声明它做出的每个可验证数学断言，`sympy_expr` 为 SymPy 可解析形式；`reply_markdown` 中用 `{{assert:N}}` 占位符锚定每条断言在文中的位置。
- **流式与验证的顺序契约**：`delta` 事件可先行渲染（占位符显示 ⏳ 验证中），流结束后后端逐条验证并发出 `assertion` 事件，前端据状态更新徽标——**任何断言在对应 assertion 事件到达且 status=verified 前绝不显示 ✅**。
- **验证算法（按断言类型）**：恒等式 → `simplify(lhs - rhs) == 0`；解集 → 解集无序集合比较；导数/积分 → 符号 diff/integrate 对照 `expected`；`expected` 由引擎按类型复算或语义比较，不信任 LLM 自报结果。
- 结果：通过 → ✅；失败 → ❌（修正标注后展示）；`sympy_expr` 解析失败 → ⚠️ 未验证。
- 悬挂占位符兜底：断言事件超时（默认 30s）未到达或声明数少于占位符数 → 对应锚定按 ⚠️ 未验证处理，不悬挂 ⏳。
- 不变量：验证失败/未验证的断言绝不展示为"已验证"。

### 4.3 纠错模式（需求 4）

当阶段 1 检测到问题中含错误答案/错误假设：

1. 区分三种错误：**问题本身含错误假设** / **推理过程错** / **结论错但过程对**——处理方式不同。
2. 给出**可能的错误联想路径**：推断用户为何会这样想（如把 (a+b)² 错写成 a²+b² → 分配律滥用），用**最小反例**纠正，不直接说"错了"。
3. 阶段 2 纠错模式结构化输出 `error_type / cause / example`（error_type ∈ hypothesis / process / conclusion），写入 misconception 表（§6/§7），反哺测试选题（§5）。
4. 纠错后提供一道**同型验证题**（可选按钮，生命周期接口见 §8 `/api/exercise/*`），答对才记 `掌握`；答错记 `mistaken`（命中已知 misconception）或 `confused`（其他）。

### 4.4 绘图数据流

`plot_spec` → `POST /api/plot`：SymPy `lambdify` 采样（函数/参数/极坐标/隐式）→ 返回点集 → JSXGraph 渲染。滑动条防抖 100ms 重算（localhost 延迟可忽略）。回答中的公式可一键"画出来"（公式 → `/api/plot/from-expr` → plot_spec → 同一渲染路径）。**聊天触发的自动绘图**：阶段 1 的 `needs_plot` + `plot_spec` 随 SSE `meta` 事件下发（§8），前端据此渲染绘图面板。

## 5. 入口测试（需求 1）

- 题库：LLM 按档位生成 + SymPy 验证答案，存 `question` 表（DDL 见 §7）；**容量策略**：每档位上限（默认 50 题），超出按 `used_count` LRU 淘汰；**生成归属**：`/api/assess/next` 命中缓存直接返回，未命中时同步生成（LLM + SymPy 验证）后缓存——首启首次取题有一次生成延迟；**选题规则**：按档位随机抽取，该档位高频 misconception 关联知识点加权（权重 = 1 + 0.5·log(count)，见 §6）。题库自动增长机制属后续可选，见 §12。
- 首次启动展示向导：选年龄段/学历 → 对应档位题库抽 3-5 题。
- 每题测两个维度：**概念熟悉度**（"听过这个吗？"→ 术语接受度估计）+ **计算正确性**（算一遍）。
- 结果 → 画像档位（小学/初中/高中/大学/研究）+ 三维系数（familiarity / computation / terminology），注入 system prompt 控制解释的专业程度。**档位规则**：`level_band` 初值 = 用户所选学历；测试后**解释档位 = 基础档位经系数调整**（确定性规则）：`familiarity` 与 `terminology` 均 ≥ 0.7 且 `computation` ≥ 0.7 → 升一档；任一系数 ≤ 0.3 → 降一档；其余保持，结果夹在 primary..research 范围。测试结果与所选学历冲突时以测试结果为准并提示用户。`age_band` 与知识水平正交：仅用于语风（child/teen → 儿童语气，adult/researcher → 成人语气）与内容边界（未成年人），不参与难度推导。
- **维度映射与聚合**：概念题（"听过这个吗？"）更新 `familiarity`，并据"能看懂符号吗"的回答同步更新 `terminology` 估计；计算题更新 `computation`；3-5 题结果取均值（熟悉/正确 = 1，不熟/错误 = 0，部分 = 0.5）。
- 可跳过（默认档位：高中）；可随时在设置中重测。

**首启流程（无 key 门禁）**：
- 未配置 key → 应用首页为**配置引导页**（BYOK 录入）；聊天/测试均依赖 LLM，无 key 时不开放。
- 有 key 未测试 → 显示测试向导（可跳过）。
- 已测试 → 聊天主页。

## 6. 记忆系统

三层核心记忆（会话历史 / 知识点时间线 / 用户画像）+ 错误联想库（衍生存储），全部本地 SQLite：

| 层 | 内容 | 用途 |
|---|---|---|
| 会话历史 | 全部消息（含 topics 标注），跨会话全文检索 | "之前问过 X"、相关历史摘要 |
| 知识点时间线 | 知识点 + 状态（问过/困惑/答错/掌握）+ 时间与次数 | 关联卡片、难度参考 |
| 用户画像 | 档位 + 三维系数 | 控制解释专业程度 |
| 错误联想库 | misconception 表（kp、原因、最小反例、次数） | 纠错模式上下文 + 反哺测试选题 |

生命周期：

- 写入：阶段 2 结束后异步写入（知识点归一化合并别名、画像增量更新、misconception 累计计数）。
- 读取：阶段 2 上下文组装时读取（该知识点历史 + 相关知识点 ≤3 + 相关问答摘要 + 高频 misconception）。
- 检索实现：历史全文检索优先 SQLite FTS5（虚拟表与同步触发器随主表在实现时创建），否则 LIKE 兜底。
- 管理：记忆面板可见、可编辑、一键清空、导出 JSON（导出结构：`{profile, topics, messages, misconceptions}`）。**清空范围**：会话历史、知识点、画像、misconception、usage_log；**保留 api_key**（避免清空后无法使用）；清空后 profile 回到未测试态，重新触发测试向导。
- 审计：所有更新走 `memory_event` 追加式日志；**例外：一键清空为整表删除**（隐私优先，不保留审计）；回滚语义 = 清空 + `POST /api/memory/import` 导入导出的 JSON 快照（设置页"从备份恢复"）。

## 7. 数据模型（SQLite DDL）

```sql
-- 用户画像（单用户本地应用，user_id 固定 'local'）
CREATE TABLE profile (
  user_id        TEXT PRIMARY KEY,
  age_band       TEXT,                -- child / teen / adult / researcher
  level_band     TEXT,                -- primary / junior / senior / college / research
  familiarity    REAL DEFAULT 0.5,    -- 概念熟悉度 0~1
  computation    REAL DEFAULT 0.5,    -- 计算正确性 0~1
  terminology    REAL DEFAULT 0.5,    -- 术语接受度 0~1
  updated_at     TEXT
);

-- 知识点记忆
CREATE TABLE knowledge_point (
  id             TEXT PRIMARY KEY,    -- slug，如 binomial-expansion
  name           TEXT NOT NULL,       -- 展示名
  domain         TEXT,                -- algebra / calculus / geometry / ...
  aliases        TEXT DEFAULT '[]',   -- JSON 数组，归一化合并用
  first_seen_at  TEXT,
  updated_at     TEXT
);

CREATE TABLE kp_status (
  kp_id          TEXT PRIMARY KEY REFERENCES knowledge_point(id),
  status         TEXT DEFAULT 'asked',  -- asked / confused / mistaken / mastered
  last_asked_at  TEXT,
  ask_count      INTEGER DEFAULT 0,
  mistake_count  INTEGER DEFAULT 0
);

-- 错误联想库（纠错模式上下文 + 反哺测试选题）
CREATE TABLE misconception (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  kp_id          TEXT REFERENCES knowledge_point(id),
  error_type     TEXT,                -- hypothesis / process / conclusion
  cause          TEXT,                -- 错误联想原因（如 分配律滥用）
  example        TEXT,                -- 最小反例
  count          INTEGER DEFAULT 1,   -- 出现次数，选题按此排序
  last_seen_at   TEXT
);

-- 入口测试题库（固定容量缓存，见 §5）
CREATE TABLE question (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  level_band     TEXT,                -- 对应档位
  kind           TEXT,                -- concept / computation
  body           TEXT NOT NULL,       -- JSON：题干 + 选项/期望答案
  answer_verified TEXT,               -- SymPy 验证过的答案（仅 computation 类必填；concept 类为空——概念题为经验性判断，无计算答案）
  used_count     INTEGER DEFAULT 0,   -- 出题次数，LRU 淘汰依据
  created_at     TEXT
);

-- 会话历史
CREATE TABLE message (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id     TEXT,
  role           TEXT,                -- user / assistant / system
  content_md     TEXT,
  topics_json    TEXT DEFAULT '[]',
  created_at     TEXT
);
CREATE INDEX idx_msg_session ON message(session_id, created_at);

-- 记忆事件（追加式审计日志）
CREATE TABLE memory_event (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  type           TEXT,                -- profile_update / kp_update / export（清空不留审计，见 §6）
  payload        TEXT,                -- JSON
  created_at     TEXT
);

-- API key（keyring 句柄，不存明文）
CREATE TABLE api_key (
  provider       TEXT PRIMARY KEY,    -- anthropic / openai / deepseek / custom
  key_ref        TEXT,                -- keyring 服务名/用户名
  base_url       TEXT,                -- OpenAI-compatible 自定义端点
  model          TEXT,
  created_at     TEXT
);

-- LLM 用量记账（本机，见 §9）
CREATE TABLE usage_log (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  provider       TEXT,
  model          TEXT,
  tokens_in      INTEGER,
  tokens_out     INTEGER,
  latency_ms     INTEGER,
  created_at     TEXT
);
```

## 8. API 设计（localhost）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/chat` | 阶段 1+2 完整管线，SSE 流式（请求体 `{message, session_id?}`） |
| POST | `/api/plot/from-expr` | 公式 → plot_spec（复用阶段 1 分析的 needs_plot 分支，"画出来"按钮入口） |
| POST | `/api/assess/submit` | 入门测试逐题提交，返回画像更新（向导首步同时提交 age_band / level_band） |
| GET | `/api/assess/next` | 取下一道测试题（按档位选题；缓存未命中时同步生成） |
| POST | `/api/exercise/generate` | 生成同型验证题（LLM + SymPy 验证答案），请求体 `{kp_id?}` |
| POST | `/api/exercise/submit` | 提交习题答案：SymPy 与已验证答案比较判定 → 更新 kp_status（答对→mastered；答错→mistaken 命中已知 misconception / confused 其他）→ 返回判定与反馈 |
| POST | `/api/plot` | plot_spec → 点集（SymPy 采样） |
| GET | `/api/memory/topics` | 知识点时间线 |
| GET | `/api/memory/history?q=` | 会话历史检索 |
| GET | `/api/memory/profile` | 画像读取 |
| PUT | `/api/memory` | 编辑记忆条目 |
| DELETE | `/api/memory` | 一键清空 |
| GET | `/api/memory/export` | 导出 JSON |
| POST | `/api/memory/import` | 导入 JSON 快照（回滚/从备份恢复） |
| GET/POST | `/api/settings/keys` | BYOK 多厂商录入/列表（掩码显示，不返回明文） |
| DELETE | `/api/settings/keys/{provider}` | 删除某厂商 key；主 provider 由设置页当前选中项决定（存 config） |
| POST | `/api/settings/retest` | 触发重测 |
| GET | `/api/usage` | 本机记账（每 key 用量/费用估算） |

**SSE 事件契约（/api/chat）**：

```
event: meta       {question_type, contains_error, topics, needs_plot, plot_spec?}
event: assertion  {index, claim, status: verified|failed|unverified}
event: delta      {text}              -- reply_markdown 流式增量
event: card       {related_topics: [...], history_link: {topic, session_ref}}
event: exercise   {followup_exercise}
event: done       {}
event: error      {code, message}
```

上表为事件**类型目录**而非时序；实际顺序见 §4.2：`meta` → `delta` 流（占位符 ⏳）→ 流结束后的 `assertion` → `card`/`exercise` → `done`。

- `exercise` 事件：纠错模式回答末尾内联附一道同型题；"再练一道"按钮 → `POST /api/exercise/generate` 显式请求另一道（按钮是唯一显式触发）。

错误约定：统一 `{"error": {"code", "message", "detail?"}}`；LLM 调用失败时聊天接口降级为"重试中"，不中断会话。

## 9. 错误处理与验证状态

- LLM 调用失败 → 重试一次 → 仍失败返回"重试中"提示。
- 结构化输出解析失败 → 带上次输出重试一次。
- SymPy 验证失败区分两种：**断言错误**（LLM 幻觉）→ 修正标注后展示；**验证器无能为力** → ⚠️ 未验证标注。
- 不变量：**验证失败的断言绝不展示为"已验证"**。
- 未成年人/隐私：数据仅存本机；记忆可一键清空；导出功能供用户自持数据。
- 成本：本机记账（记录每次调用的 token 数与耗时，存 `usage_log`）；费用估算需用户在设置页配置各厂商单价（存 `~/.matheye/config.json`），未配置时仅显示用量。

## 10. 测试策略

| 层 | 工具 | 重点用例 |
|---|---|---|
| 分析器 | pytest + mock LLM | 输出 schema 校验、错误检测触发、compute 路由 |
| 记忆 | pytest | 知识点归一化合并别名、画像增量更新、事件追加 |
| 数学引擎 | pytest | SymPy 往返一致性（parse_expr↔lambdify）、验证器正反例 |
| 管线集成 | pytest | 阶段 1→2 全链路、纠错模式分支、验证徽标、断言锚定（占位符↔assertion 事件） |
| LLM 网关 | pytest + mock | 重试、结构化输出解析失败重试、失败降级（"重试中"） |
| SSE 契约 | pytest | 事件序列与载荷 schema（meta/delta/assertion/card/exercise/done/error） |
| E2E | Playwright | 启动 → 入门测试 → 提问 → 绘图 → 关联卡片 → 记忆更新闭环 |

关键不变量（写成自动化断言）：

1. `memory_event` 只增不改（例外：一键清空为整表删除，见 §6）；
2. 验证失败的断言不展示为"已验证"；
3. 任何数学结论仅在对应 assertion 事件 status=verified 后显示 ✅（LLM 不判定数学正确性）；验证前/失败显式标注 ⏳/❌/⚠️。

## 11. UI 布局（5 区块，中文界面）

1. **对话区**：消息流，KaTeX 渲染 + 验证徽标 + 关联卡片（可点击追问）。
2. **动态图面板**：JSXGraph，滑动条/缩放/拖拽；回答中公式可一键"画出来"。
3. **记忆面板**：画像档位、知识点时间线、历史检索；可编辑/清空/导出。
4. **设置**：BYOK 多厂商录入（掩码显示）、模型选择、重测入口、用量统计。
5. **入门测试向导**：首次启动引导。

各区块对应接口：对话区 → `/api/chat`；绘图面板 → `/api/plot`、`/api/plot/from-expr`；记忆面板 → `/api/memory/*`；设置 → `/api/settings/*`、`/api/usage`；测试/习题 → `/api/assess/*`、`/api/exercise/*`。

## 12. 后续可选（不阻塞 MVP）

- PyInstaller 打包为自包含桌面应用（架构不变）。
- 局域网访问（手机同网使用，FastAPI 绑定 0.0.0.0 + 只读鉴权）。
- 向量语义记忆（本地 embedding 或用户 key 调 embedding API）。
- 题库自动增长机制（MVP 为按需生成的固定容量缓存）。
