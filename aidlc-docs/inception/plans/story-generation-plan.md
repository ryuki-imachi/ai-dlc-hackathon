# Story Generation Plan — マカセル

> **AI-DLC Inception Phase / User Stories — Planning 成果物**
>
> 5/10 締切に間に合わせるため、本プランは Light 深さで進める。
> ユーザーへの大規模な質問ラウンドは行わず、
> [`requirements.md`](../requirements/requirements.md) と
> [`user-stories-assessment.md`](./user-stories-assessment.md) の合意済み事項を踏襲する。

## 採用済みの方針（事前確定）

| 項目 | 値 | 出典 |
|---|---|---|
| Story Breakdown Approach | 段階ベース（段階 1 / 2 / 3）+ 機能ベース | assessment.md |
| Story Format | "As a / I want / So that" + Given-When-Then の Acceptance Criteria | INVEST 標準 |
| Story 数 | 5〜10 件（段階 1: 3 / 段階 2: 3〜4 / 段階 3: 1〜2） | assessment.md |
| ペルソナ | 鈴木健太 1 名（マジメ・完璧主義タイプ）、Light 深さ | requirements.md §3 |
| Acceptance Criteria 粒度 | 1 ストーリー あたり 2〜3 個 | Light depth |

## Story Generation Steps

- [ ] Step 1: personas.md を生成（鈴木健太 1 名、Light）
- [ ] Step 2: stories.md ヘッダー（プロダクト概要 / 段階構造 / 凡例）を生成
- [ ] Step 3: 段階 1 のストーリー（3 件）を生成
- [ ] Step 4: 段階 2 のストーリー（3〜4 件）を生成
- [ ] Step 5: 段階 3 のストーリー（1〜2 件）を生成
- [ ] Step 6: ストーリー一覧サマリと MVP/決勝マッピング表を生成
- [ ] Step 7: aidlc-state.md / audit.md を更新

## Question Embedding（最小限）

レビュー v2 で「ペルソナ深掘りしない」方針が確定しているため、追加質問は **行わない**。
合理的なデフォルトで進め、生成後にユーザー承認を取る。

## Risks
- ストーリー数が少なすぎると書類審査で物足りない印象を与える可能性 → 7 件前後を狙う
- Acceptance Criteria が抽象的だと予選デモ判定が曖昧 → Given-When-Then で具体化
