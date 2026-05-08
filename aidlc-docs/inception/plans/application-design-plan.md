# Application Design Plan — マカセル

> **AI-DLC Inception Phase / Application Design — Planning 成果物**
> **Depth**: Standard。5/10 締切に間に合わせるため、追加質問は事前に AskUserQuestion で実施済み。

## 確定済みの技術選定（事前確定）

| 項目 | 値 | 確定経路 |
|---|---|---|
| AgentCore 採用範囲 | Runtime + Memory + Gateway + Identity フル | 追加質問 / 2026-05-08 |
| 音声入力 STT | Amazon Transcribe Streaming | 追加質問 |
| フロントエンド | Next.js 15 + Amplify Hosting | 追加質問 |
| LLM モデル | Bedrock Claude Sonnet 4.6 | requirements.md / レビュー v1 |
| リージョン | ap-northeast-1（東京） | requirements.md Q-D3 |
| データストア | DynamoDB / 30 日 TTL / PITR OFF | requirements.md Q-D2 |
| 連携先 | Google Calendar（必須）+ Gmail（次点）+ Slack（任意） | requirements.md Q-C3 |
| IaC | **AWS CDK（TypeScript）主導 + Amplify Hosting 軽め共存**。バックエンドは CDK、Hosting / CI-CD / Auth クライアント SDK は Amplify を活用 | ADR-6 / 2026-05-08 |
| API Layer | **API Gateway HTTP API + Lambda**（AppSync は不採用） | ADR-6 / 2026-05-08 |

## 採用しないもの（Out of Scope）

- 通知系（プッシュ・音声）
- SNS / タイムライン / ランキング / 退会防止 UX
- 端末ネイティブ通知制御
- Agentic Skill 蓄積機能（補足扱い、決勝段階で実装余力次第）

## Generation Steps

- [x] components.md — 主要コンポーネントと責務を定義
- [x] component-methods.md — 主要メソッドの I/F とシグネチャ（業務ロジック詳細は Functional Design へ）
- [x] services.md — サービス層の orchestration パターン
- [x] component-dependency.md — 依存関係マトリクス + データフロー図
- [x] application-design.md — 上記を統合した 1 枚版（書類審査の主要提出物）

## 設計方針

- **C4 モデル**: Context + Container レベルまで描く（Component レベルは component-methods.md で別管理）
- **AWS 構成図**: ASCII テキストで記述、後日 drawio 化検討
- **粒度**: 「アイデアと技術のバランス」をハッカソン審査員が読み取れる粒度。深掘りしすぎない
