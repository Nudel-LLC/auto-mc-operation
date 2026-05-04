# Components — auto-mc-operation

> **📑 ID の正本**: 本ドキュメントは **`D-NN` / `A-N` / `I-NN` / `P-N` / `S-N`** すべての **正本**(定義元)。横断的な ID マッピングは `aidlc-docs/inception/id-index.md` 参照。

DDD レイヤ別のコンポーネント定義。Q1=B(中粒度・柔軟調整) に従い、**約 52 コンポーネント** を 5 レイヤに配置(Repository トレイトファミリー D-14 を 1 行に集約しているため、実質抽象は約 31)。

## 1. ドメイン層(`crates/domain`)— 外部依存ゼロ

### Entities / Aggregate Roots
| # | コンポーネント | 責務 |
|---|--------------|------|
| **D-1** | `User`(集約ルート) | ユーザーアカウント、OAuth トークン参照、同意状態、設定(移動時間バッファ等) |
| **D-2** | `Case`(集約ルート) | 案件 1 件。所属事務所(`office_id`)・案件名・PR要素要否・締切・複数 Schedule を集約 |
| **D-3** | `Schedule`(`Case` 内 Entity) | 候補スロット 1 つ。`{slot_id, start_at, end_at, tz, raw_text, confidence, overlap_status, is_chosen}`。1 case で 0/1/N 個が `is_chosen=1` |
| **D-4** | `Entry`(集約ルート) | エントリー 1 件。`{case_id, draft_id, status: pending|submitted|confirmed|declined|superseded}`。1 case = 1 active entry(履歴は `superseded`) |
| **D-5** | `Decline`(集約ルート) | 辞退送信 1 件。`{case_id, triggered_by_kind: case|private_event|manual, triggered_by_case?, triggered_by_note?, decline_draft, status, sent_at}` |
| **D-6** | `Message`(集約ルート) | 受信メール 1 件のメタ。`{message_id, history_id, classification, classified_by, needs_review}` |
| **D-6.5** | `Office`(集約ルート) | 事務所マスタ。`{id, sender_domain, display_name, aliases, is_blocked}`。`OfficePattern` を 1:1 でサブエンティティとして保持 |

### Value Objects
| # | コンポーネント | 責務 |
|---|--------------|------|
| **D-7** | `EmailAddress` / `OfficeName` / `Compensation` 等の VO | 不変条件付きの値型(空文字禁止、フォーマット検証等) |
| **D-8** | `TimeRange` | 区間値(`[start, end)`)、重複判定の演算子・移動時間バッファ拡張 |
| **D-9** | `ClassificationLabel` | enum: `Recruitment` / `Decision` / `Other` |
| **D-10** | `OverlapStatus` | enum: `Free` / `PartialOverlap(slot_id)` / `FullOverlap(slot_id)` |
| **D-11** | `ConsentVersion` | 同意ポリシー版数 + agreed_at |

### Domain Services
| # | コンポーネント | 責務 |
|---|--------------|------|
| **D-12** | `OverlapDetector` | 移動時間バッファ込み重複判定(プライベート busy 区間 + 確定予定 vs 仮予定の区別)。**PBT 対象の核心ロジック** |
| **D-13** | `ScheduleAggregator` | 複数スロットの判定結果から案件全体の可否(エントリー可能 / 辞退対象)を集約 |

### Repository Traits(domain で定義、infrastructure で実装)
| # | コンポーネント | 責務 |
|---|--------------|------|
| **D-14** | `UserRepository` / `CaseRepository` / `EntryRepository` / `DeclineRepository` / `MessageRepository` / `PrCorpusRepository` / `DeclineCorpusRepository` / `ConsentRepository` / `OfficeRepository` / `OfficePatternRepository` / `ClassificationRuleRepository` / `AuditLogRepository` | ドメインオブジェクトの永続化トレイト(D1 や KV の存在に依存しない) |

### External Service Ports(F-11 抽象化レイヤ)
| # | コンポーネント | 責務 |
|---|--------------|------|
| **D-15** | `MailRepository`(ポート) | メール取得・下書き作成・送信(Gmail / Outlook / IMAP に共通) |
| **D-16** | `CalendarRepository`(ポート) | freeBusy 取得・イベント作成/更新/削除(Google / Outlook / Apple に共通) |
| **D-17** | `NotificationChannel`(ポート) | メッセージ送信・ボタン操作 受信(LINE / Slack 等に共通) |
| **D-18** | `LlmClient`(ポート) | プロンプト送信・JSON レスポンス取得(Claude / GPT / Gemini に共通) |
| **D-18.5** | `OAuthExchanger`(ポート) | 認可コード ↔ アクセス/リフレッシュトークン交換、トークンリフレッシュ(Google OAuth / Phase 3 で Microsoft / Apple ID 等に拡張可)。F-11 拡張性方針との整合のため、application 層が直接 I-11 OAuth2Service を呼ばずポート経由で利用 |

### Domain Errors
| # | コンポーネント | 責務 |
|---|--------------|------|
| **D-19** | `DomainError` enum | Q3=C により **U2-EC-04 の 4 カテゴリ** を最上位:`Transient(reason)` / `Recoverable(action_required)` / `DataIssue(field)` / `Permanent(reason)`。各カテゴリの下にドメイン別 variants をネスト |

---

## 2. アプリケーション層(`crates/application`)— ユースケース

各ユースケースは **コマンド型** + **`execute(cmd) -> Result<Output, DomainError>`** インターフェース。Q7=A により saga パターンで失敗時補償。

| # | コンポーネント | 責務 |
|---|--------------|------|
| **A-1** | `OnboardUserUseCase` | F-03 / F-07: OAuth 連携 + 同意取得 + 初期セットアップ(Watch 登録・Pub/Sub 設定) |
| **A-2** | `IngestMailUseCase` | U1-01: Gmail Push 通知 → メッセージ取得 → `messages` 保存 → `classify_queue` 投入 |
| **A-3** | `ClassifyMailUseCase` | U1-02: ルールベース判定 → 不能なら Haiku 呼び出し → 結果保存 → `extract_queue`(recruitment 時)/ `notify_queue` + `calendar_queue` 両方(decision 時。services.md §3 参照)に enqueue |
| **A-4** | `ExtractCaseUseCase` | U2-01〜U2-04: 案件情報抽出 → schedules 配列化 → `cases`/`schedules` 保存 → `availability_queue` 投入 |
| **A-5** | `CheckAvailabilityUseCase` | U3-01〜U3-04: freeBusy 取得 → OverlapDetector → 結果集約 → `notify_queue` 投入 |
| **A-6** | `ComposeEntryDraftUseCase` | U4-01〜U4-04: PR 要素判定 → Few-shot 取得 → Haiku 生成 → Gmail 下書き作成 → `notify_queue` 投入 |
| **A-7** | `ManageCalendarUseCase` | U6-01〜U6-05: 仮予定登録(全候補)/ 確定更新 + 他削除 / 辞退削除 を一括対応 |
| **A-8** | `DetectAndDeclineConflictsUseCase` | U7-01〜U7-04: 決定後の重複検出 → 辞退下書き生成(Few-shot 含む) → 承認待ち通知 → 承認後送信 |
| **A-9** | `NotifyUserUseCase` | U5-01〜U5-06: LINE Flex Message 送信 / Postback 受信ハンドリング |
| **A-10** | `RotateGmailWatchUseCase` | U1-EC-03: Cron 起動、Watch 期限延長 |

---

## 3. インフラストラクチャ層(`crates/infrastructure`)— 外部 API / DB アダプタ

| # | コンポーネント | 責務 |
|---|--------------|------|
| **I-1** | `GmailAdapter` | `MailRepository` を実装。Gmail API REST 直叩き(`reqwest`)、Pub/Sub Watch 設定 |
| **I-2** | `GoogleCalendarAdapter` | `CalendarRepository` を実装。`freeBusy.query` 主体、events.* は本サービス作成イベントのみ |
| **I-3** | `LineMessagingAdapter` | `NotificationChannel` を実装。Webhook 署名検証、Reply / Push API、Flex Message |
| **I-4** | `AnthropicClient` | `LlmClient` を実装。Claude Haiku、Tool use(JSON schema 強制)、prompt cache |
| **I-5** | `D1Repositories` | 各 `*Repository` ドメイントレイトを `sqlx`(または `worker::D1`)で実装 |
| **I-6** | `KvStore` | KV ラッパー(冪等性キー、ルールキャッシュ、セッション) |
| **I-7** | `R2Storage` | R2 ラッパー(メール原文 30 日保管、Logpush 出力先、CSV 一時 URL[Phase 2]) |
| **I-8** | `QueueProducer` / `QueueConsumer` | Cloudflare Queues 抽象化(Q2=A により ステップ別 7 Queue) |
| **I-9** | `ClassificationRuleEngine` | U1-02 のルールベース分類(KV からルール取得、AND/OR/閾値評価) |
| **I-10** | `CryptoService` | F-09: AES-256-GCM 暗号化/復号、`key_id` ベースの世代管理 |
| **I-11** | `OAuth2Service` | F-03: `yup-oauth2` ラッパー、リフレッシュ自動化 |

---

## 4. プレゼンテーション層(`crates/presentation`)— Workers ハンドラ

| # | コンポーネント | 責務 |
|---|--------------|------|
| **P-1** | `LineWebhookHandler` | `POST /webhook/line` — 署名検証 → Postback / Message パース → A-9 へ委譲 |
| **P-2** | `PubSubWebhookHandler` | `POST /webhook/pubsub` — JWT 検証 → A-2 起動 |
| **P-3** | `OAuthCallbackHandler` | `GET /oauth/callback` — code 交換 → A-1 起動 |
| **P-4** | `OnboardingHandler` | `GET /onboard/start` — Bot からの遷移、OAuth URL 発行 |
| **P-5** | `QueueConsumerWorker` | Queues Consumer(各キューに 1 つずつ) |
| **P-6** | `CronWorker` | Cron Triggers ハンドラ(Watch 更新、メトリクス集計、コーパス収集) |
| **P-7** | `AdminApiHandler` | `*/admin/*` — Bearer 認証 → 運用コマンド(DLQ 再処理 / メトリクス取得 / ルール CRUD) |

---

## 5. 共有層(`crates/shared`)

| # | コンポーネント | 責務 |
|---|--------------|------|
| **S-1** | `Logger` / `MetricsRecorder` | F-06: 構造化ログ(`request_id`, `actor`, `action_source`)+ メトリクス(F-14)+ シークレット redact |
| **S-2** | `MessageCatalog` | F-08: ユーザー向け文言定数(`messages/ja.rs`)+ 専門用語禁止 lint 用辞書 |
| **S-3** | `ErrorClassifier` | `DomainError` から U2-EC-04 4 カテゴリへの分類 + 復旧ガイダンス生成 |
| **S-4** | `TestSupport`(test feature) | Q8=A: F-13(D-15〜D-18.5)ポートのモック実装(`MockMailRepository` / `MockCalendarRepository` / `MockNotificationChannel` / `MockLlmClient` / `MockOAuthExchanger`)、フィクスチャローダー |

---

## 6. コンポーネント数まとめ

| レイヤ | 主要コンポーネント数 |
|--------|---------------------|
| Domain(Entity / VO / Service / Port / Error) | 20(D-1〜D-19 + D-18.5 OAuthExchanger) |
| Application(UseCase) | 10 |
| Infrastructure(Adapter / Service) | 11 |
| Presentation(Handler / Worker) | 7 |
| Shared | 4 |
| **合計** | **52** |

> Repository トレイトファミリー(D-14)を 1 行に集約しているため、**実質的なドメイン抽象は約 31**(D-1〜D-19 + D-18.5 のうち D-14 をサブトレイト分割した場合の総数)。Q1=B「ユニットに合わせて柔軟」「過細化避ける」方針に沿い、関連性の高いリポジトリは同じトレイトファミリーにまとめてある。

詳細なメソッドシグネチャは `component-methods.md`、依存関係は `component-dependency.md` を参照。
