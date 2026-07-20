# 第四部分 工程化与框架 · 详解

> 本文是《AI Agent 开发学习路线》第四部分（第 11-14 章）的逐条释义。聚焦把 Agent 从「demo 能跑」推进到「生产可用」的工程原理，代码以 Go 为主。

---

## 第 11 章 主流 Agent 开发框架

**先建立一个判断**：框架的价值是封装「LLM 调用 + 工具循环 + 记忆 + 编排」这些重复劳动，但也带来「黑盒、抽象泄漏、版本绑定」的代价。Agent 逻辑的核心（ReAct 循环、工具调用协议）本身并不复杂（见第一、二部分的代码），**是否用框架取决于团队规模与迭代速度，而非技术必要性**。Go 生态尤其如此——很多团队直接基于 HTTP + 自研轻量循环，而非套重型框架。

### 11.1 LangChain 核心组件与用法

- **定位**：Python 生态最流行的 LLM 应用开发框架，提供全套抽象。
- **核心抽象**：
  - **Model I/O**：统一不同 LLM 的调用接口（Prompt 模板、LLM/ChatModel、输出解析器）。
  - **Chain**：把多个步骤串成流水线（LCEL 表达式语言用 `|` 组合）。
  - **Retrieval**：文档加载、分块、向量存储、检索器（RAG 全套）。
  - **Agent + Tool**：封装工具调用循环。
- **核心原理**：用统一抽象屏蔽底层差异，代价是抽象层数多、调试链路长。
- **取舍**：适合快速搭 RAG/原型；生产环境常因抽象过重而被诟病「难调试、性能不透明」。

### 11.2 LangGraph 图式编排

- **定位**：LangChain 团队推出的**有状态、图式**编排框架，是对 Chain「线性流水线」局限的修正。
- **核心原理**：把 Agent 建模为**状态图（StateGraph）**——节点是计算单元（LLM 调用/工具/函数），边是转移条件，图中维护一个贯穿全程的 **State**。这天然支持循环（ReAct 的本质是带环的图）、条件分支、人在回路的中断/恢复。
- **为何重要**：Agent 的控制流本质是图而非链（有循环、有回退、有分支），LangGraph 用「图 + 显式状态」精确表达了这一点，是目前生产级 Agent 编排的主流范式。对应第二部分讲的状态机驱动思想。

### 11.3 LlamaIndex 数据框架

- **定位**：专注 **RAG / 数据接入** 的框架，强于「把私有数据喂给 LLM」。
- **核心能力**：数据连接器（各种数据源）、索引结构（向量索引、树索引、关键词索引、知识图谱索引）、查询引擎、Node 后处理（rerank、过滤）。
- **取舍**：RAG 场景比 LangChain 更专精、抽象更贴合数据检索；但 Agent 编排能力弱于 LangGraph。二者常组合使用。

### 11.4 AutoGen 多 Agent 框架

- **定位**：微软出品，专注**多 Agent 对话式协作**。
- **核心原理**：把一切协作建模为 Agent 之间的「对话」，通过 `ConversableAgent` 收发消息，用 GroupChat + GroupChatManager 管理群聊发言权（对应第三部分 9.6 群聊模式）。内置 UserProxyAgent 支持人在回路和代码执行。
- **取舍**：多 Agent 实验/研究场景强大；对话驱动的范式在复杂确定性流程中较难精确控制。

### 11.5 CrewAI 角色协作框架

- **定位**：以「**角色扮演的团队**」为核心隐喻的多 Agent 框架。
- **核心抽象**：Agent（有 role/goal/backstory）、Task（任务）、Crew（团队）、Process（协作流程：sequential 顺序 / hierarchical 层级）。
- **核心原理**：把第三部分的「按角色分工 + Orchestrator 编排」封装成声明式 API，开发者只需定义角色和任务，框架负责编排。上手快，但灵活度受框架约定限制。

### 11.6 Semantic Kernel

- **定位**：微软的企业级 SDK，支持 C#/Python/Java，强调与现有企业系统集成。
- **核心概念**：Plugin（技能封装）、Planner（自动规划调用哪些技能）、Memory、Connector。设计偏工程化、企业友好。

### 11.7 Dify / Coze 等低代码平台

- **定位**：可视化、低代码的 Agent/工作流搭建平台。
- **核心价值**：拖拽式编排、内置 RAG/工具/模型管理，让非专业开发者也能搭 Agent。
- **取舍**：起步极快、适合标准场景；深度定制、复杂逻辑、性能极致优化时会触及平台天花板，需要回退到代码开发。

### 11.8 OpenAI Agents SDK / Assistants API

- **定位**：OpenAI 官方的 Agent 构建方案。Assistants API 在服务端托管了线程（Thread）、工具（Code Interpreter、File Search、Function Calling）、运行状态；Agents SDK 是更轻量的编排层，核心概念是 Agent、Handoff（Agent 间移交）、Guardrail（输入输出护栏）。
- **取舍**：与 OpenAI 生态深度绑定、开箱即用；跨模型迁移性差。

### 11.9 框架选型对比与取舍

| 需求 | 推荐 | 原因 |
| --- | --- | --- |
| 快速搭 RAG 原型 | LlamaIndex / LangChain | RAG 抽象完备 |
| 生产级复杂编排 | LangGraph | 图 + 显式状态，可控可调试 |
| 多 Agent 研究 | AutoGen / CrewAI | 对话/角色协作原生支持 |
| 企业系统集成 | Semantic Kernel | 多语言、工程化 |
| 非开发者搭建 | Dify / Coze | 低代码可视化 |
| Go 后端服务 | 轻量自研 + MCP | 生态成熟框架少，自研循环更可控 |

**Go 团队的现实**：主流 Agent 框架以 Python 为主，Go 生态相对薄弱。多数 Go 后端选择**自研轻量 Agent 循环（第二部分的代码即为骨架）+ 标准协议（MCP/A2A）接入能力**，把可控性和性能握在自己手里。

---

## 第 12 章 Agent 工程化实践

这一章是 demo 与生产的分水岭。核心命题：**如何让一个概率性、有状态、慢且贵的系统变得可维护、可测试、可控成本。**

### 12.1 Agent 项目结构设计

清晰的分层，隔离「易变的 Prompt/模型」与「稳定的业务逻辑」：

```
agent/
├── prompt/         # Prompt 模板，独立管理、可版本化
├── tool/           # 工具实现，每个工具一个文件，实现统一接口
├── memory/         # 记忆存储与检索
├── llm/            # LLM Provider 抽象层，隔离具体厂商
├── loop/           # Agent 核心循环（ReAct/Plan-Execute）
├── router/         # 意图路由
└── observability/  # 追踪、日志、指标
```

**核心原则**：LLM Provider、Prompt、工具都应可插拔（依赖倒置），业务逻辑不直接依赖某个具体模型或框架。

### 12.2 配置管理与环境隔离

- 模型端点、API Key、超时、预算等全部走配置，不硬编码。
- 多环境（dev/test/prod）隔离，尤其是**模型选择**（dev 用便宜模型省钱，prod 用强模型）。
- **Secrets 仅从环境变量/密钥管理系统读取**（对应安全规范），绝不入代码库。

```go
type AgentConfig struct {
    ModelName    string        `env:"AGENT_MODEL"`
    MaxSteps     int           `env:"AGENT_MAX_STEPS" default:"10"`
    StepTimeout  time.Duration `env:"AGENT_STEP_TIMEOUT" default:"30s"`
    TokenBudget  int           `env:"AGENT_TOKEN_BUDGET" default:"50000"`
    // APIKey 仅从环境读取，绝不写入配置文件或代码
    APIKey       string        `env:"LLM_API_KEY"`
}
```

### 12.3 Prompt 版本管理

Prompt 是 Agent 的「源代码」，必须像代码一样管理：

- **纳入版本控制**，每次改动可追溯、可回滚。
- **版本化 + 灰度**：新 Prompt 先小流量验证，对比指标再全量。
- **与评估集绑定**：改 Prompt 必跑回归测试集（见第 14 章），防止「修一个坏三个」。

### 12.4 工具与能力的模块化组织

统一接口 + 注册表（第二部分 5.8），启动时按 Agent 职责组装不同工具集。工具应无状态、幂等、可独立测试。

### 12.5 依赖注入与可测试性设计

**LLM 调用是概率性的、慢的、花钱的，绝不能在单测里真实调用。** 必须把 LLM Provider 抽象成接口并注入，测试时用 mock 替换：

```go
// 生产用真实 LLM，测试注入 mock，实现确定性单测
type Agent struct {
    llm    LLMProvider // 接口，非具体实现
    memory MemoryStore
    tools  *Registry
}

// 测试用 mock：预设返回，让 Agent 循环逻辑可被确定性验证
type mockLLM struct {
    responses []ChatResponse
    idx       int
}

func (m *mockLLM) Chat(ctx context.Context, req ChatRequest) (ChatResponse, error) {
    r := m.responses[m.idx]
    m.idx++
    return r, nil // 按预设脚本返回，验证循环/工具调度是否正确
}
```

**核心原理**：把「不确定的 LLM 推理」与「确定的编排/工具/解析逻辑」解耦。后者才是你能写单测保证正确性的部分；前者用评估集（第 14 章）而非单测来保证质量。

### 12.6 异步与并发处理

- 独立工具调用并行执行（第二部分 5.4）。
- 多 Agent 并行（`errgroup` / `trpc.GoAndWait`）。
- 用 `context` 统一控制超时、取消、传递链路 ID。

```go
// 用 errgroup 并行执行独立子任务，任一失败即取消全部
func runParallel(ctx context.Context, tasks []Task) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]string, len(tasks))
    for i, t := range tasks {
        i, t := i, t
        g.Go(func() error {
            out, err := t.Run(ctx)
            if err != nil {
                return fmt.Errorf("任务 %d 失败: %w", i, err)
            }
            results[i] = out
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

### 12.7 流式输出（Streaming）实现

用户体验的关键：LLM 逐 token 生成，必须流式返回而非等全部生成完。后端用 SSE/WebSocket 把 token 增量推给前端（对应第三部分 AG-UI）。

**工程难点**：流式场景下如何解析结构化输出（工具调用 JSON 是逐片段到达的）？需要**增量 JSON 解析**或等待关键字段完整后再触发工具。

### 12.8 状态持久化与断点续跑

长任务/多轮会话必须持久化状态（当前步骤、中间结果、消息历史），支持：

- **崩溃恢复**：服务重启后从断点继续。
- **人在回路暂停**：等待用户确认时挂起状态，响应后恢复。

**核心原理**：把 Agent 的执行状态显式外部化为可序列化的数据结构（对应 LangGraph 的 State / checkpoint 思想），而非藏在内存调用栈里。

```go
// 可持久化的执行状态：序列化落库，支持恢复
type AgentState struct {
    SessionID    string    `json:"session_id"`
    Goal         string    `json:"goal"`
    Messages     []Message `json:"messages"`
    CurrentStep  int       `json:"current_step"`
    Results      map[int]string `json:"results"`
    Status       string    `json:"status"` // running/paused/completed/failed
}
```

### 12.9 成本控制与 Token 优化

**成本是 Agent 生产化最现实的约束**（多轮调用叠加，费用极易失控）：

- **模型分层**：简单任务用便宜模型（第一部分 2.9 的路由）。
- **上下文压缩**：滚动摘要、截断、选择性检索（第二部分 7.2）。
- **步数/预算熔断**：LoopController（第二部分 4.6）硬性封顶。
- **缓存**：见 12.10。
- **Prompt 精简**：去冗余描述，Few-shot 示例够用即可。

### 12.10 缓存策略（语义缓存、结果缓存）

- **精确缓存**：相同输入直接返回缓存结果（key = hash(prompt)）。
- **语义缓存**：语义相近的问题命中缓存（用 embedding 相似度匹配），命中率更高但需容忍近似。
- **工具结果缓存**：幂等工具的结果按参数缓存（如查询类）。

```go
// 语义缓存：新问题 embedding 与缓存问题比相似度，超阈值即命中
type SemanticCache struct {
    store     VectorStore
    threshold float64 // 相似度阈值，如 0.95
}

func (c *SemanticCache) Get(ctx context.Context, query string) (string, bool) {
    hits, err := c.store.Search(ctx, query, 1)
    if err != nil || len(hits) == 0 {
        return "", false
    }
    if hits[0].Score >= c.threshold { // 足够相似才命中，避免答非所问
        return hits[0].Answer, true
    }
    return "", false
}
```

---

## 第 13 章 可观测性与调试

**核心命题**：Agent 是黑盒 + 非确定性 + 多步，传统日志不足以定位问题。可观测性回答「它到底做了什么、为什么这么做、哪一步出错、花了多少」。

### 13.1 Agent 执行轨迹（Trace）追踪

**这是 Agent 可观测性的核心。** 一次 Agent 运行是一棵调用树：Run → 多个 Step → 每个 Step 含 LLM 调用 + 工具调用。必须完整记录每一步的输入、输出、耗时、token、决策依据（Thought）。

**核心原理**：借鉴分布式追踪的 **Trace/Span** 模型——一次运行是一个 Trace，每个 LLM 调用/工具调用是一个 Span，Span 有父子关系构成树，用统一的 trace_id 串联全链路。

```go
// Span：一次 LLM/工具调用的追踪单元，构成 Trace 树
type Span struct {
    TraceID   string          // 一次 Agent 运行的全局 ID
    SpanID    string
    ParentID  string          // 父 Span，构成调用树
    Name      string          // 如 "llm.chat"、"tool.query_tickets"
    Input     json.RawMessage
    Output    json.RawMessage
    StartTime time.Time
    Duration  time.Duration
    Tokens    int
    Error     string
}
```

### 13.2 日志设计与结构化日志

- **结构化日志**（JSON）而非纯文本，带 trace_id、session_id、step 等字段，便于检索关联。
- 遵循项目日志规范：带足够上下文、级别准确、敏感信息脱敏、高频路径避免日志轰炸。
- **记录关键决策点**：模型选了什么工具、为什么（Thought）、工具返回什么——这是排查「Agent 为何跑偏」的一手证据。

### 13.3 LangSmith / LangFuse 等观测平台

- **LangSmith**（LangChain 生态）、**LangFuse**（开源）：专为 LLM 应用设计的观测平台，可视化 Trace 树、记录每步 prompt/response/token/成本、支持标注与评估。
- 核心价值：把 13.1 的 Trace 数据可视化 + 可回放 + 可分析，是调试复杂 Agent 的利器。

### 13.4 指标监控（延迟、Token、成功率、成本）

必须监控的核心指标：

- **延迟**：端到端 + 各步（首 token 时间 TTFT、单步耗时）。
- **Token / 成本**：每次运行的消耗，用于成本告警。
- **成功率 / 完成率**：任务是否达成目标。
- **步数分布**：异常增多可能意味着 Agent 在打转。
- **工具调用统计**：调用频次、失败率、耗时。

### 13.5 中间步骤的可视化

把 Trace 树渲染成时间线/树形图，直观看到「Agent 想了什么 → 做了什么 → 得到什么」。这是理解非线性执行路径的必需品。

### 13.6 错误定位与回放

- **回放（Replay）**：保存完整 Trace（含 LLM 输入输出），能复现问题现场——因为 LLM 非确定，没有回放几乎无法复现偶发 bug。
- **分层定位**：先看是哪一步失败（编排层 / LLM 决策 / 工具执行 / 解析），再深入。

### 13.7 A/B 测试与灰度发布

- 新 Prompt / 新模型 / 新工具先灰度小流量，对比核心指标（成功率、成本、延迟、用户满意度）。
- 用真实流量验证，因为离线评估集无法覆盖所有真实分布。

---

## 第 14 章 评估与测试

**核心命题**：Agent 输出是概率性的、开放的，没有唯一正确答案，「断言相等」的传统测试方法失效。**没有评估体系，Agent 的迭代就是盲目试错。**

### 14.1 Agent 评估的挑战

- **非确定性**：同样输入多次输出可能不同。
- **开放性**：一个任务有多种正确解法/表述。
- **多步复合**：最终失败可能源于中间任一步，需要分步归因。
- **主观性**：「回答好不好」部分依赖人类判断。

### 14.2 端到端评估与分步评估

- **端到端评估**：只看最终结果是否达成目标（黑盒）。贴近真实价值，但失败时难定位。
- **分步评估**：评估每一步的质量（工具选对没、参数抽对没、检索准不准）。便于归因，但步步都对不代表整体成功。
- **实践**：两者结合——端到端看价值，分步看归因。

### 14.3 基准数据集与测试集构建

- 收集真实/构造的「输入 + 期望结果（或评判标准）」样本集。
- 覆盖正常场景、边界场景、对抗场景、历史 badcase。
- **测试集是评估的地基**，质量比数量重要；持续用线上 badcase 补充。

### 14.4 LLM-as-a-Judge 评估法

**用一个强 LLM 来评判 Agent 输出的质量**——解决开放性输出无法用规则判分的难题。

**核心原理**：给评判模型提供评分标准（rubric）、原始问题、Agent 答案（可选参考答案），让它按维度打分并给理由。

```go
// LLM-as-a-Judge：用强模型按 rubric 对开放性答案打分
func judge(ctx context.Context, judgeLLM LLMProvider, question, answer string) (Score, error) {
    prompt := fmt.Sprintf(`你是严格的评估专家。按以下维度对答案打分（1-5）并说明理由，输出 JSON：
- 相关性：是否切题
- 准确性：事实是否正确
- 完整性：是否完整回答

问题：%s
答案：%s`, question, answer)
    resp, err := judgeLLM.Chat(ctx, ChatRequest{
        Messages:    []Message{{Role: "user", Content: prompt}},
        Temperature: 0, // 评判要稳定可复现
    })
    if err != nil {
        return Score{}, fmt.Errorf("评判失败: %w", err)
    }
    return parseScore(resp.Content)
}
```

**注意事项（关键）**：Judge 本身有偏差——位置偏好（偏好靠前答案）、长度偏好（偏好更长答案）、自我偏好（偏好同源模型输出）。需通过打乱顺序、控制长度、多次评判取平均等手段缓解。

### 14.5 人工评估与标注

LLM-as-a-Judge 无法完全替代人。高价值/高风险场景需人工评估，并用人工标注结果**校准 Judge 模型**（对齐人类偏好）。

### 14.6 任务完成率、准确率、鲁棒性指标

- **任务完成率**：达成目标的比例（最重要的业务指标）。
- **准确率**：有标准答案时的正确率。
- **鲁棒性**：对输入扰动（错别字、歧义、异常输入）的稳定性。
- **效率**：完成任务的平均步数/token/耗时。

### 14.7 回归测试与持续评估

- 每次改 Prompt/模型/工具，跑评估集做回归，量化影响再决定是否上线。
- 接入 CI：评估指标低于阈值则阻断合入（对应项目「CI 通过才能合并」的规范）。
- **持续评估**：线上采样 + 定期跑评估集，监控质量漂移（模型升级、数据分布变化都可能导致退化）。

### 14.8 对抗性测试

主动构造攻击性/边界输入测试鲁棒性与安全：Prompt Injection、越狱尝试、超长输入、矛盾指令、诱导越权。目的是在上线前暴露安全与稳定性缺陷（对应第五部分安全章节）。

### 14.9 评估工具与平台

- **RAGAS**：专注 RAG 评估（忠实度、答案相关性、上下文精度/召回）。
- **LangFuse / LangSmith**：评估 + 观测一体。
- **自建评估流水线**：结合测试集 + LLM-as-a-Judge + 指标看板，接入 CI。

---

## 小结

第四部分是「让 Agent 活下来」的工程内核：

- **框架（第 11 章）**：LangGraph 的「图 + 显式状态」代表了生产级编排的方向；Go 团队现实是自研轻量循环 + 标准协议。框架是加速器，不是必需品。
- **工程化（第 12 章）**：核心是**解耦不确定的 LLM 与确定的业务逻辑**（依赖注入让后者可测），加上配置化、Prompt 版本管理、状态持久化、成本控制与缓存。
- **可观测性（第 13 章）**：Trace/Span 模型是理解非确定性黑盒的钥匙——没有完整 Trace 就无法回放，无法回放就无法调试偶发问题。
- **评估（第 14 章）**：传统「断言相等」失效，用**分步 + 端到端评估 + LLM-as-a-Judge + 人工校准 + 回归测试**构建质量护城河。**没有评估，迭代即盲目。**

一条贯穿主线：**Agent 的非确定性，要求工程上用「可观测（Trace）+ 可评估（Judge/指标）+ 可回放 + 可回归」四件套来驯服，而不是靠传统的确定性测试。**
