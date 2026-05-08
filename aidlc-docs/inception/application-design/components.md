# Components — マカセル

> **AI-DLC Inception Phase / Application Design 成果物**
>
> 主要コンポーネントとその責務・インタフェースを定義する。
> 詳細な業務ロジックは Construction フェーズの Functional Design で記述する。
>
> ### 設計変更履歴
> - **2026-05-08 (ADR-1' 改訂版)**: AgentCore 採用範囲を Runtime + Identity のみに絞り、過剰実装を回避。
>   旧 C-5 Memory Service・旧 C-6 External API Gateway（AgentCore Gateway）を削除。
>   外部 API 呼び出しは **Bedrock Converse API の tool_use** + Lambda Tool（新 C-5 / C-6）で代替。
> - **2026-05-08 (ADR-6)**: C-2 API Layer を **AppSync → API Gateway HTTP API + Lambda** に切り替え。
>   Amplify Gen2 の Lambda 30 秒タイムアウト強制問題と Python Lambda 混在の摩擦を回避し、CDK 主導の構成に統一。Subscription は MVP で廃止。

---

## C-1. Voice / Text Frontend（フロントエンド）

| 項目 | 内容 |
|---|---|
| 責務 | ユーザーとの **音声・テキスト両方の入力 UI**、サボリスト表示、代行下書き表示。**音声・通知出力は行わない** |
| 技術 | Next.js 15（App Router）+ Tailwind CSS + Amplify Hosting |
| 主要画面 | (1) ホーム（サボリスト表示 + テキスト入力欄 + マイクボタン） / (2) 下書き確認モーダル / (3) ログ表示 / (4) 設定（OAuth 連携） |
| 入力 IF | (a) マイクからの WebSocket 音声ストリーミング（メイン） / (b) テキスト入力欄から GraphQL Mutation（サブ） / (c) タップ操作（補助） |
| 出力 IF | 画面表示のみ（テキスト） |
| 認証 | Amazon Cognito（Amplify Auth Gen2） |

## C-2. API / BFF Layer（API ゲートウェイ層）

| 項目 | 内容 |
|---|---|
| 責務 | フロントエンドと AI Agent / データストア間のリクエストルーティング。Cognito 認証、レート制御 |
| 技術 | **API Gateway HTTP API + Lambda Resolver**（CDK 主導で軽量化、ADR-6 で AppSync から切り替え） |
| 主要 API | (1) `POST /v1/instructions`（テキスト入力経路）/ (2) `GET /v1/sabori-items?date=`/ (3) `POST /v1/sabori-items/{itemId}/mark`/ (4) `POST /v1/delegations/{itemId}/approve`/ (5) `POST /v1/delegations/{itemId}/reject`/ (6) `GET /v1/delegation-logs`/ (7) `PUT /v1/settings` |
| 認証連携 | Cognito User Pool Authorizer（JWT 検証）を HTTP API に設定 |
| 配信方式 | リアルタイム配信は MVP では実装せず、Mutation 系 API の同期応答 + 必要に応じてフロントのポーリング。Subscription（旧 AppSync 機能）は廃止 |

## C-3. Voice Stream Handler（音声ストリーミングハンドラ）

| 項目 | 内容 |
|---|---|
| 責務 | ブラウザからの WebSocket 音声ストリームを受け、Amazon Transcribe Streaming へ転送、テキスト化結果を AI Agent に渡す |
| 技術 | API Gateway WebSocket + Lambda、Amazon Transcribe Streaming SDK |
| Input | フロントエンドからの音声バイナリ（PCM 16kHz） |
| Output | テキスト化された発話 → AI Agent（C-4） |
| エラー処理 | 認識失敗 / タイムアウト / セッション切断 |

## C-4. AI Agent (AgentCore Runtime + Bedrock tool_use)

| 項目 | 内容 |
|---|---|
| 責務 | 音声入力テキストから意図を解釈し、Bedrock Converse API の **tool_use ループ** で Lambda Tool を呼び出して Google API 操作を実行。サボリ判定・代行下書き生成までを担う |
| 技術 | Amazon Bedrock AgentCore Runtime、Strands Agents（Python）、Bedrock Converse API（tool_use）、Claude Sonnet 4.6 |
| 内部構成 | (1) Strands Agent エントリポイント / (2) tool_use ループ / (3) Tool 呼び出し（Calendar Tool / Gmail Tool に invoke） / (4) Identity から OAuth token 取得は Tool 側で実施 |
| Input | ユーザー発話テキスト、コンテキスト（ユーザー ID / セッション） |
| Output | アクション結果（テキスト or 構造化データ） |

> **設計変更（2026-05-08）**: 旧 C-5 Memory（AgentCore Memory）と旧 C-6 External API Gateway（AgentCore Gateway）は削除。
> 会話履歴は MVP で持たない（必要なら DynamoDB で簡易記憶を後付け）。外部 API は新 C-5 / C-6 の Lambda Tool に置き換え。

## C-5. Calendar Tool Lambda（tool_use Tool 実体）

| 項目 | 内容 |
|---|---|
| 責務 | Bedrock Converse API の tool_use 経由で AI Agent から呼ばれ、Google Calendar API を操作する |
| 技術 | AWS Lambda（Python）+ Google Calendar API SDK + AgentCore Identity SDK |
| 主要操作 | (1) `list_events(date)` / (2) `update_attendance(eventId, status)` |
| 認証 | Lambda 内で AgentCore Identity から有効な Google OAuth Access Token を取得 |
| Input | `userId`、操作種別、操作パラメータ |
| Output | Google API のレスポンスを構造化して返す（tool_result として AI Agent に返却） |

## C-6. Gmail Tool Lambda（tool_use Tool 実体）

| 項目 | 内容 |
|---|---|
| 責務 | Bedrock Converse API の tool_use 経由で AI Agent から呼ばれ、Gmail API を操作する |
| 技術 | AWS Lambda（Python）+ Gmail API SDK + AgentCore Identity SDK |
| 主要操作 | (1) `list_recent(limit)` / (2) `send(to, subject, body)` |
| 認証 | Lambda 内で AgentCore Identity から有効な Google OAuth Access Token を取得 |
| Input | `userId`、操作種別、操作パラメータ |
| Output | Google API のレスポンスを構造化して返す |

## C-7. Identity Provider (AgentCore Identity)

| 項目 | 内容 |
|---|---|
| 責務 | ユーザーごとの Google OAuth 2.0 トークン（Calendar / Gmail スコープ）を保持・更新する |
| 技術 | Amazon Bedrock AgentCore Identity（OAuth2 Provider 機能、`GoogleOauth2`） |
| 連携 | Cognito ユーザーと AgentCore Workload Identity を紐付け |
| トークン更新 | リフレッシュトークンで自動更新 |
| 利用元 | C-5 / C-6 / C-9 から `get_oauth_token(userId, "google")` で取得 |
| 採用理由 | 自前で Secrets Manager + リフレッシュトークン管理を書くより、Identity を使う方が実装負荷が低い（ADR-1'） |

## C-8. Sabori Logic Lambda（サボリ判定ロジック）

| 項目 | 内容 |
|---|---|
| 責務 | カレンダーから「サボってよい予定」の候補抽出ロジック（ルール + LLM）。日次バッチも担当 |
| 技術 | AWS Lambda（Python or TypeScript）+ EventBridge Scheduler（日次起動） |
| Input | ユーザー ID、対象日 |
| Output | サボリスト（JSON） → DynamoDB に書き込み |

## C-9. Delegation Executor Lambda（代行アクション実行）

| 項目 | 内容 |
|---|---|
| 責務 | 承認された代行アクションを実行、結果を DynamoDB に記録。Tool Lambda（C-5 / C-6）を呼び出す形 |
| 技術 | AWS Lambda + Tool Lambda invoke |
| Input | 承認済みアクション ID、ユーザー ID |
| Output | 実行結果（成功 / 失敗）→ DynamoDB ログ |

## C-10. Data Store（データ層）

| 項目 | 内容 |
|---|---|
| 責務 | サボリスト / 代行ログ / ユーザー設定の永続化 |
| 技術 | Amazon DynamoDB（東京リージョン、PITR OFF、TTL 30 日） |
| パーティション設計 | 単一テーブル（PK = USER#<userId>, SK = ITEM#<type>#<timestamp>） |
| 主要エンティティ | (1) SaboriItem / (2) DelegationLog / (3) UserSettings |

## C-11. Observability（観測性、最低限）

| 項目 | 内容 |
|---|---|
| 責務 | ログ集約・基本メトリクス・エラー検知 |
| 技術 | CloudWatch Logs / Metrics、AWS X-Ray（Lambda・AgentCore Runtime） |
| 範囲 | LLM 呼び出し成功率・レイテンシ、Google API 呼び出し成功率、コスト目安（手動確認）|
| 注意 | Q-F2 で Optional 成果物「観測性設計」は省略方針。最低限のセットのみ |

---

## コンテキスト図（C4 Level 1）

```
                  ┌──────────────────────┐
                  │  鈴木健太（ユーザー）│
                  │   音声で指示する      │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │      マカセル         │
                  │  （本ハッカソン製品） │
                  └─────┬───────────┬────┘
                        │           │
            ┌───────────┘           └───────────┐
            ▼                                    ▼
  ┌──────────────────┐                ┌─────────────────────┐
  │  Google Calendar │                │      Gmail          │
  │     (外部 SaaS)  │                │   (外部 SaaS)       │
  └──────────────────┘                └─────────────────────┘
```

---

## コンテナ図（C4 Level 2）

```
                 ┌─────────────────────────────────────────┐
                 │       Voice / Text Frontend (C-1)       │
                 │   Next.js 15 + Amplify Hosting          │
                 │  🎤 音声(メイン)  ⌨️ テキスト(サブ)     │
                 └────────────┬──────────────┬─────────────┘
                              │              │
              HTTP API (Cognito)         WebSocket (音声)
              [POST /v1/instructions]    [音声バイナリ]
                              │              │
                              ▼              ▼
                  ┌──────────────┐  ┌──────────────────────┐
                  │  API Layer   │  │ Voice Stream Handler │
                  │    (C-2)     │  │       (C-3)          │
                  │ APIGW HTTP + │  │ APIGW WS + Lambda    │
                  │ Lambda       │  │                      │
                  └──────┬───────┘  └─────────┬────────────┘
                         │                    │
                         │              Transcribe Streaming
                         │                    │
                         ▼                    ▼
                  ┌────────────────────────────────────────┐
                  │       AI Agent (C-4)                   │
                  │  AgentCore Runtime + Strands Agent     │
                  │  Bedrock Converse API (tool_use loop)  │
                  └──┬──────────────────┬──────────────────┘
                     │ Bedrock Converse │ tool_use → invoke
                     ▼                  ▼
              ┌──────────┐        ┌──────────────────────────┐
              │ Bedrock  │        │ Tool Lambdas             │
              │Claude 4.6│        │ ┌──────────────────────┐ │
              └──────────┘        │ │ Calendar Tool (C-5)  │ │
                                  │ │ Gmail Tool   (C-6)   │ │
                                  │ └─────────┬────────────┘ │
                                  └───────────┼──────────────┘
                                              │ get_oauth_token()
                                              ▼
                                  ┌──────────────────────────┐
                                  │ AgentCore Identity (C-7) │
                                  │ Google OAuth2 Provider   │
                                  └─────────────┬────────────┘
                                                │
                                  Google APIs（Calendar / Gmail）
                                                │
                                                ▼
                                          (外部 SaaS)

         ┌────────────────┐  ┌──────────────────────┐
         │ Sabori Logic   │  │ Delegation Executor  │
         │ Lambda (C-8)   │  │ Lambda (C-9)         │
         └────────┬───────┘  └──────────┬───────────┘
                  │                     │
                  └─────────┬───────────┘
                            ▼
                   ┌─────────────────────┐
                   │   DynamoDB (C-10)   │
                   │   ap-northeast-1    │
                   └─────────────────────┘
                            │
                            ▼
                   ┌─────────────────────┐
                   │ Observability (C-11)│
                   │  CloudWatch / X-Ray │
                   └─────────────────────┘
```

> 旧 C-5 Memory（AgentCore Memory）・旧 C-6 External API Gateway（AgentCore Gateway）は不採用。
> tool_use + Lambda Tool で代替（新 C-5 / C-6）。
