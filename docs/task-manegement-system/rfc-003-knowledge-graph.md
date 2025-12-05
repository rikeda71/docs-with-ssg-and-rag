# RFC-003: ナレッジグラフと推薦システムの実装

**ステータス**: Proposed
**作成日**: 2025-12-05
**担当者**: Data Engineering Team
**想定工数**: 14週間
**優先度**: P1 (High)

## エグゼクティブサマリー

タスク、プロジェクト、ユーザー、ドキュメント間の関係性をグラフデータベースで管理し、インテリジェントな推薦とインサイトを提供します。属人化の解消、ナレッジ共有の促進、業務効率の向上を実現します。

**期待効果**:
- 関連情報の発見時間を **80% 削減**
- ナレッジの再利用率を **3倍** に向上
- 引き継ぎ時間を **2週間 → 3日** に短縮

## 1. 背景と動機

### 1.1 現状の課題

**情報の孤立**:
- タスクの背景情報が散在 (Wiki, Slack, メール)
- 類似タスクの発見が困難
- 過去のノウハウが活用されない

**属人化**:
- 特定のタスクに詳しい人が不明
- 担当者変更時の知識ロス
- ドメイン知識の可視化不足

### 1.2 ソリューション

グラフデータベース (Neo4j) を活用したナレッジグラフ:

**エンティティ**:
- Task
- Project
- User
- Document
- Tag
- Skill

**関係性**:
- ASSIGNED_TO (User ← Task)
- BELONGS_TO (Task → Project)
- DEPENDS_ON (Task → Task)
- RELATED_TO (Task ↔ Task)
- HAS_SKILL (User → Skill)
- REFERENCES (Task → Document)
- TAGGED_WITH (Task → Tag)

## 2. アーキテクチャ

### 2.1 グラフモデル

```cypher
// ノード作成
CREATE (t:Task {
  id: 'task-123',
  title: 'Implement authentication',
  status: 'done'
})

CREATE (u:User {
  id: 'user-456',
  name: 'Alice'
})

CREATE (s:Skill {
  name: 'Authentication',
  category: 'Security'
})

// 関係作成
CREATE (u)-[:ASSIGNED_TO]->(t)
CREATE (u)-[:HAS_SKILL {level: 'expert'}]->(s)
CREATE (t)-[:REQUIRES_SKILL]->(s)
```

### 2.2 推薦アルゴリズム

**タスク推薦**:
```cypher
// 類似タスクの検索
MATCH (t:Task {id: $taskId})-[:TAGGED_WITH]->(tag:Tag)
MATCH (similar:Task)-[:TAGGED_WITH]->(tag)
WHERE similar.id <> $taskId
  AND similar.status = 'done'
RETURN similar, count(tag) as commonTags
ORDER BY commonTags DESC
LIMIT 10
```

**エキスパート発見**:
```cypher
// タスクに適したエキスパートを見つける
MATCH (t:Task {id: $taskId})-[:REQUIRES_SKILL]->(skill:Skill)
MATCH (u:User)-[has:HAS_SKILL]->(skill)
WHERE has.level IN ['expert', 'advanced']
RETURN u, collect(skill.name) as skills
ORDER BY size(skills) DESC
```

**ナレッジ推薦**:
```cypher
// 関連ドキュメントの推薦
MATCH (t:Task {id: $taskId})
MATCH (similar:Task)-[:RELATED_TO]-(t)
MATCH (similar)-[:REFERENCES]->(doc:Document)
RETURN doc, count(*) as relevance
ORDER BY relevance DESC
```

## 3. 推薦機能

### 3.1 タスク作成時の推薦

```tsx
const TaskRecommendations = ({ taskTitle, tags }) => {
  const { data } = useQuery(RECOMMEND_QUERY, {
    variables: { title: taskTitle, tags }
  });
  
  return (
    <div className="recommendations">
      <h3>Similar tasks you might want to reference:</h3>
      {data.similarTasks.map(task => (
        <TaskCard key={task.id} task={task} />
      ))}
      
      <h3>Recommended experts:</h3>
      {data.experts.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
      
      <h3>Related documents:</h3>
      {data.documents.map(doc => (
        <DocumentLink key={doc.id} doc={doc} />
      ))}
    </div>
  );
};
```

### 3.2 インサイトダッシュボード

```graphql
query TaskInsights($taskId: ID!) {
  task(id: $taskId) {
    insights {
      # 類似タスクの平均完了時間
      estimatedDuration
      
      # よく一緒に完了されるタスク
      relatedTasks {
        task
        correlation
      }
      
      # このタスクに適したチームメンバー
      recommendedAssignees {
        user
        matchScore
        reason
      }
      
      # 参考になるドキュメント
      relevantDocuments {
        document
        relevanceScore
      }
    }
  }
}
```

## 4. グラフ構築パイプライン

### 4.1 データ同期

```python
class GraphSyncPipeline:
    def sync_task(self, task):
        # タスクノードの作成/更新
        self.neo4j.run("""
            MERGE (t:Task {id: $id})
            SET t.title = $title,
                t.status = $status,
                t.updated_at = timestamp()
        """, id=task.id, title=task.title, status=task.status)
        
        # 関係の構築
        if task.assignee_id:
            self.create_assignment_relation(task.id, task.assignee_id)
        
        if task.tags:
            self.create_tag_relations(task.id, task.tags)
        
        # 類似度計算 (非同期)
        self.queue_similarity_calculation(task.id)
    
    def calculate_similarity(self, task_id):
        # TF-IDF, BERT embedding などで類似度計算
        similar_tasks = self.find_similar_tasks(task_id)
        
        for similar_task, score in similar_tasks:
            if score > 0.7:
                self.neo4j.run("""
                    MATCH (t1:Task {id: $id1})
                    MATCH (t2:Task {id: $id2})
                    MERGE (t1)-[r:RELATED_TO]-(t2)
                    SET r.similarity = $score
                """, id1=task_id, id2=similar_task, score=score)
```

## 5. パフォーマンス最適化

**インデックス**:
```cypher
CREATE INDEX task_id FOR (t:Task) ON (t.id)
CREATE INDEX user_id FOR (u:User) ON (u.id)
CREATE INDEX tag_name FOR (tag:Tag) ON (tag.name)
CREATE FULLTEXT INDEX task_search FOR (t:Task) ON EACH [t.title, t.description]
```

**キャッシュ戦略**:
- 推薦結果を Redis にキャッシュ (TTL: 1時間)
- グラフクエリ結果のメモ化
- プリコンピュートされたランキング

## 6. 成功指標

| メトリクス | 目標 |
|-----------|------|
| 関連情報発見時間 | 12分 → 2分 (83%削減) |
| 推薦受諾率 | > 60% |
| ナレッジ再利用率 | 3倍向上 |
| グラフクエリレイテンシ | < 100ms (P95) |
| ナレッジカバレッジ | > 80% |

## 7. 実装計画

### Phase 1 (Week 1-6): グラフ基盤構築
- Neo4j セットアップ
- データ同期パイプライン
- 基本的なクエリAPI

### Phase 2 (Week 7-10): 推薦エンジン
- 類似度計算アルゴリズム
- 推薦API実装
- A/Bテスト基盤

### Phase 3 (Week 11-14): UI/UX
- 推薦ウィジェット
- インサイトダッシュボード
- フィードバックループ

---

**関連ドキュメント**:
- [Architecture](./architecture.md)
- [Data Pipeline](../data/pipeline.md)
- [Neo4j Best Practices](../database/neo4j.md)
