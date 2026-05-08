# Component Methods — マカセル

> **AI-DLC Inception Phase / Application Design 成果物**
>
> 主要メソッドのシグネチャと I/O のみを定義する。詳細な業務ロジックは Construction フェーズの Functional Design で記述する。
> TypeScript 風の型表記。
>
> ### 設計変更（2026-05-08, ADR-1' 改訂版）
> Memory Service / External API Gateway（AgentCore Gateway）は不採用。
> 外部 API は **Bedrock Converse API の tool_use** + Lambda Tool（C-5 Calendar Tool / C-6 Gmail Tool）で実装。

---

## C-2. API / BFF Layer（API Gateway HTTP API + Lambda Resolver）

```typescript
// 共通の型定義（フロント・バック共有）
type SaboriItem = {
  itemId: string;
  userId: string;
  date: string;          // YYYY-MM-DD
  source: "calendar" | "gmail";
  title: string;
  reason: string;        // AI が生成したサボってよい根拠
  status: "pending" | "marked_as_sabori" | "delegated" | "executed";
  createdAt: string;
};

type DelegationDraft = {
  itemId: string;
  // 代行アクションの 3 タイプ：
  //   request_to_others = 他者への依頼（中心）。誰かに代わってもらう
  //   decline           = 断り・欠席連絡。自分が降りる
  //   reschedule        = 延期・再調整
  draftType: "request_to_others" | "decline" | "reschedule";
  body: string;
  recipient: string;     // 「依頼」なら依頼先、「断り」なら断り先（モック宛先 or 実宛先）
  status: "draft" | "approved" | "rejected" | "executed" | "failed";
};

type DelegationLog = {
  logId: string;
  itemId: string;
  userId: string;
  executedAt: string;
  result: "success" | "failure";
  errorDetail?: string;
};

// ====== HTTP API Endpoints ======
// 全エンドポイントに Cognito User Pool Authorizer を適用、JWT 検証済みの userId を context から取得

// テキスト入力（音声経路と合流するエントリポイント）
// POST /v1/instructions
//   Body: { text: string, sessionId?: string }
//   → AI Agent invoke（tool_use ループ）→ AgentResponse を同期で返す
type PostInstructionRequest = { text: string; sessionId?: string };
type AgentResponse = { kind: "ack" | "draft_ready" | "error"; payload: unknown };

// サボリスト一覧取得
// GET /v1/sabori-items?date=YYYY-MM-DD
type GetSaboriItemsResponse = { items: SaboriItem[] };

// 項目を「サボる」状態に
// POST /v1/sabori-items/{itemId}/mark
type MarkAsSaboriResponse = { item: SaboriItem };

// 代行下書きの取得
// GET /v1/delegations/{itemId}/draft
type GetDraftResponse = { draft: DelegationDraft | null };

// 代行承認 → 実行トリガ（同期で実行結果まで返すか、ack のみで非同期実行かは Functional Design で決定）
// POST /v1/delegations/{itemId}/approve
type ApproveDelegationResponse = { draft: DelegationDraft };

// 代行却下
// POST /v1/delegations/{itemId}/reject
type RejectDelegationResponse = { draft: DelegationDraft };

// 代行ログ
// GET /v1/delegation-logs?limit=20
type GetLogsResponse = { logs: DelegationLog[] };

// ユーザー設定（自律実行モード ON/OFF など、決勝向け）
// PUT /v1/settings
type PutSettingsRequest = { autonomousMode?: boolean };
type PutSettingsResponse = { settings: UserSettings };
```

> **MVP では Subscription を持たない**。代行ステータスの変化が必要な画面は、`approveDelegation` のレスポンスを同期で受け取る or `getDelegationDraft` を短期ポーリング（必要なら決勝までに WebSocket 化を再検討）。

---

## C-3. Voice Stream Handler（API Gateway WebSocket + Lambda）

```typescript
interface VoiceSession {
  sessionId: string;
  userId: string;
  state: "idle" | "listening" | "transcribing" | "agent_processing";
}

interface VoiceStreamHandler {
  // WebSocket 接続時
  onConnect(userId: string): VoiceSession;

  // 音声バイナリチャンク受信
  onAudioChunk(sessionId: string, chunk: ArrayBuffer): void;

  // Transcribe からの結果コールバック
  onTranscriptionResult(sessionId: string, text: string, isFinal: boolean): void;

  // 確定したテキストを Agent に渡す
  forwardToAgent(sessionId: string, text: string): Promise<AgentResponse>;

  // セッション終了
  onDisconnect(sessionId: string): void;
}
```

---

## C-4. AI Agent（AgentCore Runtime / Strands Agent + Bedrock tool_use）

```python
# Agent エントリポイント（Runtime invoke）
class MakaseruAgent:
    """Strands Agent。Bedrock Converse API の tool_use ループで Lambda Tool を呼び出す"""

    def invoke(self, user_id: str, session_id: str, intent: AgentIntent) -> AgentResponse:
        """音声テキストから意図を解釈し、必要に応じて tool_use ループで Tool 呼出"""
        ...

# 意図種別
AgentIntent = Union[
    ReadSaboriListIntent,       # 「今日のサボリリスト読んで」など
    MarkAsSaboriIntent,         # 「これサボる」「明日の朝会、いらない」
    ApproveDelegationIntent,    # 「うん」「いいよ」「それで」
    RejectDelegationIntent,     # 「やめる」「やっぱ違う」
    GenericQuestionIntent,      # その他の問い
]

# Bedrock Converse API に渡す tools 定義
TOOL_SPECS = [
    {
        "toolSpec": {
            "name": "list_calendar_events",
            "description": "指定日の Google Calendar イベント一覧を取得",
            "inputSchema": {
                "json": {
                    "type": "object",
                    "properties": {
                        "user_id": {"type": "string"},
                        "date": {"type": "string", "description": "YYYY-MM-DD"},
                    },
                    "required": ["user_id", "date"],
                }
            },
        }
    },
    {
        "toolSpec": {
            "name": "update_calendar_attendance",
            "description": "Google Calendar イベントの参加ステータスを更新",
            "inputSchema": { ... },
        }
    },
    {
        "toolSpec": {
            "name": "send_gmail",
            "description": "Gmail で送信",
            "inputSchema": { ... },
        }
    },
    # ...
]


def tool_use_loop(initial_messages: list, max_iters: int = 5) -> str:
    """Bedrock Converse API の tool_use ループ。
    Tool 呼び出しが返ってきたら対応する Lambda を invoke して、
    結果を tool_result として再度 Converse API に渡す"""
    messages = initial_messages
    for _ in range(max_iters):
        resp = bedrock.converse(
            modelId="apac.anthropic.claude-sonnet-4-6-...",
            messages=messages,
            toolConfig={"tools": TOOL_SPECS},
        )
        if resp["stopReason"] != "tool_use":
            return extract_text(resp)
        # tool_use のリクエストを取り出して該当 Lambda を invoke
        tool_results = invoke_tool_lambdas(resp["output"]["message"]["content"])
        messages.extend([
            resp["output"]["message"],
            {"role": "user", "content": tool_results},
        ])
    raise MaxIterationsExceeded()
```

---

## C-5. Calendar Tool Lambda（tool_use Tool 実体）

```python
class CalendarToolHandler:
    """Bedrock tool_use から呼ばれる Lambda。Google Calendar API を操作"""

    def list_events(self, user_id: str, date: str) -> list[CalendarEvent]:
        """指定日の予定一覧を取得"""
        token = identity.get_oauth_token(user_id, provider="google")
        return google_calendar.list(token, date)

    def update_attendance(self, user_id: str, event_id: str, status: Literal["accepted", "declined", "tentative"]) -> UpdateResult:
        """イベントの参加ステータスを更新"""
        token = identity.get_oauth_token(user_id, provider="google")
        return google_calendar.update_attendance(token, event_id, status)
```

---

## C-6. Gmail Tool Lambda（tool_use Tool 実体）

```python
class GmailToolHandler:
    """Bedrock tool_use から呼ばれる Lambda。Gmail API を操作"""

    def list_recent(self, user_id: str, limit: int = 20) -> list[Email]:
        """最近の受信メール一覧"""
        token = identity.get_oauth_token(user_id, provider="google")
        return gmail.list(token, limit)

    def send(self, user_id: str, to: str, subject: str, body: str) -> SendResult:
        """メール送信"""
        token = identity.get_oauth_token(user_id, provider="google")
        return gmail.send(token, to, subject, body)
```

---

## C-7. Identity Provider（AgentCore Identity）

```python
class IdentityProvider:
    """Google OAuth 2.0 トークンの取得・更新を AgentCore Identity 経由で行う"""

    def get_oauth_token(user_id: str, provider: Literal["google"]) -> OAuthToken:
        """有効な access_token を取得（必要に応じて refresh）"""
        ...

    def initiate_oauth_flow(user_id: str, provider: Literal["google"], scopes: list[str]) -> AuthorizationUrl:
        """初回 OAuth 同意フローを開始"""
        ...

    def revoke_oauth(user_id: str, provider: Literal["google"]) -> None:
        ...
```

---

## C-8. Sabori Logic Lambda

```typescript
interface SaboriLogic {
  // 日次バッチ（EventBridge Scheduler から起動）
  generateDailySaboriList(userId: string, date: string): Promise<SaboriItem[]>;

  // オンデマンド再生成（フロント or Agent から）
  regenerateSaboriList(userId: string, date: string): Promise<SaboriItem[]>;

  // 内部ロジック（AI Agent への問い合わせ + ルールベース判定の組合せ）
  private askAgentForCandidates(userId: string, date: string): Promise<Candidate[]>;
  private applyHeuristics(candidates: Candidate[]): SaboriItem[];
}
```

---

## C-9. Delegation Executor Lambda

```typescript
interface DelegationExecutor {
  // 承認された代行アクションを実行
  execute(itemId: string, userId: string): Promise<DelegationLog>;

  // タイプ別の実行ハンドラ
  private executeEmailReply(draft: DelegationDraft): Promise<ExecutionResult>;
  private executeCalendarDecline(draft: DelegationDraft): Promise<ExecutionResult>;

  // 失敗時のリトライ・記録
  private recordFailure(itemId: string, error: Error): Promise<DelegationLog>;
}
```

---

## C-10. Data Store（DynamoDB アクセスパターン）

| Pattern | PK | SK | 用途 |
|---|---|---|---|
| 1 | `USER#<userId>` | `SABORI#<date>#<itemId>` | 日付別サボリスト |
| 2 | `USER#<userId>` | `DRAFT#<itemId>` | 代行下書き |
| 3 | `USER#<userId>` | `LOG#<timestamp>#<logId>` | 代行ログ（30 日 TTL） |
| 4 | `USER#<userId>` | `SETTINGS` | ユーザー設定（自律実行モード等） |

```typescript
interface DataAccess {
  putSaboriItem(item: SaboriItem): Promise<void>;
  listSaboriItemsByDate(userId: string, date: string): Promise<SaboriItem[]>;

  putDelegationDraft(draft: DelegationDraft): Promise<void>;
  getDelegationDraft(userId: string, itemId: string): Promise<DelegationDraft | null>;
  updateDelegationDraftStatus(userId: string, itemId: string, status: DraftStatus): Promise<void>;

  putDelegationLog(log: DelegationLog): Promise<void>;
  listRecentLogs(userId: string, limit: number): Promise<DelegationLog[]>;

  getUserSettings(userId: string): Promise<UserSettings>;
  putUserSettings(userId: string, settings: UserSettings): Promise<void>;
}
```

---

> **注**: 業務ルール（しきい値、エラーリトライ回数、自律実行の判定ロジック等）は Functional Design（per-unit, Construction）で詳細化する。ここではメソッドシグネチャのみ。
