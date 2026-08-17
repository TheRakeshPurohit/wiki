# OKM：Object-Keyspace Mapping（对象键空间映射）

**Status:** 独立组件
**关联文档：** [KV 存储引擎](kv-storage-engine.md) — 底层架构与设计模式
**关联文档：** [Aura 架构](aura-architecture.md) — 双引擎模式

---

OKM 是对标 ORM 的范式——ORM 将对象映射到关系表，OKM 将对象映射到 KV 键空间。通过自定义过程宏 `#[derive(KvEncode)]` + 数字命名空间 ID，构建零成本抽象语义数据层：开发侧如同 ORM 般声明式，编译后退化为纯指针偏移计算。

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

### TypedTable：泛型类型安全抽象

在裸字节 KV 引擎之上封装一层声明式泛型表，交付 ORM 级别的开发体验：

```rust
use std::marker::PhantomData;

/// 通用泛型表空间——PhantomData 编译期类型检查，运行时零开销
pub struct TypedTable<K, V> {
    _key_type: PhantomData<K>,
    _val_type: PhantomData<V>,
}

impl<K, V> TypedTable<K, V> where
    K: serde::Serialize,
    V: serde::Serialize + serde::de::DeserializeOwned
{
    /// 类型化写入——编译器保证 Key 和 Value 类型匹配
    pub fn put_record(&self, batch: &mut Vec<(Vec<u8>, Vec<u8>)>, key: K, value: V) {
        let serialized_key = bincode::serialize(&key).unwrap();
        let serialized_val = bincode::serialize(&value).unwrap();
        batch.push((serialized_key, serialized_val));
    }
}
```

`TypedTable<UserSessionKey, SessionData>` 在编译期锁死了 Key 和 Value 的类型。传错类型直接编译失败，不需要运行时调试。

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

    #[test]
    fn test_key_ddl_stability_protection() {
        // 1. 构造确定性的业务测试数据
        let test_key = UserSessionKey {
            org_id: uuid::Uuid::parse_str("67e55044-10b1-426f-9247-bb680e5fe0c8").unwrap(),
            user_id: "user_88",
            session_id: "sess_99",
        };

        // 2. 硬编码历史版本生成的真实物理二进制（hex 16 进制）
        // 任何改动前缀或字段顺序的行为都会导致编码不匹配，CI/CD 立刻报错
        let expected = hex::decode(
            "736573733a67e5504410b1426f9247bb680e5fe0c83a757365725f38383a736573735f3939"
        ).unwrap();

        assert_eq!(
            test_key.encode(), expected,
            "DDL 物理 Key 编码发生非预期漂移！将导致线上老数据失联！"
        );
    }
}
```

这段测试的物理含义：`hex::decode("736573733a...")` 是 `sess:67e55044...:user_88:sess_99` 的 UTF-8 字节序列。任何人改动 `UserSessionKey` 的 `encode()` 方法（改前缀、调字段顺序、换序列化格式），测试立刻失败，阻止合入主分支。这是 KV 系统替代 DDL 约束的最终防线——编译期类型安全 + 运行时 hex 校验，双保险。

### 判定

SQL 的核心价值是面向人类的结构化纪律。KV 只要贯彻「代码即 DDL 的强类型编码 + 多版本 Enum 懒迁移 + 双写 Key 指针契约 + TypedTable 泛型抽象 + hex 硬编码单元测试」，就同时获得了：编译期 Schema 安全（Rust 编译器）+ 运行时极致性能（LSM-Tree）+ 零停机演进（版本化 Enum）+ 团队可维护性（Struct 注释即文档）+ 编码漂移防护（hex 单元测试）。在 Schema 安全性上完成对 SurrealDB（无模式）和 PostgreSQL（运行时 DDL 锁）的双向反超。

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
#[derive(KvEncode)]
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
#[derive(KvEncode)]
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

## 过程宏实现源码

### 宏编译器驱动（`kv_codec_derive/src/lib.rs`）

```rust
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::quote;
use syn::{parse_macro_input, AttrStyle, Data, DeriveInput, Fields, Lit, Meta, Type, Expr};

#[proc_macro_derive(KvEncode, attributes(kv_ns))]
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
            _ => panic!("OKM 键空间只接受 [u8; N]、基础数值类型和 String（String 为变长字段）"),
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
                Ok(Self { #( #fields ),* })
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
```

### 运行时模块 + 测试（`kv_codec/src/lib.rs`）

```rust
pub use kv_codec_derive::KvEncode;

#[cfg(test)]
mod tests {
    use super::*;

    #[derive(KvEncode)]
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
    #[derive(KvEncode, Clone)]
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

## TypedCollection：编译期对象空间容器

OKM 的上层封装——通过 PhantomData 泛型对底层裸字节 KV 进行类型死锁，对外暴露 100% 类型安全的 ORM 级接口，运行时零开销。

### 实体声明：Key 字段 + Value 字段

结构体中一部分字段由宏提取用于编排 Key，剩下的自动作为 Value Payload：

```rust
use kv_codec::KvEncode;
use serde::{Serialize, Deserialize};

/// 声明式 DDL 契约——看起来像 ORM 实体表
#[derive(KvEncode, Serialize, Deserialize, Debug, Clone)]
#[kv_ns(1)]  // 永久死锁在 Namespace ID 1
pub struct UserSessionRecord {
    // 【KEY 区】：宏提取 → 大端序定长物理 Key
    pub org_id: [u8; 16],
    pub user_id: u64,

    // 【VALUE 区】：业务资产 → 序列化为磁盘行数据
    pub session_token: String,
    pub ip_address: u32,
    pub is_active: bool,
}
```

### EntityKeyGenerator：Key/Value 自动分离 trait

宏在编译期自动为结构体实现 `EntityKeyGenerator`，在不解构的前提下分离 Key 字段与 Value 字段：

```rust
/// 过程宏全自动生成的辅助 trait
pub trait EntityKeyGenerator {
    const NAMESPACE_ID: u16;
    fn generate_compiled_key(&self) -> Vec<u8>;
}

// 宏自动生成的实现
impl EntityKeyGenerator for UserSessionRecord {
    const NAMESPACE_ID: u16 = 1;

    fn generate_compiled_key(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(2 + 16 + 8); // 26 字节
        buf.extend_from_slice(&Self::NAMESPACE_ID.to_be_bytes()); // 2 字节命名空间
        buf.extend_from_slice(&self.org_id);                        // 16 字节
        buf.extend_from_slice(&self.user_id.to_be_bytes());        // 8 字节
        buf
    }
}
```

Key 字段（`org_id`、`user_id`）通过 `generate_compiled_key()` 编排为固定宽度二进制；Value 字段（`session_token`、`ip_address`、`is_active`）通过 `bincode::serialize()` 序列化为完整 Payload。两部分在同一个原子 Batch 内提交。

### TypedCollection 容器

→ 生产实现中 `KvBackend` 替换为 `AuraStorage` trait（见 [Aura 架构 §5.1](aura-architecture.md#51-存储引擎抽象trait-分离)），通过 `Arc<dyn AuraStorage>` 实现 Fjall/SlateDB 运行时切换。

```rust
use std::marker::PhantomData;
use std::sync::Arc;

/// 零状态、零运行时开销的编译期对象管理器
pub struct TypedCollection<T> {
    _marker: PhantomData<T>,  // 编译期类型锁死，运行时 0 字节
    pub raw_backend: Arc<dyn AuraStorage>,  // FjallEngine 或 SlateEngine
}

impl<T> TypedCollection<T> where
    T: Serialize + serde::de::DeserializeOwned + Clone + EntityKeyGenerator
{
    pub fn new(backend: Arc<dyn KvBackend>) -> Self {
        Self { _marker: PhantomData, raw_backend: backend }
    }

    /// 类 ORM 体验：将结构体原生存入系统
    pub fn save(&self, batch: &mut WriteBatch, entity: &T) {
        // 1. 宏编译期硬编码的 Key 构造器（零运行时字符串解析）
        let physical_key = entity.generate_compiled_key();
        // 2. 整个对象序列化为二进制 Value Payload
        let physical_value = bincode::serialize(entity).unwrap();
        // 3. 推入原子写入批次
        batch.push((physical_key, physical_value));
    }

    /// 类 ORM 体验：通过 Key 字段精准点查
    pub fn find_by_key(&self, key_spec: &T) -> Option<T> {
        let target_key = key_spec.generate_compiled_key();
        // let raw = self.raw_backend.get(&target_key)?;
        // let entity: T = bincode::deserialize(&raw).unwrap();
        // Some(entity)
        None  // 模拟
    }
}
```

### 上层业务代码的极致清爽

```rust
fn process_login(collection: &TypedCollection<UserSessionRecord>, data: UserSessionRecord) {
    let mut batch = Vec::new();

    // 零原始字节操作、零手动前缀拼接、零恶心分隔符字符串
    collection.save(&mut batch, &data);

    println!("写入 Namespace: {}", UserSessionRecord::NAMESPACE_ID);
}
```

开发人员面对的是纯粹的 Rust 领域模型。宏在编译期把所有类型信息蒸发，直接打穿底层 KV 引擎的硬件极限性能。上层是 ORM 般的声明式体验，底层是裸字节级的指针偏移计算。

### 运行时冲突检测方向

TypedCollection 可以扩展乐观冲突检测——基于 MVCC 版本号的 CAS 乐观锁重试循环，抵御多线程并发对同一 Session 的更新擦除：

```rust
impl<T> TypedCollection<T> where T: EntityKeyGenerator {
    /// 乐观锁写入：版本号不匹配时自动重试
    pub fn save_with_cas(&self, batch: &mut WriteBatch, entity: &T, expected_version: u64) -> bool {
        let key = entity.generate_compiled_key();
        // let current = self.raw_backend.get(&key)?;
        // let current_version = u64::from_be_bytes(current[0..8]);
        // if current_version != expected_version { return false; } // 版本冲突
        // 写入新版本...
        true
    }
}
```

## Openraft 状态机集成

OKM 与 Raft 共识层的集成点——状态机将 Raft 日志条目 Apply 到 Fjall，实现本地物理盘的分布式一致性写入。SlateDB 模式不需要 Raft——S3 自身提供 HA，计算节点无状态。

### 核心结构

```rust
use std::sync::Arc;
use openraft::{RaftStateMachine, StateMachineData, StateMachineError, LogId};
use fjall::{Config, Keyspace, PartitionHandle};
use serde::{Serialize, Deserialize};

/// Raft 状态转换指令——所有写操作必须通过此枚举广播
#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum RaftCommand {
    UpdateActorState {
        agent_id: String,
        serialized_context: Vec<u8>,  // CBOR 编码的 Actor 状态快照
    },
    TerminateActor {
        agent_id: String,
    },
}

/// 分布式状态机——内嵌 Fjall 物理分区
pub struct FjallStateMachine {
    pub keyspace: Keyspace,
    pub actor_partition: PartitionHandle,  // Actor 状态分区
    pub meta_partition: PartitionHandle,   // Raft 元数据分区（last_log_id 等）
}
```

### 初始化

```rust
impl FjallStateMachine {
    pub fn new(path: &str) -> Self {
        let keyspace = Keyspace::open_default(path)
            .expect("无法初始化 Fjall 物理存储区");
        let actor_partition = keyspace.open_partition("actors", Config::default())
            .expect("无法开辟智能体物理状态分区");
        let meta_partition = keyspace.open_partition("meta", Config::default())
            .expect("无法开辟集群元数据分区");
        Self { keyspace, actor_partition, meta_partition }
    }
}
```

### RaftStateMachine trait 实现

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct StateMachineResponse {
    pub success: bool,
}

impl StateMachineData for FjallStateMachine {}

impl RaftStateMachine<openraft::TypeConfig> for Arc<FjallStateMachine> {
    /// 从 meta_partition 读取上一次成功执行的 Raft 日志 ID，防止状态漂移
    async fn applied_state_id(
        &self,
    ) -> Result<Option<LogId<u32>>, StateMachineError<openraft::TypeConfig>> {
        if let Ok(Some(bytes)) = self.meta_partition.get("last_log_id") {
            let log_id = postcard::from_bytes(&bytes).unwrap();
            Ok(Some(log_id))
        } else {
            Ok(None)
        }
    }

    /// 核心 Apply 逻辑：Raft 日志 → Fjall 物理写入
    async fn apply<I>(
        &self,
        entries: I,
    ) -> Result<Vec<StateMachineResponse>, StateMachineError<openraft::TypeConfig>>
    where
        I: IntoIterator<Item = openraft::Entry<openraft::TypeConfig>> + Send,
    {
        let mut responses = Vec::new();

        for entry in entries {
            let log_id = entry.log_id;
            if let openraft::EntryPayload::Normal(cmd) = entry.payload {
                match cmd {
                    RaftCommand::UpdateActorState { agent_id, serialized_context } => {
                        // Fjall 单次写入，自带崩溃保护
                        self.actor_partition.insert(
                            agent_id.as_bytes(), serialized_context
                        ).expect("Fjall 本地物理写入异常");
                        responses.push(StateMachineResponse { success: true });
                    }
                    RaftCommand::TerminateActor { agent_id } => {
                        self.actor_partition.remove(agent_id.as_bytes())
                            .expect("Fjall 本地擦除异常");
                        responses.push(StateMachineResponse { success: true });
                    }
                }
            }

            // 实时固化最新 LogId 到元数据分区
            let log_bytes = postcard::to_stdvec(&log_id).unwrap();
            self.meta_partition.insert("last_log_id", log_bytes).unwrap();
        }

        // 强制刷盘到 NVMe——Raft 共识的持久化保证
        self.keyspace.persist().unwrap();
        Ok(responses)
    }
}
```

### 物理保证

- **原子性**：Fjall 的 `keyspace.persist()` 保证一次 apply 内所有写入原子落盘
- **崩溃恢复**：重启后从 `meta_partition` 读取 `last_log_id`，重放未 Apply 的 Raft 日志
- **错误保护**：Fjall 写入失败触发 panic（系统级保护），不会静默丢数据
- **与 OKM 的关系**：`RaftCommand::UpdateActorState` 的 `serialized_context` 可以是 OKM 编码的实体——`AuraCollection.save()` 生成的 `(key, value)` 对通过 Raft 广播后，状态机将其 Apply 到 Fjall。SlateDB 模式下直接走 SlateDB async API 写入 S3，无需 Raft 状态机。
