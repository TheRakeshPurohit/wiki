# 结构化日志设计

## 为什么不用 Elasticsearch

### 从 grep 到 ES：同一个错误的放大

早期的日志分析模式是 lamp 时代的**文本 + grep**——日志写成纯文本，用正则搜索。随着日志量增长，这一模式暴露出根本性的局限。但 ES 的做法是：把文本 + grep 的思路原封不动搬到了分布式集群上。这不是解决，是**天真地放大**。

业界很多人实际上没有仔细考虑过这个问题。当 grep 无法应对日志规模时，直觉反应是建立索引——然后选了 ES，因为 Elasticsearch 具备搜索能力。但 ES 的倒排索引是为**全文搜索**设计的（搜一篇论文里提到"量子"的所有段落），不是为**结构化查询**设计的（查某个 user_id 在某个时间窗口内的错误日志）。

### 倒排索引存日志是错误的抽象

ES 的核心是倒排索引——为每个 token 建立"文档→位置"的映射。这个结构对日志来说是错配的。

**日志的"停用词"问题极其严重。** 日志中的高频词（`conn`、`request`、`handler`、`POST`、`200`）会命中大量记录。搜一个 `conn`，可能返回一半的日志。这在 ES 的倒排索引里是正常行为——它忠实地执行了你指定的查询。但对日志分析来说，这是灾难：

```
# 你想找：数据库连接超时
# ES 返回：所有包含 "conn" 的日志（10 万条）
# 其中真正相关的：3 条
```

你需要不断加限定条件来缩小范围，最终写的查询比直接 grep 还复杂。

用倒排索引存日志，和用全内存缓存（[Redis](redis-critique.md)）一样颟顸而荒谬——近乎烧纸币取暖。（Redis 作为缓存同样违反存储器金字塔第一原理：全内存拒绝分层，把所有温度的数据压在同一层昂贵介质上。详见 [Redis 批判 §5](redis-critique.md)。）

### ES 太臃肿，尤其 ELK

Elasticsearch + Logstash + Kibana（ELK）是三个独立系统：

- **Logstash**：用 JVM 跑一个数据管道，本质是正则替换
- **Elasticsearch**：用 JVM 跑一个搜索引擎，索引你不需要的倒排
- **Kibana**：用 Node.js 跑一个 Web UI，画你不需要的图表

三个 JVM 进程，三套配置，三套运维。一个中等规模的日志集群（每天几十 GB），ES 需要 3+ 节点、每节点 16GB+ 内存、堆内存配置、GC 调优、索引生命周期管理、模板、映射、分片策略。而同样的数据量，PostgreSQL 的 jsonb 列一张表即可满足，单节点，内存需求小一个数量级。

### 索引推断的陷阱

ES 会自动推断字段类型并建索引。看似方便，实则危险——它试图**取各家之短**，结果哪家的长处都没拿到：

- 一个 `user_id` 字段被推断为 `text`，自动分词，搜 `u_abc` 会命中 `u_abc123` 和 `u_abc456`
- 一个 `duration_ms` 被推断为 `long`，无法做范围聚合的直方图
- 一个嵌套的 `metadata` 对象被拍平，丢失结构

你必须显式定义 mapping，但大多数人不会这么做——因为他们选择了 ES 就是为了"不用定义 schema"。这是一个自我矛盾。

## 核心原则

**语义清晰，存储免解析，查询精确。**

日志不是给人读的文本流，是给程序消费的结构化事件。每条日志是一个自描述的数据记录：

- **语义清晰**：字段有明确含义，不需要猜测上下文
- **存储免解析**：写入数据库时直接插入，不需要正则提取、不需要 Grok、不需要 Logstash 管道
- **查询精确**：按字段精确匹配，不是全文模糊搜索

## 日志格式

每条日志是一个 JSON 对象，由 structlog 生成：

```json
{
  "event": "engine.db-restore-context",
  "level": "info",
  "timestamp": "2026-08-07T14:30:00.123Z",
  "user_id": "u_abc123",
  "session_id": "s_xyz789",
  "duration_ms": 42.3,
  "message_count": 12
}
```

### 字段规范

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `event` | string | ✅ | 模块前缀.事件名，如 `engine.db-restore-context` |
| `level` | string | ✅ | 日志级别：debug / info / warning / error |
| `timestamp` | string | ✅ | ISO 8601 UTC 时间戳 |
| `user_id` | string | ⚠️ | 用户标识，用户相关日志必须带 |
| `session_id` | string | ⚠️ | 会话标识，用户相关日志必须带 |

其余字段为事件特定上下文，自由扩展。

### 事件命名

采用 `模块.动作-限定词` 格式，由 structlog processor 自动注入模块前缀：

```
engine.db-restore-context     ← skillforge.engine 模块
server.agent-run-start        ← skillforge.server 模块
infra.script-exec-start       ← skillforge.infra 模块
```

模块名从 `structlog.get_logger(__name__)` 的 logger name 自动提取，无需手动维护。

## 推荐的存储方案

核心要求：**支持半结构化数据**（jsonb / variant / JSON 类型），主流关系型数据库都可以。

### 第一选择：PostgreSQL (jsonb)

PostgreSQL 的 jsonb 是日志存储的理想选择：

- **写入免解析**：JSON 对象直接插入，不需要预处理
- **查询精确**：按任意字段精确匹配，支持 GIN 索引加速
- **半结构化**：嵌套对象、数组、混合类型全部支持
- **主流成熟**：运维团队都熟悉，不需要额外学习

```sql
CREATE TABLE logs (
    id         bigserial PRIMARY KEY,
    ts         timestamptz NOT NULL DEFAULT now(),
    event      text NOT NULL,
    level      text NOT NULL,
    user_id    text,
    session_id text,
    data       jsonb NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_logs_event ON logs (event);
CREATE INDEX idx_logs_user ON logs (user_id, ts DESC);
CREATE INDEX idx_logs_session ON logs (session_id, ts DESC);
CREATE INDEX idx_logs_ts ON logs (ts DESC);
CREATE INDEX idx_logs_data ON logs USING gin (data);
```

查询示例：

```sql
-- 某用户最近的错误日志
SELECT * FROM logs
WHERE user_id = 'u_abc123' AND level = 'error'
ORDER BY ts DESC LIMIT 50;

-- 某个事件的 P99 耗时
SELECT percentile_cont(0.99) WITHIN GROUP (ORDER BY (data->>'duration_ms')::numeric)
FROM logs
WHERE event = 'engine.db-restore-context'
  AND ts > now() - interval '1 hour';

-- jsonb 按任意键查询
SELECT * FROM logs
WHERE data @> '{"model": "gpt-4"}';
```

### 第二选择：ClickHouse

适合日志量极大（每天 TB 级）的场景。列式存储 + 向量化执行，聚合查询极快。

```sql
CREATE TABLE logs (
    ts         DateTime64(3, 'UTC'),
    event      LowCardinality(String),
    level      LowCardinality(String),
    user_id    Nullable(String),
    session_id Nullable(String),
    data       String  -- JSON 字符串
) ENGINE = MergeTree()
ORDER BY (event, ts)
PARTITION BY toYYYYMM(ts);
```

### 第三选择：JSONL 文件

最简单的方案，适合开发和小规模部署。查询用 `duckdb` 直接查：

```bash
duckdb -c "SELECT * FROM read_json('logs/*.jsonl') WHERE event = 'engine.db-restore-context'"
```

### Lakehouse

同样可行，但需要持久服务进行攒批写入，否则存在数据丢失风险。适合已有 Lakehouse 基础设施的团队。

### 不推荐的方案

| 方案 | 问题 |
|------|------|
| Elasticsearch | 倒排索引与日志查询模式不匹配，资源消耗大，运维复杂 |
| MongoDB | 介于关系型和文档型之间，查询能力不如 PostgreSQL jsonb |
| Loki | 本质是 grep 的云端版，标签索引 + 全文扫描，量大时慢 |

## 设计决策

### 为什么模块前缀自动注入

手动维护事件名前缀（如 `engine.db-save-context`）容易遗漏且不可维护。通过 structlog 的 `__name__` + processor 自动注入：

```python
# 每个模块创建自己的 logger
logger = structlog.get_logger(__name__)

# event 名保持简洁
logger.info("db-restore-context", session_id=sid, duration_ms=42)

# processor 自动注入前缀，输出：
# event = "engine.db-restore-context"
```

### 为什么用 UTC 时间戳

- 避免时区转换的歧义
- 排序天然正确
- 存储和传输无损

### 为什么不用结构化日志库的 JSON Schema

JSON Schema 验证日志格式会增加运行时开销。结构化日志的核心价值在于"字段是可预测的"，而不是"字段是被强制约束的"。通过命名规范（`event`、`level`、`timestamp`）和类型约定（统一使用 ISO 时间戳、数字用 `duration_ms` 而非 `duration`）来保证一致性。

## 交叉引用

- [KV 存储引擎架构](kv-storage-engine.md) — 嵌入式存储引擎选型
- [Lakehouse 研究](lakehouse-research.md) — Lakehouse 架构与日志攒批
- [统一数据层](unified-data-layer.md) — 多模态数据存储分析
- [Arrow HTAP 引擎](arrow-unified-htap-engine.md) — 列式分析与日志聚合
- Aura 架构 [§6.15 MQ 分解架构](aura-architecture.md#615-mq-分解架构) — 日志作为边界事件的存储模式
