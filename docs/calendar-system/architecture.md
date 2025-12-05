# カレンダーシステム アーキテクチャ設計

## エグゼクティブサマリー

本ドキュメントでは、エンタープライズグレードのカレンダーシステムの技術アーキテクチャを定義します。マイクロサービスアーキテクチャ、イベント駆動設計、およびクラウドネイティブな原則に基づき、高可用性、スケーラビリティ、セキュリティを実現します。

**主要な技術選定**:
- **バックエンド**: Go (gRPC services), Node.js (BFF layer)
- **フロントエンド**: React + TypeScript, React Native
- **データストア**: PostgreSQL (primary), Redis (cache), Elasticsearch (search)
- **メッセージング**: Apache Kafka
- **インフラ**: AWS (EKS, RDS, ElastiCache, MSK)

## 1. アーキテクチャ原則

### 1.1 コアプリンシプル

#### スケーラビリティ
- **水平スケーリング**: すべてのサービスをステートレスに設計し、必要に応じてスケールアウト可能
- **データパーティショニング**: ユーザーIDベースのシャーディングにより、データベース層のスケーラビリティを確保
- **CDN活用**: 静的リソースとグローバルに分散したエッジキャッシュ

#### 可用性と信頼性
- **マルチAZ展開**: すべてのコンポーネントを複数のアベイラビリティゾーンに配置
- **サーキットブレーカー**: 障害の伝播を防ぐため、全ての外部呼び出しにサーキットブレーカーを実装
- **優雅な劣化**: 部分的な障害時でも基本機能を提供できる設計
- **目標**: 99.95% の可用性（年間ダウンタイム 4.38時間）

#### パフォーマンス
- **レスポンス時間**: P95 で 200ms 以内、P99 で 500ms 以内
- **スループット**: 10,000 リクエスト/秒を処理可能
- **データベースクエリ**: インデックス最適化により、すべてのクエリを 50ms 以内に実行
- **リアルタイム更新**: WebSocket による双方向通信で 100ms 以内の更新通知

#### セキュリティ
- **ゼロトラスト原則**: すべての通信を認証・認可
- **データ暗号化**: 転送中（TLS 1.3）および保存時（AES-256）の暗号化
- **最小権限の原則**: IAM ポリシーとRBACによる厳密なアクセス制御
- **コンプライアンス**: GDPR, SOC 2 Type II, ISO 27001 準拠

#### 保守性
- **モジュラー設計**: 疎結合なマイクロサービスによる独立したデプロイとスケーリング
- **オブザーバビリティ**: 分散トレーシング、メトリクス、ログの統合監視
- **Infrastructure as Code**: Terraform によるインフラ管理
- **自動化**: CI/CD パイプラインによる継続的デリバリー

## 2. 高レベルアーキテクチャ

### 2.1 システム概要図

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Web App    │  │  Mobile App  │  │  API Client  │         │
│  │(React + TS)  │  │(React Native)│  │(External)    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   AWS API Gateway + CloudFront (CDN)                     │  │
│  │   - Rate Limiting  - DDoS Protection  - WAF              │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    BFF (Backend for Frontend)                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   GraphQL API Layer (Node.js + Apollo Server)            │  │
│  │   - Request Aggregation  - Response Caching              │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Microservices Layer                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐ │
│  │  Calendar   │ │   Event     │ │  Resource   │ │  User    │ │
│  │  Service    │ │  Service    │ │  Service    │ │  Service │ │
│  │  (Go)       │ │  (Go)       │ │  (Go)       │ │  (Go)    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └──────────┘ │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐ │
│  │ Notification│ │  Sync       │ │ Analytics   │ │  Search  │ │
│  │ Service     │ │  Service    │ │ Service     │ │  Service │ │
│  │ (Go)        │ │  (Go)       │ │ (Go)        │ │  (Go)    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Event Streaming Layer                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   Apache Kafka (AWS MSK)                                 │  │
│  │   Topics: calendar.events, notifications, sync.requests  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                       Data Layer                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐ │
│  │ PostgreSQL  │ │    Redis    │ │Elasticsearch│ │   S3     │ │
│  │   (RDS)     │ │(ElastiCache)│ │   (ES)      │ │(Storage) │ │
│  │  Primary DB │ │   Cache     │ │   Search    │ │  Files   │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Observability & Operations                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐ │
│  │ Prometheus  │ │   Grafana   │ │    Jaeger   │ │   ELK    │ │
│  │  Metrics    │ │ Dashboards  │ │   Tracing   │ │   Logs   │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 レイヤー責務

| レイヤー | 責務 | 技術スタック |
|---------|------|------------|
| Client | ユーザーインターフェース、クライアントサイドロジック | React, React Native, TypeScript |
| API Gateway | ルーティング、認証、レート制限、DDoS防御 | AWS API Gateway, CloudFront, WAF |
| BFF | リクエスト集約、データ変換、キャッシング | Node.js, Apollo GraphQL |
| Microservices | ビジネスロジック、データ管理 | Go, gRPC, Protocol Buffers |
| Event Streaming | 非同期処理、イベントソーシング | Apache Kafka (MSK) |
| Data | データ永続化、検索、キャッシング | PostgreSQL, Redis, Elasticsearch |
| Observability | 監視、ログ、トレーシング | Prometheus, Grafana, Jaeger, ELK |

## 3. マイクロサービス設計

### 3.1 Calendar Service

**責務**: カレンダーの作成、管理、共有機能

**主要API**:
```protobuf
service CalendarService {
  rpc CreateCalendar(CreateCalendarRequest) returns (Calendar);
  rpc GetCalendar(GetCalendarRequest) returns (Calendar);
  rpc UpdateCalendar(UpdateCalendarRequest) returns (Calendar);
  rpc DeleteCalendar(DeleteCalendarRequest) returns (Empty);
  rpc ListCalendars(ListCalendarsRequest) returns (ListCalendarsResponse);
  rpc ShareCalendar(ShareCalendarRequest) returns (ShareResponse);
}
```

**データモデル**:
```go
type Calendar struct {
    ID          string
    OwnerID     string
    Name        string
    Description string
    Color       string
    TimeZone    string
    IsPublic    bool
    Settings    CalendarSettings
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

**スケーリング戦略**:
- ユーザーIDベースのパーティショニング
- 読み取りレプリカによる負荷分散
- Redis キャッシング（TTL: 5分）

### 3.2 Event Service

**責務**: イベント（ミーティング、予定）のCRUD操作、リカレンス処理

**主要API**:
```protobuf
service EventService {
  rpc CreateEvent(CreateEventRequest) returns (Event);
  rpc GetEvent(GetEventRequest) returns (Event);
  rpc UpdateEvent(UpdateEventRequest) returns (Event);
  rpc DeleteEvent(DeleteEventRequest) returns (Empty);
  rpc ListEvents(ListEventsRequest) returns (ListEventsResponse);
  rpc SearchEvents(SearchEventsRequest) returns (SearchEventsResponse);
  rpc GetEventOccurrences(GetOccurrencesRequest) returns (OccurrencesResponse);
}
```

**データモデル**:
```go
type Event struct {
    ID              string
    CalendarID      string
    OrganizerID     string
    Title           string
    Description     string
    Location        string
    StartTime       time.Time
    EndTime         time.Time
    TimeZone        string
    IsAllDay        bool
    RecurrenceRule  *RecurrenceRule
    Attendees       []Attendee
    Reminders       []Reminder
    Status          EventStatus  // CONFIRMED, TENTATIVE, CANCELLED
    Visibility      Visibility   // PUBLIC, PRIVATE, CONFIDENTIAL
    Attachments     []Attachment
    ConferenceData  *ConferenceData
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

type RecurrenceRule struct {
    Frequency   Frequency  // DAILY, WEEKLY, MONTHLY, YEARLY
    Interval    int
    Count       int
    Until       *time.Time
    ByDay       []Weekday
    ByMonth     []int
    ByMonthDay  []int
}
```

**複雑なロジック**:
1. **リカレンス計算**: RFC 5545 (iCalendar) 準拠の繰り返しイベント生成
2. **コンフリクト検出**: 時間帯の重複をリアルタイムでチェック
3. **タイムゾーン変換**: 複数タイムゾーン間の正確な時刻変換

**パフォーマンス最適化**:
- イベント検索に Elasticsearch を使用（フルテキスト検索、ファジーマッチング）
- 頻繁にアクセスされるイベントは Redis にキャッシュ
- リカレンスイベントは事前計算してマテリアライズドビューに格納

### 3.3 Resource Service

**責務**: 会議室、設備などのリソース管理と予約

**主要API**:
```protobuf
service ResourceService {
  rpc CreateResource(CreateResourceRequest) returns (Resource);
  rpc GetResource(GetResourceRequest) returns (Resource);
  rpc UpdateResource(UpdateResourceRequest) returns (Resource);
  rpc DeleteResource(DeleteResourceRequest) returns (Empty);
  rpc ListResources(ListResourcesRequest) returns (ListResourcesResponse);
  rpc CheckAvailability(AvailabilityRequest) returns (AvailabilityResponse);
  rpc ReserveResource(ReserveRequest) returns (Reservation);
  rpc ReleaseResource(ReleaseRequest) returns (Empty);
}
```

**データモデル**:
```go
type Resource struct {
    ID          string
    Name        string
    Type        ResourceType  // ROOM, EQUIPMENT, PARKING
    Location    Location
    Capacity    int
    Features    []Feature     // PROJECTOR, WHITEBOARD, VIDEO_CONF
    IsAvailable bool
    Metadata    map[string]string
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

type Reservation struct {
    ID         string
    ResourceID string
    EventID    string
    UserID     string
    StartTime  time.Time
    EndTime    time.Time
    Status     ReservationStatus
    CreatedAt  time.Time
}
```

**ビジネスロジック**:
- **競合制御**: 楽観的ロック（バージョニング）による同時予約の防止
- **自動解放**: 使用されていない予約の自動キャンセル（15分ルール）
- **スマート推奨**: 空き状況、設備、参加者の位置を考慮した会議室提案

### 3.4 Notification Service

**責務**: リマインダー、通知の管理と送信

**主要API**:
```protobuf
service NotificationService {
  rpc CreateNotification(CreateNotificationRequest) returns (Notification);
  rpc SendNotification(SendNotificationRequest) returns (SendResponse);
  rpc ScheduleNotification(ScheduleRequest) returns (ScheduledNotification);
  rpc GetUserNotifications(GetNotificationsRequest) returns (NotificationsResponse);
  rpc MarkAsRead(MarkAsReadRequest) returns (Empty);
  rpc UpdatePreferences(UpdatePreferencesRequest) returns (Preferences);
}
```

**通知チャネル**:
- **Email**: Amazon SES（送信レート: 50,000通/日）
- **Push**: Firebase Cloud Messaging, APNs
- **SMS**: Amazon SNS（重要通知のみ）
- **In-App**: WebSocket によるリアルタイム通知

**配信戦略**:
- **バッチ処理**: 定期リマインダーはバッチで処理（5分間隔）
- **優先度キュー**: 緊急通知は即座に送信
- **レート制限**: ユーザーあたりの通知数を制限（スパム防止）
- **リトライロジック**: 指数バックオフによる再送（最大3回）

### 3.5 Sync Service

**責務**: 外部カレンダーシステムとの同期

**対応システム**:
- Google Calendar (CalDAV/CardDAV)
- Microsoft Outlook (Exchange Web Services)
- Apple Calendar (CalDAV)
- iCal (RFC 5545 format)

**主要API**:
```protobuf
service SyncService {
  rpc ConnectExternalCalendar(ConnectRequest) returns (Connection);
  rpc SyncCalendar(SyncRequest) returns (SyncResponse);
  rpc GetSyncStatus(GetSyncStatusRequest) returns (SyncStatus);
  rpc DisconnectCalendar(DisconnectRequest) returns (Empty);
  rpc ResolveConflict(ConflictResolutionRequest) returns (Resolution);
}
```

**同期戦略**:
- **増分同期**: 変更差分のみを同期（フルシンクは初回のみ）
- **双方向同期**: 変更を両方向に反映（コンフリクト解決ロジックあり）
- **同期頻度**: 5分間隔（ユーザー設定で変更可能）
- **コンフリクト解決**: Last-Write-Wins（タイムスタンプベース）、ユーザー選択オプション

**データ整合性**:
- **ベクタークロック**: 分散システム間の因果関係追跡
- **イベントソーシング**: すべての変更をイベントログに記録
- **監査ログ**: 同期履歴の完全な追跡

## 4. データアーキテクチャ

### 4.1 データベース設計

**PostgreSQL スキーマ**:

```sql
-- Calendars Table
CREATE TABLE calendars (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    color VARCHAR(7),
    timezone VARCHAR(50) NOT NULL,
    is_public BOOLEAN DEFAULT FALSE,
    settings JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    CONSTRAINT valid_color CHECK (color ~ '^#[0-9A-Fa-f]{6}$')
);

CREATE INDEX idx_calendars_owner_id ON calendars(owner_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_calendars_updated_at ON calendars(updated_at DESC);

-- Events Table (Partitioned by month)
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calendar_id UUID NOT NULL REFERENCES calendars(id),
    organizer_id UUID NOT NULL REFERENCES users(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    location VARCHAR(500),
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL,
    timezone VARCHAR(50) NOT NULL,
    is_all_day BOOLEAN DEFAULT FALSE,
    recurrence_rule JSONB,
    status VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED',
    visibility VARCHAR(20) NOT NULL DEFAULT 'PUBLIC',
    attachments JSONB,
    conference_data JSONB,
    metadata JSONB,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    CONSTRAINT valid_status CHECK (status IN ('CONFIRMED', 'TENTATIVE', 'CANCELLED')),
    CONSTRAINT valid_visibility CHECK (visibility IN ('PUBLIC', 'PRIVATE', 'CONFIDENTIAL')),
    CONSTRAINT valid_time_range CHECK (end_time > start_time)
) PARTITION BY RANGE (start_time);

-- Create partitions for each month
CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
-- ... (additional partitions)

CREATE INDEX idx_events_calendar_id ON events(calendar_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_organizer_id ON events(organizer_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_time_range ON events(start_time, end_time) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_search ON events USING gin(to_tsvector('english', title || ' ' || description));

-- Attendees Table
CREATE TABLE attendees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    response_status VARCHAR(20) DEFAULT 'NEEDS_ACTION',
    is_optional BOOLEAN DEFAULT FALSE,
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT valid_response_status CHECK (response_status IN ('NEEDS_ACTION', 'ACCEPTED', 'DECLINED', 'TENTATIVE')),
    CONSTRAINT unique_event_attendee UNIQUE (event_id, email)
);

CREATE INDEX idx_attendees_event_id ON attendees(event_id);
CREATE INDEX idx_attendees_user_id ON attendees(user_id);

-- Resources Table
CREATE TABLE resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    location_building VARCHAR(100),
    location_floor INTEGER,
    location_room VARCHAR(50),
    capacity INTEGER,
    features TEXT[],
    is_available BOOLEAN DEFAULT TRUE,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    CONSTRAINT valid_type CHECK (type IN ('ROOM', 'EQUIPMENT', 'PARKING', 'OTHER'))
);

CREATE INDEX idx_resources_type ON resources(type) WHERE deleted_at IS NULL;
CREATE INDEX idx_resources_features ON resources USING gin(features);

-- Reservations Table
CREATE TABLE reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id UUID NOT NULL REFERENCES resources(id),
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED',
    version INTEGER DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT valid_reservation_status CHECK (status IN ('CONFIRMED', 'CANCELLED', 'NO_SHOW')),
    CONSTRAINT valid_reservation_time CHECK (end_time > start_time),
    CONSTRAINT unique_resource_time UNIQUE (resource_id, start_time, end_time)
);

CREATE INDEX idx_reservations_resource_time ON reservations(resource_id, start_time, end_time) WHERE status = 'CONFIRMED';
CREATE INDEX idx_reservations_event_id ON reservations(event_id);
```

### 4.2 データパーティショニング戦略

**水平パーティショニング**:
- **Events テーブル**: 月単位でパーティション（過去のデータアクセス頻度低減）
- **Audit ログ**: 日単位でパーティション（データ保持ポリシー適用容易化）

**シャーディング**:
- **Phase 1**: 単一データベース（~5M ユーザーまで）
- **Phase 2**: ユーザーIDベースのシャーディング（5M+ ユーザー）
  - Shard キー: `user_id % shard_count`
  - クロスシャードクエリは避け、アプリケーション層で集約

### 4.3 キャッシング戦略

**Redis キャッシュ階層**:

```yaml
Cache Layers:
  L1 (Application):
    - In-memory cache (Go sync.Map)
    - TTL: 30 seconds
    - Use case: Hot data (current user's calendar)

  L2 (Redis):
    - Distributed cache
    - TTL: 5 minutes (calendars), 2 minutes (events)
    - Use case: Frequently accessed data
    - Keys: "calendar:{id}", "event:{id}", "user:{id}:calendars"

  L3 (Database Read Replicas):
    - PostgreSQL read replicas (3 replicas)
    - Load balancing: Round-robin
    - Use case: Cache miss時のフォールバック
```

**キャッシュ無効化**:
- **Write-Through**: 書き込み時に即座にキャッシュを更新
- **Event-Driven Invalidation**: Kafka イベントによる分散キャッシュ無効化
- **TTL ベース**: 定期的な自動削除

**キャッシュキー設計**:
```
calendar:{calendar_id}
user:{user_id}:calendars
event:{event_id}
event:{event_id}:attendees
resource:{resource_id}:availability:{date}
```

### 4.4 Elasticsearch インデックス設計

**イベント検索インデックス**:

```json
{
  "mappings": {
    "properties": {
      "event_id": { "type": "keyword" },
      "calendar_id": { "type": "keyword" },
      "organizer_id": { "type": "keyword" },
      "title": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text" },
      "location": { "type": "text" },
      "start_time": { "type": "date" },
      "end_time": { "type": "date" },
      "attendee_emails": { "type": "keyword" },
      "attendee_names": { "type": "text" },
      "tags": { "type": "keyword" },
      "status": { "type": "keyword" },
      "created_at": { "type": "date" },
      "suggest": {
        "type": "completion",
        "contexts": [
          {
            "name": "user_id",
            "type": "category"
          }
        ]
      }
    }
  },
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 2,
    "refresh_interval": "5s"
  }
}
```

**検索クエリ例**:
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "quarterly business review",
            "fields": ["title^3", "description", "location"],
            "fuzziness": "AUTO"
          }
        }
      ],
      "filter": [
        {
          "range": {
            "start_time": {
              "gte": "2025-01-01",
              "lte": "2025-12-31"
            }
          }
        },
        {
          "term": { "status": "CONFIRMED" }
        }
      ]
    }
  },
  "sort": [
    { "start_time": "asc" }
  ],
  "highlight": {
    "fields": {
      "title": {},
      "description": {}
    }
  }
}
```

## 5. インフラストラクチャ

### 5.1 AWS アーキテクチャ

**コンピュートリソース**:
- **EKS Cluster**: Kubernetes 1.28
  - Node Groups:
    - General Purpose (t3.large): 10-50 nodes (auto-scaling)
    - Compute Optimized (c6i.xlarge): 5-20 nodes
  - Pod Auto-scaling: HPA + KEDA (Kubernetes Event Driven Autoscaling)
  - Cluster Autoscaler: ノード数の自動調整

**データベース**:
- **RDS PostgreSQL 15**:
  - Instance: db.r6g.2xlarge (8 vCPU, 64 GB RAM)
  - Multi-AZ: 3 AZ deployment
  - Read Replicas: 3 replicas (異なるAZ)
  - Storage: 2 TB gp3 (16,000 IOPS)
  - Backup: 自動バックアップ (7日保持) + スナップショット

**キャッシュ**:
- **ElastiCache Redis**:
  - Instance: cache.r6g.xlarge
  - Cluster Mode: Enabled (3 shards, 2 replicas per shard)
  - Total Capacity: 192 GB
  - Eviction Policy: allkeys-lru

**メッセージング**:
- **MSK (Managed Streaming for Kafka)**:
  - Kafka Version: 3.5
  - Brokers: 6 brokers (2 per AZ)
  - Instance: kafka.m5.xlarge
  - Storage: 1 TB per broker
  - Topics: calendar.events, notifications, sync.requests

**ストレージ**:
- **S3**:
  - Attachments Bucket: Standard storage class
  - Backups Bucket: Glacier storage class
  - Lifecycle Policy: 30日後に Infrequent Access、90日後に Glacier

**ネットワーク**:
- **VPC**: 10.0.0.0/16
  - Public Subnets: 3 (各AZ)
  - Private Subnets: 3 (各AZ)
  - Database Subnets: 3 (各AZ)
- **NAT Gateway**: 3 (各AZ、冗長性確保)
- **Application Load Balancer**:
  - Cross-Zone Load Balancing: Enabled
  - SSL/TLS: AWS Certificate Manager
  - Target Groups: gRPC services

### 5.2 コスト見積もり

**月間コスト概算**（5,000ユーザー想定）:

| リソース | 仕様 | 月額コスト |
|---------|------|----------|
| EKS Cluster | 1 cluster + 20 nodes (t3.large avg) | $2,880 |
| RDS PostgreSQL | db.r6g.2xlarge Multi-AZ + replicas | $3,200 |
| ElastiCache | cache.r6g.xlarge cluster | $1,100 |
| MSK | 6 kafka.m5.xlarge brokers | $2,160 |
| S3 | 500 GB storage + transfer | $50 |
| ALB | 2 load balancers | $40 |
| Data Transfer | 5 TB/month | $450 |
| CloudWatch | Metrics + Logs | $200 |
| **合計** | | **$10,080/月** |

**年間コスト**: 約 $120,960 (約 ¥18M、1$ = 150円換算)

**スケーリングシナリオ**:
- 10,000 ユーザー: 約 $18,000/月
- 50,000 ユーザー: 約 $45,000/月

## 6. セキュリティアーキテクチャ

### 6.1 認証・認可

**認証 (AuthN)**:
- **OAuth 2.0 + OpenID Connect**:
  - Authorization Server: Auth0 / AWS Cognito
  - Token Type: JWT (JSON Web Token)
  - Token Lifetime: Access Token (15分), Refresh Token (7日)
  - MFA: TOTP (Time-based One-Time Password) 必須

**認可 (AuthZ)**:
- **RBAC (Role-Based Access Control)**:
  ```yaml
  Roles:
    - Admin: Full access to all resources
    - Owner: Full access to owned calendars/events
    - Editor: Can modify calendar/events
    - Viewer: Read-only access
    - Guest: Limited access to specific events
  ```

- **ABAC (Attribute-Based Access Control)**:
  - リソース属性（visibility, sharing_settings）に基づく動的アクセス制御
  - ポリシー例: "Private events は owner と explicit attendees のみアクセス可能"

**API認証**:
```go
// JWT Validation Middleware
func ValidateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := extractToken(r)

        // Verify signature
        claims, err := verifyToken(token)
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Check expiration
        if claims.ExpiresAt < time.Now().Unix() {
            http.Error(w, "Token expired", http.StatusUnauthorized)
            return
        }

        // Add user context
        ctx := context.WithValue(r.Context(), "user_id", claims.UserID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 6.2 データ保護

**暗号化**:
- **転送中**: TLS 1.3 (全通信)
- **保存時**:
  - RDS: AES-256 encryption at rest
  - S3: Server-Side Encryption (SSE-S3)
  - Redis: Encryption in transit and at rest
- **Application Level**:
  - PII (個人識別情報) は追加の暗号化層（AES-256-GCM）
  - Encryption Keys: AWS KMS (Key Management Service)

**データマスキング**:
```go
// Sensitive data masking
func MaskEmail(email string) string {
    parts := strings.Split(email, "@")
    if len(parts) != 2 {
        return "***"
    }

    username := parts[0]
    domain := parts[1]

    if len(username) <= 2 {
        return "**@" + domain
    }

    return username[:2] + "***@" + domain
}

// Example: john.doe@example.com → jo***@example.com
```

**データ保持ポリシー**:
- **アクティブデータ**: 無期限（ユーザーがアクティブな限り）
- **削除済みデータ**: 30日間ソフトデリート後に完全削除
- **監査ログ**: 7年間保持（コンプライアンス要件）
- **バックアップ**: 90日間保持

### 6.3 ネットワークセキュリティ

**Defense in Depth**:
```
Internet
    ↓
CloudFront + WAF (Layer 7 DDoS protection, SQL injection filtering)
    ↓
API Gateway (Rate limiting, IP whitelist)
    ↓
Application Load Balancer (SSL termination)
    ↓
EKS Cluster (Network Policies, Security Groups)
    ↓
Services (mTLS, Service Mesh)
    ↓
Data Layer (Private subnets, No internet access)
```

**Network Policies (Kubernetes)**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: calendar-service-policy
spec:
  podSelector:
    matchLabels:
      app: calendar-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: bff-layer
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### 6.4 コンプライアンス

**GDPR対応**:
- **Right to Access**: ユーザーデータのエクスポートAPI提供
- **Right to Erasure**: 完全なデータ削除機能（30日以内）
- **Data Portability**: 標準フォーマット（iCal, JSON）でのデータエクスポート
- **Consent Management**: 明示的な同意管理システム

**監査**:
- すべての API アクセスをログに記録
- 監査ログは改ざん防止（Write-Once storage）
- 定期的なセキュリティ監査（四半期ごと）
- ペネトレーションテスト（年2回）

## 7. 可観測性とモニタリング

### 7.1 メトリクス収集

**Prometheusメトリクス**:

```go
// Custom metrics
var (
    eventCreationDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "event_creation_duration_seconds",
            Help:    "Duration of event creation operations",
            Buckets: prometheus.DefBuckets,
        },
        []string{"status"},
    )

    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_websocket_connections",
            Help: "Number of active WebSocket connections",
        },
    )

    cacheHitRate = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "cache_hits_total",
            Help: "Total number of cache hits",
        },
        []string{"cache_layer", "key_type"},
    )
)
```

**収集するメトリクス**:
- **Infrastructure**: CPU, Memory, Disk I/O, Network
- **Application**: Request rate, Error rate, Duration (RED method)
- **Business**: Events created, Meetings scheduled, Active users
- **Database**: Query latency, Connection pool, Deadlocks
- **Cache**: Hit rate, Miss rate, Evictions

### 7.2 分散トレーシング

**Jaeger統合**:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func CreateEvent(ctx context.Context, req *CreateEventRequest) (*Event, error) {
    tracer := otel.Tracer("calendar-service")
    ctx, span := tracer.Start(ctx, "CreateEvent")
    defer span.End()

    // Validate request
    ctx, validateSpan := tracer.Start(ctx, "ValidateRequest")
    if err := validateRequest(req); err != nil {
        validateSpan.SetStatus(codes.Error, err.Error())
        validateSpan.End()
        return nil, err
    }
    validateSpan.End()

    // Save to database
    ctx, dbSpan := tracer.Start(ctx, "SaveToDatabase")
    event, err := db.SaveEvent(ctx, req)
    dbSpan.End()

    // Publish event
    ctx, publishSpan := tracer.Start(ctx, "PublishKafkaEvent")
    err = kafka.Publish(ctx, "calendar.events", event)
    publishSpan.End()

    span.SetAttributes(
        attribute.String("event.id", event.ID),
        attribute.String("calendar.id", event.CalendarID),
    )

    return event, nil
}
```

**トレース戦略**:
- すべてのサービス間呼び出しをトレース
- データベースクエリの詳細なトレース
- 外部API呼び出しのトレース
- サンプリングレート: 100%（開発）、10%（本番）

### 7.3 ログ管理

**構造化ログ**:

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()

logger.Info("Event created",
    zap.String("event_id", event.ID),
    zap.String("user_id", user.ID),
    zap.String("calendar_id", calendar.ID),
    zap.Time("start_time", event.StartTime),
    zap.Duration("duration", time.Since(start)),
)
```

**ログレベル**:
- **ERROR**: システムエラー、例外
- **WARN**: 予期しない状況、リトライ
- **INFO**: 重要なビジネスイベント
- **DEBUG**: 詳細なデバッグ情報（開発環境のみ）

**ELKスタック**:
- **Elasticsearch**: ログストレージ
- **Logstash**: ログ集約、パース
- **Kibana**: 可視化、検索
- **Retention**: 30日間（ホット）、90日間（ウォーム）、365日間（コールド）

### 7.4 アラート設定

**重要なアラート**:

```yaml
Alerts:
  - name: HighErrorRate
    condition: error_rate > 5%
    duration: 5m
    severity: critical
    notification: PagerDuty

  - name: HighLatency
    condition: p95_latency > 500ms
    duration: 10m
    severity: warning
    notification: Slack

  - name: DatabaseConnectionPoolExhausted
    condition: db_connections > 90%
    duration: 5m
    severity: critical
    notification: PagerDuty

  - name: CacheHitRateLow
    condition: cache_hit_rate < 70%
    duration: 15m
    severity: warning
    notification: Slack

  - name: KafkaConsumerLag
    condition: consumer_lag > 10000
    duration: 10m
    severity: critical
    notification: PagerDuty
```

## 8. ディザスタリカバリ

### 8.1 バックアップ戦略

**RDS バックアップ**:
- **自動バックアップ**: 毎日 3:00 AM UTC
- **スナップショット**: 週次手動スナップショット（日曜日）
- **クロスリージョンレプリケーション**: us-west-2 (DR site)
- **リストア時間**: 30分以内（ポイントインタイムリカバリ）

**S3バックアップ**:
- **Versioning**: Enabled
- **Cross-Region Replication**: DR site
- **Lifecycle Policy**: 90日後に Glacier

### 8.2 DR計画

**RPO (Recovery Point Objective)**: 5分
**RTO (Recovery Time Objective)**: 1時間

**フェイルオーバー手順**:
1. DNS フェイルオーバー（Route 53 Health Check）
2. DRサイトのRDSリードレプリカを昇格
3. アプリケーションをDRリージョンで起動
4. トラフィックをDRサイトにルーティング

**定期的な訓練**:
- DR訓練: 四半期ごと
- カオスエンジニアリング: 月次（本番環境で実施）

---

**ドキュメントバージョン**: 1.0
**最終更新日**: 2025-12-05
**レビュアー**: Technical Architecture Review Board
**次回レビュー**: 2025-03-05
