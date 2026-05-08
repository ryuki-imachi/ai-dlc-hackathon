# 🔖 セッション中断メモ — 2026-05-09 中断時点

> このファイルは AI-DLC セッションを再開する際の引き継ぎメモ。
> Claude Code を新しいセッションで起動したら、まずこのファイルを読むこと。

---

## 中断ポジション

- **中断日時**: 2026-05-09
- **現在の Stage**: **INCEPTION フェーズ完了直前 — Units Generation の承認待ち**
- **Stage の状態**: 必須成果物はすべて生成済み、ユーザー承認のみ未取得
- **次のアクション候補**:
  1. Units Generation の最終承認を取得して INCEPTION 完了マーク
  2. 応募文（800 字）ドラフト作成
  3. Git リポジトリ Public 化と応募フォーム送信（**締切: 5/10**）
  4. Construction フェーズ開始（5/10 以降）

---

## ⏰ 直近マイルストーン

| 期日 | やること |
|---|---|
| **2026-05-10** | **応募締切**。Inception フェーズ完了 + 応募文 + Git Public + 応募フォーム送信 |
| 2026-05-12 正午 | 運営宛メールでリポジトリ URL 連絡 |
| 2026-05-15 | 書類審査結果通知 |
| 2026-05-30 | 予選会＠麻布台ヒルズ（MVP デモ） |
| 2026-06-26 | 決勝＠AWS Summit Japan Day2 |

**残り時間（5/10 締切まで）**: 約 1 日。応募文だけは最低限必要。

---

## プロダクト情報（最新）

| 項目 | 確定値 |
|---|---|
| プロダクト名 | **マカセル** |
| Intent コピー | TBD（応募文作成時に確定。候補は [requirements.md §1.1](inception/requirements/requirements.md) に保管） |
| 路線 | 八木案ベース「サボってよいリスト AI 秘書」 |
| メインペルソナ | マジメ・完璧主義タイプ（28〜32 歳、自分でやりたい意欲強、効率化志向）。深掘りしない方針 |
| MVP 体験範囲 | 段階 1〜2 を実装、段階 3 の境地は決勝で表現 |
| 入力 IF | 音声（メイン） + テキスト（サブ） |
| 連携 | Google カレンダー必須 / Gmail 次点 / Slack 任意 |
| リージョン | ap-northeast-1 一本 |
| データ | DynamoDB / 30 日 TTL / PITR OFF |
| 成功指標 | 決勝優勝（6/26） |
| Optional 成果物 | やらない |

## 技術スタック（確定）

- **フロント**: Next.js 15 + Amplify Hosting + Tailwind
- **認証**: Cognito User Pool（CDK 構築） + Amplify Auth クライアント SDK
- **API**: API Gateway HTTP API + Cognito Authorizer + Lambda（**AppSync は不採用** — ADR-6）
- **音声**: API Gateway WebSocket + Amazon Transcribe Streaming
- **AI**:
  - Bedrock Claude Sonnet 4.6（Converse API + tool_use ループ）
  - **AgentCore Runtime**（Strands Agent ホスティング、採用）
  - **AgentCore Identity**（Google OAuth トークン管理、採用）
  - AgentCore Memory / Gateway は **不採用**（ADR-1' 改訂版で方針変更）
- **データ**: DynamoDB 単一テーブル（東京）
- **観測性**: CloudWatch + X-Ray（最低限）
- **IaC**: AWS CDK（TypeScript）主導 + Amplify Hosting 軽め共存

## ADR 一覧

| ADR | 内容 |
|---|---|
| ADR-1（廃案） | AgentCore フル採用 — 過剰実装と判断、廃止 |
| ADR-1' | AgentCore Runtime + Identity のみ採用、Memory / Gateway は tool_use + Lambda Tool で代替 |
| ADR-2 | ap-northeast-1 単一リージョン |
| ADR-3 | 通知・音声出力なし（依存を促さないため） |
| ADR-4 | Next.js + Amplify Hosting |
| ADR-5 | Amazon Transcribe Streaming |
| ADR-6 | API Layer は AppSync → API Gateway HTTP API + Lambda（Amplify Gen2 の Python Lambda 摩擦回避） |

詳細: [application-design.md §8](inception/application-design/application-design.md)

---

## 生成済み成果物（書類審査必須）

| 成果物 | パス | 状態 |
|---|---|---|
| Workspace Detection | [aidlc-state.md](aidlc-state.md) | ✅ |
| Requirements Analysis | [requirements.md](inception/requirements/requirements.md) v3 | ✅ |
| User Stories | [stories.md](inception/user-stories/stories.md) / [personas.md](inception/user-stories/personas.md) | ✅ |
| Workflow Planning | [execution-plan.md](inception/plans/execution-plan.md) | ✅ |
| Application Design | [application-design.md](inception/application-design/application-design.md) ほか 4 件 | ✅ |
| Units Generation | [unit-of-work.md](inception/application-design/unit-of-work.md) ほか 2 件 | ✅（承認待ち） |
| Audit Log | [audit.md](audit.md) | ✅（最新まで反映） |

### Unit 一覧（要約）

| Unit | 名前 | Scope | 工数感 |
|---|---|---|---|
| U-0 | Foundation | MVP | 0.5〜1 人日 |
| U-1 | Identity & Google Tools | MVP | 1〜1.5 人日 |
| U-2 | AI Agent Core | MVP | 2〜3 人日 |
| U-3 | Sabori Domain | MVP | 1.5〜2 人日 |
| U-4 | Delegation Domain | MVP | 2 人日 |
| U-5 | Voice Input Pipeline | MVP | 1.5〜2 人日 |
| U-6 | Frontend | MVP | 2〜3 人日 |
| U-7 | Autonomous Mode | 決勝 | 1.5 人日 |

合計: MVP 約 11〜15 人日 + 決勝拡張 約 3〜4 人日

---

## 残課題リスト

### 🔴 5/10 までに対応必須
- [ ] **タレカツさんの追加レビュー意見を反映**（[`pre-ai-dlc/04-brainstorm/team/after.md`](../pre-ai-dlc/04-brainstorm/team/after.md) — Inception 後に出た意見、本文は下記参照）
- [ ] Units Generation の最終承認（実質形式的、上記を確認）
- [ ] aidlc-state.md / audit.md の Inception 完了マーク
- [ ] **応募文（800 字）作成**（提案書 03-proposal.md は古いので、requirements.md ベースで書き直し）
- [ ] Intent コピー 1 行を最終決定（候補は requirements.md §1.1）
- [ ] Git リポジトリ Public 化
- [ ] 応募フォーム送信（5/10 締切）

#### 📝 タレカツさんの追加レビュー（after.md より、4 論点）

1. **当人にサボる意識がない問題**: AI に任せただけで「サボった自覚」が薄く、周囲から見ると単なるブッチ。サボった分のフォロー機能 / サボリ自覚化 UI を要検討
2. **押し付け合いの連鎖問題**: AI が「言いくるめ」で他者に押し付ける → 相手側も同じ AI を使い始めると、押し付け合いの連鎖が起こる。複数ユーザー世界での倫理ライン
3. **サボれることを示されてもサボれない問題**: マジメ・完璧主義タイプには「サボっていい」と提示するだけでは罪悪感で動けない。AI が積極的に背中を押す「後押し」が必要
4. **「AI → API → 人」の構図**（新規）: 表面はユーザーが AI に頼んでいるが、実態は AI が API 経由で他者を間接的に動かしている。**「人をダメにする」テーマの深い解釈**として、プレゼン上のキラーフレーズになり得る

→ 設計への影響:
- requirements.md §1.2「内部 Intent」に **D（AI→API→人）の構図** を追記検討
- stories.md US-2.4 を **「サボリ自覚化ログ」** に格上げ検討（A 対策）
- 段階 2 の代行下書き UI に **「後押しメッセージ」** を組み込む検討（C 対策）
- 倫理ドキュメント（Construction フェーズ）に **「押し付け合いの連鎖」** リスクを追加（B 対策）
- 決勝デモのキメ画候補に「マカセルの裏側で起きていること」演出を追加（D 活用）

要否は再開時に判断する。**特に D はプレゼン上の強力ネタなので、応募文 800 字に反映する価値あり**。

### 🟡 5/10〜5/15 で対応
- [ ] チーム体制確定（Q-F1 = D 未定のまま）
- [ ] Google OAuth 申請
- [ ] CDK プロジェクト初期化（U-0 Foundation スタックから着手）
- [ ] 「マカセル」の商標調査・ドメイン仮押さえ

### 🟢 Construction フェーズ移行後
- [ ] U-0 → U-1 → U-2 → U-3/U-5 並列 → U-4 → U-6 統合の順で実装
- [ ] Functional Design / NFR Requirements / NFR Design / Infrastructure Design は per-unit で軽量実施
- [ ] Bedrock Service Quotas 引上げ申請（必要に応じて）
- [ ] プロンプトインジェクション対策（NFR フェーズで判断）

### 🔵 決勝（6/26）に向けて
- [ ] U-7 Autonomous Mode 実装
- [ ] 30 日後ストーリーの演出データ準備
- [ ] デモシナリオの 3 分版・10 分版作成

---

## 再開時の最初のアクション

1. **このファイルを読む**（再開コンテキスト復元）
2. [audit.md](audit.md) の最終エントリを確認（時系列を把握）
3. [aidlc-state.md](aidlc-state.md) で Stage Progress を確認
4. ユーザーに **「応募文ドラフトから始めますか？それとも別の作業からですか？」** を確認
5. 残り時間（5/10 までの残時間）に応じて優先順位を再確認

---

## 議論のスタンス（中断時点で確立した方針）

ユーザーとの設計議論で確立されたスタンス。再開後も尊重する:

- **過剰実装は避ける**: AgentCore は Runtime + Identity のみ、tool_use + Lambda Tool で済ませる方針（ADR-1'）
- **シンプル優先**: AppSync → HTTP API のように、複雑性を持ち込まない選択を取る（ADR-6）
- **ペルソナ深掘りしない**: マジメ・完璧主義タイプで固定、それ以上の人物像作り込みはしない（レビュー v2）
- **Optional 成果物は省く**: 必須成果物だけで勝負（Q-F2）
- **デモ表現は具体要件**: 「天才と思わせる」のような抽象表現は避け、NFR-DEMO-1〜4 の具体要件で書く（レビュー v1）
- **音声出力・通知はしない**: アプリ依存を促さない方針（要件）
- **テキスト入力もサポート**: 音声がメインだが、テキスト入力もサブとして用意（レビュー v3）

---

## ファイルマップ

```
aidlc-docs/
├── RESUMPTION-NOTE.md            # ★ このファイル（再開時に最初に読む）
├── aidlc-state.md                # 全体ステータス
├── audit.md                      # 監査ログ（時系列）
└── inception/
    ├── requirements/
    │   ├── requirements.md                       # 要件 v3
    │   ├── requirement-verification-questions.md # 質問（回答済）
    │   ├── requirements-review-v1.md             # レビュー v1（ユーザー記入）
    │   └── requirements-review-v2.md             # レビュー v2（ユーザー記入）
    ├── user-stories/
    │   ├── personas.md
    │   └── stories.md
    ├── application-design/
    │   ├── application-design.md                 # ★ 統合版
    │   ├── components.md
    │   ├── component-methods.md
    │   ├── services.md
    │   ├── component-dependency.md
    │   ├── unit-of-work.md                       # ★ Unit 定義
    │   ├── unit-of-work-dependency.md
    │   └── unit-of-work-story-map.md
    └── plans/
        ├── execution-plan.md                     # Workflow Planning
        ├── user-stories-assessment.md
        ├── story-generation-plan.md
        ├── application-design-plan.md
        └── unit-of-work-plan.md
```
