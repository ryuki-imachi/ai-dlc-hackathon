# Application Design — マカセル

> **AI-DLC Inception Phase / Application Design 統合ドキュメント**
>
> **作成日**: 2026-05-08
> **入力**: [`requirements.md`](../requirements/requirements.md), [`stories.md`](../user-stories/stories.md)
> **詳細**:
> - [`components.md`](./components.md) — コンポーネント定義と責務
> - [`component-methods.md`](./component-methods.md) — メソッドシグネチャ
> - [`services.md`](./services.md) — サービス層オーケストレーション
> - [`component-dependency.md`](./component-dependency.md) — 依存関係とデータフロー

---

## 0. エグゼクティブサマリー

「マカセル」は、八木案「サボってよいリスト AI 秘書」を **Bedrock AgentCore** で実装するハッカソン提案。
ユーザーは Google カレンダーと Gmail を連携し、AI Agent が毎日「サボってよい予定」を抽出して **承認なしで自律的に代行アクション（①他者への依頼 ②断り・欠席連絡 ③延期調整）を実行** する。
特に **「他者への依頼」を中心に据える** ことで、「ブッチ」ではなく「お願いして代わってもらう」社会的に成立する形を取る。

> **完全自律設計（2026-05-09 改訂）**: 旧設計の「下書き生成 → ユーザー承認 → 送信」のフローは廃止。
> AI は内部で下書きを生成しつつ、そのまま送信まで一気通貫で実行する。
> ユーザー入力（音声 / テキスト）は **任意の追加ヒント** として扱う。
> 暴走対策として、対象を明示ホワイトリストに限定 + モック宛先のみ + 緊急停止スイッチ + 事後ログ閲覧 の 4 段防御を取る。

> **「人をダメにする」の真の射程**: 表層は「ユーザー × AI の閉じた体験」だが、
> 裏層では **AI が API 越しに他者（同僚・上司・取引先）まで間接的に動かしている**。
> ユーザーは「サボった自覚」が薄れ、社会全体で「言いくるめの押し付け合い」が連鎖する悪夢を内包する。
> ハッカソンではホワイトハット実装に留めつつ、コンセプトとしては振り切る。

技術ハイライト:
- **AgentCore は最小限**（Runtime + Identity のみ）+ **Bedrock Converse API tool_use** で外部 API を扱う
- **Amazon Transcribe Streaming** による音声入力
- **Bedrock Claude Sonnet 4.6** による意図解釈・代行下書き生成
- **ap-northeast-1 単一リージョン** で完結

「人をダメにする」テーマに対し、**判断する力・実行する力を AI に取り上げられる** という回答を、技術スタックで実装する。

---

## 1. アーキテクチャ概要

### 1.1 システムコンテキスト（C4 Level 1）

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

### 1.2 コンテナ図（C4 Level 2）

```
   [Browser]
   ┌──────────────────────────────────────────┐
   │ Voice / Text Frontend (C-1)              │
   │ Next.js 15 + Amplify Hosting             │
   │ ┌─マイク─┐ ┌─テキスト入力欄─────────┐  │
   │ │  🎤   │ │ ［今日の朝会、サボる…］│  │
   │ └────────┘ └──────────────────────────┘  │
   └────┬─────────────────────────────┬───────┘
        │ HTTPS (Cognito JWT)         │ WebSocket
        │ POST /v1/instructions       │ (音声バイナリ)
        ▼                             ▼
   ┌──────────────────┐  ┌────────────────────┐
   │ API (C-2)        │  │ Voice Stream (C-3) │
   │ APIGW HTTP API + │  │ APIGW WS + Lambda  │
   │ Cognito Authorizer│ │ + Transcribe       │
   │ + Lambda         │  │                    │
   └─────┬────────────┘  └─────────┬──────────┘
         │                       │
         └────────┬──────────────┘
                  ▼
   ┌──────────────────────────────────────┐
   │ AI Agent (C-4)                       │
   │ AgentCore Runtime + Strands Agents   │
   │ Bedrock Converse API (tool_use)      │
   └──┬──────────────────┬──────────┬─────┘
      │ Bedrock          │ tool_use │ get_oauth_token()
      ▼                  ▼          ▼
   ┌────────┐    ┌──────────────┐  ┌──────────────┐
   │Bedrock │    │Calendar Tool │  │  AgentCore   │
   │Claude  │    │Lambda (C-5)  │  │  Identity    │
   │Sonnet  │    │              │  │   (C-7)      │
   │ 4.6    │    │Gmail Tool    │  │ OAuth2 Token │
   └────────┘    │Lambda (C-6)  │  │  Provider    │
                 └──────┬───────┘  └──────┬───────┘
                        │                 │
                        └────────┬────────┘
                                 ▼
                  Google APIs (Calendar / Gmail)

   ┌─────────────────────┐    ┌─────────────────────┐
   │ Sabori Logic (C-8)  │    │ Delegation Exec     │
   │ Lambda + Scheduler  │    │ Lambda (C-9)        │
   │   (Agent invoke)    │    │  (Tool Lambda 呼出) │
   └──────────┬──────────┘    └──────────┬──────────┘
              └───────────┬──────────────┘
                          ▼
                ┌─────────────────────┐
                │ DynamoDB (C-10)     │
                │ ap-northeast-1      │
                │ TTL 30d / PITR OFF  │
                └─────────────────────┘
                          │
                          ▼
                ┌─────────────────────┐
                │ Observability(C-11) │
                │ CloudWatch + X-Ray  │
                └─────────────────────┘
```

> Memory（旧 C-5）と Gateway（旧 C-6）は **不採用**。tool_use + Lambda Tool で代替。

---

## 2. コンポーネント要約

| ID | コンポーネント | 責務 | 主要技術 |
|---|---|---|---|
| C-1 | Voice / Text Frontend | 音声入力（メイン）+ テキスト入力（サブ）UI、サボリスト表示、画面のみで応答 | Next.js 15 + Amplify Hosting + Tailwind |
| C-2 | API / BFF Layer | REST HTTP API、Cognito 認証、レート制御 | API Gateway HTTP API + Cognito Authorizer + Lambda |
| C-3 | Voice Stream Handler | 音声 → Transcribe → Agent 転送 | API Gateway WebSocket + Lambda + Transcribe Streaming |
| C-4 | AI Agent | 意図解釈・tool_use ループでツール呼び出し・代行下書き生成 | AgentCore Runtime + Strands Agents + Bedrock Converse API（**tool_use**）+ Claude Sonnet 4.6 |
| C-5 | Calendar Tool Lambda | tool_use の Tool 実体。Google Calendar 操作 | Lambda + Google Calendar API + Identity 連携 |
| C-6 | Gmail Tool Lambda | tool_use の Tool 実体。Gmail 操作 | Lambda + Gmail API + Identity 連携 |
| C-7 | Identity Provider | Google OAuth トークン管理（アウトバウンド認証） | **AgentCore Identity**（OAuth2 Provider） |
| C-8 | Sabori Logic Lambda | 日次サボリスト生成 | Lambda + EventBridge Scheduler + AI Agent 呼び出し |
| C-9 | Delegation Executor Lambda | 承認後の代行実行 | Lambda + Tool Lambda（C-5/C-6）呼び出し |
| C-10 | Data Store | サボリスト・代行ログ・設定 | DynamoDB（東京、TTL 30d） |
| C-11 | Observability | ログ・メトリクス | CloudWatch + X-Ray |

> **設計変更（2026-05-08, ADR-1' 改訂版）**: 旧 C-5 Memory Service と C-6 External API Gateway（AgentCore Gateway）は不採用。
> 旧 C-6 の役割は **Bedrock tool_use + Lambda Tool** に置き換え（新 C-5 / C-6）。Memory は MVP では不要のため削除。

詳細は [`components.md`](./components.md) 参照。

---

## 3. サービス層

| ID | サービス | 責務 |
|---|---|---|
| S-1 | Voice Interaction Service | 音声入力 → STT → Agent → 結果配信のフロー |
| S-2 | Daily Sabori Generation Service | 朝のサボリスト生成（cron / 初回ログイン） |
| S-3 | Delegation Approval & Execution Service | 「サボる」発話 → 下書き → 承認 → 実行 |
| S-4 | OAuth Bootstrap Service | Google OAuth 初期連携フロー |
| S-5 | Autonomous Execution Service | 自律実行モード（決勝向け） |

詳細は [`services.md`](./services.md) 参照。

---

## 4. データモデル（DynamoDB 単一テーブル）

| Pattern | PK | SK | エンティティ |
|---|---|---|---|
| 1 | `USER#<userId>` | `SABORI#<date>#<itemId>` | サボリ項目 |
| 2 | `USER#<userId>` | `DRAFT#<itemId>` | 代行下書き |
| 3 | `USER#<userId>` | `LOG#<timestamp>#<logId>` | 代行ログ（30 日 TTL） |
| 4 | `USER#<userId>` | `SETTINGS` | ユーザー設定 |

GSI: 不要（MVP 範囲）。

---

## 5. 主要シーケンス: 音声でサボる → 承認 → 実行

`tool_use` ループの動きを明示する。

```
ユーザー発話              Voice Stream    AI Agent           Bedrock Converse    Calendar Tool   Identity     Google API   DynamoDB
「明日の朝会、いらない」    │              │                  │                   │               │            │            │
   │─ WebSocket 音声 ────▶│              │                  │                   │               │            │            │
   │                      │─ Transcribe ▶│                  │                   │               │            │            │
   │                      │  text         │                  │                   │               │            │            │
   │                      │─ Runtime ────▶│                  │                   │               │            │            │
   │                      │  invoke       │─ Converse ──────▶│                   │               │            │            │
   │                      │               │  (tools 定義)    │                   │               │            │            │
   │                      │               │◀── tool_use ─────│                   │               │            │            │
   │                      │               │  list_events 要求│                   │               │            │            │
   │                      │               │── invoke ───────────────────────────▶│               │            │            │
   │                      │               │                  │                   │─ get_token ──▶│            │            │
   │                      │               │                  │                   │◀──────────────│            │            │
   │                      │               │                  │                   │─ list ──────────────────────▶│           │
   │                      │               │                  │                   │◀────────────────────────────│           │
   │                      │               │◀── tool_result ──│                   │               │            │            │
   │                      │               │── Converse ─────▶│                   │               │            │            │
   │                      │               │   (再推論)       │                   │               │            │            │
   │                      │               │◀── final text ───│                   │               │            │            │
   │                      │               │── putDraft ──────────────────────────────────────────────────────────────────▶│
   │                      │               │── Subscription notify                                                          │
   │◀── 下書き表示 ────────────────────────────────────────────────────────────────────────────────────────────────────────│

「うん、それで」          │              │                  │                   │               │            │            │
   │─ WebSocket ─────────▶│              │                  │                   │               │            │            │
   │                      │─ invoke ─────▶│                  │                   │               │            │            │
   │                      │               │─ Converse ──────▶│                   │               │            │            │
   │                      │               │◀── tool_use ─────│ approve_delegation│               │            │            │
   │                      │               │── Executor invoke                                                              │
   │                      │               │   ──▶ Calendar Tool（update_attendance）                                       │
   │                      │               │   Tool 内で Identity → Google API                                              │
   │                      │               │── putLog ────────────────────────────────────────────────────────────────────▶│
   │◀── 「代行完了」画面表示 ──────────────────────────────────────────────────────────────────────────────────────────────│
```

ポイント:
- AI Agent (Strands on Runtime) は **Bedrock Converse API の tool_use ループ** を回し、必要に応じて Lambda Tool を呼ぶ
- Lambda Tool 内部で **AgentCore Identity から OAuth トークン取得** → Google API 呼び出し
- AgentCore Gateway を使わないため、MCP プロトコルではなく素の Lambda invoke で完結する
- **テキスト入力経路**: 上記シーケンスの「WebSocket 音声 → Transcribe」を **HTTP API `POST /v1/instructions`** に置き換えるだけで、AI Agent invoke 以降は同じフロー
- **🆕 完全自律経路（メイン）**: 上記の「ユーザー発話」は不要。EventBridge Scheduler が日次トリガで AI Agent を直接 invoke、tool_use ループでサボリ判定 → 代行送信まで一気通貫で実行（ユーザー承認ステップなし）

---

## 6. 主要 NFR への対応

| NFR | 対応 |
|---|---|
| **NFR-DEMO-1**「人をダメにする」が 15 秒で伝わる | フロントの初期 LP に Intent コピー **「マカセル — 任せ続けたら、もう戻れない。」** を配置 |
| **NFR-DEMO-2** 二重構造（助ける↔奪う） | 代行ログ画面で「AI が代行した数」を可視化、Skill 連動は補足扱い |
| **NFR-DEMO-3** 裏で AI が何をしているか | AppSync Subscription で代行ステータスを画面更新、ログ画面で履歴可視化 |
| **NFR-DEMO-4** アプリ依存促さない | 通知系（プッシュ・音声）を一切実装しない |
| **音声入力（メイン）+ テキスト入力（サブ）** | US-1.3 / US-2.2 を WebSocket + Transcribe + Agent と GraphQL Mutation の 2 経路で実装 |
| **東京リージョン** | 全コンポーネント ap-northeast-1 にデプロイ |
| **コスト軽量** | Sonnet 4.6 を使用、Memory Strategy は summarization のみ、Gateway は最小ターゲット |

---

## 7. 採用 AWS サービス一覧

| カテゴリ | サービス | 用途 |
|---|---|---|
| AI / Agent | Amazon Bedrock | Claude Sonnet 4.6（意図解釈・代行下書き）+ Converse API **tool_use** |
| AI / Agent | Bedrock AgentCore Runtime | エージェントホスティング（Strands Agent） |
| AI / Agent | Bedrock AgentCore Identity | Google OAuth トークン管理（アウトバウンド認証） |
| ~~AgentCore Memory~~ | **不採用** | MVP では不要。必要になれば DynamoDB 簡易記憶 or 後付け |
| ~~AgentCore Gateway~~ | **不採用** | Bedrock Converse API の tool_use + Lambda Tool で代替 |
| 音声 | Amazon Transcribe Streaming | リアルタイム STT |
| フロント | **AWS Amplify Hosting** | Next.js 15 ホスティング + CI/CD（Amplify は Hosting レイヤとして残存） |
| 認証 | **Amazon Cognito User Pool**（CDK 構築） | HTTP API Authorizer に直接統合 |
| 認証 | **Amplify Auth クライアント SDK**（フロント） | Next.js から Cognito にサインインする JS ライブラリとして利用 |
| API | **Amazon API Gateway HTTP API** | REST API（Cognito JWT Authorizer 適用） |
| API | API Gateway WebSocket | 音声ストリーミング |
| ~~AppSync~~ | **不採用** | Python Lambda 混在の摩擦と Subscription を必須としないため、HTTP API + Lambda に切り替え（ADR-6） |
| コンピュート | AWS Lambda | API Resolver / Sabori Logic / Executor |
| スケジューラ | Amazon EventBridge Scheduler | 日次サボリスト生成トリガ |
| データ | Amazon DynamoDB | 単一テーブル、TTL 30 日 |
| 観測性 | Amazon CloudWatch + AWS X-Ray | ログ・メトリクス・トレース |
| IaC | **AWS CDK（TypeScript）主導 + Amplify Hosting 軽め共存** | バックエンドリソース（Cognito / API GW / Lambda / DynamoDB / AgentCore）はすべて CDK で定義。Amplify は Hosting / CI/CD と Auth クライアント SDK のみ利用（Amplify Gen2 の `defineAuth` / `defineData` / `defineFunction` は不採用） |

---

## 8. 設計上の意思決定（ADR ライト）

### ADR-1（廃案）: AgentCore フル採用
- **背景**: ハッカソン審査「アイデアと技術のバランス」を満たすため、当初は Runtime + Memory + Gateway + Identity のフル採用を選択
- **状態**: **2026-05-08 に廃案** — 過剰実装と判断、ADR-1' に置換

### ADR-1' (改訂版): AgentCore は最小限、tool_use 中心へピボット
- **改訂日**: 2026-05-08
- **背景**: ADR-1（フル採用）から「過剰にしない」方向に変更したいというユーザー判断（レビュー）。CDK 実装可能性調査で alpha 版 L2 のリスクや、Gateway Target の OpenAPI スキーマ管理コストが見えたことも背景
- **方針**:
    - **AgentCore Runtime** は採用（Strands Agent ホスティング、技術アピールも兼ねる）
    - **AgentCore Identity** は採用（Google OAuth トークン管理 = リフレッシュトークン更新・暗号化保存・ユーザー分離を AWS に任せる）
    - **AgentCore Memory** は不採用（MVP では会話履歴なしで成立、必要なら DynamoDB 簡易記憶で代替）
    - **AgentCore Gateway** は不採用（**Bedrock Converse API の tool_use** + Lambda Tool で代替）
- **トレードオフ**:
    - メリット: 実装スコープ削減、CDK alpha 依存を Runtime + Identity の 2 つに絞れる、tool_use は Lambda の機能設計と直結し見通しが良い
    - デメリット: Gateway を使わないので「MCP として外部 API を統一」のアピールは弱まる
- **採用しない選択肢**:
    - AgentCore ゼロ（Runtime まで外す）: 「アイデアと技術のバランス」のうち技術側が薄くなる
    - Memory も使う: MVP では履歴連動の体験をしないため不要

### ADR-2: ap-northeast-1 単一リージョン
- **背景**: 個人データ（カレンダー / メール）は東京推奨、レビュー v1 で Sonnet 4.6 利用可確認済
- **選択肢**: ①us-west-2 主 ②東京一本 ③マルチリージョン
- **決定**: ②東京一本
- **トレードオフ**: AgentCore の一部機能やモデルが ap-northeast-1 で未提供の場合がある
- **緩和策**: Construction フェーズで各サービスの東京対応状況を確認、必要なら個別にクロスリージョン推論を採用

### ADR-2: ap-northeast-1 単一リージョン
- **背景**: 個人データ（カレンダー / メール）は東京推奨、レビュー v1 で Sonnet 4.6 利用可確認済
- **選択肢**: ①us-west-2 主 ②東京一本 ③マルチリージョン
- **決定**: ②東京一本
- **トレードオフ**: AgentCore の一部機能やモデルが ap-northeast-1 で未提供の場合がある
- **緩和策**: Construction フェーズで各サービスの東京対応状況を確認、必要なら個別にクロスリージョン推論を採用

### ADR-3: 通知・音声出力をしない
- **背景**: 要件「アプリ依存させたくない」「音声出力しない」
- **選択肢**: ①通知あり ②通知なし、音声出力あり ③両方なし
- **決定**: ③両方なし
- **トレードオフ**: ユーザーへのリーチが弱い
- **緩和策**: ユーザー側からの能動的な音声問い合わせで体験を担保

### ADR-4: Next.js + Amplify Hosting
- **背景**: 要件「ペルソナ深掘りしない」「軽量に進める」、リュウキ氏の学習ニーズ
- **選択肢**: ①PWA + Vite ②Next.js + Amplify ③Next.js + CDK ④React Native
- **決定**: ②Next.js + Amplify
- **トレードオフ**: Amplify の制約に縛られる
- **緩和策**: Auth / Hosting は Amplify、複雑な部分は CDK スタックで補強

### ADR-5: Amazon Transcribe Streaming
- **背景**: US-1.3 / US-2.2 で音声入力が必須
- **選択肢**: ①Web Speech API ②Transcribe Streaming ③Nova Sonic ④段階的
- **決定**: ②Transcribe Streaming
- **トレードオフ**: WebSocket セッションのコスト・実装負荷
- **緩和策**: AWS スタックで「アイデアと技術のバランス」を担保

### ADR-6: API Layer を AppSync → API Gateway HTTP API + Lambda に切り替え
- **改訂日**: 2026-05-08
- **背景**: 当初 Amplify Gen2 + AppSync を採用予定だったが、(a) Amplify Gen2 の `defineFunction()` は Node.js (TypeScript) のみで、本案の **Python Lambda（Strands Agent / Tool）が混在しづらい**、(b) Amplify が Lambda タイムアウトを 30 秒に強制上書きする既知バグ（GitHub Issue #2026）が tool_use ループの実行時間と衝突するリスク、(c) Subscription（リアルタイム配信）は MVP で必須ではない、という 3 点が判明
- **選択肢**: ①AppSync 維持 + 権限手動配線 ②API Gateway HTTP API + Lambda（CDK 主導）③AppSync 維持 + Subscription 不使用
- **決定**: ②API Gateway HTTP API + Lambda
- **トレードオフ**:
    - メリット: タイムアウト制約なし、Python Lambda と一貫した CDK 構成、フロント↔バック契約は素朴な REST で見通し良好
    - デメリット: 自動 CRUD Resolver / 自動 Subscription / 自動型生成の旨味は失う。フロントとバックの型は手動で揃える（OpenAPI / 自前生成 or 簡易共有）
- **採用しない選択肢**:
    - AppSync 維持: 上記 (a)(b) のフリクションが Construction フェーズで時間を奪う
    - Subscription なし AppSync: 旨味の半減と複雑性のトレードオフが悪い
- **影響**: 主要 API は `POST /v1/instructions`（テキスト入力）/ `GET /v1/sabori-items` / `POST /v1/sabori-items/{id}/mark` / `POST /v1/delegations/{id}/approve` 等の HTTP エンドポイントに統一
- **Amplify の残存範囲**: Amplify は **完全には捨てない**。**Amplify Hosting**（Next.js のデプロイ + CI/CD）と **Amplify Auth クライアント SDK**（フロントから Cognito にサインインするライブラリ）は残す。捨てるのは Amplify Gen2 の `defineAuth` / `defineData` / `defineFunction` 系の **バックエンド IaC 機能のみ**

---

## 9. リスクと緩和策

| リスク | 影響 | 緩和策 |
|---|---|---|
| AgentCore Gateway / Identity が ap-northeast-1 で未提供 | 高 | Construction フェーズ初期に確認、必要ならクロスリージョン推論 |
| 5/30 までに 11 コンポーネント全てを動かせない | 高 | C-1〜C-10 のうち優先度に応じて段階的実装、C-11（Observability）は最低限のみ |
| Transcribe の発話認識精度（独り言レベル） | 中 | カスタム語彙登録、フォールバックで再発話依頼 |
| Bedrock スロットリング | 中 | 5/10 までに Service Quotas 引き上げ申請 |
| AgentCore Runtime コールドスタート | 中 | EventBridge Scheduler で 5 分おきウォーミング invoke |
| Google OAuth 申請の遅延 | 中 | 5/10 中に申請、内部テストモードで MVP 構築 |

---

## 10. 残論点（次工程: Units Generation / Construction）

| 論点 | 次工程 |
|---|---|
| Unit 分解（コンポーネント単位 vs 機能単位 vs 段階単位） | Units Generation |
| 各コンポーネントの実装工数見積もり | Units Generation |
| Functional Design の per-unit 詳細 | Construction Functional Design |
| プロンプトインジェクション対策の実装方法 | Construction NFR Requirements |
| 自律実行モード（S-5）のしきい値ロジック | 決勝向け Construction |
| 商標・ドメインの最終確認 | 5/10 までに別途実施 |
