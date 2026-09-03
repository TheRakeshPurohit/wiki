# OKM：Object-Keyspace Mapping（对象键空间映射）

**Status:** 独立组件
**关联文档：** [KV 存储引擎](kv-storage-engine.md) — 底层架构与设计模式
**关联文档：** [Aura 架构](aura-architecture.md) — 双引擎模式

---

OKM 是对标 ORM 的范式——ORM 将对象映射到关系表，OKM 将对象映射到 KV 键空间。通过自定义过程宏 `#[derive(KeyEncode)]` / `#[derive(ValueEncode)]` + 数字命名空间 ID，构建零成本抽象语义数据层：开发侧如同 ORM 般声明式，编译后退化为纯指针偏移计算。

## 代码即 DDL：KV 的开发者体验保障

SQL 的核心价值不是执行性能，而是关系模型交付的可读性、建模规整度和团队协作确定性。裸露的二进制字节 Key 会退化为面条代码——团队无法理解 Key 的排列规则。解决方案：用 Rust 类型系统替代 SQL DDL，把 Schema 正确性从运行时数据库引擎上提到编译期编译器。

### 强类型 Key 编码：Struct 即 DDL

SQL 定义表结构，Rust Struct 定义 Key 编码。编译器保证格式一致：

```rust
/// UserSessionKey 就是你的 DDL。
/// 跨整个框架强制约束 Key 的合法性。
pub struct UserSessionKey {
    pub org_id: [u8; 16],     // 固定 16 字节 UUID 二进制
    pub user_id: [u8; 16],    // 固定 16 字节 UUID 二进制
    pub session_id: [u8; 16], // 固定 16 字节 UUID 二进制
}

impl UserSessionKey {
    /// 编码器 = 物理 Schema 的序列化引擎。
    /// 所有字段固定宽度，无分隔符，无动态解析。
    /// Key 布局：sess: (5) + org (16) + user (16) + session (16) = 53 字节
    pub fn encode(&self) -> Vec<u8> {
        let mut key = Vec::with_capacity(53);
        key.extend_from_slice(b"sess:");           // 5 字节命名空间前缀
        key.extend_from_slice(&self.org_id);       // 16 字节固定宽度
        key.extend_from_slice(&self.user_id);      // 16 字节固定宽度
        key.extend_from_slice(&self.session_id);   // 16 字节固定宽度
        key
    }
}
```

任何试图写入错误格式的代码在编译期直接报错。不需要运行时校验。

> **注**：本例的 `"sess:"` 字符串前缀是**演化起点**（用于引出下文「字符串前缀的两大软肋」），不是推荐方案——正式方案是下文数字命名空间版的 `UserSessionKey`。

### 版本化 Enum 懒迁移：零停机 Schema 演进

SQL 靠 `ALTER TABLE` 做迁移，需要锁表。KV 靠版本化 Enum 做懒迁移——读取时按版本自动升级，无停机：

```rust
#[derive(Serialize, Deserialize)]
pub enum MemoryValuePayload {
    V1(AgentMemoryV1),
    V2(AgentMemoryV2),  // 新增字段：多模态、特征标签
}

// 读取时按版本匹配，老数据在内存中无感升级
match database.get(&key) {
    MemoryValuePayload::V1(old) => upgrade_v1_to_v2(old),
    MemoryValuePayload::V2(current) => current,
}
```

只有被读到的老记录才升级。未读到的继续以旧格式存储，不浪费写入带宽。这是 SQL `ALTER TABLE` 无法做到的——PG 的迁移必须遍历全表重写所有行。

### 指针契约：Edge Struct 模拟外键

SQL 的外键关系在 KV 中通过显式的正向+反向双写 Key 维护：

```rust
/// Session → Messages 的 1:N 外键关系
pub struct SessionToMessageEdge {
    pub session_id: [u8; 16],  // 固定 16 字节 UUID 二进制
    pub message_id: [u8; 16],  // 固定 16 字节 UUID 二进制
}

impl SessionToMessageEdge {
    /// 正向：从会话找消息
    /// Key 布局：edge:s2m: (9) + session (16) + message (16) = 41 字节
    pub fn forward_key(&self) -> Vec<u8> {
        let mut key = Vec::with_capacity(41);
        key.extend_from_slice(b"edge:s2m:");
        key.extend_from_slice(&self.session_id);
        key.extend_from_slice(&self.message_id);
        key
    }
    /// 反向：从消息反查会话（等价于 SQL Foreign Key 联合索引）
    /// Key 布局：edge:m2s: (9) + message (16) + session (16) = 41 字节
    pub fn reverse_key(&self) -> Vec<u8> {
        let mut key = Vec::with_capacity(41);
        key.extend_from_slice(b"edge:m2s:");
        key.extend_from_slice(&self.message_id);
        key.extend_from_slice(&self.session_id);
        key
    }
}
```

写入时同时 `put` 正向和反向 Key。删除时同时 `delete` 两侧。Edge Struct 的代码注释就是 E-R 图的文档化——比 SQL DDL 更显式，因为关系编码逻辑和 Key 生成逻辑在同一处。

### SurrealDB/Mongo 的 DDL 缺陷

无模式（Schema-less）是项目 0→1 阶段的「致幻剂」——开发速度极快，但到 1→10 的工业级阶段会成为数据脏乱差的万恶之源。

**弱约束导致应用层逻辑爆炸**。PostgreSQL 的 DDL 具备刚性物理约束——`NOT NULL`、`CHECK`、类型定义在撞击数据库外壳的瞬间就拦截非法数据。无模式数据库是「来者不拒的垃圾桶」：数字字段可以写成字符串 `"25"` 甚至数组 `[25]`。代价是所有消费端代码（微服务、Agent 工具）必须写满防御性类型判断和解析重试。数据库偷的懒，全部变成应用层的代码债务。

**查询优化器形同虚设**。PG 的优化器之所以强大，因为有刚性 DDL、确定的数据类型和索引元数据——优化器执行前就能精确计算磁盘步长和统计分布。无模式数据库不知道下一行 JSON 会冒出什么结构，查询引擎必须在运行时做大量动态类型断言和反射，CPU 算力被白白浪费在类型解析上。

**「代码即 DDL」如何反超**：Rust 强类型结构体把约束从运行时（DB 级）提到编译期（Rust 级）——脏数据在 `cargo check` 阶段就被熔断，连生成的资格都没有。同时不需要 PG 的运行时 DDL 锁开销和 SQL 字符串解析税。既消灭了无模式的脏数据风险，又白嫖了 KV 引擎的硬件响应速度。

### DDL 稳定性单元测试：硬编码 hex 字节防漂移

KV 系统最怕的风险：某次代码迭代改动了 Key 的前缀字符串（如 `"cfg:app:"` 错打成 `"config:app:"`），导致线上老数据永远读不出来。解决方案：编写自动化 Schema 偏移检测单元测试，用硬编码的历史 hex 字节锁定 Key 编码：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// 与「DDL 结构体」一致的数字命名空间版 UserSessionKey（ns=1，id 全部为数字）
    pub struct UserSessionKey {
        pub org_id: [u8; 16],     // 16 字节
        pub user_id: u64,         // 8 字节数字 ID
        pub session_id: [u8; 16], // 16 字节
    }

    impl UserSessionKey {
        pub fn encode(&self) -> Vec<u8> {
            let mut key = Vec::with_capacity(42);
            key.extend_from_slice(&1u16.to_be_bytes()); // ns=1 → [0x00, 0x01]
            key.extend_from_slice(&self.org_id);
            key.extend_from_slice(&self.user_id.to_be_bytes());
            key.extend_from_slice(&self.session_id);
            key
        }
    }

    #[test]
    fn test_key_ddl_stability_protection() {
        // 1. 构造确定性的业务测试数据
        let test_key = UserSessionKey {
            org_id: uuid::Uuid::parse_str("67e55044-10b1-426f-9247-bb680e5fe0c8").unwrap().into_bytes(),
            user_id: 88,
            session_id: [0x99; 16],
        };

        // 2. 硬编码历史版本生成的真实物理二进制（hex 16 进制）
        // 任何改动 ns 前缀、字段顺序或宽度的行为都会导致编码不匹配，CI/CD 立刻报错
        let expected = hex::decode(
            "000167e5504410b1426f9247bb680e5fe0c8000000000000005899999999999999999999999999999999"
        ).unwrap();

        assert_eq!(
            test_key.encode(), expected,
            "DDL 物理 Key 编码发生非预期漂移！将导致线上老数据失联！"
        );
    }
}
```

这段测试的物理含义：`hex::decode("000167e5...")` 是 42 字节的 ns 编码（`[0x00, 0x01]` + org UUID 16B + u64(88) 8B + session `0x99`×16B）。任何人改动 `UserSessionKey` 的 `encode()` 方法（改 ns、调字段顺序、换宽度），测试立刻失败，阻止合入主分支。这是 KV 系统替代 DDL 约束的最终防线——编译期类型安全 + 运行时 hex 校验，双保险。

### 判定

SQL 的核心价值是面向人类的结构化纪律。KV 只要贯彻「代码即 DDL 的强类型编码 + 多版本 Enum 懒迁移 + 双写 Key 指针契约 + hex 硬编码单元测试」，就同时获得了：编译期 Schema 安全（Rust 编译器）+ 运行时极致性能（LSM-Tree）+ 零停机演进（版本化 Enum）+ 团队可维护性（Struct 注释即文档）+ 编码漂移防护（hex 单元测试）。在 Schema 安全性上完成对 SurrealDB（无模式）和 PostgreSQL（运行时 DDL 锁）的双向反超。

## 问题：字符串前缀的两大软肋

字符串前缀方案 `"user_sessions:"`（14 字节）在亿级数据量下：
- **空间浪费**：相同前缀重复存储，冲到亿级时浪费数 GB 存储空间（S3 传输带宽费）
- **变长偏移**：每个 Key 总长度动态变化，反序列化需要变长偏移计算

## 解决方案：数字命名空间字典

引入**编译期常量**，将字符串前缀压缩为固定 2 字节 u16 命名空间 ID：

| 字符串方案 | 数字方案 | 压缩率 |
|:--|:--|:--|
| `"user_sessions:"` (14 字节) | `u16::to_be_bytes()` → 2 字节 | **85%** |

命名空间 ID 由 `#[kv_ns(N)]` 在编译期折叠为常量（指令立即数），写 key 时零运行时查找——比任何 L1 常驻的 HashMap 还快（无 hash、无 load）。

全局命名空间字典（编译期确定）：

```
Namespace 1 → sessions
Namespace 2 → users
Namespace 3 → logs
```

**设计取舍：为何命名空间留在代码、不落 KV**

- **ns 硬编码在代码里并没问题**：KV 的访问模式本身就在代码里——二进制 key、无分隔符、每段长度都由代码定义。ns 在代码里，和整套 key 布局在代码里是同一件事。
- **把 ns 放 KV、让过程宏去读是自举（bootstrap）问题**：宏在编译期运行，KV 那刻还没被这份源码播种，鸡生蛋。
- **即便允许编译期读 KV，也几乎没有好处**：要单独写一个程序维护元数据 → 两份代码（KV 元数据 + 源码）容易不一致；编译不透明（依赖正在跑的服务）；状态还需同步。全是负担。
- **运行时"开放"注册无意义**：访问模式在代码里，加一个 ns 仍要改代码，动态注册换不来动态访问。

结论：ns 的唯一真源就是**代码（编译期常量）**；KV 只存这些 key 之下的数据，不存字典本身。

**编号分配与位宽：编号手动稳定、位宽锁死 u16**

- **编号的职责**：nsid 是**判别位，不是排序键**——它是 key 最外层的区分前缀，同一 ns 的 key 天然因前缀字节相同而聚集扫描，**数值是否相邻无语义价值**。因此没有动机为"有序"重排，只需 append-only 唯一分配（1、2、3…）永不改号。
- **为何不按声明序自增编号**：`#[derive]` 按源码出现顺序处理，若让全局计数器自动分配（`C` 插在 `A`、`B` 之间则 `B` 变 3），**位置相关编号在插入时必然重排后缀**——任何基于排序/排名的编号（声明序、字母序、任意全序下的 rank）都不稳定，而 nsid 物理上烙在磁盘 key 里，重排即旧数据失联。
- **为何不用按 struct 名字 hash 编号**：hash 确实顺序无关（插 `C` 不重排任何人），正面解决自增编号的死穴，但它输在最小位宽这条硬目标上——hash 输出铺满整个值空间（u32=4 字节），位宽按 hash 位宽算而非按 ns 总数算，比固定 u16 还大一倍；若截断 hash 回到小位宽（如 8 位），生日碰撞概率 ≈ N²/(2·256)，N=50 时已有约 4.9 对预期碰撞，**必然要再写一个编译期查重+扰动装置**——而一旦有"收集全部名字→查重→保证唯一"的通道，这条通道里直接发计数器就行，hash 反成装饰。故 hash 只在小位宽目标不存在时才成立（如 ns 来自多租户/插件/跨 crate 动态注册、无统一改号处），本系统 ns 全在代码、编译期已知，不在此列。
- **为何不自主任由"运行期自动推导位宽"（OnceLock/fetch_max 方案，思路有价值）**：曾考虑让每个 derive 生成一条 `NS_WIDTH.fetch_max(我的所需位数)` 语句，用一个全局 `NS_WIDTH`（OnceLock/AtomicU8）在运行期**初始化时**由全体 struct 共同推进成最大值，编码/解码偏移从 `width` 语义值读取（`bytes[width..width + len]`），从而按"当前 ns 集合最大值"定最小位宽、绕过 derive 扫不到全代码的编译期死题。
  - **机制自洽、思路成立**：`fetch_max` 满足交换律、顺序无关，且只是把"按最大值定最小位宽+自动维护"从编译期挪到运行期初始化，一次设值后恒定——**推导出的位宽并不因此就不相干**。
  - **为何否决**：位宽从此绑定**全局 ns 集合**而非单个 key 自身——加第 256 个 ns 时，**所有无关的、早已存在的 key** 前缀从 1B 变 2B，key 布局被悄然重定义，且发生在"加一两个 struct"这种最不起眼的日常提交里。后果是滚动更新/灰度时**新旧版本二进制在线并存、宽度不同，新写读不出老、老读不出新**，属不可计划的兼容事故；而宽度既非源码显式常量、又由 build 内容推导，hex 稳定性测试也拦不住——**这正是我们拒绝"插入重排/数量变化改编码"的同一类病，只是换到运行期复发**。2.4% 的省字节买不回来。
  - **此与编译期推导的差异**：运行期方案绕开了编译期跨 item 通信的死题（机制可跑），但它和编译期推导**共享同一个病根**——"位宽随全局 ns 数量变化"即"key 布局与全局数量耦合"，与显式焊死的 u16 相比，只是把不可靠换了个位置犯病。
- **位宽锁死 u16（2 字节）**：支撑 65535 个 ns；本系统 ns 数量级为几十上百、增长缓慢（不会剧烈变动成千上万行新增），但 2B vs 1B 仅差每 key 1 字节（≈2.4% 存储/传输成本），换来**零机器、免维护、天然正确、永无临界点**——容头部空间大到 ns 增长永远够不到，从结构上消灭"加一两个越界即全库重排"这类 bug。`#[kv_ns(1)]` 编码为 2 字节 `[0x00, 0x01]`。不使用更小宽度也因**非字节对齐前缀**（如 4 位 nibble）需位操作、偏移跨字节，敲碎"纯指针切片零解析"的定长基石。

**术语与多租户**

- **namespace = 表/集合（KV 母语）**：此处的 namespace 不是"容器/作用域"，而是 KV/noSQL 里"表/集合"的标准同义词（Cassandra 的 keyspace 同理）——它只是 key 前缀的判别位，本身不存在逻辑意义上的"表"（无表级 schema 约束、无列/类型），故叫 namespace 在 KV 母语里最准。
- **多租户键结构**
  - **key 形状**：若有多个租户，key 形如 `[ns_id/表][tenant_id]…`——表/ns 在最外层，后跟 tenant_id。
  - **没有 `tenant_ns_id` 一层**：本系统**不暴露"用户可编程的查询/schema 面"**（多租户由 API 网关承载、租户共享同一套表），不存在租户自定义 namespace 需求。这不是"KV 不如 SQL 动态"——**SQL 在程序内部同样是写死的代码**，即席动态来自对外暴露的查询面，而非存储本身更动态（论证见 [kv-storage-engine](kv-storage-engine.md)「什么时候用 SQL」）。
  - **不在最外层另套 tenant_id**：即便对 SaaS 应用，按租户做外层隔离也是过度隔离——租户共享同一套表/schema，多租户由 API 网关承载；tenant_id 只是 key 里的一个判别字段，不是分区/表边界。

## DDL 结构体：全定长、零浪费

```rust
#[derive(KeyEncode)]
#[kv_ns(1)]  // 编译期翻译为 [0x00, 0x01] 2 字节大端序前缀
pub struct UserSessionKey {
    pub org_id: [u8; 16],     // 16 字节
    pub user_id: u64,          // 8 字节
    pub session_id: [u8; 16], // 16 字节
}
// 物理总长度 = 2(ns) + 16 + 8 + 16 = 42 字节，零字节浪费
```

对照**二级索引可变长**：姓名→id 索引用变长 key——判别文本放最前、UTF-8 原样存储即天然字典序扫描序；连续多个字符串字段同样支持（每个字段独立 `[len: u16]` 长度前缀 + UTF-8 字节，宏逐个推进 offset 游标）。索引 key 的末尾**挟带主表主键**，value 留空——前缀扫到索引条目后，从 key 尾部提出 `user_id` 回查主表：

```rust
/// User 表按 (name, city) 组合字段的二级索引：
/// [索引字段…]@[主表主键 user_id]，Value 留空（锚点位，仅作前缀扫描回查）
#[derive(KeyEncode)]
#[kv_ns(2)]  // 编译期翻译为 [0x00, 0x02]
pub struct UserIndexNameCityKey {
    pub name: String,    // 变长索引字段 1：[len: u16] + UTF-8 字节
    pub city: String,    // 变长索引字段 2：同样 [len: u16] + UTF-8 字节
    pub user_id: [u8; 16], // 主表主键：前缀扫到此处后提出该 id 回查 User 主表
}
// 物理布局 = 2(ns) + (2+len(name)) + (2+len(city)) + 16(user_id) 字节；
// 变长在前 ⇒ user_id 前的偏移须运行期游标推进（消除 compile-time 偏移死锁，零解析让位于扫描能力）
```

- 命名 `UserIndexNameCityKey` = 主表 `User` + 索引标记 `Index` + 索引字段 `NameCity`，一眼看出它是什么表、什么字段的副索引。
- **统一策略：ID 一律放 Key 末尾，搜索用前缀扫描**。`user_id` 作末尾锚点（Value 留空），`scan(idx, scan_fn)` 以"前 idx 个字段"为扫描前缀、把命中 key 逐一 decode 成 `Vec<Self>`；是否唯一（Vec 长度）由调用方判断，不必在布局上切换 unique/非 unique 两套。只有纯粹的点查热路径（索引字段业务唯一、从不扫多条）才值得把 ID 移入 Value 让 get 单发。完整论证见 [kv-storage-engine](kv-storage-engine.md)「索引主键 ID：统一放 Key 末尾，Value 留空」。
- 索引字段（`name`/`city`，变长）放**最前**参与前缀扫描；主表主键 `user_id`（定长 16B）放**末尾**作回查锚点——扫到即提 id，value 留空不占索引格子。
- 本结构体**没有**尾随的 `Ok(Self { … })` 环形字段——变量部分只有两个 `String`，尾部 `user_id` 为定长。连续多字符串不冲突：`let _len` 可被同名 shadow、不同字段名天然各异、`offset += 2 + _len` 把游标推进到下一字段起点；也演示了**变长字段之后跟定长字段**——此时该定长字段偏移只能靠运行期游标推进（先累加各 `2+len` 到它，再 `+= 16`），这正是"变长之后的字段失去编译期偏移"的实例。`scan(idx, scan_fn)` 的前缀扫描调用见下文[运行时模块 + 测试](#运行时模块--测试kv_codecsrclibrs)的测试用例。

**定宽的边界：主键定宽、索引可变长**

- **定宽针对"结构"，不针对"数据"**：定宽说的是 key 的**结构**（几个字段、什么类型），不是字段里的**数据**。姓名等文本是数据，天生不定长——拿定宽去装它，"不够"或"浪费"是把定宽这个外来约束强加给本该自由的字段，自找别扭。
- **判据是访问模式**：
  - **只需精确点查**（给键 → 拿值，永不前缀/范围扫描）→ 可 hash 成定宽 key，整套定宽方案保留；代价是拿不回原值、不再按文本字典序、极端情形需防碰撞（128-bit 可忽略）。仅在 100% 点查时才选。
  - **需要前缀/范围扫描**（如"查所有姓 Zha 的"、按名排序）→ **绝不能 hash**，须保留原始文本作**变长 key**。LSM 天然按 key 字节排序，UTF-8 原样存储即免费获得字典序范围扫描——这是 hash 永远给不回来的能力，此时该 keyspace 整体放弃定宽。
- **姓名→id 之类的索引几乎必然走变长**：它当索引就是为了回答前辍/范围查询，纯点查不如下游直接查文件。故姓名类索引用变长布局是主流情形。
- **变长布局编码**：
  - **长度前缀** `[len: u16][name bytes]`：最通用，能装任意二进制、无转义负担；长度字节参与排序但无害（前缀稳定字段挡在前面）。
  - **NUL 结尾** `[name bytes][0x00]`：省一字节，但需处理内嵌 NUL 的转义正确性。
  - **倾向长度前缀**：通用、可测、不引入转义新负担；姓名这点长度开销不值得换 NUL 方案。
- **Rust offset 算术的部分放宽**：定宽字段在变长字段**之前**时，前面的偏移仍是编译期已知、仍零解析——只有变长之后的字段落到运行期。故"定宽前缀 + 末尾变长尾缀"保留大部分零解析收益。但姓名索引的文本是判别位、要放最前参与扫描，变长在最开头，此放宽对它不适用，需整键变长。
- **定宽是主键空间的资产，不是全局教条**："零解析零浪费"值得付 offset 算术成本，因为主键结构刚硬、且在热路径；姓名类索引扫描导向、读取热度次于主键，解析代价是一次读长度，微不足道，为它强行 padding/截断是在优化错的维度。**正确模型是两套布局 regime 并存**：主键定宽 + 索引可变长，不是互相否定、是各归其位。
- **hex 稳定性测试不丢**：变长锁的不再是"具体文本的字节"（本无固定字节），而是**编码方案本身**——长度前缀布局、长度字节端序、`len` 取值上限等，用固定样本锁住，防止日后无意改编码致旧 key 失联。防线照旧，只是锁的对象变了。

## 编码 Wrapper 类型：字段级压缩（声明即自动编解码）

`KeyEncode` 把 kv-storage-engine 的编码原理封装成**字段级 wrapper 类型**——字段类型本身即契约，声明即自动编解码：

```rust
#[derive(KeyEncode)]
#[kv_ns(4)]  // logs
pub struct LogEntryKey {
    pub ts_offset: Offset<i32>,   // 分布集中 → 基准 + 有符号偏移（事件时间）
    pub err_type:   Enum<u8>,     // 低基数 → 枚举编号（日志错误类型）
}

#[derive(KeyEncode)]
#[kv_ns(5)]  // metrics（append-only 序列）
pub struct MetricKey {
    pub seq_delta: Delta<i32>,    // 序列相邻差 → 常为正小值，配 VarInt 更省
    pub kind:      Enum<u16>,     // 低基数指标种类
}
```

| wrapper | 维度 | 编码 | 读路径 |
|:--|:--|:--|:--|
| `Enum<T>` | 基数 | 值 → 编号（**位宽锁死 `T` 的固定字节宽**，与命名空间位宽同一决策，见下文） | O(1) 查表回译 |
| `Offset<T>` | 分布 | `值 − 基准`（有符号；基准取近今，负偏移表达更早值） | 基准 + 偏移 |
| `Delta<T>` | 序列 | 与前值差（可变长更省） | 顺序累加（随机/乱序访问须顺序重建） |
| `VarInt<T>` | 位宽 | 常见小值 1 字节、大值扩展（LEB128） | 逐字节 continuation bit |
| `Rle<T>` | 重复 | 连续相同 → `(值, 次数)` | 展开 |
| `Quant<T>` | 量纲 | 时间降精度 / 归桶（ns→ms / 小时桶） | 放缩 |
| `Reverse<T>` | 排序方向 | **补码逐位反转**（`!bits.to_be_bytes()`），正序字节序变倒序——prefix scan 首条即最新（时间线、最新优先列表） | 同样反转回译 |

**宏只生成对应 wrapper 的编解码**：字段类型即契约——`Enum<u8>` 告诉框架"这字段只存 1 字节编号"，`Offset<i32>` 告诉框架"存有符号相对值"，`Delta<i32>` 告诉框架"与前值求差"。原理与取舍见 [kv-storage-engine](kv-storage-engine.md)「编码优化」。

**`Reverse<T>`：补码反转实现倒序扫描**。对定宽整型（`i64`/`u64`/`i32`/`u32`/`u16`/`u8`）逐位取反后按大端序写入——大值变小字节序列、小值变大，字节序扫描顺序与数值序完全颠倒，prefix scan 天然"最新优先"，无需应用层排序或二级索引。整数补码表示本身有序（含负数），取反不破坏可比较性，回译即再取反。**编译期门卫**：`Reverse<T>` 借助 trait 实现约束只接受可反转类型——宏展开出的 `where` 子句要求 `T: Reversible`（blanket impl 覆盖全部定宽整型，其余类型不实现），不可反转的类型（浮点、字符串、数组）误用直接编译报错，而非静默产出乱序字节：

```rust
pub trait Reversible: Copy {
    /// 编码：bits.to_be_bytes() 逐位取反；解码：再取反还原
    fn reverse_encode(self) -> [u8; Self::WIDTH];
    fn reverse_decode(bytes: &[u8]) -> Self;
}

// blanket impl 只覆盖定宽整型——这是能力白名单，不在名单上的类型写进 Reverse<T> 即编译失败
macro_rules! impl_reversible {
    ($($t:ty),*) => { $(
        impl Reversible for $t {
            const WIDTH: usize = core::mem::size_of::<$t>();
            fn reverse_encode(self) -> [u8; Self::WIDTH] {
                let mut b = self.to_be_bytes();
                b.iter_mut().for_each(|x| *x = !*x);   // 补码逐位反转
                b
            }
            fn reverse_decode(bytes: &[u8]) -> Self {
                let mut b: [u8; Self::WIDTH] = bytes[..Self::WIDTH].try_into().unwrap();
                b.iter_mut().for_each(|x| *x = !*x);
                <$t>::from_be_bytes(b)
            }
        }
    )* };
}
impl_reversible!(u8, u16, u32, u64, i8, i16, i32, i64);

// 宏为 Reverse<T> 字段展开的编解码自动带 where 约束（编译期检查）
// 反例：Reverse<f64> → the trait bound `f64: Reversible` is not satisfied
// 反例：Reverse<String> → the trait bound `String: Reversible` is not satisfied
```

浮点为何不在名单：IEEE 754 字节序与数值序不同构（负数位模式反而更大），逐位反转得到的是乱序字节——排序正确性会被静默破坏，这类错误必须挡在编译期而不是靠测试兜底。若确需倒序浮点，先 Quant 归桶成整型再 Reverse（组合 wrapper），转换语义显式可见。

**Enum 位宽锁死类型字节宽，不按基数推导**：`Enum<T>` 的物理宽度 = `T` 的字节宽（`Enum<u8>` = 1 字节、`Enum<u16>` = 2 字节），由编码方自己选定。不采用"位宽 = ⌈log₂基数⌉"的最小位宽方案，理由与命名空间锁死 u16 同根：

- **推导在编译期做不到**：宏只看得见单个 derive item，数不到全代码库的枚举变体总数；要跨 item 收集基数就得引入 `LazyLock` 式运行期登记——运行时参与编码计算，过于动态，也伤性能。
- **更致命的是数据格式漂移**：即便推导成功，前一个位宽填满后新增枚举变体会让位宽自动扩展一格——**所有历史 key 的布局被悄然重定义**，旧数据集体失联。这与 ns 位宽"运行期 fetch_max 推导"被否决是同一类病，只是搬到字段级复发。
- **固定宽度 + 补位余量的代价可忽略**：单字段 1 字节 vs 4 bit 的差异摊到整条 key 上不到 2%，换来"新增变体永不改布局"的结构性稳定。变体总数逼近位宽上限时，编码方**显式**把 `Enum<u8>` 升格为 `Enum<u16>` 并走版本化 Enum 懒迁移——布局变更是一个可见的、被 hex 测试拦住的动作，不是静默漂移。

与**键布局（字段顺序/主键定宽 vs 索引变长）正交**：wrapper 是**压缩语义**（怎么存），不是**结构语义**（放哪个位置）；两者可叠加（`Offset<i32>` 字段本身仍参与前缀扫描、`Enum` 字段可作索引判别位）。

## Value 侧：版本化 payload 与 TLV 扩展区

Key 层已零浪费（ns 字典 + 定宽偏移），但 value 用 postcard——**非自描述格式**，字段按位置而非按 tag 解码。这使「加字段」成为整行打包布局的结构性代价：旧数据没有字段位置信息，新 reader 无法跳过不认识的尾部。解法不在换布局（见 kv-storage-engine「value 打包粒度」的论证），而在让序列化格式本身可演进。OKM 提供两级机制，按需取用。

### L1：版本化 payload（最简）

value 头带 1 字节 `schema_version`（`u8`——位宽锁死所选类型的固定字节宽，与 Enum/ns 同一决策：版本数不可能逼近 256，u16 是无谓浪费；0 保留为未版本化初始布局）：

```rust
#[derive(ValueEncode)]
#[kv_version(2)]                       // 显式声明当前版本，宏写入 value 头
pub struct AgentMemoryValue {
    pub content: [u8; 64],             // 定长热区字段
    pub score:   i32,
}
```

宏展开的 `encode_value` / `decode_value`：

```rust
impl AgentMemoryValue {
    pub const SCHEMA_VERSION: u8 = 2;

    pub fn encode_value(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(1 + 64 + 4);
        buf.push(Self::SCHEMA_VERSION);            // 1 字节版本头
        buf.extend_from_slice(&self.content);
        buf.extend_from_slice(&self.score.to_be_bytes());
        buf
    }

    pub fn decode_value(bytes: &[u8]) -> Result<Self, &'static str> {
        match bytes[0] {
            2 => Self::decode_v2(&bytes[1..]),     // 当前版本直解
            1 => Self::decode_v1(&bytes[1..]),     // 旧版本按旧布局解，读时升级
            _ => Err("未知 schema 版本"),
        }
    }
}
```

**懒迁移**：`decode_v1` 解出后按 v2 语义补默认值、`encode_value` 写回——Key 不变、零 DDL、新老行共存。纪律：字段只允许**尾部追加**（v1 的字段在 v2 中位置不变），中间插字段 = 破坏旧 reader，被版本号 bump 拦住。

### L2：Tagged/TLV 扩展区（主轴）

L1 的痛点是每加一个字段都要 bump 版本 + 写旧版 decoder。TLV 把「演进」从版本分派降为**字段自描述**：payload 分两段，定长热区（现有 `KeyEncode` 直出，位置契约不变）+ tagged 扩展区（`field_id: u8 + len: u16 + bytes`）：

```rust
#[derive(ValueEncode)]
#[kv_version(2)]
pub struct AgentMemoryValue {
    pub content: [u8; 64],             // 热区：定长偏移直读
    pub score:   i32,

    #[kv_ext(id = 1)]                  // 扩展区：新字段声明即归段
    pub tags:    Vec<String>,
    #[kv_ext(id = 2)]
    pub flags:   u8,
}
```

物理布局：

```
[ ver:1B | content:64B | score:4B | ext_count:u8 | (id:1B len:2B bytes)... ]
  └── 热区，偏移编译期已知 ──┘  └──── tagged 扩展区，逐项跳读 ────┘
```

- **旧 reader 兼容**：v1 的 decoder 只读到热区末尾，扩展区整段跳过（`ext_count` 之后按 len 跳项）——加字段零迁移、无需写旧版 decoder，这是对 L1 最大痛点的消除。
- **新 reader 读扩展字段**：扫 tagged 段按 `id` 匹配，未知 id 跳过（向前兼容后继版本）。
- **`id` 编码方手动分配**，宏不自动编号——自动编号又是一个跨 item 收集状态的需求，与「宏只看得见单个 derive item」的约束同构；手动 id 让字段删除后的 id 不被复用，旧数据不误读。
- **代价**：每项 3 字节头 + `ext_count` 1 字节，扩展区变长破坏定长容量预算（与 String 字段同策略：含扩展字段的结构体 `encode_value` 退回运行期游标，key 层不受影响——扩展区只存在于 value）。

### L3：热/冷分仓（按需升级）

L2 的扩展区字段读路径是逐项跳读（O(字段数)）。当某个扩展字段升级为高频单字段热路径时，把它**提升进热区**：挪到热区字段列表尾部（旧 reader 凭版本号分派，热区扩展对 v1 是布局变更 → bump 版本，走一次显式迁移），这是唯一的显式动作，被 hex 测试拦住，不静默。

### 与 wrapper 体系的关系

wrapper 是 key 层的**压缩语义**（怎么存一个字段），TLV 是 value 层的**结构语义**（字段之间怎么排布），正交可叠加：扩展区的 `bytes` 内部同样可以用 postcard 编 Struct，热区字段照常享受定宽偏移。原理与取舍的完整论证见 [kv-storage-engine](kv-storage-engine.md)「value 打包粒度：整行 vs 每字段一个 key」。

## 过程宏实现源码

### 宏编译器驱动（`kv_codec_derive/src/lib.rs`）

```rust
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::quote;
use syn::{parse_macro_input, AttrStyle, Data, DeriveInput, Fields, Lit, Meta, Type, Expr};

#[proc_macro_derive(KeyEncode, attributes(kv_ns, kv))]
pub fn derive_kv_encode(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // 编译期提取数字 Namespace ID，转为 u16 大端序 2 字节
    let ns_id = extract_ns_id(&input);
    let ns_bytes = ns_id.to_be_bytes();

    let fields = match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(fields_named) => &fields_named.named,
            _ => panic!("只支持带命名字段的结构体"),
        },
        _ => panic!("只支持结构体"),
    };

    let name = &input.ident;
    let field_names: Vec<_> = fields.iter().map(|f| f.ident.clone().unwrap()).collect();
    let mut encode_tokens = TokenStream2::new();
    let mut decode_tokens = TokenStream2::new();
    let mut capacity_tokens = TokenStream2::new();
    let mut encode_prefix_tokens = TokenStream2::new(); // search 用：编码前 N 个字段做扫描前缀
    let mut current_offset = 2; // 前缀永远锁死在 2 字节位置
    let mut field_idx = 0usize;

    // 检测结构体是否含变长字段（当前仅 String）；决定容量预算与解码偏移策略
    let has_var = fields.iter().any(|f| match &f.ty {
        Type::Path(tp) => tp.path.is_ident("String"),
        _ => false,
    });

    for field in fields {
        let field_name = &field.ident;
        let field_type = &field.ty;

        match field_type {
            Type::Path(tp) if tp.path.is_ident("String") => {
                // 变长：长度前缀 [len: u16] 大端序 + UTF-8 字节
                encode_tokens.extend(quote! {
                    let _s: &[u8] = self.#field_name.as_bytes();
                    buf.extend_from_slice(&(_s.len() as u16).to_be_bytes());
                    buf.extend_from_slice(_s);
                });
                // search 前缀：仅当该字段被纳入前缀（idx < n）时编码，否则到此截断
                encode_prefix_tokens.extend(quote! {
                    if idx > #field_idx {
                        let _s: &[u8] = self.#field_name.as_bytes();
                        buf.extend_from_slice(&(_s.len() as u16).to_be_bytes());
                        buf.extend_from_slice(_s);
                    }
                });
                decode_tokens.extend(quote! {
                    // `offset` 由 decode 函数体顶部 `let mut offset = 2usize` 提供（见下节 expanded 的 decode）
                    let _len = u16::from_be_bytes(bytes[offset..offset + 2].try_into().unwrap()) as usize;
                    let #field_name = String::from_utf8(
                        bytes[offset + 2..offset + 2 + _len].to_vec()
                    ).map_err(|_| "UTF-8 解码失败")?;
                    offset += 2 + _len;
                });
            }
            Type::Array(type_array) => {
                let len = &type_array.len;
                encode_tokens.extend(quote! {
                    buf.extend_from_slice(&self.#field_name);
                });
                encode_prefix_tokens.extend(quote! {
                    if idx > #field_idx {
                        buf.extend_from_slice(&self.#field_name);
                    }
                });
                capacity_tokens.extend(quote! { + #len });
                decode_tokens.extend(quote! {
                    let mut #field_name = [0u8; #len];
                    #field_name.copy_from_slice(&bytes[offset..offset + #len]);
                    offset += #len;
                });
                current_offset += syn_parse_len(len);
            }
            Type::Path(tp) if tp.path.is_ident("u64") || tp.path.is_ident("i64") => {
                encode_tokens.extend(quote! {
                    buf.extend_from_slice(&self.#field_name.to_be_bytes());
                });
                encode_prefix_tokens.extend(quote! {
                    if idx > #field_idx {
                        buf.extend_from_slice(&self.#field_name.to_be_bytes());
                    }
                });
                capacity_tokens.extend(quote! { + 8 });
                decode_tokens.extend(quote! {
                    let #field_name = #field_type::from_be_bytes(
                        bytes[offset..offset + 8].try_into().unwrap()
                    );
                    offset += 8;
                });
                current_offset += 8;
            }
            Type::Path(tp) if tp.path.is_ident("u32") || tp.path.is_ident("i32") => {
                encode_tokens.extend(quote! {
                    buf.extend_from_slice(&self.#field_name.to_be_bytes());
                });
                encode_prefix_tokens.extend(quote! {
                    if idx > #field_idx {
                        buf.extend_from_slice(&self.#field_name.to_be_bytes());
                    }
                });
                capacity_tokens.extend(quote! { + 4 });
                decode_tokens.extend(quote! {
                    let #field_name = #field_type::from_be_bytes(
                        bytes[offset..offset + 4].try_into().unwrap()
                    );
                    offset += 4;
                });
                current_offset += 4;
            }
            Type::Path(tp) if is_coding_wrapper(&tp.path) => {
                // 编码 wrapper（Enum / Offset / Delta / VarInt / Rle / Quant，见「编码 Wrapper 类型」）：
                // 字段类型即契约，编解码统一由 emit_wrapper_codec 生成；定宽字段在变长字段之前仍保留编译期偏移
                let (enc, dec, cap, width) = emit_wrapper_codec(&tp.path, quote! { #field_name });
                encode_tokens.extend(enc.clone());
                encode_prefix_tokens.extend(quote! { if idx > #field_idx { #enc } });
                capacity_tokens.extend(cap);
                decode_tokens.extend(dec);
                current_offset += width;
            }
            _ => panic!("OKM 键空间只接受 [u8; N]、基础数值类型、编码 wrapper 和 String（String 为变长字段）"),
        }
        field_idx += 1;
    }

    // 含变长字段时无法编译期预算精确容量，退回无预分配；纯定长保留 with_capacity
    let capacity_or_plain = if has_var {
        quote! { Vec::new() }
    } else {
        quote! { Vec::with_capacity(2 #capacity_tokens) }
    };

    let expanded = quote! {
        impl #name {
            pub const NAMESPACE_ID: u16 = #ns_id;

            pub fn encode(&self) -> Vec<u8> {
                let mut buf = #capacity_or_plain;
                buf.extend_from_slice(&[#(#ns_bytes),*]);
                #encode_tokens
                buf
            }

            pub fn decode(bytes: &[u8]) -> Result<Self, &'static str> {
                if bytes.len() < 2 || bytes[0..2] != [#(#ns_bytes),*] {
                    return Err("数据损坏或 Namespace 契约冲突");
                }
                // 运行期偏移游标；纯定长结构体下每次递增都是编译期常量，LLVM 折叠后仍为立即数寻址（零解析保留）
                let mut offset = 2usize;
                #decode_tokens
                if bytes.len() != offset {
                    return Err("数据长度与 Schema 不符");
                }
                Ok(Self { #( #field_names ),* })
            }

            /// 编码"前 n 个字段"作为扫描前缀（n=索引字段个数）；
            /// 变长字段仅在自己被纳入前缀（idx < n）时才写出长度前缀与字节，之后字段截断。
            /// 由 search 调用，构造 LSM 前缀扫描范围。
            pub fn encode_prefix(&self, n: usize) -> Vec<u8> {
                let mut buf = Vec::new();
                buf.extend_from_slice(&[#(#ns_bytes),*]);
                #encode_prefix_tokens
                buf
            }

            /// 前缀扫描：对 `encode_prefix(idx)` 命中的所有 key 逐一 decode 成 Self 收集进 Vec。
            /// `idx`（索引字段个数）决定编码前缀到第几个字段；`scan_fn` 由 KV 层注入（如 `kv.scan(prefix)` 返回 key 集合）。
            /// ID 一般编码在 Key 末尾，用户自行从 Vec 尾项取最后一字段。
            pub fn scan<F>(&self, idx: usize, scan_fn: F) -> Vec<Self>
            where
                F: FnOnce(&[u8]) -> Vec<Vec<u8>>,
            {
                let prefix = self.encode_prefix(idx);
                scan_fn(&prefix)
                    .iter()
                    .filter_map(|k| Self::decode(k).ok())
                    .collect()
            }
        }
    };
    TokenStream::from(expanded)
}

fn extract_ns_id(input: &DeriveInput) -> u16 {
    for attr in &input.attrs {
        if attr.style == AttrStyle::Outer && attr.path().is_ident("kv_ns") {
            if let Meta::List(meta_list) = &attr.meta {
                let expr: Expr = meta_list.parse_args().expect("kv_ns 格式错误");
                if let Expr::Lit(expr_lit) = expr {
                    if let Lit::Int(lit_int) = expr_lit.lit {
                        return lit_int.base10_parse::<u16>().expect("Namespace ID 必须是 u16");
                    }
                }
            }
        }
    }
    panic!("必须标注 #[kv_ns(N)]");
}

fn syn_parse_len(expr: &syn::Expr) -> usize {
    if let syn::Expr::Lit(expr_lit) = expr {
        if let syn::Lit::Int(lit_int) = &expr_lit.lit {
            return lit_int.base10_parse::<usize>().unwrap();
        }
    }
    panic!("无法解析数组长度");
}

/// 编码 wrapper 类型（压缩语义，非结构语义）：Enum / Offset / Delta / VarInt / Rle / Quant，见「编码 Wrapper 类型」。
fn is_coding_wrapper(path: &syn::Path) -> bool {
    path.segments
        .last()
        .map(|s| {
            matches!(
                s.ident.to_string().as_str(),
                "Enum" | "Offset" | "Delta" | "VarInt" | "Rle" | "Quant"
            )
        })
        .unwrap_or(false)
}

/// 编码 wrapper → (encode, decode, capacity, 字节宽)。
/// 字段类型即契约：Enum 存 u8 编号、Offset 存有符号基准差（基准近今）、Delta 存相邻差……（原理见 kv-storage-engine「编码优化」）
fn emit_wrapper_codec(
    path: &syn::Path,
    field: proc_macro2::TokenStream,
) -> (proc_macro2::TokenStream, proc_macro2::TokenStream, proc_macro2::TokenStream, usize) {
    let w = path.segments.last().map(|s| s.ident.to_string()).unwrap_or_default();
    match w.as_str() {
        "Enum" => (
            quote! { buf.extend_from_slice(&[#field]); },       // 低基数：值 → u8 编号（映射为编译期常量）
            quote! { let #field = bytes[offset]; offset += 1; }, // 回译：编号 → 枚举值由外层 value 类型查表
            quote! { + 1 },
            1,
        ),
        "Offset" => (
            quote! { buf.extend_from_slice(&(#field as i32).to_be_bytes()); }, // 有符号基准差
            quote! {
                let #field = i32::from_be_bytes(bytes[offset..offset + 4].try_into().unwrap());
                offset += 4;
            },
            quote! { + 4 },
            4,
        ),
        "Delta" => (
            quote! { buf.extend_from_slice(&(#field as i32).to_be_bytes()); }, // 相邻差（配 VarInt 更省，示意为 i32）
            quote! {
                let #field = i32::from_be_bytes(bytes[offset..offset + 4].try_into().unwrap());
                offset += 4;
            },
            quote! { + 4 },
            4,
        ),
        "VarInt" => (
            // LEB128：小值 1 字节、大值逐字节扩展；encode 由运行时循环产出（宽度不定，无 capacity 预算）
            quote! {{
                let mut v = #field as u64;
                loop {
                    let mut byte = (v & 0x7F) as u8;
                    v >>= 7;
                    if v != 0 { byte |= 0x80; }
                    buf.push(byte);
                    if v == 0 { break; }
                }
            }},
            quote! {{
                let mut v: u64 = 0;
                let mut shift = 0;
                loop {
                    let byte = bytes[offset];
                    offset += 1;
                    v |= ((byte & 0x7F) as u64) << shift;
                    if byte & 0x80 == 0 { break; }
                    shift += 7;
                }
                let #field = v as _;
            }},
            quote! { /* 变长，无固定预算 */ },
            0, // 宽度不定：含 VarInt 字段的结构体退化为运行期游标（同变长字段策略）
        ),
        "Rle" => (
            // (值, 次数) 对：值定宽 4B + 次数 u16；示意单游程编解码
            quote! {{
                buf.extend_from_slice(&(#field as i32).to_be_bytes());
                // 次数由上层游程检测填入，此处示意为 1
                buf.extend_from_slice(&1u16.to_be_bytes());
            }},
            quote! {{
                let #field = i32::from_be_bytes(bytes[offset..offset + 4].try_into().unwrap());
                let _count = u16::from_be_bytes(bytes[offset + 4..offset + 6].try_into().unwrap());
                offset += 6;
            }},
            quote! { + 6 },
            6,
        ),
        "Quant" => (
            // 量纲缩放：ns → ms（÷1_000_000），存 i64 毫秒
            quote! {{
                let ms = (#field as i64) / 1_000_000;
                buf.extend_from_slice(&ms.to_be_bytes());
            }},
            quote! {{
                let ms = i64::from_be_bytes(bytes[offset..offset + 8].try_into().unwrap());
                let #field = (ms * 1_000_000) as _;
                offset += 8;
            }},
            quote! { + 8 },
            8,
        ),
        "Reverse" => (
            // 补码逐位反转：大端字节序扫描天然倒序（最新优先）；T: Reversible 编译期门卫
            quote! {{
                let mut b = ::kv_codec::Reversible::reverse_encode(#field);
                buf.extend_from_slice(&b);
            }},
            quote! {{
                let #field = ::kv_codec::Reversible::reverse_decode(&bytes[offset..]);
                offset += <#field_ty as ::kv_codec::Reversible>::WIDTH;
            }},
            quote! { + <#field_ty as ::kv_codec::Reversible>::WIDTH },
            0, // 宽度由 T::WIDTH 决定（u64/i64 为 8），编译期常量
        ),
        _ => panic!("未知编码 wrapper: {w}"),
    }
}
```

### 运行时模块 + 测试（`kv_codec/src/lib.rs`）

```rust
pub use kv_codec_derive::KeyEncode;

#[cfg(test)]
mod tests {
    use super::*;

    #[derive(KeyEncode)]
    #[kv_ns(1)]
    pub struct UserSessionKey {
        pub org_id: [u8; 16],
        pub user_id: u64,
        pub session_id: [u8; 16],
    }

    #[test]
    fn test_okm_encode_decode() {
        let key = UserSessionKey {
            org_id: [0xAA; 16],
            user_id: 99,
            session_id: [0xBB; 16],
        };

        let bin = key.encode();

        // 验证物理总长度：2(ns) + 16 + 8 + 16 = 42 字节
        assert_eq!(bin.len(), 42);

        // 验证前缀：[0x00, 0x01] = Namespace 1 的大端序
        assert_eq!(bin[0..2], [0x00, 0x01]);

        // 验证反序列化
        let restored = UserSessionKey::decode(&bin).unwrap();
        assert_eq!(restored.org_id, key.org_id);
        assert_eq!(restored.user_id, key.user_id);
        assert_eq!(restored.session_id, key.session_id);
    }

    #[test]
    fn test_ddl_hex_stability() {
        let key = UserSessionKey {
            org_id: [1u8; 16],
            user_id: 50,
            session_id: [2u8; 16],
        };

        // 历史 hex 物理特征死锁——任何改动 CI/CD 立刻阻断
        let expected = "000101010101010101010101010101010101000000000000003202020202020202020202020202020202";
        assert_eq!(hex::encode(key.encode()), expected);
    }

    // 二级索引：ID 统一放 Key 末尾，Value 留空；前缀扫描返回 Vec<Self>
    #[derive(KeyEncode, Clone)]
    #[kv_ns(2)]
    pub struct UserIndexNameCityKey {
        pub name: String,     // 变长索引字段（长度前缀 [len: u16] + UTF-8）
        pub city: String,     // 变长索引字段
        pub user_id: [u8; 16], // 主表主键：放末尾作判别位与回查锚点
    }

    #[test]
    fn test_scan_prefix() {
        let a = UserIndexNameCityKey { name: "zhang".into(), city: "sh".into(), user_id: [0u8; 16] };
        let b = UserIndexNameCityKey { name: "zhang".into(), city: "sh".into(), user_id: [1u8; 16] };
        let c = UserIndexNameCityKey { name: "li".into(), city: "sh".into(), user_id: [2u8; 16] };

        // 构造"按 name+city 前缀扫描"：idx=2 把 name、city 纳入前缀，user_id 截断成尾部判别位
        let probe = UserIndexNameCityKey { name: "zhang".into(), city: "sh".into(), user_id: [0u8; 16] };
        let prefix = probe.encode_prefix(2);
        assert!(a.encode().starts_with(&prefix));
        assert!(b.encode().starts_with(&prefix));
        assert!(!c.encode().starts_with(&prefix));   // 不同 city 不在该前缀下

        // scan(2)：KV 侧按前缀命中 a、b 两条，decode 成 Vec<Self>，user_id 从末尾提出
        let scan_fn = |p: &[u8]| {
            [&a, &b, &c]
                .iter()
                .filter(|k| k.encode().starts_with(p))
                .map(|k| k.encode())
                .collect::<Vec<_>>()
        };
        let matches = probe.scan(2, scan_fn);
        assert_eq!(matches.len(), 2);
        assert_eq!(matches[0].user_id, [0u8; 16]);
        assert_eq!(matches[1].user_id, [1u8; 16]);
        // 唯一性由调用方判断：length>1 即非唯一，无需切换布局
    }
}
```

### ValueEncode 派生宏（`kv_codec_derive/src/lib.rs`）

KeyEncode 之外的第二个派生宏——只认 value 侧属性（`#[kv_version]` 结构体级、`#[kv_ext]` 字段级），展开 `encode_value` / `decode_value`：

```rust
#[proc_macro_derive(ValueEncode, attributes(kv_version, kv_ext))]
pub fn derive_value_encode(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // 编译期提取版本号（u8），0 保留为未版本化初始布局
    let version = extract_version(&input);

    let fields = match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(fields_named) => &fields_named.named,
            _ => panic!("只支持带命名字段的结构体"),
        },
        _ => panic!("只支持结构体"),
    };

    let name = &input.ident;
    let field_names: Vec<_> = fields.iter().map(|f| f.ident.clone().unwrap()).collect();
    let mut hot_encode = TokenStream2::new();     // 热区：定宽偏移直写
    let mut hot_decode = TokenStream2::new();
    let mut ext_encode = TokenStream2::new();     // 扩展区：TLV 逐项
    let mut ext_decode = TokenStream2::new();
    let mut ext_fields: Vec<(String, proc_macro2::TokenStream)> = Vec::new();

    for field in fields {
        let field_name = &field.ident;
        let field_type = &field.ty;

        // 字段级属性分派：#[kv_ext(id = N)] → 扩展区；未标注 → 热区
        let ext_id = extract_ext_id(field);

        match (ext_id, field_type) {
            (None, Type::Array(ta)) => {
                let len = &ta.len;
                hot_encode.extend(quote! { buf.extend_from_slice(&self.#field_name); });
                hot_decode.extend(quote! {
                    let mut #field_name = [0u8; #len];
                    #field_name.copy_from_slice(&bytes[offset..offset + #len]);
                    offset += #len;
                });
            }
            (None, Type::Path(tp)) if is_numeric(tp) => {
                let width = numeric_width(tp);   // u64/i64→8, u32/i32→4 …
                hot_encode.extend(quote! {
                    buf.extend_from_slice(&self.#field_name.to_be_bytes());
                });
                hot_decode.extend(quote! {
                    let #field_name = #field_type::from_be_bytes(
                        bytes[offset..offset + #width].try_into().unwrap());
                    offset += #width;
                });
            }
            (None, Type::Path(tp)) if tp.path.is_ident("String") => {
                // 热区变长：长度前缀 [len: u16] + UTF-8
                hot_encode.extend(quote! {
                    let _s = self.#field_name.as_bytes();
                    buf.extend_from_slice(&(_s.len() as u16).to_be_bytes());
                    buf.extend_from_slice(_s);
                });
                hot_decode.extend(quote! {
                    let _len = u16::from_be_bytes(bytes[offset..offset+2].try_into().unwrap()) as usize;
                    let #field_name = String::from_utf8(bytes[offset+2..offset+2+_len].to_vec())
                        .map_err(|_| "UTF-8 解码失败")?;
                    offset += 2 + _len;
                });
            }
            (Some(id), _) => {
                // 扩展区字段：postcard 打包进 TLV 项（id:1B + len:2B + bytes）
                ext_fields.push((id.to_string(), quote! { #field_type }));
                ext_encode.extend(quote! {{
                    let payload = postcard::to_allocvec(&self.#field_name).unwrap();
                    buf.push(#id);
                    buf.extend_from_slice(&(payload.len() as u16).to_be_bytes());
                    buf.extend_from_slice(&payload);
                }});
                ext_decode.extend(quote! {{
                    // 逐项扫描：匹配 id 解码，未知 id 按 len 跳过（向前兼容）
                    let (item_id, payload) = next_tlv(bytes, &mut offset);
                    if item_id == #id {
                        #field_name = postcard::from_bytes(&payload).unwrap();
                    }
                }});
            }
            _ => panic!("ValueEncode 热区只接受 [u8; N]、数值类型与 String；结构化字段请标注 #[kv_ext]"),
        }
    }

    let expanded = quote! {
        impl #name {
            pub const SCHEMA_VERSION: u8 = #version;

            pub fn encode_value(&self) -> Vec<u8> {
                let mut buf = Vec::new();
                buf.push(Self::SCHEMA_VERSION);          // 1 字节版本头
                #hot_encode
                // 扩展区：ext_count + TLV 项（热区字段越多越早写，位置契约不变）
                buf.push(#ext_count);
                #ext_encode
                buf
            }

            pub fn decode_value(bytes: &[u8]) -> Result<Self, &'static str> {
                match bytes[0] {
                    Self::SCHEMA_VERSION => Self::decode_current(&bytes[1..]),
                    // 旧版本按旧布局解，读时升级（懒迁移）；未知版本拒绝
                    _ => Err("未知或过期 schema 版本"),
                }
            }
        }
    };
    TokenStream::from(expanded)
}

/// TLV 读取原语：offset 处取 (id, payload)，offset 前移
fn next_tlv(bytes: &[u8], offset: &mut usize) -> (u8, &[u8]) {
    let id = bytes[*offset];
    let len = u16::from_be_bytes(bytes[*offset+1..*offset+3].try_into().unwrap()) as usize;
    let payload = &bytes[*offset+3..*offset+3+len];
    *offset += 3 + len;
    (id, payload)
}

fn extract_version(input: &DeriveInput) -> u8 { /* 解析 #[kv_version(N)]，同 extract_ns_id */ }
fn extract_ext_id(field: &syn::Field) -> Option<u8> { /* 解析 #[kv_ext(id = N)] */ }
```

与 KeyEncode 的分工：宏编译器同一 crate 内两个入口，共享 `is_numeric` 等类型工具；ValueEncode 不接受 `#[kv(reverse)]`（排序方向是 key 的物理问题）与 `#[kv_ns]`（命名空间是 key 的编排问题）——挂错侧的属性直接编译报错。

## syn→quote 类型映射规则

| 字段类型 | syn 分析 | quote 生成 | 字节宽度 |
|:--|:--|:--|:--|
| `[u8; N]` | `Type::Array` | `extend_from_slice(&self.field)` | N（固定） |
| `u64` / `i64` | `Type::Path` | `extend_from_slice(&self.field.to_be_bytes())` | 8（固定） |
| `u32` / `i32` | `Type::Path` | `extend_from_slice(&self.field.to_be_bytes())` | 4（固定） |
| `String` | `Type::Path`（`is_ident("String")`） | 长度前缀 `[len: u16]` + `extend_from_slice(bytes)` | 2 + len（**变长**） |

说明：定长字段在变长字段**之前**时，其偏移仍是编译期常量、运行期游标仅做常量累加（LLVM 折叠为立即数寻址，零解析保留）；变长字段之后的字段偏移必须运行期推进。判别位文本放最前（姓名索引）时整键变长、无编译期偏移可言。

## OKM 的物理收益

- **85% 前缀压缩**：14 字符串 → 2 数字字节
- **100% 定长 Key**：所有字段在磁盘上的绝对字节偏移量被编译期死锁，反序列化纯指针切片，零解析
- **65535 命名空间**：u16 支撑 65535 个不同的键空间，覆盖网关 + Agent 全场景
- **Cache Locality 极致**：LSM-Tree 布隆过滤器拦截、内存 Seek 查找时 CPU 缓存局部性极好
- **零运行时开销**：无正则/split/AST，查询路径随 `cargo build --release` 固化为机器码

## 双派生宏：KeyEncode / ValueEncode / KvRecord

一个 struct 混装 key 字段与 value 字段、靠属性或注释分区，会让单 derive 宏承担「挑字段」逻辑且语义模糊——key 字段与 value 字段的物理命运本来就不同（前者进字节序排序，后者进 postcard blob），强塞进一个 struct 是声明层的假象。OKM 的最终形态是**三个宏各司其职**：

- `KeyEncode`——物理键序（Key 侧）
- `ValueEncode`——载荷演进（Value 侧）
- `KvRecord`——把 K/V 关联进存取接口（组装层，不碰编码）

### KeyEncode：Key 侧派生宏

```rust
#[derive(KeyEncode)]
#[kv_ns(6)]                            // ns 只属于 key 侧
pub struct AgentMemoryKey {
    pub session_id: [u8; 16],
    #[kv(reverse)]                     // 排序方向是 key 的物理问题
    pub timestamp: u64,
}
```

全字段即 key 字段，宏无「挑字段」逻辑；`#[kv(reverse)]` 排序方向、`#[kv_ns]` 命名空间都挂在自己的宏上，边界天然清晰。展开即「过程宏实现源码」的 key 编码路径（定宽偏移、wrapper 编解码、`encode_prefix`/`scan`）。

### ValueEncode：Value 侧派生宏

```rust
#[derive(ValueEncode)]
#[kv_version(2)]                       // 版本头只属于 value 侧
pub struct AgentMemoryValue {
    pub content: [u8; 64],
    #[kv_ext(id = 1)]                  // TLV 扩展区也是 value 侧的事
    pub tags: Vec<String>,
}
```

全字段即 value 字段，postcard 整行打包 + 版本头 + TLV 扩展区（见「Value 侧」章节）。value 侧永远不需要排序方向属性——`#[kv(reverse)]` 在 ValueEncode 上直接编译报错，错误归属明确。

### KvRecord：K/V 关联宏（组装层）

```rust
#[derive(KvRecord)]
#[kv_record(key = AgentMemoryKey, value = AgentMemoryValue)]
pub struct AgentMemory;
```

职责只有一件事：把两侧绑定进同一个原子 Batch 的存取接口——生成 `Collection` trait 绑定（`save` 积批次 / `find` 点查 / `write` 原子提交，见「Collection trait」节）。不碰编码，不做字段处理。

### 双 struct 的代价与诚实性

业务代码从「一个 struct」变「两个 struct」，读整条记录要 K/V 拼合。但这本是事实的显式化：key 字段进字节序排序、value 字段进序列化 blob，物理命运不同。分开声明让每侧 derive 回归单 item 纯函数，属性归属无歧义，分区猜测整个消失。

### Collection trait + KvCollection：容器契约与实现

`KvRecord` 展开出的容器与 `AuraStorage` 绑定（见 [Aura 架构 §3.1](aura-architecture.md#31-存储引擎抽象trait-分离)）。**`Collection` 是对不同引擎、不同容器暴露的统一接口**——上层业务依赖 trait，不依赖 `KvCollection` 具体类型；与 `AuraStorage` trait + `FjallEngine/SlateEngine` 的装配画风统一：

```rust
/// 容器契约：K/V 编排 + 批次缓冲，引擎无关——对上层暴露的接口面
pub trait Collection<K: KeyEncode, V: ValueEncode> {
    /// 编码 → 积入内部批次（不落盘）
    fn save(&mut self, key: &K, value: &V);
    /// 点查：key.encode() → 引擎 get → decode（版本分派）
    fn find(&self, key: &K) -> Option<V>;
    /// 内部批次一次 WAL 提交，随后清空。
    /// 纪律：save 积累后必须 write——漏调即丢写，无隐式自动 flush（原子性优先于便利）。
    fn write(&mut self) -> Result<(), StorageError>;
}

/// Collection 的标准实现：泛型约束到 AuraStorage；批次缓冲与引擎句柄即全部状态
pub struct KvCollection<A: AuraStorage, K: KeyEncode, V: ValueEncode> {
    backend: A,
    batch: Vec<(Vec<u8>, Vec<u8>)>,
}

impl<A: AuraStorage, K: KeyEncode, V: ValueEncode> Collection<K, V> for KvCollection<A, K, V> {
    fn save(&mut self, key: &K, value: &V) {
        let physical_key = key.encode();
        let physical_value = value.encode_value();
        self.batch.push((physical_key, physical_value));
    }

    fn find(&self, key: &K) -> Option<V> {
        let raw = self.backend.get(&key.encode())?;
        V::decode_value(&raw).ok()
    }

    fn write(&mut self) -> Result<(), StorageError> {
        let result = self.backend.write(&self.batch);
        self.batch.clear();
        result
    }
}

/// 跨 collection 原子写：编码进外部共享批次，不触碰内部缓冲。
/// 主表 + 二级索引等必须同一次 WAL 提交的场景用它，提交交回引擎句柄。
/// （save_into 不在 trait 上——它依赖具体批次机制，属实现侧能力）
impl<A: AuraStorage, K: KeyEncode, V: ValueEncode> KvCollection<A, K, V> {
    pub fn save_into(&self, batch: &mut WriteBatch, key: &K, value: &V) {
        batch.push((key.encode(), value.encode_value()));
    }
}
```

两个设计判据：

- **接口与实现分离的分工**：`Collection` trait 是上层可见的接口面（save/find/write），`KvCollection` 是它的标准实现。换引擎（Fjall ↔ SlateDB）不动 trait；若未来出现不同容器实现（如带缓存的 Collection、只读快照 Collection），实现 trait 即可接入，上层代码零改动。
- **类型参数落在字段泛型上**（`backend: A`、方法约束 `K`/`V`），不需要空转的 `PhantomData` marker——类型锁死由泛型约束本身完成。批次缓冲内聚于实现体，`save` 多次积累、`write` 一次原子落盘；多路并发各持实例或加锁，是调用方的并发选择，不焊死在容器 API 里。

#### 跨 collection 原子提交

内部批次是各 collection 私有的，两个 collection 各自 `write()` 是两次独立 WAL 提交——主表 + 二级索引这种要求同一原子单元的场景会被它堵死（索引行与主表行必须同生同死，见 [kv-storage-engine](kv-storage-engine.md)「二级索引的更新」）。跨 collection 原子性本质是**引擎层的属性**（WAL 一次提交），不是 collection 层能造出来的——解法是回到引擎句柄，让 collection 退回它本来擅长的：只做编码：

```rust
// 主表 + 二级索引两个 collection，共享一个引擎级批次
let mut batch = WriteBatch::new();
users.save_into(&mut batch, &user_key, &user_value);
idx.save_into(&mut batch, &index_key, &EMPTY_VALUE);   // 索引行 value 留空（锚点位）
engine.write(batch)?;                                   // 一次 WAL，原子

// 两条路径的分工：
// 单 collection 常规写 → save 积内部缓冲 + write（便利路径）
// 跨 collection 原子写 → save_into 编进共享批次 + engine.write（一致性路径）
```

`save_into` 不触碰内部缓冲，两条路径互不干扰；不提供「共享缓冲」式的花活——那只是把 batch 挪个地方存，原子性还是得靠引擎一次提交。

#### 存储的落点：三层归属

实际 KV 存储不在宏层，归属自上而下三层，各管一段：

```
KeyEncode / ValueEncode 派生宏    ← 纯编译期编解码，零 I/O，不知道任何引擎
        ↓ 展开出的纯函数
Collection trait ← KvCollection  ← backend: A (AuraStorage)，唯一摸引擎的字段
        ↓ trait 分派
AuraStorage（FjallEngine / SlateEngine）   ← 真实 KV 存储在这层
```

- **宏层刻意无存储**：编码契约（字节怎么排）与引擎可替换性（字节放哪）解耦，宏才能保持单 item 纯函数。`encode()` / `encode_value()` 只做 `Vec<u8>` 进出。
- **`A: AuraStorage` 是引擎的唯一入口**，构造时注入——由上层装配处决定 Fjall 还是 SlateDB：

```rust
// 装配处：引擎选择 + 生命周期归这里管，不归宏、不归 Collection 本身
let engine: Arc<dyn AuraStorage> = match deploy_mode {
    DeployMode::Local => Arc::new(FjallEngine::open("/var/lib/aura/keyspace")?),
    DeployMode::Cloud => Arc::new(SlateEngine::open_s3("s3://aura-bucket/db").await?),
};
let mut collection = KvCollection::new(engine);

// 使用处：save 积累 → write 一次原子落盘，全链路不感知引擎差异
collection.save(&key, &value);
collection.write()?;   // 一次 WAL 提交，原子性落在引擎层
```

- **原子性落在引擎层**：`write` 把积攒的物理字节一次提交给 AuraStorage 实现体（Fjall keyspace / SlateDB 实例）；实例生命周期归装配处管理，Collection 只持句柄。

### 上层业务代码

完整链路——先由 `KeyEncode` / `ValueEncode` 派生宏生成的编码函数构造 K/V，再交给依赖 `Collection` trait 的业务函数：

```rust
// 第一步：用派生宏展开的编码器构造 K/V（纯函数，零 I/O）
fn build_kv(content: [u8; 64], score: i32, tags: Vec<String>,
            session_id: [u8; 16], timestamp: u64) -> (AgentMemoryKey, AgentMemoryValue) {
    let key = AgentMemoryKey { session_id, timestamp };   // KeyEncode：#[kv_ns(6)] 已随类型
    let value = AgentMemoryValue { content, score, tags }; // ValueEncode：#[kv_version(2)] 已随类型
    (key, value)
}

// 第二步：业务函数依赖 Collection trait——泛型分派，不感知具体容器与引擎
fn process_login<C: Collection<AgentMemoryKey, AgentMemoryValue>>(
                 collection: &mut C,
                 key: AgentMemoryKey, value: AgentMemoryValue) -> Result<(), StorageError> {
    // 零原始字节操作、零手动前缀拼接、零分区猜测
    collection.save(&key, &value);
    collection.write()?;

    println!("写入 Namespace: {}", AgentMemoryKey::NAMESPACE_ID);
    Ok(())
}

// 调用处：编码构造 → trait 分派 → 引擎落盘
let (key, value) = build_kv(content, score, tags, session_id, timestamp);
process_login(&mut collection, key, value)?;
```

### 运行时冲突检测方向

Collection 可扩展乐观冲突检测——基于 MVCC 版本号的 CAS 乐观锁重试循环，抵御多线程并发对同一记录的更新擦除：

```rust
impl<A: AuraStorage, K: KeyEncode, V: ValueEncode> KvCollection<A, K, V> {
    /// 乐观锁写入：版本号不匹配时返回 false，调用方重试
    pub fn save_with_cas(&mut self, key: &K, value: &V, expected_version: u64) -> bool {
        let physical_key = key.encode();
        // let current = self.backend.get(&physical_key)?;
        // let current_version = u64::from_be_bytes(current[0..8].try_into().unwrap());
        // if current_version != expected_version { return false; } // 版本冲突
        // 编码积入批次，write 时一并原子提交...
        true // 占位返回：实现示意省略，仅示意 CAS 成功路径
    }
}
```

## Python 侧实现：SlateDB 绑定

SlateDB 官方 Python 绑定（`pip install slatedb`，UniFFI 桥接 Rust 内核）提供了 OKM 所需的全部底层原语，OKM 范式可以在 Python 侧等价实现：

| OKM 需要的原语 | SlateDB Python API |
|:--|:--|
| 点查 | `await db.get(key)` |
| 前缀/范围扫描 | `scan_prefix` + `KeyRange` |
| 原子批量写（索引一致性） | `WriteBatch` + `db.write()` |
| 事务 | `db.begin(IsolationLevel.SERIALIZABLE_SNAPSHOT)` |

OKM 的本质是编译在 Key 字节序上的应用层范式——数字命名空间字典、复合键编码、大端序、前缀扫描语义全部作用在 `bytes` 层，与存储引擎和宿主语言无关。Rust 版的「Struct 即 DDL」在 Python 侧对应 dataclass：

```python
@dataclass(frozen=True)
class UserSessionKey:
    org_id: bytes      # 16 字节，编码时校验长度
    user_id: int       # u64 数字 ID
    session_id: bytes  # 16 字节

    def encode(self) -> bytes:
        assert len(self.org_id) == 16 and len(self.session_id) == 16
        return (
            (1).to_bytes(2, "big")             # ns=1 → b"\x00\x01"
            + self.org_id                      # 16B
            + self.user_id.to_bytes(8, "big")  # 8B u64
            + self.session_id                  # 16B
        )
```

过程宏对应物是类装饰器或 `__init_subclass__` 自动生成 encode/decode，配合 mypy/pyright 静态检查约束字段类型，Collection trait 的类型安全容器也能以泛型 `Generic[T]` 等价搭建。

### 保障级别的结构性差距

范式 100% 可移植，但保障从「编译期硬约束」降为「静态检查 + 运行时断言」：

- **类型错误后移**：Rust 版错误 Key 编译期直接报错；Python 的类型注解是工具链约束而非语言约束，绕过注解的代码照样能跑，硬保证只剩 `assert` 运行时断言
- **编码开销非零**：Rust 版编码退化为 memcpy 级指针偏移；Python 版是字节拼接 + 属性访问，慢 1–2 个数量级。KV 写路径由 I/O 主导时通常无感，极热路径有感知
- **宏机制差异**：Rust 过程宏在编译期生成代码，零运行时成本；Python 装饰器/元类是运行时注册，有一次性启动开销

判定：Python 侧等价物是「OKM 的动态语言版本」——范式相同，保障级别由宿主语言能力决定。
