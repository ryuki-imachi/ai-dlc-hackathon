# Services — マカセル

> **AI-DLC Inception Phase / Application Design 成果物**
>
> サービス層のオーケストレーションを定義する。
>
> ### 設計変更履歴
> - **2026-05-08 (ADR-1' 改訂版)**: AgentCore Memory / Gateway を不採用とし、**Bedrock Converse API の tool_use** + Lambda Tool（C-5/C-6）に置き換え。Identity（C-7）は引き続き採用。
> - **2026-05-08 (ADR-6)**: API Layer を **AppSync → API Gateway HTTP API + Lambda** に切り替え。Subscription を MVP で廃止し、Mutation 系 API の同期応答 + 必要に応じてポーリングに集約。

---

## S-1. Voice / Text Interaction Service（音声・テキスト対話オーケストレーション）

| 項目 | 内容 |
|---|---|
| 責務 | フロントの **音声入力（メイン）または テキスト入力（サブ）** を受け、AI Agent を呼び出し、結果をフロントに返す一連のフロー。両入力経路は最終的に同じ AI Agent invoke に集約される |
| 関連コンポーネント | C-1（Frontend）、C-2（API Layer）、C-3（Voice Stream Handler）、C-4（AI Agent） |
| 主要ステップ（音声経路） | (1) WebSocket 接続 → (2) 音声ストリーム → (3) Transcribe → (4) Agent invoke（内部で tool_use ループ）→ (5) 結果配信 |
| 主要ステップ（テキスト経路） | (1) `POST /v1/instructions { text, sessionId? }`（HTTP API + Cognito Authorizer）→ (2) Lambda Resolver → (4) Agent invoke（内部で tool_use ループ）→ (5) 結果配信 |
| 同期/非同期 | フロント↔WebSocket は同期的、フロント↔HTTP API も同期的、Agent 内部は Bedrock Converse の tool_use ループを最大 N 回まで回す |
| エラー処理 | Transcribe 失敗→画面に再発話依頼（テキスト入力で続行可）、Agent 失敗→画面にエラー表示、tool_use ループ上限到達→失敗としてユーザーに通知 |

### Sequence

```
User           Frontend         WS Handler      Transcribe       AI Agent       Bedrock/Tools
 │  発話         │                  │              │                │                │
 ├──────────────▶│                  │              │                │                │
 │               ├─ WS connect ────▶│              │                │                │
 │               ├─ audio chunks ──▶├─ stream ────▶│                │                │
 │               │                  │              ├─ text ────────▶│                │
 │               │                  │              │                ├─ LLM call ────▶│
 │               │                  │              │                │◀───────────────┤
 │               │                  │              │                │ tool calls...  │
 │               │                  │              │                │◀───────────────┤
 │               │                  │              │◀── result ─────┤                │
 │               │◀── status ───────┤              │                │                │
 │               │   updates        │              │                │                │
 │  画面確認     │                  │              │                │                │
```

---

## S-2. Daily Sabori Generation Service（日次サボリスト生成）

| 項目 | 内容 |
|---|---|
| 責務 | 毎朝、各ユーザーのサボリストを AI に生成させ、DynamoDB に書き込む |
| 関連コンポーネント | C-8（Sabori Logic Lambda）、C-4（AI Agent）、C-6（Gateway）、C-10（DynamoDB）|
| トリガ | EventBridge Scheduler（cron(0 6 * * ? *) JST）、または初回ログイン時オンデマンド |
| 同期/非同期 | 完全非同期、結果はサブスクライブで通知 |
| 失敗時 | 1 回リトライ、それでも失敗ならログ出力 |

### Sequence

```
EventBridge      Sabori Lambda     AI Agent           Calendar API      DynamoDB
   │                 │                │                    │               │
   ├─ cron ─────────▶│                │                    │               │
   │                 ├─ invoke ──────▶│                    │               │
   │                 │                ├─ list_events ─────▶│               │
   │                 │                │◀───────────────────┤               │
   │                 │                │ filter & rank      │               │
   │                 │◀── candidates ─┤                    │               │
   │                 ├─ putSaboriItems ────────────────────────────────────▶│
   │                 │                │                    │               │
```

---

## S-3. Autonomous Delegation Service（自律代行サービス）

| 項目 | 内容 |
|---|---|
| 責務 | サボリ判定 → 代行下書き生成 → 自律送信までを **ユーザー承認なしで一気通貫** に実行する。2026-05-09 設計変更で「承認」ステップを廃止 |
| 関連コンポーネント | C-4（AI Agent）、C-5/C-6（Tool Lambda）、C-8（Sabori Logic）、C-9（Delegation Executor）、C-10（DynamoDB） |
| トリガ | (a) 日次バッチ（EventBridge Scheduler）、(b) 任意のユーザー追加指示（音声 / テキスト）、(c) ユーザーが連携した直後の初回バックフィル |
| セーフガード | (a) ホワイトリスト判定（FR-1.8）、(b) モック宛先のみ、(c) 緊急停止スイッチ（FR-1.9）、(d) 事後ログ閲覧（FR-1.3-A） |
| 主要状態遷移 | pending → in_safelist? → drafted → executed (or filtered_out / failed) |

### State Machine

```
[pending]
    │ AI が判定
    ├─ ホワイトリスト範囲外 ──▶ [filtered_out]（何もしない）
    │
    ▼ ホワイトリスト範囲内
[in_safelist]
    │ AI が下書き生成（内部処理、ユーザー非表示）
    ▼
[drafted]
    │ AI が即座に送信
    ▼
[executed]  または  [failed]（リトライ 1 回 → ログに失敗記録）
```

> 旧版の `marked_as_sabori` / `approved` / `rejected` ステートは削除。ユーザー承認はないため、ステート遷移は AI 内部で完結する。

---

## S-4. OAuth Bootstrap Service（Google OAuth 連携初期化）

| 項目 | 内容 |
|---|---|
| 責務 | ユーザーの初回 Google OAuth 連携を AgentCore Identity 経由で行う |
| 関連コンポーネント | C-1（Frontend）、C-2（API）、C-7（Identity）|
| ステップ | (1) ユーザーが「Google 連携」ボタン → (2) Identity が認証 URL 生成 → (3) ユーザーが Google で同意 → (4) コールバック → (5) Identity がトークン保管 |
| 必要 Scope | `https://www.googleapis.com/auth/calendar` + `https://www.googleapis.com/auth/gmail.send` |
| トークン更新 | Identity 内部でリフレッシュ、失効時はフロントに再連携を促す |

---

## S-5. Quiet Mode Service（戻れない境地モード、決勝向け）

| 項目 | 内容 |
|---|---|
| 責務 | 段階 2「TODO リストなのに、見ない」を実現。代行ログを時間経過で自動隠蔽し、ユーザーがマカセルを意識しなくてもアクションが完了する状態を作る |
| 関連コンポーネント | C-1（Frontend）、C-10（DynamoDB のログ参照）、C-2（API Layer） |
| Scope | **決勝向け**。MVP では実装せず、UX 演出のみ |
| ロジック | (a) ログ表示の自動フィルタ（成功した代行は N 日経過でデフォルト非表示）、(b) ダッシュボードのサマリ表示（中身を意識せず数だけ把握）、(c) 30 日後ストーリーの演出データ |
| 注意 | 旧 S-5「自律実行モード」は段階 1 に統合されたため不要。決勝向けに役割を「戻れない境地」演出に振り直し |

---

## サービス間オーケストレーション一覧

| サービス | トリガ | 主要 Output | 連動先 |
|---|---|---|---|
| S-1 Voice Interaction | ユーザー発話 | Agent Response | S-3 |
| S-2 Daily Sabori Generation | EventBridge / 初回ログイン | SaboriItem[] | フロント表示 |
| S-3 Delegation Approval & Execution | 「サボる」発話 / Mutation | DelegationLog | フロント表示 |
| S-4 OAuth Bootstrap | ユーザーの連携操作 | OAuth Token 保管 | S-2 / S-3 利用可能化 |
| S-5 Autonomous Execution | （決勝のみ） | 自律実行ログ | S-3 を承認なしで起動 |
