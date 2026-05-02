# Component Methods — auto-mc-operation

各コンポーネントのメソッドシグネチャ(Rust トレイト / 構造体)。**詳細業務ロジックは Functional Design(per-unit, Construction)で確定** する想定で、ここではインターフェース契約を定義する。

エラー型は **`DomainError`**(Q3=C: 4 カテゴリの最上位 + ドメイン別ネスト)を統一使用。

```rust
// crates/domain/src/error.rs
pub enum DomainError {
    Transient { source: TransientCause, retryable: bool },
    Recoverable { action: UserAction, message_key: MessageKey },
    DataIssue { field: &'static str, raw_value: Option<String> },
    Permanent { reason: String, audit_required: bool },
}
```

---

## 1. ドメイン層トレイト

### D-12 OverlapDetector

```rust
// 移動時間バッファ込みの重複判定 — PBT 対象核心
pub struct OverlapDetector { buffer_minutes: u32 }

impl OverlapDetector {
    pub fn new(buffer_minutes: u32) -> Self;
    /// 既存予定リストと候補スロットから重複ステータスを判定
    pub fn check(
        &self,
        candidate: &Schedule,
        existing: &[ExistingEvent],   // {range, kind: Tentative|Confirmed|Private}
    ) -> OverlapStatus;
}
```

### D-15 MailRepository(F-11 ポート)

```rust
#[async_trait]
pub trait MailRepository: Send + Sync {
    async fn fetch_message(&self, user_id: &UserId, message_id: &MessageId) -> Result<RawMail, DomainError>;
    async fn list_history(&self, user_id: &UserId, start_history_id: HistoryId) -> Result<Vec<MessageRef>, DomainError>;
    async fn create_draft(&self, user_id: &UserId, draft: &EmailDraft) -> Result<DraftId, DomainError>;
    async fn send_draft(&self, user_id: &UserId, draft_id: &DraftId) -> Result<MessageId, DomainError>;
    async fn watch(&self, user_id: &UserId, topic: &PubSubTopic) -> Result<WatchExpiry, DomainError>;
    async fn list_sent(&self, user_id: &UserId, query: &MailQuery) -> Result<Vec<MailMetadata>, DomainError>;
}
```

### D-16 CalendarRepository(F-11 ポート、Q3 プライバシー対応)

```rust
#[async_trait]
pub trait CalendarRepository: Send + Sync {
    /// freeBusy 情報のみ取得(プライベート予定の中身は取得しない)
    async fn busy_ranges(&self, user_id: &UserId, range: &TimeRange) -> Result<Vec<TimeRange>, DomainError>;

    /// 本サービスが作成した [仮]/[確定] イベントの詳細取得
    async fn list_owned_events(&self, user_id: &UserId, range: &TimeRange) -> Result<Vec<OwnedEvent>, DomainError>;

    async fn create_tentative(&self, user_id: &UserId, case_id: &CaseId, slot: &Schedule) -> Result<EventId, DomainError>;
    async fn promote_to_confirmed(&self, user_id: &UserId, event_id: &EventId) -> Result<(), DomainError>;
    async fn delete_event(&self, user_id: &UserId, event_id: &EventId) -> Result<(), DomainError>;
    async fn delete_by_case(&self, user_id: &UserId, case_id: &CaseId) -> Result<u32, DomainError>; // 返り値 = 削除件数
}
```

### D-17 NotificationChannel(F-11 ポート)

```rust
#[async_trait]
pub trait NotificationChannel: Send + Sync {
    async fn send_text(&self, user_id: &UserId, message_key: MessageKey, params: &TemplateParams) -> Result<(), DomainError>;
    async fn send_action_buttons(&self, user_id: &UserId, payload: &ActionPayload) -> Result<(), DomainError>;
    async fn verify_webhook(&self, headers: &HttpHeaders, raw_body: &[u8]) -> Result<WebhookEvent, DomainError>;
}
```

### D-18 LlmClient(F-11 ポート)

```rust
#[async_trait]
pub trait LlmClient: Send + Sync {
    /// 構造化抽出(JSON Schema 強制)
    async fn extract<T: DeserializeOwned + JsonSchema>(
        &self,
        prompt: &PromptTemplate,
        input: &str,
    ) -> Result<T, DomainError>;

    /// 自由文生成(プロンプトキャッシュ対応)
    async fn generate(
        &self,
        cached_prefix: &CachedPrompt,
        dynamic_suffix: &str,
        max_tokens: u32,
    ) -> Result<String, DomainError>;
}
```

### D-14 リポジトリトレイト(抜粋)

```rust
#[async_trait]
pub trait CaseRepository: Send + Sync {
    async fn save(&self, case: &Case) -> Result<(), DomainError>;
    async fn find(&self, case_id: &CaseId) -> Result<Option<Case>, DomainError>;
    async fn list_by_user_in_range(&self, user_id: &UserId, range: &TimeRange) -> Result<Vec<Case>, DomainError>;
    async fn update_status(&self, case_id: &CaseId, status: CaseStatus) -> Result<(), DomainError>;
}

#[async_trait]
pub trait MessageRepository: Send + Sync {
    async fn save(&self, msg: &Message) -> Result<(), DomainError>;
    async fn mark_classified(&self, message_id: &MessageId, label: ClassificationLabel, by: ClassifiedBy) -> Result<(), DomainError>;
    async fn list_needs_review(&self, user_id: &UserId, limit: u32) -> Result<Vec<Message>, DomainError>;
}

// PrCorpus / DeclineCorpus / Consent / OfficePattern / ClassificationRule / AuditLog も同様の CRUD パターン
```

---

## 2. アプリケーション層 ユースケース

```rust
// 共通: 各ユースケースは Command 型と execute 関数を持つ
pub struct UseCaseCtx<R> { pub repo: R, pub logger: Logger, pub clock: Clock }

// A-1
pub struct OnboardUserCommand { pub line_user_id: LineUserId, pub auth_code: AuthCode }
pub trait OnboardUserUseCase {
    async fn execute(&self, cmd: OnboardUserCommand) -> Result<UserId, DomainError>;
}

// A-2
pub struct IngestMailCommand { pub user_id: UserId, pub history_id: HistoryId }
pub trait IngestMailUseCase {
    async fn execute(&self, cmd: IngestMailCommand) -> Result<Vec<MessageId>, DomainError>; // enqueue 件数
}

// A-3
pub struct ClassifyMailCommand { pub user_id: UserId, pub message_id: MessageId }
pub trait ClassifyMailUseCase {
    async fn execute(&self, cmd: ClassifyMailCommand) -> Result<ClassificationOutcome, DomainError>;
}
pub struct ClassificationOutcome {
    pub label: ClassificationLabel,
    pub by: ClassifiedBy,        // Rule | Llm
    pub confidence: f32,
    pub next_step: Option<QueueName>,  // extract_queue / decision_queue / None
}

// A-4
pub struct ExtractCaseCommand { pub user_id: UserId, pub message_id: MessageId }
pub trait ExtractCaseUseCase {
    async fn execute(&self, cmd: ExtractCaseCommand) -> Result<CaseId, DomainError>;
}

// A-5
pub struct CheckAvailabilityCommand { pub user_id: UserId, pub case_id: CaseId }
pub trait CheckAvailabilityUseCase {
    async fn execute(&self, cmd: CheckAvailabilityCommand) -> Result<AvailabilityResult, DomainError>;
}
pub struct AvailabilityResult {
    pub per_slot: Vec<(SlotId, OverlapStatus)>,
    pub overall: AvailabilityVerdict,  // Available { available_slots } | Unavailable
}

// A-6
pub struct ComposeEntryDraftCommand { pub user_id: UserId, pub case_id: CaseId, pub chosen_slot: SlotId }
pub trait ComposeEntryDraftUseCase {
    async fn execute(&self, cmd: ComposeEntryDraftCommand) -> Result<DraftId, DomainError>;
}

// A-7
pub enum CalendarOperation {
    RegisterTentativeAll(CaseId),
    PromoteAndCleanup { case_id: CaseId, chosen_slot: SlotId },
    DeleteAllByCase(CaseId),
}
pub trait ManageCalendarUseCase {
    async fn execute(&self, op: CalendarOperation, user_id: &UserId) -> Result<(), DomainError>;
}

// A-8
pub struct DetectAndDeclineConflictsCommand { pub user_id: UserId, pub confirmed_case: CaseId, pub confirmed_slot: SlotId }
pub trait DetectAndDeclineConflictsUseCase {
    async fn execute(&self, cmd: DetectAndDeclineConflictsCommand) -> Result<DeclineProposals, DomainError>;
}
pub struct DeclineProposals { pub items: Vec<DeclineDraft> } // ユーザー承認待ちで返す

// A-9
pub trait NotifyUserUseCase {
    async fn notify(&self, user_id: &UserId, event: NotificationEvent) -> Result<(), DomainError>;
    async fn handle_postback(&self, payload: PostbackPayload) -> Result<(), DomainError>;
}
```

---

## 3. インフラストラクチャ層 主要構造体

```rust
// I-9
pub struct ClassificationRuleEngine { rules_kv: KvStore }
impl ClassificationRuleEngine {
    pub async fn evaluate(&self, mail: &MailMetadata) -> Result<ClassificationOutcome, DomainError>;
    pub async fn upsert_rule(&self, rule: ClassificationRule) -> Result<(), DomainError>;
    pub async fn list_candidates_for_promotion(&self) -> Result<Vec<DomainPromotionCandidate>, DomainError>;
}

// I-10
pub struct CryptoService { keys: SecretBundle }
impl CryptoService {
    pub fn encrypt(&self, plaintext: &[u8]) -> Result<EncryptedBlob, DomainError>; // {key_id, nonce, ct, tag}
    pub fn decrypt(&self, blob: &EncryptedBlob) -> Result<Vec<u8>, DomainError>;   // 旧 key_id にも対応
    pub fn current_key_id(&self) -> KeyId;
}

// I-8 Queues
pub struct QueueProducer<T: Serialize> { binding: cloudflare::Queue }
impl<T> QueueProducer<T> {
    pub async fn enqueue(&self, msg: T, idempotency_key: Option<&str>) -> Result<(), DomainError>;
    pub async fn enqueue_with_delay(&self, msg: T, delay_seconds: u32) -> Result<(), DomainError>;
}
```

---

## 4. プレゼンテーション層 ハンドラ

```rust
// P-1
pub async fn handle_line_webhook(req: Request, env: Env) -> Result<Response, worker::Error>;
// 内部で NotificationChannel::verify_webhook → イベント種別で A-9 などへ dispatch

// P-2
pub async fn handle_pubsub_webhook(req: Request, env: Env) -> Result<Response, worker::Error>;
// JWT 検証 → A-2 起動

// P-5
pub async fn run_queue_consumer<C: ConsumerLogic>(batch: MessageBatch, env: Env) -> Result<(), worker::Error>;
// Queue ごとに ClassifyMail / ExtractCase / CheckAvailability / ComposeDraft / Notify / ManageCalendar の各ユースケースを呼ぶ
```

---

## 5. メソッド設計の注記

- **Q4=C 完全 API 仕様**: 各 HTTP ハンドラのリクエスト/レスポンス・認証ヘッダ・レート制限は `application-design.md` の API 設計セクションで一元管理
- **Q5=B 全列+制約+主要IDX**: メソッドが受け取る型(Case / Schedule / Entry 等)は D1 スキーマと 1:1 で `application-design.md` のデータモデル節と同期
- **Q7=A eventually consistent / saga**: 各ユースケースは **失敗時補償ハンドラ** を内部に持ち、Queue 経由で次ステップへ繋ぐ。ユースケース完了時点で D1 トランザクションをコミットし、後段失敗時は補償操作を別ユースケース起動でロールバック
- **Q8=A モック境界**: D-15〜D-18 トレイトに対する `MockMailRepository` / `MockCalendarRepository` / `MockNotificationChannel` / `MockLlmClient` を `crates/shared/test_support` に置き、ユニット / 結合テストで使用

詳細なエラーハンドリング・バリデーション・業務ルールは **Functional Design ステージ(per-unit, Construction)** で確定する。
