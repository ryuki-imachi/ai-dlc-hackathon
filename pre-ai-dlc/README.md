# pre-ai-dlc

AWS AI-DLC ハッカソン 2026 提案 **「サボリスト」** の AI-DLC 着手前資料一式。
このディレクトリ全体を AI-DLC のインプットとして渡すことを想定している。

## 読む順序

時系列で番号を付けてある。上から順に読めば、思考の流れが追える。

| # | ファイル | 役割 |
|---|---|---|
| 1 | [`01-memo.md`](./01-memo.md) | 初回ブレスト（個人）。アイデアの発散記録 |
| 2 | [`02-concept.md`](./02-concept.md) | 初回コンセプト整理。memo から方向性を絞り込んだメモ |
| 3 | [`03-proposal.md`](./03-proposal.md) | ハッカソン提案書（**この時点での仮版**） |
| 4 | [`04-brainstorm/`](./04-brainstorm/) | proposal を叩き台にした全員ブレスト。提案書への追加・修正材料 |

## ディレクトリ構成

```
pre-ai-dlc/
├── README.md           ← このファイル
├── 01-memo.md
├── 02-concept.md
├── 03-proposal.md
└── 04-brainstorm/
    ├── personas/       ← 7観点のペルソナ視点ブレスト
    │   ├── 01-engineer-not-ai.md
    │   ├── 02-ai-engineer.md
    │   ├── 03-strategy-consultant.md
    │   ├── 04-business-developer.md
    │   ├── 05-ux-designer.md
    │   ├── 06-pr-marketer.md
    │   └── 07-legal-ethics.md
    └── team/           ← チームメンバー個別の意見
        ├── umitsu.md
        ├── yagi.md
        └── yakumo.md
```

## 注意事項

- `03-proposal.md` は `04-brainstorm/` の前に書かれた**仮版**。最新の論点・反対意見・追加アイデアは `04-brainstorm/` 側にある。両方を統合した最終版はまだ存在しない。
- `04-brainstorm/personas/` は AI による多面評価、`04-brainstorm/team/` は実在のチームメンバーによる意見。重みづけが必要なら区別すること。

## AI-DLC でやってほしいこと

この一式を踏まえて、AI-DLC の **Intent ドキュメント** を起こしたい。
`03-proposal.md` をベースに、`04-brainstorm/` のフィードバックを反映した上で、AI-DLC の Unit 分解まで進めるのがゴール。
