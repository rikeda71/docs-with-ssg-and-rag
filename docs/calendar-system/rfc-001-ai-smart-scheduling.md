# RFC-001: AI スマートスケジューリング機能の実装

**ステータス**: Proposed
**作成日**: 2025-12-05
**担当者**: ML Platform Team
**レビュアー**: Engineering Leadership, Product Management
**想定工数**: 16 週間 (4エンジニア)
**優先度**: P1 (High)

## エグゼクティブサマリー

本提案は、機械学習を活用した AI スマートスケジューリング機能を Calendar System に実装するものです。ユーザーの過去の行動パターン、参加者の空き時間、会議室の空き状況、移動時間などを考慮し、最適なミーティング時間を自動提案します。

**期待される効果**:
- ミーティング設定時間を平均 **8.5分 → 30秒** に短縮（94% 削減）
- スケジューリング関連の往復メール数を **70% 削減**
- 会議室の稼働率を **42% → 65%** に向上
- ユーザー満足度を **4.5/5.0** に改善

**投資対効果**:
- 年間時間削減価値: **¥180M**（全社5,000名規模）
- 開発コスト: **¥48M**（初年度）
- ROI: **275%**（初年度）

## 1. 背景と動機

### 1.1 現状の課題

現在のカレンダーシステムでは、ミーティングのスケジューリングに以下の問題があります：

#### 時間のかかる調整プロセス
- **手動調整**: 参加者全員の空き時間を目視で確認し、3〜5往復のメールで調整
- **平均所要時間**: 8.5分/ミーティング
- **年間コスト**: 全社で約520,000時間（¥156M相当）の損失

#### 非効率な時間枠選択
- **ピーク時間帯の集中**: 10:00-11:00, 14:00-15:00 に予約が集中（全体の40%）
- **移動時間の未考慮**: 連続するミーティング間の移動時間が不足し、15%が遅刻
- **タイムゾーンの複雑性**: グローバルチームとの調整に平均45分

#### 会議室リソースの最適化不足
- **無駄な予約**: 実際の参加者数に対して過大な会議室を予約（30%のケース）
- **空き室情報の不足**: 適切な会議室を見つけるのに平均5分
- **設備ミスマッチ**: 必要な設備（プロジェクター、ビデオ会議）がない会議室を予約（12%）

### 1.2 ユーザーからのフィードバック

社内アンケート（n=1,200）の結果：
- **87%**: "ミーティングのスケジューリングに時間がかかりすぎる"
- **72%**: "最適な時間枠を見つけるのが困難"
- **68%**: "会議室の予約が煩雑"
- **要望**: "AIが自動的に最適な時間を提案してほしい"（92%）

### 1.3 競合分析

| 製品 | AI スケジューリング | 主要機能 | 制約 |
|------|-------------------|---------|------|
| Google Calendar | ✓ (限定的) | Time Insights | 単純なパターン認識のみ |
| Microsoft FindTime | ✓ | 投票ベース | 複数候補から選択が必要 |
| x.ai (Amy) | ✓✓ | 自然言語処理 | 高コスト、外部サービス |
| Calendly | ✗ | 自動予約 | 一対一のみ、複雑な制約に非対応 |
| **提案システム** | ✓✓✓ | ML ベース、多要素最適化 | - |

## 2. 技術的アプローチ

### 2.1 システムアーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    User Interface Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │Smart Schedule│  │ Suggestion   │  │  Feedback       │  │
│  │   Widget     │  │  Panel       │  │  Collection     │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    API Layer (GraphQL)                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  suggestMeetingTimes(input: MeetingRequest)          │  │
│  │  rankSuggestions(candidates: [TimeSlot])             │  │
│  │  provideFeedback(suggestionId: ID, rating: Int)      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              Smart Scheduling Service (Go)                  │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │ Constraint  │ │  Candidate   │ │   ML Ranking     │    │
│  │  Solver     │ │  Generator   │ │   Service        │    │
│  └─────────────┘ └──────────────┘ └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  ML Inference Service                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  TensorFlow Serving / TorchServe                     │  │
│  │  Models: Time Preference, Conflict Prediction,       │  │
│  │          Resource Optimization                        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              Feature Store & Training Pipeline              │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │  Feast      │ │  Airflow     │ │   Model          │    │
│  │(Feature Eng)│ │(Orchestration)│ │  Registry        │    │
│  └─────────────┘ └──────────────┘ └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                             │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │ PostgreSQL  │ │  BigQuery    │ │    S3            │    │
│  │(Operational)│ │(Analytics DW)│ │(Model Artifacts) │    │
│  └─────────────┘ └──────────────┘ └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 機械学習モデル設計

#### モデル1: 時間帯優先度予測 (Time Preference Model)

**目的**: ユーザーごとの好ましいミーティング時間帯を予測

**アーキテクチャ**: LightGBM Gradient Boosting

**特徴量** (45次元):
```python
User Features (15):
- user_id (categorical, embedding)
- user_role (categorical)
- user_department (categorical)
- work_start_hour (numerical)
- work_end_hour (numerical)
- timezone (categorical)
- typical_meeting_count_per_day (numerical)
- avg_meeting_duration (numerical)
- preference_morning_meetings (boolean)
- preference_afternoon_meetings (boolean)
- preference_back_to_back (boolean)
- historical_accept_rate (numerical, 0-1)
- historical_reschedule_rate (numerical, 0-1)
- avg_response_time_hours (numerical)
- seniority_level (ordinal, 1-5)

Temporal Features (12):
- day_of_week (categorical, one-hot encoded)
- hour_of_day (numerical, 0-23)
- is_monday (boolean)
- is_friday (boolean)
- week_of_month (numerical, 1-5)
- is_quarter_end (boolean)
- days_until_weekend (numerical)
- hour_since_work_start (numerical)
- hour_until_work_end (numerical)
- is_lunch_time (boolean, 11:30-13:30)
- is_peak_hour (boolean, 10-11 or 14-15)
- season (categorical, Q1-Q4)

Context Features (10):
- meeting_duration (numerical, minutes)
- attendee_count (numerical)
- is_recurring (boolean)
- meeting_type (categorical: 1on1, team, all-hands, external)
- is_cross_timezone (boolean)
- requires_room (boolean)
- requires_video_conf (boolean)
- organizer_is_executive (boolean)
- avg_attendee_seniority (numerical)
- estimated_preparation_time (numerical)

Historical Patterns (8):
- same_attendees_meeting_count (numerical)
- avg_previous_meeting_hour (numerical)
- previous_meeting_acceptance_rate (numerical)
- consecutive_meeting_count_today (numerical)
- total_meeting_hours_this_week (numerical)
- days_since_last_similar_meeting (numerical)
- typical_meeting_interval (numerical)
- meeting_density_score (numerical, 0-1)
```

**ラベル**: `meeting_score` (0-1, ユーザーの満足度を反映)
- 受諾 → 1.0
- 受諾+リスケなし+出席 → 1.0
- 受諾+リスケあり → 0.6
- 辞退 → 0.0
- フィードバック評価を反映 (ユーザーが提供した場合)

**学習データ**:
- 過去2年間のミーティングデータ: 500万件
- ユーザーフィードバック: 15万件
- 更新頻度: 週次

**モデル評価**:
- **Offline Metrics**:
  - AUC-ROC: 0.87
  - Precision@5: 0.82 (上位5件の提案の正確性)
  - NDCG@10: 0.79 (ランキング品質)
- **Online Metrics** (A/B test):
  - Acceptance Rate: 78% (baseline: 45%)
  - User Satisfaction: 4.6/5.0

#### モデル2: コンフリクト予測 (Conflict Prediction Model)

**目的**: スケジュールコンフリクトの発生確率を予測

**アーキテクチャ**: Deep Neural Network (TensorFlow)

```python
# Model Architecture
model = Sequential([
    Input(shape=(32,)),
    Dense(128, activation='relu'),
    BatchNormalization(),
    Dropout(0.3),
    Dense(64, activation='relu'),
    BatchNormalization(),
    Dropout(0.2),
    Dense(32, activation='relu'),
    Dense(1, activation='sigmoid')
])

# Loss: Binary Cross-Entropy
# Optimizer: Adam (lr=0.001)
# Metrics: Precision, Recall, F1-score
```

**特徴量**:
- 前後のイベント間隔
- 移動時間の必要性
- 参加者の同時刻の他の予定
- 会議室の二重予約リスク
- 過去のドタキャン率

**目標精度**:
- Precision: 0.85 (誤検知を低く)
- Recall: 0.90 (見逃しを最小化)

#### モデル3: リソース最適化 (Resource Optimization Model)

**目的**: 会議室・設備の最適な割り当て

**アプローチ**: 強化学習 (Deep Q-Network)

**状態空間**:
- 現在の会議室予約状況
- 参加者の現在位置
- 必要な設備リスト
- 時間帯

**行動空間**:
- 会議室選択 (100+ rooms)
- 時間帯調整 (±30分)

**報酬関数**:
```python
def calculate_reward(action):
    reward = 0

    # 会議室サイズの適合性
    if room_capacity * 0.5 <= attendee_count <= room_capacity * 0.9:
        reward += 10
    else:
        reward -= 5

    # 移動時間の最小化
    avg_travel_time = calculate_avg_travel_time(attendees, room)
    reward -= avg_travel_time * 0.5

    # 設備の適合性
    if required_equipment.issubset(room_equipment):
        reward += 15
    else:
        reward -= 20

    # リソース利用効率
    room_utilization = get_room_utilization(room_id)
    reward += (room_utilization - 0.5) * 10  # Target: 50-70%

    # ユーザーフィードバック
    if user_accepted:
        reward += 20

    return reward
```

### 2.3 制約ソルバー (Constraint Solver)

**アルゴリズム**: Integer Linear Programming (ILP) + Heuristics

**制約条件**:

```python
Hard Constraints (必須):
- HC1: 参加者全員が空いている時間
- HC2: 会議室が空いている時間
- HC3: 業務時間内 (各ユーザーのタイムゾーン考慮)
- HC4: 最小会議間隔 (前後5分以上)
- HC5: 移動時間の確保 (物理的な会議室の場合)

Soft Constraints (最適化目標):
- SC1: ユーザーの好みの時間帯 (weight: 0.3)
- SC2: 会議の連続を避ける (weight: 0.2)
- SC3: ランチタイムを避ける (weight: 0.15)
- SC4: 金曜日午後を避ける (weight: 0.1)
- SC5: 会議室サイズの適合性 (weight: 0.15)
- SC6: 移動時間の最小化 (weight: 0.1)
```

**最適化問題の定式化**:

```
Maximize:
  Σ(w_i * score_i)  for all soft constraints

Subject to:
  All hard constraints are satisfied

Where:
  w_i = weight of soft constraint i
  score_i = satisfaction score of constraint i (0-1)
```

**ソルバー**: Google OR-Tools (CP-SAT Solver)

**パフォーマンス**:
- 候補生成時間: < 2秒 (95th percentile)
- 最適解の品質: 90% of optimal (証明可能)

### 2.4 API設計

**GraphQL Mutation**:

```graphql
mutation SuggestMeetingTimes($input: MeetingRequestInput!) {
  suggestMeetingTimes(input: $input) {
    suggestions {
      id
      timeSlot {
        start
        end
        timezone
      }
      room {
        id
        name
        location
        capacity
        equipment
      }
      score
      confidence
      reasoning {
        primaryFactors
        warnings
        alternatives
      }
      attendeeAvailability {
        userId
        isAvailable
        conflicts {
          eventId
          title
          canMove
        }
      }
    }
    metadata {
      totalCandidatesEvaluated
      processingTimeMs
      modelVersion
    }
  }
}

# Input Type
input MeetingRequestInput {
  title: String!
  duration: Int!  # minutes
  attendees: [AttendeeInput!]!
  requiredEquipment: [Equipment!]
  preferredTimeRange: TimeRangeInput
  recurrence: RecurrenceInput
  priority: Priority
  location: LocationPreference
}
```

**レスポンス例**:

```json
{
  "data": {
    "suggestMeetingTimes": {
      "suggestions": [
        {
          "id": "sug_1a2b3c",
          "timeSlot": {
            "start": "2025-12-10T14:00:00Z",
            "end": "2025-12-10T15:00:00Z",
            "timezone": "Asia/Tokyo"
          },
          "room": {
            "id": "room_501",
            "name": "Conference Room A",
            "location": "5F, Building 1",
            "capacity": 10,
            "equipment": ["PROJECTOR", "WHITEBOARD", "VIDEO_CONF"]
          },
          "score": 0.92,
          "confidence": 0.88,
          "reasoning": {
            "primaryFactors": [
              "All attendees available",
              "Preferred time slot (afternoon)",
              "Room has required video conferencing"
            ],
            "warnings": [
              "Sarah has a meeting 30 min before (travel time tight)"
            ],
            "alternatives": [
              "Wednesday 10:00 AM also highly scored (0.89)"
            ]
          },
          "attendeeAvailability": [
            {
              "userId": "user_123",
              "isAvailable": true,
              "conflicts": []
            },
            {
              "userId": "user_456",
              "isAvailable": true,
              "conflicts": [
                {
                  "eventId": "evt_789",
                  "title": "Team Sync",
                  "canMove": true
                }
              ]
            }
          ]
        },
        {
          "id": "sug_2b3c4d",
          "timeSlot": {
            "start": "2025-12-11T10:00:00Z",
            "end": "2025-12-11T11:00:00Z",
            "timezone": "Asia/Tokyo"
          },
          "score": 0.89,
          "confidence": 0.91,
          "reasoning": {
            "primaryFactors": [
              "All attendees available with no conflicts",
              "Larger room available for comfort"
            ],
            "warnings": [],
            "alternatives": []
          }
        }
      ],
      "metadata": {
        "totalCandidatesEvaluated": 247,
        "processingTimeMs": 1850,
        "modelVersion": "v2.3.1"
      }
    }
  }
}
```

### 2.5 フィードバックループ

**明示的フィードバック**:
```graphql
mutation ProvideFeedback($input: FeedbackInput!) {
  provideFeedback(input: $input) {
    success
    message
  }
}

input FeedbackInput {
  suggestionId: ID!
  wasAccepted: Boolean!
  rating: Int  # 1-5
  rejectionReason: RejectionReason
  comments: String
}

enum RejectionReason {
  TIME_NOT_SUITABLE
  ROOM_NOT_SUITABLE
  ATTENDEE_CONFLICT
  PREFER_DIFFERENT_DAY
  OTHER
}
```

**暗黙的フィードバック** (自動収集):
- 提案の受諾/拒否
- 提案後の手動調整
- ミーティングの実際の出席率
- リスケジュールの頻度
- ミーティング後のキャンセル

**フィードバックの活用**:
1. **リアルタイム学習**: オンライン学習でモデルを継続的に改善
2. **A/B テスト**: 新しいモデルバージョンを段階的にロールアウト
3. **パーソナライゼーション**: ユーザー固有のパターンを学習

## 3. 実装計画

### 3.1 フェーズ分け

#### Phase 1: MVP (Week 1-6)

**目標**: 基本的なスマートスケジューリング機能

**スコープ**:
- ✓ 単純なルールベースの提案エンジン
- ✓ 参加者の空き時間チェック
- ✓ 会議室の空き状況確認
- ✓ 上位3件の時間枠提案
- ✓ 基本的なUI/UX

**成功基準**:
- レスポンス時間 < 3秒
- 提案精度 60% 以上
- 100名のベータテスター

**デリバラブル**:
- Smart Scheduling Service (Go)
- GraphQL API
- React コンポーネント
- 基本的なメトリクス収集

#### Phase 2: ML Integration (Week 7-12)

**目標**: 機械学習モデルの統合

**スコープ**:
- ✓ 時間帯優先度予測モデルの実装
- ✓ Feature Store の構築 (Feast)
- ✓ モデルトレーニングパイプライン (Airflow)
- ✓ TensorFlow Serving / TorchServe のデプロイ
- ✓ A/B テスト基盤

**成功基準**:
- モデル AUC-ROC > 0.85
- 推論レイテンシ < 100ms
- 提案精度 75% 以上

**デリバラブル**:
- ML Inference Service
- Training Pipeline
- Model Registry (MLflow)
- A/B テストフレームワーク

#### Phase 3: Advanced Features (Week 13-16)

**目標**: 高度な最適化と統合

**スコープ**:
- ✓ コンフリクト予測モデル
- ✓ リソース最適化 (強化学習)
- ✓ クロスタイムゾーン最適化
- ✓ リカレンスイベント対応
- ✓ フィードバックループの完全実装

**成功基準**:
- 提案精度 85% 以上
- ユーザー満足度 4.5/5.0
- 全社ロールアウト準備完了

**デリバラブル**:
- Production-ready ML models
- 完全な監視・アラート
- ドキュメント完成
- トレーニング資料

### 3.2 タイムライン

```
Week  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16
Phase 1 [===MVP===]
           └─ Beta Launch (Week 4)
              └─ User Feedback (Week 5-6)

Phase 2            [===ML Integration===]
                      └─ Model Training (Week 8-10)
                         └─ A/B Test (Week 11-12)

Phase 3                              [===Advanced===]
                                        └─ Full Launch (Week 16)

Milestones:
• Week 4:  MVP Beta Launch
• Week 6:  Phase 1 Complete, Go/No-Go Decision
• Week 12: ML Models in Production (10% traffic)
• Week 14: Ramp to 50% traffic
• Week 16: Full Launch (100% traffic)
```

### 3.3 リソース計画

**エンジニアリングチーム**:
- Backend Engineer (Go): 2名 (16週間)
- ML Engineer: 1名 (12週間、Week 5-16)
- Frontend Engineer (React): 1名 (8週間、Week 1-4, 13-16)
- Data Engineer: 1名 (8週間、Week 7-14)
- QA Engineer: 0.5名 (継続)

**インフラコスト**:
| リソース | 仕様 | 月額コスト |
|---------|------|----------|
| ML Training | p3.2xlarge (GPU) x 2 | $1,800 |
| Model Serving | c5.2xlarge x 3 | $720 |
| Feature Store | r5.xlarge + S3 | $400 |
| BigQuery | 10 TB storage + queries | $600 |
| **合計** | | **$3,520/月** |

**総コスト見積もり**:
- 人件費: ¥40M (エンジニア工数)
- インフラ: ¥1.7M (4ヶ月)
- その他: ¥6.3M (ツール、ライセンス等)
- **合計**: **¥48M**

## 4. リスクと軽減策

### 4.1 技術的リスク

| リスク | 影響度 | 確率 | 軽減策 |
|--------|--------|------|--------|
| ML モデルの精度不足 | 高 | 中 | - Phase 1 はルールベースでフォールバック<br>- 継続的なモデル改善<br>- ユーザーフィードバックによる学習 |
| 推論レイテンシが高い | 中 | 低 | - モデルの量子化・最適化<br>- キャッシング戦略<br>- バッチ推論 |
| データ品質の問題 | 高 | 中 | - データバリデーションパイプライン<br>- 異常値検出<br>- 人間によるレビュー |
| スケーラビリティ | 中 | 低 | - 水平スケーリング設計<br>- ロードテスト (10,000 req/s)<br>- 段階的ロールアウト |
| コールドスタート問題 | 中 | 高 | - 新規ユーザーはルールベース<br>- 類似ユーザーの転移学習<br>- 明示的な設定オプション |

### 4.2 プロダクトリスク

| リスク | 影響度 | 確率 | 軽減策 |
|--------|--------|------|--------|
| ユーザー受容性が低い | 最高 | 中 | - 段階的ロールアウト<br>- A/B テスト<br>- 手動オプションの維持<br>- 丁寧なオンボーディング |
| プライバシー懸念 | 高 | 低 | - データ最小化原則<br>- 透明性の確保 (説明可能なAI)<br>- Opt-out オプション<br>- プライバシーポリシーの明確化 |
| 過度な自動化への依存 | 中 | 中 | - 最終決定権はユーザー<br>- 推奨理由の明示<br>- 簡単な上書き機能 |

### 4.3 運用リスク

| リスク | 影響度 | 確率 | 軽減策 |
|--------|--------|------|--------|
| モデルドリフト | 中 | 高 | - 継続的なモニタリング<br>- 週次モデル再トレーニング<br>- パフォーマンス劣化アラート |
| フィードバックループのバイアス | 中 | 中 | - サンプリング戦略<br>- 反事実的サンプルの追加<br>- 定期的なバイアス監査 |
| インフラ障害 | 高 | 低 | - ルールベースへの自動フォールバック<br>- 多重冗長化<br>- 詳細なRunbook |

## 5. 成功指標 (KPI)

### 5.1 ビジネスメトリクス

| 指標 | ベースライン | 目標 (Phase 3) | 測定方法 |
|------|-------------|---------------|---------|
| ミーティング設定時間 | 8.5分 | 30秒 | クライアント側タイマー |
| 提案受諾率 | N/A (新機能) | 75% | 受諾数/提案数 |
| 往復メール数 | 平均3.2往復 | 0.5往復 | メールログ分析 |
| 会議室稼働率 | 42% | 65% | 予約データ分析 |
| ユーザー満足度 | 3.8/5.0 | 4.5/5.0 | NPS調査 |
| 機能利用率 | 0% | 80% | アクティブユーザー率 |

### 5.2 技術メトリクス

| 指標 | 目標 | 測定方法 |
|------|------|---------|
| API レスポンス時間 (P95) | < 2秒 | Prometheus |
| ML 推論レイテンシ (P95) | < 100ms | Model server metrics |
| モデル AUC-ROC | > 0.85 | Offline evaluation |
| システム可用性 | > 99.9% | Uptime monitoring |
| エラー率 | < 0.1% | Error tracking |

### 5.3 ML モデルメトリクス

| モデル | 指標 | 目標 | 測定頻度 |
|--------|------|------|---------|
| Time Preference | Precision@5 | > 0.80 | 週次 |
| Time Preference | User Satisfaction | > 4.3/5.0 | 月次 |
| Conflict Prediction | Precision | > 0.85 | 週次 |
| Conflict Prediction | Recall | > 0.90 | 週次 |
| Resource Optimization | Room Utilization | > 60% | 週次 |
| All Models | Inference Latency | < 100ms | リアルタイム |

## 6. A/B テスト戦略

### 6.1 実験設計

**Hypothesis**: AI スマートスケジューリングは、ミーティング設定時間を大幅に削減し、ユーザー満足度を向上させる

**Variants**:
- **Control (A)**: 既存の手動スケジューリング
- **Treatment (B)**: AI スマートスケジューリング (MVP)
- **Treatment (C)**: AI + ML モデル統合

**サンプルサイズ**: 各グループ 300ユーザー (α=0.05, β=0.2, MDE=20%)

**実験期間**: 4週間

**ランダム化**: ユーザーIDベースのハッシュ (安定した割り当て)

### 6.2 評価指標

**Primary Metric**: ミーティング設定時間の削減率

**Secondary Metrics**:
- 提案受諾率
- ユーザー満足度 (調査)
- 機能利用率
- 往復メール数

**Guardrail Metrics** (悪化させてはいけない):
- システムエラー率 < 0.5%
- ページロード時間 < 2秒
- ミーティングキャンセル率 (変化なし)

### 6.3 段階的ロールアウト

```
Week 1-2:  Internal dogfooding (50 engineers)
Week 3-4:  Beta users (5% of users, ~250 people)
Week 5-6:  Gradual ramp (10% → 25%)
Week 7-8:  Majority rollout (50% → 75%)
Week 9-10: Full launch (100%)

Decision Points:
- Week 2: Go/No-Go for beta expansion
- Week 4: Evaluate beta metrics, decide on ramp
- Week 6: Mid-point review, adjust if needed
- Week 10: Post-launch review
```

## 7. プライバシーとコンプライアンス

### 7.1 データ利用ポリシー

**収集するデータ**:
- ミーティングのメタデータ (時間、参加者、場所)
- ユーザーの受諾/拒否アクション
- 明示的なフィードバック (評価、コメント)
- 集約された行動パターン

**収集しないデータ**:
- ミーティングの内容・議事録
- 個人的な会話
- 第三者のメタデータ (同意なし)

### 7.2 透明性と制御

**ユーザーへの説明**:
```
「スマートスケジューリングは、あなたの過去のミーティングパターン
（好みの時間帯、会議室選択など）を学習し、最適な時間枠を提案します。

使用するデータ:
• 過去のミーティング履歴
• カレンダーの空き状況
• 会議室・設備の利用パターン
• フィードバック

提案の理由を常に表示し、いつでも手動で調整できます。」
```

**ユーザーコントロール**:
- Opt-out オプション (いつでも無効化可能)
- データ削除リクエスト (GDPR Right to Erasure)
- 学習データのエクスポート機能
- プライバシー設定のカスタマイズ

### 7.3 説明可能性 (Explainable AI)

**推奨理由の表示**:
```json
{
  "reasoning": {
    "primaryFactors": [
      "All 5 attendees are available",
      "Matches your preferred afternoon time (14:00-16:00)",
      "Conference Room A has required video conferencing equipment"
    ],
    "score_breakdown": {
      "availability": 1.0,
      "time_preference": 0.92,
      "room_suitability": 0.88,
      "travel_time": 0.85
    },
    "warnings": [
      "John has a meeting 30 minutes before (tight schedule)"
    ]
  }
}
```

**SHAP値による特徴量重要度**:
- ユーザーが要求した場合、モデルの意思決定を詳細に説明
- 「なぜこの時間を提案したのか?」に対する定量的な回答

## 8. モニタリングとアラート

### 8.1 ダッシュボード

**ビジネスダッシュボード** (Grafana):
- 日次アクティブユーザー数
- 提案受諾率 (トレンド)
- 平均ミーティング設定時間
- ユーザー満足度スコア
- 会議室稼働率

**技術ダッシュボード**:
- API レスポンス時間 (P50, P95, P99)
- エラー率 (4xx, 5xx)
- ML 推論レイテンシ
- スループット (req/s)
- リソース使用率 (CPU, Memory)

**ML モデルダッシュボード**:
- 予測精度 (日次)
- モデルドリフト検出
- 特徴量分布の変化
- フィードバックスコアのトレンド
- A/B テスト結果

### 8.2 アラート

**Critical (PagerDuty)**:
```yaml
- Alert: HighErrorRate
  Condition: error_rate > 5%
  Duration: 5 minutes
  Action: Page on-call engineer

- Alert: MLModelServing Down
  Condition: model_server_health == 0
  Duration: 2 minutes
  Action: Auto-fallback to rule-based + Page

- Alert: DataPipelineFailure
  Condition: airflow_dag_failed
  Duration: immediate
  Action: Page ML team
```

**Warning (Slack)**:
```yaml
- Alert: ModelDriftDetected
  Condition: drift_score > 0.3
  Duration: 1 day
  Action: Notify ML team

- Alert: LowAcceptanceRate
  Condition: acceptance_rate < 60%
  Duration: 1 hour
  Action: Notify Product team

- Alert: HighLatency
  Condition: p95_latency > 3 seconds
  Duration: 10 minutes
  Action: Notify Backend team
```

## 9. 依存関係とブロッカー

### 9.1 技術的依存

- ✓ Calendar Service API の安定性 (既存)
- ✓ Event Service の拡張 (Week 2-3 で対応)
- ✓ BigQuery データウェアハウスのセットアップ (Week 1)
- △ Feature Store (Feast) の導入 (Week 7-8、新規)
- △ ML Platform の構築 (Week 7-10、新規)

### 9.2 組織的依存

- **Legal Team**: プライバシーポリシーのレビュー (Week 3)
- **Compliance Team**: GDPR 対応の確認 (Week 4)
- **Data Governance**: データ利用の承認 (Week 2)
- **Product Team**: UI/UX レビュー (Week 4, 12, 15)
- **Executive Sponsor**: 予算承認 (Week 0)

## 10. 次のステップ

### 10.1 承認プロセス

1. **Technical Design Review**: 2025-12-10 (完了)
2. **Security Review**: 2025-12-15
3. **Privacy Review**: 2025-12-18
4. **Budget Approval**: 2025-12-20
5. **Final Go/No-Go**: 2025-12-22

### 10.2 開始前の準備

- [ ] プロジェクトキックオフミーティング
- [ ] Jiraエピック・ストーリーの作成
- [ ] GitHubリポジトリのセットアップ
- [ ] AWSインフラのプロビジョニング
- [ ] ステークホルダー向けキックオフプレゼン

---

**質問・フィードバック**:
Slack: #calendar-ai-scheduling
Email: ml-platform-team@company.com

**関連ドキュメント**:
- [Calendar System Architecture](./architecture.md)
- [ML Platform Architecture](../ml-platform/architecture.md)
- [Privacy Policy](../legal/privacy-policy.md)

**更新履歴**:
- 2025-12-05: 初版作成 (v1.0)
