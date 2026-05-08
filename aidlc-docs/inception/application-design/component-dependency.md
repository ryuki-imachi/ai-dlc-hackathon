# Component Dependency — マカセル

> **AI-DLC Inception Phase / Application Design 成果物**
>
> コンポーネント間の依存関係とコミュニケーションパターンを記述する。
>
> ### 設計変更履歴
> - **2026-05-08 (ADR-1' 改訂版)**: 旧 C-5 Memory / 旧 C-6 Gateway を削除。新 C-5 Calendar Tool Lambda / 新 C-6 Gmail Tool Lambda に置換。AI Agent → Tool Lambdas の依存は **Bedrock Converse API tool_use → Lambda invoke** で実現。
> - **2026-05-08 (ADR-6)**: C-2 API Layer を **AppSync → API Gateway HTTP API + Lambda** に切り替え。Subscription 経路を削除し、フロント↔C-2 は同期 HTTP のみ。

---

## 1. 依存マトリクス

行 = 呼び出し元、列 = 呼び出し先。`✓` は同期呼び出し、`◯` は非同期、`-` は依存なし。

| From \ To | C-1 FE | C-2 API | C-3 Voice | C-4 Agent | C-5 CalTool | C-6 GmailTool | C-7 ID | C-8 Sabori | C-9 Exec | C-10 DDB | C-11 Obs |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **C-1 Frontend** | - | ✓ | ✓ | - | - | - | - | - | - | - | - |
| **C-2 API** | - | - | - | ✓ | - | - | - | ✓ | ✓ | ✓ | ◯ |
| **C-3 Voice** | - | - | - | ✓ | - | - | - | - | - | - | ◯ |
| **C-4 Agent** | - | - | - | - | ✓ | ✓ | - | - | - | - | ◯ |
| **C-5 Calendar Tool** | - | - | - | - | - | - | ✓ | - | - | - | ◯ |
| **C-6 Gmail Tool** | - | - | - | - | - | - | ✓ | - | - | - | ◯ |
| **C-7 Identity** | - | - | - | - | - | - | - | - | - | - | ◯ |
| **C-8 Sabori** | - | - | - | ✓ | - | - | - | - | - | ✓ | ◯ |
| **C-9 Executor** | - | - | - | - | ✓ | ✓ | - | - | - | ✓ | ◯ |
| **C-10 DynamoDB** | - | - | - | - | - | - | - | - | - | - | ◯ |
| **C-11 Observability** | - | - | - | - | - | - | - | - | - | - | - |

外部 SaaS（Google Calendar / Gmail）への依存は **C-5 / C-6 Tool Lambda 経由でのみ発生**。
Tool Lambda 内では C-7 Identity から OAuth トークンを取得して呼び出す。
LLM（Bedrock Claude）への依存は C-4 Agent 経由でのみ発生（Bedrock Converse API tool_use ループ）。

---

## 2. コミュニケーションパターン

### 2.1 同期 (Sync)
- **HTTP API (REST over HTTPS)**: C-1 → C-2（Cognito User Pool Authorizer で JWT 検証）
- **WebSocket**: C-1 → C-3（音声ストリーミング、Cognito JWT）
- **Lambda invoke / SDK**: C-2 → C-4 / C-8 / C-9
- **Bedrock Converse API tool_use → Lambda invoke**: C-4 → C-5 / C-6
- **AgentCore Identity SDK**: C-5 / C-6 → C-7
- **HTTPS（Google API SDK）**: C-5 / C-6 → 外部 Google API
- **DynamoDB SDK**: C-2 / C-8 / C-9 → C-10

### 2.2 非同期 (Async)
- **EventBridge Scheduler → Lambda**: 日次トリガで C-8 Sabori Logic を起動
- **CloudWatch Logs / Metrics**: 全コンポーネントから C-11 Observability へ非同期書き込み

> **MVP では Subscription / Push 系の非同期配信は持たない**（ADR-6）。代行ステータス更新は Mutation の同期応答で受け取るか、必要なら短期ポーリングで補完。

### 2.3 認証伝搬
- フロント（C-1）→ Cognito JWT → API（C-2）
- API → AgentCore Workload Identity → Agent（C-4）
- Agent（C-4）→ tool_use → Tool Lambda（C-5 / C-6）→ Identity（C-7）→ Google OAuth Token → Google API

---

## 3. データフロー図（音声でサボる→承認→実行）

```
[ユーザー発話「明日の朝会、いらない」]
                │
                ▼ WebSocket (音声バイナリ)
        ┌───────────────┐
        │  Voice Stream │
        │   Handler     │ ──→ Transcribe Streaming
        │     (C-3)     │ ←── テキスト「明日の朝会、いらない」
        └───────┬───────┘
                │ AgentCore Runtime invoke (text + userId + sessionId)
                ▼
        ┌───────────────────┐
        │   AI Agent (C-4)  │
        │  Bedrock Converse │
        │      tool_use     │
        └─┬─────────────────┘
          │
          │ ① Converse 呼出（list_calendar_events tool 要求）
          │
          │ ② tool_use → Lambda invoke
          │      ├─ C-5 Calendar Tool ─→ Identity (C-7) ─→ Google API
          │      └─ tool_result を Agent に返す
          │
          │ ③ 再度 Converse（最終応答 = markAsSabori 意図と判定）
          │
          │ ④ markAsSabori → DynamoDB (C-10)
          ▼
        ┌───────────────┐
        │  HTTP API     │ ─→ Mutation 系 API のレスポンスで C-1 に状態返却
        │   (C-2)       │    （Subscription なし、必要ならフロントが GET でポーリング）
        └───────┬───────┘
                │ generateDelegationDraft trigger（同期 or バックグラウンド）
                ▼
        ┌───────────────┐
        │   AI Agent    │ ─→ Bedrock Converse で断り文生成
        │     (C-4)     │ ─→ DynamoDB に DelegationDraft 保存
        └───────┬───────┘
                │ レスポンス or 次回ポーリングで C-1 にモーダル表示
                ▼
[ユーザー発話「うん、それで」]
                │
                ▼（同様のフロー）
        ┌───────────────┐
        │   AI Agent    │ ─→ approveDelegation 認識
        │     (C-4)     │
        └───────┬───────┘
                │ DelegationExecutor invoke
                ▼
        ┌───────────────┐
        │  Delegation   │ ─→ C-5 Calendar Tool invoke (update_attendance)
        │   Executor    │       └─→ Identity (C-7) ─→ Google API
        │    (C-9)      │ ─→ DynamoDB に DelegationLog 保存
        └───────┬───────┘
                │
                ▼ HTTP API レスポンス（同期）
        ┌───────────────┐
        │  Frontend     │ 「代行完了」を画面表示（音声出力なし）
        │     (C-1)     │
        └───────────────┘
```

---

## 4. 障害伝搬と境界

| 障害発生元 | 影響範囲 | 緩和策 |
|---|---|---|
| Bedrock スロットリング | Agent / Sabori Logic | リトライ + 5/10 までに Service Quotas 引上申請 |
| Google API レート制限 | Gateway / Sabori Logic / Executor | Exponential Backoff、ユーザー通知（画面表示） |
| AgentCore Runtime コールドスタート | Voice Interaction | EventBridge Scheduler で 5 分おきウォーミング invoke |
| Transcribe ネットワーク切断 | Voice Stream | フロント側で再接続、ユーザーに再発話依頼 |
| DynamoDB 一時障害 | 全機能 | Exponential Backoff、ユーザー通知 |
| Cognito 認証切れ | Frontend | リフレッシュ → 失敗時はログイン画面に戻す |

---

## 5. 開発時の依存順序（実装順の参考）

1. **基盤** — Amplify Gen2 プロジェクト初期化、Cognito、CDK スタック雛形
2. **C-7 Identity** — Google OAuth Provider 設定と Workload Identity
3. **C-5 / C-6 Tool Lambda** — Calendar Tool / Gmail Tool（Identity 経由で Google API 呼び出し）の素単体テストまで
4. **C-10 DynamoDB** — 単一テーブル設計とアクセスパターン実装
5. **C-4 Agent** — Strands Agent + Bedrock Converse API tool_use ループ実装、Tool Lambda と疎通
6. **C-8 Sabori Logic** — 日次サボリスト生成
7. **C-2 API + C-1 Frontend（テキスト UI ファースト）** — 音声なしで一連のフローを通せる状態
8. **C-3 Voice Stream + Transcribe** — 音声入力対応
9. **C-9 Delegation Executor** — 代行実行統合
10. **C-11 Observability** — ログ・メトリクス整備
11. **決勝向け S-5 Autonomous Execution** — 自律実行モード（DynamoDB ログ参照）
