# KDL 与配置/序列化格式全景对比

**状态**：分析与选型参考
**日期**：2026-08-13
**核心观点**：KDL 的节点模式是有主次的「语义树」，映射类似 HTML/XML 或命令行调用的逻辑树；而 TOML 是扁平键值对，JSON/YAML 是嵌套数据网络。这是与所有主流程配置格式分道扬镳的哲学底色。

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

**作为对照**：JSON 中一切都是数据——键值对、数组元素之间没有"主次之分"，表达同样的语义必须通过人为约定的特殊键名（如 `_type`、`__meta__`）打补丁。

```json
{
  "type": "button",
  "id": "submit-btn",
  "children": [ "提交" ]
}
```

**三格式底层对比**：抛开生态与历史兼容的包袱，将 KDL 作为纯粹的"现代数据交互格式"考察时，它是唯一一个把**语义控制权**内嵌在语法骨子里、同时具备**代码级防错性**的树形格式。

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

## 一、主流格式战略权衡

### 主流格式对比总表

| 格式 | 语法哲学 | 核心痛点 | 传输抗损 | 体积/语义 | 生态位 |
|:--|:--|:--|:--|:--|:--|
| **KDL** | 有主次的语义树（节点名+args+props+children） | 节点名语法上不可省略，纯匿名数据被迫多写一层壳 | 极高。层级靠 `{}` 锁死，缩进不敏感，流式友好（一行一节点，类 JSONL） | 极高密度。混合语义表达力，连 minified JSON 都压得过，可用 `;` 压缩 | 新势力（2024 后成标准），生态需自建 |
| **TOML** | 基于节（Section）的扁平 `键 = 值` | 深度嵌套时表头冗长（如 `[a.b.c.d]`） | 中。不依赖缩进与花括号，但节结构靠换行 | 扁平，层级浅就很好用 | 极成熟（Cargo、Python 标准选择），大众熟悉 |
| **YAML** | 依赖缩进的数据网/图 | 缩进地狱；过度猜测类型（挪威问题） | 极差。缩进敏感，空格丢失/合并冲突即破坏结构 | 极紧凑（无括号逗号），但不可压缩 | 运维与 CI/CD 绝对主流（Kubernetes、GitHub Actions） |
| **JSON** | 机器友好的标准数据模型 | 满屏逗号、冒号和双引号；无混合语义须 `type` tag | 极高。层级靠 `{} []` 锁死 | 嵌套数据网络，语言级原生映射 | 全语言默认标准 |
| **HTML** | 标记语言（Markup） | 过于冗长，强制闭合标签导致体积虚胖 | 高。标签树结构 | 密度低但节点树模型完美 | 浏览器唯一渲染骨架 |

> 解析器复杂度是 KDL 的隐藏优势：规范仅几页纸，各语言解析行为高度一致；YAML 规范超 80 页，易因缩进/特殊符号跨语言解析报错。

> **设计哲学佐证**：KDL 拥抱花括号 `{}` 不是向 C 家族审美妥协，而是与现代系统语言（Rust、Zig、Swift、Go、Carbon）一致的工程推理——显式的可见边界对传输、合并、复制粘贴中的缩进错乱免疫（详见 [modern-language-design.md](modern-language-design.md) 的「语法边界：花括号 vs 缩进 vs `end`」）。这也顺带解释了 TOML 的生态位：TOML 的扁平 `键 = 值` 天然适合**层级不深的场景**（Cargo、pyproject 等顶层配置），一旦要求深层树状嵌套，表头就会冗长到 `[a.b.c.d]` 的灾难——它牺牲树的表达能力，换来的正是「不依赖缩进、也不依赖花括号」的扁平稳健。

### 选型判断

**第一性判别维度：配置元素是否需要「语义身份」标识。**

要不要选 KDL，就看所表达的结构里，每个对象是否天然"知道自己是哪种东西"：

- **XML 式节点树配置** → **必选 KDL**。node 天然是「身份 + 容器」，与标记语言的树模型同构，无需任何判别字段。
- **没有匿名节点的数据**（每个条目都必须标注"它是谁"，否则语义不清）→ **选 KDL**。节点名就是身份，内建免费。
- **其余**（纯匿名数据 / 表 dump / 扁平 map，元素身份无关紧要）→ JSON/YAML/TOML。此时节点名反而是必须存在的负担（节点名语法上不可省略），KDL 被迫多写一层壳。

当结构里不存在"身份"需求时，KDL 的上述减负优势就没有用武之地，反而变成多余的命名负担。**判断维度单一，即看身份是否内生于结构。**

**速记**：`有 type + 树形 → KDL`。严格讲，**type/语义身份是判据**，树形是 KDL 的底层模型（有向树）——是它必然的结构底色，不与 type 并列。

按此判据映射到常见场景：

- **选 JSON**：标准的 Web API、前后端传输、或纯粹的对象 dump（如从数据库导出的单条 user 记录）——元素身份由外层 schema 界定，无内置类型概念
- **选 TOML**：扁平键值为主、生态依赖成熟的场景（Cargo、Python 广泛采用），嵌套深度有限
- **选 YAML**：已被 Kubernetes/GitHub Actions 等运维与 CI/CD 生态绑定，但需承受缩进与类型猜测风险
- **选 KDL**：由人类手写或维护的复杂树状文件，或天然带身份标签的数据——流水线声明（CI/CD）、UI 组件网格布局、微型规则引擎配置、含复杂元数据的方法调用配置、以及任何本需手工 `type` tag 的异构对象集合

### type tag 代价：KDL 节点名 vs JSON 的判别字段

KDL 节点名与 JSON 的 `type` 判别字段承担**同一个语义角色**——告诉消费者"我是哪种东西"——但 KDL 把它内建于语法，JSON 要靠手写。这层代价不只是"多打几个字符"，而是横跨书写端到消费端的六条，可分成两组：

**书写端（写入时已付出）**

1. **多占空间**——每个元素重复 `"type":"xxx"` 一行。
2. **命名冲突**——`type` 常撞保留字或既有的业务/数据字段，被迫改名 `kind`/`_type` 或做 Serde rename。
3. **tag 必须稳定位于第一位但字典无序**——判别字段依赖位置，一旦自动格式化/重排即破坏，易错。

**消费端 / 维护端（即使前三条全避开，代价仍在）**

4. **需要一个「tag→schema」的显式映射层**——`type` 字段不会自动让对应结构正确还原。消费端必须额外配置判别机制：serde 的 `#[serde(tag="type")]`、pydantic 的 `discriminator`、动态语言里手写 switch。KDL 的节点名本身就是 schema 锚点，**直接**驱动分发，无此映射概念。这是结构性差异——映射层是 JSON 正确加载的**先决存在**，不是可选项。
5. **tag 与内容之间没有「同生共死」约束，错配到运行期才暴露**——字典允许 `{"type":"circle","radius":2}` 的内容被误改/复制成 `{"type":"circle","width":2}`，没有任何机制强制 tag 与其余字段一致；type 拼写（`Circle`/`circle`/`circ`）也只在消费端运行时 switch 才炸。KDL 里节点名与它承载的 props/子节点是同一 token 序列的一体，改名必须整体搬运，错名在语法/校验期即被识别。
6. **单一判别字段撑不起多维 / 层级分类**——type tag 是扁平的单维度。对象想同时按"形状"与"优先级"分类，或类型需要嵌套（android 里的 button 与 web 里的 button 不同）时，JSON 只能选一个作判别字段或手工设计嵌套结构。KDL 节点名是单一主身份，但可靠嵌套子节点表达层级——分类天然有深度。

前三条是书写端痛点，后三条是消费端/维护端痛点。**"写出来容易"不等于"读回来容易"**——选型判据必须同时覆盖两端，只看任何一端都会高估 JSON 的适用面，判断维度见前一节的 [选型判断](#选型判断)。

### 压缩性 vs 紧凑性：删空格不行，但配置场景连 minified JSON 都压得过

「数据密度高」要拆成两个不同维度，否则会误读：

- **结构性紧凑**（KDL 的主战场）：靠节点名 + 位置参数 + 裸键属性省字符。JSON 的键**必须**带双引号（`"key"`——语法强制而非风格），每个键值还带一个冒号 `:` 和外层 `{ }`;KDL 的 props 键是**裸标识符**、用 `=` 分隔、扁平并列时无外层包裹。结果：**键值配置场景下，KDL 连压缩到零空白的 JSON 都压得住**：

```json
{"host":"localhost","port":8080}   // 35 字符
```

```kdl
server host="localhost" port=8080  // 32 字符，还带着语义节点名
```

- **可压缩性**：KDL **删空格不行**——token 之间靠空格分隔，删掉即粘连成一个词（`user 001` → `user001`，语法破）;但**删换行可以**——换行只是「节点终止」的触发信号，可用分号 `;` 等价替换，得到合法紧凑形式：

```kdl
user 001 active=#true;user 002 active=#false;
```

TOML/YAML **结构性依赖换行**（节、数组表、缩进无法压缩），KDL 在这一维度完胜;唯一输给 JSON 的是「纯无键名裸值数组 dump」（如 `[1,2,3]`）这类非配置形态——JSON 删到零空白仍更短。但对配置而言，这是一个无关边缘。

**结论**：KDL 的「数据密度高」是**双层**的——结构性省符号 + `;` 可压缩;在配置主战场，它连 minified JSON 都压得过。唯一需保留 token 间空格的代价，是这不构成对 JSON 的体积劣势（引号开销远大于空格开销）。

### 传输鲁棒性：类型固化

数据交换中最怕"数据类型在传输过程中变异"。KDL 的字面量设计严格，且拥有原生强制的类型标签（Type Tags），可直接在数据中焊死类型：

```kdl
transaction amount=(f64)100.0 timestamp=(datetime)"2026-08-13T10:00:00Z"
```

不管接收端用 Python、Go 还是 Rust，解析器看到 `(f64)` 与 `(datetime)` 都绝不会误认成整数或纯字符串。

- **对照的弱者**：JSON 只有基础 String/Number/Boolean，遇日期/UUID/Base64 只能前后端约定"用字符串传"，机器无法在语法层做强校验；YAML 则过度猜测类型（挪威问题：国家代码 `NO` 被识别为布尔 `false`）——这两者的类型困境在总表「传输抗损」列已概述，此处聚焦 KDL 的解法。

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

### 极端场景：适用与不适用

#### 适用：结构化指令 / 关系 / 图表类数据

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

此类数据以 KDL 传输时，可读性、紧凑度与安全性均表现优异。

#### 不适用：纯巨量对象矩阵（Data Dump）

当数据是纯粹从数据库倒出来的、包含十万条整齐划一用户信息的矩阵（大批量用户列表）：

- **JSON 最直接**：`[{"id":1, "name":"a"}, {"id":2, "name":"b"}]`，代码里直接转为 `List[User]`
- **KDL 的取舍**：用位置参数 `user 1 "a"` 还是具名属性 `user id=1 name="a"`。因模型比纯 JSON 复杂，大规模"纯表格式数据 dump"时，解析成高级语言对象的内存开销和转换步骤会略多


### 工具链：nushell 内置 KDL

流行的独立 KDL 命令行工具 `kq` 已停滞（4-5 年未更新），无法完全兼容 2024 年底发布的 KDL v2 标准。但 **nushell 0.114.0（2026-06）起内置 `from kdl` / `to kdl`**，消除了日常使用最大的障碍：

```nu
"node attr=1 {bloc}" | from kdl
```

输出为正则节点行（`name` / `args` / `props` / `children`），可直接进入 nushell 的管道生态做过滤和处理——无需为了工具链而引入 `/bin` 级的外部依赖。`open` 对 `.kdl` 后缀也自动触发解析。详见 [Nushell 介绍](nushell-introduction.md) 的内置格式处理一节。只有需要与 `jq`（而非 nushell）协作时，才用 `ckdl`/`kdljs` 将 KDL 转为 JSON 再输入 `jq`。

## 二、KDL → dict/struct 转换约定

**先看真实实现**（Python/ckdl 的节点→值转换逻辑，代码可直接说明这套约定如何落地）：

```python
def _kdl_node_value(node) -> Any:
    """把单个 KDL 节点转换成其"值"（dict / 标量 / list）。
    - 有 children：`{child_name: child_value}`；同名兄弟合并为 list。
    - 有 props：并入同一 dict（props 是 this 节点的直接键值元数据）。
    - 无 props 无 children：单 arg → 标量，多 arg → list，无 arg → None。
    """
    out: dict[str, Any] = {}
    multi: dict[str, list[Any]] = {}
    for child in node.children:
        multi.setdefault(child.name, []).append(_kdl_node_value(child))
    for name, vals in multi.items():
        out[name] = vals if len(vals) > 1 else vals[0]
    out.update(dict(node.properties))

    args = list(node.args)
    if out:  # 有 props/children → dict 结构；剩余 args 附为 _args
        if args:
            out["_args"] = args if len(args) > 1 else args[0]
        return out
    # 纯 args 节点：单值 / 列表 / 空
    if len(args) == 1:
        return args[0]
    if args:
        return args
    return None
```

**基础转换的 schema-aware 列表修正**：`_kdl_node_value` 忠实按 KDL 语法折叠——`path "skills"` 单 arg 折叠成标量 `"skills"`。若消费端字段声明为 `list[X]`（如 `path: list[str]`），这个阶段要把这类单值标量**按字段标注归一成单元素列表**。以下 `_coerce_known_lists` 递归遍历结果，用 Pydantic 字段标注推导应作为列表的路径（放在加载后、交给 pydantic 校验前）：

```python
def _coerce_known_lists(data: Any, field: Any) -> Any:
    from typing import get_origin, get_args

    def _is_list(ann) -> bool:
        # 容忍 Optional / Union / Annotated / types.UnionType（X | None）
        if ann is None:
            return False
        origin = get_origin(ann)
        if origin is list:
            return True
        if origin is Annotated and get_args(ann):
            return _is_list(get_args(ann)[0])
        if origin is Union or origin is UnionType:
            return any(_is_list(a) for a in get_args(ann) if a is not type(None))
        return False

    # 先判「字段声明为 list」再判数据类型——否则单 dict 会先走 dict 分支，
    # 被当普通对象展开，永远到不了列表归一。
    if _is_list(getattr(field, "annotation", None)):
        if isinstance(data, list):
            # 已列表：取 list[X] 的元素类型 X 递归每个元素，避免二次包裹
            elem_field = _FieldLike(get_args(getattr(field, "annotation", None))[0])
            return [_coerce_known_lists(v, elem_field) for v in data]
        return [data]  # 单值（标量或单 dict）→ 包成单元素列表

    if isinstance(data, dict):
        # model 类直接取 model_fields；否则从标注下钻（含 dict[str,T] 用 T）
        if isinstance(field, type) and issubclass(field, BaseModel):
            child, fields = field, field.model_fields
        else:
            child, fields = _child_model(field), _child_model(field).model_fields
        # 键值 map（dict[str,Model]）时键不一定是字段名：命中字段用字段，否则用 child 统一处理
        return {
            key: (
                _coerce_known_lists(v, fields[key])
                if (fields and key in fields)
                else _coerce_known_lists(v, child)
            )
            for key, v in data.items()
        }
    return data
```

要点：
- `_is_list` 必须同时识别 `typing.Union` 与 `types.UnionType`——Python 3.10+ 的 `X | None` 用后者，只判 `typing.Union` 会漏掉 `path: list[str] | None` 这类字段（实测把 `extractors` 误判为非列表）。
- **分支顺序关键**：先判断「字段声明为 list」再判断数据类型，否则单 `dict` 值走 dict 分支被当对象展开，无法归一。
- `_child_model`/`_FieldLike` 是递归下钻子模型（剥 Annotated/Union、`dict[str,T]` 取 `T`）的辅助，这里省略其定义。

把 KDL 转成嵌套 dict/struct 供配置框架（pydantic-settings、serde 等）消费时，遵循如下约定：

- **节点名 = 字段键**：顶层节点名即顶层字段键，节点值由其内容决定。
- **单 arg 子节点自动并入父 kv**——无 props/children 的子节点，其单参数**直接作为 `{子节点名: 标量}` 并入父 dict**：`path "skills"` → `{"path": "skills"}`；多 arg 则是 `{子节点名: [a, b]}` 列表。props（元数据）与子节点（值/结构）统一归入同一 dict，混合也成立：`server host="0.0.0.0" { port 8080 }` → `{"host": ..., "port": 8080}`。
- **列表的三种表达**（随数据形态选）：

  | 写法 | 语义 |
  |:--|:--|
  | `path "a"` | `{"path": "a"}` 标量 |
  | `path "a" "b" "c"` | `{"path": ["a","b","c"]}` **扁平列表（推荐）** |
  | 换行/`;` 分隔重复节点 `path "a"` `path "b"` | `{"path": ["a","b"]}` 扁平 |
  | 多 arg 重复节点 `path "a" "b"` + `path "c" "d"` | `{"path": [["a","b"],["c","d"]]}` **嵌套列表** |

- **列表 = 重复同名节点**：同名子节点连续出现多次（如 `extractors type="header"` ×3）→ `[{...}, {...}]`，与 TOML 的 `[[key]]` 数组表语义一致；鉴别的 `type` 判别联合在此自然成立（各元素类型按其 `type` 分发到对应子类）。
- **同行吸收陷阱**：`path "a" path "b"` 写在同一行**不产生两个子节点**——`path` 被前一个节点吸收为第二个 arg（→ `["a","path","b"]`）。重复节点务必用换行或 `;` 分隔。**多值放 args 与重复节点不要混用**（否则得到嵌套列表而非扁平）。
- 含 `{` `}` 等特殊字符的字符串值改用**位置参数**（而非 props），避免花括号被误判为子节点块起始。

这条约定是「KDL 扁平比 JSON 更紧凑」在配置反方向的关键闭合：KDL 把"列表"表达为重复节点、把"值"压入单节点位置参数，正好映射回 dict 的 kv 结构，无需额外 type tag。

> **这不是 Python 局部的约定，而是跨生态统一语义**：Rust 生态的 `kdl-rs`（KDL 官方序列化库）`serde` 反序列化遵循**完全一致**的七条规则——顶层节点名=map key、单 arg 节点→标量、多 arg→sequence、有 props→map、有 children→map(子名=key)、props+children 合并单 map、重复节点名→sequence。Python（pydantic-settings + ckdl）与 Rust（serde + kdl-rs）就对同一套 KDL 反序列化语义达成一致，这正是"KDL 节点名 = type tag"优势的落地证明：它把"身份"内建于语法，两端都无需额外判别映射层。

### 谁补充「列表还是标量」的类型信息

因为 KDL 语法不改写"单数还是复数"，消费端**必须有类型来源**才能把单值节点归成单元素列表。三种生态差异巨大：

| 生态 | 类型信息来源 | 单值 → 列表的处理 |
|:--|:--|:--|
| **Rust serde**（kdl-rs） | 结构体字段声明 `Vec<T>`（编译期） | ✅ **原生解决**，无需额外工作 |
| **Python pydantic-settings** | 字段声明 `list[T]`（运行时反射） | ✅ 需反射归一（skillforge 的 `_coerce_known_lists`） |
| **Nushell**（`from kdl`） | ❌ 无类型系统，表格模型 | ⚠️ 必须显式 `--list-fields` 点出列表字段 |

- **serde 是类型驱动的反序列化**：`deserialize_any`（无约束）才会把单 arg 当标量；当目标字段是 `Vec<String>`，serde 强制走 `deserialize_seq`，KDL 解析器即使看到单个 `"skills"` 也作为序列的单元素返回。因此 Rust 侧**单值列表问题不存在**——目标类型自动决定。
- **Python 需反射节点定义**：pydantic-settings 知道 `path: list[str]`，加载时用字段标注把单值包成列表（skillforge `_coerce_known_lists` 递归下钻子模型，含 `dict[str,T]`、`X|None`、`Annotated`、`types.UnionType`）。
- **Nushell 是无类型的**：`from kdl` 只给 `[[name,args,props,children]]` 表格，函数面不知道 `skills.path` 该是列表。必须用 `--list-fields ["skills.path" "test.messages"]` 显式声明，否则单值节点退化为标量。**这是 nushell 的结构性限制（无类型系统），不是实现可绕过的**——除非像 `kdl to-record --list-fields` 那样把列表字段写死。

结论：**配置文件的权威解析方应是有类型的生态（Rust/Python）**，nushell 只作辅助查看，接受手动声明列表字段的成本。

## 三、KDL 风格指南

### 行长度纪律：用子节点下沉，不用硬性计数

KDL 单行高密度是核心优势，但一个节点（节点名 + 参数 + 属性）承载语义过重时，一行会过长、难以阅读。正确做法**不是**设「节点名、属性最多 ≤3 个」这类硬性数字上限——数字是错误信号：行长的真正决定因素是单位语义的字符密度，两个短标量参数的视觉负担与两个长字符串参数差一个数量级；且硬性砍个数会对抗格式自身的单行表达力。规则应引导使用 KDL 内置的逃生机制——**子节点块（`{}`）下沉**：

> **行长度纪律**：当一行（节点名 + 参数 + 属性）感官过重或超过约 100 字符时，将该节点的*修饰性*维度下沉到 `{}` 子节点块——保留主内容在 args（通常 ≤2–3 个），把标签组、元数据属性、长文本移到子节点或 props。下沉的是**次要维度**，不是机械地砍个数。

上文 song 示例（[什么是 KDL](#什么是-kdl) 的结构表达 纵向排版）即为正例：

```kdl
song "Hotel California" {
    meta rating=5.0 released=1976
    genres "Rock" "Classic Rock"
}
```

边界权衡：args（主内容）保持少，props（元数据）可适度放宽。判断标准是「这行读起来是否顺畅」，不是「数了几个」。

### 优先子节点原则（主动版）

行长度纪律是*被动*触发（变长了才下沉）；对**配置/模板这类人写的多字段结构**，反过来应*主动*默认用子节点、单行只留核心短 kv：

```kdl
// ✅ 主动版：单行只留核心短 kv，其余全部下沉子节点
engine type="agno" {
    data_dir ".hermes_data"
    user_subdir true
}

// ❌ 被动版（等价但单行看不出结构）：info/字段全挤一行
engine type="agno" data_dir=".hermes_data" user_subdir=true
```

判断标准是字段的**语义层级**而非字符数：真正该留在单行的是「标识/主体」这一级（type 定引擎种类、name 定身份、model 定模型），被下沉的是「修饰/状态/扩展示例」级（路径、开关、端口、格式串）。字段重要性差不多时，**默认全当子节点**——单行无法承载多个同级结构，一旦字段一多，一行挤几个 kv 的结构是读不出来的。

> 平衡点：核心主键留单行（1–2 个短 kv），其余一律 `{}` 子节点。proportional vs 单元计数——先看这行读不读得出结构，再谈字数。

## 四、关键概念澄清与工程避坑

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

## 五、CLI / 结构化指令 / Agent Skill 场景的维度级优势

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

## 六、"内 JSON、外 KDL"：双轨制落地

### 现实约束：LLM 的 Function Calling 已被 JSON 硬编码

目前全球所有主流大模型（OpenAI、Anthropic、阿里、深度求索等）的 Function Calling 在 API 层面已被硬编码为 JSON 格式（返回 `arguments: "{\n \"param\": 1\n}"` 这样的 JSON 字符串）。若强行逼大模型通过纯文本 Prompt 盲写 KDL，不只丧失官方 Function Calling 的原生强约束，还会导致幻觉率飙升。

但这绝不意味着 Agent Skill 场景就"没办法"用 KDL。真实工程落地中，业界普遍采用**"内 JSON、外 KDL"双轨制**——KDL 与 JSON 不是竞争关系，而是互补的上下游搭档。

### 解法一：外置声明用 KDL，运行时转化给 AI（最成熟、最推荐）

当 Skill 由人类开发者编写与维护（如 `config.kdl` 中的 auth、skills、server 等配置）时，坚持用 KDL 作为用户/开发者唯一交互接口。大模型只认 JSON，但 Python/Rust 后台是灵活的：

1. **人类编写**：开发者写一份清晰的 `skills.kdl`
2. **系统加载**：Python 后台启动时用 `ckdl` 读取，自动转为内存中的 JSON Schema / Pydantic 模型
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

### 解法三：纯 Python 配置加载——KDL → Pydantic 薄桥接

上面两个解法都围绕「AI 网络侧」的 JSON 硬编码。若你的场景是**纯 Python 后台加载本地配置文件**（无 AI 通信），则无需任何中转——但要做好生态的现实预期：

**生态现状**：截至 2026 年中，PyPI 上**没有** `pydantic-settings` 之于 KDL 的等价桥接库，但有成熟的 KDL 解析器 **`ckdl`**（tjol/ckdl，C11 实现 + Python 绑定，完整支持 KDL 1.0 & 2.0.0）。配置加载层是空白，需自家粘一层。好在这一层很薄——`pydantic-settings` 本就把「读取来源 → 校验进模型」抽象为 `SettingsSource`，自定义一个 KDL 来源即可，依赖仅 `pydantic-settings` + `ckdl`：

```python
class Settings(BaseSettings):
    host: str
    port: int
    model_config = SettingsConfigDict(settings_source=None)  # 只留自定义源

def kdl_source(settings, **kwargs):
    return ckdl.parse(kdl_str)  # ckdl 解析 → 节点树
```

更简者可直接把解析结果转成 dict 后 `Model.model_validate(d)`，不走 pydantic-settings、不声明额外来源。

**为什么选 `ckdl`**（实测于 2026-08，KDL v2 语法 `enable=#true port=8080`）：完整支持 KDL 2.0.0，`#true` 正确解析为布尔 `True`，数字直接解析为 **int**（`port=8080` → `8080`，而非 float）——可直接命中 Pydantic 的 `int` 字段，免去一层值归一。API 为节点树（`Node.args`/`Node.properties`），底层是 C11 流式 SAX 解析器，性能与准确性俱佳。

### 路线总结

完全不需要为"大模型 API 只支持 JSON"绝望，工程中可以这样分工：

1. **配置管理 / 技能声明 / 路由表（开发者侧）**：坚决用 KDL，干掉团队维护复杂 Agent 框架时被 TOML 嵌套折磨、被 YAML 缩进坑害的痛点
2. **大模型输入输出通信（AI 网络侧）**：坚决用官方 JSON Function Calling，顺应生态基础设施，确保高稳定性
3. **两者之间**：用 Python/Rust 代码做一层自动化的 Schema 翻译

## 七、用 KDL 重写块级配置：以 Nginx 为例

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

## 八、KDL 与嵌入式脚本配置语言（Lua / Scheme·Steel）

以上各节的对照（KDL vs JSON/YAML/TOML）都发生在**静态数据格式**这一轴内。配置还常遇到另一种需求——**运行期可编程逻辑**（条件、循环、动态求值），此时引入的是图灵完备的**嵌入式脚本配置语言**：Lua（Neovim、Hyprland、各类游戏引擎的默认嵌入脚本）与 Scheme（Rust 生态的 Steel）。这条轴与静态格式正交：静态格式在**加载期**即定死结构，脚本语言把一部分决策推迟到**运行期**。

> **选型判据分开看**：本文 §[选型判断](#选型判断) 回答「要不要语义身份」（静态格式内部的选择）；本节回答「要不要在运行期算配置」（静态数据 vs 可编程脚本）。两条轴独立。多数配置落在「静态 + 语义身份」象限（选 KDL）；只有确需动态逻辑时才进入本节的脚本语言竞争。

### 三者的本质定位

- **KDL —— 纯静态数据语言**。不是编程语言，不能写逻辑，只负责把树状结构表达得干净。解析结果与宿主语言对象在加载期一一映射，之后退出。安全性最高（无死循环、无副作用），也因此「无法扩展」——一切语义交给主体程序解析。
- **Scheme / Steel —— 函数式脚本，代码即数据**。用极少的规则（S-表达式）实现「代码即数据」；宏让配置期获得**无限的元编程能力**——可定义变量、函数甚至语法糖。Steel 是嵌入 Rust 的 Scheme 实现，把「配置即程序」带进 Rust 生态。
- **Lua —— 命令式动态脚本（通用胶水）**。特性刻意极简，为降低引擎实现难度、让非专业用户（如游戏策划）快速上手；由此本应由语言解决的问题被外包给使用者。

### 分维度对比

- **结构表达**
  - KDL：节点名 + 位置参数 + 具名属性 + 子节点块，一行承载完整树语义。
  - Scheme：S-表达式天然可表达任意树，代价是每层嵌套多一对圆括号。
  - Lua：靠函数调用嵌套或手写字面量表，繁琐且身份不内建。
- **心智负担**
  - KDL 极低：格式严谨、语义可见、无副作用。
  - Scheme 中：仅多一层圆括号的视觉成本。
  - Lua 高：「伪简单」——语法门槛低，但底层细节反直觉，资深用户被 1-based 与手写元表反复绊倒。
- **索引习惯**
  - KDL：无索引概念。
  - Scheme：列表是 car/cdr，索引不是其原生模型（向量才谈从 0 起）。
  - Lua：**1-based**。与 C/C++/Python/Rust 的 0-based 肌肉记忆相悖，在底层 C 层交互时伴随坐标转换。
- **扩展能力**
  - KDL：无——不可自定义语法与行为，全靠宿主解析。
  - Scheme：无敌的宏系统，可自由定制语法（让 Lisp「披上 KDL 的皮囊」）。
  - Lua：无语言级 OO；class 依赖 `setmetatable` 手工搭建，各家风格不一。
- **语法风格**
  - KDL：清爽、现代，显式花括号边界。
  - Scheme：极简、高度统一（括号）。
  - Lua：冗长——`local`/`then`/`do`/`end` 关键字泛滥。`end` 与 KDL 的花括号对照见 [modern-language-design.md](modern-language-design.md) 的「语法边界：花括号 vs 缩进 vs `end`」。

### 为什么 Lua 让专业程序员感到别扭

「特性简单」不等于「写起来舒服」。Lua 把引擎实现的成本转嫁给使用者：

1. **1-based 索引**：扭转 0-based 的肌肉记忆，与 C 层 API 交互伴随坐标转换。这是**取舍而非错误**——给脚本新手更直观的语义（第一个元素就叫 1），代价是贴近底层者持续的别扭。
2. **手工业级 OO**：无 `class`/`inherit` 语法，高级组件须手写 `setmetatable`，缺乏统一规范，跨项目难以通读。
3. **无法自愈的语法臃肿**：强制 `local` 声明、`end`/`then`/`do` 关键字泛滥，配置书写缺乏声明式灵活性，视觉冗长。

这三点是**为降低引擎复杂度、服务非专业用户**付出的代价；Lua 因此成为 Neovim、Hyprland、游戏引擎的默认嵌入脚本，生态巨大是它最强的选型理由。

### 三选一

- **纯配置、绝对安全、拒绝死循环** → **KDL**（Niri、Zellij 的路线）。
- **动态配置、代码即数据、要语法掌控权** → **Scheme（Steel）**：靠宏让 Lisp 拥有 KDL 的皮囊与 Lisp 的灵魂。
- **随大流、要生态、能忍受 1-based 与冗长语法** → **Lua**（Neovim、Hyprland 的路线）。

> **工程化建议**：多数场景可两层分治——用 KDL 做主配置（静态、类型安全、人可读），只在确需运行时逻辑的局部引入脚本语言，避免让整个配置变成一门可编程语言所引入的复杂度与不确定性。

## 九、KDL 作为配置生成中间层：从「终端格式」到「可复用生成源」

前文各节把 KDL 当作单一配置文件的书写格式来考察（§一 的语义树、§五 CLI 拟合、§七 重写 Nginx）。但 KDL 还有一个更深的用法维度——**它不必然是需要被目标程序直接消费的「终端格式」，而可以充当「配置生成中间层（IR）」**：同一套 KDL 节点，经过一段转换逻辑，可产出多种不同目标格式的配置。这个能力使 KDL 从「一种格式」跃迁为「配置的源代码」。

### 根级复用：把通用结构提升到生成器

当配置存在多个同类目标（多台机器、多类服务、多种导出格式）时，KDL 允许把**共享的通用结构提升到根级节点**，让每个目标只需要披上一层自己的骨架。以 sing-box 代理配置为例：DNS、入站、路由、experimental 这些**所有场景共享**的部分，独立成根级 `config.kdl`；而订阅节点（`outbounds.*.kdl`）和规则（`rule-set.kdl`）参数化后分文件存放。生成器 `glob *.kdl | flatten` 合并，再按目标意图解释节点。

关键收益：**要生成另一类配置，不必重写通用逻辑**。比如从同一套 `route`/`dns` 根节点，只需另写一份骨架（如直连网关、纯 DNS 服务器），`singbox-conf generate` 就能复用之——通用结构声明一次，多处消费。

这与「JSON 的 schema 焊死在数据结构里」形成本质对比：JSON 的顶层结构即渲染形态，改一个目标就要改写整个映射关系；KDL 的节点是**形态无关的语义单元**，解释逻辑与结构定义分离，目标变换只改「如何解释节点」，不动「节点是什么」。

### 单复数的自由：作者端收益 vs 消费端负担

§三「澄清1」从**消费端**指出 KDL 不区分单复数，把「列表还是标量」的决定权留给代码。但同一枚硬币的另一面是**作者端收益**——正是这种语法上的不区分，让配置作者获得 JSON 给不了的灵活：

- **单值与多值并存**：`outbound direct` 与 `outbound direct rule_set=x` 可以同在，无需两套 schema。
- **多源自动叠合**：多个 `.kdl` 文件的同名节点定义，经 `flatten` 后自然合并为复数语义，订阅可拆文件却无缝叠加。
- **语义层级不锁死**：一个概念在「单节点」和「多节点」之间切换，不触发结构迁移。

这构成完整的正反两面：**作者端省去「单数还是复数」的预判成本（KDL 语法不强加），消费端补上「按需判定」的类型成本（由目标生态承担）**。两者方向相反但逻辑自洽——KDL 把决定权从「写下那一刻」推迟到「消费那一刻」，以换取作者端的表达自由。

### 抽象的代价与边界

KDL 作为生成中间层，代价是明确的：**目标程序不直接认识 KDL，必须有一层转换逻辑（`kdl-to-config`/`kdl-to-outbounds`）把节点重构成目标格式**。这层抽象既是表达力的来源（可复用于多目标），也是新增的维护点（转换逻辑本身需随目标格式演进）。因此其适用边界是：**存在多个目标或需频繁重组结构时**，中间层收益大于成本；只有单一静态目标、且结构永不变化时，直接写目标格式（JSON）反而更省。