# Application Design — auto-mc-operation(統合ドキュメント)

本ドキュメントは Application Design ステージの **主成果物**。`components.md` / `component-methods.md` / `services.md` / `component-dependency.md` の内容をまとめ、**データモデル(D1 スキーマ)** と **API エンドポイント設計** を加えた完全版。

> **📑 ID 参照**: 本ドキュメントは複数 ID 体系を横断的に参照する。各 ID の正本(定義元)・主な参照先・命名規則は **`aidlc-docs/inception/id-index.md`** に集約。
> - `FR-N` / `NFR-N` / `SECURITY-NN` の正本は `inception/requirements/requirements.md`
> - `F-NN`(Foundation Story)/ `UN-NN`(Use Case Story)/ `P2-NN`(Phase 2)の正本は `inception/user-stories/stories.md`
> - `D-NN` / `A-N` / `I-NN` / `P-N` / `S-N`(設計コンポーネント)の正本は **同じフォルダの `components.md`**
> - `P1` / `P2`(ペルソナ)の正本は `inception/user-stories/personas.md`(プレゼンテーション層 `P-N` とは別物)

## 0. 設計原則(Plan セクション 3 の回答に準拠)

- **Q1 = B**: コンポーネント粒度は中粒度、ユニットに合わせて柔軟調整、過細化避ける(全 52、実質抽象 31)
- **Q2 = A**: 非同期通信は **ステップ別 Queue**(7 Queue + DLQ)
- **Q3 = C**: エラー型は **U2-EC-04 共通失敗ハンドリング 4 カテゴリ**(`Transient` / `Recoverable` / `DataIssue` / `Permanent`)を最上位、ドメイン別をネスト
- **Q4 = C**: API エンドポイントは **完全仕様**(URL + メソッド + 認証ヘッダ + レート制限 + バージョニング)
- **Q5 = B**: D1 スキーマは **全列 + 制約 + 主要 IDX + マイグレーション順序**
- **Q6 = B**: 認証は **Webhook 署名検証 + 管理 API Bearer + 将来 LIFF + JWT**(段階的)
- **Q7 = A**: トランザクション境界は **eventually consistent / saga**
- **Q8 = A**: テストモック境界は **F-11 抽象化トレイト境界**

---

## 1. C4 モデル

**P2 ペルソナ(AI 未経験者)配慮の設計反映**:
すべてのユーザー向け文言(エラー / 認可失敗 / フォールバック動線)は **`shared::MessageCatalog`(S-2)** に集約する。エラー時には「次に何のボタンをタップすればよいか」を MessageCatalog から具体ガイドとして取得し、LINE Flex Message に埋め込む(`personas.md` P2 の「専門用語ゼロ・具体ガイド必須」要件への準拠、F-08 メッセージング規約の延長)。

### 1.1 Context Diagram(システム全体と外部関係)

```mermaid
flowchart TB
    MC([👤 MC / コンパニオン<br/>P1: AI慣れ / P2: AI未経験])
    Office([事務所担当者<br/>メール送信元])
    Operator([🛠️ サービス運営者])

    System[[auto-mc-operation<br/>SaaS]]

    Gmail[(Gmail)]
    Cal[(Google Calendar)]
    LINE[(LINE Messaging API)]
    Anthropic[(Anthropic API<br/>Claude Haiku)]
    PubSub[(GCP Pub/Sub<br/>Gmail Watch)]

    MC <-- LINE Bot --> System
    MC <-- OAuth + 下書き編集 --> Gmail
    MC <-- 仮/確定予定 --> Cal
    Office -- 案件募集メール --> Gmail
    Operator -- 管理 API / アラート受信 --> System
    System -- メール取込 / 下書き作成 / 送信 --> Gmail
    System -- イベント CRUD --> Cal
    System -- 通知送受信 --> LINE
    System -- 分類・抽出・生成 --> Anthropic
    Gmail -- 新着通知 --> PubSub
    PubSub -- Push --> System

    style System fill:#4CAF50,stroke:#1B5E20,color:#fff
```

### 1.2 Container Diagram(Cloudflare 配置 — Worker のエントリポイント種別を明示)

> **重要**: Worker は **3 種類のエントリポイントハンドラ**(`fetch` / `queue` / `scheduled`)を 1 つのコードベースに持ち、起動経路ごとに異なるハンドラが呼ばれる。下図ではこれらを別コンテナとして描画。

```mermaid
flowchart TB
    subgraph EXT["🌐 外部世界"]
        MC([👤 MC<br/>P1 / P2 ペルソナ])
        Op([🛠️ サービス運営者])
        Office([事務所担当者])
        Gmail[(Gmail)]
        Cal[(Google Calendar)]
        LINE[(LINE Messaging API)]
        Anthropic[(Anthropic API<br/>Claude Haiku)]
        PubSub[(GCP Pub/Sub)]
    end

    subgraph WORKER["☁️ Cloudflare Workers(workers-rs / WASM、単一 crate)"]
        direction TB
        subgraph FH["📥 fetch handler<br/>HTTP 起動・URL 公開・外部からアクセス可"]
            P1[P-1 LineWebhook<br/>POST /webhook/line]
            P2[P-2 PubSubWebhook<br/>POST /webhook/pubsub]
            P3[P-3 OAuthCallback<br/>GET /oauth/callback]
            P4[P-4 Onboarding<br/>GET /onboard/start]
            P7[P-7 AdminApi<br/>/v1/admin/*]
        end

        subgraph QH["⚡ queue handler<br/>Queues 起動・URL 無し・外部からアクセス不可"]
            P5[P-5 QueueConsumer<br/>7 ステップ別 Consumer]
        end

        subgraph SH["⏰ scheduled handler<br/>Cron 起動・URL 無し・外部からアクセス不可"]
            P6[P-6 CronWorker<br/>Watch更新 / cleanup / archive]
        end
    end

    subgraph TRIG["⚡ 内部トリガー機構(Cloudflare 提供、URL 無し)"]
        Q[Queues × 7 + DLQ × 7<br/>classify / extract / availability /<br/>draft / notify / calendar / decline]
        Cron[Cron Triggers<br/>毎時 / 日次 / 週次]
    end

    subgraph STORAGE["💾 ストレージ・状態管理(Cloudflare)"]
        D1[(D1<br/>SQLite at edge<br/>14 テーブル)]
        KV[(KV<br/>冪等性キー /<br/>ルールキャッシュ)]
        R2[(R2<br/>メール原文 30 日 /<br/>監査ログ 12 ヶ月)]
        DO[Durable Objects<br/>per-user 状態保持<br/>hibernate 可能]
    end

    %% 外部 → fetch handler(HTTP 公開エンドポイント)
    LINE -- HTTP POST<br/>署名検証 --> P1
    PubSub -- HTTP POST<br/>JWT 検証 --> P2
    MC -- OAuth 同意<br/>後リダイレクト --> P3
    MC -- セットアップ開始 --> P4
    Op -- Bearer 認証 --> P7

    %% Office → Gmail → Pub/Sub の上流
    Office -- 案件募集メール --> Gmail
    Gmail -- 新着通知 --> PubSub

    %% fetch handler から Queues へ enqueue(内部経路)
    P1 -. enqueue .-> Q
    P2 -. enqueue .-> Q
    P3 -. enqueue .-> Q
    P7 -. enqueue<br/>(再投入) .-> Q

    %% Queues が Worker を起動(URL なし、内部機構)
    Q -- event-trigger<br/>(バッチ単位) --> P5

    %% queue handler が次ステージ Queues へ enqueue(自己循環)
    P5 -. enqueue<br/>次ステージ .-> Q

    %% Cron Triggers が scheduled handler を起動
    Cron -- scheduled<br/>(時刻起動) --> P6
    P6 -. enqueue<br/>(必要に応じ) .-> Q

    %% Workers → ストレージ(全種類のハンドラから利用、env binding)
    P1 <--> D1
    P2 <--> D1
    P3 <--> D1
    P4 <--> D1
    P5 <--> D1
    P5 <--> KV
    P5 <--> R2
    P5 <--> DO
    P6 <--> D1
    P6 <--> R2
    P7 <--> D1

    %% Workers → 外部 API(発信側)
    P1 -- 返信 / Push --> LINE
    P3 -- token 交換 --> Gmail
    P5 -- メール取得 / 下書き / 送信 --> Gmail
    P5 -- freeBusy / events --> Cal
    P5 -- Push / Reply --> LINE
    P5 -- 分類 / 抽出 / 生成 --> Anthropic
    P6 -- watch 更新 --> Gmail

    style WORKER fill:#FFE082,stroke:#F57F17,color:#000
    style FH fill:#C8E6C9,stroke:#2E7D32,color:#000
    style QH fill:#FFCCBC,stroke:#D84315,color:#000
    style SH fill:#E1BEE7,stroke:#6A1B9A,color:#000
    style TRIG fill:#FFF59D,stroke:#F57F17,color:#000
    style STORAGE fill:#B3E5FC,stroke:#0277BD,color:#000
    style EXT fill:#F5F5F5,stroke:#424242,color:#000
```

**読み方のコツ**:
- 🟢 緑(fetch handler): **インターネット経由で URL アクセスされる入口**
- 🟠 橙(queue handler): **URL を持たず、Cloudflare Queues が Cloudflare 内部機構で起動する**
- 🟣 紫(scheduled handler): **URL を持たず、Cloudflare Cron がスケジュール起動する**
- 黄色(内部トリガー機構): Queues / Cron は Cloudflare が提供する起動メカニズムで、これ自体は HTTP ではない
- 青(ストレージ): すべてのハンドラから env binding 経由でアクセス可能(設定で制限可)

### 1.3 実行モデル(エントリポイントの種別)

Worker は **イベントドリブンな短命 invocation のみ**で動作し、常駐プロセスは存在しない。3 種類のエントリポイント関数を 1 つのコードベース(`crates/presentation`)に持ち、起動経路により呼び出されるハンドラが切り替わる。

```rust
// 1. HTTP 起動 — 外部 URL からの fetch リクエスト
#[event(fetch)]
async fn fetch(req: Request, env: Env, ctx: Context) -> Result<Response, Error> { ... }

// 2. Queues 起動 — Cloudflare 内部機構からの起動(URL 無し)
#[event(queue)]
async fn queue(batch: MessageBatch, env: Env, ctx: Context) { ... }

// 3. Cron 起動 — Cloudflare スケジューラからの起動(URL 無し)
#[event(scheduled)]
async fn scheduled(event: ScheduledEvent, env: Env, ctx: Context) { ... }
```

**3 種ハンドラの比較**:

| ハンドラ | 起動経路 | URL 公開 | 外部到達可 | 担当 P-N | 主な責務 |
|---------|---------|---------|-----------|---------|---------|
| **fetch** | 外部 HTTP リクエスト | ✅ あり | ✅ 可能 | P-1 / P-2 / P-3 / P-4 / P-7 | Webhook 受信、OAuth Callback、管理 API |
| **queue** | Cloudflare Queues | ❌ 無し | ❌ 不可能 | P-5 | ステージ間の非同期処理(7 種 Queue) |
| **scheduled** | Cloudflare Cron Triggers | ❌ 無し | ❌ 不可能 | P-6 | 定期処理(Watch 更新、cleanup、archive) |

**Durable Objects は別カテゴリ**:
- 上記 3 ハンドラとは別に、**per-user 状態保持インスタンス** として Durable Objects を使用
- アイドル時 hibernate、必要時に自動再開(ステートフル、永続的だが料金は活動時のみ)
- 主な用途: Gmail Watch 状態管理、レート制限カウンタ、per-user 排他制御

### 1.4 ステージパイプライン(Queue 連鎖の俯瞰)

ユースケース処理が複数 stage を経由する様子を、Queue 連鎖の視点で俯瞰:

```mermaid
flowchart LR
    P2[P-2<br/>fetch handler<br/>POST /webhook/pubsub] --> Q1[classify_queue]
    P1[P-1<br/>fetch handler<br/>POST /webhook/line] --> Q1
    P1 --> Qcal[calendar_queue]
    P1 --> Qdec[decline_queue]

    Q1 -- event-trigger --> P5a[P-5 Consumer<br/>A-3 ClassifyMail]
    P5a --> Q2[extract_queue]
    P5a --> Q5[notify_queue]
    P5a --> Qcal

    Q2 -- event-trigger --> P5b[P-5 Consumer<br/>A-4 ExtractCase]
    P5b --> Q3[availability_queue]

    Q3 -- event-trigger --> P5c[P-5 Consumer<br/>A-5 CheckAvailability]
    P5c --> Q4[draft_queue]
    P5c --> Q5

    Q4 -- event-trigger --> P5d[P-5 Consumer<br/>A-6 ComposeDraft]
    P5d --> Q5

    Q5 -- event-trigger --> P5e[P-5 Consumer<br/>A-9 NotifyUser]
    P5e --> External_LINE[LINE Push 送信]

    Qcal -- event-trigger --> P5f[P-5 Consumer<br/>A-7 ManageCalendar]
    P5f --> Qdec

    Qdec -- event-trigger --> P5g[P-5 Consumer<br/>A-8 DetectAndDecline]
    P5g --> Q5

    P6[P-6<br/>scheduled handler] --> Q1
    P6 --> External_Gmail[Gmail Watch 更新]

    style P1 fill:#C8E6C9
    style P2 fill:#C8E6C9
    style P5a fill:#FFCCBC
    style P5b fill:#FFCCBC
    style P5c fill:#FFCCBC
    style P5d fill:#FFCCBC
    style P5e fill:#FFCCBC
    style P5f fill:#FFCCBC
    style P5g fill:#FFCCBC
    style P6 fill:#E1BEE7
    style Q1 fill:#FFF59D
    style Q2 fill:#FFF59D
    style Q3 fill:#FFF59D
    style Q4 fill:#FFF59D
    style Q5 fill:#FFF59D
    style Qcal fill:#FFF59D
    style Qdec fill:#FFF59D
```

**読み方**:
- 🟢 緑: fetch handler(外部 HTTP 入口)
- 🟠 橙: queue handler(P-5 が Queue 種別ごとに別ロジックを実行)
- 🟣 紫: scheduled handler(Cron 起動)
- 🟡 黄: Queues(URL 無し、Cloudflare 内部の配管)

P-5 は単一 Worker 関数だが、`batch.queue` の値(Queue 名)で対応するユースケース(A-3〜A-9)を切り替えて実行する。

### 1.5 Component Diagram

`component-dependency.md` セクション 1 を参照。DDD 5 レイヤと依存方向(presentation → application → domain ← infrastructure)を厳守、`cargo deny` で機械的検証。

---

## 2. コンポーネント概要

詳細は `components.md` を参照。サマリ:

| レイヤ | crate | 主要コンポーネント数 | 役割 |
|--------|-------|---------------------|------|
| domain | `crates/domain` | 19 | 純粋なビジネスロジック・トレイト・エラー型(外部依存ゼロ) |
| application | `crates/application` | 10(ユースケース) | オーケストレーション、saga、補償 |
| infrastructure | `crates/infrastructure` | 11 | 外部 API / DB / Queue / 暗号 アダプタ |
| presentation | `crates/presentation` | 7 | Workers ハンドラ |
| shared | `crates/shared` | 4 | ロガー / メッセージ / エラー分類 / テスト支援 |

メソッドシグネチャは `component-methods.md`、依存関係は `component-dependency.md`、サービス層オーケストレーションは `services.md`。

---

## 3. データモデル(D1 スキーマ)

Q5 = B により全列 + 制約 + 主要 IDX + マイグレーション順序を明記。**全 14 テーブル**(うち append-only 1: `consents`)。

> **📘 詳細リファレンス**: 本セクションは ER 図と DDL のみを掲載。**各カラムの目的・用途・どのユースケースが読み書きするか・データ保存方針** の詳細は **`data-model.md`** を参照。新規参画者・実装者はまず `data-model.md` を読むことを推奨。

### 3.1 ER 図

```mermaid
erDiagram
    users ||--o{ messages : "受信"
    users ||--o{ cases : "保有"
    users ||--o{ pr_corpus : "学習元"
    users ||--o{ decline_corpus : "学習元"
    users ||--o{ consents : "同意履歴"
    users ||--|| oauth_tokens : "保有"

    cases ||--|{ schedules : "候補スロット"
    cases ||--o{ entries : "エントリー"
    cases ||--o{ declines : "辞退"
    cases ||--o| calendar_events : "仮/確定登録"

    schedules ||--o| entries : "確定スロット"

    messages ||--o| cases : "抽出元"

    classification_rules }o--|| users : "user スコープのみ"
    office_patterns ||--o{ cases : "学習元書式"

    audit_logs }o--|| users : "操作主体"
```

### 3.2 マイグレーション順序

```
0001_init_users_consents_oauth.sql       -- ユーザー基盤(他テーブルの FK 元)
0002_init_messages_office_patterns.sql   -- メール取込基盤
0003_init_classification_rules.sql       -- 分類ルール基盤
0004_init_cases_schedules.sql            -- 案件・スロット
0005_init_entries_drafts.sql             -- エントリー
0006_init_declines.sql                   -- 辞退
0007_init_calendar_events.sql            -- カレンダー連携
0008_init_corpus.sql                     -- PR文・辞退文学習データ
0009_init_audit_logs.sql                 -- 監査
0010_indexes_extra.sql                   -- 追加インデックス
```

### 3.3 主要テーブル定義

> `created_at` / `updated_at` は全テーブルに `TEXT NOT NULL DEFAULT (datetime('now'))`。`id` は `TEXT PRIMARY KEY`(UUID v7)。

```sql
-- 0001 users
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    line_user_id TEXT NOT NULL UNIQUE,
    google_email TEXT NOT NULL,
    display_name TEXT,
    consent_version TEXT NOT NULL DEFAULT 'v1',
    travel_buffer_minutes INTEGER NOT NULL DEFAULT 60,  -- FR-3 移動時間バッファ(設定可能)
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE UNIQUE INDEX idx_users_line ON users(line_user_id);

-- 0001 oauth_tokens(F-09 暗号化、key_id は独立カラムで管理)
-- W6 修正: BLOB 内に key_id プレフィックスを含めず、独立カラム key_id を信頼の単一情報源とする
--           (検索効率と冗長排除のため (a) 案を採用)
CREATE TABLE oauth_tokens (
    user_id TEXT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    encrypted_refresh_token BLOB NOT NULL,   -- {nonce}:{ct+tag}(key_id は別カラム)
    key_id TEXT NOT NULL,                    -- 復号鍵世代の識別子(F-09 ローテーション対応)
    scope TEXT NOT NULL,                     -- "gmail.modify gmail.send calendar.events"
    expires_at TEXT,
    last_refreshed_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- 0001 consents(同意履歴、改竄防止のため append-only)
-- W7 修正: SQLite には append-only 制約がないため、UPDATE/DELETE をトリガーで拒否
CREATE TABLE consents (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    version TEXT NOT NULL,
    agreed_at TEXT NOT NULL,
    ip_hash TEXT,
    user_agent_hash TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_consents_user ON consents(user_id, agreed_at);

-- append-only を機械的に保証するトリガー(NFR-4 SECURITY-11 監査要件)
CREATE TRIGGER trg_consents_no_update BEFORE UPDATE ON consents
BEGIN SELECT RAISE(ABORT, 'consents is append-only'); END;
CREATE TRIGGER trg_consents_no_delete BEFORE DELETE ON consents
BEGIN SELECT RAISE(ABORT, 'consents is append-only'); END;

-- 0002 messages(取込メールメタ、本文は R2)
CREATE TABLE messages (
    id TEXT PRIMARY KEY,                     -- our id
    user_id TEXT NOT NULL REFERENCES users(id),
    gmail_message_id TEXT NOT NULL,
    gmail_thread_id TEXT NOT NULL,
    history_id TEXT NOT NULL,
    sender_domain TEXT NOT NULL,
    subject TEXT,
    received_at TEXT NOT NULL,
    classification TEXT,                     -- recruitment | decision | other | NULL(未分類)
    classified_by TEXT,                      -- rule | llm | NULL
    classification_confidence REAL,
    needs_review INTEGER NOT NULL DEFAULT 0,
    raw_blob_key TEXT,                       -- R2 のキー(30日で自動削除)
    raw_expires_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE UNIQUE INDEX idx_messages_gmail ON messages(user_id, gmail_message_id);
CREATE INDEX idx_messages_review ON messages(user_id, needs_review) WHERE needs_review = 1;

-- 0002 office_patterns
CREATE TABLE office_patterns (
    id TEXT PRIMARY KEY,
    sender_domain TEXT NOT NULL UNIQUE,
    office_name TEXT,
    pattern_data TEXT NOT NULL,              -- JSON: {sample_layouts, keywords, ...}
    success_count INTEGER NOT NULL DEFAULT 0,
    last_seen_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- 0003 classification_rules
CREATE TABLE classification_rules (
    id TEXT PRIMARY KEY,
    scope TEXT NOT NULL,                     -- 'global' | 'user'
    user_id TEXT REFERENCES users(id),       -- scope='user' のときのみ
    sender_domain_pattern TEXT,
    subject_keywords_json TEXT,
    body_keywords_any_json TEXT,
    body_keywords_all_json TEXT,
    target_label TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 50,
    enabled INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_rules_scope ON classification_rules(scope, user_id, enabled, priority);

-- 0004 cases
CREATE TABLE cases (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    source_message_id TEXT NOT NULL REFERENCES messages(id),
    office_id TEXT REFERENCES office_patterns(id) ON DELETE SET NULL,  -- W8 修正: 明示的 FK
    office_name TEXT NOT NULL,
    subject_name TEXT NOT NULL,
    location TEXT,
    compensation_text TEXT,
    compensation_amount INTEGER,             -- 抽出できた場合のみ
    deadline_at TEXT,
    pr_required INTEGER NOT NULL DEFAULT 0,
    other_conditions TEXT,
    extraction_warnings_json TEXT,           -- 欠落項目リスト
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | entered | confirmed | declined
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_cases_user_status ON cases(user_id, status, deadline_at);
CREATE INDEX idx_cases_user_office ON cases(user_id, office_name);

-- 0004 schedules
CREATE TABLE schedules (
    id TEXT PRIMARY KEY,
    case_id TEXT NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
    start_at TEXT NOT NULL,
    end_at TEXT,
    tz TEXT NOT NULL DEFAULT 'Asia/Tokyo',
    raw_text TEXT,
    confidence REAL NOT NULL DEFAULT 1.0,
    overlap_status TEXT,                     -- 最終判定: free | partial | full
    is_chosen INTEGER NOT NULL DEFAULT 0,    -- 確定後 1 つだけ 1
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_schedules_case ON schedules(case_id);
CREATE INDEX idx_schedules_time ON schedules(start_at, end_at);

-- 0005 entries
CREATE TABLE entries (
    id TEXT PRIMARY KEY,
    case_id TEXT NOT NULL REFERENCES cases(id),
    chosen_schedule_id TEXT REFERENCES schedules(id),
    draft_id TEXT,                           -- Gmail draft id
    pr_used INTEGER NOT NULL DEFAULT 0,
    submitted_at TEXT,                       -- ユーザー送信時刻(ユーザー操作)
    confirmed_at TEXT,
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | submitted | confirmed | declined
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_entries_case ON entries(case_id);
CREATE INDEX idx_entries_status ON entries(status);

-- 0006 declines
CREATE TABLE declines (
    id TEXT PRIMARY KEY,
    case_id TEXT NOT NULL REFERENCES cases(id),
    triggered_by_case TEXT NOT NULL REFERENCES cases(id),  -- 決定案件
    decline_draft TEXT NOT NULL,             -- 生成された辞退本文
    sent_message_id TEXT,
    status TEXT NOT NULL DEFAULT 'proposed', -- proposed | sent | failed
    sent_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_declines_case ON declines(case_id);

-- 0007 calendar_events
CREATE TABLE calendar_events (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    case_id TEXT NOT NULL REFERENCES cases(id),
    schedule_id TEXT NOT NULL REFERENCES schedules(id),
    google_event_id TEXT NOT NULL,
    state TEXT NOT NULL,                     -- tentative | confirmed | deleted
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_calevents_case ON calendar_events(case_id);
CREATE INDEX idx_calevents_user ON calendar_events(user_id, state);

-- 0008 pr_corpus / decline_corpus(本人帰属、永続)
CREATE TABLE pr_corpus (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    source_message_id TEXT,
    body TEXT NOT NULL,
    office_name TEXT,
    case_kind TEXT,                          -- "MC" | "コンパニオン" 等の推定
    char_length INTEGER NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_pr_user_office ON pr_corpus(user_id, office_name);

CREATE TABLE decline_corpus (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    source_message_id TEXT,
    body TEXT NOT NULL,
    office_name TEXT,
    char_length INTEGER NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_decline_user_office ON decline_corpus(user_id, office_name);

-- 0009 audit_logs
CREATE TABLE audit_logs (
    id TEXT PRIMARY KEY,
    user_id TEXT,                            -- system 操作のときは NULL
    actor TEXT NOT NULL,                     -- 'user:{id}' | 'system' | 'admin:{id}'
    action_source TEXT NOT NULL,             -- 'line_postback' | 'cron' | 'pubsub_push' | 'admin_api' | 'internal'
    action TEXT NOT NULL,                    -- e.g. 'classify_mail.success' | 'decline.send.failed'
    target_kind TEXT,                        -- 'message' | 'case' | 'entry' | 'decline' | ...
    target_id TEXT,
    payload_json TEXT,
    result TEXT NOT NULL,                    -- 'ok' | 'error'
    error_kind TEXT,                         -- 'transient' | 'recoverable' | 'data_issue' | 'permanent' | NULL
    correlation_id TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_audit_user_time ON audit_logs(user_id, created_at);
CREATE INDEX idx_audit_action ON audit_logs(action, created_at);
```

---

## 4. API エンドポイント設計(Q4 = C 完全仕様)

### 4.1 URL 体系・バージョニング

- ベース URL: `https://api.<base-domain>`(production)/ `https://staging.<base-domain>`(staging)
- `<base-domain>` の実値は **F-10(ドメイン取得・DNS 設定)で確定** する(本ドキュメント時点ではプレースホルダ)
- バージョニングは **URL プレフィックス**: `/v1/` を全エンドポイントに付与(MVP は v1 のみ、breaking change 時に `/v2/`)
- Webhook は `/webhook/...`(バージョンなし、外部サービスの仕様に追従)

### 4.2 認証(Q6 = B 段階的)

| エンドポイント種別 | 認証方式 | 詳細 |
|------------------|---------|------|
| **公開 Webhook** | 署名検証 | LINE: `X-Line-Signature`(HMAC-SHA256) / GCP Pub/Sub: JWT(Google が署名)/ OAuth Callback: `state` パラメータ(CSRF 対策 + 開始リクエスト紐付け、有効期間付き、サーバー側 KV で照合) |
| **管理 API** | Bearer トークン | `Authorization: Bearer <token>`、トークンは Wrangler secrets で管理、IP 許可リスト併用、**`user_id` パラメータの権限チェック必須**(運用者ミスによる他ユーザーデータ操作を防止 / SECURITY-08 BOLA 防止) |
| **ユーザー API** | (Phase 2) JWT | LIFF + JWT、IDOR 防止のため `user_id` をトークンクレームから取得し、リソース所有確認(SECURITY-08) |

### 4.3 レート制限

| 種別 | 制限 | 実装 |
|------|------|------|
| LINE Webhook | LINE 側準拠(120 req/min/bot) | バーストは Queues で吸収 |
| Pub/Sub | Google 側準拠 | 同上 |
| 管理 API | 60 req/min/token | KV カウンタ |
| ユーザー API(Phase 2) | 30 req/min/user | Durable Object でユーザー毎カウンタ |
| 外部 API への発信(Anthropic / Google / LINE) | 各サービスのクォータ準拠 | Queues + 指数バックオフ |

**LLM 呼び出し設計目標(NFR-7「月額 500 円/ユーザー」達成のため)**:
- **目標上限: ≤ 4 回 / 案件**(分類 1 + 抽出 1 + PR 文生成 0〜1 + 辞退文生成 0〜1)
- 1 ユーザー × 月 600 案件相当(20 件/日 × 30 日)を想定すると、**最大 2400 回 / ユーザー / 月**
- **設計目標値の枠組み**(Functional Design で精緻化):
  - **入力トークン上限**: 平均 ≤ 1,500 / 呼び出し(プロンプト + メール本文 + Few-shot 例 圧縮)
  - **出力トークン上限**: 平均 ≤ 400 / 呼び出し(JSON Schema 強制で簡潔化)
  - **プロンプトキャッシュヒット率**: ≥ 70%(システムプロンプト + 静的 Few-shot を `cache_control` でキャッシュ、Claude 公式仕様で入力トークン課金 90% off)
  - **コスト試算式**(Claude Haiku の参考レート $1/M input、$5/M output、為替 ¥150/$ 想定):
    - キャッシュなし: `2400 × 1500 × $1/M + 2400 × 400 × $5/M = $3.6 + $4.8 = $8.4 ≒ ¥1,260/月`(目標未達)
    - **キャッシュ 70% ヒット**: `2400 × (1500 × 0.3 + 1500 × 0.7 × 0.1) × $1/M + 2400 × 400 × $5/M = $1.6 + $4.8 = $6.4 ≒ ¥960/月`(なお目標 ¥500 未達)
    - **追加施策**(Functional Design): ルール率 50% でさらに半減 → `¥480/月` で目標達成見込み
- 詳細な閾値・実測キャリブレーションは **Functional Design ステージ(per-unit, Construction)** で確定。F-14 監視で実測値が想定を超えた場合のアラートも仕込む

### 4.4 エンドポイント一覧

| URL | Method | 認証 | レート | リクエスト | レスポンス | 担当 Handler |
|-----|--------|------|--------|----------|----------|-------------|
| `/webhook/line` | POST | LINE 署名 | LINE 仕様 | LINE Event | 200 (空) | P-1 |
| `/webhook/pubsub` | POST | Pub/Sub JWT | — | `{message: {data, attributes}}` | 204 | P-2 |
| `/onboard/start` | GET | state 検証 | — | `?line_user_id=&state=` | 302 → Google OAuth | P-4 |
| `/oauth/callback` | GET | state 検証 | — | `?code=&state=` | 200 完了画面(LINE 戻りボタン) | P-3 |
| `/v1/admin/dlq/{queue}/replay` | POST | Bearer | 60/min | `{message_ids: []}` | 200 `{replayed_count}` | P-7 |
| `/v1/admin/metrics` | GET | Bearer | 60/min | — | 200 メトリクス JSON | P-7 |
| `/v1/admin/rules` | GET / POST / PUT / DELETE | Bearer | 60/min | rule JSON | 200 | P-7 |
| `/v1/admin/users/{id}/reauth` | POST | Bearer | 60/min | — | 200 `{reauth_url}` | P-7 |

### 4.5 Webhook ペイロード受信時の冪等性

すべての Webhook ハンドラは:
1. リクエスト ID を計算(LINE: webhook event id / Pub/Sub: message id)
2. KV `idempotency:{kind}:{id}` を確認
3. 既処理ならスキップ、未処理なら 24 時間有効でマーク + 処理開始

---

## 4.6 STRIDE 脅威モデリング(NFR-4 SECURITY-11 準拠)

要件 NFR-4 SECURITY-11「設計フェーズで脅威モデリング(STRIDE)を実施」に対する、本ステージでの **配置方針**:

- 本 Application Design では **STRIDE の枠組みと適用対象** を明示するに留め、各ユニット(Foundation / メール取込 / 抽出 / カレンダー / 下書き / 通知辞退 / 監視運用)単位での **詳細実施は Functional Design ステージ** に委譲
- 各ユニットの Functional Design 成果物に `threat-model.md` を必須化(`docs/threat-models/<unit>.md` に配置)
- 適用対象の主要トラスト境界(MVP):
  | 境界 | 隣接コンポーネント | 主に検討する STRIDE カテゴリ |
  |------|-------------------|----------------------------|
  | 公開インターネット ↔ Workers Webhook | 外部 → P-1 / P-2 / P-3 / P-4 / P-7 | Spoofing, Tampering, Repudiation, DoS |
  | Workers ↔ 外部 API | I-1〜I-4 → Gmail / Calendar / LINE / Anthropic | Information Disclosure, Tampering, DoS |
  | Workers ↔ D1 / KV / R2 | application → I-5/I-6/I-7 | Tampering, Information Disclosure |
  | ユーザー ↔ LINE Bot | U → P-1 経由 | Spoofing(なりすまし)、Information Disclosure |

## 5. エラーハンドリングアーキテクチャ(Q3 = C)

### 5.1 エラー型階層

```rust
#[derive(thiserror::Error, Debug)]
pub enum DomainError {
    #[error("transient: {source:?}")]
    Transient { source: TransientCause, retryable: bool },

    #[error("recoverable: requires user action {action:?}")]
    Recoverable { action: UserAction, message_key: MessageKey },

    #[error("data issue at field {field}: {raw_value:?}")]
    DataIssue { field: &'static str, raw_value: Option<String> },

    #[error("permanent: {reason}")]
    Permanent { reason: String, audit_required: bool },
}

pub enum TransientCause {
    UpstreamTimeout(ServiceName),
    RateLimited(ServiceName),
    NetworkError(String),
    DatabaseLocked,
}

pub enum UserAction {
    ReauthGoogle,
    ReauthLine,
    ReviewMessage(MessageId),
    ApproveDecline(Vec<DraftId>),
}
```

### 5.2 カテゴリ別ハンドリング

| カテゴリ | Queue 動作 | LINE 通知 | 監査ログ | 例 |
|---------|-----------|----------|---------|-----|
| Transient | 自動リトライ(3 回 / 指数バックオフ)→ 失敗時 DLQ | 連続失敗時のみ運用者通知 | `error_kind=transient` | LLM タイムアウト、API レート制限 |
| Recoverable | 即時 DLQ なし、`needs_user_action` フラグ立て | ユーザーに復旧ボタン送付 | `error_kind=recoverable` | OAuth 失効 |
| DataIssue | リトライしない、`needs_review` フラグ立て | ユーザーに原文確認依頼 | `error_kind=data_issue` | 必須項目欠落 |
| Permanent | 即時 DLQ + 運用者通知 | 控えめ(混乱回避) | `error_kind=permanent`, `audit_required=true` | プロンプト不備、仕様外メール |

### 5.3 失敗復旧フロー(`docs/operations.md` 連携)

`U2-EC-04 AC-3` で言及。各カテゴリの「アラート種別 → 確認手順 → 復旧コマンド」を明文化。

---

## 6. テスト戦略との接続

- **F-13 + Q8 = A**: ドメインポート(`MailRepository` / `CalendarRepository` / `NotificationChannel` / `LlmClient`)に対するモックを `crates/shared/test_support` に集約
- **PBT 対象**:
  - `OverlapDetector::check`(`proptest` で乱数生成した時間範囲の対称性・推移性)
  - シリアライズ往復(`Case`/`Schedule`/`Entry` の JSON ↔ Rust 型)
  - CSV エンコード(Phase 2 で復活時)
- **ゴールデンセット**: `tests/fixtures/extraction/` に βテスター由来の匿名化メール 30〜100 件、F1 スコア閾値 0.85 以上

---

## 7. 構成方針との整合性

| 構成方針 | 反映先 |
|---------|--------|
| 拡張性方針(F-11) | D-15〜D-18 ポートが Outlook / Apple Calendar / Slack 拡張時に新規アダプタ実装で対応 |
| ドキュメント集約方針 | `docs/architecture.md`(本ドキュメントの要約)/ `docs/api-design.md`(セクション 4 抜粋)/ `docs/data-model.md`(セクション 3 抜粋) |
| システム構成図(F-12) | セクション 1.1 / 1.2 がその実体、`docs/system-overview.md` に複製 |
| テスト・監視のスコープ(F-13/F-14) | 本ドキュメントは枠組みまで、詳細は Functional Design / Build and Test |

---

## 7.4 システム構成図の粒度向上(2026-05-04)

レビュー時のフィードバック「Worker が全イベントを一身に受けているように見えて混乱」「コンテナ図の粒度を細かく」に対応。

主な変更:
- §1.2 Container Diagram を全面書き直し:
  - **fetch handler / queue handler / scheduled handler** を Worker 内の別コンテナとして可視化
  - 内部トリガー機構(Queues / Cron)を独立サブグラフに分離
  - URL 公開有無・外部到達可否を色分けで表現
- §1.3 実行モデル(エントリポイント種別)を新設:
  - 3 種ハンドラの Rust コード例(`#[event(fetch)]` / `#[event(queue)]` / `#[event(scheduled)]`)
  - URL 公開・外部到達可否・担当 P-N の比較表
  - Durable Objects は別カテゴリと明記
- §1.4 ステージパイプライン(Queue 連鎖の俯瞰)を新設:
  - fetch / queue / scheduled の各ハンドラがどう連鎖するかを Mermaid で図示
  - P-5 が Queue 種別ごとに A-3〜A-9 を切り替える点を明記

## 7.3 ID 体系の整理(2026-05-04)

レビューでの「A-N / F-NN が突然出てきても定義元が分からない」指摘を受け、横断 ID インデックス **`aidlc-docs/inception/id-index.md`** を新設。各文書冒頭に「📑 ID 参照」ヘッダを追加し、正本(定義元)が機械的に辿れるようにした。

主な構成:
- 全プレフィックス(FR, NFR, SECURITY, F, U, P2, P1/P2 ペルソナ, D, A, I, P, S, Q, Cl, C/W/S/N)を 1 表で網羅、正本ドキュメントと主な参照先を明示
- 命名規則(2 桁ゼロ埋め推奨、EC 中置語、N.5 番号挿入)を明文化
- クロスリファレンス表(Story → A-N、FR → A-N → テーブル、F-NN → 設計箇所、ペルソナ → 配慮箇所)
- 設計変更時の影響範囲チェック手順

## 7.2 レビュー反映履歴(PR #3 issue #4368912387 / 2026-05-04 — AI レビュー /review design モード)

| 項目 | 対応 |
|------|------|
| **🔴 D1 テーブル数 13 vs 14** | §8 完了基準 / §3 冒頭で「14 テーブル(append-only 1: consents)」に統一 |
| **🟡 P-4 OnboardingHandler URL 不整合** | `/oauth/start` → **`/onboard/start`** に統一(`components.md` の責務記述と整合) |
| **🟡 P-4 が依存図から欠落** | `component-dependency.md` `subgraph PR` に `P4[OnboardingHandler]` 追加 |
| **🟡 Calendar 操作名表記揺れ** | `DeleteAll` / `DeleteAllByCase` → **`DeleteAllByCase`** に統一、`slot_id` → **`chosen_slot`** に統一 |
| **🟢 ConsentRepo 列が依存マトリクスに無い** | `ConsentRepo` 列を追加、A-1 / A-2 / A-4 / A-6 に ✓ |
| **🟢 OAuth2 認可コード交換のドメインポート未抽象化** | **D-18.5 `OAuthExchanger`** を新規ポートとして追加(F-11 拡張性方針との整合)、A-1 が利用 |
| **🟢 NFR-7 コスト試算枠組みが手薄** | §4.3 にコスト試算式・トークン上限・キャッシュヒット率の枠組みを明記(月額 ¥480 試算で目標達成見込み) |
| **💡 STRIDE 配置** | §4.6 を新設し「枠組みは Application Design、詳細は Functional Design ステージ per-unit」と委譲方針明記 |
| **💡 F-3 → FR-3 表記** | `users.travel_buffer_minutes` の SQL コメントを FR-3 に修正(設計-要件トレーサビリティ向上) |

## 7.1 レビュー反映履歴(PR #3 issue #4365084235 / 2026-05-03)

レビュー結果を以下のように反映:

| 項目 | 対応 |
|------|------|
| **追加要望**: 主要テーブル定義の用途が不明 | `data-model.md` を新規作成し、14 テーブルの全カラムについて **目的・用途・読み書きユースケース・データ保存方針** を網羅(本ドキュメント Section 3 から詳細を移管) |
| **C1**: A-3(decision)流れ先 3 説 | `services.md` で `notify_queue` + `calendar_queue` の **両 enqueue 方針** に統一、シーケンス図と Queue 構成表を一致 |
| **C2**: 存在しない F-15 参照 | `components.md` の F-15 → F-13 に修正 |
| **C3**: A-10 RotateGmailWatch 設計片手落ち | `component-methods.md` にシグネチャ追加、`component-dependency.md` 依存マトリクスに行追加、`services.md` に Cron 起動シーケンス追加 |
| **W4**: コンポーネント数表記不一致 | `components.md` L3 を「約 51(実質抽象 30)」に修正、`application-design.md` と統一 |
| **W5**: A-9 NotifyUser CaseRepo 依存漏れ | 依存マトリクスに ✓ 追加(case_id を渡され Flex Message 用にサマリ取得する設計) |
| **W6**: oauth_tokens key_id 二重保存 | BLOB プレフィックスから `key_id` を除去、独立カラムを単一情報源に |
| **W7**: consents append-only 機械的保証なし | UPDATE/DELETE を拒否する SQLite トリガーを追加 |
| **W8**: cases.office_id FK 未宣言 | `REFERENCES office_patterns(id) ON DELETE SET NULL` を明示 |
| **S9**: state パラメータ説明 | 「`state`(CSRF 対策 + 開始紐付け、有効期間付き、KV 照合)」に整理 |
| **S10**: QueueEnvelope.retry_count 意図 | 「ステップ別 Queue 連鎖時の通算試行回数」用と注記 |
| **S11**: ベース URL F-10 参照 | `<base-domain>` の実値が F-10 で確定する旨を明記 |
| **S12**: Cloud Pub/Sub / GCP Pub/Sub 表記揺れ | 「GCP Pub/Sub」に統一 |
| **S13**: 管理 API BOLA 防止 | 4.2 認証表に `user_id` パラメータ権限チェック必須を明記 |
| **S14**: LLM 呼び出し回数上限 | 4.3 レート制限節に「≤ 4 回 / 案件、最大 2400 回 / ユーザー / 月」を設計目標として記載 |
| 上流ドキュメント(`requirements.md`)陳腐化 | F8 を Phase 2 へ移行注記、4.2 MVP 含めないものに追記、トレーサビリティ表 Q20 / FR-8 にも反映 |
| P2 ペルソナ反映 | セクション 1.0 に「ユーザー向け文言は `MessageCatalog`(S-2)集約、エラー時に具体ガイド埋め込み」を追記 |

## 8. 完了基準

本 Application Design ステージで以下が確定:

- ✅ 5 レイヤ・約 52 コンポーネントの責務とインターフェース
- ✅ 10 ユースケースのコマンド型・Result 型
- ✅ 7 ステップ別 Queue + DLQ + saga 補償パターン
- ✅ **14 D1 テーブル**(うち append-only 1: `consents`)のスキーマ + 主要インデックス + マイグレーション順序
- ✅ Webhook / 管理 API / Phase 2 ユーザー API のエンドポイント仕様
- ✅ U2-EC-04 統合の 4 カテゴリエラー型階層

詳細な業務ロジック・閾値・タイムアウト値・バリデーションは **Functional Design ステージ(per-unit, Construction)** で確定する。
