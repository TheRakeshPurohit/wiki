# 无状态 Agent 架构

*一套区别于传统 Agent 框架的架构：无状态、云原生、serverless。记忆设计（checkpoint、检索、提取）见 [Agent 记忆选型](agent-memory.md)，记忆系统实现见 [Krystallizer](krystallizer.md)，Aura 引擎本体见 [Aura 架构](aura-architecture.md)。*

## 核心命题：turn 是唯一执行单位

传统 Agent 是常驻进程：内存里持有会话历史，进程死了会话就丢，多实例部署要解决粘性与状态同步。本架构把循环退化为纯函数：

```
f(session, user_input) -> session'
```

一个 turn = 接口拿到请求数据 → 从 Krystallizer 以 full 模式取会话 → 执行（LLM 调用 + 工具调用处理）→ 把新增消息 append 回 Krystallizer。循环在调用之间不持有任何状态，不改写历史——持久化与恢复全部沉到记忆系统。

这个推论的直接收益是**部署形态与执行位置解耦**：turn 落在哪个实例上无所谓，本地 CLI 就是循环执行 turn；分布式环境（Aura / 函数计算）里下一个 turn 可以在任何地方执行。无状态不是运维特性，是结构性质——scale-to-zero、多实例、跨机迁移全部免费获得。

理论根据：Aura 的「API 有状态、运维无状态」——ctx.state 有状态编程，状态持久化/恢复/scale-to-zero 沉底层。会话即数据（session as data）是同一立场的 agent 落地。

## 组件

| 组件 | 隐喻域 | 职责 | 形态 |
|:--|:--|:--|:--|
| **Krystallizer** | 结晶 | 会话真相唯一持有者：append-only 会话 + prefix checkpoint + 视图裁剪；技能图谱与涌现 | 记忆面（mem-core 嵌入或独立服务） |
| **Prism** | 棱镜 | 入口：鉴权、请求解析、把 turn 投进场域。WS 网关 + CLI（包装 WS） | 基于 Aura，纯入口，无队列无状态 |
| **Gravity** | 引力 | turn 执行器：取会话 → 跑 turn → 存增量。turn 被会话真相牵引运转；单趟执行（一个 turn 一趟），非常驻循环 | Aura 中的 Actor 类型（partition key = session_id）；自带 CLI 驱动循环，本地模式即循环执行 turn |
| **Probe** | 探针 | skill 运行时：容器化执行环境 + 操作触手。skill 定义从 Krystallizer 实时拉取，执行结果作为 tool result 回流 | Aura 基座组件，也可独立部署为远程服务 |

```
         请求
          │
       ┌──▼──┐  turn 投递    ┌─────────────────┐
       │Prism├──────────────►│ Gravity (Actor) │
       └─────┘               └──┬────────┬─────┘
                                │        │ tool call
              fetch session     │        ▼
              append delta   ┌──▼────────┐  skill spec  ┌───────┐
              (full mode)    │Krystallizer├─────────────►│ Probe │
                             │(memory +   │  (涌现式)     │local/ │
                             │ skill 图谱)│◄─────────────│remote │
                             └───────────┘  tool result └───────┘
```

### Prism：入口收缩

Prism 基于 Aura，刻意薄：鉴权 + 解析 + 投递。队列、重试、超时、跨机调度是 Aura 场域模型的本职——在 Aura 旁边再造带队列的网关违反 Aura 的 MQ 分解裁决（不引入独立队列组件）。Prism 只负责把用户侧请求翻译成场域内的一次 turn 投递。

**接口是 WS，CLI 包装 WS。** 对外只暴露一种协议：WS 网关（长连接承载 turn 投递与流式回推），CLI 是 WS 客户端的薄包装——本地命令与远程调用走同一条协议路径，不维护第二套 RPC 面。WS 的有状态性被 Prism 的位置吸收：连接的对面就是 Aura 场域（常驻），turn 的执行体（Gravity）仍然无状态单趟——连接钉在 Prism 上，不钉在 Gravity 上，scale-to-zero 语义不受损。

### Gravity：循环即函数

Gravity 的全部职责：Krystallizer full 模式取会话 → LLM 调用与工具调用循环 → 新增消息 append 回去。本质上每个 Gravity 是**单趟**的：一个 turn 一趟执行，跑完即止——没有常驻循环实体，「循环」只是 turn 的连续投递。名字取引力（Gravity）而非轨道（Orbit）正在于此：轨道是常驻物体的属性，引力是每趟都被施加的力——turn 落地即被会话真相牵引，执行完就释放，下趟另行落点。

实现为 Rust 外壳 + 多语言内嵌（Aura 既定拓扑：Rust 高性能外壳，Steel/Python/Wasm 经 PyO3/Wasmtime 内嵌，零 IPC）。Gravity 本身不感知工具脚本的语言——tool call 进、tool result 出，语言多样性在 Probe 侧解决。

**LLM 调用归属 Gravity**：压缩等尾提示词触发的 LLM 调用由 Gravity 实现——上下文缓存与模型（及账户）绑定，只有发起推理的一侧才知道模型信息。Krystallizer 只提供原语（分支注入、摘要收集、checkpoint 写入），不持有任何模型身份；唯一例外是它内部的向量嵌入（嵌入空间与数据正确性绑定，归记忆系统自持）。详见 [Krystallizer](krystallizer.md)。

**压缩的双模式**：

- **被动（默认）**：压缩决策在 Krystallizer——它持有全部上下文，信息最全（消息数、token 量、权重状态），阈值判断不该由调用方做。Gravity `fetch_session` 时，若达到压缩条件，返回的视图尾部**自带压缩尾提示词**：Gravity 收到什么执行什么，不感知这是压缩 turn；LLM 输出中的 `memory_store`/`checkpoint` tool call 自然回流，Krystallizer 在 append 时识别并处理（摘要入 checkpoint、facts 入图谱、截断旧消息）。对 Gravity 而言压缩 turn 与普通 turn 无差别——调用只有一种模式。
- **主动（外部信号）**：上游推理变慢、用户长时间无输入等事件只有 Gravity 感知得到（Krystallizer 看不见传输层与时间）。Gravity 主动调用压缩原语，**直接结束本轮**——本轮没有正常回复，是纯压缩 turn，不混合「回答 + 压缩」两个任务。

**pending 输入缓冲**：用户输入可能多条（连发几条才合并为一个 turn）。fetch 会话时 pending 输入落进 Krystallizer 的缓冲；压缩发生时缓冲一并进入 prompt 视图（压缩看到完整信息），commit 时缓冲随截断一并清理。缓冲在任何路径下都不丢输入：不压缩则正常 turn 消费缓冲，压缩则被摘要吸收。

**Krystallizer 侧的实现形态**：被动模式无独立接口——压缩逻辑藏在 fetch（视图组装时决定是否注入尾提示词）与 append（识别 memory tool call、写 checkpoint）内部，Gravity 完全无感；主动模式暴露显式原语 `summarize(session_id)`。两模式共享同一条压缩路径，入口不同而已。

## Surface 设计：注入与触发


### Prefix Checkpoint

跟 AI 聊天时，每说一句话，AI 都要从头读一遍之前的所有对话。对话越长，AI 读得越慢、越贵。

传统做法是"记住最近 N 条"——但每次新消息来了，原来的 N 条就变了（最旧的被挤出去，最新的被加进来），AI 相当于每次都要重新读一遍。

Prefix Checkpoint 的做法不同：

1. 对话积累到一定长度（比如 100 条），AI 自己总结一段摘要："用户在讨论项目架构，决定用 SurrealDB……"
2. 这段摘要**固定下来，永远不变**。后续所有对话都在它后面追加。
3. 因为开头不动，AI 可以"跳过"已经理解的部分，只处理新消息——就像看书翻到夹了书签的那页，不用从第一页重新读。

摘要就是"前缀"（Prefix），固定不变；"Checkpoint"是存档点。每次新存档后，旧存档清空，重新累积。

**为什么传统压缩无法利用缓存**：传统做法是把 100 条消息发给一个单独的模型（或单独发起一次 API 调用）来生成摘要。这个调用和之前的对话没有缓存关系——即使你用同一个模型，它看到的是一段全新的文本，上下文缓存从零开始计算，缓存命中率 0%。

Prefix Checkpoint 不同：100 条消息已经在缓存中（之前每轮对话都在用）。尾提示词只是在末尾追加了一小段新文本。模型处理时，前面 99% 的 token 直接走缓存，只有最后的提示词 + 用户问题是新计算。

```
传统压缩：
  [100 条消息] → 新 API 调用 → 全部重新计算 → 输出摘要
  缓存命中率：0%

Prefix Checkpoint：
  [100 条消息（缓存中）] + [尾提示词 + 用户问题]
  缓存命中率：~99%
```

结果：又快又好，消耗少。快是因为缓存，好是因为用的是同一个强模型（不是小模型），消耗少是因为大部分 token 不需要重新计算。而且不需要改造 Agent loop——压缩通过正常 tool call 完成。

**"辅助模型"≈传统模式**：能配置独立辅助模型的 Agent 系统，本质上就是传统压缩——需要单独发起一次 LLM 调用，无法利用缓存。所谓"快速模型"只是相对的：100 条消息发过去，小模型也要从头算一遍，延迟可能只少 30-50%，但质量差很多。Prefix Checkpoint 连辅助模型都不需要——压缩就是 Agent 正常回答的一部分。

### 记忆控制

Prefix Checkpoint 不仅是一个缓存优化机制，更是一个**记忆控制框架**。

AI 每次回复前都会读一遍完整的对话。如果我们想让 AI 做某件事（比如"把刚才聊的偏好记下来"），只需要在对话末尾加一句话就行了。

具体来说，当对话累积到 100 条时，记忆系统会在用户下一次提问时，在提问前面插入一段指令：

```
[之前的所有对话]           ← AI 已经理解了（缓存中）
[记忆系统插入的指令]        ← "先回答用户的问题，然后调用
                              memory_checkpoint(session_id, summary)
                              将对话压缩为摘要"
[用户的问题]               ← 用户真正想问的
```

AI 看到这段指令后，先回答用户问题，然后调用 `memory_checkpoint` 工具生成对话摘要并存入 checkpoint 表。用户完全无感——他只是问了一个问题，AI 回答了，但后台顺便完成了压缩。

**控制机制**：Agent 是被动的——它只看到 prompt 和可用工具，按照 prompt 中的指令调用工具。记忆系统通过控制两件事实现完全控制：
1. **注入什么** — 尾提示词（决定 Agent 做什么）
2. **提供什么工具** — Agent 可调用的工具列表（决定 Agent 能做什么）

**这使得记忆系统可以做到**：
- 控制压缩时机（阈值到达时注入压缩指令）
- 控制提取内容（提示词指定提取偏好/事实/决策）
- 控制存储方式（工具决定写入哪个表、什么格式）
- 不改 Agent loop — 所有控制通过 prompt + 工具实现

**类比**：Agent 是一个"没有主见的执行者"——它有能力（LLM 推理 + 工具调用），但没有意图。记忆系统通过 prompt 注入意图，Agent 负责执行。这和操作系统的"系统调用"类似：内核提供能力（syscall），用户态程序决定何时调用。

### 触发时机

三种记忆，三种触发机制：

| 记忆类型 | 工具 | 谁触发 | 机制 | 优点 | 缺点 |
|:--|:--|:--|:--|:--|:--|
| **长期记忆** | `memory_store` | **LLM**（主动判断） | LLM 在对话过程中判断"这条值得记住"时调用 | 精准，只存有价值的 | 依赖 LLM 判断力，可能遗漏 |
| **短期记忆** | `memory_checkpoint` | **记忆系统**（阈值触发） | 消息累积到 N 条时，prompt 注入尾提示词，Agent 框架执行 DB 操作 | 低频调用，开销可控 | N 条内未压缩（但原始消息仍在 session 中） |
| **记忆检索** | `memory_search` | **LLM**（主动判断） | LLM 在对话过程中判断"需要查一下"时调用，同时搜索 memories + session_checkpoints | 跨会话知识检索，支持历史对话摘要 | 依赖 LLM 判断力，可能遗漏 |
| 短期记忆变体 | `memory_checkpoint` | **定时任务** | 用户不活跃时（如凌晨）后台触发 | 不影响用户体验，利用闲置算力 | 需要定时任务基础设施 |
| 短期记忆变体 | `memory_checkpoint` | **流式计算** | session 闲置 + cache 快过期时触发，续期 checkpoint | 防止 cache 过期失效 | 需要监听 session 活跃状态 |
| 短期记忆变体 | `memory_checkpoint` | **图谱聚簇** | 检测到对话主题切换时触发，按语义边界压缩 | 摘要更清晰 | 需要主题检测能力（图谱化记忆） |

### 主动触发（长期记忆）

Prefix Checkpoint 的记忆控制不仅用于压缩，还用于**主动记忆**—— LLM 在对话过程中判断"这条值得记住"时，主动调用 `memory_store` 工具，存入 `memories` 表。

触发方式是**用户要求或暗示**：
- 用户明确说"记住这个"、"以后都这样做"
- 用户表达了偏好、习惯、决策（LLM 判断值得长期保存）
- LLM 发现了重要的事实或上下文

LLM 不主动捕捉执行过程中产生的知识——它用判断力筛选，只存储有价值的信息，精准但依赖判断力。

### 其他触发机制

**夜间定时触发**：用户不活跃时（如凌晨），后台定时任务触发压缩。利用闲置算力完成压缩，不影响白天的用户体验。适合压缩成本较高（长对话、大模型）的场景。

**长期闲置触发（流式计算）**：session 长时间无活动时触发压缩，可结合 cache 到期时间——在 cache 快过期时主动续期。本质是**流式计算**：持续维护热缓存，防止 cache 失效导致下次对话时全量重算。和 Prefix Checkpoint 配合：压缩后新 checkpoint 作为前缀，cache 重新命中。

**主题转换触发**：检测到对话主题切换时触发压缩。按语义边界压缩，摘要更清晰（"前半段讨论架构，后半段讨论部署"比"100 条消息的混合摘要"更有用）。需要主题检测能力，与图谱化记忆配合——图谱中的聚簇边界就是天然的主题边界。

这是记忆系统的六种触发方式，按记忆类型分组：

| 记忆类型 | 方式 | 时机 | 谁触发 | 存什么 |
|:--|:--|:--|:--|:--|
| **长期记忆** | 主动触发 | 对话过程中 | **LLM**（主动判断） | LLM 判断值得记住的 → `memories` 表 |
| **短期记忆** | Prefix Checkpoint | 阈值到达时 | **记忆系统** | 对话摘要 → `session_checkpoints` 表 |
| **记忆检索** | 主动触发 | 对话过程中 | **LLM**（主动判断） | 搜索 `memories` + `session_checkpoints` |
| 短期记忆变体 | 夜间定时 | 用户不活跃时 | **定时任务** | 同 Prefix Checkpoint |
| 短期记忆变体 | 长期闲置 | session 闲置 + cache 快过期 | **流式计算** | 续期 checkpoint |
| 短期记忆变体 | 主题转换 | 语义边界 | **图谱聚簇** | 按主题分段的摘要 |

### 注入方式

| 方式 | 机制 | cache 友好度 |
|:--|:--|:--|
| **全量注入** | 每轮把所有历史注入 prompt | 差（内容每轮变化，上下文缓存失效） |
| **top-K 检索** | 每轮检索相关记忆注入 | 中（检索结果可能变化） |
| **checkpoint + 增量** | 不可变 checkpoint + 最近 N 条消息 | 高（checkpoint 固定，可永久缓存） |

#### checkpoint + 增量机制

类似 event-sourcing 的快照模式。消息逐条写入 session_messages 表，达到阈值时压缩为 checkpoint（写入 session_checkpoints 表），后续所有未压缩消息即为增量：

```
msg_001 ... msg_100    ← 消息追加到 session_messages 表
              ↓ 达到阈值（100 条）
ckpt_0: summary="用户讨论了项目架构，决定用 SurrealDB..."
              ↓ 写入 session_checkpoints 表，不可变
msg_101 ... msg_150    ← 全部是增量（当前所有未压缩消息）
              ↓ 再次达到阈值
ckpt_1: summary="..."
msg_151 ...            ← 新的增量
```

**读取流程**：

消息一条条追加到 prompt 中（不是每次重新拼接）。LLM API 的多轮对话机制天然缓存前面所有 tokens：

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]       ← 前面的 tokens 走上下文缓存
Turn 3: [ckpt] + [msg_101] + [msg_102] + [msg_103]
...
Turn N: 达到阈值 → 融入下一个 Agent turn 完成压缩
Turn N+1: [ckpt_1] + [用户问题 X] + [Agent 回答 Y]
```

**写入流程**：
1. 每条消息追加到 session_messages 表
2. 检查距上次 checkpoint 的消息数是否 >= 阈值
3. 达到阈值 → 标记需要压缩 → 下一个用户提问时，将尾提示词作为普通消息追加到 prompt 末尾

**Prefix Checkpoint**：阈值到达后，下一个用户提问时，记忆系统注入尾提示词（不是 system prompt）：

```
上下文缓存中（不变）：
  [ckpt] + [msg_101..msg_150]

新追加的消息（唯一未缓存的部分）：
  "先回答用户的问题，然后调用 memory_checkpoint(session_id, summary)
   将对话压缩为摘要"

Agent 一次 turn 完成两件事（注意顺序）：
  1. "Y"                              ← 先回答用户问题（用户体验优先）
  2. tool_call: memory_checkpoint(    ← 再压缩对话为摘要
       session_id, "这段对话讨论了...")
```

**历史纯净性**：尾提示词是临时的——只存在于当次 turn 的 prompt 中，从不写入 session。session 里永远是纯净的历史：

```
session 中存储的：
  [checkpoint_1] + [用户原始消息 X] + [Agent 回答 Y]

而不是：
  [checkpoint_1] + [尾提示词 + 用户原始消息 X] + [Agent 回答 Y]
```

即使 prompt 中为了触发压缩而注入了尾提示词，session 中也不会残留——用户消息通过 `push_user_message` 缓冲在内存中，`get_context()` 构建上下文时才加入，`write_assistant_message()` 写入 DB 后清空。session 里永远是纯净的历史：
- 历史可重放——任何时候重新加载 session，内容都是真实的对话
- 无伪指令——session 中不会残留系统指令
- 逻辑连贯——checkpoint + 原始消息 + 回答，构成完整的因果链

**顺序很重要**：必须先回答用户问题，再做记忆操作。用户问了一个问题，如果先看到一堆工具调用在跑，体验很差。提示词中明确要求"先回答，再处理记忆"——用户看到的第一个输出就是答案，记忆操作是"顺便"完成的。

**为什么这样设计**：
- **不改 Agent loop** — 压缩通过正常 tool call 完成，记忆系统完全控制
- **不单独调 LLM** — 复用 Agent 的正常 turn，answer + compress 共享上下文缓存
- **cache 天然命中** — 历史部分全部缓存，只有尾提示词 + 用户问题是新 token
- **用户体验无感** — 正常回答用户，同时后台完成压缩和记忆提取
- **历史自动清理** — turn 结束后旧消息不再需要，历史变为 `checkpoint_1 + X + Y`

**为什么 cache 友好**：checkpoint 一旦写入不可变，后续所有 turn 共享同一个 checkpoint 文本。LLM API 的 prompt caching 将 checkpoint 部分缓存在 GPU 显存中，只有增量部分每次变化。随着增量消息增多、下一次 checkpoint 触发，增量被压缩为新的固定摘要，缓存再次命中。

**与 Agno 滑动窗口的对比**：Agno 的 `add_history_to_context` 每次从 DB 读最近 N 条完整消息。随着新消息到来，N 条的组成不断变化（旧的被挤出、新的被加入），上下文缓存每轮失效。checkpoint 模式下，历史被压缩为固定摘要，只有未压缩的增量部分变化，cache 持续命中。

### 框架适配

**铁律：mem 模块厚，agent 适配层薄。**

适配层只做三步调用，不实现记忆逻辑：

```python
# 适配层（~20 行）
memory.push_user_message(session_id, user_message)   # 缓冲用户消息
context = memory.get_context(session_id)              # checkpoint + DB + pending + 尾提示词
agent.additional_input = context                       # 注入到 agent
agent.run()
memory.write_assistant_message(session_id, assistant)  # 写入 DB + 清空 pending
```

压缩检测、尾提示词注入、pending 管理全部在 WorkingMemory 内部。换框架时只重写适配层（~20 行），Engine 完全复用。

框架适配之上是会话即数据的无状态模式——三步调用收缩为「取会话 → 执行 → 存会话」，压缩决策沉到 Krystallizer 内部（被动模式 Gravity 无感），见本文压缩双模式一节与 [Krystallizer](krystallizer.md)。

**自带 CLI 驱动循环**：Gravity 提供本地 CLI 形态——进程内循环执行 turn（取会话 → 跑 → 存，下一 turn），这是本地/单机模式；同一 Gravity 函数在 Aura 中注册为 Actor 类型时，每条 turn 事件触发一趟执行，是分布式模式。同一执行函数，三种驱动：CLI for 循环（本地）、Aura Actor（分布式）、函数计算（serverless）——CLI 就是函数的 for 循环，Actor 就是函数的单次调用。

Aura 中 Gravity 是一个 Actor 类型：同一会话串行（Actor 单线程语义，partition key = session_id），不同会话并行；turn 之间 scale-to-zero，`on_sleep`/`on_wake` 退化为存取两个动作。流式输出经高频 emit 事件转 SSE 推送——传输面由 Prism 的 WS 网关承载（Gravity 与 Prism 之间仍是场域事件，无直接连接）。

### Probe：执行与触手

Probe 是 skill 的运行时环境——**执行只提供运行时，不在 Krystallizer 中执行**。容器隔离，安全；skill 代码不被信任，Probe 给它一个受限世界。

两种部署形态，同等支持：

- **Aura 内嵌**：作为 Aura 执行基座（Wasmtime 沙箱谱系的重隔离端——Wasm 管不动真文件系统/真网络/系统包时，容器顶上），场域内调用触达。
- **远程触手**：GitHub Actions runner 模式——Probe 主动 outbound 注册 + 拉任务（不开入站端口），部署在用户自己的电脑或目标服务器上，就是那台机器的操作触手：部署在哪，就能操作哪。控制面（Aura/Gravity）永远可达 Probe 的 outbound 连接，Probe 所在网络的入站拓扑无关紧要。

**skill 分发：每次 tool call 实时拉取，零缓存。** 涌现的前提是零陈旧窗口——一个实例踩坑解决后存进图谱，任何地方的下一次执行立即拿到新版。skill 生命周期对齐到 tool call 粒度，与「调用只有一种模式」同构：skill 拉取是普通读取，不是需要失效策略的缓存问题。

**拉取路径：Probe → Gravity → Krystallizer，不直连。** 三条理由：访问控制——Krystallizer 只需信任 Gravity 一层，容器（可能跑不信任 skill）不持有数据面凭证；网络拓扑——远程 Probe 只有 outbound 可达控制面，未必能直连存储网；注入点——Gravity 代取时做视图处理与 `tool_invoke_count` 权重回写，这是涌现回路的数据关口，直连会绕开。多一跳 RTT 被 LLM 推理间隙完全吸收（拉取只发生在 tool call 时，个位数次数），不构成瓶颈。

### skill 涌现闭环

skill 不是静态文件，是 Krystallizer 图谱中高工具指数的子图（KDL 原子事实 + 权重系统）。闭环：

```
执行经验（踩坑/解法/操作序列）→ 存入图谱（原子事实）
        ↓ 聚类涌现
高权重子图 = skill 边界 → Gravity 运行时发现（向量搜索 + 权重排序）
        ↓ tool call
Probe 拉取执行 → tool_invoke_count 回写权重 → 影响下次排序
```

涌现的原料回路复用 Krystallizer 既有机制（权重 delta 计数、图聚类），不新建系统。

## 传输裁决：入口 WS，内部无连接

WS 用在唯一有状态的位置：Prism 与客户端之间（用户到入口，长连接天然贴合交互会话）。WS 不进入执行路径——Gravity 与 Prism/Probe 之间是 Aura 场域事件，无直接连接；turn 的执行体无状态单趟，连接钉在 Prism（常驻）上，不钉在 Gravity 上。

这条裁决替代了此前的「HTTP + SSE 默认」方案。修正的理由：连接状态的问题不在 WS 本身，在**连接钉在哪**。Prism 基于 Aura（常驻、多实例由场域调度），把连接收在 Prism 上，WS 的负载均衡/断线重连由 Aura 的连接面统一解决一遍，不会渗入执行层；而 HTTP+SSE 方案会让每个组件各自暴露 HTTP 端点，入口协议碎片化。CLI 包装 WS 后，全部客户端（人、CLI、脚本）走同一条协议，**调用只有一种模式**在传输层也成立。

中途交互仍被 turn 模型消解——人在 turn 进行中插话 = 新输入 → 新 turn（append 进会话，并发收敛由 Krystallizer 会话控制原语裁决）；权限确认是会话里的一个待决 turn，不是 WS 上的特权 frame。

## 与传统 Agent 架构的对照

| 维度 | 传统（常驻进程） | 本架构（无状态） |
|:--|:--|:--|
| 会话持有 | 进程内存 | Krystallizer（append-only + checkpoint） |
| 循环形态 | 常驻 loop，内存累积 | 纯函数 `f(session, input) -> session'` |
| 多实例 | 粘性会话 / 状态同步 | 免费——turn 落任何实例 |
| 挂起恢复 | 自研持久化 | scale-to-zero = 存取两个动作 |
| 部署 | 常驻服务 | 本地循环 / Aura Actor / 函数计算同构 |
| 工具执行 | 进程内或 RPC | Probe（容器/触手），skill 实时涌现 |
| 传输 | 常见 WS 长连接贯穿执行层 | 入口 WS（Prism），执行路径无连接 |

## 交叉引用

- **[Krystallizer](krystallizer.md)**：记忆系统实现——会话控制原语、full 模式、视图层裁剪、技能图谱。
- **[Agent 记忆选型](agent-memory.md)**：记忆架构通用分析——Surface/Engine 两层、注入方式、外部开源方案对比。
- **[Aura 架构](aura-architecture.md)**：引擎基座——场域模型、Actor 语义、MQ 分解、多语言内嵌拓扑。
- **[图谱化记忆](graph-memory.md)**：skill 涌现的概念设计——计算时机光谱、聚簇策略、权重系统。
