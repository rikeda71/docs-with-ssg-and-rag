# RFC-003: パフォーマンス最適化 - N+1問題の解決とキャッシュ戦略の改善

**ステータス**: Proposed
**作成日**: 2025-12-05
**担当者**: Performance Engineering Team
**レビュアー**: Engineering Leadership, SRE Team
**想定工数**: 8 週間 (3エンジニア)
**優先度**: P0 (Critical)

## エグゼクティブサマリー

本提案は、カレンダーシステムの深刻なパフォーマンス問題を解決するための包括的な最適化計画です。現在、N+1クエリ問題、非効率なキャッシュ戦略、データベースホットスポットにより、ピーク時のレスポンス時間が5秒を超え、ユーザー体験が著しく低下しています。

**現状の問題**:
- イベント一覧取得のレスポンス時間: **P95 = 4.2秒、P99 = 8.5秒**
- N+1クエリによる無駄なDB呼び出し: **1リクエストあたり平均 87クエリ**
- データベースCPU使用率: ピーク時 **92%** (頻繁にスロットリング発生)
- キャッシュヒット率: **38%** (目標70%に対して大幅に低い)

**目標**:
- レスポンス時間を **P95 < 200ms、P99 < 500ms** に改善 (95% 削減)
- クエリ数を **1リクエストあたり 5クエリ以下** に削減 (94% 削減)
- データベースCPU使用率を **< 60%** に削減
- キャッシュヒット率を **> 80%** に向上

**期待される効果**:
- ユーザー満足度の大幅改善 (NPS +25ポイント)
- インフラコストの **30% 削減** (年間 ¥36M)
- システム可用性の向上 (99.5% → 99.95%)
- 将来的なスケーラビリティの確保

## 1. 現状分析とボトルネック特定

### 1.1 パフォーマンスプロファイリング

**Application Performance Monitoring (APM) データ分析**:

```
Top 5 Slowest Endpoints (P95 latency):
1. GET /api/events?calendar_id={id}       4,200ms  ← Critical!
2. GET /api/calendars/{id}/events         3,800ms  ← Critical!
3. GET /api/events/{id}                   1,500ms
4. POST /api/events                         850ms
5. GET /api/resources/availability          720ms

Database Query Analysis:
Total Queries: 127,450/min (peak)
Slow Queries (>100ms): 12,340/min (9.7%)
Top Slow Query: SELECT * FROM attendees WHERE event_id = ? (executed 45,000/min)
```

**トレーシング分析 (Jaeger)**:

```
Trace: GET /api/calendars/{id}/events (Total: 4,200ms)
├─ GraphQL Resolver (100ms)
├─ Calendar Service: GetEvents (4,050ms)  ← Bottleneck!
│  ├─ DB: SELECT events (120ms)
│  ├─ For each event (50 events):
│  │  ├─ DB: SELECT attendees (45ms × 50 = 2,250ms)  ← N+1!
│  │  ├─ DB: SELECT resources (38ms × 50 = 1,900ms)  ← N+1!
│  │  └─ User Service: GetUserBatch (8ms × 50 = 400ms)  ← N+1!
│  └─ Total: 4,670ms
└─ Response Formatting (50ms)

Issue: N+1 queries for attendees, resources, and users
```

### 1.2 データベース分析

**Slow Query Log**:

```sql
-- Top 3 Slowest Queries

-- Query 1: Attendees (executed 45,000 times/min)
-- Average execution: 45ms, Total time: 33.75min/min
SELECT * FROM attendees WHERE event_id = $1;

-- Query 2: Resources (executed 40,000 times/min)
-- Average execution: 38ms, Total time: 25.33min/min
SELECT r.*, res.* FROM reservations res
JOIN resources r ON r.id = res.resource_id
WHERE res.event_id = $1;

-- Query 3: Users (executed 150,000 times/min)
-- Average execution: 8ms, Total time: 20min/min
SELECT * FROM users WHERE id = $1;

-- Total wasted DB time: 79.08 minutes per minute (!)
```

**EXPLAIN ANALYZE**:

```sql
EXPLAIN ANALYZE
SELECT * FROM events
WHERE calendar_id = 'cal_123'
  AND start_time >= '2025-12-01'
  AND start_time < '2026-01-01'
  AND deleted_at IS NULL;

-- Result:
Seq Scan on events_2025_12  (cost=0.00..5432.00 rows=150 width=1024)
                              (actual time=0.045..124.231 rows=150 loops=1)
  Filter: ((calendar_id = 'cal_123') AND (deleted_at IS NULL) AND ...)
  Rows Removed by Filter: 48850
Planning Time: 0.523 ms
Execution Time: 124.891 ms

-- Issue: Sequential scan instead of index scan!
```

**Missing Indexes**:

```sql
-- Missing composite index
CREATE INDEX idx_events_calendar_time_deleted ON events(calendar_id, start_time)
WHERE deleted_at IS NULL;

-- After adding index:
Index Scan using idx_events_calendar_time_deleted  (cost=0.43..12.45 rows=150)
                                                    (actual time=0.028..1.234 rows=150)
Execution Time: 1.456 ms  ← 99% improvement!
```

### 1.3 キャッシュ分析

**Current Cache Hit Rate**: 38%

**Cache Key Distribution**:
```
Key Pattern                     | Hit Rate | Eviction Rate | TTL
--------------------------------|----------|---------------|-----
event:{id}                      | 45%      | High (25%)    | 5 min
calendar:{id}                   | 62%      | Low (3%)      | 5 min
user:{id}                       | 28%      | Very High(40%)| 5 min  ← Problem!
event:{id}:attendees            | N/A      | N/A           | Not cached!
calendar:{id}:events            | N/A      | N/A           | Not cached!
```

**問題点**:
1. **ユーザー情報のキャッシュ**: 頻繁にevictされる（メモリ不足）
2. **関連データの未キャッシュ**: attendees, resources がキャッシュされていない
3. **TTLの非最適化**: すべて一律5分（データ特性を考慮していない）
4. **キャッシュキーの粒度**: 粗すぎてヒット率が低い

## 2. 最適化戦略

### 2.1 N+1クエリの解決

#### 戦略1: DataLoader パターンの導入

**Before (N+1 problem)**:

```go
// BAD: N+1 query
func (s *EventService) GetEvents(calendarID string) ([]*Event, error) {
    events, err := s.db.Query("SELECT * FROM events WHERE calendar_id = $1", calendarID)
    if err != nil {
        return nil, err
    }

    for _, event := range events {
        // N+1: Each event triggers separate queries
        event.Attendees, _ = s.getAttendees(event.ID)      // Query 1, 2, 3, ...
        event.Resources, _ = s.getResources(event.ID)      // Query 1, 2, 3, ...
        event.Organizer, _ = s.getUserByID(event.OrganizerID)  // Query 1, 2, 3, ...
    }

    return events, nil
}

// Total queries: 1 + N*3 = 1 + 50*3 = 151 queries for 50 events!
```

**After (DataLoader)**:

```go
// GOOD: Batched queries with DataLoader
type DataLoaders struct {
    AttendeeLoader *dataloader.Loader
    ResourceLoader *dataloader.Loader
    UserLoader     *dataloader.Loader
}

func (s *EventService) GetEvents(ctx context.Context, calendarID string) ([]*Event, error) {
    events, err := s.db.Query("SELECT * FROM events WHERE calendar_id = $1", calendarID)
    if err != nil {
        return nil, err
    }

    // Extract all IDs
    eventIDs := extractIDs(events)
    userIDs := extractUserIDs(events)

    // DataLoader batches and caches automatically
    loaders := GetDataLoaders(ctx)

    // Batch load attendees (1 query for all events)
    attendeesMap, _ := loaders.AttendeeLoader.LoadMany(ctx, eventIDs)

    // Batch load resources (1 query for all events)
    resourcesMap, _ := loaders.ResourceLoader.LoadMany(ctx, eventIDs)

    // Batch load users (1 query for all unique users)
    usersMap, _ := loaders.UserLoader.LoadMany(ctx, userIDs)

    // Assemble results
    for _, event := range events {
        event.Attendees = attendeesMap[event.ID]
        event.Resources = resourcesMap[event.ID]
        event.Organizer = usersMap[event.OrganizerID]
    }

    return events, nil
}

// Total queries: 1 + 3 = 4 queries for any number of events!
// 151 queries → 4 queries (97% reduction)
```

**DataLoader Implementation**:

```go
// AttendeeLoader
func NewAttendeeLoader(db *Database) *dataloader.Loader {
    return dataloader.NewBatchedLoader(
        func(ctx context.Context, eventIDs []string) []*dataloader.Result {
            // Single batch query
            query := `
                SELECT event_id, user_id, email, name, response_status, is_optional
                FROM attendees
                WHERE event_id = ANY($1)
                ORDER BY event_id, created_at
            `

            rows, err := db.Query(ctx, query, pq.Array(eventIDs))
            if err != nil {
                return errorResults(len(eventIDs), err)
            }

            // Group by event_id
            attendeesByEvent := make(map[string][]*Attendee)
            for rows.Next() {
                var a Attendee
                rows.Scan(&a.EventID, &a.UserID, &a.Email, &a.Name, &a.ResponseStatus, &a.IsOptional)
                attendeesByEvent[a.EventID] = append(attendeesByEvent[a.EventID], &a)
            }

            // Return results in the same order as keys
            results := make([]*dataloader.Result, len(eventIDs))
            for i, eventID := range eventIDs {
                results[i] = &dataloader.Result{Data: attendeesByEvent[eventID]}
            }
            return results
        },
        dataloader.WithCache(&dataloader.NoCache{}),  // Use Redis for caching
    )
}
```

#### 戦略2: GraphQL N+1の解決

**Before**:

```graphql
# GraphQL Query
{
  calendar(id: "cal_123") {
    events {
      id
      title
      attendees {     # N+1: Each event triggers separate resolver
        user {        # N+1: Each attendee triggers separate resolver
          id
          name
          avatar
        }
      }
    }
  }
}
```

**After (DataLoader in GraphQL resolvers)**:

```typescript
// GraphQL Resolver with DataLoader
const resolvers = {
  Query: {
    calendar: async (_, { id }, { dataloaders }) => {
      return dataloaders.calendarLoader.load(id);
    },
  },

  Calendar: {
    events: async (calendar, _, { dataloaders }) => {
      // Batched load
      return dataloaders.eventLoader.loadMany(calendar.eventIds);
    },
  },

  Event: {
    attendees: async (event, _, { dataloaders }) => {
      // Batched load - automatically deduplicates and batches
      return dataloaders.attendeeLoader.load(event.id);
    },
  },

  Attendee: {
    user: async (attendee, _, { dataloaders }) => {
      // Batched load - automatically deduplicates
      return dataloaders.userLoader.load(attendee.userId);
    },
  },
};

// DataLoader Context (created per request)
function createContext({ req }) {
  return {
    dataloaders: {
      calendarLoader: new DataLoader(batchGetCalendars),
      eventLoader: new DataLoader(batchGetEvents),
      attendeeLoader: new DataLoader(batchGetAttendees),
      userLoader: new DataLoader(batchGetUsers),
    },
  };
}
```

**結果**:
- クエリ数: 1 + N*(1 + M*1) = 数百クエリ → 4クエリ
- レスポンス時間: 4,200ms → 180ms (96% 改善)

### 2.2 データベース最適化

#### 最適化1: インデックスの追加

**Missing Indexes Analysis**:

```sql
-- Analyze query patterns
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename IN ('events', 'attendees', 'resources', 'reservations')
ORDER BY tablename, attname;

-- Identify missing indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, tablename;
```

**Recommended Indexes**:

```sql
-- 1. Composite index for event queries (most critical)
CREATE INDEX CONCURRENTLY idx_events_calendar_time_deleted
ON events(calendar_id, start_time)
WHERE deleted_at IS NULL;

-- 2. Index for attendee lookups (solve N+1)
CREATE INDEX CONCURRENTLY idx_attendees_event_id
ON attendees(event_id)
INCLUDE (user_id, email, name, response_status, is_optional);

-- 3. Index for resource reservations (solve N+1)
CREATE INDEX CONCURRENTLY idx_reservations_event_resource
ON reservations(event_id, resource_id)
INCLUDE (start_time, end_time, status);

-- 4. Index for availability queries
CREATE INDEX CONCURRENTLY idx_reservations_resource_time
ON reservations(resource_id, start_time, end_time)
WHERE status = 'CONFIRMED';

-- 5. Partial index for active events
CREATE INDEX CONCURRENTLY idx_events_active
ON events(organizer_id, start_time)
WHERE deleted_at IS NULL AND status = 'CONFIRMED';

-- 6. Index for user calendar memberships
CREATE INDEX CONCURRENTLY idx_calendar_members_user
ON calendar_members(user_id)
INCLUDE (calendar_id, role);

-- 7. Covering index for event list queries
CREATE INDEX CONCURRENTLY idx_events_list_covering
ON events(calendar_id, start_time DESC)
INCLUDE (id, title, organizer_id, end_time, status)
WHERE deleted_at IS NULL;
```

**Impact Estimation**:

| Query Pattern | Before | After | Improvement |
|--------------|--------|-------|------------|
| Event list by calendar | 124ms | 1.2ms | **99%** |
| Attendees by event | 45ms | 0.8ms | **98%** |
| Resource availability | 85ms | 3.5ms | **96%** |
| User's calendars | 62ms | 2.1ms | **97%** |

**Index Maintenance**:

```sql
-- Monitor index usage
CREATE VIEW index_usage AS
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan::float / GREATEST(seq_scan + idx_scan, 1) AS index_usage_ratio
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Identify unused indexes (for cleanup)
SELECT * FROM index_usage WHERE idx_scan < 100 AND index_size > '10 MB';

-- Reindex periodically (during low traffic)
REINDEX INDEX CONCURRENTLY idx_events_calendar_time_deleted;
```

#### 最適化2: クエリの最適化

**Optimized Query Examples**:

```sql
-- BEFORE: Inefficient query with subqueries
SELECT
    e.*,
    (SELECT COUNT(*) FROM attendees WHERE event_id = e.id) as attendee_count,
    (SELECT name FROM users WHERE id = e.organizer_id) as organizer_name,
    (SELECT name FROM resources r
     JOIN reservations res ON res.resource_id = r.id
     WHERE res.event_id = e.id
     LIMIT 1) as room_name
FROM events e
WHERE e.calendar_id = $1
  AND e.start_time >= $2
  AND e.start_time < $3
  AND e.deleted_at IS NULL;

-- Query time: ~450ms for 50 events

-- AFTER: Optimized with JOINs and aggregation
WITH event_attendee_counts AS (
    SELECT event_id, COUNT(*) as count
    FROM attendees
    WHERE event_id IN (
        SELECT id FROM events
        WHERE calendar_id = $1
          AND start_time >= $2
          AND start_time < $3
          AND deleted_at IS NULL
    )
    GROUP BY event_id
),
event_resources AS (
    SELECT DISTINCT ON (res.event_id)
        res.event_id,
        r.name as room_name
    FROM reservations res
    JOIN resources r ON r.id = res.resource_id
    WHERE res.event_id IN (
        SELECT id FROM events
        WHERE calendar_id = $1
          AND start_time >= $2
          AND start_time < $3
          AND deleted_at IS NULL
    )
)
SELECT
    e.*,
    COALESCE(ac.count, 0) as attendee_count,
    u.name as organizer_name,
    er.room_name
FROM events e
LEFT JOIN event_attendee_counts ac ON ac.event_id = e.id
LEFT JOIN users u ON u.id = e.organizer_id
LEFT JOIN event_resources er ON er.event_id = e.id
WHERE e.calendar_id = $1
  AND e.start_time >= $2
  AND e.start_time < $3
  AND e.deleted_at IS NULL
ORDER BY e.start_time DESC;

-- Query time: ~8ms for 50 events (98% improvement!)
```

#### 最適化3: Connection Pooling の最適化

**Before**:

```go
// Poor connection pool settings
db, err := sql.Open("postgres", dsn)
db.SetMaxOpenConns(100)  // Too high, causes contention
db.SetMaxIdleConns(10)   // Too low, frequent reconnections
db.SetConnMaxLifetime(time.Hour)  // Too long
```

**After**:

```go
// Optimized connection pool
db, err := sql.Open("postgres", dsn)

// Formula: MaxOpenConns = ((core_count * 2) + effective_spindle_count)
// For our DB: 8 cores, SSD → (8*2 + 1) = 17
db.SetMaxOpenConns(20)  // Slightly above formula

// Keep idle connections warm (reduce latency)
db.SetMaxIdleConns(10)  // 50% of max

// Prevent stale connections
db.SetConnMaxLifetime(30 * time.Minute)

// Prevent connection leaks
db.SetConnMaxIdleTime(10 * time.Minute)

// Monitor pool stats
go monitorConnectionPool(db)
```

**Monitoring**:

```go
func monitorConnectionPool(db *sql.DB) {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        stats := db.Stats()

        metrics.Gauge("db.connections.open", stats.OpenConnections)
        metrics.Gauge("db.connections.in_use", stats.InUse)
        metrics.Gauge("db.connections.idle", stats.Idle)
        metrics.Counter("db.connections.wait_count", stats.WaitCount)
        metrics.Histogram("db.connections.wait_duration", stats.WaitDuration.Milliseconds())

        // Alert if pool is saturated
        if float64(stats.InUse)/float64(stats.MaxOpenConnections) > 0.9 {
            alerts.Warning("Database connection pool near saturation")
        }
    }
}
```

### 2.3 キャッシュ戦略の改善

#### 改善1: 多層キャッシュアーキテクチャ

```
┌─────────────────────────────────────────────────────────┐
│                  L1: In-Memory Cache                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │  sync.Map (Go) / LRU Cache                       │  │
│  │  - Hot data only (current user's calendar)       │  │
│  │  - TTL: 30 seconds                               │  │
│  │  - Max size: 10,000 entries                      │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                         ↓ (on miss)
┌─────────────────────────────────────────────────────────┐
│                 L2: Distributed Cache                   │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Redis (ElastiCache)                             │  │
│  │  - Shared across all service instances          │  │
│  │  - TTL: Variable (1min - 24hours)               │  │
│  │  - Eviction: allkeys-lru                        │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                         ↓ (on miss)
┌─────────────────────────────────────────────────────────┐
│                     L3: Database                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  PostgreSQL                                      │  │
│  │  - Source of truth                               │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**実装**:

```go
type CacheManager struct {
    l1 *sync.Map           // In-memory cache
    l2 *redis.Client       // Redis cache
    db *Database           // PostgreSQL
}

func (c *CacheManager) Get(ctx context.Context, key string) (interface{}, error) {
    // L1: Check in-memory cache
    if val, ok := c.l1.Load(key); ok {
        metrics.Counter("cache.l1.hit").Inc()
        return val, nil
    }
    metrics.Counter("cache.l1.miss").Inc()

    // L2: Check Redis
    val, err := c.l2.Get(ctx, key).Result()
    if err == nil {
        metrics.Counter("cache.l2.hit").Inc()

        // Populate L1
        c.l1.Store(key, val)
        time.AfterFunc(30*time.Second, func() { c.l1.Delete(key) })

        return val, nil
    }
    metrics.Counter("cache.l2.miss").Inc()

    // L3: Fetch from database
    data, err := c.db.Get(ctx, key)
    if err != nil {
        return nil, err
    }

    // Populate caches (write-through)
    c.Set(ctx, key, data, getTTL(key))

    return data, nil
}

func (c *CacheManager) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) {
    // L1
    c.l1.Store(key, value)
    time.AfterFunc(30*time.Second, func() { c.l1.Delete(key) })

    // L2
    c.l2.Set(ctx, key, value, ttl)
}
```

#### 改善2: キャッシュキーの最適化

**Before**:

```
event:{event_id}                    # Too granular, low hit rate
user:{user_id}                      # Evicted frequently
```

**After**:

```
# Hierarchical cache keys
calendar:{calendar_id}:summary              # TTL: 10 min
calendar:{calendar_id}:events:{month}       # TTL: 5 min (partitioned by month)
event:{event_id}:full                       # TTL: 5 min
event:{event_id}:attendees                  # TTL: 2 min
user:{user_id}:profile                      # TTL: 30 min (user data changes infrequently)
resource:{resource_id}:availability:{date}  # TTL: 1 min

# Batch cache keys (reduce round trips)
events:batch:{event_ids_hash}               # TTL: 5 min
users:batch:{user_ids_hash}                 # TTL: 30 min
```

#### 改善3: Cache Stampede 対策

**Problem**: キャッシュ期限切れ時に大量のリクエストがDBに殺到

**Solution: Probabilistic Early Expiration**:

```go
func (c *CacheManager) GetWithPEE(ctx context.Context, key string, ttl time.Duration, fetchFn func() (interface{}, error)) (interface{}, error) {
    val, expiry, err := c.getWithExpiry(ctx, key)

    if err == nil {
        // Calculate probability of early refresh
        timeRemaining := expiry.Sub(time.Now())
        delta := ttl - timeRemaining
        beta := 1.0  // Tuning parameter

        // Probability increases as expiry approaches
        prob := -beta * math.Log(rand.Float64())

        if timeRemaining.Seconds() > prob*delta.Seconds() {
            // Use cached value
            return val, nil
        }
    }

    // Refresh cache (single-flight to prevent stampede)
    result, err, _ := c.singleflight.Do(key, func() (interface{}, error) {
        data, err := fetchFn()
        if err != nil {
            return nil, err
        }

        c.Set(ctx, key, data, ttl)
        return data, nil
    })

    return result, err
}
```

#### 改善4: Cache Invalidation Strategy

**Event-Driven Invalidation**:

```go
// When event is updated, invalidate related caches
func (s *EventService) UpdateEvent(ctx context.Context, eventID string, updates *EventUpdates) error {
    // Update database
    event, err := s.db.UpdateEvent(ctx, eventID, updates)
    if err != nil {
        return err
    }

    // Invalidate caches
    go s.invalidateCaches(ctx, event)

    // Publish invalidation event (for other service instances)
    s.kafka.Publish(ctx, "cache.invalidate", CacheInvalidationEvent{
        Keys: []string{
            fmt.Sprintf("event:%s:full", eventID),
            fmt.Sprintf("event:%s:attendees", eventID),
            fmt.Sprintf("calendar:%s:events:*", event.CalendarID),
        },
        Timestamp: time.Now(),
    })

    return nil
}

func (s *EventService) invalidateCaches(ctx context.Context, event *Event) {
    keys := []string{
        fmt.Sprintf("event:%s:full", event.ID),
        fmt.Sprintf("event:%s:attendees", event.ID),
    }

    // Invalidate all affected calendar caches
    pattern := fmt.Sprintf("calendar:%s:events:*", event.CalendarID)
    calendarKeys, _ := s.cache.Keys(ctx, pattern)
    keys = append(keys, calendarKeys...)

    // Delete in batch
    s.cache.Del(ctx, keys...)
}
```

## 3. 実装計画

### 3.1 フェーズ分け

#### Phase 1: Quick Wins (Week 1-2)

**目標**: 即座に効果が出る改善

**タスク**:
- [ ] 不足しているインデックスの追加
- [ ] Connection pooling settings の最適化
- [ ] 明らかに非効率なクエリの書き直し
- [ ] Redis cache TTL の最適化

**期待効果**:
- レスポンス時間 30% 改善
- DB CPU 使用率 20% 削減

#### Phase 2: N+1 Resolution (Week 3-5)

**目標**: N+1問題の完全解決

**タスク**:
- [ ] DataLoader の実装
- [ ] GraphQL resolvers の最適化
- [ ] Batch loading endpoints の追加
- [ ] 統合テストとパフォーマンステスト

**期待効果**:
- クエリ数 95% 削減
- レスポンス時間 80% 改善

#### Phase 3: Advanced Caching (Week 6-8)

**目標**: キャッシュ戦略の全面刷新

**タスク**:
- [ ] 多層キャッシュアーキテクチャの実装
- [ ] Cache stampede 対策
- [ ] Event-driven cache invalidation
- [ ] Cache monitoring dashboard

**期待効果**:
- キャッシュヒット率 80% 達成
- レスポンス時間 さらに 50% 改善

### 3.2 ロールアウト計画

```
Week 1:   Quick wins deployment (low risk)
Week 2:   Monitor and measure impact
Week 3-4: DataLoader implementation and testing
Week 5:   Gradual rollout (10% → 50% → 100%)
Week 6-7: Advanced caching implementation
Week 8:   Final rollout and optimization
```

## 4. 成功指標 (KPI)

### 4.1 パフォーマンス目標

| メトリクス | 現状 | 目標 | 測定方法 |
|-----------|------|------|---------|
| API レスポンス (P95) | 4,200ms | < 200ms | APM |
| API レスポンス (P99) | 8,500ms | < 500ms | APM |
| DB クエリ数/リクエスト | 87 | < 5 | Query logging |
| Slow queries (>100ms) | 9.7% | < 1% | DB monitoring |
| DB CPU 使用率 (peak) | 92% | < 60% | CloudWatch |
| キャッシュヒット率 | 38% | > 80% | Redis metrics |

### 4.2 ビジネスインパクト

| メトリクス | 期待値 |
|-----------|--------|
| ユーザー満足度 (NPS) | +25 pts |
| ページ離脱率削減 | -40% |
| インフラコスト削減 | ¥36M/年 |
| システム可用性 | 99.5% → 99.95% |

## 5. リスクと軽減策

| リスク | 影響 | 確率 | 軽減策 |
|--------|------|------|--------|
| インデックス作成時のロック | 中 | 低 | CONCURRENTLY オプション使用 |
| Cache invalidation のバグ | 高 | 中 | 徹底的なテスト、段階的ロールアウト |
| DataLoader の複雑性 | 中 | 中 | 十分な単体テスト、ドキュメント整備 |

## 6. 監視とアラート

```yaml
Alerts:
  - name: SlowAPIResponse
    condition: p95_latency > 500ms
    action: Slack notification

  - name: HighDBCPU
    condition: db_cpu > 70%
    action: Auto-scaling + Alert

  - name: LowCacheHitRate
    condition: cache_hit_rate < 70%
    action: Investigate + Alert
```

---

**関連ドキュメント**:
- [Architecture](./architecture.md)
- [Database Optimization Guide](../database/optimization.md)

**更新履歴**:
- 2025-12-05: 初版作成 (v1.0)
