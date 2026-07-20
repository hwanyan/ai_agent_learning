# 第三部分 多 Agent 与协作 · 详解

> 本文是《AI Agent 开发学习路线》第三部分（第 9-10 章）的逐条释义。聚焦多 Agent 系统的核心原理与通信协议的底层机制，代码以 Go 为主。

---

## 第 9 章 多 Agent 系统

### 9.1 多 Agent 系统的动机与优势

单 Agent 在三种情况下会遇到硬性瓶颈，这是拆分多 Agent 的**根本动机**，而非「显得高级」：

1. **上下文污染**：一个 Agent 塞入所有工具（几十上百个）、所有领域知识、所有历史，导致：
   - 工具选择准确率随工具数量增加而**显著下降**（模型在长列表中选错概率上升）。
   - 上下文被无关信息稀释，关键指令的「注意力权重」被摊薄。
2. **职责冲突**：同一个 System Prompt 里塞入「你要严谨审查」和「你要大胆创新」这类矛盾人设，模型行为会震荡。
3. **能力与成本错配**：简单子任务不该用最强最贵的模型，但单 Agent 无法在一次任务内切换模型。

多 Agent 的本质优势是**关注点分离（Separation of Concerns）在 LLM 语境下的复现**：每个 Agent 拥有独立的、精简的上下文（专属 System Prompt + 专属工具子集 + 专属记忆），从而各自达到更高的可靠性。

**关键权衡（务必牢记）**：多 Agent 不是免费的。它引入了三类新成本：
- **通信开销**：Agent 间传递信息本身要消耗 token 和往返延迟。
- **错误累积**：N 个 Agent 串联，端到端成功率是各环节成功率的连乘。
- **一致性难题**：分布式的上下文难以保持全局一致，容易出现「A 以为的和 B 以为的不一样」。

工程铁律：**能用单 Agent + 好的工具组织解决的，不要拆多 Agent。只有当上下文确实装不下、或职责确实互斥时才拆。**

### 9.2 Agent 角色分工与职责划分

划分角色的两种主流维度：

- **按职能（Functional）**：规划者、执行者、审查者、总结者。类比软件团队的架构师/程序员/测试。
- **按领域（Domain）**：SQL Agent、代码 Agent、检索 Agent、邮件 Agent。类比按业务线分工。

划分的核心原则是**高内聚、低耦合、单一职责**：一个 Agent 的 System Prompt 应该能用一句话概括它的职责。如果你需要用「并且」「同时还要」来描述一个 Agent，它大概率该被拆分。

```go
// 每个 Agent = 独立人设 + 独立工具子集 + 独立模型选择
type AgentSpec struct {
    Name         string
    SystemPrompt string       // 单一、明确的职责描述
    Tools        []ToolHandler // 只给它职责所需的工具
    Model        LLMProvider   // 按职责复杂度选模型：审查用强模型，格式化用弱模型
}

var plannerSpec = AgentSpec{
    Name:         "planner",
    SystemPrompt: "你是任务规划专家。只负责把用户目标拆解为有序、可执行的子任务清单，不负责执行。",
    Tools:        nil, // 规划者通常不需要工具，只做推理
    Model:        strongModel,
}

var executorSpec = AgentSpec{
    Name:         "executor",
    SystemPrompt: "你是任务执行者。严格按给定的单个子任务调用工具完成，不做额外规划。",
    Tools:        []ToolHandler{queryTool, updateTool},
    Model:        cheapModel, // 执行确定性强，可用便宜模型
}
```

### 9.3 Agent 间通信协议与消息传递

通信是多 Agent 系统的「神经系统」，核心要解决三个问题：**传什么、怎么路由、如何保证时序**。

- **消息内容（传什么）**：不能只传自然语言（易歧义、难解析），生产系统通常传**结构化消息**：发送方、接收方、消息类型、载荷、关联 ID（用于串联一次任务的多条消息）。
- **路由（怎么找到接收方）**：点对点（指定接收者）、发布订阅（按 topic）、黑板（共享空间广播）。
- **时序与一致性**：异步消息需处理乱序、重复（幂等）、丢失（重试/超时）。

```go
// 结构化 Agent 消息：ConversationID 用于把一次任务的所有往返消息串成一条链路
type AgentMessage struct {
    ID             string          `json:"id"`
    ConversationID string          `json:"conversation_id"` // 追踪一次协作的全链路
    From           string          `json:"from"`
    To             string          `json:"to"` // 空表示广播
    Type           MessageType     `json:"type"` // request/response/inform/error
    Payload        json.RawMessage `json:"payload"`
    Timestamp      int64           `json:"timestamp"`
}

type MessageType string
const (
    MsgRequest  MessageType = "request"  // 请求对方做事
    MsgResponse MessageType = "response" // 对请求的答复
    MsgInform   MessageType = "inform"   // 单向通知，不需回复
    MsgError    MessageType = "error"    // 错误上报
)

// 消息总线：负责路由与投递，是解耦 Agent 的关键设施
type MessageBus interface {
    Publish(ctx context.Context, msg AgentMessage) error
    Subscribe(agentName string) (<-chan AgentMessage, error)
}
```

**核心原理**：把 Agent 之间的直接函数调用改为通过消息总线的异步通信，实现**时间解耦（不必同时在线）、空间解耦（不必知道对方在哪）、实现解耦（不必知道对方内部逻辑）**。这与微服务架构的消息中间件同理。

### 9.4 协作模式（协作、竞争、辩论）

- **协作（Cooperation）**：Agent 各做一部分，拼成完整结果。最常见。
- **竞争（Competition）**：多个 Agent 用不同方法解同一问题，取最优解。用冗余换质量。
- **辩论（Debate）**：多个 Agent 对同一问题给出观点并相互批判、迭代，最后收敛。

**辩论模式的核心原理**：利用「不同视角的相互质疑」逼近正确答案，本质是用多 Agent 实现了 Self-Consistency 的「显式化 + 可交互化」。对开放性、易幻觉的问题（如事实判断、方案评审）提升显著；对有唯一确定解的问题收益有限。

```go
// 辩论模式骨架：两个 Agent 交替质疑，裁判 Agent 收敛结论
func Debate(ctx context.Context, question string, rounds int,
    proposer, critic, judge *Agent) (string, error) {

    answer, _ := proposer.Run(ctx, question)
    for i := 0; i < rounds; i++ {
        // 批判者找漏洞
        critique, _ := critic.Run(ctx,
            fmt.Sprintf("针对问题「%s」，指出以下答案的问题：\n%s", question, answer))
        // 提议者根据批判修正
        answer, _ = proposer.Run(ctx,
            fmt.Sprintf("问题：%s\n你的答案：%s\n收到批评：%s\n请修正", question, answer, critique))
    }
    // 裁判基于最终答案定稿（避免无限辩论）
    return judge.Run(ctx, fmt.Sprintf("问题：%s\n候选答案：%s\n请给出最终定论", question, answer))
}
```

### 9.5 编排者-执行者（Orchestrator-Worker）模式

**最实用、最主流**的多 Agent 拓扑。一个中央 Orchestrator（编排者）负责：接收目标 → 拆解 → 分派给合适的 Worker → 收集结果 → 决定下一步/合成最终答案。Worker 之间**互不通信**，只与 Orchestrator 对话。

**为什么主流**：拓扑简单（星形，非网状），信息流清晰可控，易调试，避免了网状通信的组合爆炸。缺点是 Orchestrator 是单点瓶颈和单点故障。

```go
type Orchestrator struct {
    workers map[string]*Agent // 按能力注册的 worker
    llm     LLMProvider
}

func (o *Orchestrator) Solve(ctx context.Context, goal string) (string, error) {
    plan, err := o.decompose(ctx, goal) // 编排者拆解目标为带 worker 指派的子任务
    if err != nil {
        return "", fmt.Errorf("目标拆解失败: %w", err)
    }
    results := make(map[int]string)
    for _, task := range plan.Steps {
        worker, ok := o.workers[task.AssignedTo]
        if !ok {
            return "", fmt.Errorf("找不到能处理 %s 的 worker", task.AssignedTo)
        }
        // 只把该子任务 + 必要的前置结果给 worker，隔离上下文
        out, err := worker.Run(ctx, o.buildTaskPrompt(task, results))
        if err != nil {
            return "", fmt.Errorf("worker %s 执行失败: %w", task.AssignedTo, err)
        }
        results[task.ID] = out
    }
    return o.synthesize(ctx, goal, results) // 编排者合成最终答案
}
```

### 9.6 群聊（Group Chat）模式

所有 Agent 在一个共享对话空间中，由一个「发言管理器（Speaker Selection）」决定下一个由谁发言。核心难点是**发言权调度**：轮询（简单但低效）、由管理器 LLM 动态选择（智能但增加调用）、基于规则（如按角色顺序）。

与 Orchestrator 模式的本质区别：群聊是**共享上下文**（所有 Agent 看到全部对话），Orchestrator 是**隔离上下文**（Worker 只看到自己的任务）。群聊信息更充分但上下文膨胀更快、成本更高。

### 9.7 层级化 Agent 团队

Orchestrator 模式的递归推广：Worker 本身可以是一个子团队的 Orchestrator，形成树状层级。用于超复杂任务的分层治理（公司级 → 部门级 → 小组级）。每层只关心自己这一层的分解，用分层控制复杂度。

### 9.8 共享上下文与黑板机制

**黑板模式（Blackboard）**：一块所有 Agent 都能读写的共享内存空间。Agent 观察黑板状态，在合适时机贡献自己的部分，逐步逼近解。

**核心原理**：将「谁在什么时候做什么」从硬编码流程中解放出来，改为**由数据状态驱动**——Agent 看到黑板上出现了它能处理的信息就行动。适合解空间不确定、需要多专家机会主义式协作的问题。

```go
// 黑板：并发安全的共享状态空间
type Blackboard struct {
    mu      sync.RWMutex // 保护 entries 的并发读写
    entries map[string]any
    version int64        // 版本号：Agent 据此判断黑板是否有更新
}

func (b *Blackboard) Write(key string, val any) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.entries[key] = val
    b.version++ // 每次写入递增，供订阅者感知变化
}

func (b *Blackboard) Read(key string) (any, bool) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    v, ok := b.entries[key]
    return v, ok
}
```

### 9.9 任务分配与负载均衡

- **静态分配**：按角色/能力固定指派（Orchestrator 的常见做法）。
- **动态分配**：按当前负载、Agent 空闲状态实时分派（类似工作队列 + worker pool）。
- **能力匹配**：维护 Agent 能力注册表，按任务需求匹配最合适的 Agent。

生产实现常用「任务队列 + 空闲 worker 抢占」模型，天然负载均衡。

### 9.10 冲突解决与共识达成

多 Agent 给出矛盾结论时的收敛机制：

- **投票**：多数决（适合离散选项）。
- **权重仲裁**：按 Agent 可信度/专业度加权。
- **裁判 Agent**：专门的 judge 综合各方意见定夺（见辩论模式）。
- **人工介入**：分歧过大或高风险时上升到人（Human-in-the-Loop）。

**核心原理**：多 Agent 系统必须有明确的「决策终结机制」，否则会陷入无限协商。任何协作流程都要预设**收敛条件与兜底裁决者**。

---

## 第 10 章 Agent 通信与协议标准

协议标准解决的是**互操作性（Interoperability）**问题：让不同厂商、不同框架、不同团队开发的 Agent 与工具能够即插即用，而非每对接一个就重写一套胶水代码。

### 10.1 Model Context Protocol（MCP）

**MCP 是什么**：Anthropic 提出的开放协议，用于标准化「LLM 应用如何连接外部数据源和工具」。类比理解：**MCP 之于 AI 应用，如同 USB-C 之于硬件设备**——统一的接口标准，一次实现，处处可用。

**要解决的核心痛点**：在 MCP 之前，每个 Agent 应用要接入一个新工具/数据源（数据库、文件系统、API），都得写专属适配代码，形成 M×N 的集成爆炸（M 个应用 × N 个工具）。MCP 把它降为 M+N：工具方实现一次 MCP Server，应用方实现一次 MCP Client。

**架构三要素**：
- **Host**：运行 LLM 的应用（如 IDE、聊天客户端）。
- **Client**：Host 内部与 Server 一对一连接的连接器。
- **Server**：暴露能力的服务端，提供三类原语。

**三类核心原语（Primitives）——这是 MCP 的精髓**：
- **Tools（工具）**：可被模型调用的函数（有副作用，如写文件、发请求）。由**模型**决定调用。
- **Resources（资源）**：可读取的数据（如文件内容、数据库记录）。由**应用**决定何时加载，类似 GET 请求。
- **Prompts（提示模板）**：预定义的可复用提示模板。由**用户**主动触发（如 slash command）。

区分这三者的控制权归属（模型/应用/用户）是理解 MCP 设计哲学的关键。

**传输层**：基于 JSON-RPC 2.0。本地用 stdio（标准输入输出），远程用 HTTP + SSE（Server-Sent Events）/ Streamable HTTP。

```go
// MCP 基于 JSON-RPC 2.0 的请求结构（简化）
type JSONRPCRequest struct {
    JSONRPC string          `json:"jsonrpc"` // 固定 "2.0"
    ID      int             `json:"id"`
    Method  string          `json:"method"` // 如 "tools/list"、"tools/call"、"resources/read"
    Params  json.RawMessage `json:"params"`
}

// tools/call 的典型交互：Client 请求 Server 执行某工具
type ToolCallParams struct {
    Name      string          `json:"name"`
    Arguments json.RawMessage `json:"arguments"`
}

// Server 端注册工具的能力声明（tools/list 时返回）
type MCPToolDefinition struct {
    Name        string          `json:"name"`
    Description string          `json:"description"`
    InputSchema json.RawMessage `json:"inputSchema"` // JSON Schema 描述参数
}
```

**能力协商（Capability Negotiation）**：连接建立时，Client 和 Server 通过 `initialize` 握手交换各自支持的能力（有哪些 tools/resources/prompts、支持哪些特性），这让协议可扩展、可向后兼容。

### 10.2 Agent-to-Agent（A2A）协议

**A2A 是什么**：Google 主导提出的开放协议，用于**Agent 与 Agent 之间**的直接协作与通信。

**与 MCP 的本质区别（高频考点）**：
- **MCP 解决 Agent ↔ 工具/数据** 的连接（垂直整合，给 Agent 加能力）。
- **A2A 解决 Agent ↔ Agent** 的协作（水平整合，让 Agent 互相委托任务）。
- 二者互补而非竞争：一个 Agent 可以用 MCP 获取工具能力，同时用 A2A 与其他 Agent 协作。

**核心概念**：
- **Agent Card**：一份公开的元数据（通常在 `/.well-known/agent.json`），声明该 Agent 的身份、能力、支持的技能（skills）、认证方式、端点。**这是 A2A 的服务发现基础**——一个 Agent 通过读取另一个 Agent 的 Card 来判断「能不能、该不该把任务委托给它」。
- **Task**：A2A 的核心工作单元。委托方（Client Agent）向被委托方（Remote Agent）发起一个 Task，Task 有明确的生命周期状态（submitted → working → input-required → completed/failed/canceled）。
- **Message / Part**：任务中传递的内容单元，Part 支持多模态（文本、文件、结构化数据）。
- **Artifact**：Task 产出的结果物。

```go
// Agent Card：A2A 的服务发现载体，声明「我是谁、我能干什么」
type AgentCard struct {
    Name         string   `json:"name"`
    Description  string   `json:"description"`
    URL          string   `json:"url"` // A2A 服务端点
    Version      string   `json:"version"`
    Capabilities struct {
        Streaming         bool `json:"streaming"`         // 是否支持流式
        PushNotifications bool `json:"pushNotifications"` // 是否支持异步推送
    } `json:"capabilities"`
    Skills []AgentSkill `json:"skills"` // 具体能提供的技能
}

type AgentSkill struct {
    ID          string   `json:"id"`
    Name        string   `json:"name"`
    Description string   `json:"description"`
    Examples    []string `json:"examples"`
}
```

**关键设计原则**：A2A 强调 Agent 之间是**「不透明」（opaque）协作**——委托方不需要知道被委托方内部用什么模型、什么框架、什么工具，只通过标准接口交互。这保护了各方的实现细节与知识产权，是跨组织 Agent 协作的前提。

**长任务支持**：Agent 任务可能耗时很长（几分钟到几小时），A2A 原生支持：
- **流式（SSE）**：实时推送任务进度。
- **推送通知（Push Notification）**：任务完成后通过 Webhook 回调，无需 Client 长连接轮询。

### 10.3 AG-UI 协议

**AG-UI（Agent-User Interaction Protocol）是什么**：专注于**Agent 与前端用户界面**之间实时交互的开放协议。如果说 MCP 连的是「后端工具」、A2A 连的是「其他 Agent」，那么 AG-UI 连的是「人机交互界面」。

**要解决的核心痛点**：Agent 的执行是多步、流式、有中间状态的（思考中、调用工具中、等待用户确认），传统的「请求-完整响应」HTTP 模型无法优雅地把这些中间过程实时渲染到 UI。AG-UI 定义了一套**标准化的事件流**，让前端能实时、增量地渲染 Agent 的运行过程。

**核心原理——基于事件的流式协议**：Agent 后端向前端发送一系列标准化事件，前端根据事件类型增量更新 UI。典型事件类型：
- **文本流事件**：`TEXT_MESSAGE_START` / `TEXT_MESSAGE_CONTENT` / `TEXT_MESSAGE_END`——逐字渲染回复。
- **工具调用事件**：`TOOL_CALL_START` / `TOOL_CALL_ARGS` / `TOOL_CALL_END`——展示「正在调用 XX 工具」。
- **状态事件**：`STATE_SNAPSHOT` / `STATE_DELTA`——同步 Agent 与 UI 的共享状态（如表单、进度）。
- **生命周期事件**：`RUN_STARTED` / `RUN_FINISHED` / `RUN_ERROR`。

```go
// AG-UI 事件：后端流式推送，前端按 Type 增量渲染
type AGUIEvent struct {
    Type      string          `json:"type"` // 如 TEXT_MESSAGE_CONTENT、TOOL_CALL_START
    MessageID string          `json:"messageId,omitempty"`
    Delta     string          `json:"delta,omitempty"` // 增量文本片段
    Data      json.RawMessage `json:"data,omitempty"`
}

// 通过 SSE 向前端流式推送事件
func streamAgentRun(w http.ResponseWriter, events <-chan AGUIEvent) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    flusher, _ := w.(http.Flusher)
    for ev := range events {
        payload, _ := json.Marshal(ev)
        fmt.Fprintf(w, "data: %s\n\n", payload) // SSE 格式
        flusher.Flush() // 立即推送，实现实时渲染
    }
}
```

**核心价值——双向状态同步**：AG-UI 不只是单向输出，还支持**人在回路（Human-in-the-Loop）**的双向交互：Agent 可以中途请求用户输入/确认（如「是否确认删除？」），UI 把用户响应回传，Agent 继续执行。这让「Agent 主导、人类监督/干预」的交互模式成为可能。

### 10.4 工具与资源的标准化描述

三大协议的共同底层是**用 JSON Schema 描述能力**：无论是 MCP 的 tool inputSchema、A2A 的 skill、还是工具参数，都用 Schema 声明「输入长什么样、输出长什么样」。这让能力可被机器自动发现、校验、调用。**Schema 是 Agent 生态互操作的通用语言**。

### 10.5 跨 Agent 的上下文传递

跨 Agent/跨协议传递上下文的核心挑战与原则：

- **上下文裁剪**：不能把发起方的全部上下文原样传给下游（隐私、成本、噪声），只传任务必需的最小信息。
- **身份与溯源**：传递时要携带发起者身份、任务链路 ID（对应 9.3 的 ConversationID），保证可追溯、可审计。
- **格式统一**：跨协议边界时做上下文格式转换（如 A2A 的 Message ↔ 内部消息格式）。

### 10.6 协议的安全与鉴权

协议开放带来的安全面必须重点防护：

- **认证（Authentication）**：Agent/Client 接入时验证身份。A2A 复用标准 Web 认证（OAuth2、API Key、mTLS），认证方案在 Agent Card 中声明。
- **授权（Authorization）**：验证身份后还要校验权限——这个 Agent/用户能不能调这个工具、访问这份资源（最小权限原则）。
- **传输安全**：远程通信强制 TLS。
- **注入防护**：跨 Agent 传来的内容视为不可信输入，警惕借协议通道传播的 Prompt Injection（见第 15 章）。
- **工具执行隔离**：MCP Server 暴露的工具若涉及命令执行、文件操作，必须沙箱化 + 参数白名单校验，绝不能因为「来自可信协议」就放松校验。

**核心原理**：协议标准化降低了集成成本，但**同时扩大了攻击面**——任何能通过标准协议接入的一方都可能是攻击入口。安全模型必须遵循「零信任」：不因对方走了标准协议就默认可信，每个边界都要独立鉴权与校验。

---

## 小结

第三部分从「单体」走向「协作」，核心原理链条：

- **为何要多 Agent（9.1）**：本质是「关注点分离」在 LLM 上下文限制下的必然产物，用独立精简上下文换取单点可靠性——但要为通信开销、错误累积、一致性难题付出代价。
- **怎么组织（9.5-9.8）**：Orchestrator-Worker 是主流（星形拓扑、上下文隔离、易调试）；群聊共享上下文更充分但更贵；黑板模式由数据状态驱动协作。
- **怎么收敛（9.10）**：多 Agent 必须有明确的决策终结机制与兜底裁决者。
- **怎么互操作（第 10 章）**：三大协议各司其职——**MCP 连工具/数据（垂直加能力）、A2A 连 Agent（水平促协作）、AG-UI 连用户界面（实时人机交互）**，共同底层是 JSON Schema 描述能力 + 零信任安全模型。

理解这三大协议的**控制权归属**（MCP 原语分属模型/应用/用户）与**连接对象差异**（工具 / Agent / UI），是把握整个 Agent 互操作生态的钥匙。
