# RFC-002: カスタムワークフローエンジンの実装

**ステータス**: Proposed
**作成日**: 2025-12-05
**担当者**: Platform Team
**想定工数**: 12週間
**優先度**: P1 (High)

## エグゼクティブサマリー

No-code/Low-code でカスタムワークフローを構築できるワークフローエンジンを実装します。各チームが独自の業務プロセスに合わせたタスク管理フローを設計できるようにします。

**期待効果**:
- 各チームの生産性を **40% 向上**
- ワークフローカスタマイズ時間を **90% 削減** (コード不要)
- 業務プロセスの標準化と最適化

## 1. 背景と動機

### 1.1 現状の課題

**硬直的なワークフロー**:
- 固定されたステータス遷移 (TODO → In Progress → Done)
- チームごとの業務プロセスに対応できない
- カスタマイズにエンジニアリングリソースが必要

**具体例**:
- 開発チーム: Code Review → QA → Staging → Production
- デザインチーム: Draft → Review → Revision → Approved
- 営業チーム: Lead → Qualified → Proposal → Closed

### 1.2 ソリューション

視覚的にワークフローを設計できるビジュアルエディタと、柔軟なワークフローエンジンを提供:

**機能**:
- ドラッグ&ドロップでステータスと遷移を定義
- 条件分岐 (if/else, switch)
- 自動化アクション (通知、フィールド更新、外部API呼び出し)
- 承認フロー
- SLA設定とエスカレーション

## 2. アーキテクチャ

### 2.1 ワークフロー定義 (YAML/JSON)

```yaml
name: "Development Workflow"
version: "1.0"

statuses:
  - id: backlog
    name: "Backlog"
    type: initial
  
  - id: in_progress
    name: "In Progress"
    
  - id: code_review
    name: "Code Review"
    conditions:
      - field: has_pull_request
        operator: equals
        value: true
        
  - id: qa
    name: "QA Testing"
    
  - id: done
    name: "Done"
    type: final

transitions:
  - from: backlog
    to: in_progress
    triggers:
      - type: manual
        allowed_roles: [developer, lead]
        
  - from: in_progress
    to: code_review
    triggers:
      - type: automatic
        conditions:
          - field: pull_request_url
            operator: not_empty
    actions:
      - type: notify
        recipients: [code_reviewers]
        template: "Code review needed"
        
  - from: code_review
    to: qa
    triggers:
      - type: manual
        requires_approval: true
        approvers: 2
        
  - from: qa
    to: done
    triggers:
      - type: manual
    actions:
      - type: webhook
        url: "https://deploy.example.com/trigger"

sla:
  - status: code_review
    max_duration: 2days
    escalate_to: [tech_lead]
```

### 2.2 ワークフローエンジン

```go
type WorkflowEngine struct {
    definition *WorkflowDefinition
    stateStore StateStore
}

func (e *WorkflowEngine) Transition(taskID string, toStatus string) error {
    // 現在のステータスを取得
    currentStatus := e.stateStore.GetStatus(taskID)
    
    // 遷移が許可されているか確認
    transition := e.definition.FindTransition(currentStatus, toStatus)
    if transition == nil {
        return ErrInvalidTransition
    }
    
    // 条件チェック
    if !e.checkConditions(taskID, transition.Conditions) {
        return ErrConditionsNotMet
    }
    
    // 承認が必要な場合
    if transition.RequiresApproval {
        return e.requestApproval(taskID, transition)
    }
    
    // ステータス更新
    e.stateStore.UpdateStatus(taskID, toStatus)
    
    // アクション実行
    go e.executeActions(taskID, transition.Actions)
    
    return nil
}
```

## 3. ビジュアルエディタ

React Flow を使用したドラッグ&ドロップエディタ:

```tsx
const WorkflowEditor = () => {
  const [nodes, setNodes] = useState([]);
  const [edges, setEdges] = useState([]);
  
  return (
    <ReactFlow
      nodes={nodes}
      edges={edges}
      onNodesChange={onNodesChange}
      onEdgesChange={onEdgesChange}
      onConnect={onConnect}
    >
      <Background />
      <Controls />
      <MiniMap />
    </ReactFlow>
  );
};
```

## 4. 成功指標

| メトリクス | 目標 |
|-----------|------|
| ワークフロー作成時間 | < 30分 |
| カスタムワークフロー採用率 | > 70% |
| ワークフロー実行成功率 | > 99.5% |
| ビジネスプロセス改善 | 40% 効率化 |

---

**関連ドキュメント**:
- [Architecture](./architecture.md)
- [Workflow DSL Specification](./workflow-dsl.md)
