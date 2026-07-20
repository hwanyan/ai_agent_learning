# AI Agent 开发学习路线

> 一份面向从入门到进阶的 AI Agent 开发系统化学习目录。本文件仅列出目录与知识点，不含具体释义。

---

## 第一部分 基础理论与背景

### 第 1 章 AI Agent 概述
- Agent 的定义与核心特征（自主性、反应性、主动性、社会性）
- Agent 与传统程序 / 工作流 / Chatbot 的区别
- Agent 与 LLM 的关系
- Agent 的发展历史（符号主义 Agent、强化学习 Agent、LLM Agent）
- 典型应用场景（编码助手、客服、数据分析、自动化办公、科研助手）
- Agent 的能力边界与局限性
- 单 Agent 与多 Agent 的概念区分

### 第 2 章 大语言模型基础
- Transformer 架构核心原理
- 预训练、微调、对齐（RLHF / DPO）
- Token、Tokenizer 与上下文窗口
- 采样参数（temperature、top-p、top-k、frequency penalty）
- 涌现能力与规模效应
- 指令遵循与对话能力
- 模型的知识截止与幻觉问题
- 开源模型与闭源模型生态（GPT、Claude、Gemini、Llama、Qwen、DeepSeek 等）
- 模型选型的评估维度（能力、成本、延迟、上下文长度、工具调用能力）

### 第 3 章 Prompt Engineering 基础
- Prompt 的基本结构（System / User / Assistant）
- Zero-shot 与 Few-shot 提示
- Chain-of-Thought（思维链）提示
- Self-Consistency（自洽性）
- ReAct（推理 + 行动）范式
- Prompt 模板与变量注入
- 角色扮演与人设设定
- 输出格式约束（JSON、XML、Markdown）
- Prompt 调试与迭代方法
- 提示词攻击与防御（Prompt Injection、越狱）

---

## 第二部分 Agent 核心能力

### 第 4 章 Agent 架构范式
- ReAct 架构
- Plan-and-Execute 架构
- Reflexion（自我反思）架构
- Tree of Thoughts（思维树）
- Graph of Thoughts（思维图）
- Reasoning + Acting 循环设计
- Router / Orchestrator 模式
- 事件驱动 Agent 架构
- 状态机与工作流驱动的 Agent

### 第 5 章 工具调用（Tool / Function Calling）
- Function Calling 原理与协议
- 工具定义与 JSON Schema
- 工具选择与参数抽取
- 并行工具调用与串行工具调用
- 工具调用结果的解析与回填
- 工具执行的错误处理与重试
- 工具权限控制与安全沙箱
- 自定义工具的封装规范
- 工具描述编写最佳实践
- 动态工具加载与工具路由

### 第 6 章 规划与推理（Planning & Reasoning）
- 任务分解（Task Decomposition）
- 子任务依赖管理
- 层次化规划（Hierarchical Planning）
- 动态重规划（Replanning）
- 目标导向的推理
- 反思与自我修正机制
- 推理路径的评估与剪枝
- 长程任务的规划策略
- 规划失败的兜底策略

### 第 7 章 记忆系统（Memory）
- 短期记忆与长期记忆
- 上下文窗口管理与压缩
- 对话历史的存储与检索
- 情节记忆（Episodic Memory）
- 语义记忆（Semantic Memory）
- 记忆的写入、更新与遗忘策略
- 记忆摘要与蒸馏
- 记忆与向量数据库的结合
- 用户画像与个性化记忆

### 第 8 章 检索增强生成（RAG）
- RAG 的基本原理与流程
- 文档加载与解析（PDF、HTML、Markdown、代码）
- 文本分块（Chunking）策略
- Embedding 模型与向量化
- 向量数据库（FAISS、Milvus、Pinecone、Chroma、pgvector）
- 相似度检索与混合检索（关键词 + 向量）
- 重排序（Rerank）
- 查询改写与查询扩展
- 多路召回与融合
- RAG 的评估指标（召回率、准确率、忠实度）
- Agentic RAG 与自适应检索
- GraphRAG 与知识图谱增强

---

## 第三部分 多 Agent 与协作

### 第 9 章 多 Agent 系统
- 多 Agent 系统的动机与优势
- Agent 角色分工与职责划分
- Agent 间通信协议与消息传递
- 协作模式（协作、竞争、辩论）
- 编排者-执行者（Orchestrator-Worker）模式
- 群聊（Group Chat）模式
- 层级化 Agent 团队
- 共享上下文与黑板机制
- 任务分配与负载均衡
- 冲突解决与共识达成

### 第 10 章 Agent 通信与协议标准
- Model Context Protocol（MCP）
- Agent-to-Agent（A2A）协议
- AG-UI 协议
- 工具与资源的标准化描述
- 跨 Agent 的上下文传递
- 协议的安全与鉴权

---

## 第四部分 工程化与框架

### 第 11 章 主流 Agent 开发框架
- LangChain 核心组件与用法
- LangGraph 图式编排
- LlamaIndex 数据框架
- AutoGen 多 Agent 框架
- CrewAI 角色协作框架
- Semantic Kernel
- Dify / Coze 等低代码平台
- OpenAI Agents SDK / Assistants API
- 框架选型对比与取舍

### 第 12 章 Agent 工程化实践
- Agent 项目结构设计
- 配置管理与环境隔离
- Prompt 版本管理
- 工具与能力的模块化组织
- 依赖注入与可测试性设计
- 异步与并发处理
- 流式输出（Streaming）实现
- 状态持久化与断点续跑
- 成本控制与 Token 优化
- 缓存策略（语义缓存、结果缓存）

### 第 13 章 可观测性与调试
- Agent 执行轨迹（Trace）追踪
- 日志设计与结构化日志
- LangSmith / LangFuse 等观测平台
- 指标监控（延迟、Token、成功率、成本）
- 中间步骤的可视化
- 错误定位与回放
- A/B 测试与灰度发布

### 第 14 章 评估与测试
- Agent 评估的挑战
- 端到端评估与分步评估
- 基准数据集与测试集构建
- LLM-as-a-Judge 评估法
- 人工评估与标注
- 任务完成率、准确率、鲁棒性指标
- 回归测试与持续评估
- 对抗性测试
- 评估工具与平台

---

## 第五部分 安全、部署与优化

### 第 15 章 Agent 安全与合规
- Prompt Injection 攻击与防御
- 越权与权限最小化原则
- 工具调用的沙箱隔离
- 敏感数据脱敏与保护
- 输出内容审核与过滤
- 幻觉检测与事实核查
- 人在回路（Human-in-the-Loop）审批
- 数据隐私与合规（GDPR、数据出境）
- 供应链安全与依赖审计

### 第 16 章 部署与运维
- Agent 服务化与 API 封装
- 容器化与云原生部署
- 弹性伸缩与高可用
- 模型网关与多模型路由
- 限流、降级与熔断
- 灰度发布与回滚
- 私有化部署与本地推理
- 边缘部署与轻量化

### 第 17 章 性能与成本优化
- 上下文压缩与裁剪
- 模型蒸馏与小模型替代
- 推理加速（量化、KV Cache、批处理）
- 分层模型调用（大小模型协同）
- 提示词精简与复用
- 并行化与流水线优化
- 成本监控与预算控制

---

## 第六部分 进阶专题

### 第 18 章 强化学习与 Agent 自我进化
- 基于反馈的持续学习
- 强化学习基础（RL、RLHF）
- 从环境交互中学习
- 经验回放与技能积累
- 自我博弈与自我改进
- 在线学习与离线学习

### 第 19 章 多模态 Agent
- 视觉理解与图像输入
- 语音识别与合成
- 文档与表格理解
- 图文混合推理
- GUI Agent（屏幕理解与操作）
- 具身智能（Embodied Agent）
- 多模态工具调用

### 第 20 章 Computer Use 与自动化 Agent
- 浏览器自动化 Agent
- 桌面 / GUI 操作 Agent
- 屏幕截图理解与元素定位
- 操作规划与执行反馈
- 代码执行 Agent（Code Interpreter）
- RPA 与 Agent 融合

### 第 21 章 Coding Agent 专题
- 代码理解与代码库导航
- 代码生成与补全
- 自动化测试与调试
- 代码审查 Agent
- 多文件编辑与重构
- Agentic 编码工作流
- IDE 集成与开发者体验

### 第 22 章 前沿趋势与研究方向
- 长上下文与无限上下文
- Agent 的自主性与对齐
- 世界模型与规划
- Agent 编排的标准化
- Agent 市场与生态
- AGI 与 Agent 的未来
- 关键论文与研究前沿追踪

---

## 附录

### 附录 A 学习资源
- 经典论文清单
- 开源项目推荐
- 官方文档与教程
- 社区与技术博客

### 附录 B 实战项目建议
- 入门级项目
- 进阶级项目
- 综合级项目

### 附录 C 常用工具与平台速查
- 模型 API 与服务
- 向量数据库
- 开发框架
- 观测与评估工具
