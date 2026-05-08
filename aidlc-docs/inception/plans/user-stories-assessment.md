# User Stories Assessment — マカセル

## Request Analysis
- **Original Request**: AWS AI-DLC ハッカソン 2026 提案「マカセル」を構築。書類審査（5/15）で評価される Inception 成果物の 1 つとして User Stories が必須
- **User Impact**: Direct（ユーザーが日常的に対話するアプリ）
- **Complexity Level**: Medium（音声入力 / 外部 API / AI 代行が組み合わさる）
- **Stakeholders**: ハッカソン審査員（書類審査・予選・決勝） / チームメンバー（5/30 までに MVP 開発に着手）

## Assessment Criteria Met

### High Priority Indicators
- [x] **New User Features**: Greenfield、すべてが新機能
- [x] **Multi-Persona Systems**: メインペルソナ 1 名で運用するが、外部受信者（メールの宛先など）も間接的なステークホルダー
- [x] **Complex Business Logic**: 八木案 3 段階の旅路 / AI 代行ロジック / 音声入力ハンドリング

### Medium Priority Factors
- [x] **Scope**: フロント / バック / AI / 外部 API にまたがる
- [x] **Stakeholders**: ハッカソン審査員 + チームメンバーへの伝達が必要
- [x] **Testing**: Acceptance Criteria が無いと予選 / 決勝デモの完成判定が曖昧になる

### Benefits
- 書類審査の必須提出物を満たす
- チームメンバー（umitsu / yagi / yakumo / tarekatsu）と共通理解を構築できる
- 段階 1〜2 の MVP / 段階 3 の決勝拡張の境界をストーリー単位で明示できる

## Decision

**Execute User Stories**: **Yes**

**Reasoning**:
書類審査で User Stories は必須提出物の 1 つとして明記されている（提案書 03-proposal.md §1.3）。
レビュー v2 で「ペルソナ深掘りしない」方針が示されたため、**Light depth** で実施し、
INVEST を満たす最小限のストーリー（5〜10 件）と、簡素なペルソナ記述で書類審査通過を狙う。

## Expected Outcomes
- INVEST 準拠の 5〜10 件のユーザーストーリー
- 簡素な personas.md（鈴木健太 1 名、TASKILL の毒性が刺さる構造を明記）
- 各ストーリーに Acceptance Criteria（Given-When-Then 形式）
- MVP / 決勝の境界がストーリー単位で読み取れる状態

## Story Breakdown Approach

**選択**: **段階ベース（八木案の段階 1〜3）+ 機能ベースのハイブリッド**

理由:
- 八木案の 3 段階の旅路がプロダクトの背骨、これに沿って整理するのが自然
- 段階内では機能（連携 / 代行 / 表示）でブレークダウン
- MVP（段階 1〜2）と決勝（段階 3）の境界が明示される
- ペルソナ別ブレークダウンは取らない（ペルソナ 1 名固定のため不要）

## Story Granularity

- **Light depth**: 1 ストーリー = 1 INVEST 単位、Acceptance Criteria 2〜3 個
- **Story Format**: 「〜として、〜したい、なぜなら〜」+ Acceptance Criteria（Given-When-Then）
- **Story 数の目安**: 5〜10 件（段階 1: 3 件 / 段階 2: 3〜4 件 / 段階 3: 1〜2 件）

## Mandatory Story Artifacts
- [x] stories.md（INVEST 準拠、Acceptance Criteria 込み）
- [x] personas.md（鈴木健太 1 名、Light 深さ）
- [x] ストーリーごとに段階・MVP/決勝境界をマッピング
