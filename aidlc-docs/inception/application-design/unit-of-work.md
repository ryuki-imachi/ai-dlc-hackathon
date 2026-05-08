# Unit of Work — マカセル

> **AI-DLC Inception Phase / Units Generation 成果物**
>
> **作成日**: 2026-05-08
> **入力**: [requirements.md](../requirements/requirements.md), [stories.md](../user-stories/stories.md), [application-design.md](./application-design.md)
> **方針**: ハイブリッド型分解（機能ドメイン + 横断基盤 + 拡張）

---

## 0. Unit 一覧サマリ

| Unit | 名前 | 責務（一行） | Scope | 関連コンポーネント |
|---|---|---|---|---|
| **U-0** | Foundation | CDK 共通基盤 / Cognito / DynamoDB / Observability の土台 | MVP 必須 | C-10, C-11, Cognito |
| **U-1** | Identity & Google Tools | AgentCore Identity + Calendar/Gmail Tool Lambda | MVP 必須 | C-5, C-6, C-7 |
| **U-2** | AI Agent Core | AgentCore Runtime + Strands Agent + tool_use ループ | MVP 必須 | C-4 |
| **U-3** | Sabori Domain | 日次サボリスト生成 + 関連 API | MVP 必須（段階1） | C-2 の一部, C-8 |
| **U-4** | Autonomous Delegation Domain | 自律代行（判定 → 下書き → 送信を承認なしで一気通貫）+ 関連 API + セーフガード | MVP 必須（段階 1 の核） | C-2 の一部, C-9 |
| **U-5** | Voice Input Pipeline | WebSocket + Transcribe Streaming + Voice Stream Handler | MVP 必須 | C-3 |
| **U-6** | Frontend | Next.js + Amplify Hosting + テキスト入力 UI | MVP 必須 | C-1 |
| **U-7** | Quiet Mode（戻れない境地） | ログ自動隠蔽 + ダッシュボードサマリ + 30 日後ストーリー演出 | 決勝のみ | C-1 / C-9 拡張 |

合計 **8 Unit**（MVP: 7 件 / 決勝: 1 件）

---

## 1. U-0 Foundation

| 項目 | 内容 |
|---|---|
| Intent | アプリ全体の土台インフラを 1 度立ち上げる。Cognito / DynamoDB / 共通 Observability を提供する |
| 主要成果物 | (a) CDK モノレポ初期化と共通 construct / (b) Cognito User Pool + App Client / (c) DynamoDB 単一テーブル + TTL 設定 / (d) CloudWatch Logs / Metrics の最低限セット |
| 受け入れ基準 | Cognito でサインアップ／サインイン可能、DynamoDB に PutItem / Query が通る、Lambda の標準ログが CloudWatch に流れる |
| 依存 | なし（最上流） |
| 関連 Story | 全 Story の前提（直接マップなし） |
| 想定工数 | 0.5〜1 人日 |
| MVP / 決勝 | MVP 必須 |
| 想定実装ディレクトリ | `cdk/lib/stacks/foundation-stack.ts` |

## 2. U-1 Identity & Google Tools

| 項目 | 内容 |
|---|---|
| Intent | アウトバウンド認証を AgentCore Identity に集約し、Calendar / Gmail を tool_use 用 Lambda Tool として実装する |
| 主要成果物 | (a) AgentCore Identity OAuth2 Credential Provider（Google）/ Workload Identity / (b) Calendar Tool Lambda（list_events / update_attendance）/ (c) Gmail Tool Lambda（list_recent / send） |
| 受け入れ基準 | テストユーザーで Calendar イベント一覧取得・参加ステータス更新が成功 / Gmail 送信が成功（モック宛先で OK） |
| 依存 | U-0 |
| 関連 Story | US-1.1（OAuth 連携）、US-2.3（代行実行のパス） |
| 想定工数 | 1〜1.5 人日（OAuth 申請の実時間を含めると伸びる可能性） |
| MVP / 決勝 | MVP 必須 |
| 想定実装ディレクトリ | `cdk/lib/stacks/identity-tools-stack.ts`、`lambdas/calendar_tool/`、`lambdas/gmail_tool/` |

## 3. U-2 AI Agent Core

| 項目 | 内容 |
|---|---|
| Intent | ユーザーの自然言語入力を解釈し、tool_use ループで Tool Lambda を呼びながら最終応答を返す中核エージェント |
| 主要成果物 | (a) AgentCore Runtime（コンテナ）/ (b) Strands Agent エントリポイント / (c) Bedrock Converse API tool_use ループ実装 / (d) Tool 定義（U-1 の Lambda を Tool として登録） |
| 受け入れ基準 | サンプル発話「明日の朝会、サボりたい」に対し、tool_use で Calendar Tool を呼んで該当イベントを特定し、構造化結果を返す |
| 依存 | U-0、U-1 |
| 関連 Story | US-1.3, US-2.1, US-2.2（音声/テキストどちらも） |
| 想定工数 | 2〜3 人日 |
| MVP / 決勝 | MVP 必須 |
| 想定実装ディレクトリ | `agent/`、`cdk/lib/stacks/ai-agent-stack.ts` |

## 4. U-3 Sabori Domain

| 項目 | 内容 |
|---|---|
| Intent | 「サボってよいリスト」の生成・取得・「サボる」状態化までの段階1機能を担う |
| 主要成果物 | (a) Sabori Logic Lambda（日次バッチ + オンデマンド再生成）/ (b) HTTP API: `GET /v1/sabori-items`, `POST /v1/sabori-items/{id}/mark` / (c) DynamoDB SaboriItem アクセスパターン実装 / (d) EventBridge Scheduler 設定 |
| 受け入れ基準 | カレンダー連携済みユーザーに対し、日次で SaboriItem が DynamoDB に書き込まれる / API でリスト取得・mark が動作する |
| 依存 | U-0、U-1、U-2 |
| 関連 Story | US-1.1, US-1.2, US-1.3 |
| 想定工数 | 1.5〜2 人日 |
| MVP / 決勝 | MVP 必須（段階1） |
| 想定実装ディレクトリ | `lambdas/sabori_logic/`、`cdk/lib/stacks/sabori-stack.ts` |

## 5. U-4 Autonomous Delegation Domain（自律代行）

| 項目 | 内容 |
|---|---|
| Intent | 段階 1 の核となる **自律代行フロー** を担う。サボリ判定 → 下書き生成 → 自律送信までを承認なしで一気通貫実行 |
| 主要成果物 | (a) Delegation Executor Lambda（U-1 の Tool Lambda を呼ぶ）/ (b) HTTP API: `GET /v1/delegation-logs`（事後ログ閲覧）, `PUT /v1/settings`（緊急停止スイッチ）/ (c) DynamoDB DelegationLog のアクセスパターン / (d) ホワイトリスト判定ロジック / (e) 緊急停止スイッチ（FR-1.9） |
| 受け入れ基準 | サボリ判定された項目に対し、ホワイトリスト範囲内なら **承認なしで** Tool 経由で代行送信 → ログ保存。範囲外は何もしない。緊急停止 ON で全実行が止まる |
| 依存 | U-0、U-1、U-2、U-3 |
| 関連 Story | US-1.2（自律実行）, US-1.3（ログ確認）, US-1.5（セーフガード） |
| 想定工数 | 2 人日 |
| MVP / 決勝 | MVP 必須（段階 1 の核） |
| 想定実装ディレクトリ | `lambdas/delegation_executor/`、`cdk/lib/stacks/delegation-stack.ts` |

> **設計変更（2026-05-09）**: 旧版の「下書き生成 → 承認 API → 実行」フローを廃止。承認 API（`/approve`, `/reject`）は削除、自律実行 + 事後ログ閲覧の構成に変更。

## 6. U-5 Voice Input Pipeline

| 項目 | 内容 |
|---|---|
| Intent | フロントから送られる音声を Transcribe Streaming で文字起こしし、AI Agent invoke へ転送する音声入力経路を構築 |
| 主要成果物 | (a) API Gateway WebSocket API / (b) Voice Stream Handler Lambda（接続・音声チャンク受信・Transcribe 連携・Agent invoke）/ (c) フロント側マイク取り込み（U-6 の一部と協調） |
| 受け入れ基準 | フロントのマイクボタンから「明日の朝会、サボりたい」と発話すると、文字起こし結果が U-2 の Agent に渡り、応答が返る |
| 依存 | U-0、U-2（Agent invoke）、U-6（フロント側） |
| 関連 Story | US-1.3, US-2.2 の音声経路 |
| 想定工数 | 1.5〜2 人日 |
| MVP / 決勝 | MVP 必須 |
| 想定実装ディレクトリ | `lambdas/voice_stream_handler/`、`cdk/lib/stacks/voice-stack.ts` |

## 7. U-6 Frontend

| 項目 | 内容 |
|---|---|
| Intent | ユーザーが触れる Next.js アプリ。サボリスト表示・テキスト入力欄・マイクボタン・下書きモーダル・ログ画面を提供 |
| 主要成果物 | (a) Next.js 15 アプリ（App Router）/ (b) Amplify Hosting + CI/CD / (c) Cognito サインイン UI（Amplify Auth クライアント SDK）/ (d) HTTP API クライアント（fetch ラッパ） / (e) WebSocket クライアント（音声送信） |
| 受け入れ基準 | サインイン → ホーム画面でサボリスト表示 → テキストまたは音声で「サボる」発話 → 下書きモーダル表示 → 承認 → 完了画面までが UI 上で動く |
| 依存 | U-0（Cognito）、U-3 / U-4（HTTP API）、U-5（WebSocket） |
| 関連 Story | US-1.1〜US-2.4 のフロント側全般 |
| 想定工数 | 2〜3 人日 |
| MVP / 決勝 | MVP 必須 |
| 想定実装ディレクトリ | `frontend/` |

## 8. U-7 Quiet Mode（戻れない境地、決勝向け）

| 項目 | 内容 |
|---|---|
| Intent | 段階 2「TODO リストなのに、見ない」を実現。代行ログを時間経過で自動隠蔽し、ダッシュボードのサマリだけを見せる |
| 主要成果物 | (a) ログ自動隠蔽ロジック（成功した代行は N 日経過でデフォルト非表示）/ (b) ダッシュボードサマリ画面 / (c) 30 日後ストーリー演出（デモ用ダミーデータ） |
| 受け入れ基準 | デフォルトでは過去 N 日内の最新ログのみ表示、サマリで「件数」だけ把握できる / 30 日デモシナリオが再生可能 |
| 依存 | U-4 が動作している前提 |
| 関連 Story | US-2.1, US-2.2 |
| 想定工数 | 1.5 人日 |
| MVP / 決勝 | **決勝のみ**（MVP では UX 演出のみ） |
| 想定実装ディレクトリ | `frontend/`（ダッシュボード追加）、`lambdas/delegation_executor/`（隠蔽ロジック追加） |

> **設計変更（2026-05-09）**: 旧 U-7「Autonomous Mode（自律実行モード）」は段階 1 に統合されたため不要。役割を「戻れない境地（Quiet Mode）」の UX 演出に振り直し。

---

## 9. 全体工数感（参考）

| Phase | 含む Unit | 合計工数感 |
|---|---|---|
| 5/10 までの書類審査向け | （成果物のみ、コードは書かない） | 既消化 |
| 5/10〜5/30（予選 MVP） | U-0〜U-6（7 Unit） | 約 11〜15 人日 |
| 5/30〜6/26（決勝拡張） | U-7 + 仕上げ | 約 3〜4 人日 |

> 工数感は粗い見積り。チーム体制が決まり次第、詳細見直し。

---

## 10. コード組織戦略（Greenfield）

```
makaseru/
├── README.md
├── cdk/                       # IaC: 各 Unit のスタックをここで定義
│   ├── bin/cdk.ts
│   ├── lib/
│   │   ├── stacks/
│   │   │   ├── foundation-stack.ts          # U-0
│   │   │   ├── identity-tools-stack.ts      # U-1
│   │   │   ├── ai-agent-stack.ts            # U-2
│   │   │   ├── sabori-stack.ts              # U-3
│   │   │   ├── delegation-stack.ts          # U-4 (+ U-7 拡張)
│   │   │   ├── voice-stack.ts               # U-5
│   │   │   └── api-stack.ts                 # HTTP API ルート集約
│   │   └── constructs/                      # 共通 L3
│   ├── package.json
│   └── tsconfig.json
├── frontend/                  # U-6: Next.js 15
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── package.json
├── agent/                     # U-2 の Strands Agent (Python)
│   ├── pyproject.toml
│   └── src/agent/
├── lambdas/                   # 各 Unit の Python Lambda
│   ├── calendar_tool/         # U-1
│   ├── gmail_tool/            # U-1
│   ├── sabori_logic/          # U-3
│   ├── delegation_executor/   # U-4 (+ U-7)
│   ├── voice_stream_handler/  # U-5
│   └── shared/                # 共通: DynamoDB アクセス、ロガー、型
├── aidlc-docs/                # AI-DLC 成果物（既存）
└── pre-ai-dlc/                # 事前ブレスト（既存）
```

### 命名規則

| 対象 | 規則 |
|---|---|
| CDK Stack | `<DomainName>Stack` (例: `FoundationStack`) |
| Lambda 関数名 | `makaseru-<unit>-<purpose>` (例: `makaseru-tool-calendar`) |
| HTTP API パス | `/v1/<resource>[/{id}/<action>]` |
| DynamoDB Table | 1 テーブルのみ: `MakaseruTable` |
| 環境変数 | UPPER_SNAKE_CASE |

### デプロイ単位

- **CDK Stack 単位でデプロイ**。Unit ↔ Stack はほぼ 1:1
- 共通の HTTP API は `api-stack.ts` で集約（Stack 間で Resolver 関数 ARN を参照）
- `cdk deploy --all` で全 Unit を一括デプロイ可能、特定 Unit のみは `cdk deploy <StackName>` で個別

---

## 11. Unit ↔ ADR の対応

| Unit | 関連 ADR | ハイライト |
|---|---|---|
| U-1 | ADR-1' | AgentCore Identity を採用、Gateway は不採用 |
| U-2 | ADR-1', ADR-5 | AgentCore Runtime + tool_use ループ + Sonnet 4.6 |
| U-3, U-4, U-6 | ADR-6 | API Gateway HTTP API + Lambda（AppSync 不採用） |
| U-5 | ADR-5 | Amazon Transcribe Streaming |
| U-6 | ADR-4 | Next.js + Amplify Hosting（軽め共存） |
| 全 Unit | ADR-2 | ap-northeast-1 単一リージョン |

---

## 12. 残論点（Construction で詰める）

- 各 Unit の Functional Design（業務ロジック詳細）
- NFR 詳細（コスト・レイテンシ目標）
- Cognito 認証フローのセッション管理
- 自律実行モードのしきい値ロジック
- HTTP API の OpenAPI 仕様化（型共有手段）
