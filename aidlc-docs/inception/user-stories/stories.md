# ユーザーストーリー — auto-mc-operation

このドキュメントは User Stories の本体です。
`personas.md` の **プライマリペルソナ 2 名**(P1: AI に慣れた MC = 佐藤 美咲さん / P2: AI 未経験の MC = 中村 ゆかりさん)を主語として記述します。

---

## 構成方針(Plan セクション 8 準拠 + P2 ペルソナ反映 + レビュー反映)
- **Foundation Epic** + **MVP Use Case Epics(8 個)** + **Phase 2 Epic 概略**
- 粒度: 最小ユーザーアクション = 1 Story、各 Story に 3〜5 個の Given-When-Then 受入条件
- メタ: Priority(MoSCoW) + Size(S/M/L) + Type(User Story / Enabler Story / Edge Case)
- トレーサビリティ: 各 Story に Implements(FR/NFR / 個別 SECURITY-XX)を付与
- エッジケースは独立 Story(`-EC-` プレフィックス)
- **ユーザー向けメッセージング(LINE 通知・同意画面・エラー文言)は P2(AI 未経験者)を最低基準として設計** — 専門用語禁止、失敗時は具体的アクションを必ず明示
- **拡張性方針**: メール / カレンダー / 通知の各外部サービスは **抽象化レイヤ(ポート/トレイト)で差し替え可能** に設計し、Phase 3 で Outlook / Apple Calendar / Slack 等への拡張を **新規アダプタの追加だけ** で実現できるようにする(F-11 で詳細化)
- **ドキュメント集約方針**: AI 用ガイダンスファイル(`CLAUDE.md` / `AGENTS.md` / `.claude/`)を除き、すべてのドキュメントは **`docs/` 配下に集約** する(`docs/architecture.md` / `docs/environments.md` / `docs/cloudflare-cost.md` / `docs/secrets-management.md` / `docs/messaging-conventions.md` / `docs/system-overview.md` 等)
- **全体システム構成図**: Foundation Epic 完了時点で `docs/system-overview.md` に Mermaid 形式の構成図を作成する(F-12 で詳細化)

---

# 🟦 Foundation Epic — 基盤・最初に開発

ユーザーが直接体験しない Enabler Stories だが、後続のユーザーストーリーすべての前提となる。
**Foundation 配下は Type = Enabler Story、ペルソナは MC を含むがアクター視点は「サービス開発者・運用者として」記述する**(Enabler の慣習)。

---

### Story F-01: プロジェクトのディレクトリ構成・開発ルール整備(DDD レイヤ + AI 階層依存ガード)

**As a** サービス開発者
**I want** Cargo workspace を **DDD(ドメイン駆動設計)** のレイヤ構造で分離し、AI コーディングエージェントが階層依存ルールを破れない仕組みを整備する
**so that** 後続の開発が一貫した依存方向(ドメイン非依存・インフラ依存可)のもとで進み、AI 生成コードでも構造が崩れない

**Priority**: MUST
**Size**: L
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (DDD レイヤ workspace 構造)**:
  - **Given** リポジトリ直下に `Cargo.toml` がない状態
  - **When** Cargo workspace を初期化する
  - **Then** `crates/` 配下に以下の DDD レイヤ別 crate が分離される:
    - `domain`: エンティティ・値オブジェクト・ドメインサービス・リポジトリトレイト(外部依存ゼロ)
    - `application`: ユースケース(domain のみに依存、infra は DI で注入)
    - `infrastructure`: 外部 API(Gmail / Calendar / LINE / Anthropic)・DB アダプタ(domain のリポジトリトレイトを実装)
    - `presentation`: Cloudflare Workers ハンドラ(application を呼び出す薄いレイヤ)
    - `shared`: ユーティリティ・エラー型(domain にも依存させない最小)
- **AC-2 (依存方向の機械的ガード)**:
  - **Given** DDD レイヤ間の依存ルール(presentation → application → domain ← infrastructure)
  - **When** 開発者または AI エージェントがコードを書く
  - **Then** `cargo deny` または `cargo-modules` の依存グラフ検査で逆方向依存(例: domain が infrastructure に依存)があれば **CI で fail** する
- **AC-3 (AI 向けドキュメント)**:
  - **Given** AI コーディングエージェントが本リポジトリで作業する
  - **When** タスクを開始する
  - **Then** ルートに `ARCHITECTURE.md`(または `.agent/architecture.md`)があり、レイヤ責務・依存方向・禁止事項を読まずにコードを書けない仕組み(CLAUDE.md / AGENTS.md などからの参照)
- **AC-4 (lint / format)**:
  - **Given** Rust コードベース
  - **When** `cargo fmt --all -- --check` および `cargo clippy --all-targets -- -D warnings` を実行する
  - **Then** すべてのコードがフォーマット・lint 警告 0 で通る
- **AC-5 (コミット規約)**:
  - **Given** CLAUDE.md で定義された日本語コミットメッセージ規約
  - **When** 開発者がコミットを作成する
  - **Then** type: メッセージ形式が CI でチェックされ、不適合は merge できない
- **AC-6 (CI/CD 基盤の整備 = GitHub Actions)**:
  - **Given** GitHub リポジトリ
  - **When** Pull Request を作成する
  - **Then** GitHub Actions が以下を自動実行する: `cargo fmt --check` / `cargo clippy -D warnings` / `cargo test --all` / `cargo audit` / commitlint / 依存方向グラフ検査(AC-2)
  - **Then** main へのマージは CI 全項目グリーンが必須

**Traceability**:
- Implements: 7.1 言語・フレームワーク, NFR-4 SECURITY-10(依存管理), NFR-6(テスト・PBT 基盤)

---

### Story F-02: Cloudflare 基盤(Workers / D1 / KV / R2 / Queues / Cron / Durable Objects)のセットアップ

**As a** サービス開発者
**I want** Cloudflare アカウントと Wrangler 設定を整備し、3 環境の役割と権限・課金プランを明確にした上で各リソースを Infrastructure-as-Code で定義する
**so that** MVP 段階から運用までの環境の役割が明確で、コスト・権限の混乱なく開発・運用できる

**Priority**: MUST
**Size**: L
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (Wrangler 設定)**:
  - **Given** `wrangler.toml` が未作成
  - **When** Wrangler でプロジェクトを初期化する
  - **Then** `wrangler.toml` に Workers / D1 binding / KV namespace / R2 bucket / Queues producer-consumer / Cron triggers / Durable Object class binding がすべて定義される
- **AC-2 (3 環境の役割と権限を明記)**:
  - **Given** local / staging / production の 3 環境
  - **When** デプロイ・アクセス・操作を行う
  - **Then** 以下の役割と権限が `docs/environments.md` に明記され、Cloudflare API トークンが各環境に分離される:
    | 環境 | 用途 | デプロイ権限 | データアクセス | LINE/Gmail/Calendar 接続先 |
    |------|------|-------------|---------------|----------------------------|
    | **local** | 開発者本人のローカル開発・単体テスト | 開発者本人(`wrangler dev`) | 開発者本人のみ・モックデータ | モック / 開発者の個人 LINE グループ |
    | **staging** | 統合テスト・ステージング検証 | 開発者(CI 経由) | 開発者のみ(βテスターは含めない) | 専用 staging LINE Bot / staging Google アプリ |
    | **production** | 本番運用(αテスター・βテスター・本番ユーザー全員ここ) | 開発者(CI 経由・main ブランチ承認後) | エンドユーザー(自データのみ) + 運用者(管理 API のみ) | production LINE Bot / production Google アプリ |
  - **Note (環境名 / β テスター配置の経緯)**: 開発者ローカル環境を `dev` ではなく **`local`** と明示することで、Cloudflare デプロイを伴わない「ローカル限定」を強調。**β テスターは production 環境** で受け入れることで、本番と同じ通信経路・通知チャネルでテストでき、staging 構成と本番構成のズレに起因する事故を避ける(staging はあくまで開発者の最終確認環境)
- **AC-2.5 (Cloudflare リソースの用途明示)**:
  - **Given** `wrangler.toml` で定義する各 Cloudflare リソース
  - **When** 設計レビュー
  - **Then** 各リソースの用途を `docs/architecture.md` に明記する:
    | リソース | 用途 |
    |---------|------|
    | **Workers** | アプリ本体(リクエスト処理・Queue Consumer・Cron) |
    | **D1 (SQLite)** | リレーショナル DB(案件・エントリー・カレンダー連携・PR 文学習データ) |
    | **KV** | セッション・冪等性キー・ルールベース分類の allowlist/blocklist など短期 / 高頻度参照のキーバリュー |
    | **R2** | メール本文の原文(短期保管)・添付・**Logpush の出力先**(監査ログ保存) |
    | **Queues** | 非同期処理(メール取り込み → 分類 → 抽出 → 通知の各ステップを疎結合化、バックプレッシャ制御、リトライ / DLQ) |
    | **Cron Triggers** | 定期処理(Gmail Watch 再登録、PR 文学習バッチ、トークン期限監視 等) |
    | **Durable Objects** | per-user 強整合状態(Gmail Watch 履歴 ID、レート制限カウンタ 等) |
  - **Note**: **R2 は Logpush セットアップ時に手動で bucket を作成する必要があります**(自動作成されない)。このため `wrangler.toml` で R2 binding を明示するのと、Logpush ジョブで対象 bucket を指定する 2 ステップが必要です
- **AC-3 (Cloudflare 課金プラン明確化)**:
  - **Given** MVP 段階の処理量(1 ユーザー × 20 件/日 × LLM 数 req)
  - **When** Cloudflare の利用量を見積もる
  - **Then** 以下が `docs/cloudflare-cost.md` に明記される:
    - **Workers Free**(無料)では **Queues が使えない**ため、**Workers Paid プラン($5/月 ≒ ¥800/月)が必須**
    - D1 / KV / R2 / Cron / Durable Objects は MVP 規模では無料枠内に収まる
    - NFR-7 月額 500 円/ユーザー目標 vs. 固定費 $5/月の関係を明示(ブレイクイーブン: 約 2 ユーザーから黒字)
- **AC-4 (D1 マイグレーション基盤の整備)**:
  - **Given** D1 データベースが空
  - **When** マイグレーション CLI(`wrangler d1 migrations apply`)を実行する
  - **Then** マイグレーションファイル(`migrations/0001_init.sql` 等)が適用される基盤が動作する
  - **Note**: **個別テーブルの定義(列・FK・インデックス)は Application Design / Functional Design ステージで決定**(本 Story はマイグレーション実行の枠組みのみを整える)
- **AC-5 (暗号化)**:
  - **Given** D1 / KV / R2 のデフォルト設定
  - **When** リソースを作成する
  - **Then** 暗号化が有効(SECURITY-01 準拠)で、TLS 1.2+ の通信のみ受け付ける

**Traceability**:
- Implements: 7.2 ホスティング, 7.3 データ永続化, NFR-4 SECURITY-01, NFR-7(コスト)

---

### Story F-03: Google OAuth(Gmail / Calendar)認可フローの構築

**As a** MC(セットアップ時の利用者として)
**I want** 自分の Google アカウントを安全にサービスと連携したい
**so that** Gmail と Google カレンダーへの読み書きをサービスに代行してもらえる

**Priority**: MUST
**Size**: L
**Type**: User Story(セットアップフロー)

**Acceptance Criteria**:
- **AC-1 (OAuth フロー開始 / ボタンタップで完結)**:
  - **Given** ユーザーが LINE 公式アカウントを友達追加した状態
  - **When** Bot が初回ようこそメッセージで「Google と連携する」**ボタン**(Quick Reply / Action Button)を表示する
  - **Then** ユーザーが**ボタンをタップするだけ**でセットアップ用 URL に遷移する(文字入力は一切要求しない)
- **AC-1.5 (OAuth 画面の URL ドメインの位置づけ)**:
  - **Given** OAuth フロー
  - **When** ユーザーがボタンをタップする
  - **Then** 1) Bot が返す URL は **サービス側ドメイン**(例: `https://auth.example.com/start`)で、その内部から `accounts.google.com/o/oauth2/...` へ 302 リダイレクトする(Google 側ドメインで同意画面が表示される)、2) **Callback URL** はサービス側ドメインで、Google Cloud Console の OAuth クライアント設定にあらかじめ登録する必要がある
- **AC-2 (スコープ — Gmail / Calendar の read+write を明示)**:
  - **Given** Google OAuth 同意画面
  - **When** ユーザーが同意ボタンを押す
  - **Then** 以下のスコープが付与される(read+write を明示):
    - **Gmail (write)**: `https://www.googleapis.com/auth/gmail.modify`(下書き作成・送信・既読化に必要)
    - **Gmail (send)**: `https://www.googleapis.com/auth/gmail.send`(エントリー / 辞退メール送信)
    - **Gmail (readonly)**: 上記 modify スコープに包含(別個では不要)
    - **Calendar (write)**: `https://www.googleapis.com/auth/calendar.events`(仮予定登録 / 確定更新 / 削除)
    - **Calendar (readonly)**: 上記 events スコープに包含(別個では不要)
  - **Then** これらの read+write 権限は **F-07 の同意画面でユーザーに明示**(平易な言葉で「メールを読み書き / カレンダー予定を作成・更新・削除」と表示)し、必須スコープであることを伝える
- **AC-3 (リフレッシュトークン保管 + 暗号鍵管理は F-09 に委譲)**:
  - **Given** ユーザーが同意した
  - **When** リフレッシュトークンを取得する
  - **Then** トークンは D1 に AES-GCM で暗号化して保存される
  - **Note**: **暗号鍵の管理方法(保管場所・ローテーション戦略)は F-09(暗号鍵・シークレット管理)で先に決定** され、本 Story はその仕組みを利用する側
- **AC-4 (セットアップ完了通知)**:
  - **Given** OAuth 完了
  - **When** Bot がコールバックを受信する
  - **Then** ユーザーに LINE で「セットアップ完了」+ プライバシーポリシー同意ボタン(F-07 へ繋がる)が届く
- **AC-5 (リフレッシュ)**:
  - **Given** 有効期限切れアクセストークン
  - **When** リフレッシュ処理が走る
  - **Then** `yup-oauth2` で自動更新され、失敗時はユーザーに LINE で**「もう一度 Google と連携」ボタン**を表示する
- **AC-6 (P2 配慮: ガイド付き誘導 + ボタンのみで完結)**:
  - **Given** AI 未経験ユーザー(P2)が初回セットアップ中
  - **When** Bot がセットアップを案内する
  - **Then** 1) すべての操作は**ボタンタップのみ**で完結(文字入力なし)、2) 「Google と連携を始める」と平易表記(「OAuth 認可」とは書かない)、3) 想定所要時間(例: 約 1 分)を併記、4) 完了後に LINE に戻る方法を画面内のボタンとして明示

**Traceability**:
- Implements: FR-1, FR-3, FR-4, FR-6, FR-7, NFR-4 SECURITY-01 / SECURITY-06 / SECURITY-08, NFR-5, F-08(P2 配慮), F-09(暗号鍵管理)

---

### Story F-04: LINE Messaging API Bot 連携セットアップ

**As a** サービス開発者
**I want** LINE 公式 Bot を staging / production の 2 環境分作成し、Webhook を Cloudflare Workers に向け、署名検証を含めた基盤を整える
**so that** ユーザーとの双方向コミュニケーション(通知 + 承認/却下のボタン操作)が安全に行える

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (Bot 作成 — 環境別)**:
  - **Given** LINE Developers Console
  - **When** Messaging API チャネルを作成する
  - **Then** **staging / production の 2 環境分** Bot を作成(誤通知防止のため)。dev は専用 Bot を作らず、開発者個人の LINE グループ(またはローカルログ出力)に向ける
  - **Then** 各環境の Channel access token / Channel secret は対応する環境の Wrangler secrets に登録される
- **AC-2 (Webhook 設定 — 仮置き、全体設計は別ステージ)**:
  - **Given** デプロイ済み Workers の Webhook エンドポイント(例: `POST /webhook/line`)
  - **When** LINE Console で Webhook URL を設定する
  - **Then** ユーザーのメッセージ・ポストバックが Workers に POST される
  - **Note**: 本 Story は **LINE Webhook の受け口を作る最小実装**。**API エンドポイントの全体設計(URL 体系・認証ヘッダ・レート制限・バージョニング)は Application Design ステージで実施** する
- **AC-3 (署名検証 — 仕組みの説明込み)**:
  - **Given** LINE からの POST リクエスト
  - **When** Workers がリクエストを受信する
  - **Then** `X-Line-Signature` ヘッダの値を、Channel secret と body から計算した値と一致するか確認し、不一致は 401 を返す(SECURITY-05)
  - **Note (用語解説)**: 「**HMAC-SHA256**」とは、SHA-256 ハッシュ関数を使って **共有秘密鍵(Channel secret)+ メッセージ本文** から偽造困難な署名を作る仕組み。LINE 側で署名を生成 → ヘッダに添付 → サービス側で同じ計算をして一致するか確認することで、リクエストが本物の LINE から来たことを検証する。**一致 ≠ ハッシュが同じ ではなく、共有秘密鍵を持っていないと作れない署名**である点が重要
- **AC-4 (返信送信)**:
  - **Given** 受信した event の `reply_token`
  - **When** Bot が返信を送信する
  - **Then** LINE Messaging API の reply エンドポイントに正しく POST され、200 が返る
- **AC-5 (P2 配慮: ユーザー操作はボタンのみで完結)**:
  - **Given** ユーザーへの設定変更や承認/却下のリクエスト
  - **When** Bot がインタラクションを設計する
  - **Then** **テキスト入力を要求しない**設計とする(LINE Quick Reply / Postback Action / Flex Message のボタンを多用)。例外は将来「請求 CSV 2026-04」のような明確なコマンドのみで、それも候補ボタン化を検討する

**Traceability**:
- Implements: FR-5, NFR-4 SECURITY-05, F-08(P2 配慮)

---

### Story F-05: Anthropic API クライアント(Claude Haiku)の整備 + 二段分類への対応

**As a** サービス開発者
**I want** Anthropic API クライアントを `reqwest` で実装し、API キーをローテーション可能な設計で管理する
**so that** 分類・抽出・PR 文生成が統一インタフェースで Claude Haiku を呼び出せ、かつ運用上のキー漏洩リスクを最小化できる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (リクエスト型)**:
  - **Given** `serde` で定義された Anthropic Messages API のリクエスト/レスポンス型
  - **When** クライアント関数を呼び出す
  - **Then** モデル名(`claude-haiku-4-5-20251001`)、システムプロンプト、ユーザーメッセージ、最大トークン数を引数で渡せる
- **AC-2 (タイムアウト)**:
  - **Given** Anthropic API 呼び出し
  - **When** 25 秒以内にレスポンスが返らない
  - **Then** `AbortSignal` で中断し、エラーを返す(NFR-1)
- **AC-3 (リトライ)**:
  - **Given** 5xx またはレート制限(429)
  - **When** 最初の呼び出しが失敗する
  - **Then** 指数バックオフで最大 3 回リトライする
- **AC-4 (API キー管理 + ローテーション可能設計)**:
  - **Given** Anthropic API キー
  - **When** Workers から呼び出す
  - **Then** キーは Wrangler secrets(F-09 と連携)から取得され、ログ出力では自動マスクされる(SECURITY-03 / SECURITY-09)
  - **Then** **キーローテーション戦略**:
    1. 環境変数を `ANTHROPIC_API_KEY_PRIMARY` と `ANTHROPIC_API_KEY_SECONDARY` の **2 本** 構成にする
    2. ローテーション時は: SECONDARY に新キーを設定 → デプロイ → SECONDARY を PRIMARY に昇格 → 旧 PRIMARY を SECONDARY 削除 → デプロイ
    3. 上記により **無停止でローテーション** でき、緊急時(漏洩疑い)も SECONDARY を即無効化できる
- **AC-5 (二段分類への対応 — ルールベース → Haiku)**:
  - **Given** メール分類は **ルールベース第 1 段 + Claude Haiku 第 2 段** のパイプライン(U1-02 で詳細化)
  - **When** 本クライアントレイヤを呼び出す
  - **Then** **ルールベースで判定可能だった場合は Anthropic API を呼ばない** インタフェースになっている(`ClassificationDecision::RuleBased(label)` または `ClassificationDecision::NeedsLLM(prompt)` のような Result 型を返す)
  - **Note**: これにより NFR-7 月額 500 円目標達成に必要な LLM 呼び出し削減を、コードの構造から強制する

**Traceability**:
- Implements: FR-1, FR-2, FR-4, NFR-1, NFR-7, NFR-4 SECURITY-03 / SECURITY-09, F-09(キー管理)

---

### Story F-06: 構造化ロギング・監査ログ基盤

**As a** サービス開発者
**I want** すべての処理に構造化ログ(timestamp / request_id / user_id / level / message)を付与し、監査ログは別ストアに永続化する
**so that** 障害解析・セキュリティ監査・利用状況把握が可能になる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (構造化ログ + 操作者追跡)**:
  - **Given** Workers のリクエストハンドラ
  - **When** リクエストを受け付ける
  - **Then** `request_id` を生成し、以降のすべてのログに含まれる(NFR-8)
  - **Then** 監査ログには **`actor`(操作者)** と **`action_source`(発生元)** を必ず付与する:
    - `actor` の値: `user:<user_id>`(エンドユーザー操作) / `system`(自動処理 / Cron / Queue Consumer) / `admin:<admin_id>`(運用者操作)
    - `action_source` の値: `line_postback` / `line_message` / `cron_trigger` / `pubsub_push` / `admin_api` / `internal` 等
  - **Then** 後から「いつ・誰が・どの経路で」その操作を起動したかを完全追跡できる
- **AC-2 (PII / シークレット除外)**:
  - **Given** メール本文・OAuth トークン・API キー等の機密情報
  - **When** ログ出力関数を呼ぶ
  - **Then** 機密情報フィールドは自動的にマスクされる(SECURITY-03)
- **AC-3 (監査ログの永続化)**:
  - **Given** 案件分類・エントリー送信・辞退送信・カレンダー操作・LINE Bot 操作
  - **When** いずれかの操作が成功または失敗する
  - **Then** 監査ログ(`actor` / `action_source` / `target` / `result` / `timestamp` を含む)として R2 に Logpush 経由で送出され、12 ヶ月保管される(NFR-5)
- **AC-4 (アクセスログ)**:
  - **Given** Workers / Webhook エンドポイント
  - **When** リクエストが到達する
  - **Then** Cloudflare のアクセスログが Logpush で R2 / SIEM に送出される(SECURITY-02)

**Traceability**:
- Implements: NFR-8, NFR-4 SECURITY-02 / SECURITY-03, NFR-5

---

### Story F-08: ユーザー向けメッセージング規約の整備(P2 対応)

**As a** サービス開発者
**I want** ユーザーに表示するすべての文言(LINE 通知 / 同意画面 / エラー / セットアップ案内)に統一規約を設ける
**so that** AI 未経験のペルソナ P2 でも迷わず使え、P1 にとっても丁寧で信頼できる体験になる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (用語の言い換え辞書 + AI への遵守強制)**:
  - **Given** ユーザー向け文言を **メッセージ定数ファイル**(例: `crates/presentation/src/messages/ja.rs`)に集約する設計
  - **When** 開発者または AI コーディングエージェントが文言を書く
  - **Then** 「OAuth 認可」「API キー」「LLM」「トークン」「エンドポイント」等の専門用語が使われていないかを `cargo deny`/カスタム lint で **CI で機械的に検出** し、含まれていればビルド失敗とする
  - **Then** 言い換え辞書(専門用語 → 平易な言い換え対応表)は `docs/messaging-conventions.md` に明記され、CLAUDE.md / AGENTS.md / `.agent/` から参照される
  - **Then (AI が遵守する仕組み)**: AI コーディングエージェントは作業開始時に上記ドキュメントを読まないとタスクを進められない仕掛け(CLAUDE.md / pre-commit hook で警告)とする
  - **Note (用語解説)**: ここでの「メッセージ定数化」とは、ユーザー向け文言をコード本文に直書きせず別ファイルに集約する習慣のこと(将来の多言語対応・一括レビュー・lint 適用が容易になる)。**「i18n」(internationalization の略)はチーム内で意味共有済みの開発用語として、ストーリー / アーキテクチャ文書では普通に使ってよい**(ただしユーザー向け文言には登場させない)
- **AC-2 (失敗時の次アクション必須化)**:
  - **Given** エラー通知メッセージ
  - **When** メッセージを生成する
  - **Then** 「次に何をすればよいか」を **具体的なボタンまたは文言** として必ず含める(例: 「LINE で『もう一度 Google と連携』ボタンを押してください」)
- **AC-3 (LINE 内完結の優先)**:
  - **Given** ユーザー操作フロー
  - **When** 設計する
  - **Then** 可能な限り **LINE トーク内のボタン操作で完結** させ、外部 URL に飛ばす場合は理由と所要時間を事前告知する
- **AC-4 (読みやすさ)**:
  - **Given** すべてのユーザー向け文言
  - **When** レビューする
  - **Then** 中学生が読んで意味が分かるレベル(難読漢字・カタカナ専門語を最小化)

**Traceability**:
- Implements: NFR-8(エラー通知の平易化), 関連: P2 ペルソナ全 AC

---

### Story F-07: プライバシーポリシー・同意管理基盤

**As a** MC(初回利用者として)
**I want** プライバシーポリシーを読み、明示的に同意した上でサービスを使い始めたい
**so that** メール本文や個人情報の取扱いについて納得した状態で利用できる(個情法・電気通信事業法対応)

**Priority**: MUST
**Size**: M
**Type**: User Story(セットアップフロー)

**Acceptance Criteria**:
- **AC-1 (ポリシー提示)**:
  - **Given** Google OAuth 完了直後
  - **When** Bot がセットアップ完了通知を返す
  - **Then** プライバシーポリシー(利用目的・データ保存方針・第三者提供 [Anthropic 海外移転含む])を含む URL が同梱される
- **AC-2 (明示同意)**:
  - **Given** ポリシー画面のチェックボックス
  - **When** ユーザーが同意して送信する
  - **Then** 同意レコード(user_id / version / agreed_at)が D1 の `consents` テーブルに保存される
- **AC-3 (未同意の制御)**:
  - **Given** ユーザーが未同意状態
  - **When** Push 通知でメールが到達する
  - **Then** メール処理は実行されず、LINE で同意を促す通知のみ送る
- **AC-4 (ポリシー更新時の再同意)**:
  - **Given** ポリシーの version が更新された
  - **When** 既存ユーザーの次回アクション時
  - **Then** 再同意画面が表示され、同意完了まで処理を停止する
- **AC-5 (P2 配慮: 3 行サマリ)**:
  - **Given** AI 未経験ユーザー(P2)が同意画面を開く
  - **When** ポリシー本文を読む前
  - **Then** **3 行サマリ**(「① あなたの Gmail とカレンダーを Bot が見ます ② AI(Claude)に内容を渡して整理します ③ 重要な情報は短期間だけ預かります」のような平易な日本語)+ 「詳細を見る」展開リンクで提示する

**Traceability**:
- Implements: NFR-5(プライバシー・同意管理), F-08(P2 配慮)

---

### Story F-09: 暗号鍵・シークレットの管理基盤(F-03 / F-05 の前提)

**As a** サービス開発者
**I want** OAuth リフレッシュトークン暗号化用の鍵 / Anthropic API キー / LINE Channel secret 等のシークレットを **保管場所・アクセス経路・ローテーション戦略** を統一的に定義する
**so that** F-03(OAuth)F-04(LINE)F-05(Anthropic)各 Story が「鍵をどう持つか」をその都度設計せずに済み、漏洩リスクを最小化できる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (シークレット分類 — すべて 2 本構成 / key-id 付き)**:
  - **Given** サービスが扱うシークレットを以下に分類する
  - **When** 設計レビュー
  - **Then** `docs/secrets-management.md` に以下が明記される(**全シークレットを `_PRIMARY` / `_SECONDARY` の 2 本構成**にし、表記の整合を取る):
    | 種類 | 保管場所(変数名) | アクセス経路 | ローテーション頻度 |
    |------|-----------------|-------------|-------------------|
    | Anthropic API キー | Wrangler secrets(`ANTHROPIC_API_KEY_PRIMARY` / `ANTHROPIC_API_KEY_SECONDARY`) | env binding | 90 日 / 漏洩疑い時即時 |
    | LINE Channel access token | Wrangler secrets(`LINE_TOKEN_PRIMARY` / `LINE_TOKEN_SECONDARY`、環境別) | env binding | 90 日 |
    | LINE Channel secret(Webhook 署名検証用) | Wrangler secrets(`LINE_CHANNEL_SECRET_PRIMARY` / `LINE_CHANNEL_SECRET_SECONDARY`) | env binding | LINE 側ローテに追従 |
    | Google OAuth Client Secret | Wrangler secrets(`GOOGLE_CLIENT_SECRET_PRIMARY` / `GOOGLE_CLIENT_SECRET_SECONDARY`) | env binding | 180 日 / Google Cloud で発行 |
    | OAuth リフレッシュトークン暗号化鍵 | Wrangler secrets(`TOKEN_ENC_KEY_PRIMARY` / `TOKEN_ENC_KEY_SECONDARY`、各鍵に **`key-id`** を付与) | env binding | 180 日 |
- **AC-2 (Cloudflare ベストプラクティスへの準拠)**:
  - **Given** Cloudflare 公式が提供するシークレット交換のベストプラクティス
  - **When** 設計を行う
  - **Then** 以下を採用する:
    - **Wrangler Secrets** を主要保管場所とする(平文で `.env` / リポジトリに置かない)
    - **環境変数として binding** し、コード内では `env.ANTHROPIC_API_KEY_PRIMARY` 等で参照(直接ハードコード禁止)
    - シークレット更新は **`wrangler secret put` コマンド経由のみ**(GitHub Actions の OIDC 連携で CI からも更新可)
    - **Secret Store**(Cloudflare の集中管理機能)が将来 GA したら移行を検討(MVP 段階では Wrangler Secrets で十分)
- **AC-3 (リフレッシュトークン暗号化)**:
  - **Given** D1 に保管する OAuth リフレッシュトークン
  - **When** 永続化する
  - **Then** **AES-256-GCM** で暗号化され、暗号鍵は `TOKEN_ENC_KEY_PRIMARY`(Wrangler secrets)から取得する
  - **Then** ノンスはレコードごとにランダム生成し、`{key_id}:{nonce}:{ciphertext}:{tag}` 形式で保存(`key_id` でどの世代の鍵かを識別、ローテーション時の復号トライに使用)
- **AC-4 (鍵ローテーション戦略 — 全シークレット共通)**:
  - **Given** すべてのシークレットに `_PRIMARY` / `_SECONDARY` の 2 本構成
  - **When** ローテーションを実施する
  - **Then** 以下の手順で **無停止移行**:
    1. `_SECONDARY` に新キーを設定 → デプロイ
    2. 暗号化用途の鍵は `key_id` 付きでレコードに新世代を記録、復号は両鍵を試行
    3. 全アクティブセッションが新鍵で動くのを確認後、`_SECONDARY` を `_PRIMARY` に昇格 → 旧鍵を `_SECONDARY` から削除
    4. 暗号化済みレコードはバックグラウンドで新鍵に再暗号化(マイグレーション Job)
- **AC-4 (シークレットの非ログ化)**:
  - **Given** ロガー / エラーレポート
  - **When** シークレットを含む値が誤って渡される
  - **Then** F-06 のマスキング機構によりログには `***` で出力される(SECURITY-03)
- **AC-5 (アクセス権限の最小化)**:
  - **Given** Cloudflare API トークン
  - **When** CI / 開発者が利用する
  - **Then** 環境別・操作別に分離(staging deploy 用、production deploy 用、read-only 監視用 等)、本番デプロイは main ブランチからのみ可能(SECURITY-06)

**Traceability**:
- Implements: NFR-4 SECURITY-01 / SECURITY-03 / SECURITY-06 / SECURITY-09, F-03(リフレッシュトークン)/ F-05(Anthropic キー)/ F-04(LINE secret)

---

### Story F-10: ドメイン取得・DNS 設定(OAuth Callback / API エンドポイント用)

**As a** サービス開発者
**I want** サービス用のドメインを取得し、Cloudflare DNS / Workers Routes を設定する
**so that** Google OAuth Callback URL や LINE Webhook URL を安定したドメインで提供でき、本番運用で URL を変更しなくて済む

**Priority**: MUST
**Size**: S
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (ドメイン取得)**:
  - **Given** プロジェクト用に独立したドメインが必要
  - **When** ドメインを取得する
  - **Then** サービス名に対応した独自ドメイン(例: `auto-mc-operation.com` または短縮形)を取得し、Cloudflare の DNS で管理する
- **AC-2 (環境別サブドメイン)**:
  - **Given** local / staging / production の 3 環境(F-02 AC-2)
  - **When** Workers Routes を設定する
  - **Then** 以下のサブドメインで分離される:
    - `local` 環境: ローカル `wrangler dev`(ドメイン不要)
    - `staging` 環境: `staging.<base-domain>`
    - `production` 環境: `<base-domain>`(または `api.<base-domain>`)
- **AC-3 (TLS / HSTS)**:
  - **Given** 取得したドメイン
  - **When** Cloudflare で proxy 有効化
  - **Then** TLS 終端は Cloudflare で自動取得(Let's Encrypt 互換)、HSTS は `max-age=31536000; includeSubDomains`(NFR-4 SECURITY-04 準拠)

**Traceability**:
- Implements: NFR-4 SECURITY-04, F-03(OAuth Callback)/ F-04(LINE Webhook)

---

### Story F-11: 外部サービスの抽象化レイヤ(将来拡張用ポート/トレイト)

**As a** サービス開発者
**I want** メール / カレンダー / 通知の各外部サービスを抽象化したトレイト(ポート)で扱う
**so that** Phase 3 で Outlook / Apple Calendar / Slack 等への拡張時に **新規アダプタ実装の追加だけ** で対応でき、ビジネスロジック側を変更しなくて済む

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (ドメイン層のトレイト定義)**:
  - **Given** `crates/domain` レイヤ
  - **When** 設計する
  - **Then** 以下のリポジトリ / サービストレイトを定義する(MVP では Google / LINE 実装のみ):
    - `MailRepository`: `fetch_new_messages` / `create_draft` / `send_message` / `get_thread` 等(Gmail / Outlook / IMAP に共通の操作)
    - `CalendarRepository`: `list_events` / `create_event` / `update_event` / `delete_event` 等(Google Calendar / Outlook Calendar / Apple Calendar に共通)
    - `NotificationChannel`: `send_message` / `send_action_buttons` / `verify_signature` 等(LINE / Slack / Email 通知に共通)
- **AC-2 (infrastructure 層に各アダプタ)**:
  - **Given** `crates/infrastructure` レイヤ
  - **When** 実装する
  - **Then** MVP では `gmail.rs` / `google_calendar.rs` / `line.rs` の 3 アダプタを実装し、それぞれが上記トレイトを satisfy する
- **AC-3 (将来追加の容易性)**:
  - **Given** Phase 3 で Outlook 対応を追加するシナリオ
  - **When** 新規アダプタ `outlook.rs` を `crates/infrastructure` に追加
  - **Then** **ビジネスロジック(domain / application)を一切変更せず**、設定で利用アダプタを切り替えられる(`Box<dyn MailRepository>` の DI)
- **AC-4 (テスト容易性)**:
  - **Given** 上記トレイト
  - **When** 単体テスト
  - **Then** モック実装(`MockMailRepository` 等)を `crates/shared/test_support` に置き、ビジネスロジックを外部 API なしでテストできる

**Traceability**:
- Implements: 7.1 言語・フレームワーク, 4.2 Phase 3 拡張ユーザー, NFR-2 スケーラビリティ

---

### Story F-12: 全体システム構成図の整備

**As a** サービス開発者・新規参画者
**I want** Foundation Epic 完了時点で全体のシステム構成が一目で分かるドキュメントを持つ
**so that** 後続フェーズ(Application Design / Code Generation)・新メンバーオンボーディング・運用時の障害解析で参照できる

**Priority**: MUST
**Size**: S
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (構成図の作成)**:
  - **Given** Foundation Epic の F-01〜F-11 が完了
  - **When** ドキュメントを整備
  - **Then** `docs/system-overview.md` に **Mermaid 形式** の以下の図が含まれる:
    - **C4 Context 図**(MC ユーザー / 事務所 / Google / Anthropic / LINE / サービス本体 の関係)
    - **C4 Container 図**(Cloudflare Workers / D1 / KV / R2 / Queues / Durable Objects の構成)
    - **データフロー図**(メール受信 → 分類 → 抽出 → 通知 → 承認 → 送信 → カレンダー反映)
- **AC-2 (DDD レイヤ図)**:
  - **Given** F-01 で定義した DDD レイヤ
  - **When** 図示
  - **Then** `domain` / `application` / `infrastructure` / `presentation` / `shared` の依存方向と F-11 のトレイトとの関係が明示される
- **AC-3 (運用視点)**:
  - **Given** 環境別構成(F-02 AC-2)
  - **When** 図示
  - **Then** local / staging / production それぞれの接続先(LINE Bot / Google アプリ / Anthropic キー)が明示される

**Traceability**:
- Implements: 全 Foundation Story(F-01〜F-11)の俯瞰

---

# 🟩 MVP Epic — Use Case 1: メール受信〜分類(F1)

---

### Story U1-01: Gmail に着信したメールが自動でシステムに取り込まれる

**As a** MC
**I want** 案件募集メールが届いたら自動的にシステムが受信する
**so that** メールを自分でチェックしなくても処理パイプラインが起動する

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (Watch 登録)**:
  - **Given** OAuth 認可済みユーザー
  - **When** Cron Trigger が日次で `users.watch` を呼び出す
  - **Then** Gmail Pub/Sub Watch が更新され、有効期限内に維持される
- **AC-2 (Push 受信)**:
  - **Given** 新着メールが Gmail に到達
  - **When** Pub/Sub から Workers の Webhook に通知が届く
  - **Then** Workers は `historyId` を取得し、対応するメッセージを `users.messages.get` で取り込む
- **AC-3 (Queue 投入)**:
  - **Given** 取得したメッセージ
  - **When** 取り込みが成功する
  - **Then** `messages` テーブルに最低限のメタデータが保存され、Cloudflare Queues に「分類タスク」が enqueue される
- **AC-4 (GCP Pub/Sub の自動セットアップ)**:
  - **Given** ユーザー初回セットアップ時
  - **When** F-03 OAuth 完了直後
  - **Then** **システムが自動で**以下を行う:
    1. ユーザー個別のサービスアカウント or 共有のサービスアカウントで GCP Pub/Sub トピック / サブスクリプションを確保(共有を推奨)
    2. Gmail API `users.watch` を呼び、トピックに通知設定
    3. Pub/Sub サブスクリプションの Push エンドポイントを Workers の Webhook URL に設定
  - **Note**: GCP Pub/Sub の **トピック作成・IAM 設定はサービス運営者側で初期セットアップが必要**(プロジェクトレベルでの 1 回作業)。各ユーザーの `users.watch` 呼び出しは自動

**Traceability**:
- Implements: FR-1(Push 通知部分), F-03

---

### Story U1-02: 取り込まれたメールが「ルールベース → Claude Haiku」の二段で 3 値分類される

**As a** MC
**I want** 受信メールが「案件募集 / 決定連絡 / その他」に自動分類される
**so that** 自分が見るべきメールだけを優先して扱え、LLM コストも最小化できる(NFR-7 月額 500 円目標)

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (第 1 段: ルールベース分類 — βテストでチューニング可能な構造)**:
  - **Given** Queue にエンキューされた「分類タスク」
  - **When** Consumer が処理を開始する
  - **Then** **まず Anthropic API を呼ばずに** ルールベース分類を試みる
  - **Then** ルールは **データとして外部化**(ハードコードしない、KV / D1 で管理)し、運用者が βテストフィードバックに応じて **追加・削除・閾値調整** できる構造とする:
    - **ルール構造**: `{ scope: "global" | "user", source_domain_pattern: string, subject_keywords: string[], body_keywords_any: string[], body_keywords_all: string[], target_label: "recruitment" | "decision" | "other", priority: number }`
    - **AND / OR 条件**: `subject_keywords` と `body_keywords_all` は AND マッチ、`body_keywords_any` は OR マッチ、組み合わせ可能
    - **scope**: `global`(全ユーザー共通の事務所ルール) / `user`(個人の追加事務所・例外ルール)
  - **Then** ルールベースで判定確定したメールは **その時点で結果を返し、Anthropic API を呼ばない**
- **AC-1.5 (登録済み事務所のスコープ)**:
  - **Given** 「登録済み事務所ドメイン」リスト
  - **When** ルール参照
  - **Then** **全ユーザー共通の `global` 事務所マスタ + ユーザー個別の `user` 追加事務所** の 2 段持ち:
    - `global` 事務所マスタはサービス運営者が βテスト・運用を通して育てる(全ユーザーが恩恵を受ける)
    - `user` 個別事務所は自分専用(他ユーザーには見えない、プライバシー)
    - 判定時はまず `user` を参照、ヒットしなければ `global` を参照
- **AC-2 (第 2 段: Haiku フォールバック)**:
  - **Given** ルールベースで判定不能だった(`Indeterminate`)メール
  - **When** Consumer が処理を続ける
  - **Then** Claude Haiku に件名・本文(先頭 N 文字)・送信元ドメインを渡し、`{recruitment | decision | other}` のいずれかが返る
  - **Then** 信頼度スコアが閾値(例 0.6)未満 → ラベルを `other` + `needs_review = true` にフォールバック
- **AC-3 (ルールの自動拡張)**:
  - **Given** Haiku 第 2 段で `recruitment` と確定したメール
  - **When** その送信元ドメインを集計
  - **Then** **複数回 `recruitment` が出たドメインを allowlist 候補として運用者に提示** (将来的にルールベース第 1 段へ昇格させ、LLM 呼び出しを削減)
- **AC-4 (永続化 + コスト記録)**:
  - **Given** 分類結果
  - **When** 保存処理が走る
  - **Then** `messages.classification` 更新 + `audit_logs` に `{classified_by: "rule" | "llm", input_tokens, output_tokens}` を記録
  - **Then** ルールベース判定率(月次)を集計可能(NFR-7 のコスト改善トラッキング)

**Traceability**:
- Implements: FR-1, NFR-1, NFR-7, F-05(二段分類インタフェース)

---

### Story U1-03: プリフィルタにより明らかな非対象メールは LLM 前で除外される

**As a** MC
**I want** プロモーションメール等は LLM で判定する前にスキップされる
**so that** 不要な LLM コストとレイテンシが発生しない(NFR-7 月額 500 円目標)

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (送信元ドメインフィルタ)**:
  - **Given** 既知の非案件ドメイン(`*-noreply@google.com` 等)
  - **When** Push 通知でメールが到達する
  - **Then** LLM 呼び出しをスキップし、`other` として記録する
- **AC-2 (Gmail ラベル尊重)**:
  - **Given** Gmail のスパム / プロモーション ラベル
  - **When** 取り込み時にラベルを確認する
  - **Then** これらのラベル付きは LLM 処理対象外とする
- **AC-3 (フィルタの再学習可能性)**:
  - **Given** ユーザーが LINE で「これは案件です」と訂正
  - **When** 訂正イベントが届く
  - **Then** その送信元はフィルタから除外され、以降は LLM 処理される

**Traceability**:
- Implements: FR-1, NFR-7

---

### Story U1-EC-01: [Edge] 分類 LLM がタイムアウトした場合に DLQ に退避される

**As a** MC
**I want** LLM 呼び出しが失敗してもメールが失われない
**so that** 復旧後に再処理でき、案件機会を取りこぼさない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (リトライ)**:
  - **Given** Anthropic API が 25 秒タイムアウト
  - **When** Consumer が失敗
  - **Then** Queues の自動リトライで最大 3 回再試行される
- **AC-2 (DLQ)**:
  - **Given** リトライがすべて失敗
  - **When** 最終失敗が記録される
  - **Then** Dead Letter Queue に移送され、運用者に LINE 通知が届く
- **AC-3 (DLQ からの復旧フロー)**:
  - **Given** DLQ に滞留しているメッセージ
  - **When** 運用者が原因(LLM 障害 / プロンプト不備 / OAuth 失効 等)を特定して修正
  - **Then** **明示的な再処理コマンド**(管理 API / 運用 CLI)で該当メッセージを Queue に再投入できる
  - **Then** ユーザーには「処理を再開しました(対象 N 件)」と LINE で通知
  - **Note**: **失敗復旧フロー全体を `docs/operations.md` に明文化**(「いつ通知が来るか」「どこを見ればいいか」「再処理コマンドの実行手順」を含む)

**Traceability**:
- Implements: FR-1, NFR-1, NFR-8

---

### Story U1-EC-02: [Edge] Push 通知の重複が冪等性キーで防止される

**As a** MC
**I want** 同じメールが二重処理されない
**so that** 重複した LINE 通知や仮予定が作られない

**Priority**: **SHOULD**(α テスト 1 ユーザー段階では実害が少ないが、β テスト以降は MUST)
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (冪等性キー)**:
  - **Given** Pub/Sub から同一 `historyId` の Push が再送される
  - **When** Workers が処理を試みる
  - **Then** KV に保存された冪等性キーを参照し、既処理ならスキップする

**Note (フェーズによる必要性)**:
- **α テスト(関係者 1 人)** では発生頻度が極めて低く、二重通知が来てもユーザーが気付くだけで実害が少ないため **後回し可**
- **β テスト(複数ユーザー、production 環境)** では Pub/Sub の at-least-once 配信が無視できなくなり、通知重複・仮予定重複が起きやすいため **MUST 化**
- 実装コストは小さい(KV `put-if-absent` 1 行)ので、α 段階でも軽く入れておくのは推奨

**Traceability**:
- Implements: FR-1

---

### Story U1-EC-03: [Edge] Gmail Watch の有効期限切れを自動再登録する

**As a** MC
**I want** Gmail Watch が切れても自動で再登録される
**so that** Push 通知が止まる事故が起きない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (Cron 再登録)**:
  - **Given** Cron Trigger が定時実行
  - **When** Watch 有効期限が 24 時間以内
  - **Then** `users.watch` を再呼び出しして延長する
- **AC-2 (失敗通知)**:
  - **Given** Watch 再登録が失敗(OAuth 失効等)
  - **When** エラーが発生
  - **Then** ユーザーに LINE で「再認可してください」と通知

**Traceability**:
- Implements: FR-1, NFR-3

---

# 🟩 MVP Epic — Use Case 2: 案件情報抽出(F2)

---

### Story U2-00: Haiku プロンプト設計と評価ハーネスの整備(F2 全体の前提)

**As a** サービス開発者
**I want** 案件情報抽出に使う Claude Haiku のプロンプトを **バージョン管理 + 評価ハーネス付き** で設計する
**so that** プロンプト改良時に **既存抽出精度を退行させない** ことを機械的に検証でき、βテストでの改善ループが回せる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (プロンプトの外部化)**:
  - **Given** 抽出プロンプト
  - **When** 設計
  - **Then** プロンプトは **コードに直書きせず** `crates/infrastructure/src/prompts/extraction_v1.md` 等にバージョン番号付きで保存し、`include_str!` で読み込む(Git で diff レビュー可能に)
- **AC-2 (出力スキーマの厳格化)**:
  - **Given** Haiku の応答
  - **When** プロンプトを書く
  - **Then** **Tool use(function calling)で JSON Schema を強制** し、`serde` 型と一致しないレスポンスは LLM 側で再生成させる(自由形式のテキスト返答禁止)
- **AC-3 (評価ハーネス)**:
  - **Given** ゴールデンセット(βテスター提供の実メール 30〜100 件 + 期待出力)を `tests/fixtures/extraction/` に保存
  - **When** プロンプトを変更する
  - **Then** `cargo test extraction_eval` で全フィクスチャに対する F1 スコア / 必須項目正解率 / 信頼度分布 が出力され、**閾値を下回ったら CI で fail**(NFR-6 PBT 拡張)
- **AC-4 (Few-shot 例の自動更新)**:
  - **Given** βテスト中に蓄積される正解抽出データ
  - **When** プロンプトを更新する
  - **Then** 過去成功例から事務所別の Few-shot 例を自動選択してプロンプトに組み込む(U2-04 と連携)
- **AC-5 (プロンプトキャッシュ)**:
  - **Given** Anthropic Messages API のプロンプトキャッシュ機能
  - **When** プロンプトを構築
  - **Then** **静的部分(システムプロンプト + Few-shot 例)を `cache_control` でキャッシュし**、変動部分(メール本文)のみリクエスト毎に変える(NFR-7 コスト削減)

**Traceability**:
- Implements: FR-2, NFR-6, NFR-7, F-05

---

### Story U2-01: 「案件募集」メールから日程・場所・報酬・事務所名・案件名・締切等が構造化される

**As a** MC
**I want** メール本文を自分で読まなくても、必要な情報が自動で構造化される
**so that** 一目で案件の概要が把握でき、エントリー判断が早くなる

**Priority**: MUST
**Size**: L
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (LLM 抽出)**:
  - **Given** 分類が `recruitment` のメール
  - **When** 抽出 Consumer が起動する
  - **Then** Claude Haiku が以下のフィールドを JSON で返す: `office_name`, `subject_name`, `schedules[]`, `location`, `compensation`, `deadline_at`, `pr_required`, `other_conditions`
- **AC-2 (永続化)**:
  - **Given** 抽出された JSON
  - **When** `serde` でデシリアライズに成功する
  - **Then** `cases` および `schedules` テーブルに保存される
- **AC-3 (事務所名の推定)**:
  - **Given** 送信元アドレスのドメインまたは署名
  - **When** `office_name` が本文から取れない
  - **Then** 既知のドメイン → 事務所マスタを参照して埋める(FR-2)

**Traceability**:
- Implements: FR-2

---

### Story U2-02: 複数日程・複数時間枠の案件は schedules 配列として全候補が抽出される

**As a** MC
**I want** 「○月△日 10-12 時 / 14-17 時 / 翌日 10-15 時」のような複数候補が個別に管理される
**so that** どの候補スロットにエントリーしているか・確定したかを正確に追跡できる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (配列抽出 — 各要素の構成を明示)**:
  - **Given** 複数日程記載のメール
  - **When** LLM が抽出する
  - **Then** `schedules: [{ slot_id, start_at, end_at, tz, raw_text, confidence }, ...]` として全候補が返る(最低 1 件以上)。各要素は以下の通り:
    - `slot_id`: スロット識別子(UUID v7、`case_id` と組み合わせて一意)
    - `start_at`: ISO 8601(例: `2026-05-15T10:00:00+09:00`)
    - `end_at`: ISO 8601、未記載なら `null`(終了未定スロット)
    - `tz`: タイムゾーン(MVP は `Asia/Tokyo` 固定だが将来拡張のため明示)
    - `raw_text`: 元メール内の該当表現(例: 「5/15 10:00〜12:00」、抽出根拠の保持)
    - `confidence`: LLM の抽出信頼度(0.0〜1.0、閾値未満は `extraction_warnings` 行き)
- **AC-2 (関連付け)**:
  - **Given** 抽出された複数スロット
  - **When** 永続化する
  - **Then** すべて同一 `case_id` で `schedules` テーブルに保存され、相互参照可能

**Traceability**:
- Implements: FR-2(複数日程対応 / PR #3 review line 120)

---

### Story U2-03: 必須項目欠落時に LINE で警告通知される

**As a** MC
**I want** 抽出に失敗した項目があれば LINE で教えてもらいたい
**so that** 重要情報の見落とし(例: 報酬不明)に気付ける

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (欠落検出)**:
  - **Given** 抽出結果に必須項目(`office_name` / `subject_name` / `schedules` / `location` / `compensation` / `pr_required`)が `null`
  - **When** 検証ステップが走る
  - **Then** 欠落項目が `extraction_warnings` に記録される
- **AC-2 (LINE 通知)**:
  - **Given** 欠落あり
  - **When** 通知ステップが起動
  - **Then** LINE で「案件 X: 報酬が抽出できませんでした。原文をご確認ください」と通知

**Traceability**:
- Implements: FR-2, FR-5

---

### Story U2-04: 事務所ごとの書式パターンが学習・蓄積される

**As a** MC
**I want** 同じ事務所からの案件は精度高く抽出されるようになる
**so that** 利用するほどシステムが賢くなる

**Priority**: SHOULD
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (パターン保存)**:
  - **Given** 抽出が成功した事務所
  - **When** 抽出ステップ完了時
  - **Then** その事務所の典型的な書式パターン(セクション構造のキー)を `office_patterns` テーブルに記録する
- **AC-2 (Few-shot プロンプト)**:
  - **Given** 同じ事務所からの新しいメール
  - **When** 次回抽出時
  - **Then** 過去の成功例を Few-shot 例としてプロンプトに含める(精度向上)

**Traceability**:
- Implements: FR-2

---

### Story U2-EC-01: [Edge] LLM が JSON を返さなかった場合に再試行 + フォールバック

**As a** MC
**I want** LLM が想定外の応答を返してもエラーで止まらない
**so that** 一時的な不調で案件処理が完全停止しない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (再試行)**:
  - **Given** デシリアライズ失敗(JSON 不整合)
  - **When** 再試行する
  - **Then** プロンプトに「JSON のみで応答してください」を強調して 1 回再試行
- **AC-2 (フォールバック)**:
  - **Given** 再試行も失敗
  - **When** 失敗が確定
  - **Then** メールを `needs_manual_review` にマークし、LINE で原文確認を依頼

**Traceability**:
- Implements: FR-2

---

### Story U2-EC-02: [Edge] 報酬や日程が不明瞭な記載でも null として記録される

**As a** MC
**I want** 「応相談」「未定」のような曖昧表現でもクラッシュしない
**so that** 部分的な情報でも以降の処理(候補確認等)に進める

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (null 許容)**:
  - **Given** 報酬欄が「応相談」
  - **When** 抽出する
  - **Then** `compensation = null` として保存され、警告通知のみ行う
- **AC-2 (日程の自然言語)**:
  - **Given** 「来月の土日」のような相対表現
  - **When** LLM がパースを試みる
  - **Then** メール受信日基準で具体日に解決し、解決不能なら `null` + 警告

**Traceability**:
- Implements: FR-2

---

### Story U2-EC-03: [Edge] PR 要素必要性の判定ヒューリスティクス(U4 PR 文生成への入力)

**As a** MC
**I want** PR 文を求めない事務所には PR 文を入れない
**so that** 不要な定型文を避け、印象を悪くしない

**Priority**: SHOULD
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (キーワード判定)**:
  - **Given** 「自己 PR を添えて」「実績をお書きください」等のキーワード
  - **When** 抽出時に判定
  - **Then** `pr_required = true` を返す
- **AC-2 (デフォルト)**:
  - **Given** 明示的な記載なし
  - **When** 不明確
  - **Then** `pr_required = false` をデフォルトとする
- **AC-3 (下流ストーリーとの関連明示)**:
  - **Given** `pr_required` の判定結果
  - **When** 抽出後の処理
  - **Then** `cases.pr_required` として永続化され、**U4-02(PR 文を含む下書き作成)** がこれを参照して挙動を切り替える(`true` ならフル PR 文、`false` なら短い挨拶のみ)

**Traceability**:
- Implements: FR-2, FR-4(U4-02 へ連携)

---

### Story U2-EC-04: [Edge] 抽出失敗・部分失敗の統一フロー(共通失敗ハンドリング基盤)

**As a** サービス開発者
**I want** 案件抽出に限らずパイプライン全体の失敗ケース(LLM 失敗 / 必須項目欠落 / API 失敗 / OAuth 失効)を **共通フロー** で扱う
**so that** ユーザー / 運用者の両方にとって失敗時の体験が一貫し、抜け漏れによる事故を防げる

**Priority**: MUST
**Size**: M
**Type**: Edge Case(Enabler 寄り)

**Acceptance Criteria**:
- **AC-1 (失敗カテゴリの定義)**:
  - **Given** パイプライン全体の失敗種別
  - **When** 設計
  - **Then** 以下を統一カテゴリとして `crates/domain/src/error.rs` に列挙:
    - `Transient`(一時的、自動リトライで回復): API 5xx / レート制限 / ネットワーク
    - `Recoverable`(ユーザー操作で回復): OAuth 失効 / トークン無効
    - `DataIssue`(データ不備、人手確認): 必須項目欠落 / JSON パース失敗 / 抽出 confidence 低
    - `Permanent`(回復不能): プロンプト不備 / 仕様外メール
- **AC-2 (カテゴリ別の対応)**:
  - **Given** 失敗カテゴリ
  - **When** ハンドリング
  - **Then**:
    - `Transient` → Queues の自動リトライ + DLQ
    - `Recoverable` → ユーザーに LINE で具体的な復旧ボタン送付(「もう一度 Google と連携」等)
    - `DataIssue` → ユーザーに LINE で原文確認依頼 + `needs_manual_review` フラグ
    - `Permanent` → 運用者に LINE 通知 + 監査ログ記録、ユーザー通知は控えめ(混乱を避ける)
- **AC-3 (ドキュメント化)**:
  - **Given** 失敗フロー全体
  - **When** ドキュメント整備
  - **Then** `docs/operations.md` の「失敗とリカバリ」セクションに 4 カテゴリ × 復旧手順を表形式で明文化

**Traceability**:
- Implements: FR-1〜FR-8 全般のエラー処理基盤, NFR-3 可用性, NFR-8 監視・運用

---

# 🟩 MVP Epic — Use Case 3: カレンダー空き確認(F3)

---

### Story U3-01: 各候補スロットが Google カレンダーと突き合わせて重複判定される

**As a** MC
**I want** 抽出された各スロットについて自分のカレンダーが空いているか自動判定される
**so that** エントリー可否の判断が手作業より早くなる

**Priority**: MUST
**Size**: L
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (個別判定 — 「自分」= 本サービス側の自動処理)**:
  - **Given** `schedules[]` に複数候補
  - **When** 判定が走る
  - **Then** **本サービスの Workers / Queue Consumer** が各スロットを独立に判定し、`{slot_id, status: free|partial_overlap|full_overlap}` を返す(FR-3)
  - **Note**: ストーリー内の「自分のカレンダー」「自動判定される」の **「自分」はユーザー(MC)を指す**(ユーザーから見て「自分のカレンダーが、サービス側で自動判定される」)。一方、内部処理の主語は **「本サービス(システム)」** であることを明示
- **AC-2 (Calendar 取得 — プライベート情報の最小権限化)**:
  - **Given** ユーザーの Google カレンダー
  - **When** 該当時間帯の予定を取得する
  - **Then** **`freeBusy.query` API** を主に使用し、**「予定があるかどうか(busy 区間のみ)」** を取得する(本文・出席者・場所等のプライベート情報は取得しない)
  - **Then** 必要なときのみ(自分が登録した `[仮]` / `[確定]` 予定の更新 / 削除のため)`events.list` で対象イベントの詳細を取得し、**サービス側の D1 には保存しない**(揮発処理)
  - **Then** 同意画面(F-07)で「**カレンダーの予定の中身は読みません。空き時間だけ確認します**」と P2 配慮の平易な日本語で明示
- **AC-3 (集約 — 「仮」同士の重なりは許容、業界の通常運用に準拠)**:
  - **Given** 全スロットの判定結果
  - **When** 集約する
  - **Then** 以下のロジックで判定:
    - 既存の `[仮]` 予定と重複 → **エントリー可能**(同じ日程に複数の仮案件をかぶせるのは業界では通常運用)
    - 既存の `[確定]` 予定と重複 → **エントリー不可**(本ブッキングを侵さない)
    - 既存のプライベート予定(本サービスが作成していない busy 区間)と重複 → **エントリー不可**(デフォルト)
  - **Then** 1 つでも上記基準でエントリー可能なスロットがあればエントリー候補、すべて不可なら辞退対象とフラグ立て
  - **Note (業務パターン明記)**: ユーザーは 1 つの日程に **複数の仮案件をかぶせてエントリー** し、最初に決定した案件だけ採用してその他の事務所には辞退連絡を送る、という運用が一般的(要件 2.2 ステップ 6〜8)。本サービスはこのパターンを **積極的にサポート** する(U7 辞退連絡半自動化と連携)

**Traceability**:
- Implements: FR-3, NFR-1, NFR-5(プライバシー — 予定本文は取得しない), FR-7(辞退連絡と連携)

---

### Story U3-02: 部分重複(時間帯の一部が被る)も検知される

**As a** MC
**I want** 既存予定と部分的に重なる候補も「重複」として警告される
**so that** 「微妙に被る案件にうっかりエントリー」を防げる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (区間判定)**:
  - **Given** 既存予定 `[a, b)` と候補 `[c, d)`
  - **When** 重複判定する
  - **Then** `c < b && a < d` の判定で部分重複も検出される(NFR-6 PBT 対象)
- **AC-2 (区分表示)**:
  - **Given** 部分重複検出
  - **When** 結果を返す
  - **Then** `partial_overlap` 区分で返し、LINE 通知でも明示する

**Traceability**:
- Implements: FR-3, NFR-6

---

### Story U3-03: 移動時間バッファ(前後 1 時間)を考慮した判定

**As a** MC
**I want** 物理的に間に合わない案件は重複扱いになる
**so that** 「前後の予定が近すぎて移動できない」事故を防げる

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (バッファ拡張)**:
  - **Given** 既存予定 `[a, b)`
  - **When** 重複判定する
  - **Then** 既存予定を `[a - 1h, b + 1h)` に拡張して候補と比較する(FR-3)
- **AC-2 (設定可能性)**:
  - **Given** バッファは設定値
  - **When** 設定変更が反映される
  - **Then** デフォルト 60 分、将来は LINE / 設定で変更可能(Phase 2)

**Traceability**:
- Implements: FR-3

---

### Story U3-04: 仮予定 / 確定予定 / プライベート予定の区別(プライバシー配慮)

**As a** MC
**I want** 自分が立てた仮予定 / 確定予定 / プライベート予定が区別され、**プライベート予定の中身はサービス側に渡さない**
**so that** 「仮予定だから動かせる」「確定だから絶対無理」という判断が明確になり、かつプライベート情報の漏洩を心配しなくて済む

**Priority**: MUST(プライバシーに直結するため SHOULD → MUST に格上げ)
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (タイトル接頭辞判定 — 本サービスが作成した予定のみ詳細取得)**:
  - **Given** 本サービスが過去に登録した `[仮]` / `[確定]` プレフィックスを持つ予定
  - **When** 判定する
  - **Then** これらは本サービスが作成・所有する予定なので詳細(タイトル・説明欄)を取得しても問題ない
  - **Then** `[仮]` 接頭辞 → 仮予定、`[確定]` または `[仮]` 削除済 → 確定予定として扱う
- **AC-2 (それ以外のカレンダー予定 = プライベートとして扱う)**:
  - **Given** 本サービスが作成していない予定(他アプリが作成したプライベート予定 / 仕事 / 個人予定)
  - **When** 重複判定で参照する
  - **Then** **`freeBusy.query` API のみで「busy か free か」だけを判定**、タイトル / 場所 / 説明 / 出席者 / 添付などの詳細は **取得しない・サーバーに保存しない**
- **AC-3 (UI / 通知での扱い)**:
  - **Given** 重複判定結果に「プライベート予定と被っている」スロットがある
  - **When** ユーザーに LINE 通知
  - **Then** 「**この時間帯はあなたの他の予定と重なっています**」とだけ伝え、**プライベート予定の内容は表示しない**(タイトルや場所を Bot メッセージに含めない)
- **AC-4 (同意画面での明示)**:
  - **Given** F-07 のプライバシーポリシー同意画面
  - **When** カレンダー権限について説明
  - **Then** 「**プライベートな予定の中身は読みません。本サービスが登録した『[仮]』『[確定]』予定だけ管理します**」と明記し、ユーザーの不安を解消する

**Traceability**:
- Implements: FR-3, FR-6, NFR-5(プライバシー), F-08(P2 配慮)

---

### Story U3-EC-01: [Edge] Google Calendar API 失敗時のリトライと通知

**As a** MC
**I want** Calendar 取得が失敗しても、案件処理が中断されたことを把握できる
**so that** 知らないうちに重複判定がスキップされていた、を防げる

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (リトライ)**:
  - **Given** API が 5xx
  - **When** 判定処理中
  - **Then** 指数バックオフで最大 3 回再試行する
- **AC-2 (失敗通知)**:
  - **Given** 全リトライ失敗
  - **When** 失敗確定
  - **Then** LINE で「案件 X のカレンダー判定に失敗しました。手動でご確認ください」と通知

**Traceability**:
- Implements: FR-3, NFR-3, FR-5

---

# 🟩 MVP Epic — Use Case 4: エントリー下書き作成(F4)

---

### Story U4-01: エントリー可能な案件について返信下書きが Gmail 下書きフォルダに保存される

**As a** MC
**I want** エントリーする返信メールの下書きを自動で作ってもらいたい
**so that** 自分で文面を書く手間が減り、送信前に確認だけすればよい

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (下書き作成)**:
  - **Given** カレンダー判定が `free` または `partial_overlap` で、ユーザーが LINE で「エントリー」を承認(下書きトリガ)
  - **When** 下書き生成が起動する
  - **Then** Gmail API `drafts.create` で下書きが保存される
- **AC-2 (返信ヘッダ)**:
  - **Given** 元メールの Message-ID
  - **When** 下書きを作成する
  - **Then** `In-Reply-To` と `References` ヘッダが正しく設定され、Gmail 上でスレッド継続される
- **AC-3 (本文)**:
  - **Given** 抽出された案件情報 + ユーザー名
  - **When** 本文テンプレートを適用
  - **Then** 「お世話になっております。〜エントリー希望いたします。〜」の定型 + 案件番号・希望スロットを含む

**Traceability**:
- Implements: FR-4

---

### Story U4-02: PR 要素が必要な案件には過去メール学習 PR 文が含まれる

**As a** MC
**I want** PR が求められる案件には過去の自分らしい PR 文が自動で挿入される
**so that** 毎回似た文面を書き起こす手間が省け、口調が一貫する

**Priority**: MUST
**Size**: L
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (条件判定)**:
  - **Given** `pr_required = true`
  - **When** 下書き生成
  - **Then** PR 文セクションが本文に追加される
- **AC-2 (学習データ参照)**:
  - **Given** `pr_corpus` テーブルに過去エントリーメールが蓄積
  - **When** Claude Haiku で生成
  - **Then** 過去 3〜5 件を Few-shot 例としてプロンプトに含め、案件の種別・場所に合わせた PR 文を生成
- **AC-3 (字数調整)**:
  - **Given** 案件の規模(報酬・場所)
  - **When** 生成
  - **Then** 短(2〜3 行)/ 中(4〜6 行)/ 長(7〜10 行)の長さを自動選択

**Traceability**:
- Implements: FR-4

---

### Story U4-03: 過去エントリーメールから PR 文学習データが自動収集される

**As a** MC
**I want** 自分が過去に書いたエントリー文を勝手に集めて学習データにしてほしい
**so that** わざわざ手動で「これを学習させる」操作をしなくて済む

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (収集対象)**:
  - **Given** Gmail の Sent フォルダ
  - **When** 初期セットアップ時 + 定期的に
  - **Then** 直近 N 通(例: 50 通)の自分が送信したエントリー風メールを取得
- **AC-2 (本人帰属)**:
  - **Given** 収集メール
  - **When** 永続化
  - **Then** `pr_corpus` テーブルに保存(NFR-5: 本人作成文章は永続保管)
- **AC-3 (フィルタ)**:
  - **Given** Sent フォルダの全メール
  - **When** 収集判定
  - **Then** 「お世話に」「エントリー」「希望」等のキーワードを含むメールに限定

**Traceability**:
- Implements: FR-4, NFR-5

---

### Story U4-04: 下書き作成完了時に LINE で確認を促す

**As a** MC
**I want** 下書きができたら LINE で知らせてほしい
**so that** Gmail を開いて確認・送信に進める

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (通知)**:
  - **Given** 下書き作成完了
  - **When** 通知が起動
  - **Then** LINE で「案件 X の下書きを作成しました。Gmail で確認・送信してください」と通知 + Gmail 下書きへの URL リンク

**Traceability**:
- Implements: FR-4, FR-5

---

### Story U4-EC-01: [Edge] PR 文学習データが不足している場合のデフォルト

**As a** MC
**I want** 初回利用時(学習データ少)でも妥当な PR 文が出る
**so that** サービス使い始めから恥ずかしくない文面が使える

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (フォールバック)**:
  - **Given** `pr_corpus` 件数 < 3
  - **When** PR 文生成
  - **Then** デフォルトテンプレート(汎用的な MC 自己紹介)を使用し、ユーザー編集を促すコメントを付ける

**Traceability**:
- Implements: FR-4

---

### Story U4-EC-02: [Edge] Gmail 下書き作成 API 失敗時の通知

**As a** MC
**I want** 下書き作成が失敗したら気付ける
**so that** 知らないうちにエントリー機会を逃さない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (失敗通知)**:
  - **Given** Gmail API 失敗
  - **When** 最終リトライ後も失敗
  - **Then** LINE で「下書き作成に失敗しました。再試行 / 手動 / 後ほど のいずれかを選んでください」と通知 + 操作ボタン

**Traceability**:
- Implements: FR-4, FR-5

---

# 🟩 MVP Epic — Use Case 5: LINE 通知(F5)

---

### Story U5-01: 案件募集を検出時に案件サマリと操作ボタン付き通知を受け取る

**As a** MC
**I want** 案件が来たら LINE で要点だけサクッと教えてほしい
**so that** メールを開かなくても判断できる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (Flex Message)**:
  - **Given** 抽出 + カレンダー判定が完了
  - **When** 通知が起動
  - **Then** LINE Flex Message で「事務所 / 案件名 / 日程候補(空き表示) / 場所 / 報酬」のサマリ + 「エントリー / 却下 / 詳細」ボタンが届く
- **AC-2 (タイミング)**:
  - **Given** メール受信時刻
  - **When** 通知到達まで
  - **Then** **5 分以内**(NFR-1)
- **AC-3 (ポストバック)**:
  - **Given** ユーザーが「エントリー」ボタンを押す
  - **When** Webhook が受信
  - **Then** 該当案件の下書き生成フローに進む

**Traceability**:
- Implements: FR-5, NFR-1

---

### Story U5-02: 抽出に欠落があれば LINE で警告を受け取る

**As a** MC
**I want** 抽出に失敗した部分があれば手動確認を促してほしい
**so that** 重要情報の見落としを防げる

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (警告メッセージ)**:
  - **Given** `extraction_warnings` に項目あり
  - **When** 通知ステップ
  - **Then** 「⚠️ 案件 X: [報酬] が抽出できませんでした」+ Gmail 原文へのリンクが届く

**Traceability**:
- Implements: FR-5, FR-2

---

### Story U5-03: 重複検出時(辞退候補)に通知を受け取る

**As a** MC
**I want** 決定済み案件と重複する仮案件があれば早めに教えてほしい
**so that** 辞退連絡を遅らせない

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (辞退候補リスト)**:
  - **Given** 決定連絡を検出
  - **When** 重複検出ロジックが走る
  - **Then** 重複する仮案件を一覧で LINE 通知し、「辞退下書き作成」ボタンを添える

**Traceability**:
- Implements: FR-5, FR-7

---

### Story U5-04: エントリー下書き作成完了時に通知を受け取る(別途 U4-04 を参照)

**As a** MC
**I want** 下書き完了の事実を確認したい
**so that** 次のアクション(Gmail で確認・送信)に進める

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1**: U4-04 の AC-1 と同等

**Traceability**:
- Implements: FR-5, FR-4

---

### Story U5-05: 決定連絡を検出時に通知を受け取る

**As a** MC
**I want** 「採用決定」のメールを受け取ったらすぐ気付きたい
**so that** カレンダー更新と辞退連絡の準備に着手できる

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (検出 → 通知)**:
  - **Given** 分類 = `decision`
  - **When** 通知ステップ
  - **Then** LINE で「🎉 決定: 事務所 X / 案件 Y」+「カレンダー更新 / 辞退候補確認」ボタンが届く

**Traceability**:
- Implements: FR-5, FR-1, FR-6

---

### Story U5-06: LINE のアクションボタンで半自動操作ができる

**As a** MC
**I want** LINE のボタンタップだけで承認・却下・辞退送信が完結する
**so that** Gmail / カレンダーを開く回数を減らせる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (Postback)**:
  - **Given** ボタン付きメッセージ
  - **When** ユーザーがタップ
  - **Then** Postback event が Webhook に届き、対応するアクション(エントリー / 却下 / 辞退送信)が実行される
- **AC-2 (確認応答)**:
  - **Given** アクション実行
  - **When** 完了
  - **Then** LINE で結果(成功 / 失敗)が返信される

**Traceability**:
- Implements: FR-5, FR-7

---

### Story U5-EC-01: [Edge] LINE Messaging API のレート制限対応

**As a** MC
**I want** 短時間に通知が連続しても抜け落ちない
**so that** 重要な案件を見逃さない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (レート制御)**:
  - **Given** LINE API のレート制限(429)
  - **When** Workers から push する
  - **Then** Queues に再エンキューし、指数バックオフで再送する

**Traceability**:
- Implements: FR-5, NFR-3

---

# 🟩 MVP Epic — Use Case 6: カレンダー自動管理(F6)

---

### Story U6-01: エントリー下書き作成時にカレンダーへ「[仮]」予定が登録される

**As a** MC
**I want** エントリーしたら自動で仮予定がカレンダーに入る
**so that** 「あの案件入れたっけ?」と不安にならない

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (登録)**:
  - **Given** 下書き作成完了
  - **When** カレンダー登録ステップ
  - **Then** 各候補スロットに `[仮] {案件名} - {事務所名}` のタイトルで Calendar イベントが作成される
- **AC-2 (色分け)**:
  - **Given** 仮予定
  - **When** 作成
  - **Then** 視覚的に区別できる固定色(例: グレー)を割り当てる
- **AC-3 (説明欄)**:
  - **Given** 案件情報
  - **When** イベント作成
  - **Then** description に元メールへのリンク + 報酬 + その他条件を含める

**Traceability**:
- Implements: FR-6

---

### Story U6-02: 複数候補スロットがある案件は同一案件 ID で関連付けて全候補登録される

**As a** MC
**I want** 候補が 3 つあれば 3 つとも仮で押さえておきたい
**so that** どれが採用されても他をクリーンに削除できる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (一括登録)**:
  - **Given** `schedules[]` 複数
  - **When** 仮予定登録
  - **Then** すべてのスロットが登録され、`extendedProperties.private.case_id` に共通 ID が付与される
- **AC-2 (DB 連携)**:
  - **Given** 登録結果
  - **When** 永続化
  - **Then** `calendar_events` テーブルに `case_id`, `slot_id`, `event_id` の対応が保存される

**Traceability**:
- Implements: FR-6(複数日程対応 / PR #3 review line 120)

---

### Story U6-03: 決定連絡検出時に該当スロットが「[仮]」→「確定」に更新される

**As a** MC
**I want** 採用決定したスロットが自動で確定状態に変わる
**so that** 自分でカレンダーを更新する手間がない

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (タイトル更新)**:
  - **Given** 決定連絡を分類 + 該当スロットを特定
  - **When** カレンダー更新
  - **Then** イベントタイトルから `[仮]` プレフィックスが削除され、色を確定色(例: 緑)に変更
- **AC-2 (ステータス保存)**:
  - **Given** 確定
  - **When** DB 更新
  - **Then** `cases.status = confirmed`, `schedules[selected].confirmed = true`

**Traceability**:
- Implements: FR-6, FR-1(decision 分類)

---

### Story U6-04: 決定時に他の候補スロットは自動削除される

**As a** MC
**I want** 採用されなかったスロットの仮予定が勝手に消える
**so that** カレンダーが古い候補で散らからない

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (削除)**:
  - **Given** 同一 `case_id` の他スロット
  - **When** 1 スロットが確定
  - **Then** 他のスロットの Calendar イベントが削除される
- **AC-2 (履歴)**:
  - **Given** 削除実行
  - **When** 監査ログ記録
  - **Then** `audit_logs` に削除イベントが残る

**Traceability**:
- Implements: FR-6

---

### Story U6-05: 辞退承認時に同一案件 ID の全候補スロットが一括削除される

**As a** MC
**I want** 辞退すると決めた案件の仮予定が一括で消える
**so that** 削除漏れの不安がない

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (一括削除)**:
  - **Given** ユーザーが LINE で辞退承認
  - **When** カレンダー削除ステップ
  - **Then** `case_id` で紐づく全イベントを削除する

**Traceability**:
- Implements: FR-6, FR-7

---

### Story U6-EC-01: [Edge] Calendar API 失敗時のリトライと通知

**As a** MC
**I want** カレンダー操作が失敗しても気付ける
**so that** カレンダーと DB が乖離した状態を放置しない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (リトライ + 通知)**:
  - **Given** Calendar API 5xx
  - **When** 最終リトライも失敗
  - **Then** LINE で「カレンダー更新失敗」+ 手動操作リンクを通知

**Traceability**:
- Implements: FR-6, NFR-3, FR-5

---

# 🟩 MVP Epic — Use Case 7: 辞退連絡 半自動(F7)

---

### Story U7-01: 決定案件と日程重複する仮案件が自動検出される

**As a** MC
**I want** 決定後に重複している仮案件のリストが自動で出てくる
**so that** 辞退連絡漏れを防げる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (検出ロジック)**:
  - **Given** 決定スロット `[c, d)` と全仮案件のスロット
  - **When** 検出処理
  - **Then** 移動時間バッファ込みで重複する仮案件を全て返す(FR-3 と同ロジック)

**Traceability**:
- Implements: FR-7, FR-3

---

### Story U7-02: 各事務所宛の辞退メール下書きが生成され、LINE で承認依頼が届く

**As a** MC
**I want** 辞退メールの文面を AI が用意して、自分は承認するだけにしたい
**so that** 個別文面作成の手間が消える

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (下書き生成)**:
  - **Given** 重複する仮案件のリスト
  - **When** 辞退下書き生成が走る
  - **Then** 各事務所宛に「お世話になっております。〜大変恐縮ですが今回は辞退させていただきます。〜」の定型を生成
- **AC-2 (承認依頼)**:
  - **Given** 下書き生成完了
  - **When** LINE 通知
  - **Then** 「以下の辞退メールを送信しますか?[送信 / プレビュー / キャンセル]」が一覧で届く

**Traceability**:
- Implements: FR-7, FR-5

---

### Story U7-03: LINE 承認後に Gmail API で辞退メールが送信される

**As a** MC
**I want** ボタン 1 つで辞退メールが送信される
**so that** 複数事務所への送信を素早く完了できる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (送信)**:
  - **Given** ユーザーが「送信」承認
  - **When** Gmail API `messages.send`(件名・本文・In-Reply-To 維持)
  - **Then** 各事務所宛に送信され、送信成功 ID が返る
- **AC-2 (カレンダー連携)**:
  - **Given** 送信成功
  - **When** 後続処理
  - **Then** 該当案件のカレンダーイベントを削除(U6-05 へ連携)

**Traceability**:
- Implements: FR-7, FR-6

---

### Story U7-04: 辞退メール送信履歴が監査ログに記録される

**As a** MC
**I want** 後から「いつ・どの事務所に・何を送ったか」を追跡できる
**so that** 万一トラブルがあった時に証跡を提示できる

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (監査記録)**:
  - **Given** 送信実行
  - **When** 監査ログ書き込み
  - **Then** `audit_logs` に `{action: decline_sent, office, case_id, sent_at, message_id}` を記録(NFR-8)

**Traceability**:
- Implements: FR-7, NFR-8

---

### Story U7-EC-01: [Edge] 辞退メール送信失敗時の再試行とユーザー通知

**As a** MC
**I want** 送信失敗時に「送れていない」とすぐ気付ける
**so that** 信頼を損なう「実は辞退連絡が届いていなかった」を防げる

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (リトライ)**:
  - **Given** Gmail API 失敗
  - **When** 自動リトライ
  - **Then** 指数バックオフで最大 3 回再試行
- **AC-2 (失敗通知 + 再操作)**:
  - **Given** 全リトライ失敗
  - **When** 失敗確定
  - **Then** LINE で「辞退送信失敗: 事務所 X / 再試行ボタン」を表示

**Traceability**:
- Implements: FR-7, FR-5

---

# 🟩 MVP Epic — Use Case 8: 請求 CSV エクスポート(F8)

---

### Story U8-01: LINE で「請求CSV [年月]」と指示すれば一時 URL が返される

**As a** MC
**I want** 月末に LINE から CSV ダウンロード URL を取得できる
**so that** PC を開いて確定申告 / 営業資料を作る作業が早くなる

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (コマンド受付)**:
  - **Given** LINE で「請求CSV 2026-04」のメッセージ
  - **When** Bot が受信
  - **Then** 該当月の確定案件を CSV 化し、R2 に保存して短期(15 分)有効な URL を返す
- **AC-2 (URL の安全性)**:
  - **Given** 一時 URL
  - **When** 生成
  - **Then** 推測困難なトークン付き署名 URL を発行し、有効期限後はアクセス不可

**Traceability**:
- Implements: FR-8, NFR-4 SECURITY-08

---

### Story U8-02: CSV には月・事務所名・案件名・日程・場所・報酬・ステータスが含まれる

**As a** MC
**I want** Excel で開いてそのまま使える CSV にしてほしい
**so that** 加工なしに集計や請求書作成に使える

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (列構成)**:
  - **Given** 確定案件
  - **When** CSV 化
  - **Then** ヘッダ: `month, office_name, subject_name, start_at, end_at, location, compensation, status` の順
- **AC-2 (確定スロット)**:
  - **Given** 複数候補があった案件
  - **When** 出力
  - **Then** 確定スロット 1 件のみ出力(PR #3 review line 120 反映)

**Traceability**:
- Implements: FR-8

---

### Story U8-03: 文字コードは UTF-8 BOM 付きで Excel で文字化けしない

**As a** MC
**I want** CSV を Excel でダブルクリックで開いて日本語が崩れない
**so that** 文字コード変換等の手間がない

**Priority**: MUST
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (BOM)**:
  - **Given** CSV 生成
  - **When** バイト列を出力
  - **Then** 先頭に UTF-8 BOM(`EF BB BF`)が付与され、Excel でそのまま開ける

**Traceability**:
- Implements: FR-8

---

### Story U8-04: デフォルトで確定案件のみ出力、辞退/未決定はオプション

**As a** MC
**I want** 請求用なので確定だけ欲しい(辞退の振り返りは別途)
**so that** 集計の手間を最小化できる

**Priority**: SHOULD
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (デフォルト)**:
  - **Given** 「請求CSV 2026-04」
  - **When** 出力
  - **Then** ステータス = `confirmed` のみが含まれる
- **AC-2 (オプション)**:
  - **Given** 「請求CSV 2026-04 全件」
  - **When** 出力
  - **Then** 全ステータスが含まれる

**Traceability**:
- Implements: FR-8

---

### Story U8-EC-01: [Edge] 該当月にデータがない場合の挙動

**As a** MC
**I want** データがなくてもエラーで止まらない
**so that** 「メッセージがおかしくて壊れた?」と不安にならない

**Priority**: SHOULD
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (空 CSV)**:
  - **Given** 該当月確定案件 0 件
  - **When** 出力
  - **Then** ヘッダーのみの空 CSV が生成され、URL が返る
- **AC-2 (LINE 注釈)**:
  - **Given** 0 件
  - **When** URL 返信
  - **Then** 「該当月の確定案件はありません」と注記される

**Traceability**:
- Implements: FR-8

---

# 🟨 Phase 2 Epic — マルチテナント・拡張機能(タイトル + 概要のみ)

Q5 = B により Phase 2 はタイトルと概要のみ記述する。詳細化は Phase 2 開始時に別途実施。

| ID | タイトル | 概要 |
|----|---------|------|
| **P2-01** | マルチテナント認証 | Auth0 / Clerk / 自前で複数ユーザーアカウントを管理し、テナント分離(D1 行レベル / Cloudflare Access)を実装 |
| **P2-02** | Web ダッシュボード(実績登録 UI) | Foundation Epic の MVP では UI なし方針だが、Phase 2 で実績登録・案件一覧・設定変更の最小 Web UI を追加(`leptos` / `yew` または React) |
| **P2-03** | 課金システム | Stripe Subscription による月額 500 円課金、無料トライアル、解約フロー |
| **P2-04** | Cloud Run / Cloud SQL 移行 | スケール上限到達時、Cloudflare → GCP に段階的移行(Drizzle / sqlx スキーマ互換、データ移行スクリプト) |
| **P2-05** | Gmail 以外のメール対応 | Outlook / iCloud / IMAP 連携。アダプタ層を抽象化 |
| **P2-06** | Google Calendar 以外対応 | Outlook Calendar / Apple Calendar 連携 |
| **P2-07** | エントリー送信の完全自動化(オプション) | ユーザー設定で「事務所別に完全自動」を有効化可能に |

---

# 📊 トレーサビリティ表(Story → 要件)

| Story ID | Implements (FR) | Implements (NFR) |
|----------|----------------|------------------|
| F-01 | — | NFR-4 SECURITY-10 |
| F-02 | — | NFR-2 / NFR-4 SECURITY-01 |
| F-03 | FR-1, FR-3, FR-4, FR-6, FR-7 | NFR-4 SECURITY-01/06/08, NFR-5 |
| F-04 | FR-5 | NFR-4 SECURITY-05 |
| F-05 | FR-1, FR-2, FR-4 | NFR-1, NFR-7, NFR-4 SECURITY-03/09 |
| F-06 | — | NFR-8, NFR-4 SECURITY-02/03, NFR-5 |
| F-07 | — | NFR-5, F-08 |
| F-08 | — | NFR-8(P2 ペルソナ配慮全般) |
| F-09 | — | NFR-4 SECURITY-01/03/06/09(F-03/04/05 の前提) |
| U1-01 | FR-1 | — |
| U1-02 | FR-1 | NFR-1, NFR-7 |
| U1-03 | FR-1 | NFR-7 |
| U1-EC-01 | FR-1 | NFR-1, NFR-8 |
| U1-EC-02 | FR-1 | — |
| U1-EC-03 | FR-1 | NFR-3 |
| U2-01 | FR-2 | — |
| U2-02 | FR-2 | — |
| U2-03 | FR-2, FR-5 | — |
| U2-04 | FR-2 | — |
| U2-EC-01 | FR-2 | — |
| U2-EC-02 | FR-2 | — |
| U2-EC-03 | FR-2 | — |
| U3-01 | FR-3 | NFR-1 |
| U3-02 | FR-3 | NFR-6 |
| U3-03 | FR-3 | — |
| U3-04 | FR-3, FR-6 | — |
| U3-EC-01 | FR-3, FR-5 | NFR-3 |
| U4-01 | FR-4 | — |
| U4-02 | FR-4 | — |
| U4-03 | FR-4 | NFR-5 |
| U4-04 | FR-4, FR-5 | — |
| U4-EC-01 | FR-4 | — |
| U4-EC-02 | FR-4, FR-5 | — |
| U5-01 | FR-5 | NFR-1 |
| U5-02 | FR-5, FR-2 | — |
| U5-03 | FR-5, FR-7 | — |
| U5-04 | FR-5, FR-4 | — |
| U5-05 | FR-5, FR-1, FR-6 | — |
| U5-06 | FR-5, FR-7 | — |
| U5-EC-01 | FR-5 | NFR-3 |
| U6-01 | FR-6 | — |
| U6-02 | FR-6 | — |
| U6-03 | FR-6, FR-1 | — |
| U6-04 | FR-6 | — |
| U6-05 | FR-6, FR-7 | — |
| U6-EC-01 | FR-6, FR-5 | NFR-3 |
| U7-01 | FR-7, FR-3 | — |
| U7-02 | FR-7, FR-5 | — |
| U7-03 | FR-7, FR-6 | — |
| U7-04 | FR-7 | NFR-8 |
| U7-EC-01 | FR-7, FR-5 | — |
| U8-01 | FR-8 | NFR-4 SECURITY-08 |
| U8-02 | FR-8 | — |
| U8-03 | FR-8 | — |
| U8-04 | FR-8 | — |
| U8-EC-01 | FR-8 | — |

---

# 📈 Story 統計

| Epic | Story 数 | うち User Story | うち Enabler | うち Edge Case |
|------|---------|----------------|--------------|----------------|
| Foundation | 12 | 2 (F-03, F-07) | 10 (F-01/02/04/05/06/08/09/10/11/12) | 0 |
| Use Case 1 (F1) | 6 | 3 | 0 | 3 |
| Use Case 2 (F2) | 8 | 4 | 1 (U2-00) | 4 (含 U2-EC-04) |
| Use Case 3 (F3) | 5 | 4 | 0 | 1 |
| Use Case 4 (F4) | 6 | 4 | 0 | 2 |
| Use Case 5 (F5) | 7 | 6 | 0 | 1 |
| Use Case 6 (F6) | 6 | 5 | 0 | 1 |
| Use Case 7 (F7) | 5 | 4 | 0 | 1 |
| Use Case 8 (F8) | 5 | 4 | 0 | 1 |
| Phase 2 | 7 (概略) | — | — | — |
| **合計(MVP+Foundation)** | **60** | **36** | **11** | **14** |

> レビュー反映により Foundation Epic に F-10(ドメイン取得)/ F-11(抽象化レイヤ)/ F-12(システム構成図)を追加、Use Case 2 に U2-00(プロンプト設計)/ U2-EC-04(共通失敗ハンドリング)を追加。計 60 Story で MVP を網羅。
