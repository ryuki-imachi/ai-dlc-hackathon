# Unit of Work Plan — マカセル

> **AI-DLC Inception Phase / Units Generation — Planning 成果物**
> **Depth**: Light。5/10 締切に間に合わせるため、追加質問は省略し、これまでの確定事項から合理的に分解する。

## 採用済みの方針（事前確定）

| 項目 | 値 | 出典 |
|---|---|---|
| デプロイモデル | サーバレス（Lambda + API GW + DynamoDB + AgentCore Runtime） | Application Design |
| IaC | AWS CDK（TypeScript）主導 + Amplify Hosting 軽め共存 | ADR-6 |
| 単一リポジトリ構成 | モノレポ（フロント + バック + IaC を 1 リポジトリで管理） | デフォルト方針 |
| Unit 粒度 | 機能ドメイン + 横断機能で 5〜8 Unit | 本プラン |

## Decomposition Approach（分解アプローチ）

**ハイブリッド型**: 機能ドメイン（Sabori Generation / Delegation Flow） + 横断基盤（Foundation / Identity & Tools / AI Agent / Voice Input / Frontend）+ 拡張（Autonomous Mode）

選択理由:
- Story の段階（1, 2, 3）と Unit が綺麗に対応する
- 横断基盤を分離することで MVP の依存関係グラフが明確になる
- AgentCore Runtime と Lambda Tool は別 Unit にして進化余地を残す

## モノレポのコード組織戦略（Greenfield）

```
makaseru/
├── README.md
├── cdk/                       # CDK アプリ（IaC、Unit 0〜7 のスタック）
│   ├── bin/cdk.ts
│   ├── lib/
│   │   ├── stacks/
│   │   │   ├── foundation-stack.ts        # Unit 0
│   │   │   ├── identity-tools-stack.ts    # Unit 1
│   │   │   ├── ai-agent-stack.ts          # Unit 2
│   │   │   ├── sabori-stack.ts            # Unit 3
│   │   │   ├── delegation-stack.ts        # Unit 4
│   │   │   ├── voice-stack.ts             # Unit 5
│   │   │   └── ...
│   │   └── constructs/
│   ├── package.json
│   └── tsconfig.json
├── frontend/                  # Next.js 15 (Unit 6)
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── package.json
├── agent/                     # Python: Strands Agent (Unit 2 内部)
│   ├── pyproject.toml
│   └── src/
├── lambdas/                   # Python: Lambda Tools / Sabori / Executor / Voice
│   ├── calendar_tool/
│   ├── gmail_tool/
│   ├── sabori_logic/
│   ├── delegation_executor/
│   ├── voice_stream_handler/
│   └── shared/
├── aidlc-docs/                # AI-DLC 成果物（既存）
└── pre-ai-dlc/                # 事前ブレスト（既存）
```

## Generation Steps

- [x] unit-of-work.md — Unit 定義と責務、コード組織戦略
- [x] unit-of-work-dependency.md — Unit 間の依存マトリクス
- [x] unit-of-work-story-map.md — ストーリー ⇆ Unit のマッピング

## Question Embedding

レビュー v2 / v3 を踏まえて「過剰な質問は避ける」方針。
追加質問は **行わない**。Construction フェーズで per-unit に詰める。
