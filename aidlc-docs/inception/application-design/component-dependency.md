# Component Dependency — auto-mc-operation

> **📑 ID 参照**: `D-NN` / `A-N` / `I-NN` / `P-N` / `S-N` の **正本は `components.md`**、`F-NN` の正本は `inception/user-stories/stories.md`。横断 ID 一覧は `aidlc-docs/inception/id-index.md`。

依存関係マトリクスと通信パターン。**DDD の依存方向(presentation → application → domain ← infrastructure、shared は最下層)** を機械的に守る(F-01 AC-2)。

## 1. レイヤ間の依存方向(C4 Component 図)

```mermaid
flowchart TB
    subgraph PR["🟦 presentation (Workers ハンドラ)"]
        P1[LineWebhookHandler]
        P2[PubSubWebhookHandler]
        P3[OAuthCallbackHandler]
        P4[OnboardingHandler]
        P5[QueueConsumer]
        P6[CronWorker]
        P7[AdminApi]
    end

    subgraph APP["🟩 application (UseCase)"]
        A1[OnboardUser]
        A2[IngestMail]
        A3[ClassifyMail]
        A4[ExtractCase]
        A5[CheckAvailability]
        A6[ComposeDraft]
        A7[ManageCalendar]
        A8[DetectDecline]
        A9[NotifyUser]
        A10[RotateGmailWatch]
    end

    subgraph DOM["🟨 domain (Entity / VO / Service / Port / Error)"]
        D12[OverlapDetector]
        Ports["Ports<br/>MailRepo / CalRepo<br/>NotifChan / LlmClient"]
        Aggs["Aggregates<br/>User / Case / Entry / Decline<br/>Message / PrCorpus / DeclineCorpus"]
    end

    subgraph INF["🟥 infrastructure (Adapter)"]
        I1[GmailAdapter]
        I2[GoogleCalendarAdapter]
        I3[LineMessagingAdapter]
        I4[AnthropicClient]
        I5[D1Repos]
        I6[KvStore]
        I7[R2Storage]
        I8[Queues]
        I9[ClassRuleEngine]
        I10[CryptoService]
        I11[OAuth2Service]
    end

    subgraph SH["⬛ shared"]
        S1[Logger]
        S2[MessageCatalog]
        S3[ErrorClassifier]
        S4[TestSupport]
    end

    PR --> APP
    APP --> DOM
    INF -- implements --> DOM
    APP -. via DI .-> INF
    PR -. uses .-> SH
    APP -. uses .-> SH
    INF -. uses .-> SH

    style PR fill:#BBDEFB,stroke:#1565C0,color:#000
    style APP fill:#C8E6C9,stroke:#2E7D32,color:#000
    style DOM fill:#FFF59D,stroke:#F57F17,color:#000
    style INF fill:#FFCCBC,stroke:#D84315,color:#000
    style SH fill:#E0E0E0,stroke:#424242,color:#000
```

**禁止依存(`cargo deny` / `cargo-modules` で CI 検出)**:
- ❌ `domain → infrastructure`(domain は外部依存ゼロ)
- ❌ `domain → application`(domain は最下層)
- ❌ `application → presentation`
- ❌ `presentation → infrastructure`(必ず application を経由)
- ❌ `infrastructure → presentation`

## 2. ユースケース → トレイトのマトリクス

各ユースケースが利用するドメイントレイト(各セルに ✓ がある = 依存):

| UseCase / Trait | UserRepo | CaseRepo | EntryRepo | DeclineRepo | MessageRepo | PrCorpus | DeclineCorpus | ConsentRepo | OfficePat | ClassRule | AuditLog | MailPort | CalPort | NotifPort | LlmPort | OAuthPort | Crypto |
|-----------------|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
| A-1 OnboardUser | ✓ |   |   |   |   |   |   | ✓ |   |   | ✓ | ✓ |   |   |   | ✓ | ✓ |
| A-2 IngestMail | ✓ |   |   |   | ✓ |   |   | ✓ |   |   | ✓ | ✓ |   |   |   |   |   |
| A-3 ClassifyMail | ✓ |   |   |   | ✓ |   |   |   |   | ✓ | ✓ |   |   |   | ✓ |   |   |
| A-4 ExtractCase | ✓ | ✓ |   |   | ✓ |   |   |   | ✓ |   | ✓ |   |   |   | ✓ |   |   |
| A-5 CheckAvail | ✓ | ✓ |   |   |   |   |   |   |   |   | ✓ |   | ✓ |   |   |   |   |
| A-6 ComposeDraft | ✓ | ✓ | ✓ |   |   | ✓ |   |   | ✓ |   | ✓ | ✓ |   |   | ✓ |   |   |
| A-7 ManageCal | ✓ | ✓ | ✓ |   |   |   |   |   |   |   | ✓ |   | ✓ |   |   |   |   |
| A-8 DetectDecline | ✓ | ✓ | ✓ | ✓ |   |   | ✓ |   |   |   | ✓ | ✓ |   |   | ✓ |   |   |
| A-9 NotifyUser | ✓ | ✓ |   |   |   |   |   |   |   |   | ✓ |   |   | ✓ |   |   |   |
| A-10 RotateGmailWatch | ✓ |   |   |   |   |   |   |   |   |   | ✓ | ✓ |   |   |   |   |   |

**注**: A-2 IngestMail は処理開始時に最新同意確認のため `ConsentRepo` に依存(F-07 連携)。新規ポート `OAuthPort` は S2 で追加(下記)。

## 3. 通信パターン

### 3.1 同期 vs 非同期

| 連携 | 種別 | チャネル |
|------|------|---------|
| `Presentation → Application`(HTTP リクエスト処理) | 同期 | 関数呼び出し |
| `Application → Application`(ユースケース間連携) | **非同期 9 割 + 同期 1 割** | 主に Queues、Postback 即時応答のみ同期 |
| `Application → Infrastructure(Repo)` | 同期 | DI 注入 + async 関数 |
| `Application → Infrastructure(外部 API)` | 同期(awaitable) | DI 注入 + async 関数(内部リトライ・タイムアウト管理) |
| `Infrastructure → External`(Gmail / Calendar / LINE / Anthropic) | 非同期(HTTP) | `reqwest` + `AbortSignal` |
| `Cron → Application` | 同期(Worker 起動) | scheduled handler |

### 3.2 メッセージ形式(Queue ペイロード)

すべての Queue メッセージは共通エンベロープ:

```rust
#[derive(Serialize, Deserialize)]
pub struct QueueEnvelope<T> {
    pub message_id: Uuid,
    pub correlation_id: Uuid,           // 同一フローの全メッセージ共通
    pub user_id: UserId,
    pub timestamp: DateTime<Utc>,
    pub idempotency_key: String,
    /// アプリ独自カウンタ。Cloudflare Queues が提供する `message.attempts` は
    /// 「この Queue でのみの試行回数」のため、ステップ別 Queue を連鎖したときの
    /// 通算試行回数を保持するためにペイロード側でも管理する。
    /// 同 Queue 内での試行カウントは `message.attempts` を優先利用する。
    pub retry_count: u32,
    pub payload: T,
}
```

ペイロード型例:

```rust
pub enum ClassifyPayload {
    New { message_id: MessageId },
}

pub enum ExtractPayload {
    Pending { message_id: MessageId, label: ClassificationLabel },
}

pub enum AvailabilityPayload {
    Check { case_id: CaseId },
}

pub enum DraftPayload {
    Compose { case_id: CaseId, chosen_slot: SlotId, requires_pr: bool },
}

pub enum NotifyPayload {
    CaseSummary { case_id: CaseId },
    DraftReady { case_id: CaseId, draft_id: DraftId },
    DeclineProposal { proposals: Vec<DeclineDraft> },
    Error { kind: ErrorKind, hint: MessageKey },
}

pub enum CalendarPayload {
    RegisterTentativeAll(CaseId),
    PromoteAndCleanup { case_id: CaseId, chosen_slot: SlotId },
    DeleteAllByCase(CaseId),
}

pub enum DeclinePayload {
    Detect { confirmed_case: CaseId, confirmed_slot: SlotId },
}
```

### 3.3 冪等性

- すべての Queue メッセージは `idempotency_key` を持つ(KV で重複検知)
- ユースケースは **冪等に書く**(同じ `idempotency_key` で 2 回呼ばれても結果同じ)
- HTTP ハンドラも冪等(Pub/Sub の at-least-once 配信に対応)

## 4. データ永続化マッピング

**ドメイン集約 ↔ D1 テーブル の対応**(全 15 テーブル網羅。集約とテーブルが 1:1 でないものは脚注参照):

| ドメイン集約 | D1 テーブル | 暗号化 | 備考 |
|-------------|-----------|--------|------|
| User | `users` | — | プロフィール・設定 |
| User(関連) | **`oauth_tokens`** | AES-256-GCM(F-09 / `key_id` 別カラム) | User 集約に内包する別テーブル(1:1) |
| Consent | `consents`(`users` に FK) | — | append-only(トリガー) |
| Message | `messages`(`users` に FK) | — | 本文は R2 |
| Office | `offices` | — | 事務所マスタ。`offices.id` を案件 / コーパスから参照 |
| OfficePattern | `office_patterns`(`offices` に 1:1) | — | 抽出ヒント詳細サブテーブル |
| ClassificationRule | `classification_rules`(+ KV キャッシュ) | — | global / user 両スコープ |
| Case | `cases` | — | 集約ルート |
| Schedule | `schedules`(`cases` に FK) | — | Case の構成要素 |
| Entry | `entries`(`cases` に FK / 1 case = 1 active) | — | 履歴は `status='superseded'` |
| Decline | `declines`(`cases` に FK / `triggered_by_kind` で分類) | — | proposed/approved/sent/failed |
| (Case 関連) | **`calendar_events`**(`users` / `cases` / `schedules` に FK) | — | A-7 ManageCalendar が CRUD する独立テーブル(1 Case = N 候補スロット = N events) |
| PrCorpus | `pr_corpus`(`users` に FK) | — | 本人作成・永続 |
| DeclineCorpus | `decline_corpus`(`users` に FK) | — | 本人作成・永続 |
| AuditLog | `audit_logs` | — | F-06 構造化ログから 12 ヶ月保管 |

**脚注**:
- `oauth_tokens` は User 集約に紐づくが、暗号化要件と更新サイクルが異なるため **物理的には独立テーブル**(F-09 鍵ローテーションで再暗号化のトランザクションを限定するため)
- `calendar_events` は Case 配下の派生実体で、Google Calendar 側 ID(`google_event_id`)とサービス所有状態(`tentative`/`confirmed`/`deleted`)を保持

詳細スキーマは `application-design.md` のデータモデル節および `data-model.md` を参照。

## 5. 横断的関心事

| 関心事 | 適用箇所 |
|-------|---------|
| **構造化ログ** | すべてのレイヤで `Logger` 経由(F-06) |
| **エラー分類** | `application` レイヤで `ErrorClassifier` を通して U2-EC-04 4 カテゴリに振り分け |
| **メッセージ国際化** | `presentation` / `application` で `MessageCatalog` 経由(F-08) |
| **テストダブル** | `crates/shared/test_support` の Mock 実装(Q8=A) |
| **メトリクス** | `Logger` 内に `MetricsRecorder` を仕込み Workers Analytics Engine へ送信(F-14) |
| **リトライ** | `Queues` のリトライポリシー(infrastructure 層)+ ユースケース内のドメイン固有再試行 |

## 6. 主要シーケンス(エンドツーエンド フロー)

`services.md` の Saga 図参照。`A-2 → A-3 → A-4 → A-5 → A-6 → A-9 → (Postback) → A-7` がメインフロー。

決定後の補完フロー: `A-3(decision) → A-7 PromoteAndCleanup → A-8 DetectDecline → A-9 → (Postback) → A-8 send → A-7 DeleteAllByCase(declined)`。
