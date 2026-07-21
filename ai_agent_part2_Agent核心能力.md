# 第二部分 Agent 核心能力 · 详解

> 本文是《AI Agent 开发学习路线》第二部分（第 4-8 章）的逐条释义。代码以 Go 为主，聚焦生产级实践与硬核细节。

---

## 第 4 章 Agent 架构范式

架构范式决定了「LLM 的推理循环如何组织」。第一部分的 ReAct 循环是最小内核，本章的所有范式都是它的增强变体。

### 4.1 ReAct 架构

**Thought → Action → Observation** 交替循环，边推理边行动，用工具结果修正下一步决策。

- **优点**：简单、通用、可解释（每步 Thought 可审计）。
- **缺点**：无全局规划，长任务容易「走一步看一步」跑偏；每步都要完整推理，token 消耗大。
- **适用**：工具数量适中、步数可控（<10 步）的任务。

工程要点：**必须设步数上限 + 单步超时 + 循环检测**（模型反复调同一工具同一参数说明卡死）。

```go
type stepGuard struct {
    maxSteps    int
    seen        map[string]int // 工具调用指纹 -> 出现次数
    repeatLimit int
}

func (g *stepGuard) checkRepeat(call ToolCall) error {
    key := call.Name + "|" + call.Arguments
    g.seen[key]++
    if g.seen[key] > g.repeatLimit {
        return fmt.Errorf("检测到重复调用 %s，疑似陷入循环", call.Name)
    }
    return nil
}
```

### 4.2 Plan-and-Execute 架构

先由「规划器」一次性产出完整计划（步骤列表），再由「执行器」逐步执行。

- **对比 ReAct**：ReAct 是「边想边做」，Plan-and-Execute 是「先想清楚再做」。
- **优点**：全局视角、步骤明确、执行器可用更便宜的模型、可并行执行无依赖步骤。
- **缺点**：计划僵化，现实偏离计划时需要**重规划（Replan）**。

```go
type Plan struct {
    Steps []PlanStep `json:"steps"`
}
type PlanStep struct {
    ID        int    `json:"id"`
    Action    string `json:"action"`     // 描述要做什么
    DependsOn []int  `json:"depends_on"` // 依赖的前置步骤，用于判断能否并行
}

func (a *Agent) PlanAndExecute(ctx context.Context, goal string) (string, error) {
    plan, err := a.planner.MakePlan(ctx, goal) // 强模型规划
    if err != nil {
        return "", fmt.Errorf("规划失败: %w", err)
    }
    results := make(map[int]string)
    for _, step := range plan.Steps {
        out, err := a.executor.Do(ctx, step, results) // 弱模型/工具执行
        if err != nil {
            // 执行偏离预期 -> 触发重规划
            plan, err = a.planner.Replan(ctx, goal, plan, step, err)
            if err != nil {
                return "", err
            }
            continue
        }
        results[step.ID] = out
    }
    return a.summarize(ctx, goal, results)
}
```

### 4.3 Reflexion（自我反思）架构

Agent 执行后自我评价结果，若不满意则总结失败原因（写入记忆），带着教训重试。用「反思」把一次性执行变成可迭代改进的闭环。

- **核心**：`执行 → 评估 → 反思 → 重试`，反思结果作为下一轮的额外上下文。
- **适用**：有明确成败判据的任务（代码能否通过测试、答案是否被校验器接受）。

```go
func (a *Agent) RunWithReflection(ctx context.Context, task string) (string, error) {
    var reflections []string
    const maxAttempts = 3
    for attempt := 0; attempt < maxAttempts; attempt++ {
        result, _ := a.attempt(ctx, task, reflections)
        if ok, _ := a.evaluate(ctx, task, result); ok {
            return result, nil
        }
        // 失败：让模型反思「为什么失败、下次怎么改」
        r := a.reflect(ctx, task, result)
        reflections = append(reflections, r)
    }
    return "", fmt.Errorf("经 %d 次尝试仍未通过评估", maxAttempts)
}
```

### 4.4 Tree of Thoughts（思维树）

把推理建模为树：每个节点是一个中间思考，向下扩展多个候选分支，通过评估+搜索（BFS/DFS）探索多条路径并回溯。适合需要探索、可回溯的难题（数学证明、复杂规划）。代价是调用量成倍增长。

### 4.5 Graph of Thoughts（思维图）

ToT 的推广：思考节点组成有向图，允许分支聚合、复用中间结果。表达力更强，实现复杂度也更高，工业界应用较少。

### 4.6 Reasoning + Acting 循环设计

设计一个健壮的循环需要收敛控制五要素：

- **终止条件**：模型给出 Final Answer / 无工具调用。
- **步数上限**：硬性防死循环。
- **单步超时**：`context.WithTimeout` 兜底。
- **循环检测**：识别重复调用。
- **成本熔断**：累计 token/费用超预算即停。

```go
type LoopController struct {
    maxSteps, step int
    tokenBudget    int
    tokenUsed      int
}

func (c *LoopController) shouldContinue() error {
    if c.step >= c.maxSteps {
        return errors.New("达到最大步数")
    }
    if c.tokenUsed >= c.tokenBudget {
        return errors.New("超出 token 预算，熔断")
    }
    return nil
}
```

### 4.7 Router / Orchestrator 模式

前置一个轻量「路由器」先判断意图，再分发到对应的专用 Agent/工作流。用便宜模型做分类，把复杂推理留给下游，是**降本增效的关键设计**。

```go
type Intent string
const (
    IntentQuery  Intent = "query"
    IntentCreate Intent = "create"
    IntentChat   Intent = "chat"
)

func (r *Router) Route(ctx context.Context, input string) (Intent, error) {
    // 用小模型 + 强格式约束做意图分类，temperature=0
    resp, err := r.cheapLLM.Classify(ctx, input, []Intent{IntentQuery, IntentCreate, IntentChat})
    if err != nil {
        return IntentChat, err // 兜底走通用对话
    }
    return resp, nil
}
```

### 4.8 事件驱动 Agent 架构

Agent 不再是同步请求-响应，而是订阅事件（消息队列、Webhook、定时器）异步触发。适合长时运行、需与外部系统解耦的场景（如监控告警自动处置）。要点：幂等处理、状态持久化、失败重试。

### 4.9 状态机与工作流驱动的 Agent

对确定性强、合规要求高的流程，用显式状态机约束 Agent 的行动空间——每个状态只允许特定工具/转移，把 LLM 的「自由发挥」关进笼子。这是本项目 `fsm` 模块的思路：**LLM 负责状态内的判断，状态机负责流程的可控**。

**架构选型速查**：任务简单→ReAct；步骤清晰可拆→Plan-and-Execute；有明确成败判据→Reflexion；流程强合规→状态机；多意图入口→Router。

---

## 第 5 章 工具调用（Tool / Function Calling）

工具调用是 Agent 从「只会说」到「能做事」的分水岭。

### 5.1 Function Calling 原理与协议

模型并不真正执行函数，而是**输出一个结构化的调用意图**（函数名 + 参数 JSON），由你的代码执行后把结果回喂。整个链路：

```
你提供工具定义（JSON Schema） → 模型输出 tool_call → 你执行 → 结果作为 role=tool 消息回填 → 模型继续
```

### 5.2 工具定义与 JSON Schema

工具的 Schema 是模型「读说明书」的唯一依据，描述质量直接决定调用准确率。

```go
type Tool struct {
    Type     string       `json:"type"` // "function"
    Function ToolFunction `json:"function"`
}
type ToolFunction struct {
    Name        string          `json:"name"`
    Description string          `json:"description"` // 极其重要：写清何时用、用来做什么
    Parameters  json.RawMessage `json:"parameters"`  // JSON Schema
}

var queryTicketsTool = Tool{
    Type: "function",
    Function: ToolFunction{
        Name:        "query_tickets",
        Description: "根据地区、紧急程度和日期查询工单列表。当用户询问工单数量或状态时使用。",
        Parameters: json.RawMessage(`{
            "type": "object",
            "properties": {
                "region": {"type": "string", "description": "地区名，如 华南、华北"},
                "level":  {"type": "string", "enum": ["普通","紧急"], "description": "紧急程度"},
                "date":   {"type": "string", "description": "查询日期，ISO8601 格式 YYYY-MM-DD"}
            },
            "required": ["region"]
        }`),
    },
}
```

### 5.3 工具选择与参数抽取

模型根据用户意图 + 工具描述选工具、从自然语言抽参数。常见坑：

- **参数幻觉**：模型编造用户没提供的参数值 → 用 `required` 约束 + 校验缺失时反问。
- **枚举越界**：不在 `enum` 内的值 → Schema 用 `enum` 硬约束。
- **日期/相对时间**：「昨天」需转成绝对日期 → 在工具层做归一化，别指望模型算准。

### 5.4 并行工具调用与串行工具调用

现代模型可在一轮里返回多个 tool_call。无依赖的应**并行执行**降延迟；有依赖的必须串行。

```go
func (a *Agent) executeToolCalls(ctx context.Context, calls []ToolCall) []Message {
    results := make([]Message, len(calls))
    var wg sync.WaitGroup
    for i, call := range calls {
        wg.Add(1)
        go func(i int, call ToolCall) { // 并行执行独立工具
            defer wg.Done()
            out := a.executeTool(ctx, call)
            results[i] = Message{Role: "tool", ToolCallID: call.ID, Content: out}
        }(i, call)
    }
    wg.Wait() // 生产中建议用 errgroup + context 控制超时与取消
    return results
}
```

> 注意：Go 中并行执行需保证每个工具本身并发安全，且用 `context` 统一控制超时。项目里常用 `trpc.GoAndWait` 做并发编排。

### 5.5 工具调用结果的解析与回填

工具结果必须以 `role: "tool"` 且带 `tool_call_id` 回填，模型才能对应上是哪次调用的结果。结果过大时要先摘要/截断，否则撑爆上下文。

### 5.6 工具执行的错误处理与重试

**关键理念：工具报错不要直接中断 Agent，而要把错误信息作为 Observation 回喂给模型，让它自我修正**（如改参数重试、换工具、或如实告知用户）。

```go
func (a *Agent) executeTool(ctx context.Context, call ToolCall) string {
    fn, ok := a.registry[call.Name]
    if !ok {
        return fmt.Sprintf("错误：工具 %s 不存在，请从可用工具中选择", call.Name)
    }
    out, err := fn.Invoke(ctx, call.Arguments)
    if err != nil {
        // 把错误转成模型可理解的自然语言，而非直接 return err
        return fmt.Sprintf("工具执行失败：%v。请检查参数或尝试其他方式", err)
    }
    return out
}
```

### 5.7 工具权限控制与安全沙箱

- **最小权限**：每个 Agent 只注册它需要的工具。
- **高危操作审批**：删除、转账、外发类工具走人工确认（Human-in-the-Loop）。
- **沙箱隔离**：代码执行类工具放容器/受限进程，禁网络、限资源。
- **参数二次校验**：绝不信任模型给的参数，执行前按业务规则校验（如 SQL 白名单、路径限制）。

### 5.8 自定义工具的封装规范

统一接口 + 注册表，让工具即插即用：

```go
type ToolHandler interface {
    Name() string
    Schema() json.RawMessage
    Invoke(ctx context.Context, args string) (string, error)
}

type Registry struct {
    tools map[string]ToolHandler
}

func (r *Registry) Register(h ToolHandler) {
    r.tools[h.Name()] = h // 启动时集中注册，便于按 Agent 组装不同工具集
}
```

### 5.9 工具描述编写最佳实践

- 描述写清**「什么时候该用」**，而不仅是「这是什么」。
- 参数逐个写 `description`，标明格式、单位、示例。
- 用 `enum` 收敛取值，用 `required` 明确必填。
- 工具数量控制在合理范围（过多会降低选择准确率），太多时用工具路由/分组。

### 5.10 动态工具加载与工具路由

工具很多时，先用检索/分类挑出与当前任务相关的一小撮工具再喂给模型，既省 token 又提升选择准确率——本质是「工具版的 RAG」。

---

## 第 6 章 规划与推理（Planning & Reasoning）

### 6.1 任务分解（Task Decomposition）

把大目标拆成可执行子任务，是复杂 Agent 的第一步。拆解粒度要适中：太粗执行器搞不定，太细规划开销大。

### 6.2 子任务依赖管理

用 DAG 表达依赖，拓扑排序确定执行顺序，无依赖的可并行。

```go
// 基于依赖做拓扑排序，返回可执行批次（同批次内可并行）
func topoBatches(steps []PlanStep) [][]PlanStep {
    indeg := map[int]int{}
    graph := map[int][]int{}
    byID := map[int]PlanStep{}
    for _, s := range steps {
        byID[s.ID] = s
        indeg[s.ID] += len(s.DependsOn)
        for _, dep := range s.DependsOn {
            graph[dep] = append(graph[dep], s.ID)
        }
    }
    var batches [][]PlanStep
    ready := collectZeroIndeg(indeg)
    for len(ready) > 0 {
        var batch []PlanStep
        var next []int
        for _, id := range ready {
            batch = append(batch, byID[id])
            for _, nb := range graph[id] {
                if indeg[nb]--; indeg[nb] == 0 {
                    next = append(next, nb)
                }
            }
        }
        batches = append(batches, batch)
        ready = next
    }
    return batches
}
```

### 6.3 层次化规划（Hierarchical Planning）

高层产出粗粒度里程碑，每个里程碑再细化为子计划。用分层控制复杂度，避免一次性规划过深导致模型「想不清」。

### 6.4 动态重规划（Replanning）

现实几乎必然偏离初始计划（工具失败、数据不符预期）。触发重规划的信号：步骤失败、结果与假设矛盾、出现新信息。重规划时把「已完成的、失败原因」一并给模型，避免重蹈覆辙。

### 6.5 目标导向的推理

始终让模型「盯着最终目标」判断当前步是否有助于达成，防止在细节里迷失。实践上可在每轮上下文里复述目标（goal reminder）。

### 6.6 反思与自我修正机制

见 4.3 Reflexion。关键是**可量化的评估信号**：能自动判成败的任务，反思才有效。

### 6.7 推理路径的评估与剪枝

多路径探索（ToT/Self-Consistency）时，用启发式或 LLM 打分给路径评分，及时砍掉低分分支控制成本。

### 6.8 长程任务的规划策略

- **分段 + 检查点**：把长任务切段，每段完成后持久化状态，支持断点续跑。
- **上下文管理**：长任务上下文必然溢出，需滚动摘要（见第 7 章）。
- **外部化记忆**：把中间产物写入外部存储（文件/DB），而非全塞进上下文。

### 6.9 规划失败的兜底策略

- 重规划次数上限，超限则降级为「向用户澄清」。
- 部分完成也返回已有成果 + 明确说明未完成项。
- 绝不「假装成功」——宁可如实报告失败。

---

## 第 7 章 记忆系统（Memory）

LLM 无状态，记忆系统赋予 Agent「连续性」和「个性化」。

### 7.1 短期记忆与长期记忆

- **短期记忆**：当前会话上下文（消息历史），随会话结束消失，受上下文窗口限制。
- **长期记忆**：跨会话持久化（用户偏好、历史事实），存于 DB/向量库，按需检索。

### 7.2 上下文窗口管理与压缩

上下文是最稀缺资源。三种压缩策略：

- **截断**：保留最近 N 条 + System（最简单，会丢信息）。
- **摘要**：把旧对话滚动压缩成摘要（保信息，费一次 LLM 调用）。
- **选择性检索**：只把与当前问题相关的历史片段召回（结合向量检索）。

```go
type MemoryManager struct {
    maxTokens int
    llm       LLMProvider
}

// 滑动窗口 + 滚动摘要：超预算时把最旧的一批消息压成摘要
func (m *MemoryManager) Compact(ctx context.Context, msgs []Message) ([]Message, error) {
    if estimateTokensMsgs(msgs) <= m.maxTokens {
        return msgs, nil
    }
    system, rest := splitSystem(msgs)
    keepRecent := rest[len(rest)-6:] // 保留最近 6 条原文
    older := rest[:len(rest)-6]
    summary, err := m.llm.Summarize(ctx, older) // 旧对话压缩成一条摘要
    if err != nil {
        return nil, fmt.Errorf("历史摘要失败: %w", err)
    }
    return append(append(system,
        Message{Role: "system", Content: "早期对话摘要：" + summary}),
        keepRecent...), nil
}
```

### 7.3 对话历史的存储与检索

按 `session_id` 持久化消息，支持会话恢复。高频读写建议 Redis 缓存 + DB 落盘。

### 7.4 情节记忆（Episodic Memory）

记录「发生过的具体事件」（某次交互、某次任务的过程与结果），供后续「回忆」相似情境。常以向量化存储，按相似度召回。

### 7.5 语义记忆（Semantic Memory）

存储抽象的事实与知识（用户是 VIP、公司政策），与具体事件无关。可用键值/图结构存储。

### 7.6 记忆的写入、更新与遗忘策略

- **写入**：不是所有对话都值得记，用 LLM 判断「这条是否含值得长期记住的信息」。
- **更新**：新信息与旧记忆冲突时（用户改了偏好），要更新而非追加。
- **遗忘**：TTL 过期、重要性衰减、容量淘汰，防止记忆无限膨胀。

### 7.7 记忆摘要与蒸馏

定期把碎片化记忆归纳成更高层的画像（把 10 次「点了拿铁」蒸馏成「偏好拿铁」）。

### 7.8 记忆与向量数据库的结合

长期记忆本质上是一个 “写入 -> 存储 -> 检索 -> 注入” 的闭环。

长期记忆的主流实现：记忆条目 embedding 后存向量库，交互时用当前语境检索最相关的 k 条注入上下文。这与 RAG 技术栈完全共用。

```go
type MemoryStore interface {
    Save(ctx context.Context, userID string, mem MemoryItem) error
    Recall(ctx context.Context, userID, query string, topK int) ([]MemoryItem, error)
}

// 交互前召回相关记忆，注入 System Prompt
func (a *Agent) withMemory(ctx context.Context, userID, input string) ([]Message, error) {
    mems, err := a.memory.Recall(ctx, userID, input, 3)
    if err != nil {
        return nil, err
    }
    var sb strings.Builder
    for _, m := range mems {
        sb.WriteString("- " + m.Content + "\n")
    }
    return []Message{
        {Role: "system", Content: "关于该用户的已知信息：\n" + sb.String()},
        {Role: "user", Content: input},
    }, nil
}
```

设计这类系统，需要留意以下三个工程难点：

1. 记忆的“写入”不是存原文，而是存“反思”：
    直接存原文（“今天心情不好”）太琐碎且无用。成熟的做法是：定时让一个 “记忆总结模型” 去压缩对话，比如将 100 条琐碎聊天提炼成 1 条：“用户是一位注重隐私、偏好极简风格的 Java 后端开发者”。存这条“反思”比存原始日志价值大得多。

2. K 值的黄金法则：
    Top-K 值（取几条）很关键。取 1 条太武断，取 10 条可能把上下文撑满（浪费昂贵 Token）。通常建议 K 值取 3 ~ 5 之间，并在 Prompt 中明确区分“记忆”和“当前问题”。

3. 时序衰减（Recency Bias）：
    检索不能只看语义，还要结合时间。如果检索出的 3 条记忆全是 2 年前的，而忽略 1 周前的，就会出偏差。成熟的向量检索往往会加入 时间戳（timestamp） 作为过滤条件，或者在使用 Re-rank（重排序）模型时加权时效性。

总结一句话：如果把大模型比作 CPU，向量库就是它的“外置 RAM”。你部署的 Agent 服务在启动时挂载一个 Chroma 或 Milvus 实例，然后用你熟悉的 Go 代码在中间件层（Interceptor）里调用它——这样，你的应用就拥有了真正的、长久的“记忆大脑”。

### 7.9 用户画像与个性化记忆

聚合长期记忆形成结构化画像（偏好、历史、权限），驱动个性化响应。注意隐私合规：敏感信息脱敏、可删除、最小化收集。

---

## 第 8 章 检索增强生成（RAG）

RAG 是解决「知识截止 + 幻觉 + 私有数据」的核心手段，也是企业级 Agent 落地最高频的技术。

### 8.1 RAG 的基本原理与流程

```
离线：文档 → 解析 → 分块 → embedding → 存入向量库
在线：用户问题 → embedding → 检索相关块 → 拼进 Prompt → LLM 基于检索内容作答
```

一句话：**先查资料再答题**，让模型的回答有据可依。

### 8.2 文档加载与解析

不同格式需不同解析器：PDF（注意版面/表格/扫描件 OCR）、HTML（去标签噪声）、Markdown（保留标题层级）、代码（按函数/类切分保结构）。**解析质量是 RAG 的第一道天花板**——垃圾进，垃圾出。

### 8.3 文本分块（Chunking）策略

- **固定长度**：按 token 切，简单但可能切断语义。
- **按结构**：按段落/标题/代码块切，保语义完整。
- **重叠切分（overlap）**：相邻块重叠一部分，避免边界信息丢失。
- **块大小权衡**：太小丢上下文，太大稀释相关性 + 费 token。

```go
// 带重叠的分块：overlap 保证跨块信息不丢
func chunkText(text string, chunkSize, overlap int) []string {
    runes := []rune(text)
    var chunks []string
    for start := 0; start < len(runes); start += chunkSize - overlap {
        end := start + chunkSize
        if end > len(runes) {
            end = len(runes)
        }
        chunks = append(chunks, string(runes[start:end]))
        if end == len(runes) {
            break
        }
    }
    return chunks
}
```

### 8.4 Embedding 模型与向量化

Embedding 把文本映射为语义向量，语义相近的向量距离近。选型看：维度、语种支持、领域适配、成本。**查询与文档必须用同一个 embedding 模型**，否则向量空间不一致，检索全乱。

#### 8.4.1 什么是“向量化”？——文字的“坐标系转换”

- 核心逻辑：Embedding 模型本质上是一个“编码器”。它把一段人类语言（如“苹果很好吃”）读进去，吐出一个几百或几千维的浮点数数组，例如：[0.12, -0.98, 0.45, ... , 0.77]。

- 几何意义：这个数组代表了该文本在高维空间里的一个“坐标点”。

    “苹果很好吃”和“香蕉不错”这两个文本生成的坐标会离得很近（距离小）。

    “苹果很好吃”和“坦克很重”这两个文本的坐标会离得非常远（距离大）。

- 总结：向量检索的本质，就是在高维空间里“查坐标”——把你输入的“问题坐标”拿去数据库里找离它最近的“文档坐标”。

#### 8.4.2 选型四要素：维度、语种、领域、成本

选错模型是导致召回效果极差的头号元凶。选型时需要权衡这四个维度：

1. 维度（Dimension）

    是什么：指向量数组的长度，常见的有 384、768、1536、甚至 4096。

    权衡：维度越高，理论上能“浓缩”的语义信息就越丰富、越精准；但维度越高，占用存储空间越大，检索时的计算速度越慢。

    建议：如果你的场景是简单分类（如情感正负），小维度够用；如果是复杂长文档匹配，必须上高维度大模型。

2. 语种支持（Language Support）

    是什么：模型训练时用了哪国的语料。

    陷阱：如果你用英文专用模型（如早期的 text-embedding-ada-002）去处理纯中文用户问题，模型会把中文当作“乱码”向量化，导致“苹果”和“橘子”在英文语料映射下全是近义词，检索稀碎。

    建议：中文场景首选 BGE（BAAI）、M3E 或 通义千问的 Qwen-Embedding；国际通用场景选 Multilingual-E5。

3. 领域适配（Domain Fit）

    是什么：通用模型学的是“互联网百科”的知识分布。

    痛点：如果你的资料库里全是“医疗CT影像诊断报告”或“金融研报里的专业黑话”，通用模型会将这些专业词向量化得极其模糊，因为训练它时没见过这些词。

    建议：如果找不到对应的垂直领域专业 Embedding 模型，可以考虑用微调（Fine-tune）通用模型，或者干脆用大模型（如 GPT-4）生成领域专用伪数据来训练。

4. 成本（Cost）

    包含显性的 API 调用费（如 OpenAI 按每 1M token 计费），和隐性的硬件资源费（如果自建开源模型，需要占用 GPU 显存）。为了省钱把维度从 1536 降到 384，如果导致召回下降，不如不降。

#### 8.4.3 致命红线：查询与文档必须用同一个模型

因为不同公司（甚至同一公司的不同版本）训练出的 Embedding 模型构建的“高维坐标系”是完全不同的。

假设文档库用的是 模型 A（坐标系原点在左下角，X轴代表“动物性”）。

用户查询时换了 模型 B（坐标系原点在右上角，X轴代表“可食用性”）。

虽然你把“苹果”都传进去了，但在模型 A 的空间里，苹果 坐标为 (10, 2)；在模型 B 的空间里，苹果 坐标为 (-3, 9)。

当你用模型 B 生成的“查询向量”去模型 A 构建的“数据库索引”里找邻居时，相当于“用北京的地铁图去查上海的公交线路”。检索出的结果永远是 风马牛不相及 的乱码文档，因为向量空间的物理排列规则已经崩了。

#### 8.4.4 工程落地建议（Go 技术栈实操）

将下面的规则写进项目 README 的显眼位置：

1. 固化模型 ID：在 config.yaml 里直接锁死 Embedding 模型的名称和版本（如 BAAI/bge-m3）。只要数据库里存了向量，绝对不能随意更换这个配置。

2. 离线与在线对齐：写数据（存入向量库）时用的模型，必须和读数据（检索向量库）时用的模型是完全相同的同一行代码调用。建议将其封装成一个单例（Singleton）的 Go 包。

3. 版本迁移策略：如果实在要升级 Embedding 模型（比如从 V1 升到 V2），必须双写或全量重算——把所有旧向量删掉，用新模型把所有文档重新 Embedding 一遍再存进去。绝对不能新旧混用。

总结一句话：Embedding 模型决定了你业务认知世界的“世界观”。选模型时参考四要素，定下来后绝不变换，否则整个智能检索系统将瞬间报废。

### 8.5 向量数据库

FAISS（库，自托管）、Milvus（分布式）、Pinecone（托管云）、Chroma（轻量）、pgvector（PostgreSQL 扩展，适合已有 PG 的团队）。选型看：数据规模、是否需要过滤、运维能力、是否要和现有 DB 融合。

### 8.6 相似度检索与混合检索

- **向量检索**：擅长语义相似（「汽车」匹配「轿车」），但对精确关键词（型号、编号）弱。
- **关键词检索（BM25）**：擅长精确匹配，不懂语义。
- **混合检索**：两者结果融合（如 RRF 算法），互补，是生产标配。

```go
// RRF（Reciprocal Rank Fusion）融合两路召回结果
func fuseRRF(vectorHits, keywordHits []string, k int) []string {
    const rrfK = 60.0
    score := map[string]float64{}
    for rank, id := range vectorHits {
        score[id] += 1.0 / (rrfK + float64(rank+1))
    }
    for rank, id := range keywordHits {
        score[id] += 1.0 / (rrfK + float64(rank+1))
    }
    return topKByScore(score, k)
}
```

### 8.7 重排序（Rerank）

初步召回（快、粗）后，用更精细的 Cross-Encoder 重排模型对候选精排，把最相关的排到前面。「先粗召回多一些，再精排取前几」是提升 RAG 质量的高性价比手段。

### 8.8 查询改写与查询扩展

- **改写**：把口语化/有指代的问题改写成适合检索的完整查询（多轮对话尤其需要）。
- **扩展**：生成多个变体查询/子查询分别检索再合并（如 HyDE、Multi-Query）。

### 8.9 多路召回与融合

从多个数据源/多种策略并行召回，融合去重。适合知识分散在多个库的场景。

### 8.10 RAG 的评估指标

- **召回率/命中率**：相关文档是否被检索到（检索质量）。
- **准确率/精确率**：检索结果中相关的比例。
- **忠实度（Faithfulness）**：回答是否忠于检索内容、有没有编造（生成质量）。
- **答案相关性**：回答是否切题。
- 工具：RAGAS 等。**没有评估就没有优化方向**。

### 8.11 Agentic RAG 与自适应检索

传统 RAG 是「固定检索一次」；Agentic RAG 让 Agent 自主决定：**是否要检索、检索几次、改写查询、评估检索够不够、不够再检**。把检索变成 Agent 循环里的一个工具，更智能也更贵。

```go
// 把检索封装成工具，交给 Agent 按需自主调用（Agentic RAG 的核心思路）
var retrieveTool = Tool{
    Type: "function",
    Function: ToolFunction{
        Name: "retrieve_knowledge",
        Description: "当需要查询产品文档、政策或历史工单等外部知识时调用。" +
            "如果一次检索结果不足以回答，可改写查询再次调用。",
        Parameters: json.RawMessage(`{
            "type":"object",
            "properties":{"query":{"type":"string","description":"检索关键词或问题"}},
            "required":["query"]
        }`),
    },
}
```

### 8.12 GraphRAG 与知识图谱增强

把知识组织成实体-关系图，检索时利用图结构做多跳推理（「A 的上级的负责项目」）。擅长关系型、全局性问题，构建成本高。适合知识关联紧密、需要跨文档归纳的场景。

---

## 小结

第二部分是 Agent 的「五脏六腑」：

- **第 4 章 架构范式**：组织推理循环的骨架——ReAct/Plan-Execute/Reflexion 各有适用面。
- **第 5 章 工具调用**：让 Agent 能行动的「手」，核心是 Schema 设计 + 错误回喂 + 安全管控。
- **第 6 章 规划推理**：处理复杂任务的「大脑前额叶」——分解、依赖、重规划。
- **第 7 章 记忆系统**：跨越 LLM 无状态缺陷的「海马体」——短期压缩 + 长期向量召回。
- **第 8 章 RAG**：对抗幻觉与知识截止的「外接知识库」，企业落地第一刚需。

贯穿全章的工程主线：**收敛控制（步数/超时/预算/循环检测）、错误回喂而非中断、安全最小权限、上下文即成本**。这些原则将在第四、五部分的工程化与安全章节被进一步系统化。
