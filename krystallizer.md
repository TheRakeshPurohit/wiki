# Krystallizer — Agent 记忆核心

*会话控制 + 原子事实图 + 混合检索的嵌入式记忆系统：机制与实现设计。属性图的概念定位见 [图谱化记忆](graph-memory.md)，通用记忆架构分析见 [Agent 记忆选型](agent-memory.md)。*

Krystallizer 是从 skillforge `src/mem` 独立出来的记忆系统，落在 [KV 存储引擎](kv-storage-engine.md)架构上：会话控制原语（branch / tail / summarize）、原子事实图（append-only KDL 事实 + 权重）、自建混合检索（arroy 向量 + 手写 BM25 + RRF）。嵌入式 Rust 核心（mem-core）+ Python 绑定（PyO3），零 LLM 依赖、零传输依赖。

## 双接口面：会话控制与记忆

记忆系统暴露且仅暴露两个接口面（机制与策略分离——核心只回答"怎么做"，"什么时候做"归调用方）：

### 接口面 1：会话控制（完全控制模式）

三个可组合的原语，agent 显式驱动：

```
branch(fork_point)    从任意前缀点分叉会话。checkpoint 不可变（前缀模型），
                      分叉零拷贝：分支共享 checkpoint 前缀，增量消息各走各的。

tail(prompt)          单轮分支。分叉 → 在尾部注入 prompt → 该轮运行 → 分支丢弃。
                      主会话历史不受影响（历史纯净性保证，结构性成立）。

summarize(...)        组合操作：注入压缩指令（经 tail）→ 收集摘要 → 截断被压缩
                      的消息 → 写新 checkpoint。会话收缩。
```

尾提示词注入和摘要化都只是分支操作：tail 是单轮分支，summarize 是注入+截断。一个机制，没有特例。

**触发策略上移到调用方。** 核心提供原语；*何时* 压缩（阈值、会话结束、显式命令）是 agent 侧策略。旧的自动触发退化为调用方的一种可选策略，不是核心行为。把策略埋进存储层意味着调用方无法关闭它、无法按 session 改阈值、无法在其他时机触发压缩——这是旧 `ConversationMemory` 把 `checkpoint_threshold` 硬接线进 `get_context()` 的教训。

### 接口面 2：记忆

两种模式：

- **Full 模式（full-session mode）**——`get_context` 返回完整上下文（checkpoint 摘要 + DB 增量 + pending 用户消息）。会话本身即记忆，检索不参与。
- **Assist 模式**——`store` / `search` / `forget` 操作 flat 记忆与图事实。检索到的上下文是**辅助**：与会话并列注入，从不替换会话。

## 会话即数据（无状态 agent）

Full 模式的架构推论：agent 循环退化为纯函数。

```
f(session, user_input) -> session'
```

Agent 按 `session_id` 取完整会话 → 执行 → 把新会话写回。循环对会话的全部职责是**追加**（含工具结果写入）：从不改写历史、从不从零拼装上下文、调用之间不持有会话状态。所有函数调用——记忆调用也不例外——对循环都是普通记录，调用只有一种模式。

这与 Aura 的「API 有状态、运维无状态」立场同构：会话持久化与恢复沉到记忆系统，actor/executor 在调用之间无状态，scale-to-zero 顺势免费获得——`on_sleep`/`on_wake` 退化为存取两个动作。

**视图层裁剪是另一套机制，不是循环逻辑。** 把函数调用格式从序列化视图中裁掉，发生在记忆系统的序列化路径上。存储层永远持有完整、未裁剪的会话；视图被裁剪，存储不被裁剪，裁剪严格只在读侧——任何写路径裁剪会让「存储的会话」与「agent 看到的会话」分叉成两个真相。

**记忆调用的消解（否决）**。曾评估过在视图中把记忆召回的调用记录替换为召回的上下文。否决：收益只是省一次往返，代价是循环必须知道哪些调用是记忆调用、在轮中重写上下文——插记录是数据层的事，重写轮中上下文成了循环的私活，一类函数调用获得循环内的特权待遇，破坏 append-only 不变量。无论是否采用消解，把行为放视图层都成立，而消解本身不值得。

**消解的缓存代价有界（供参考）**。前缀缓存是逐请求对最长公共前缀匹配的，不是逐会话整体匹配。把最后两条记录（tool_call + tool_result）消解成一条合并记录只改尾部；往前一百条字节不变、缓存照常命中。每轮损失限于被替换的尾部（几 KB 的重算），与分支剪枝同机制：追加 + 局部尾改，前缀保持稳定。「改写历史使缓存全量失效」是误解——改写点就是尾部。

**Agent 四分**：UI、智能会话上下文（会话+记忆接口面）、执行（工具）、循环。循环收缩为：取会话 → 跑 turn → 存会话。

**代价与转移**：

- 传输量增大：每次调用传输完整会话（平均几十条，视图中已裁剪函数调用格式）。LLM 请求本来就是全量上下文，这只是序列化成本，不是扩展性问题。
- 状态没有消失——是转移了。记忆系统从此承担持久化、并发写、视图时裁剪，成为会话真相的唯一持有者；多写者语义和读取时压缩是它的问题，但不阻塞本设计。

## 写入-读取循环

```
写入: 原始输入 → [阶段1: 规范化+分类] → [阶段2: 分类提取原子事实] → Fjall（工作记忆）→ 权重积累
                                                                    ↓ (weight > threshold)
读取: LanceDB（长期记忆）→ 向量 + 图混合检索 → 按权重排序返回
                    ↓ (weight < threshold)
冷存: S3 → 需要时 RAG 检索
```

和双轨记忆架构的生命周期完全对齐。新增的环节是：**权重决定了原子事实在双轨中的位置**，而不是简单的"新=热，旧=冷"。

### 提取机制的三个方案（演进记录）

#### 并行提取机制（⚠️ 过时，被Prefix Checkpoint 方案替代）

> 此方案要求对 Agent 进行改造（在 system prompt 中指示 LLM 同时输出回复和抽取原子事实的函数调用），存在三个问题：(1) 需要改造 Agent 框架；(2) 函数调用可能干扰 LLM 对用户回复的注意力；(3) 没有考虑上下文缓存（Context Cache）——每轮都引入新的函数调用 token，缓存命中率低。

记忆更新通过 LLM 的并行函数调用在后台发生。模型在一次响应中输出一组工具调用，Hermes 解析后异步执行。用户看到流畅的文本回复，系统在后台完成多维状态更新——不需要一个庞大的函数，模型动态编排细粒度工具。

#### 纯追加指令（⚠️ 过时，被Prefix Checkpoint 替代）

阈值到达后，在 prompt 末尾追加压缩指令，LLM 直接输出 checkpoint + memories（不通过 tool call）。

```
Turn 1..N: 正常对话，[checkpoint] + 增量逐条追加到 prompt，上下文缓存持续命中
              ↓ 达到阈值（100 条）
Turn N+1:  prompt 末尾追加 "分析以上对话，输出 checkpoint 和 memories"
           → LLM 直接输出结果，指令本身不写入 session
Turn N+2:  新 checkpoint + 新增量，重新开始
```

优势：LLM 注意力完全集中在压缩任务上（不需要同时回答用户），提取质量可能更高。
问题：(1) 需要改 Agent loop（识别特殊输出并处理）；(2) 压缩和回答分开，两次 LLM 调用；(3) 压缩时机不在记忆系统控制下。

#### 原位检查点压缩（Prefix Checkpoint，当前方案）

替代上述两种方案。核心思路：压缩不是单独的 LLM 调用，而是融入 Agent 的正常对话 turn——下一个用户提问时，注入尾提示词（不是 system prompt），Agent 一次 turn 同时完成提取和回答。

**四步闭环**：
1. **拦截与触发**：记忆系统监控 token 长度，达到阈值时修改用户消息注入压缩指令，前文 token 绝对不变，100% 命中上下文缓存
2. **原位函数调用**：主模型在完整上下文基础上，按顺序执行 generate_summary() + create_checkpoint()
3. **正常回复用户**：完成元任务后，继续基于压缩后上下文回答用户原始问题
4. **历史净化**：turn 结束后还原用户消息为原始内容，保证历史可重放、无伪指令

```
Turn 1..N: 正常对话，[checkpoint] + 增量逐条追加到 prompt，上下文缓存持续命中
              ↓ 达到阈值（100 条）
Turn N+1:  用户提问 X
           prompt 末尾追加: "先回答用户的问题，然后分析以上对话，
                           调用 memory_store 提取记忆，标记 checkpoint"
           → Agent 一次 turn: 先回答 Y，再 tool_call(memory_store) + tool_call(checkpoint)
Turn N+2:  历史变为 [ckpt_1] + [X] + [Y]，重新开始
```

**未压缩内容在 session 中**：阈值内的消息虽然还没被压缩为 checkpoint 或提取为长期记忆，但原始内容完整保存在 session_messages 表中，Agent 通过增量消息直接看到。不是"丢失"或"滞后"，只是还没被压缩——原始数据始终可用。

**历史纯净性**：尾提示词是临时的——只存在于当次 turn 的 prompt 中，从不写入 session。session 中存储的是 `[checkpoint] + [用户原始消息] + [Agent 回答]`，而不是 `[checkpoint] + [尾提示词 + 用户原始消息] + [Agent 回答]`。turn 结束后，如果 prompt 中修改了用户消息（追加指令），需要还原为原始内容，保证历史可重放、无伪指令。

在会话控制原语下，历史纯净性从"靠小心代码维持的约定"升级为结构性性质：尾提示词经 tail（单轮分支）注入，分支用完即弃，主会话历史天然不被污染。

**优势**：
- 不改 Agent loop——压缩通过正常 tool call 完成，记忆系统完全控制
- 不单独调 LLM——复用 Agent 的正常 turn，answer + compress 共享上下文缓存
- 用户体验无感——正常回答用户，同时后台完成压缩和记忆提取
- cache 天然命中——历史部分全部缓存，只有尾提示词 + 用户问题是新 token

**三种方案对比**：

| 维度 | 并行提取（旧） | 纯追加指令（旧） | Prefix Checkpoint（新） |
|:--|:--|:--|:--|
| Agent 改造 | 需要 | 需要 | 不需要 |
| LLM 调用 | 每轮 | 压缩单独一次 | 复用正常 turn |
| 注意力 | 分散 | 集中在压缩 | 分散（但 tool call 机制保障） |
| 控制权 | Agent | 不确定 | 记忆系统 |

### 基础策略：Checkpoint 压缩

Prefix Checkpoint 方案下，压缩融入 Agent 的正常 turn——阈值到达时，注入尾提示词，Agent 一次 turn 完成：

```
100 条消息 → 尾提示词 → Agent 一次 turn:
  1. 回答用户（LLM 直接输出）
  2. memory_store(...)  ← LLM 主动判断，提取长期记忆（flat 或 graph facts）
  3. memory_checkpoint(...) ← Agent 框架执行，纯 DB 操作
```

#### 一次调用双层输出

核心思想不变：一次 LLM 调用同时产出两层输出。只是实现方式从"直接输出 JSON"变为"通过 tool call"：

| 维度 | 旧方案（CheckpointCompressor） | 当前方案（Prefix Checkpoint） |
|:--|:--|:--|
| 触发方式 | 独立 LLM 调用 | prompt 尾提示词 → Agent 正常 turn |
| 输出方式 | LLM 直接输出 JSON（checkpoint + memories） | LLM 回答用户 + tool call（memory_store + memory_checkpoint） |
| 缓存 | 不共享（新 API 调用） | 共享上下文缓存（同一 turn） |
| 长期记忆 | LLM 在压缩调用中提取 | LLM 在 tool call 中提取 |

两种方案都是一次调用产出两层，但当前方案复用 Agent 的正常 turn，cache 命中率从 0% 提升到 ~99%。

#### 提取格式演进

| 阶段 | 提取格式 | 输出 |
|:--|:--|:--|
| 当前（Phase 2.5） | flat | `{content, category, importance, tags}` |
| 未来（图谱化） | graph facts | `{subject, predicate, object, category}` |

提取机制不变（LLM 通过 `memory_store` tool call），只是输出格式从 flat 变为 structured。

**序列化与可视化**：原子事实权威格式为 **KDL**——每个 `fact` 一个节点，固定 `head`/`rel`/`tail` 三个角色子节点：

```kdl
fact {
  head  <类型> <实体值> [head_props]
  rel   <关系> [relation_arg] [relation_props]
  tail  <类型> <实体值> [tail_props]
}
```

- `head`/`tail`：arg0 = 类型标签（小写单 token），arg1 = 实体值（有空格才引号）；节点 props = 实体的属性
- `rel`：arg0 = 关系（大写下划线）；节点 props = 关系的修饰（时间 / 条件 / 原因 / 强度）
- **空 props 不写**；类型词表开放

示例（`head_props`/`relation_props`/`tail_props` 全带 + 规则/逻辑类关系）：

```kdl
fact {
  head  project "银河"  priority=high
  rel   IMPLIES reason="受控写放大、保留因果链"  confidence=0.9
  tail  architecture "事件溯源"  status=chosen since="2026-08"
}
```

KDL 可确定性转换为 Mermaid 和 Cytoscape.js 渲染（不需要 LLM 参与；bracket `head[TYPE]RELATION[props]tail[TYPE]` 仅作线内展示）。Mermaid 用 dagre 层次化布局（适合检查原子事实提取是否正确），Cytoscape.js 用力导向布局（适合检查聚簇效果）。两者同一输入、两个输出。

#### 缓存利用

Prefix Checkpoint 的缓存优势：checkpoint 不可变，可永久缓存。增量部分每次变化，但历史部分（checkpoint）走上下文缓存。

当前方案不需要独立的压缩 prompt —— 压缩通过 Agent 正常 turn 完成，system prompt + few-shot 由 Agent 框架管理，记忆系统不直接控制 prompt 缓存。

#### 提取 Prompt

提取 prompt 定义原子事实输出格式：

```
你是一个对话压缩器。分析以下对话历史，输出两部分：

1. checkpoint：用 2-3 句话概括这段对话的核心内容、关键决策和当前状态
2. facts：提取值得长期记住的信息，格式：
   {"subject": "...", "predicate": "...", "object": "...", "category": "fact|rule|logic|preference"}
```

**依赖关系**：facts 依赖 checkpoint——LLM 先理解对话产出摘要，再基于摘要提取原子事实。早期方案中两者是独立输出（无依赖），现在合并为一次调用后自然形成依赖：checkpoint 是对话的全局理解，facts 是从中提取的结构化知识。

这是 `memory_store` tool 内部的 prompt 设计，不影响 Prefix Checkpoint 的整体流程。

### 编码场景：设计文档作为代码库 Checkpoint

M 问"有没有更好的方法让 AI 更了解我们的代码，不用每次都搜索"，O 回答"每次改完让他总结/更新设计文档，因为前面的有缓存，边际成本低，下次用的时候不用大量读取了，改的时候也很精确"。

这不是一个 tips——是计算时机光谱在编码场景的具体实例。

**没有设计文档时（读时重）**：每次会话 AI 重新读代码 → 理解 → 执行 → 遗忘 → 下次重新读。O(n) 的代码扫描，理解质量取决于代码可读性和 context window。

**有设计文档时（写时重）**：首次写设计文档（理解+结构化），后续读缓存的设计文档 → 精确执行。首次成本高，后续成本低。

**设计文档 = 代码库的 Checkpoint**：
- **不可变性**：两次修改之间不变 → 可缓存（和 checkpoint 摘要的物理属性一致）
- **捕获决策**：不只是代码结构，还有"为什么这样设计" → 精确指导未来修改
- **增量更新**：改完代码后更新文档的受影响部分 → 维护成本 O(1)

**边际成本递减**：prompt caching 机制下，设计文档一旦进入缓存，后续调用中这部分 token 的计算成本趋近于零。"前面的有缓存，边际成本低"——只有新增的代码变更需要增量处理。

**精确性收益**："改的时候也很精确"——有设计文档时，AI 知道该改哪里、为什么改、改了会影响什么。没有设计文档时，AI 需要从代码反推架构，容易遗漏依赖或误解设计意图。

**与 checkpoint 压缩的同构**：

| 维度 | 对话 checkpoint | 设计文档 |
|:--|:--|:--|
| 写入时机 | 每 N 条消息压缩 | 每次代码修改后更新 |
| 内容 | 对话核心内容+决策 | 代码架构+设计决策 |
| 缓存属性 | 不可变，可永久缓存 | 两次修改间不变，可缓存 |
| 读取方式 | 注入 Agent prompt | 注入 Agent prompt |
| 边际成本 | 趋近于零（缓存命中） | 趋近于零（缓存命中） |

两者是同一个模式的不同实例：**写时投入整理成本，读时享受零匹配开销**。

## 存储实现

存储选型、key 编码、检索实现见 [KV 存储引擎](kv-storage-engine.md)（属性图编码模式、delta 追加消除 read-modify-write、二级索引更新策略、WriteBatch 事务、向量冬眠/载入生命周期）。检索侧自建：arroy（HNSW）向量 + 手写 BM25 倒排 + RRF 融合，分词与打分全链路可控——向量与全文检索是 KV 路线需自建补齐的两模块，pg_search 黑盒 tokenizer 不可控正是 KV 路线的核心换取项。

## 交叉引用

- **[图谱化记忆](graph-memory.md)**：原子事实图的概念设计——计算时机光谱、聚簇策略、权重系统、图谱化 Skill。
- **[Agent 记忆选型](agent-memory.md)**：通用记忆架构分析——Surface/Engine 两层、注入方式、外部方案对比。
- **[KV 存储引擎](kv-storage-engine.md)**：存储层承载——属性图编码模式、读改写消除、二级索引更新策略、WriteBatch 事务。
- **[缓存树和尾提示词优化](tail-prompt-optimization.md)**：尾提示词的缓存旁路机制。
