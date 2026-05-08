# Unit of Work — Dependency Matrix

> **AI-DLC Inception Phase / Units Generation 成果物**
>
> Unit 間の依存関係と実装順序を定義する。

---

## 1. 依存マトリクス

行 = 依存元（this Unit が何に依存しているか）、列 = 依存先。`✓` = 依存あり、`-` = 依存なし、`(◯)` = 弱い依存（共有型のみなど）。

| From \ To | U-0 | U-1 | U-2 | U-3 | U-4 | U-5 | U-6 | U-7 |
|---|---|---|---|---|---|---|---|---|
| **U-0 Foundation** | - | - | - | - | - | - | - | - |
| **U-1 Identity & Tools** | ✓ | - | - | - | - | - | - | - |
| **U-2 AI Agent Core** | ✓ | ✓ | - | - | - | - | - | - |
| **U-3 Sabori Domain** | ✓ | ✓ | ✓ | - | - | - | - | - |
| **U-4 Delegation Domain** | ✓ | ✓ | ✓ | ✓ | - | - | - | - |
| **U-5 Voice Input Pipeline** | ✓ | - | ✓ | - | - | - | (◯) | - |
| **U-6 Frontend** | ✓ | - | - | ✓ | ✓ | ✓ | - | - |
| **U-7 Autonomous Mode** | ✓ | - | - | - | ✓ | - | (◯) | - |

> U-5 と U-6 の「(◯)」: フロントのマイク取り込みは U-6 のフロント側コードで、WebSocket 通信は U-5 のサーバ側。両者がプロトコル仕様（音声フォーマットや WebSocket メッセージ形式）を共有する程度の弱い依存。

---

## 2. 依存グラフ

```
                         ┌──────────────┐
                         │ U-0          │
                         │ Foundation   │
                         └──┬───────────┘
                            │
       ┌────────────────────┼────────────────────┬───────────┐
       ▼                    ▼                    ▼           ▼
  ┌──────────┐       ┌─────────────┐       ┌─────────┐  ┌─────────┐
  │ U-1      │──────▶│ U-2         │       │ U-5     │  │ U-6     │
  │ Identity │       │ AI Agent    │◀──────│ Voice   │  │ Frontend│
  │ & Tools  │       │ Core        │       │ Input   │  │         │
  └──────────┘       └──────┬──────┘       └────┬────┘  └────┬────┘
                            │                   │            │
                            ▼                   │            │
                    ┌───────────────┐           │            │
                    │ U-3 Sabori    │◀──────────┘            │
                    │ Domain        │                        │
                    └───────┬───────┘                        │
                            │                                │
                            ▼                                │
                    ┌───────────────┐                        │
                    │ U-4 Delegation│◀───────────────────────┘
                    │ Domain        │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ U-7           │ 決勝のみ
                    │ Autonomous    │
                    └───────────────┘
```

---

## 3. 実装順序（推奨）

依存グラフに基づき、以下の順序で実装するのが効率的:

| Phase | Unit | 並列実装可能か |
|---|---|---|
| 1 | **U-0 Foundation** | （単独） |
| 2 | **U-1 Identity & Tools** | U-1 と U-6（フロント雛形）は **並列可** |
| 3 | **U-2 AI Agent Core** | U-1 完了後。並列で U-3 の API 設計に着手可 |
| 4 | **U-3 Sabori Domain** + **U-5 Voice Input Pipeline** | U-2 完了後、**並列実装可** |
| 5 | **U-4 Delegation Domain** | U-3 のデータモデル確定後 |
| 6 | **U-6 Frontend（仕上げ）** | U-3 / U-4 / U-5 の API 確定後に統合 |
| 7 | **U-7 Autonomous Mode** | 予選後、決勝向けに追加（U-4 拡張） |

### クリティカルパス

```
U-0 → U-1 → U-2 → U-4 → U-6（統合）
             ↑
        (U-3 並列, U-5 並列)
```

最短で MVP を組むなら、**U-0 → U-1 → U-2 → U-3/U-5 並列 → U-4 → U-6 統合** の流れ。

---

## 4. 共有リソース・契約

Unit 間で共有する情報・契約を明示する:

| 共有対象 | 提供 Unit | 利用 Unit | 形式 |
|---|---|---|---|
| Cognito User Pool ID / App Client ID | U-0 | U-2, U-3, U-4, U-5, U-6 | CDK Stack output（CfnOutput）or SSM Parameter |
| DynamoDB テーブル名 / ARN | U-0 | U-3, U-4, U-7 | 同上 |
| AgentCore Identity Workload Identity 名 | U-1 | U-2 内部 / Tool Lambda（U-1） | CDK Stack output |
| AI Agent Runtime ARN / Endpoint | U-2 | U-3, U-4, U-5 | CDK Stack output |
| Tool Lambda 関数 ARN | U-1 | U-2（tool_use 設定）, U-4 | CDK Stack output |
| 共通 TypeScript 型 / Python dataclass | U-3, U-4 が定義 | U-6 が利用 | `lambdas/shared/`、フロントは手動同期 or codegen |
| HTTP API エンドポイント URL | U-3 / U-4（U-0 集約） | U-6 | CDK Output → `amplify_outputs.json` 経由でフロント |
| WebSocket API URL | U-5 | U-6 | 同上 |

---

## 5. 並列開発の境界

5/30 までに **チーム複数人で並列実装** する場合の Unit 担当割り当て例:

| 担当（仮） | 担当 Unit | 理由 |
|---|---|---|
| Backend / Cloud | U-0, U-1, U-2 | CDK 基盤 + AgentCore + Tool Lambda |
| Backend / Domain | U-3, U-4 | ドメインロジック + HTTP API |
| Frontend | U-6 | UI / UX |
| 横断 / ペア | U-5（Voice） | バック側（Lambda）+フロント側（マイク）両方を 1 人または 2 人で |
| 全員 | テスト・統合 | 5/25〜5/29 |

> 体制未確定（requirements.md Q-F1 = D）のため、上記は参考。
> 5/10 後に体制確定を行う。

---

## 6. 障害伝搬リスク

| 障害発生元 Unit | 影響範囲 | 緩和策 |
|---|---|---|
| U-0（DynamoDB / Cognito） | 全 Unit | バックアップ・PITR は MVP では持たない、ハッカソンスコープでは即時復旧優先 |
| U-1（Identity / Tool） | U-2, U-3, U-4 | OAuth トークン期限切れ時はフロントに再連携を促す |
| U-2（AI Agent） | U-3, U-4, U-5 | tool_use ループ上限到達 / Bedrock スロットリングは画面に明示してリトライ |
| U-5（Voice） | UX のみ（フォールバックでテキスト入力可） | 音声失敗時は U-6 のテキスト入力にフォールバック |
| U-6（Frontend） | UX のみ（バックは独立稼働） | キャッシュ・ホットリロードで影響最小化 |
