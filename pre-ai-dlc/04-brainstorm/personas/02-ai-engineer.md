# 如月凛（生成AIエンジニア・29歳）の頭の中

> 「サボリスト」、AgentCoreの構成図にめちゃくちゃ綺麗に落ちる。やばい、これ作りたい。

---

- 提案書ざっと読んだ第一声、これ **AgentCore Runtime + Memory + Gateway + Identity** の練習問題かってくらいハマるテーマ。
- Intentが1行で言えるのが偉い。「怠惰の合理化を、SNSの承認欲求で駆動する」。Inceptionの出発点としてこれ以上ない。
- 「人をダメにする」って枠で振り切ってるのに、内部は **マルチエージェント × イベント駆動 × ストリーミングSNS** で、技術的には大真面目に作れる構造になってる。むしろ真面目に作るほど面白くなるやつ。
- 6 Unit 全部にエージェントが要るわけじゃない、ってのを最初に決めたい。Unit 3/6 は素のサーバレスで十分、Unit 2/5 だけがエージェント本体。これだけでコスト最適化と運用優秀性のスコア上がる。
- 引っかかるとこ3つ：①通知ミュート Unit が iOS/Android ネイティブまで踏み込むと沼る、②自動返信代行が Slack/Gmail/LINE 3面待ちは攻めすぎ（提案書通り Slack 1本でいい）、③AI称賛の品質がデモ全体の体感を支配する → ここを技術ハイライトにしたい。
- リージョンは us-west-2 主、決勝デモ用に ap-northeast-1 にエッジ（CloudFront + Amplify Hosting）。**Bedrock の Sonnet 4.6 / Haiku 4.5 が us-west-2 で安定して使える**のが理由。

## サービス割り（ざっくり頭の中で当てた）

- **Unit 1 通知ミュート**：MVPは Slack の `users.setPresence` + Workflow の自動DND で十分。決勝で PWA Notification API + Service Worker 抑制。AgentCore Gateway 越しに Slack API 叩く、Identity の OAuth2 プロバイダで 3LO トークン管理。
- **Unit 2 自動返信代行**：AgentCore Runtime に Strands Agent。インバウンドは **Slack Events API → API Gateway → SQS → EventBridge Pipes → Runtime invoke**。返信生成は Sonnet 4.6（`us.anthropic.claude-sonnet-4-5-20250929-v1:0`）、軽い分類は Haiku 4.5（`us.anthropic.claude-haiku-4-5-20251001-v1:0`）でコスト最適化。
- **Unit 3 サボリ計測**：DynamoDB 単一テーブル、`PK=USER#<id> / SK=EVENT#<ts>`。連続記録は ConditionExpression で原子的に increment。日次集計は EventBridge Scheduler → Lambda。エージェント要らん。
- **Unit 4 タイムライン**：AppSync Subscriptions + DynamoDB Streams で Fan-out。WebSocket API Gateway でも組めるけど、Cognito 認証統合まで含めると AppSync の方が早い。
- **Unit 5 AI 称賛**：見せ場。AgentCore Memory に **STM（直近のサボり投稿）** と **LTM（過去の言い訳パターン）** を分けて、Sonnet 4.6 で「あなたらしい褒め」生成。Polly Generative の `Takumi-Generative`、決勝までに Nova Sonic 入れたら最高。
- **Unit 6 退会防止UX**：これAIじゃない、フロント側のダークパターン。React + Tailwind で深く入れ子の退会フロー。最後の引き留めセリフだけ Haiku 4.5 で十分。

## 落とし穴・気になる（22日間で踏みやすい地雷）

- **Slack の3秒制約**：Slack Events API は3秒以内に200返さないとリトライしてくる。Sonnet 4.6 シンクロで叩いたら普通に超える。**最初から非同期前提（API GW → SQS → Lambda → Runtime invoke）**で組まないと予選で詰む。シンクロ前提の落とし穴、絶対にハマるやつ。
- **AgentCore Runtime のコールドスタート**：コンテナベースで初回 invoke 数秒級。デモで5秒沈黙はウケない。**EventBridge Scheduler `rate(5 minutes)` でウォームアップ叩く**運用が現実解。月数千円〜のオーダー。
- **Bedrock のクォータ**：Sonnet 4.6 系の TPM/RPM は新規アカウントで絞られてる。**5/10 までに Service Quotas 引き上げ申請**。これサボると予選デモで ThrottlingException 出る（過去案件でやらかした）。
- **AgentCore Memory の strategy 選定**：`summarization` / `semantic` / `user_preference` 全部入れると抽出 Job の課金とレイテンシで死ぬ。MVPは `summarization` 1本でいい。LTM Semantic は決勝で投入。
- **タイムラインの Fan-out コスト**：ダミーユーザー大量投入すると AppSync Subscription 接続課金が地味に効く。**MVPは10ユーザー上限**。
- **Polly Generative のレイテンシ**：`Takumi-Generative` は標準より遅い（数百msオーダー）。Nova Sonic 入れるなら WebSocket + PCM 16kHz、決勝までスコープ外推奨。
- **「サボらせ品質」の評価**：LLM-as-judge で測るとして、ルーブリックどう書く？ Bedrock Evaluations の Custom Judge をここで使いたい。決勝のドキュメント審査で数値見せたら一気に印象上がる。
- **倫理ライン**：上司に体調不良を AI が勝手に送るは業務トラブル直結。**全てモック/サンドボックス Slack Workspace内で完結**って線をドキュメントで明示しないと書類審査で減点される可能性。Bedrock Guardrails の DENIED_TOPICS で「実在の上司を匂わせる返信」を弾く。
- **コスト爆発**：Sonnet 4.6 を毎投稿で呼ぶと、ユーザー数 × 投稿頻度で簡単に月数万円。**ルーター層を Haiku 4.5、Sonnet は MVP 選定と日次称賛だけ**で1桁下げられる。

## アーキ素描

```
                        ┌─────────────────────────────┐
   [User: PWA/React] ── │ CloudFront + Amplify Hosting│
                        └──────────────┬──────────────┘
                                       │ AppSync (GraphQL + Subscriptions)
                                       ▼
                        ┌──────────────────────────────┐
                        │  AppSync API (Cognito 認証)   │
                        └────┬───────────────┬─────────┘
                  Mutations  │               │ Subscriptions
                             ▼               │
                        ┌─────────┐          │
                        │DynamoDB │──Streams─┘
                        │(Events) │
                        └────┬────┘
                             │ Stream → Pipes
                             ▼
                  ┌──────────────────────┐
                  │ EventBridge Pipes    │
                  └──┬──────────────┬────┘
                     ▼              ▼
        ┌──────────────────┐  ┌─────────────────────┐
        │ Lambda (集計/TTL)│  │ AgentCore Runtime    │
        │  Unit 3, Unit 6  │  │ (Strands Multi-Agent)│
        └──────────────────┘  │  - 返信エージェント  │
                              │  - 称賛エージェント  │
                              │  - MVP選定エージェント│
                              └────┬─────────────┬───┘
                                   ▼             ▼
                         ┌──────────────────┐ ┌────────────────────┐
                         │ AgentCore Gateway│ │ AgentCore Memory   │
                         │ + Identity(OAuth)│ │ (STM + Semantic LTM)│
                         └────────┬─────────┘ └────────────────────┘
                                  ▼
                         ┌──────────────────┐ ┌────────────────────┐
                         │ Bedrock          │ │ Polly Generative   │
                         │ Sonnet 4.6 / 4.5 │ │ (Takumi-Generative)│
                         │ Haiku 4.5        │ └────────────────────┘
                         └──────────────────┘
```

選定根拠ざっくり：

| サービス | なぜ | 代替を却下した理由 |
|---|---|---|
| AgentCore Runtime | マルチエージェント協調、JWT/SigV4、コンテナで Python ライブラリ自由 | Lambda 直叩き：状態管理が辛い／ECS：オーバーキル |
| AgentCore Memory | Strategy で STM/LTM 分離、抽出 Job 自動 | DynamoDB+自前RAG：時間ない／OpenSearch Serverless：運用負荷 |
| AgentCore Gateway | MCP化された Slack/Gmail を Identity 経由で安全に呼べる | Lambda+Secrets Manager：3LO の更新で詰む |
| AppSync | Subscriptions でタイムラインがほぼ無実装 | API GW WebSocket：認証統合が手作業 |
| DynamoDB | 単一テーブル + Streams でイベント駆動が綺麗 | Aurora Serverless v2：オーバースペック |
| Polly Generative | 「優しく褒める」声質が秀逸 | Nova Sonic：決勝の見せ場候補だが MVP では時間ない |
| CDK Python | チーム既存知見、変更を1コマンド | SAM：マルチスタックの依存解決が弱い |

MVP 規模（10ユーザー × 1日10投稿）で **月 5,000〜10,000 円**くらい。ハッカソンクレジットで余裕。

## Unit 切り直し提案

- **Unit 0「基盤・観測」を追加**：CloudWatch Logs / OTel トレース / Cognito / CDK 共通基盤を Unit 0 として依存グラフの根に置く。「観測性を最初に作る」って書けるの、AI-DLC 的に美しい。
- **Unit 2 を 2-A（受信解析・Haiku）と 2-B（返信生成・Sonnet）に分ける**：分類と生成を分けるのは Bedrock 系プロダクトの定石。評価とコスト管理の両面で意味出る。
- **Unit 5 を 5-A（称賛生成・イベント駆動）と 5-B（MVP選定・日次バッチ `cron(0 22 * * ? *)`）に分ける**：生成タイミング違うものを同 Unit にすると非機能要件が分裂する。地味に効く論点。

依存：

```
Unit 0 (基盤・観測)
   ├── Unit 1 (通知ミュート)
   ├── Unit 2-A (受信解析) ── Unit 2-B (返信生成)
   ├── Unit 3 (サボリ計測) ─┬── Unit 4 (タイムライン)
   │                        └── Unit 5-B (MVP選定 / 日次)
   └── Unit 5-A (称賛生成 / イベント駆動) ── Unit 6 (退会防止)
```

MVP（5/30）：Unit 0, 2-A, 2-B, 3, 4, 5-A。決勝（6/26）追加：Unit 1, 5-B, 6 + AgentCore Memory 強化。

## Inception で議論したい技術論点

1. AgentCore Runtime のベースイメージ、`python:3.12-slim` + Strands Agents 1.x でいい？イメージサイズ vs コールドスタートのトレードオフは？
2. 返信エージェントのモデル、Sonnet 4.6 固定 or トーン別に Haiku 4.5 / Sonnet 4.6 ルーティング？
3. AgentCore Memory の strategy、MVP は `summarization` 1本、決勝で `semantic` 追加でいい？抽出 Job のレイテンシどれくらい見る？
4. データモデル、Event テーブルと Post テーブル分けるか単一テーブルか。GSI 設計は？
5. リアルタイム配信、AppSync Subscriptions vs API GW WebSocket、チームのスキル分布的にどっち？
6. プライバシー、Slack メッセージを Bedrock に投げる前のマスキング、Comprehend PII か Bedrock Guardrails か？
7. 評価設計、「サボらせ品質」を5観点×5段階のルーブリックで書ける？LLM-as-judge は Sonnet で回す？Nova Pro でクロスチェック？
8. コストガバナンス、Bedrock 月次予算アラームを CloudWatch Alarm + SNS で 5,000 / 10,000 / 30,000 円の3段階で組む？
9. AgentCore Identity の Slack 3LO トークン refresh の運用と、失効時のフォールバック設計。
10. 観測性、OTel トレースを CloudWatch Application Signals に送るか X-Ray Native か。マルチエージェント協調のスパンをどう可視化する？
11. Sandbox Workspace 構築、ダミーユーザー10人仕込みは何日で終わる？
12. Nova Sonic の双方向音声を決勝で入れるか、Polly Generative で十分か。
13. CDK スタック分割、Network / Auth / Data / Agent / Frontend の5本でいい？
14. CI/CD、`cdk deploy` を main マージで自動化するか、決勝直前は手動デプロイで安全運用するか。
15. DynamoDB PITR、ハッカソンならコスト削減で OFF も妥当か？

---

審査観点のメモだけ最後に短く。Intent は1行で通る。Unit 分解は Unit 0 追加と 2/5 分割で AWS SA 層が「分かってる」と思う粒度になる。決勝で勝ちに行くなら **(a) マルチエージェント協調 (b) AgentCore Memory の Long-term Semantic で「あなたの過去の言い訳」RAG (c) LLM-as-judge による「サボらせ品質」評価**、この3点を技術ハイライトに据える。**過剰技術 ≠ 高評価**、賢く作りすぎず、シンプルに作って高速に検証。やりましょう、これは行ける。
