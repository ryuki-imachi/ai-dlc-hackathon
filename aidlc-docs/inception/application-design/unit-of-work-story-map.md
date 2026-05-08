# Unit of Work — Story Map

> **AI-DLC Inception Phase / Units Generation 成果物**
>
> [stories.md](../user-stories/stories.md) の各ストーリーを、どの Unit が実現するかをマッピングする。
>
> ### 設計変更履歴
> - **2026-05-09**: 完全自律設計に変更したことに伴い、ストーリー数が 9 → 7 に減少。U-4 の役割を「自律代行」に変更、U-7 を「Quiet Mode（戻れない境地）」に振り直し。

---

## 1. ストーリー → Unit マッピング

| Story ID | タイトル | Stage | Scope | 主担当 Unit | 補助 Unit |
|---|---|---|---|---|---|
| US-1.1 | Google カレンダー / Gmail 連携 | 1 | MVP | **U-1** Identity & Tools | U-6（連携 UI） |
| US-1.2 | AI による自律的なサボリ判定と代行実行（核） | 1 | MVP | **U-4** Autonomous Delegation Domain | U-2（AI Agent）, U-1（Tool）, U-3（Sabori 判定） |
| US-1.3 | 代行ログの事後確認 | 1 | MVP | **U-4** Autonomous Delegation Domain | U-6（ログ画面） |
| US-1.4 | 音声 / テキストでの追加指示（任意） | 1 | MVP | **U-2** AI Agent Core | U-5（音声経路）, U-6（テキスト入力欄） |
| US-1.5 | 暴走防止セーフガードと緊急停止 | 1 | MVP | **U-4** Autonomous Delegation Domain | U-6（緊急停止 UI） |
| US-2.1 | ログ表示の自動隠蔽 | 2 | 決勝 | **U-7** Quiet Mode | U-6（ダッシュボード） |
| US-2.2 | 30 日後の利用者ストーリー演出（デモ用） | 2 | 決勝 | **U-7** Quiet Mode | U-6（演出 UI） |

---

## 2. Unit ごとのストーリー一覧

### U-0 Foundation
- 直接マップなし（全 Story の前提）

### U-1 Identity & Google Tools
- **US-1.1** Google カレンダー / Gmail 連携（主担当）
- US-1.2 の Tool 実体（Calendar / Gmail Tool Lambda）

### U-2 AI Agent Core
- **US-1.4** 音声 / テキストでの追加指示（主担当: 意図解釈）
- US-1.2 の AI 推論部分（サボリ判定、3 タイプの代行アクション選択、下書き生成）

### U-3 Sabori Domain
- US-1.2 のサボリ判定ロジック（カレンダー / Gmail からの抽出、後押しメッセージ生成）

### U-4 Autonomous Delegation Domain
- **US-1.2** AI 自律実行の中核（主担当）
- **US-1.3** 代行ログの事後確認（主担当）
- **US-1.5** セーフガード + 緊急停止（主担当）

### U-5 Voice Input Pipeline
- US-1.4 の音声経路（補助）

### U-6 Frontend
- 全 MVP Story の UI 実装（連携 UI / ログ画面 / 緊急停止スイッチ / 入力欄）

### U-7 Quiet Mode（戻れない境地）
- **US-2.1** ログ表示の自動隠蔽（主担当）
- **US-2.2** 30 日後の利用者ストーリー演出（主担当）

---

## 3. MVP / 決勝のスコープ確認

### MVP（5/30 予選向け）に含まれる Unit と Story

```
Unit:  U-0 Foundation
       U-1 Identity & Tools
       U-2 AI Agent Core
       U-3 Sabori Domain
       U-4 Autonomous Delegation Domain
       U-5 Voice Input Pipeline
       U-6 Frontend

Story: US-1.1, US-1.2, US-1.3, US-1.4, US-1.5
```

すべての MVP Story が Unit に割り当て済み。**カバレッジ 100%**。

### 決勝（6/26 向け）の追加

```
Unit:  U-7 Quiet Mode

Story: US-2.1, US-2.2
```

---

## 4. INVEST チェック（Story と Unit の対応）

| Criterion | チェック内容 | 状態 |
|---|---|---|
| **I**ndependent | 各 Story が独立した Acceptance Criteria を持つ | ✅ |
| **N**egotiable | Unit 内で詳細を Functional Design に持ち越せる | ✅ |
| **V**aluable | 全 MVP Story が「最初から自律」の価値（または安心感）に対応 | ✅ |
| **E**stimable | Unit ごとに人日見積りを `unit-of-work.md` に記載済み | ✅ |
| **S**mall | 各 Story は 1〜3 日の Unit 内タスクに収まる | ✅ |
| **T**estable | Acceptance Criteria が Given-When-Then で記述済み | ✅ |

---

## 5. ストーリー実装の優先順位（クリティカルパス）

予選デモ（5/30）に間に合わせる場合の Story 完成順:

```
Phase A（基盤通電）: 5/15 まで
  US-1.1（OAuth 連携）→ US-1.2 の前半（サボリ判定のみ）

Phase B（自律実行 + 安全装置）: 5/22 まで
  US-1.2 の後半（自律送信、モック宛先）→ US-1.5（セーフガード + 緊急停止）

Phase C（仕上げ）: 5/29 まで
  US-1.3（ログ画面）→ US-1.4（任意の音声/テキスト追加指示）→ デモシナリオ準備 → リハーサル
```

各フェーズの完了 = 該当 Unit の受け入れ基準達成。

> **重要**: US-1.5（セーフガード）は US-1.2（自律実行）と **同時に**実装する必要がある。自律実行を先に作るとモック宛先化前にデモ環境で誤送信するリスク。

---

## 6. 残論点

- 体制確定後に Unit ↔ 担当者のマッピングを `unit-of-work.md` §9 に追加
- US-2.x（決勝向け）の演出デザインは決勝 1〜2 週前に詳細化
- 自律実行のホワイトリスト具体定義は Functional Design で詰める
