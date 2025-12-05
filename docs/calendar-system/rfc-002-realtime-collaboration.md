# RFC-002: リアルタイムコラボレーション機能の実装

**ステータス**: Proposed
**作成日**: 2025-12-05
**担当者**: Frontend Platform Team, Backend Infrastructure Team
**レビュアー**: Engineering Leadership, Product Management
**想定工数**: 12 週間 (3エンジニア)
**優先度**: P1 (High)

## エグゼクティブサマリー

本提案は、Google Docs スタイルのリアルタイムコラボレーション機能をカレンダーシステムに実装するものです。複数ユーザーが同時にイベントを編集し、変更を即座に他のユーザーに反映することで、チームでのスケジュール調整を劇的に効率化します。

**期待される効果**:
- チームミーティングの調整時間を **75% 削減** (平均12分 → 3分)
- リアルタイム更新による **コンフリクトエラーを 90% 削減**
- ユーザーエンゲージメントを **45% 向上**
- モバイル・デスクトップ間のシームレスな体験を提供

**投資対効果**:
- 年間時間削減価値: **¥95M** (全社5,000名規模)
- 開発コスト: **¥36M** (初年度)
- ROI: **164%** (初年度)

## 1. 背景と動機

### 1.1 現状の課題

現在のカレンダーシステムは、従来のリクエスト・レスポンスモデルに基づいており、以下の問題が発生しています：

#### 同時編集時のコンフリクト
- **編集コンフリクト**: 複数ユーザーが同時にイベントを編集すると、後から保存したユーザーの変更が上書きされる
- **発生頻度**: 月間約 **1,200件** のコンフリクトエラー
- **ユーザーフラストレーション**: サポートチケットの **18%** がこの問題に関連

#### 情報の同期遅延
- **ポーリング方式**: 現在は30秒ごとのポーリングで更新をチェック
- **遅延時間**: 平均 **15秒** の更新遅延（最悪 30秒）
- **影響**: リアルタイム性が求められる緊急ミーティングの調整に支障

#### チームコラボレーションの非効率性
- **非同期コミュニケーション**: チャットやメールで並行して調整が必要
- **ツール切り替え**: カレンダー ↔ Slack/Email を頻繁に往復
- **調整時間**: チームミーティング1件あたり平均 **12分**

### 1.2 ユーザーストーリー

**As a Team Lead**, I want to:
- チームメンバーと同時にカレンダーイベントを編集し、リアルタイムで変更を確認したい
- 誰が今編集中かを可視化し、コンフリクトを未然に防ぎたい
- モバイルでもデスクトップと同じリアルタイム体験を得たい

**As an Executive Assistant**, I want to:
- エグゼクティブの代理でミーティングを調整する際、リアルタイムでフィードバックを得たい
- 複数のステークホルダーと同時に予定を調整したい
- 変更履歴を追跡し、誰が何を変更したかを把握したい

### 1.3 競合分析

| 製品 | リアルタイム編集 | 同時編集者表示 | オフライン対応 | モバイル対応 |
|------|----------------|--------------|--------------|-------------|
| Google Calendar | △ (限定的) | ✗ | △ | ○ |
| Microsoft Outlook | △ (Exchange のみ) | △ | ○ | ○ |
| Notion Calendar | ○ | ○ | △ | ○ |
| Cron | ○ | ○ | △ | △ |
| **提案システム** | ○○ | ○○ | ○ | ○○ |

**差別化ポイント**:
- **完全なリアルタイム性**: < 100ms の更新遅延
- **Operational Transformation (OT)**: コンフリクトフリーな同時編集
- **豊富なプレゼンス情報**: カーソル位置、編集中フィールド、入力中表示
- **オフラインファースト**: ネットワーク断でも編集可能、再接続時に自動同期

## 2. 技術的アプローチ

### 2.1 システムアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Web Client  │  │ Mobile Client│  │Desktop Client│         │
│  │  (React)     │  │(React Native)│  │  (Electron)  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │  WebSocket      │                   │                 │
│         └─────────────────┴───────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway + Load Balancer                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   AWS ALB (Application Load Balancer)                    │  │
│  │   - WebSocket support                                    │  │
│  │   - Sticky sessions                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 Realtime Collaboration Service                  │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐        │
│  │ WebSocket   │ │   OT Engine  │ │  Presence        │        │
│  │ Manager     │ │ (Operational │ │  Manager         │        │
│  │ (Go)        │ │Transform)    │ │  (Go)            │        │
│  └─────────────┘ └──────────────┘ └──────────────────┘        │
│                                                                  │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐        │
│  │ Conflict    │ │  Version     │ │  Change          │        │
│  │ Resolution  │ │  Control     │ │  Broadcasting    │        │
│  │ (CRDT)      │ │  (Vector     │ │  (Pub/Sub)       │        │
│  └─────────────┘ │  Clocks)     │ └──────────────────┘        │
│                  └──────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Message Broker                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   Redis Pub/Sub + Redis Streams                          │  │
│  │   - Low latency (<10ms)                                  │  │
│  │   - Event streaming and replay                           │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                       Event Store                               │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐        │
│  │ PostgreSQL  │ │    Redis     │ │   Elasticsearch  │        │
│  │(Event Log)  │ │(State Cache) │ │(Search & Audit)  │        │
│  └─────────────┘ └──────────────┘ └──────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 WebSocket通信プロトコル

**接続確立**:

```javascript
// Client-side connection
const ws = new WebSocket('wss://api.calendar.company.com/realtime');

ws.onopen = () => {
  // Send authentication
  ws.send(JSON.stringify({
    type: 'AUTH',
    payload: {
      token: jwt_token,
      client_id: uuid(),
      user_id: user.id
    }
  }));

  // Subscribe to event updates
  ws.send(JSON.stringify({
    type: 'SUBSCRIBE',
    payload: {
      resource: 'event',
      resource_id: event_id
    }
  }));
};

ws.onmessage = (message) => {
  const data = JSON.parse(message.data);
  handleMessage(data);
};
```

**メッセージフォーマット**:

```typescript
// Base Message Interface
interface Message {
  type: MessageType;
  id: string;  // Unique message ID
  timestamp: number;  // Unix timestamp (ms)
  payload: any;
}

// Message Types
enum MessageType {
  // Connection
  AUTH = 'AUTH',
  AUTH_ACK = 'AUTH_ACK',
  PING = 'PING',
  PONG = 'PONG',

  // Subscription
  SUBSCRIBE = 'SUBSCRIBE',
  UNSUBSCRIBE = 'UNSUBSCRIBE',
  SUBSCRIBE_ACK = 'SUBSCRIBE_ACK',

  // Operations
  OP = 'OP',              // Operation (edit)
  OP_ACK = 'OP_ACK',      // Operation acknowledged
  OP_BROADCAST = 'OP_BROADCAST',  // Broadcasted operation

  // Presence
  PRESENCE_JOIN = 'PRESENCE_JOIN',
  PRESENCE_LEAVE = 'PRESENCE_LEAVE',
  PRESENCE_UPDATE = 'PRESENCE_UPDATE',
  PRESENCE_STATE = 'PRESENCE_STATE',

  // Sync
  SYNC_REQUEST = 'SYNC_REQUEST',
  SYNC_RESPONSE = 'SYNC_RESPONSE',

  // Errors
  ERROR = 'ERROR'
}

// Operation Message
interface OperationMessage extends Message {
  type: 'OP';
  payload: {
    resource_id: string;
    resource_type: 'event' | 'calendar';
    operation: Operation;
    version: number;  // Vector clock version
    client_id: string;
  };
}

// Operation (based on JSON Patch RFC 6902)
interface Operation {
  op: 'add' | 'remove' | 'replace' | 'move' | 'copy' | 'test';
  path: string;  // JSON Pointer (RFC 6901)
  value?: any;
  from?: string;  // For 'move' and 'copy'
}

// Example Operations
const operations: Operation[] = [
  {
    op: 'replace',
    path: '/title',
    value: 'Updated Meeting Title'
  },
  {
    op: 'replace',
    path: '/start_time',
    value: '2025-12-15T14:00:00Z'
  },
  {
    op: 'add',
    path: '/attendees/-',
    value: {
      user_id: 'user_123',
      email: 'john@example.com',
      response_status: 'NEEDS_ACTION'
    }
  },
  {
    op: 'remove',
    path: '/attendees/2'
  }
];
```

### 2.3 Operational Transformation (OT) アルゴリズム

**OT の目的**: 同時編集時のコンフリクトを自動的に解決し、最終的な一貫性を保証

**基本原則**:
1. **Convergence**: すべてのクライアントが同じ操作シーケンスを適用すると、同じ最終状態になる
2. **Causality Preservation**: 操作の因果関係を保持
3. **Intention Preservation**: ユーザーの意図した編集を可能な限り保持

**アルゴリズム実装** (Simplified):

```go
// OT Engine
type OTEngine struct {
    history []Operation
    version int
}

// Transform function: transforms operation A against operation B
func Transform(opA, opB Operation) (Operation, Operation) {
    // Example: Two users editing the same field

    // Case 1: Both replace the same path
    if opA.Op == "replace" && opB.Op == "replace" && opA.Path == opB.Path {
        // Priority: Last-Write-Wins (based on timestamp or user priority)
        // Or: Merge values (for text fields, use OT text algorithm)
        if opA.Timestamp > opB.Timestamp {
            return opA, Operation{Op: "noop"}
        } else {
            return Operation{Op: "noop"}, opB
        }
    }

    // Case 2: One adds, one removes at the same array index
    if opA.Op == "add" && opB.Op == "remove" {
        // Adjust indices
        opA.Path = adjustPathForRemove(opA.Path, opB.Path)
        return opA, opB
    }

    // Case 3: Independent paths - no conflict
    if !pathsConflict(opA.Path, opB.Path) {
        return opA, opB
    }

    // ... more cases

    return opA, opB
}

// Apply operation to document state
func (e *OTEngine) Apply(op Operation, state *Event) (*Event, error) {
    // Validate operation
    if err := e.validate(op); err != nil {
        return nil, err
    }

    // Apply JSON Patch
    newState, err := applyJSONPatch(state, op)
    if err != nil {
        return nil, err
    }

    // Add to history
    e.history = append(e.history, op)
    e.version++

    return newState, nil
}
```

**CRDT (Conflict-free Replicated Data Type) の活用**:

より高度なケースでは、CRDT を使用してコンフリクトフリーな同時編集を実現：

```go
// CRDT-based Event model
type CRDTEvent struct {
    ID          LWWID              // Last-Write-Wins ID
    Title       LWWRegister        // Last-Write-Wins Register
    StartTime   LWWRegister
    EndTime     LWWRegister
    Attendees   ORSet              // Observed-Remove Set
    Tags        GSet               // Grow-only Set
    Description RGA                // Replicated Growable Array (for rich text)
}

// LWW Register (Last-Write-Wins)
type LWWRegister struct {
    Value     interface{}
    Timestamp int64
    ClientID  string
}

func (r *LWWRegister) Set(value interface{}, timestamp int64, clientID string) {
    if timestamp > r.Timestamp ||
       (timestamp == r.Timestamp && clientID > r.ClientID) {
        r.Value = value
        r.Timestamp = timestamp
        r.ClientID = clientID
    }
}

// ORSet (Observed-Remove Set) for attendees
type ORSet struct {
    adds    map[string][]string  // element -> [unique tags]
    removes map[string][]string
}

func (s *ORSet) Add(element string, tag string) {
    s.adds[element] = append(s.adds[element], tag)
}

func (s *ORSet) Remove(element string) {
    // Observe all current tags and mark for removal
    s.removes[element] = append(s.removes[element], s.adds[element]...)
}

func (s *ORSet) Contains(element string) bool {
    addTags := s.adds[element]
    removeTags := s.removes[element]

    // Element exists if any add tag is not in remove tags
    for _, addTag := range addTags {
        removed := false
        for _, removeTag := range removeTags {
            if addTag == removeTag {
                removed = true
                break
            }
        }
        if !removed {
            return true
        }
    }
    return false
}
```

### 2.4 プレゼンス管理

**プレゼンス情報**:

```typescript
interface Presence {
  user_id: string;
  user: {
    id: string;
    name: string;
    avatar_url: string;
    color: string;  // Assigned color for cursor/highlight
  };
  cursor: {
    field: string;  // Current editing field (e.g., "title", "start_time")
    position?: number;  // For text fields
  };
  is_typing: boolean;
  last_seen: number;  // Unix timestamp
  client_id: string;
}
```

**プレゼンスブロードキャスト**:

```go
// Presence Manager
type PresenceManager struct {
    sessions map[string]*Session  // resource_id -> sessions
    redis    *redis.Client
}

func (pm *PresenceManager) JoinResource(resourceID, userID, clientID string) {
    presence := Presence{
        UserID:   userID,
        ClientID: clientID,
        JoinedAt: time.Now().Unix(),
    }

    // Add to Redis (with TTL)
    key := fmt.Sprintf("presence:%s", resourceID)
    pm.redis.HSet(ctx, key, clientID, marshalJSON(presence))
    pm.redis.Expire(ctx, key, 5*time.Minute)

    // Broadcast to other users
    pm.broadcast(resourceID, Message{
        Type: "PRESENCE_JOIN",
        Payload: presence,
    })
}

func (pm *PresenceManager) UpdatePresence(resourceID, clientID string, update PresenceUpdate) {
    // Update Redis
    key := fmt.Sprintf("presence:%s", resourceID)
    pm.redis.HSet(ctx, key, clientID, marshalJSON(update))

    // Broadcast
    pm.broadcast(resourceID, Message{
        Type: "PRESENCE_UPDATE",
        Payload: update,
    })
}

// Heartbeat to keep presence alive
func (pm *PresenceManager) Heartbeat(resourceID, clientID string) {
    key := fmt.Sprintf("presence:%s", resourceID)
    pm.redis.Expire(ctx, key, 5*time.Minute)
}
```

**UI表示**:

```tsx
// React Component for Presence
const PresenceIndicators: React.FC<{ presences: Presence[] }> = ({ presences }) => {
  return (
    <div className="presence-indicators">
      {presences.map((p) => (
        <div key={p.client_id} className="presence-avatar">
          <img src={p.user.avatar_url} alt={p.user.name} />
          <div
            className="presence-dot"
            style={{ backgroundColor: p.user.color }}
          />
          {p.is_typing && <TypingIndicator />}
        </div>
      ))}
    </div>
  );
};

// Cursor overlay for field being edited
const CursorOverlay: React.FC<{ presence: Presence }> = ({ presence }) => {
  if (!presence.cursor.field) return null;

  return (
    <div
      className="cursor-overlay"
      style={{
        borderColor: presence.user.color,
        position: 'absolute',
        // Position based on field
      }}
    >
      <span className="cursor-label">{presence.user.name}</span>
    </div>
  );
};
```

### 2.5 オフライン対応とSync

**オフラインキュー**:

```typescript
class OfflineQueue {
  private db: IndexedDB;
  private queue: Operation[] = [];

  // Add operation to offline queue
  async enqueue(operation: Operation) {
    this.queue.push(operation);
    await this.db.put('offline_queue', operation);
  }

  // When back online, sync queued operations
  async syncWhenOnline() {
    if (!navigator.onLine) return;

    const operations = await this.db.getAll('offline_queue');

    for (const op of operations) {
      try {
        await this.sendOperation(op);
        await this.db.delete('offline_queue', op.id);
      } catch (error) {
        // Conflict detected, need manual resolution
        if (error.code === 'CONFLICT') {
          await this.handleConflict(op, error.serverState);
        }
      }
    }
  }

  // Handle conflicts from offline edits
  async handleConflict(localOp: Operation, serverState: Event) {
    // Show conflict resolution UI
    const resolution = await showConflictDialog({
      local: localOp,
      server: serverState,
    });

    if (resolution === 'keep_local') {
      // Force push local changes
      await this.sendOperation(localOp, { force: true });
    } else if (resolution === 'keep_server') {
      // Discard local changes
      await this.discardOperation(localOp);
    } else {
      // Merge manually
      const merged = await showMergeEditor(localOp, serverState);
      await this.sendOperation(merged);
    }
  }
}
```

**Delta Sync**:

```go
// Delta Sync: Only send changes since last sync
type SyncRequest struct {
    ResourceID  string
    LastVersion int
}

type SyncResponse struct {
    Operations []Operation  // All ops since last version
    CurrentVersion int
    State Event  // Full state (optional, for full resync)
}

func (s *SyncService) Sync(req SyncRequest) (*SyncResponse, error) {
    // Get operations since last version
    ops, err := s.getOperationsSince(req.ResourceID, req.LastVersion)
    if err != nil {
        return nil, err
    }

    // If too many operations, send full state instead
    if len(ops) > 100 {
        state, err := s.getFullState(req.ResourceID)
        if err != nil {
            return nil, err
        }
        return &SyncResponse{
            State: state,
            CurrentVersion: state.Version,
        }, nil
    }

    return &SyncResponse{
        Operations: ops,
        CurrentVersion: s.getCurrentVersion(req.ResourceID),
    }, nil
}
```

### 2.6 スケーラビリティ設計

**水平スケーリング**:

```
┌─────────────────────────────────────────────────────────┐
│                  Client Connections                     │
│        100,000 concurrent WebSocket connections         │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│               Load Balancer (AWS ALB)                   │
│              - Sticky sessions (IP hash)                │
│              - Health checks                            │
└─────────────────────────────────────────────────────────┘
                         ↓
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ WS Node 1│ WS Node 2│ WS Node 3│  ....... │WS Node 10│
│(10k conn)│(10k conn)│(10k conn)│          │(10k conn)│
└──────────┴──────────┴──────────┴──────────┴──────────┘
     ↓           ↓           ↓         ↓          ↓
┌─────────────────────────────────────────────────────────┐
│                  Redis Pub/Sub Cluster                  │
│            - Sharded by resource_id                     │
│            - 5 master + 5 replica nodes                 │
└─────────────────────────────────────────────────────────┘
```

**接続数見積もり**:
- 想定ユーザー数: 5,000
- 同時アクティブ率: 20% → 1,000 同時ユーザー
- デバイス数/ユーザー: 平均 1.5 → 1,500 接続
- ピーク時: 2倍 → 3,000 接続
- **余裕を持った設計**: 10,000 接続/ノード × 10ノード = **100,000 接続対応**

**コスト効率化**:
- Auto-scaling: トラフィックに応じて自動でノード数を調整
- Idle connection timeout: 5分間操作がない接続は自動切断
- Compression: WebSocket メッセージを gzip 圧縮 (70% サイズ削減)

## 3. 実装計画

### 3.1 フェーズ分け

#### Phase 1: MVP - Basic Realtime Updates (Week 1-4)

**目標**: 基本的なリアルタイム更新機能

**スコープ**:
- ✓ WebSocket 接続管理
- ✓ 単一イベントのリアルタイム更新
- ✓ Redis Pub/Sub による通知
- ✓ 基本的なプレゼンス表示 (誰が見ているか)
- ✓ Web クライアントのみ対応

**成功基準**:
- WebSocket 接続成功率 > 99%
- 更新遅延 < 500ms (P95)
- 100 同時接続でのストレステスト

**デリバラブル**:
- Realtime Service (Go)
- WebSocket API
- React hooks (useRealtimeEvent)
- 基本監視ダッシュボード

#### Phase 2: Collaborative Editing (Week 5-8)

**目標**: 同時編集機能の実装

**スコープ**:
- ✓ Operational Transformation (OT) エンジン
- ✓ コンフリクト解決ロジック
- ✓ 詳細なプレゼンス情報 (カーソル位置、編集中フィールド)
- ✓ 変更履歴の保存と表示
- ✓ Undo/Redo 機能

**成功基準**:
- コンフリクトエラー率 < 1%
- OT 変換精度 > 99.9%
- 同時編集ユーザー数 10名まで対応

**デリバラブル**:
- OT Engine
- CRDT データ構造
- Collaborative UI components
- Change history API

#### Phase 3: Mobile & Offline Support (Week 9-12)

**目標**: モバイル対応とオフライン機能

**スコープ**:
- ✓ React Native クライアント対応
- ✓ オフラインキューイング
- ✓ Delta Sync 実装
- ✓ コンフリクト解決 UI
- ✓ ネットワーク再接続時の自動同期

**成功基準**:
- モバイルアプリでのリアルタイム更新
- オフライン編集の成功率 > 95%
- 同期時のコンフリクト解決率 > 90% (自動)

**デリバラブル**:
- Mobile SDKs
- Offline queue implementation
- Conflict resolution UI
- Network resilience testing

### 3.2 タイムライン

```
Week  1  2  3  4  5  6  7  8  9  10 11 12
Phase 1 [==MVP==]
           └─ Internal Testing (Week 3-4)

Phase 2         [=Collaborative=]
                   └─ Beta Launch (Week 7)

Phase 3                     [=Mobile & Offline=]
                                └─ Full Launch (Week 12)

Milestones:
• Week 2:  WebSocket infra ready
• Week 4:  MVP Internal Launch
• Week 6:  OT engine complete
• Week 8:  Beta Launch (10% users)
• Week 10: Mobile apps ready
• Week 12: Full Production Launch
```

### 3.3 リソース計画

**エンジニアリングチーム**:
- Backend Engineer (Go): 2名 (12週間)
- Frontend Engineer (React): 1名 (12週間)
- Mobile Engineer (React Native): 1名 (4週間、Week 9-12)
- DevOps Engineer: 0.5名 (継続)
- QA Engineer: 0.5名 (継続)

**インフラコスト**:

| リソース | 仕様 | 月額コスト |
|---------|------|----------|
| WebSocket Servers | c5.large x 5 (auto-scaling) | $450 |
| Redis Cluster | cache.r6g.large x 10 (5 master + 5 replica) | $1,800 |
| PostgreSQL (event log) | 追加ストレージ 500GB | $100 |
| Data Transfer | 10 TB/month | $900 |
| **合計** | | **$3,250/月** |

**総コスト見積もり**:
- 人件費: ¥32M (エンジニア工数)
- インフラ: ¥1.6M (3ヶ月 + 運用)
- その他: ¥2.4M (ツール、テスト環境等)
- **合計**: **¥36M**

## 4. リスクと軽減策

### 4.1 技術的リスク

| リスク | 影響度 | 確率 | 軽減策 |
|--------|--------|------|--------|
| WebSocket接続の不安定性 | 高 | 中 | - 自動再接続ロジック<br>- Fallback to polling<br>- 詳細なエラーハンドリング |
| OT アルゴリズムのバグ | 最高 | 中 | - 徹底的な単体テスト<br>- Property-based testing<br>- 段階的ロールアウト |
| スケーラビリティの限界 | 高 | 低 | - 水平スケーリング設計<br>- ロードテスト (100k connections)<br>- Auto-scaling 設定 |
| Redis のメモリ不足 | 中 | 低 | - メモリ監視とアラート<br>- データの定期的なクリーンアップ<br>- Eviction policy 設定 |
| オフライン時のデータ損失 | 高 | 低 | - IndexedDB への永続化<br>- 複数デバイス間の同期<br>- 定期的なバックアップ |

### 4.2 ユーザー体験リスク

| リスク | 影響度 | 確率 | 軽減策 |
|--------|--------|------|--------|
| 編集コンフリクトの混乱 | 中 | 中 | - 明確な UI フィードバック<br>- 変更履歴の可視化<br>- Undo/Redo 機能 |
| プライバシー懸念 | 中 | 低 | - プレゼンス情報の制御<br>- "見えないモード" オプション<br>- 透明性の確保 |
| バッテリー消費 (モバイル) | 低 | 中 | - Adaptive polling (フォアグラウンド時のみリアルタイム)<br>- バックグラウンド時は5分ごとのポーリング |

## 5. 成功指標 (KPI)

### 5.1 ビジネスメトリクス

| 指標 | ベースライン | 目標 (Phase 3) | 測定方法 |
|------|-------------|---------------|---------|
| チームミーティング調整時間 | 12分 | 3分 | タイマー計測 |
| 編集コンフリクトエラー | 1,200件/月 | 120件/月 | エラーログ |
| ユーザーエンゲージメント | ベースライン | +45% | DAU, セッション時間 |
| 機能利用率 | 0% | 70% | アクティブユーザー率 |
| ユーザー満足度 | 3.8/5.0 | 4.6/5.0 | NPS調査 |

### 5.2 技術メトリクス

| 指標 | 目標 | 測定方法 |
|------|------|---------|
| WebSocket 接続成功率 | > 99.5% | Connection metrics |
| 更新遅延 (P95) | < 200ms | End-to-end latency |
| 更新遅延 (P99) | < 500ms | End-to-end latency |
| OT 変換成功率 | > 99.9% | Operation logs |
| システム可用性 | > 99.9% | Uptime monitoring |
| 同時接続数 | 10,000+ | WebSocket metrics |

### 5.3 パフォーマンスベンチマーク

**目標パフォーマンス**:

```yaml
Latency:
  WebSocket RTT: <50ms (P95)
  Operation Broadcast: <100ms (P95)
  Presence Update: <50ms (P95)
  Full Sync: <1s (P95)

Throughput:
  Operations/sec: 10,000
  Messages/sec: 50,000
  Concurrent Users: 1,000

Resource Usage (per node):
  CPU: <60% (avg), <80% (peak)
  Memory: <4GB (avg), <6GB (peak)
  Network: <100Mbps (avg), <500Mbps (peak)
```

## 6. 監視とオブザーバビリティ

### 6.1 メトリクス

**WebSocket メトリクス**:
```go
// Prometheus metrics
var (
    wsConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "websocket_connections_total",
            Help: "Total number of active WebSocket connections",
        },
    )

    wsMessagesSent = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "websocket_messages_sent_total",
            Help: "Total number of messages sent",
        },
    )

    wsLatency = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "websocket_message_latency_seconds",
            Help:    "Latency of WebSocket message delivery",
            Buckets: prometheus.DefBuckets,
        },
    )
)
```

**リアルタイムダッシュボード** (Grafana):
- 現在の接続数 (リアルタイム)
- メッセージ送受信レート
- レイテンシ分布 (P50, P95, P99)
- エラー率
- Redis メモリ使用率

### 6.2 アラート

```yaml
Critical Alerts:
  - name: WebSocketConnectionsHigh
    condition: connections > 8000
    duration: 5m
    action: Auto-scale + notify

  - name: MessageLatencyHigh
    condition: p95_latency > 1s
    duration: 5m
    action: Page on-call

  - name: RedisMemoryHigh
    condition: memory_usage > 90%
    duration: 2m
    action: Page + Increase capacity

Warning Alerts:
  - name: ConnectionDropRate
    condition: disconnections > 100/min
    duration: 5m
    action: Slack notification

  - name: OTConflictRate
    condition: conflicts > 50/min
    duration: 10m
    action: Slack notification
```

## 7. セキュリティ考慮事項

### 7.1 認証・認可

**WebSocket 認証**:
```go
func (ws *WebSocketServer) authenticate(msg AuthMessage) error {
    // Validate JWT token
    claims, err := jwt.Verify(msg.Token)
    if err != nil {
        return errors.New("invalid token")
    }

    // Check token expiration
    if claims.ExpiresAt < time.Now().Unix() {
        return errors.New("token expired")
    }

    // Verify user permissions
    if !hasPermission(claims.UserID, msg.ResourceID) {
        return errors.New("unauthorized")
    }

    return nil
}
```

**リソースレベル認可**:
- イベントごとに編集権限をチェック
- Read-only ユーザーは表示のみ、編集不可
- Admin のみが削除可能

### 7.2 レート制限

```go
// Rate limiter per user
type RateLimiter struct {
    limiters map[string]*rate.Limiter
}

func (rl *RateLimiter) Allow(userID string) bool {
    limiter, exists := rl.limiters[userID]
    if !exists {
        // 100 operations per minute per user
        limiter = rate.NewLimiter(rate.Every(time.Minute/100), 10)
        rl.limiters[userID] = limiter
    }

    return limiter.Allow()
}
```

### 7.3 入力バリデーション

```go
func validateOperation(op Operation) error {
    // Validate op type
    validOps := []string{"add", "remove", "replace", "move", "copy", "test"}
    if !contains(validOps, op.Op) {
        return errors.New("invalid operation type")
    }

    // Validate path (prevent path traversal)
    if !isValidJSONPointer(op.Path) {
        return errors.New("invalid JSON pointer")
    }

    // Sanitize value (XSS prevention)
    if op.Value != nil {
        op.Value = sanitize(op.Value)
    }

    // Validate against schema
    if err := validateSchema(op); err != nil {
        return err
    }

    return nil
}
```

## 8. テスト戦略

### 8.1 単体テスト

**OT アルゴリズム**:
```go
func TestOperationalTransformation(t *testing.T) {
    tests := []struct {
        name string
        opA  Operation
        opB  Operation
        want []Operation
    }{
        {
            name: "concurrent title edits",
            opA:  Operation{Op: "replace", Path: "/title", Value: "Meeting A"},
            opB:  Operation{Op: "replace", Path: "/title", Value: "Meeting B"},
            want: []Operation{
                {Op: "replace", Path: "/title", Value: "Meeting B"},  // B wins
                {Op: "noop"},
            },
        },
        {
            name: "add and remove attendee",
            opA:  Operation{Op: "add", Path: "/attendees/-", Value: user1},
            opB:  Operation{Op: "remove", Path: "/attendees/0"},
            want: []Operation{
                {Op: "add", Path: "/attendees/-", Value: user1},
                {Op: "remove", Path: "/attendees/0"},
            },
        },
        // ... more test cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Transform(tt.opA, tt.opB)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

### 8.2 統合テスト

**WebSocket エンドツーエンド**:
```typescript
describe('Realtime Collaboration', () => {
  it('should sync changes between multiple clients', async () => {
    // Setup: 2 clients connected to same event
    const client1 = new WebSocketClient(token1);
    const client2 = new WebSocketClient(token2);

    await client1.connect();
    await client2.connect();

    await client1.subscribe('event', eventId);
    await client2.subscribe('event', eventId);

    // Client 1 edits title
    await client1.sendOperation({
      op: 'replace',
      path: '/title',
      value: 'New Title',
    });

    // Client 2 should receive update
    const update = await client2.waitForMessage('OP_BROADCAST', 5000);

    expect(update.payload.operation.path).toBe('/title');
    expect(update.payload.operation.value).toBe('New Title');
  });
});
```

### 8.3 負荷テスト

```bash
# k6 load test script
import ws from 'k6/ws';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 1000 },   // Ramp up to 1000 connections
    { duration: '5m', target: 1000 },   // Stay at 1000
    { duration: '2m', target: 5000 },   // Ramp up to 5000
    { duration: '5m', target: 5000 },   // Stay at 5000
    { duration: '2m', target: 0 },      // Ramp down
  ],
};

export default function () {
  const url = 'wss://api.calendar.company.com/realtime';
  const params = { headers: { Authorization: `Bearer ${__ENV.TOKEN}` } };

  ws.connect(url, params, function (socket) {
    socket.on('open', () => {
      socket.send(JSON.stringify({ type: 'SUBSCRIBE', payload: { ... } }));

      // Send operation every 10 seconds
      setInterval(() => {
        socket.send(JSON.stringify({
          type: 'OP',
          payload: generateRandomOperation(),
        }));
      }, 10000);
    });

    socket.on('message', (data) => {
      check(data, { 'message received': (d) => d.length > 0 });
    });

    socket.setTimeout(() => {
      socket.close();
    }, 60000);  // Close after 1 minute
  });
}
```

## 9. ロールアウト計画

### 9.1 段階的展開

```
Week 1-2:  Internal Dogfooding (Engineering team, 50 users)
Week 3-4:  Alpha Testing (Select power users, 200 users)
Week 5:    Beta Launch (10% of users, ~500 users)
Week 6:    Gradual Ramp (25% → 2,500 users)
Week 7-8:  Gradual Ramp (50% → 75%)
Week 9:    Majority (90%)
Week 10:   Full Launch (100%)

Rollback triggers:
- Error rate > 2%
- Latency P95 > 2s
- User complaints > 50/day
- WebSocket connection success rate < 95%
```

### 9.2 フィーチャーフラグ

```typescript
// Feature flag configuration
const realtimeFeatureFlags = {
  // Global enable/disable
  enabled: true,

  // Gradual rollout percentage
  rolloutPercentage: 50,

  // User segments
  enabledForSegments: ['beta_testers', 'premium_users'],

  // A/B test variant
  abTestVariant: 'realtime_v2',

  // Kill switch (emergency disable)
  killSwitch: false,
};

// Usage
if (featureFlag.isEnabled('realtime_collaboration', user.id)) {
  // Show realtime UI
} else {
  // Fallback to polling
}
```

## 10. 次のステップ

### 10.1 承認プロセス

1. **Technical Review**: 2025-12-08
2. **Security Review**: 2025-12-12
3. **Budget Approval**: 2025-12-15
4. **Final Go/No-Go**: 2025-12-18

### 10.2 開始準備

- [ ] AWS インフラのプロビジョニング (Redis, ALB)
- [ ] GitHub リポジトリのセットアップ
- [ ] 開発環境の構築
- [ ] Jiraエピック・ストーリーの作成
- [ ] キックオフミーティング

---

**質問・フィードバック**:
Slack: #realtime-collaboration
Email: platform-team@company.com

**関連ドキュメント**:
- [Calendar System Architecture](./architecture.md)
- [WebSocket Infrastructure Guide](../infrastructure/websocket.md)
- [Security Best Practices](../security/guidelines.md)

**更新履歴**:
- 2025-12-05: 初版作成 (v1.0)
