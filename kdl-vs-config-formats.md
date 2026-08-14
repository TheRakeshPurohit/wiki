# KDL 与配置/序列化格式全景对比

**状态**：分析与选型参考
**日期**：2026-08-13
**核心观点**：KDL 的节点模式是有主次的「语义树」，映射类似 HTML/XML 或命令行调用的逻辑树；而 TOML 是扁平键值对，JSON/YAML 是嵌套数据网络。这是与所有主流程配置格式分道扬镳的哲学底色。

---

## 什么是 KDL

KDL（读作 *cuddle*）是一种现代的、基于节点（Node-based）的文档语言。它不像 TOML 那样是扁平的键值对，也不像 JSON/YAML 那样是纯粹的嵌套字典/列表。KDL 的底层模型是有向树。一个 KDL 节点（Node）天然包含四个维度：

- **节点名称（Node Name）**：代表"我是谁"（如 `server`、`user`）。
- **位置参数（Arguments）**：同行内没有键名的纯值（如 `"localhost"`、`8080`），靠位置区分。
- **具名属性（Properties）**：同行内的键值对（如 `timeout=10.0`），靠键名区分。
- **子节点块（Children Block）**：嵌套在花括号 `{}` 内部的后代节点。

示例（核心模型）：节点名 + 位置参数 + 具名属性，一行表达完整语义：

```kdl
button "提交" id="submit-btn"
```

- `button`（节点名）：代表"我是谁"，具有最高的语义统治力
- `"提交"`（位置参数）：代表节点的核心内容
- `id="submit-btn"`（具名属性）：修饰节点的元数据

---

## 一、JSON 嵌套模式：数据网络，众生平等

在 JSON 中，一切都是数据。键值对之间、数组元素之间没有"主次之分"。要表达语义，必须通过人为约定的特殊键名（如 `_type`、`__meta__`）打补丁。

```json
{
  "type": "button",
  "id": "submit-btn",
  "children": [ "提交" ]
}
```

---

## 二、KDL/JSON/YAML 三格式底层对比

抛开生态与历史兼容的包袱，将 KDL 作为纯粹的"现代数据交互格式"考察时，它是唯一一个把**语义控制权**内嵌在语法骨子里、同时具备**代码级防错性**的树形格式。

### 结构表达：多维节点模型 vs 纯扁平数据模型

KDL 传递的不只是"键和值"。每个数据单元（Node）天然具备主次维度，接收方通过节点名立刻得知一行数据的语义，无需像 JSON 那样套多层字典：

```kdl
// 一个节点传递：实体类型(user)、唯一标识(001)、属性状态(active)、核心内容("Alice")
user 001 active=#true "Alice"
```

JSON/YAML 只能表达三种元素：映射（Map）、序列（List）、标量（Scalar）。面对上述混合语义，必须强行"拍扁"或"套娃"，导致传输文本含大量为凑结构而产生的冗余键名：

```json
{ "type": "user", "id": 001, "status": { "active": true }, "value": "Alice" }
```

### 三格式对比表

| 对比维度 | KDL（节点模式） | JSON（通用数据模式） | YAML（缩进数据模式） |
|:--|:--|:--|:--|
| 视觉噪音（体积） | 低。砍掉冒号和逗号，数据密度高 | 高。深层嵌套时满屏 `{} [] , ""` | 极低。无任何括号和逗号 |
| 解析器复杂度 | 极低且严谨。规范仅几页纸，各语言解析行为高度一致 | 极低。简单到可手写解析器 | 极高。规范超过 80 页，易因缩进或特殊符号导致跨语言解析报错 |
| 混合语义表达力 | 天生自带（同时传 Args 与 Props） | 无（必须靠人为约定的结构） | 无（必须靠人为约定的结构） |
| 网络传输抗损性 | 极高。层级靠 `{}` 锁死，缩进不敏感 | 极高。层级靠 `{} []` 锁死 | 极差。网络出错、Git 合并冲突或空格丢失会彻底破坏结构 |
| 流式传输（Streaming） | 极佳。一行一独立节点，天然适合按行读取的大数据流（类似 JSONL） | 较差。须依赖变体格式（如 JSON Lines），原生 JSON 必须一次性读完整个大括号 | 极差。须依赖 `---` 分隔符，解析成本高 |

### 传输鲁棒性：类型固化

数据交换中最怕"数据类型在传输过程中变异"。KDL 的字面量设计严格，且拥有原生强制的类型标签（Type Tags），可直接在数据中焊死类型：

```kdl
transaction amount=(f64)100.0 timestamp=(datetime)"2026-08-13T10:00:00Z"
```

不管接收端用 Python、Go 还是 Rust，解析器看到 `(f64)` 与 `(datetime)` 都绝不会误认成整数或纯字符串。

- **YAML（反面教材）**：过度猜测类型——著名的挪威问题（国家代码 `NO` 被识别为布尔 `false`）、版本号（`version: 2.10` 变浮点数 `2.1`），在数据传递中是灾难性的
- **JSON（勉强及格）**：只有基础的 String/Number/Boolean。遇到日期、UUID、Base64 只能由前后端约定"用字符串传"，机器无法在语法层做强校验

### 代码实战：表达"一首带属性和标签的歌曲"

为表达"歌名是主内容、流派是数组标签、评分是元数据属性"，JSON 需把结构做得臃肿：

```json
{
  "song": {
    "title": "Hotel California",
    "attributes": {
      "rating": 5.0,
      "released": 1976
    },
    "genres": ["Rock", "Classic Rock"]
  }
}
```

KDL 的立体节点模型在单一节点直接容纳所有维度，甚至无需展开子节点：

```kdl
// 节点名    位置参数(主内容)        具名属性(元数据)            位置参数列表(标签组)
song       "Hotel California"  rating=5.0 released=1976   "Rock" "Classic Rock"
```

纵向排版亦可：

```kdl
song "Hotel California" {
    meta rating=5.0 released=1976
    genres "Rock" "Classic Rock"
}
```

---

## 三、主流格式战略权衡

| 格式 | 语法哲学 | 核心痛点 | KDL 输在哪里 | KDL 赢在哪里 |
|:--|:--|:--|:--|:--|
| TOML | 基于节（Section）的扁平 `键 = 值` | 深度嵌套时表头冗长（如 `[a.b.c.d]`），极其繁琐 | 生态极其庞大成熟（Rust Cargo、Python 标准选择），大众熟悉 | 天生擅长深层树状嵌套与复杂的对象数组 |
| YAML | 依赖缩进的数据网/图 | 缩进地狱；过度猜测类型（详见 [第二章](#二kdljsonyaml-三格式底层对比) 传输鲁棒性） | 目前是运维和 CI/CD 绝对主流（Kubernetes、GitHub Actions） | 传输高度稳健。用 `{}` 划分层级，对缩进和跨平台复制不敏感 |
| JSON | 机器友好的标准数据模型 | 满屏逗号、冒号和双引号，人类手工维护和阅读时视觉噪音大 | 语言级原生映射，任何语言一行代码即可转为内置字典/对象 | 混合语义表达力。一行内容纳语义名、主内容、元数据和标签 |
| HTML | 标记语言（Markup） | 过于冗长，强制闭合标签（如 `</div>`）导致文件体积虚胖 | 全球浏览器唯一标准渲染骨架 | 极高的数据密度。砍掉所有尖括号，保留完美的节点树模型 |

---

## 四、KDL 的底层行为特征

- **换行敏感，缩进不敏感**：一行代表一个节点（或用分号 `;` 隔开）。但缩进不影响解析，花括号 `{}` 才是层级关系的绝对权威——排版凌乱不会导致解析失败。
- **严格的内置类型标签**：KDL v2 支持在解析器层面做强类型转换（如 `(f64)10.0`、`(date)"2026-08-13"`），从源头规避数据类型误判。

---

## 五、关键概念澄清与工程避坑

### 澄清 1：列表/数组的语法隐式性

KDL 语法本身不像 JSON 那样在中括号 `[]` 层面区分"单体对象"还是"对象数组"：

```kdl
token {
    extractor type="header" name="access-token"
    extractor type="cookie" name="HrmApiCookie"
    verify_as type="body" field="token"
}
```

仅看这段文件，无法从语法上断定 `verify_as` 只能写一个、`extractor` 可以写多个。

KDL 把"单数还是复数"的决定权留给你的代码（如 Pydantic 模型或 Rust 结构体）。若想在文件层向人类提示这是个列表，最佳实践是加一层复数命名的包裹节点，或利用 v2 的短横线（`-`）语法：

```kdl
extractors {
    - type="header" name="access-token"
    - type="cookie" name="HrmApiCookie"
}
```

### 澄清 2：键值对能出现在哪里

新用户极易像写 TOML/JSON 那样，在花括号 `{}` 内部新起一行直接写 `key = value`——这是语法错误。KDL 中换行代表新节点，新行开头必须是节点名，键值对只能附着在节点名称的同行参数区域，不能独立成行。

```kdl
// ❌ 键值对不能独立成行
database { host = "localhost" }

// ✅ 用节点名充当"键"，位置参数充当"值"（KDL 最佳配置风格）
database { host "localhost" }
```

### 澄清 3：位置参数和键值对的代码提取

两者解析后存放在完全不同的容器中：

- 同行内没有等号的纯值（如 `surreal "http://..."`）按顺序装在 `node.args`（Python 列表）中
- 带等号的键值对（如 `username="master"`）装在 `node.props`（Python 字典）中

设计配置时，通常将节点的灵魂主体设为参数，将修饰性细节设为属性。

### 澄清 4：工具链——nushell 内置 KDL

流行的独立 KDL 命令行工具 `kq` 已停滞（4-5 年未更新），无法完全兼容 2024 年底发布的 KDL v2 标准。但 **nushell 0.114.0（2026-06）起内置 `from kdl` / `to kdl`**，消除了日常使用最大的障碍：

```nu
"node attr=1 {bloc}" | from kdl
```

输出为正则节点行（`name` / `args` / `props` / `children`），可直接进入 nushell 的管道生态做过滤和处理——无需为了工具链而引入 `/bin` 级的外部依赖。`open` 对 `.kdl` 后缀也自动触发解析。详见 [Nushell 介绍](nushell-introduction.md) 的内置格式处理一节。只有需要与 `jq`（而非 nushell）协作时，才用 `kdl-py`/`kdljs` 将 KDL 转为 JSON 再输入 `jq`。

### 澄清 5：行长度纪律——用子节点下沉，不用硬性计数

KDL 单行高密度是核心优势，但一个节点（节点名 + 参数 + 属性）承载语义过重时，一行会过长、难以阅读。正确做法**不是**设「节点名、属性最多 ≤3 个」这类硬性数字上限——数字是错误信号：行长的真正决定因素是单位语义的字符密度，两个短标量参数的视觉负担与两个长字符串参数差一个数量级；且硬性砍个数会对抗格式自身的单行表达力。规则应引导使用 KDL 内置的逃生机制——**子节点块（`{}`）下沉**：

> **行长度纪律**：当一行（节点名 + 参数 + 属性）感官过重或超过约 100 字符时，将该节点的*修饰性*维度下沉到 `{}` 子节点块——保留主内容在 args（通常 ≤2–3 个），把标签组、元数据属性、长文本移到子节点或 props。下沉的是**次要维度**，不是机械地砍个数。

上文 song 示例（[二、KDL/JSON/YAML 三格式底层对比](#二kdljsonyaml-三格式底层对比) 的纵向排版）即为正例：

```kdl
song "Hotel California" {
    meta rating=5.0 released=1976
    genres "Rock" "Classic Rock"
}
```

边界权衡：args（主内容）保持少，props（元数据）可适度放宽。判断标准是「这行读起来是否顺畅」，不是「数了几个」。

## 六、极端场景：高光 vs 认瘪

### 高光时刻：传递"图表 / 关系 / 指令"型数据

系统需要高频传输工作流图、UI 渲染树、自动化测试步骤、或带丰富元数据的日志时：

```kdl
// KDL 版本：一眼看清步骤、重试次数和执行环境
pipeline {
    step "build" retry=3 env="PROD"
    step "test" timeout=60 {
        parallel #true
    }
}
```

这种数据用 KDL 传输，可读性、紧凑度和安全性无懈可击。

### 认瘪时刻：传递"纯巨量对象矩阵（Data Dump）"

当数据是纯粹从数据库倒出来的、包含十万条整齐划一用户信息的矩阵（大批量用户列表）：

- **JSON 最直接**：`[{"id":1, "name":"a"}, {"id":2, "name":"b"}]`，代码里直接转为 `List[User]`
- **KDL 需纠结**：用位置参数 `user 1 "a"` 还是具名属性 `user id=1 name="a"`。因模型比纯 JSON 复杂，大规模"纯表格式数据 dump"时，解析成高级语言对象的内存开销和转换步骤会略多

---

## 七、CLI / 结构化指令 / Agent Skill 场景的维度级优势

在 CLI、结构化命令调用以及 AI Agent Skill（工具调用）场景下，KDL 相比 JSON 具有压倒性的、甚至维度级别的优势。此时 KDL 已不仅是"配置格式"，而是一个天然的微型 DSL（领域特定语言）。

### 语法模型与 CLI 命令的天然拟合

CLI 命令调用的本质结构是 `命令名称 [位置参数...] [--选项=值...]`：

- **KDL 语法模型**：`节点名 [Arguments...] [Properties...]`
- **JSON 语法模型**：`{ "key": "value" }`

KDL 的设计与 CLI 调用逻辑 100% 吻合。对比表达一个复杂的 Agent Skill 调用（用 `ffmpeg` 剪辑视频，含位置参数、布尔开关、具名属性）：

```json
// JSON：必须把"位置"和"键值"塞进同一字典，或定义 args 数组 + options 字典，视觉割裂
{
  "skill": "video_clip",
  "args": ["input.mp4", "output.mp4"],
  "properties": {
    "start": "00:01:00",
    "duration": 30,
    "overwrite": true
  }
}
```

```kdl
// KDL：无需任何包装，写出来就是一段强类型的结构化命令行
video_clip "input.mp4" "output.mp4" start="00:01:00" duration=30 overwrite=#true
```

传给后台后：`node.name` 直接是 Skill 名称（`video_clip`），`node.args` 自动拿到输入输出路径，`node.props` 自动拿到裁剪参数——零转换。

### Agent Skill（工具调用）场景的三大优势

**优势 A：LLM 生成 KDL 的高成功率与低 Token 消耗**

AI Agent 调用 Skill 时，需 LLM 生成对应的结构体（Function Calling / Tool Call）。

- **Token 减重**：JSON 充斥双引号、冒号、逗号（视觉噪音）。转成 KDL 砍掉这些无用符号，LLM 生成 Token 显著减少，降低 API 成本、提升生成速度
- **语法纠错**：LLM 生成巨量 JSON 极容易因漏一个逗号 `,` 或右括号 `}` 而整段语法错误。KDL 用换行和空格分隔，对 LLM 容错率更高、生成难度更低

**优势 B：多行长文本（Prompt/Instructions）的完美容纳**

Agent Skill 常需传递大段提示词或多行脚本：

```json
// JSON 的噩梦：多行文本须手动转义 \n，整段挤在一行
{ "prompt": "你是一个助手。\n请严格遵守规则。\n不要说废话。" }
```

```kdl
// KDL 的优雅：原生支持 Roster 多行文本块 `"""`，完美保留排版
skill "coder" {
    system_prompt """
        你是一个高级 Rust 工程师。
        请编写符合安全规范的代码。
        """
}
```

**优势 C：完美的混合流式管道表达**

复杂 Agent 工作流中，Skill 常以管道（Pipeline）或有向无环图（DAG）形式串联。KDL 的嵌套模式可优雅表达"命令的嵌套与依赖"，而 JSON 表达树状依赖时会迅速演变成嵌套黑洞：

```kdl
// 用 KDL 表达一个 Agent 自动化流水线
pipeline name="daily_report" {
    fetch_data source="surrealdb" query="SELECT * FROM logs"
    analyze_sentiment model="qwen3.6-flash" {
        fallback_model "deepseek"
    }
    send_email to="boss@company.com" subject="今日简报"
}
```

### 行业背书：Zellij 全盘采用 KDL

Rust 社区的终端复用工具 [Zellij](https://zellij.dev/)（现代版 tmux）完全抛弃 JSON/YAML，其布局（Layouts）和插件控制文件全盘采用 KDL。用户或 Agent 编写界面网格、调用终端命令的配置：

```kdl
layout {
    pane size=1 borderless=true {
        plugin location="status-bar"
    }
    pane split_direction="vertical" {
        pane edit="src/main.rs"
        pane command="cargo" {
            args "test"
        }
    }
}
```

换成 JSON 后，无论是可读性还是编写体验都会遭受灾难性破坏。

**结论**：当你设计的系统/框架涉及 CLI 工具、结构化指令传递、或需让 AI Agent 高频生成与读取工具调用（Skill Call）时，KDL 在该领域的生态位与表达力远超 JSON——既让后台代码解析纯粹，也让调试日志与用户配置清晰如艺术品。

---

## 八、"内 JSON、外 KDL"：双轨制落地

### 现实约束：LLM 的 Function Calling 已被 JSON 硬编码

目前全球所有主流大模型（OpenAI、Anthropic、阿里、深度求索等）的 Function Calling 在 API 层面已被硬编码为 JSON 格式（返回 `arguments: "{\n \"param\": 1\n}"` 这样的 JSON 字符串）。若强行逼大模型通过纯文本 Prompt 盲写 KDL，不只丧失官方 Function Calling 的原生强约束，还会导致幻觉率飙升。

但这绝不意味着 Agent Skill 场景就"没办法"用 KDL。真实工程落地中，业界普遍采用**"内 JSON、外 KDL"双轨制**——KDL 与 JSON 不是竞争关系，而是互补的上下游搭档。

### 解法一：外置声明用 KDL，运行时转化给 AI（最成熟、最推荐）

当 Skill 由人类开发者编写与维护（如 `config.kdl` 中的 auth、skills、server 等配置）时，坚持用 KDL 作为用户/开发者唯一交互接口。大模型只认 JSON，但 Python/Rust 后台是灵活的：

1. **人类编写**：开发者写一份清晰的 `skills.kdl`
2. **系统加载**：Python 后台启动时用 `kdl-py` 读取，自动转为内存中的 JSON Schema / Pydantic 模型
3. **喂给大模型**：后台将转好的 JSON Schema 传给 OpenAI/Qwen 的 `tools` 参数，发动官方 Function Calling
4. **模型输出**：模型输出它最擅长的标准 JSON 调用结果
5. **系统解析执行**：后台拿到 JSON，执行技能、记录日志

**核心价值**：把 KDL 的高可读性留给人类，把 JSON 的标准化留给机器——**完全不需要依赖 AI 具备生成 KDL 的能力**。

### 解法二：利用 System Prompt 进行"无损 KDL 语法中转"

当工具的最终产物必须是 KDL 文件或指令（如类 Zellij 的平铺窗口管理器、CLI 脚本生成器），大模型必须输出 KDL 时，利用"模型输出 JSON 键值对 → 后台拦截器 → 无损组装 KDL"的模式。

Function Calling 本质上只能传键值对。利用 KDL 的底层数据模型，在 JSON 里设计一个"通用 KDL 节点描述符"Schema——向模型声明一个 `execute_kdl_node` 函数：

```json
{
  "name": "execute_kdl_node",
  "description": "生成或调用一个标准的 KDL 结构化节点指令",
  "parameters": {
    "type": "object",
    "properties": {
      "node_name": { "type": "string", "description": "KDL 节点/命令名称" },
      "args": { "type": "array", "items": { "type": "string" }, "description": "同行的纯值参数" },
      "props": { "type": "object", "description": "同行的键值对属性" }
    },
    "required": ["node_name"]
  }
}
```

模型调用 CLI 技能或生成代码时，输出它最擅长的 JSON：

```json
// AI 官方 Function Calling 输出的 JSON
{
  "node_name": "video_clip",
  "args": ["input.mp4", "output.mp4"],
  "props": { "start": "00:01:00", "duration": 30 }
}
```

后台拿到后，因 JSON 结构和 KDL 的 args/props 模型 100% 对应，两行代码即可无损组装成标准 KDL 指令：

```python
kdl_line = f"{data['node_name']} "
kdl_line += " ".join([f'"{a}"' for a in data.get('args', [])]) + " "
kdl_line += " ".join([f'{k}="{v}"' for k, v in data.get('props', {}).items()])

print(kdl_line)  # video_clip "input.mp4" "output.mp4" start="00:01:00" duration="30"
```

### 选型路线总结

完全不需要为"大模型 API 只支持 JSON"绝望，工程中可以这样分工：

1. **配置管理 / 技能声明 / 路由表（开发者侧）**：坚决用 KDL，干掉团队维护复杂 Agent 框架时被 TOML 嵌套折磨、被 YAML 缩进坑害的痛点
2. **大模型输入输出通信（AI 网络侧）**：坚决用官方 JSON Function Calling，顺应生态基础设施，确保高稳定性
3. **两者之间**：用 Python/Rust 代码做一层自动化的 Schema 翻译

---

## 九、用 KDL 重写块级配置：以 Nginx 为例

Nginx 的配置是独特的块级（Block）语法——看起来也是「节点 + 花括号」的树状结构，但它最大的痛点在于**没有真正的语法数据模型（Data Model）**：除了花括号，所有配置本质上都是"一堆空格分隔的字符串"。用 KDL（v2 标准）重写 Nginx 配置，层级结构虽然契合，但书写严谨性、数据类型的防错能力会发生质变——这是从「字符串拼接游戏」向「强类型语法树」的降维打击。

### Nginx 原生 vs KDL 直观对比

```nginx
# 原生 Nginx 配置（纯字符串堆砌，没有布尔或数字类型）
user nginx;
worker_processes auto;

http {
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        listen 443 ssl;
        server_name ://xinminghui.com;

        root /var/www/html;
        index index.html index.htm;

        location /api/ {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            add_header 'Access-Control-Allow-Origin' '*';
        }
    }
}
```

```kdl
// 对应 KDL 配置（节点名 + 位置参数）
user "nginx"
worker_processes "auto"

http {
    // 1. 强类型加持：不再是 "on"，而是真正的布尔值和数字
    sendfile #true
    keepalive_timeout 65

    server {
        // 2. 善用 KDL 属性：将端口、SSL 状态、域名优雅归类
        listen 80
        listen 443 ssl=#true
        server_name "://xinminghui.com"

        root "/var/www/html"

        // 3. 数组优雅表达：不需要在一行里用空格拼字符串
        index "index.html" "index.htm"

        location "/api/" {
            proxy_pass "http://127.0.0.1:8000"

            // 4. 属性化的 Header 映射，比原生更具"结构体"质感
            proxy_set_header Host="$host" X-Real-IP="$remote_addr"

            add_header "Access-Control-Allow-Origin" "*"
        }
    }
}
```

### 四大核心变化

**变化一：拥有真正的类型系统（Type Safety）**

Nginx 的所有数值与开关都是字符串（如 `on`/`off`、`65`/`65s`），不看文档就无法得知某指令接收什么格式。KDL 让 `sendfile #true` 是布尔、`keepalive_timeout 65` 是整数——现代 IDE、LSP 插件、Python/Rust 解析器可进行精准静态语法检查，配置写错在编译期或启动前即可被拦截。

**变化二：消除 Nginx 指令的分号 `;` 尾缀**

新手写 Nginx 配置最常犯的错误是漏掉行尾分号导致整段解析错乱。KDL 中换行即代表节点结束，分号成为可选的（仅在单行写多个节点时需用），彻底根治漏写分号的低级错误。

**变化三：高级指令的"属性化（Properties）"重构**

Nginx 表达复杂关系时指令会变得很长（如 `listen 443 ssl http2 default_server;` 在一个指令后塞了一堆混杂 Flag）。KDL 用现代具名属性收纳：`listen 443 ssl=#true http2=#true default_server=#true`。这种键值对结构传给后台可被优雅反序列化为 Python 结构体或类。

**变化四：字符串必须带双引号（引号强制化）**

Nginx 的路径和域名通常不需要加引号，但一旦含空格或特殊字符就必须加，否则崩溃，心智负担重。KDL 遵循现代编程语言规范，所有非标识符字符串（路径、域名、正则）必须用 `""` 包裹，从根本上杜绝因特殊字符、空格导致的解析割裂。

### 结论：适配场景

若打算写一个"现代化的高性能网关 / 负载均衡器"（如基于 Rust 的 Pingora 或 Go 新型网关），用 KDL 替代传统 Nginx 语法作为配置文件是绝佳决策——既继承 Nginx 花括号对人类友好的视觉层级，又把纯字符串拼接的语意升级为可用 Pydantic/Serde 强类型校验的现代软件工程标准。

---

## 选型判断

- **选 JSON**：标准的 Web API、前后端传输、或纯粹的对象 dump（如从数据库导出的单条 user 记录）
- **选 TOML**：扁平键值为主、生态依赖成熟的场景（Cargo、Python 广泛采用），嵌套深度有限
- **选 YAML**：已被 Kubernetes/GitHub Actions 等运维与 CI/CD 生态绑定，但需承受缩进与类型猜测风险
- **选 KDL**：由人类手写或维护的复杂树状文件——流水线声明（CI/CD）、UI 组件网格布局、微型规则引擎配置、含复杂元数据的方法调用配置