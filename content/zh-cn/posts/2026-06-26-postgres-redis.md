+++
title= "PostgreSQL可以做到Redis绝大部分功能"
description= "Replaced Redis with PostgreSQL"
date = "2026-06-26"
tags = [
    "database",
    "postgres",
    "redis"
]
categories = [
  "技术"
]
series = [
  "数据库"
]
+++

# PostgreSQL可以做到Redis绝大部分功能

我之前用的是一套很典型的Web应用技术栈：

- PostgreSQL负责持久化数据存储
- Redis负责缓存、发布订阅以及后台任务处理

两个数据库，两个体系需要管理，也意味着多了两处故障风险点。

后来我意识到：PostgreSQL可以做到Redis能做的功能。

于是我彻底移除了Redis，迁移过程是这样的。

## 一、设置：我使用Redis的目的

在替换之前，Redis主要处理三件事：

**1、缓存（使用率70%）**

```go
// Cache API responses
data, _ := json.Marshal(user)
rdb.Set(ctx, fmt.Sprintf("user:%d", id), data, 3600*time.Second)
```

**2、发布订阅（使用率20%）**

```go
// Real-time notifications
msg, _ := json.Marshal(map[string]interface{}{"userId": userId, "message": message})
rdb.Publish(ctx, "notifications", msg)
```

**3、后台消息队列（使用率10%）**

```go
// Using asynq (Redis-based task queue)
task := asynq.NewTask("send-email", []byte(`{"to":"user@example.com","subject":"Hi"}`))
client.Enqueue(task)
```

**痛点：**

- 需要备份两个数据库,当然好的设计是redis不持久化,假定为不可靠.
- Redis使用内存（规模化时成本很高）
- Redis持久化机制……很复杂,而且也不可靠
- Postgres和Redis之间还存在一次网络跳转开销

## 二、我为什么考虑替换Redis

**原因一：成本**

**我的Redis配置：**

- AWS ElastiCache（2GB）：每月45美元
- 若扩容至 5GB，每月费用将增至110美元

**PostgreSQL：**

- 已付费使用RDS（20GB存储）：每月50美元
- 即便增加5GB数据流量：每月仅需0.5美元

**节省成本：** 每月约100美元

**原因二：运行复杂性**

**使用 Redis：**

```
Postgres backup ✅
Redis backup ❓ (RDB? AOF? Both?)
Postgres monitoring ✅
Redis monitoring ❓
Postgres failover ✅
Redis Sentinel/Cluster ❓
```

**不使用Redis：**

```
Postgres backup ✅
Postgres monitoring ✅
Postgres failover ✅
```

系统依赖组件更少。

**原因三：数据一致性**

**经典问题：**

```go
// Update database
_, err := db.Exec(ctx, "UPDATE users SET name = $1 WHERE id = $2", name, id)

// Invalidate cache
rdb.Del(ctx, fmt.Sprintf("user:%d", id))

// ⚠️ What if Redis is down?
// ⚠️ What if this fails?
// Now cache and DB are out of sync
```

在PostgreSQL中，**这类问题通过事务即可解决。**

## 三、PostgreSQL特性

### 1、使用非日志表进行缓存

**Redis：**

```go
data, _ := json.Marshal(sessionData)
rdb.Set(ctx, "session:abc123", data, 3600*time.Second)
```

**PostgreSQL：**

```sql
CREATE UNLOGGED TABLE cache (
  key TEXT PRIMARY KEY,
  value JSONB NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_cache_expires ON cache(expires_at);
```

**插入：**

```sql
INSERT INTO cache (key, value, expires_at)
VALUES ($1, $2, NOW() + INTERVAL '1 hour')
ON CONFLICT (key) DO UPDATE
  SET value = EXCLUDED.value,
      expires_at = EXCLUDED.expires_at;
```

**读：**

```sql
SELECT value FROM cache
WHERE key = $1 AND expires_at > NOW();
```

**清理（定期运行）：**

```sql
DELETE FROM cache WHERE expires_at < NOW();
```

**什么是非日志表？**

- 跳过预写式日志（WAL）
- 写入性能大幅提升
- 崩溃后数据不保留（非常适合用作缓存！）

**表现：**

```
Redis SET: 0.05ms
Postgres UNLOGGED INSERT: 0.08ms
```

用作缓存已经完全够用。

### 2、基于LISTEN或NOTIFY实现发布订阅功能

接下来就精彩了。

PostgreSQL具有原生的发布订阅功能，但大多数开发人员并不了解。

**1）Redis的发布订阅功能**

```go
// Publisher
msg, _ := json.Marshal(map[string]interface{}{"userId": 123, "msg": "Hello"})
rdb.Publish(ctx, "notifications", msg)

// Subscriber
pubsub := rdb.Subscribe(ctx, "notifications")
defer pubsub.Close()
ch := pubsub.Channel()
for msg := range ch {
    fmt.Println(msg.Payload)
}
```

**2）PostgreSQL的发布订阅功能**

```sql
-- Publisher
NOTIFY notifications, '{"userId": 123, "msg": "Hello"}';
```

```go
// Subscriber (Go with pgx)
conn, _ := pgx.Connect(ctx, os.Getenv("DATABASE_URL"))
defer conn.Close(ctx)

_, err := conn.Exec(ctx, "LISTEN notifications")

for {
    notification, err := conn.WaitForNotification(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Payload:", notification.Payload)
}
```

**性能对比：**

```
Redis pub/sub latency: 1-2ms
Postgres NOTIFY latency: 2-5ms
```

**性能略低，但优势明显：**

- 无需额外部署中间件
- 可在事务中使用
- 可与查询语句结合使用

**3）实际应用场景：实时日志追踪**

在我的日志管理应用中，需要实现**日志实时流式推送**。

**使用Redis：**

```go
// When new log arrives
_, err := db.Exec(ctx, "INSERT INTO logs ...")
msg, _ := json.Marshal(logEntry)
rdb.Publish(ctx, "logs:new", msg)

// Frontend listens
pubsub := rdb.Subscribe(ctx, "logs:new")
ch := pubsub.Channel()
```

**问题：** 有两个操作，如果发布失败怎么办？

**使用PostgreSQL：**

```sql
CREATE FUNCTION notify_new_log() RETURNS TRIGGER AS $$
BEGIN
  PERFORM pg_notify('logs_new', row_to_json(NEW)::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER log_inserted
AFTER INSERT ON logs
FOR EACH ROW EXECUTE FUNCTION notify_new_log();
```

现在整个操作是**原子性**的：插入数据与通知推送，要么同时生效，要么都不执行。

```go
// Frontend (via SSE)
func logsStreamHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := pool.Acquire(r.Context())
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer conn.Release()

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    conn.Exec(r.Context(), "LISTEN logs_new")

    for {
        notification, err := conn.Conn().WaitForNotification(r.Context())
        if err != nil {
            break
        }
        fmt.Fprintf(w, "data: %s\n\n", notification.Payload)
        w.(http.Flusher).Flush()
    }
}
```

**结果：** 无需Redis即可实现实时日志流传输。

### 3、基于SKIP LOCKED实现任务队列

**Redis（使用asynq）：**

```go
task := asynq.NewTask("send-email", []byte(`{"to":"user@example.com","subject":"Hi"}`))
client.Enqueue(task)

// Worker
mux.HandleFunc("send-email", func(ctx context.Context, t *asynq.Task) error {
    var payload EmailPayload
    json.Unmarshal(t.Payload(), &payload)
    return sendEmail(payload)
})
```

**PostgreSQL：**

```sql
CREATE TABLE jobs (
  id BIGSERIAL PRIMARY KEY,
  queue TEXT NOT NULL,
  payload JSONB NOT NULL,
  attempts INT DEFAULT 0,
  max_attempts INT DEFAULT 3,
  scheduled_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_jobs_queue ON jobs(queue, scheduled_at) 
WHERE attempts < max_attempts;
```

**入队：**

```sql
INSERT INTO jobs (queue, payload)
VALUES ('send-email', '{"to": "user@example.com", "subject": "Hi"}');
```

**工作进程（出队）：**

```sql
WITH next_job AS (
  SELECT id FROM jobs
  WHERE queue = $1
    AND attempts < max_attempts
    AND scheduled_at <= NOW()
  ORDER BY scheduled_at
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE jobs
SET attempts = attempts + 1
FROM next_job
WHERE jobs.id = next_job.id
RETURNING *;
```

**神奇之处：** FOR UPDATE SKIP LOCKED

这让PostgreSQL**成为了无锁队列**：

- 多个工作进程可并发拉取任务
- 任务不会被重复处理
- 若工作进程崩溃，任务会自动重新变为可执行状态

**表现：**

```
Redis BRPOP: 0.1ms
Postgres SKIP LOCKED: 0.3ms
```

**对于大多数业务负载而言，性能差异可以忽略不计。**

### 4、限流

**Redis（经典限流方案）：**

```go
key := fmt.Sprintf("ratelimit:%d", userId)
count, _ := rdb.Incr(ctx, key).Result()
if count == 1 {
    rdb.Expire(ctx, key, 60*time.Second)
}
if count > 100 {
    return errors.New("rate limit exceeded")
}
```

**PostgreSQL：**

```sql
CREATE TABLE rate_limits (
  user_id INT PRIMARY KEY,
  request_count INT DEFAULT 0,
  window_start TIMESTAMPTZ DEFAULT NOW()
);

-- Check and increment
WITH current AS (
  SELECT 
    request_count,
    CASE 
      WHEN window_start < NOW() - INTERVAL '1 minute'
      THEN 1  -- Reset counter
      ELSE request_count + 1
    END AS new_count
  FROM rate_limits
  WHERE user_id = $1
  FOR UPDATE
)
UPDATE rate_limits
SET 
  request_count = (SELECT new_count FROM current),
  window_start = CASE
    WHEN window_start < NOW() - INTERVAL '1 minute'
    THEN NOW()
    ELSE window_start
  END
WHERE user_id = $1
RETURNING request_count;
```

**或者用窗口函数更简单：**

```sql
CREATE TABLE api_requests (
  user_id INT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Check rate limit
SELECT COUNT(*) FROM api_requests
WHERE user_id = $1
  AND created_at > NOW() - INTERVAL '1 minute';

-- If under limit, insert
INSERT INTO api_requests (user_id) VALUES ($1);

-- Cleanup old requests periodically
DELETE FROM api_requests WHERE created_at < NOW() - INTERVAL '5 minutes';
```

**Postgres的适用场景：**

- 需要基于复杂业务逻辑做限流（而非仅简单计数）
- 希望限流数据与业务逻辑在同一事务中处理

**Redis的适用场景：**

- 需要亚毫秒级限流
- 极高吞吐量（每秒数百万请求）,此时应该要做网关层面的限流了.

### 5、基于JSONB实现会话存储

**Redis：**

```go
data, _ := json.Marshal(sessionData)
rdb.Set(ctx, fmt.Sprintf("session:%s", sessionId), data, 86400*time.Second)
```

**PostgreSQL：**

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  data JSONB NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_sessions_expires ON sessions(expires_at);

-- Insert/Update
INSERT INTO sessions (id, data, expires_at)
VALUES ($1, $2, NOW() + INTERVAL '24 hours')
ON CONFLICT (id) DO UPDATE
  SET data = EXCLUDED.data,
      expires_at = EXCLUDED.expires_at;

-- Read
SELECT data FROM sessions
WHERE id = $1 AND expires_at > NOW();
```

**附加内容：JSONB 运算符**

你可以在会话内部进行查询：

```sql
-- Find all sessions for a specific user
SELECT * FROM sessions
WHERE data->>'userId' = '123';

-- Find sessions with specific role
SELECT * FROM sessions
WHERE data->'user'->>'role' = 'admin';
```

你用Redis做不到这一点！

## 四、实际生产环境基准测试

我用生产数据集完成了基准测试：

**1、测试设置**

- **硬件：** AWS RDS db.t3.medium（2个虚拟CPU，4GB内存）
- **数据集：** 100万条缓存条目，1万个会话
- **工具：** pgbench（自定义脚本）

**2、结果**

| 操作 | Redis | PostgreSQL | 不同之处 |
|------|-------|------------|----------|
| 缓存集 | 0.05毫秒 | 0.08毫秒 | 速度降低 60% |
| 缓存获取 | 0.04毫秒 | 0.06毫秒 | 速度降低 50% |
| 发布订阅 | 1.2毫秒 | 3.1毫秒 | 速度降低 158% |
| 队列推送 | 0.08毫秒 | 0.15毫秒 | 速度降低 87% |
| 队列弹出 | 0.12毫秒 | 0.31毫秒 | 速度降低 158% |

PostgreSQL速度较慢，但是：

- 所有操作耗时均保持在1毫秒以内
- 省去了与Redis交互的网络开销
- 降低基础设施复杂性

**3、合并执行（真正的胜利）**

**场景：** 插入数据 + 缓存失效 + 通知订阅者

**使用Redis**：

```
db.Exec(ctx, "INSERT INTO posts ...")          // 2ms
rdb.Del(ctx, "posts:latest")                     // 1ms (network hop)
rdb.Publish(ctx, "posts:new", data)              // 1ms (network hop)
// Total: ~4ms
```

**使用PostgreSQL：**

```sql
BEGIN;
INSERT INTO posts ...;                          -- 2ms
DELETE FROM cache WHERE key = 'posts:latest';  -- 0.1ms (same connection)
NOTIFY posts_new, '...';                        -- 0.1ms (same connection)
COMMIT;
-- Total: ~2.2ms
```

**当多个操作合并执行时，PostgreSQL速度更快。**

## 五、哪些场景仍建议保留Redis

如果符合以下条件，请不要替换Redis：

**1、需要极致的性能**

```
Redis: 100,000+ ops/sec (single instance)
Postgres: 10,000-50,000 ops/sec
```

如果你每秒执行数百万次缓存读取操作，那就继续使用 Redis。

**2、使用Redis特有的数据结构**

**Redis具备：**

- 有序集合（排行榜）
- HyperLogLog（基数统计）
- 地理空间索引
- Streams（高级发布订阅）

**PostgreSQL 虽有对应实现，但使用起来更为繁琐：**

```sql
-- Leaderboard in Postgres (slower)
SELECT user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;

-- vs Redis
ZREVRANGE leaderboard 0 9 WITHSCORES
```

**3、架构需要独立缓存层**

如果你的架构要求独立的缓存层（例如微服务架构,例如主从数据库），建议保留Redis。

## 六、迁移方案

不要一夜之间就彻底放弃Redis，以下是我的做法：

**第一阶段：并排共存（第1周）**

```go
// Write to both
rdb.Set(ctx, key, value, 0)
db.Exec(ctx, "INSERT INTO cache ...")

// Read from Redis (still primary)
data, _ := rdb.Get(ctx, key).Result()
```

**监控：** 对比命中率、延迟。

**第二阶段：从Postgres读取数据（第2周）**

```go
// Try Postgres first
var value string
err := db.QueryRow(ctx, "SELECT value FROM cache WHERE key = $1", key).Scan(&value)

// Fallback to Redis
if err != nil {
    data, _ = rdb.Get(ctx, key).Result()
}
```

**监控：** 错误率、性能。

**第三阶段：仅写入Postgres（第3周）**

```go
// Only write to Postgres
db.Exec(ctx, "INSERT INTO cache ...")
```

监控：所有功能是否正常运行？

**第四阶段：移除Redis（第4周）**

```
# Turn off Redis
# Watch for errors
# Nothing breaks? Success!
```

## 七、代码示例：完整实现

**1、缓存模块（PostgreSQL）**

```go
// cache.go
package cache

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v4/pgxpool"
)

type PostgresCache struct {
    pool *pgxpool.Pool
}

func NewPostgresCache(pool *pgxpool.Pool) *PostgresCache {
    return &PostgresCache{pool: pool}
}

func (c *PostgresCache) Get(ctx context.Context, key string) ([]byte, error) {
    var value []byte
    err := c.pool.QueryRow(ctx,
        "SELECT value FROM cache WHERE key = $1 AND expires_at > NOW()",
        key,
    ).Scan(&value)
    if err != nil {
        return nil, err
    }
    return value, nil
}

func (c *PostgresCache) Set(ctx context.Context, key string, value []byte, ttl time.Duration) error {
    _, err := c.pool.Exec(ctx,
        fmt.Sprintf(`INSERT INTO cache (key, value, expires_at)
         VALUES ($1, $2, NOW() + INTERVAL '%d seconds')
         ON CONFLICT (key) DO UPDATE
           SET value = EXCLUDED.value,
               expires_at = EXCLUDED.expires_at`, int(ttl.Seconds())),
        key, value,
    )
    return err
}

func (c *PostgresCache) Delete(ctx context.Context, key string) error {
    _, err := c.pool.Exec(ctx, "DELETE FROM cache WHERE key = $1", key)
    return err
}

func (c *PostgresCache) Cleanup(ctx context.Context) error {
    _, err := c.pool.Exec(ctx, "DELETE FROM cache WHERE expires_at < NOW()")
    return err
}
```

**2、发布订阅模块**

```go
// pubsub.go
package pubsub

import (
    "context"
    "encoding/json"
    "sync"

    "github.com/jackc/pgx/v4/pgxpool"
)

type PostgresPubSub struct {
    pool      *pgxpool.Pool
    mu        sync.Mutex
    listeners map[string]context.CancelFunc
}

func NewPostgresPubSub(pool *pgxpool.Pool) *PostgresPubSub {
    return &PostgresPubSub{
        pool:      pool,
        listeners: make(map[string]context.CancelFunc),
    }
}

func (ps *PostgresPubSub) Publish(ctx context.Context, channel string, message interface{}) error {
    payload, _ := json.Marshal(message)
    _, err := ps.pool.Exec(ctx, "SELECT pg_notify($1, $2)", channel, string(payload))
    return err
}

func (ps *PostgresPubSub) Subscribe(ctx context.Context, channel string, callback func(interface{})) error {
    conn, err := ps.pool.Acquire(ctx)
    if err != nil {
        return err
    }

    _, err = conn.Exec(ctx, fmt.Sprintf("LISTEN %s", channel))
    if err != nil {
        conn.Release()
        return err
    }

    ctx, cancel := context.WithCancel(ctx)
    ps.mu.Lock()
    ps.listeners[channel] = cancel
    ps.mu.Unlock()

    go func() {
        defer conn.Release()
        for {
            notification, err := conn.Conn().WaitForNotification(ctx)
            if err != nil {
                return
            }
            var payload interface{}
            json.Unmarshal([]byte(notification.Payload), &payload)
            callback(payload)
        }
    }()

    return nil
}

func (ps *PostgresPubSub) Unsubscribe(channel string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    if cancel, ok := ps.listeners[channel]; ok {
        cancel()
        delete(ps.listeners, channel)
    }
}
```

**3、任务队列模块**

```go
// queue.go
package queue

import (
    "context"
    "encoding/json"
    "time"

    "github.com/jackc/pgx/v4/pgxpool"
)

type Job struct {
    ID          int64           `json:"id"`
    Queue       string          `json:"queue"`
    Payload     json.RawMessage `json:"payload"`
    Attempts    int             `json:"attempts"`
    MaxAttempts int             `json:"max_attempts"`
    ScheduledAt time.Time       `json:"scheduled_at"`
    CreatedAt   time.Time       `json:"created_at"`
}

type PostgresQueue struct {
    pool *pgxpool.Pool
}

func NewPostgresQueue(pool *pgxpool.Pool) *PostgresQueue {
    return &PostgresQueue{pool: pool}
}

func (q *PostgresQueue) Enqueue(ctx context.Context, queue string, payload interface{}, scheduledAt time.Time) error {
    data, _ := json.Marshal(payload)
    _, err := q.pool.Exec(ctx,
        "INSERT INTO jobs (queue, payload, scheduled_at) VALUES ($1, $2, $3)",
        queue, data, scheduledAt,
    )
    return err
}

func (q *PostgresQueue) Dequeue(ctx context.Context, queue string) (*Job, error) {
    row := q.pool.QueryRow(ctx,
        `WITH next_job AS (
          SELECT id FROM jobs
          WHERE queue = $1
            AND attempts < max_attempts
            AND scheduled_at <= NOW()
          ORDER BY scheduled_at
          LIMIT 1
          FOR UPDATE SKIP LOCKED
        )
        UPDATE jobs
        SET attempts = attempts + 1
        FROM next_job
        WHERE jobs.id = next_job.id
        RETURNING jobs.*`,
        queue,
    )

    var job Job
    err := row.Scan(&job.ID, &job.Queue, &job.Payload, &job.Attempts,
        &job.MaxAttempts, &job.ScheduledAt, &job.CreatedAt)
    if err != nil {
        return nil, err
    }
    return &job, nil
}

func (q *PostgresQueue) Complete(ctx context.Context, jobID int64) error {
    _, err := q.pool.Exec(ctx, "DELETE FROM jobs WHERE id = $1", jobID)
    return err
}

func (q *PostgresQueue) Fail(ctx context.Context, jobID int64, errMsg string) error {
    _, err := q.pool.Exec(ctx,
        `UPDATE jobs
         SET attempts = max_attempts,
             payload = payload || jsonb_build_object('error', $2)
         WHERE id = $1`,
        jobID, errMsg,
    )
    return err
}
```

## 八、性能优化技巧

**1、使用连接池**
golang的数据库驱动都带连接池的功能,但是注意,如果有通知或者高并发的日志/队列写入,请单独另启一个连接池.

```go
import (
    "context"
    "github.com/jackc/pgx/v4/pgxpool"
)

pool, _ := pgxpool.Connect(ctx, os.Getenv("DATABASE_URL"))
defer pool.Close()

pool.Config().MaxConns = 20
pool.Config().MaxConnIdleTime = 30 * time.Second
pool.Config().MaxConnLifetime = 1 * time.Hour
```

**2、添加合适的索引**

```sql
CREATE INDEX CONCURRENTLY idx_cache_key ON cache(key) WHERE expires_at > NOW();
CREATE INDEX CONCURRENTLY idx_jobs_pending ON jobs(queue, scheduled_at) 
  WHERE attempts < max_attempts;
```

**3、调整PostgreSQL配置**

```
# postgresql.conf
shared_buffers = 2GB           # 25% of RAM
effective_cache_size = 6GB     # 75% of RAM
work_mem = 50MB                # For complex queries
maintenance_work_mem = 512MB   # For VACUUM
```

**4、定期维护**

```sql
-- Run daily
VACUUM ANALYZE cache;
VACUUM ANALYZE jobs;

-- Or enable autovacuum (recommended)
ALTER TABLE cache SET (autovacuum_vacuum_scale_factor = 0.1);
```

## 九、三个月后的结果

**我省下了：**

- 每月100美元（不再使用 ElastiCache）
- 备份复杂性降低50%
- 少监控一项服务
- 更简单的部署（减少一项依赖）

**我失去了：**

- 缓存操作延迟则增加约0.5毫秒
- Redis特有的数据结构（其实并不需要,海量记录的TOP排序之类的除外)

**我会再次这样做吗？** 就这个业务场景而言：会。

**是否推荐所有人都这么做？** 不推荐。

## 十、决策矩阵

**如果满足以下条件，可用Postgres替换Redis：**

- 请求访问量 TPS<3000,其实绝大多数项目达不到.
- 仅用Redis做简单缓存或者会话管理
- 缓存命中率低于95%（写入次数过多）
- 需要事务一致性
- 可以接受操作速度慢0.1-1毫秒
- 团队规模小，运维资源有限

**以下场景建议保留Redis：**

- 每秒10万次以上的操作量
- 使用Redis特有的数据结构（有序集合等）
- 配备专业的运维团队
- 亚毫秒级延迟为核心要求
- 需要跨区域地理复制

## 十一、参考资料

**1、PostgreSQL 特性**

- LISTEN/NOTIFY 官方文档
- SKIP LOCKED 语法
- UNLOGGED 表

**2、工具**

- pgBouncer - 连接池
- pg_stat_statements - 查询性能

**3、其他解决方案**

- Graphile Worker - 基于Postgres的任务队列
- pg-boss - 另一款Postgres队列实现

## 十二、最后

**我用PostgreSQL替换了Redis的这些场景：**

- 缓存 → UNLOGGED 表
- 发布订阅 → LISTEN/NOTIFY
- 任务队列 → SKIP LOCKED
- 会话存储 → JSONB 表

**结果：**

- 节省redis服务费用
- 降低了运维复杂度
- 性能略有下降（延迟增加 0.1–1ms），但可接受
- 保证了事务一致性

**适合这样做的场景：**

- 中小型应用
- 简单的缓存需求
- 希望减少系统组件、简化架构

**不适合这样做的场景：**

- 性能要求高（每秒10万次以上操作）
- 用Redis特有的功能
- 配备专职运维团队
