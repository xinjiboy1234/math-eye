# MathEye 详细设计文档

> 日期：2026-08-25
> 状态：设计阶段
> 上游文档：[技术选型评估报告](./2026-08-25-math-eye-tech-evaluation.md)
> 本文在评估报告基础上细化各子系统的模块划分、算法、数据模型与接口定义。

## 1. 系统模块划分

```
math-eye/
├── frontend/                 # Next.js 应用
│   ├── app/
│   │   ├── graph/            # 知识图谱页（迷雾地图）
│   │   ├── node/[id]/        # 知识点详情页（讲解 + 练习 + 实验）
│   │   ├── tutor/            # AI 答疑对话页
│   │   └── lab/              # 自由实验台（LaTeX + 沙箱 + 图像）
│   └── components/
│       ├── MathRenderer      # KaTeX 封装（含点击编辑）
│       ├── GraphCanvas       # Cytoscape.js 封装（高亮/迷雾逻辑）
│       ├── FunctionPlot      # JSXGraph 封装（参数滑动条联动）
│       ├── SandboxPanel      # Pyodide 加载与执行 UI
│       └── TutorChat         # 答疑对话组件（渲染结构化消息）
├── backend/                  # FastAPI 应用
│   ├── api/                  # 路由层（薄，只做校验与编排）
│   ├── services/
│   │   ├── tutor.py          # 答疑编排：画像组装 → LLM → 结构化解析
│   │   ├── mastery.py        # 水平模型：事件流 → mastery 更新
│   │   ├── graph.py          # 图谱服务：CRUD、高亮计算、重组提议
│   │   └── compute.py        # SymPy 计算代理
│   ├── llm/
│   │   ├── gateway.py        # LLM API 网关（重试、限流、成本记账）
│   │   ├── prompts/          # 分年龄段/分任务的 prompt 模板
│   │   └── schemas.py        # LLM 结构化输出的 Pydantic 模型
│   └── db/                   # SQLAlchemy 模型 + 迁移
└── docs/
```

模块边界原则：前端永不直接调用 LLM；`services/` 之间通过领域事件解耦
（如学习事件先落库，mastery 更新异步消费）。

## 2. 数据模型（完整 DDL）

```sql
-- 用户
users (
  id            TEXT PRIMARY KEY,          -- UUID
  age_band      TEXT,                      -- child / teen / adult / researcher
  created_at    TIMESTAMP
)

-- 知识点
knowledge_node (
  id            TEXT PRIMARY KEY,
  title         TEXT NOT NULL,
  description   TEXT,
  domain        TEXT NOT NULL,             -- 如 algebra / calculus / linear-algebra
  difficulty_tier INTEGER NOT NULL,        -- 同一知识点的分层难度 1~5
  embedding     BLOB,                      -- float32 向量，用于相似检索
  status        TEXT DEFAULT 'active',     -- active / draft / archived
  created_at    TIMESTAMP,
  updated_at    TIMESTAMP
)

-- 先修关系（有向边：from 是 to 的先修）
knowledge_edge (
  id            TEXT PRIMARY KEY,
  from_node_id  TEXT REFERENCES knowledge_node(id),
  to_node_id    TEXT REFERENCES knowledge_node(id),
  weight        REAL DEFAULT 1.0,          -- 相关强度
  status        TEXT DEFAULT 'confirmed',  -- confirmed / proposed / rejected
  proposed_by   TEXT,                      -- human / llm
  created_at    TIMESTAMP,
  UNIQUE(from_node_id, to_node_id)
)

-- 内容包：同一知识点挂多难度素材
content_item (
  id            TEXT PRIMARY KEY,
  node_id       TEXT REFERENCES knowledge_node(id),
  tier          INTEGER NOT NULL,          -- 对应 difficulty_tier
  kind          TEXT NOT NULL,             -- explanation / exercise / challenge
  body          JSON NOT NULL,             -- 结构化内容（LaTeX、代码、选项等）
  answer        JSON                       -- 习题答案（SymPy 可验证形式优先）
)

-- 掌握度
mastery (
  user_id       TEXT REFERENCES users(id),
  node_id       TEXT REFERENCES knowledge_node(id),
  score         REAL DEFAULT 0.0,          -- 0~1
  attempt_count INTEGER DEFAULT 0,
  last_reviewed TIMESTAMP,
  decay_rate    REAL DEFAULT 0.0,          -- 阶段2遗忘曲线参数
  PRIMARY KEY (user_id, node_id)
)

-- 学习事件（不可变日志，水平模型的唯一数据来源）
learning_event (
  id            BIGSERIAL PRIMARY KEY,
  user_id       TEXT REFERENCES users(id),
  node_id       TEXT REFERENCES knowledge_node(id),
  event_type    TEXT NOT NULL,
  -- answer_correct / answer_wrong / hint_used / explanation_read /
  -- sandbox_run / challenge_solved / tutor_question
  payload       JSON,
  created_at    TIMESTAMP
)
CREATE INDEX idx_event_user_time ON learning_event(user_id, created_at);
```

SQLite（MVP）与 Postgres 共用此结构；`BIGSERIAL` 在 SQLite 中退化为
`INTEGER PRIMARY KEY`。

## 3. 核心算法

### 3.1 掌握度更新（Elo 式，阶段 1）

每次学习事件后：

```
expected = 1 / (1 + 10 ** ((node_difficulty - user_rating) / 400))
score_new = score_old + K * (outcome - expected)
```

- `user_rating` 由该节点 mastery 映射而来（`rating = 800 + score * 1200`）
- `K` 动态取值：前 5 次尝试 K=40（快速收敛），之后 K=16
- `outcome`：答对=1，答错=0，用提示后答对=0.5
- 附带事件加权：`challenge_solved` 权重 ×1.5，纯阅读 ×0.3

### 3.2 高亮推荐（规则引擎）

对每个未掌握节点计算可学习性：

```
ready(node) = min( mastery(p) for p in prerequisites(node) )
```

状态分类：

| 条件 | 状态 | 前端表现 |
|---|---|---|
| 所有前置 mastery ≥ 0.7 且自身 < 0.3 | **可学习** | 高亮发光 |
| 存在前置 mastery ∈ [0.4, 0.7) | **需巩固前置** | 半高亮 + 提示哪个前置薄弱 |
| 自身 mastery ∈ [0.3, 0.8) | **进行中** | 常规显示进度环 |
| 自身 mastery ≥ 0.8 且超 14 天未复习 | **待复习** | 黄色标记 |
| 任一前置 mastery < 0.4 | **锁定** | 迷雾遮蔽 |

推荐排序：`priority = unlock_gain * (1 - avg_prereq_mastery) / difficulty_tier`
，取 top-N（默认 5 个）作为"当前任务列表"。

### 3.3 图谱自动重组（LLM 提议 + 人工确认）

```
输入新知识点 N:
1. embed(N.title + description) → 向量
2. 余弦相似度检索 top-K 相关节点（K=8）
3. LLM 输入：N 的描述 + K 个候选节点及其现有邻接关系
   LLM 输出（JSON）：[{from, to, relation, confidence, rationale}]
4. confidence ≥ 0.85 的提议写入 knowledge_edge(status='proposed')
5. 人工审核界面批量确认/拒绝
```

禁止路径：LLM 输出永远不直接写入 `status='confirmed'`。

### 3.4 答疑编排流程

```
POST /api/tutor/chat { node_id?, question }
  ↓
1. 组装上下文：
   - 用户画像：age_band + 相关节点 mastery + 近 20 条学习事件摘要
   - 知识点上下文：node 详情 + 当前 tier 的 content_item
   - 语风模板：prompts/{age_band}.md
  ↓
2. 判断问题类型（LLM 分类，一次廉价调用或本地规则）：
   - factual_compute → 路由 SymPy，不走生成
   - conceptual     → 苏格拉底式引导（child 模式不给直接答案）
   - proof/research → 直接深度讨论（researcher 模式）
  ↓
3. LLM 生成，强制 JSON Schema 输出：
   {
     "reply_markdown": "...",        // 含 LaTeX
     "knowledge_nodes": ["id"],      // 涉及知识点
     "difficulty_rating": 3,
     "socratic_stage": "hint|probe|reveal",
     "followup_exercise": {...}?
   }
  ↓
4. reply 中数学表达式抽取 → SymPy 复核（仅验证等式/恒等式类断言）
5. 学习事件(tutor_question)落库 → 触发 mastery 更新
6. 返回结构化消息给 TutorChat 渲染
```

## 4. API 规格

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/register` | 创建用户（匿名 UUID 即可用，后续绑定账号） |
| GET | `/api/graph?view=full\|fog` | fog 模式下锁定节点只返回 ID 不返回内容 |
| GET | `/api/nodes/{id}?tier=` | 知识点详情 + 指定难度内容包 |
| GET | `/api/nodes/{id}/prerequisites` | 前置链及用户各环节掌握度 |
| POST | `/api/events` | 批量上报学习事件，返回更新后的 mastery |
| GET | `/api/recommendations?limit=5` | 当前任务列表（高亮排序结果） |
| POST | `/api/tutor/chat` | 答疑（见 §3.4），SSE 流式返回 reply 部分 |
| POST | `/api/compute` | SymPy 代理：`{expr_latex, op}`，op ∈ solve/simplify/diff/integrate/plot_data |
| POST | `/api/graph/nodes` | 新建知识点（触发 §3.3 重组提议） |
| GET | `/api/graph/proposals` | 待审核的边提议列表 |
| POST | `/api/graph/proposals/{id}/decision` | confirm / reject |

错误约定：统一 `{ "error": { "code", "message", "detail?" } }`；
LLM 调用失败时答疑接口降级为"重试中"，不中断会话。

## 5. 前端交互细节

**图谱页（迷雾地图）**
- Cytoscape.js，力导向布局；锁定节点渲染为灰色剪影（不显示标题）
- 高亮呼吸动画区分"可学习"（主色）与"待复习"（黄色）
- 点击节点 → 侧滑抽屉显示详情 + 「开始学习」按钮

**知识点页**
- 上半区：KaTeX 讲解 + 内联可编辑公式（点击进入 LaTeX 编辑态）
- 下半区 Tab：练习 / 实验（FunctionPlot + SandboxPanel 联动）/ 答疑入口
- 实验台：滑动条改变参数 → 防抖 100ms → Pyodide 重算采样点 → 图像更新

**Pyodide 加载策略**
- 路由级懒加载，仅在进入实验/练习页时初始化
- Service Worker 缓存 pyodide wasm 与核心包，二次访问秒开
- 加载失败降级：绘图改走 `/api/compute` 的 `plot_data`

## 6. 测试策略

| 层 | 工具 | 重点用例 |
|---|---|---|
| mastery 算法 | pytest 纯单元 | 收敛性、边界值、事件加权 |
| 高亮规则 | pytest + 固定图 fixture | 每种状态分类的边界阈值 |
| SymPy 代理 | pytest | LaTeX↔SymPy 转换往返一致性 |
| LLM 编排 | mock LLM + schema 校验 | JSON 解析失败重试、类型路由 |
| 图谱重组 | 录制回放测试 | 提议永不直写 confirmed（不变量断言） |
| E2E | Playwright | 学习循环：看讲解→答题→图谱高亮变化 |

关键不变量（写成自动化断言）：
1. `learning_event` 只增不改；
2. mastery 更新是事件的纯函数（可离线重放验证）;
3. LLM 提议的边必须经人工 decision 才生效。

## 7. 非功能需求

- **性能**：图谱 ≤2000 节点时交互 ≥50fps；答疑首 token ≤2s（SSE）
- **成本护栏**：每用户每日 LLM token 配额，超限降级为缓存回答 + SymPy-only 模式
- **隐私**：未成年人数据不留存对话原文，只保留事件统计；mastery 数据支持导出/删除
- **可观测性**：LLM 网关记录每次调用的 token 数与耗时，按 user/day 聚合报表

## 8. P1 里程碑验收标准（下一步实施目标）

1. 可导入一个种子领域（建议：线性代数前 6 章，约 60 个知识点）
2. 图谱页呈现迷雾地图，高亮逻辑符合 §3.2 全部规则
3. 注册 → 学习 → 答题 → mastery 变化 → 高亮更新的完整闭环
4. 全部不变量断言通过 CI
