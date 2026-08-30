# SQL/PGQ —— 属性图查询

*SQL Property Graph Queries 在关系存储之上的落地设计。* 本文讲"图如何在关系库里定义、查询、演进"，与[图谱化记忆](graph-memory.md)互为表里：那是范式（为什么用图），这是载体（在 Postgres 栈里怎么把图建起来）。

## 一、SQL/PGQ 是什么

SQL/PGQ（ISO/IEC 9075-16）是 **SQL 标准的属性图查询规范**，不是某个数据库厂商的私有语法。它定义了如何在**关系表之上声明一张属性图**、用图匹配语法查询。

关键事实（常被误解）：
- **它不绑定 PostgreSQL**——Oracle 有完整实现，PostgreSQL 在推进中但仅实验性支持。
  "PGQ" 的 PG 指 property graph，不是 PostgreSQL。
- **图不是新存储**——是现有表的**投影视图**。用户表、组织表、边表就是图的顶点表和
  边表，`CREATE PROPERTY GRAPH` 只是一层逻辑声明，数据仍在原表。
- **图不等价于图数据库**——爱图数据库（Neo4j 等）是独立引擎；SQL/PGQ 是对关系库的
  查询扩展，让它能用图的方式遍历。

## 二、图怎么定义：在表上声明，不复制数据

```sql
CREATE PROPERTY GRAPH org_graph
  VERTEX TABLES (
    users           KEY id,
    organizations   KEY id,
    skills          KEY id
  )
  EDGE TABLES (
    org_membership KEY id
      SOURCE KEY(org_id) REFERENCES organizations(id)
      DESTINATION KEY(user_id) REFERENCES users(id),
    org_hierarchy KEY id
      SOURCE KEY(parent_org_id) REFERENCES organizations(id)
      DESTINATION KEY(org_id) REFERENCES organizations(id),
    user_relation KEY id
      SOURCE KEY(user_a) REFERENCES users(id)
      DESTINATION KEY(user_b) REFERENCES users(id)
  );
```

图像层只负责"怎么查"；**"谁能写"回到底层表权限**。这带来一个直接推论：需要严格保护的边（如用户-组织成员关系），在**边表上**用 FK + 表权限锁死写入，不开放给 LLM/Agent，图的声明对此无副作用。

## 三、核心设计判据

### 3.1 一图多关系（每个图不是只能一种关系)

一张图可以包含**多个顶点表 + 多个边表（多种关系）**。`org_graph` 里 membership、hierarchy、relation 是三种不同关系，都在同一张图内。图匹配语法一条查询可遍历多种边：

```sql
-- 找 user 所在子组织能触达的所有 skill（沿 membership + hierarchy 混合遍历）
SELECT DISTINCT s.name
FROM GRAPH_TABLE (org_graph
  MATCH (u:users)-[:org_membership]->(o:organizations)
       -[:org_hierarchy]->(o2:organizations)
       -[:skill_scope]->(s:skills)
  WHERE u.id = ?)
AS gt;
```

### 3.2 多张图的判据：能合不拆

| 判据 | 决策 |
|---|---|
| 同一信任域、关系语义相关 | **合成一张图** |
| 语义域彻底分离且从不相交查询 | 才拆多张图 |

典型例子：**身份-组织图**（users/orgs/skills/membership/hierarchy/relation）负责权限可达性；**记忆知识图**（三元组）负责语义检索——两者域不同，各自一张图。

### 3.3 跨图查询：标准不便，实践靠底层表

SQL/PGQ 标准**不提供友好的跨图查询**——图匹配一次作用于单个已命名图，无法优雅地在一条查询里从图 A 跳到图 B。但因为图只是表的投影，解法是：
- 同域多关系 → 合成一张图，图内一条匹配搞定（方便）。
- 跨域 → 不用图语法，用**底层表 JOIN** 或先查图得 id 集合再查另一侧。

**一句话：别为了"跨图"硬拆；同一信任域合并成一张图，真跨域的走底层表。**

**支撑机制（PG 官方 `CREATE PROPERTY GRAPH`）：**

- **一张表可属于多个图**——图不物理物化，是视图式投影声明（文档原话："similar to CREATE VIEW... used only when queried"）。同一底层表可被任意多个图引用，互不占为己有。据此可**按范围建层级图**：同一批基础表，声明一个子范围图只引用其中几张、再声明一个全范围图引用全部——共享底层表，层级由"图引用的表子集"表达，无需复制数据。
- **图不能引用图**——`CREATE PROPERTY GRAPH` 的入参只有表（vertex/edge table），无"以图为构件"的语法。要把多图"合并"成一张，只能新建一张更大的图、把各子图用到的表再列一遍。合并的原子是表，不是图。这与 SQL 视图可层叠（视图引用视图）不同：PGQ 图是"表之上的一层"，不是"图之上的一层"。

两条合起来：跨域想拆多图再组合做不到图上组合，所以更应同域合并成一张图、真跨域走底层表——这正是上面"一句话"的结构性根据，而非偏好。

## 四、权限可达性落地

在图谱化记忆里，权限 = 搜索时的图可达性剪枝。SQL/PGQ 就是这条判据的标准查询实现——以 PGQ 为准架一张身份-组织图，用图匹配语法表达可达性：

```sql
-- 从 user 出发，沿 org_membership + org_hierarchy 遍历到可达 skill
SELECT DISTINCT s.name
FROM GRAPH_TABLE (org_graph
  MATCH (u:users)-[:org_membership]->(o:organizations)
       -[:org_hierarchy]->(o2:organizations)
       -[:skill_scope]->(s:skills)
  WHERE u.id = ?)
AS gt;
```

以 PGQ 为标准;若某部署环境暂无法启用 PGQ 引擎，用 DuckDB 作为备选承载同一批表、以成 PGQ 语义的图查询。图声明（`CREATE PROPERTY GRAPH`）在任何 PGQ 兼容引擎上等价，表结构一旦按图设计即可跨引擎迁移。

**设计要点**：不建多张图（身份-组织一张）；一图多关系；跨域不走图语法走底层表；
用户-组织边用表权限锁死人工；严格性按边分。

## 五、严格性按边分（用户-组织的关系）

`org_membership` 是信任边界，必须**只允许人工（或经严格审批流程）写入**，不开放 Agent 写。用 SQL 结构性保证：
- 底层表权限：Agent 连接账号对此表只有 SELECT，无 INSERT/UPDATE/DELETE。
- FK 保证端点必须是真实存在的用户/组织。
- 而组织层级、用户关系是软性的，若要演进/自动维护，这层才开放给系统。

**严格性按边分**——同一张图里不同边可以有完全不同的写入控制，图像层声明只决定查询。

## 六、与图数据库 / 递归 CTE 的关系

| 方式 | 本质 | 场景 |
|---|---|---|
| 图数据库（Neo4j） | 独立引擎，原生图存储 | 重图分析、需要图算法 |
| SQL/PGQ | 关系表之上的查询扩展 | 已有关系数据，想用图方式查 |
| 递归 CTE | 纯 SQL，手写遍历 | 简单可达性/路径，小规模 |

SQL/PGQ 是"关系库 + 图查询"的甜点位：不引入独立引擎，又比手写递归 CTE 表达力强。以 PGQ 为标准；DuckDB 作为备选引擎承载同一批表，图声明跨引擎等价，按图设计的表结构可平滑迁移。

**图数据模型分类归位**：上面三种只是"承载方式"差异，底层共享同一套"图语义"光谱。图模型按"节点/边是否带属性"与"二元/多元边"两个轴分层：

| 模型 | 节点 | 边 | 属性层 | 二元边 | 代表 |
|:---|:---|:---|:---|:---|:---|
| 简单图 | ✓ | ✓ | ✗（纯拓扑） | ✓ | 图算法底层（PageRank 等） |
| 标签图 | ✓ | ✓ | 仅边类型标签 | ✓ | — |
| RDF 三元组 | ✓（subject/object） | ✓（谓词即边） | 无键值层，属性编码为边 | ✓ | 语义网 / 知识图谱 |
| **属性图 (PG)** | ✓ | ✓ | **节点 + 边都可带键值属性** | ✓ | Neo4j、SurrealDB、**PGQ**、混合存储 |
| 超图 | ✓ | ✓（可连 >2 节点） | ✓ | ✗（n 元边） | 高阶关系建模 |

**PGQ 落在"属性图"这一格**——`CREATE PROPERTY GRAPH` 明确给边声明 LABEL 和 PROPERTIES（§二）；它走的是"属性图"语义，不是 RDF 三元组。**记忆知识图**（三元组 `(s,p,o)`）则是 RDF 风格——关系完全开放、无预定义标签、属性零声明，正是上表另一格。这就是本文档 §七"权限图（属性图 / 关系封闭）vs 记忆图（三元组 / 关系开放）"两格分治的语义根据。

**"是否属性图"不是优劣，是语义层归属**：Neo4j/SurrealDB/PGQ 共享属性图语义，分野在承载方式（物化存储 vs 关系表投影）；三元组图是把一切都表达为边、不设属性容器。本文档所有 PGQ 用法都在属性图语义内。

## 七、关系类型的动态化

`CREATE PROPERTY GRAPH` 声明的是**边表（一种关系类型）**，不是每条边。关系类型一旦声明，之后加多少个实例都是纯 INSERT，不需改图定义——**实例层天然动态**。真正需要动图定义的，只有"引入一种全新关系类型"这一种情况。

是否麻烦，取决于关系类型集是封闭还是开放：

| 图 | 关系类型 | 动态化方式 | 预定义成本 |
|---|---|---|---|
| 身份-组织权限图 | 封闭小集合（membership/hierarchy/relation/skill_scope） | 稳定类型 + 实例 INSERT | 一次性几十行 DDL，可接受 |
| 记忆知识图 | 完全开放（三元组 `(s,p,o)` 任意） | relation 作行内数据，零定义 | 零 |

若要在权限图内也获得"关系类型动态涌现"，用**通用边表 + 多 label**：

```sql
CREATE TABLE edges (
  id BIGINT KEY DEFAULT,
  src_type TEXT, src_id BIGINT,
  rel      TEXT,              -- 关系类型作行内数据
  dst_type TEXT, dst_id BIGINT
);

CREATE PROPERTY GRAPH g
  VERTEX TABLES (users KEY id, orgs KEY id, skills KEY id, ...)
  EDGE TABLE edges
    KEY (src_type, src_id) REFERENCES ...
    DESTINATION KEY (dst_type, dst_id) REFERENCES ...
    LABEL (rel);              -- 关系类型变成动态 label
```

加新关系 = INSERT 一行（rel 列填新值），不用建表、不用改图。但注意边界：PGQ 的 label 集合仍需在某种程度上可预期（图匹配 `[:rel]` 要写全）；若 rel 是完全开放的涌现值，图匹配语法写不全，就回到"按 rel 列谓词过滤"——那是纯三元组表的地盘。

**结论：按图定动态度。** 权限图关系类型封闭，方案 A（通用边表 + 多 label）恰到好处，预定义一次几十行 DDL 可接受；记忆图关系开放，用纯三元组 `(s,p,o)`，零定义。若要在单图内做到完全显式动态（无任何声明），SQL/PGQ 本性不支持——类型是列约束，那正是 DuckDB/纯三元组的兜底位置。

## 交叉引用

- [图谱化记忆](graph-memory.md) —— 建模范式：为什么用图、规模判据、涌现式 Skill。
- [Agent 记忆选型](agent-memory.md) —— 记忆系统存储选型的完整性论证。