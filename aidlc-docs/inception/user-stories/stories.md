# ユーザーストーリー — auto-mc-operation

このドキュメントは User Stories の本体です。
`personas.md` のプライマリペルソナ(P1: MC / コンパニオン本人 = 佐藤 美咲さん)を主語として記述します。

---

## 構成方針(Plan セクション 8 準拠)
- **Foundation Epic** + **MVP Use Case Epics(8 個)** + **Phase 2 Epic 概略**
- 粒度: 最小ユーザーアクション = 1 Story、各 Story に 3〜5 個の Given-When-Then 受入条件
- メタ: Priority(MoSCoW) + Size(S/M/L) + Type(User Story / Enabler Story / Edge Case)
- トレーサビリティ: 各 Story に Implements(FR/NFR / 個別 SECURITY-XX)を付与
- エッジケースは独立 Story(`-EC-` プレフィックス)

---

# 🟦 Foundation Epic — 基盤・最初に開発

ユーザーが直接体験しない Enabler Stories だが、後続のユーザーストーリーすべての前提となる。
**Foundation 配下は Type = Enabler Story、ペルソナは MC を含むがアクター視点は「サービス開発者・運用者として」記述する**(Enabler の慣習)。

---

### Story F-01: プロジェクトのディレクトリ構成・開発ルール整備

**As a** サービス開発者
**I want** Cargo workspace としてプロジェクトの初期構造・lint・整形・コミットメッセージ規約を整備したい
**so that** 後続の開発が一貫したコーディング規約・自動化のもとで進められる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (workspace 構造)**:
  - **Given** リポジトリ直下に `Cargo.toml` がない状態
  - **When** Cargo workspace を初期化する
  - **Then** `crates/` 配下に `worker`(Cloudflare Workers) / `domain`(ビジネスロジック) / `infra`(外部 API クライアント) などの crate が分離される
- **AC-2 (lint / format)**:
  - **Given** Rust コードベース
  - **When** `cargo fmt --all -- --check` および `cargo clippy --all-targets -- -D warnings` を実行する
  - **Then** どのコードもフォーマット・lint 警告が 0 で通る
- **AC-3 (コミット規約)**:
  - **Given** CLAUDE.md で定義された日本語コミットメッセージ規約
  - **When** 開発者がコミットを作成する
  - **Then** type: メッセージ形式が CI でチェックされる

**Traceability**:
- Implements: 7.1 言語・フレームワーク, NFR-10 SECURITY-10(依存管理)

---

### Story F-02: Cloudflare 基盤(Workers / D1 / KV / R2 / Queues / Cron / Durable Objects)のセットアップ

**As a** サービス開発者
**I want** Cloudflare アカウントと Wrangler 設定を整備し、各リソースを Infrastructure-as-Code で定義したい
**so that** Workers から D1 / KV / R2 / Queues / Cron / Durable Objects を確実に利用でき、本番・開発環境を同一定義で再現できる

**Priority**: MUST
**Size**: L
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (Wrangler 設定)**:
  - **Given** `wrangler.toml` が未作成
  - **When** Wrangler でプロジェクトを初期化する
  - **Then** `wrangler.toml` に Workers / D1 binding / KV namespace / R2 bucket / Queues producer-consumer / Cron triggers / Durable Object class binding がすべて定義される
- **AC-2 (環境分離)**:
  - **Given** dev / staging / production の 3 環境
  - **When** `wrangler deploy --env <env>` を実行する
  - **Then** 各環境のリソースが独立してデプロイされ、production 環境のシークレットは別管理(Wrangler secrets)
- **AC-3 (D1 マイグレーション)**:
  - **Given** D1 データベースが空
  - **When** マイグレーションスクリプトを実行する
  - **Then** 全テーブル(users / messages / cases / schedules / entries / calendar_events / drafts / pr_corpus / consents / audit_logs 等)が作成され、外部キーが正しく張られる
- **AC-4 (暗号化)**:
  - **Given** D1 / KV / R2 のデフォルト設定
  - **When** リソースを作成する
  - **Then** 暗号化が有効(SECURITY-01 準拠)で、TLS 1.2+ の通信のみ受け付ける

**Traceability**:
- Implements: 7.2 ホスティング, 7.3 データ永続化, NFR-4 SECURITY-01

---

### Story F-03: Google OAuth(Gmail / Calendar)認可フローの構築

**As a** MC(セットアップ時の利用者として)
**I want** 自分の Google アカウントを安全にサービスと連携したい
**so that** Gmail と Google カレンダーへの読み書きをサービスに代行してもらえる

**Priority**: MUST
**Size**: L
**Type**: User Story(セットアップフロー)

**Acceptance Criteria**:
- **AC-1 (OAuth フロー開始)**:
  - **Given** ユーザーが LINE 公式アカウントを友達追加し「セットアップ開始」とメッセージ送信した状態
  - **When** Bot がセットアップ用一時 URL を返す
  - **Then** ユーザーがその URL を開くと Google OAuth 同意画面が表示される
- **AC-2 (スコープ)**:
  - **Given** OAuth 同意画面
  - **When** ユーザーが同意ボタンを押す
  - **Then** Gmail(send / modify / readonly drafts)+ Calendar(events) の読み書きスコープが付与される
- **AC-3 (リフレッシュトークン保管)**:
  - **Given** ユーザーが同意した
  - **When** リフレッシュトークンを取得する
  - **Then** トークンは D1 に AES 暗号化して保存され、KMS 相当のキーで管理される(SECURITY-01)
- **AC-4 (セットアップ完了通知)**:
  - **Given** OAuth 完了
  - **When** Bot がコールバックを受信する
  - **Then** ユーザーに LINE で「セットアップ完了」+ プライバシーポリシー同意リンクが届く
- **AC-5 (リフレッシュ)**:
  - **Given** 有効期限切れアクセストークン
  - **When** リフレッシュ処理が走る
  - **Then** `yup-oauth2` で自動更新され、失敗時はユーザーに LINE で再認可を促す

**Traceability**:
- Implements: FR-1, FR-3, FR-4, FR-6, FR-7, NFR-4 SECURITY-01 / SECURITY-06 / SECURITY-08, NFR-5

---

### Story F-04: LINE Messaging API Bot 連携セットアップ

**As a** サービス開発者
**I want** LINE 公式 Bot を作成し、Webhook を Cloudflare Workers に向ける
**so that** ユーザーとの双方向コミュニケーション(通知 + 承認/却下)が可能になる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (Bot 作成)**:
  - **Given** LINE Developers Console
  - **When** Messaging API チャネルを作成する
  - **Then** Channel access token と Channel secret が取得され、Wrangler secrets に登録される
- **AC-2 (Webhook 設定)**:
  - **Given** デプロイ済み Workers の Webhook エンドポイント
  - **When** LINE Console で Webhook URL を設定する
  - **Then** ユーザーのメッセージ・ポストバックが Workers に POST される
- **AC-3 (署名検証)**:
  - **Given** LINE からの POST リクエスト
  - **When** Workers がリクエストを受信する
  - **Then** `X-Line-Signature` ヘッダを HMAC-SHA256 で検証し、不一致は 401 を返す(SECURITY-05)
- **AC-4 (返信送信)**:
  - **Given** 受信した event の reply_token
  - **When** Bot が返信を送信する
  - **Then** LINE Messaging API の reply エンドポイントに正しく POST され、200 が返る

**Traceability**:
- Implements: FR-5, NFR-4 SECURITY-05

---

### Story F-05: Anthropic API クライアント(Claude Haiku)の整備

**As a** サービス開発者
**I want** Anthropic API のクライアントレイヤを `reqwest` で実装する
**so that** 分類・抽出・PR 文生成の各機能が統一インタフェースで Claude Haiku を呼び出せる

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
- **AC-4 (API キー管理)**:
  - **Given** Anthropic API キー
  - **When** Workers から呼び出す
  - **Then** キーは Wrangler secrets から取得され、ログには出力されない(SECURITY-03 / SECURITY-09)

**Traceability**:
- Implements: FR-1, FR-2, FR-4, NFR-1, NFR-7, NFR-4 SECURITY-03 / SECURITY-09

---

### Story F-06: 構造化ロギング・監査ログ基盤

**As a** サービス開発者
**I want** すべての処理に構造化ログ(timestamp / request_id / user_id / level / message)を付与し、監査ログは別ストアに永続化する
**so that** 障害解析・セキュリティ監査・利用状況把握が可能になる

**Priority**: MUST
**Size**: M
**Type**: Enabler Story

**Acceptance Criteria**:
- **AC-1 (構造化ログ)**:
  - **Given** Workers のリクエストハンドラ
  - **When** リクエストを受け付ける
  - **Then** `request_id` を生成し、以降のすべてのログに含まれる(NFR-8)
- **AC-2 (PII / シークレット除外)**:
  - **Given** メール本文・OAuth トークン・API キー等の機密情報
  - **When** ログ出力関数を呼ぶ
  - **Then** 機密情報フィールドは自動的にマスクされる(SECURITY-03)
- **AC-3 (監査ログの永続化)**:
  - **Given** 案件分類・エントリー送信・辞退送信・カレンダー操作
  - **When** いずれかの操作が成功または失敗する
  - **Then** 監査ログとして R2 に Logpush 経由で送出され、12 ヶ月保管される(NFR-5)
- **AC-4 (アクセスログ)**:
  - **Given** Workers / Webhook エンドポイント
  - **When** リクエストが到達する
  - **Then** Cloudflare のアクセスログが Logpush で R2 / SIEM に送出される(SECURITY-02)

**Traceability**:
- Implements: NFR-8, NFR-4 SECURITY-02 / SECURITY-03, NFR-5

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

**Traceability**:
- Implements: NFR-5(プライバシー・同意管理)

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

**Traceability**:
- Implements: FR-1(Push 通知部分)

---

### Story U1-02: 取り込まれたメールが Claude Haiku で 3 値分類される

**As a** MC
**I want** 受信メールが「案件募集 / 決定連絡 / その他」に自動分類される
**so that** 自分が見るべきメールだけを優先して扱える

**Priority**: MUST
**Size**: M
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (LLM 呼び出し)**:
  - **Given** Queue にエンキューされた「分類タスク」
  - **When** Consumer が処理を開始する
  - **Then** Claude Haiku に件名・本文(先頭 N 文字)・送信元ドメインを渡し、`{recruitment | decision | other}` のいずれかが返る
- **AC-2 (信頼度)**:
  - **Given** LLM のレスポンス
  - **When** 信頼度スコア(または logits 由来)が閾値(例 0.6)未満
  - **Then** ラベルを `other` にフォールバックし、`needs_review = true` を立てる(FR-1)
- **AC-3 (永続化)**:
  - **Given** 分類結果
  - **When** 保存処理が走る
  - **Then** `messages.classification` フィールドが更新され、`audit_logs` に分類イベントが記録される(NFR-8)

**Traceability**:
- Implements: FR-1, NFR-1, NFR-7

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

**Traceability**:
- Implements: FR-1, NFR-1, NFR-8

---

### Story U1-EC-02: [Edge] Push 通知の重複が冪等性キーで防止される

**As a** MC
**I want** 同じメールが二重処理されない
**so that** 重複した LINE 通知や仮予定が作られない

**Priority**: MUST
**Size**: S
**Type**: Edge Case

**Acceptance Criteria**:
- **AC-1 (冪等性キー)**:
  - **Given** Pub/Sub から同一 `historyId` の Push が再送される
  - **When** Workers が処理を試みる
  - **Then** KV に保存された冪等性キーを参照し、既処理ならスキップする

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
- **AC-1 (配列抽出)**:
  - **Given** 複数日程記載のメール
  - **When** LLM が抽出する
  - **Then** `schedules: [{start_at, end_at}, ...]` として全候補が返る(最低 1 件以上)
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

### Story U2-EC-03: [Edge] PR 要素必要性の判定ヒューリスティクス

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

**Traceability**:
- Implements: FR-2

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
- **AC-1 (個別判定)**:
  - **Given** `schedules[]` に複数候補
  - **When** 判定が走る
  - **Then** 各スロットを独立に判定し、`{slot_id, status: free|partial_overlap|full_overlap}` を返す(FR-3)
- **AC-2 (Calendar 取得)**:
  - **Given** ユーザーの全カレンダー(プライマリ + サブ)
  - **When** 該当時間帯の予定を取得する
  - **Then** Google Calendar API `events.list` を時間範囲指定で呼び出す
- **AC-3 (集約)**:
  - **Given** 全スロットの判定結果
  - **When** 集約する
  - **Then** 1 つでも `free` があればエントリー可能、すべて `full_overlap` なら辞退対象とフラグ立て

**Traceability**:
- Implements: FR-3, NFR-1

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

### Story U3-04: 仮予定 / 確定予定 / 他カレンダーが区別される

**As a** MC
**I want** 自分が立てた仮予定と確定予定とプライベート予定が区別される
**so that** 「仮予定だから動かせる」「確定だから絶対無理」という判断が明確になる

**Priority**: SHOULD
**Size**: S
**Type**: User Story

**Acceptance Criteria**:
- **AC-1 (タイトル接頭辞判定)**:
  - **Given** カレンダー予定タイトル
  - **When** 判定する
  - **Then** `[仮]` 接頭辞 → 仮予定、それ以外 → 確定予定として扱う
- **AC-2 (カレンダー別)**:
  - **Given** プライマリ以外のカレンダー(プライベート等)
  - **When** 取得する
  - **Then** タグ `private` を付け、結果に区分が含まれる

**Traceability**:
- Implements: FR-3, FR-6

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
| F-07 | — | NFR-5 |
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
| Foundation | 7 | 2 (F-03, F-07) | 5 | 0 |
| Use Case 1 (F1) | 6 | 3 | 0 | 3 |
| Use Case 2 (F2) | 7 | 4 | 0 | 3 |
| Use Case 3 (F3) | 5 | 4 | 0 | 1 |
| Use Case 4 (F4) | 6 | 4 | 0 | 2 |
| Use Case 5 (F5) | 7 | 6 | 0 | 1 |
| Use Case 6 (F6) | 6 | 5 | 0 | 1 |
| Use Case 7 (F7) | 5 | 4 | 0 | 1 |
| Use Case 8 (F8) | 5 | 4 | 0 | 1 |
| Phase 2 | 7 (概略) | — | — | — |
| **合計(MVP+Foundation)** | **54** | **36** | **5** | **13** |

> Plan の目安(60〜100)よりやや少ない 54 Story で MVP を網羅。粒度をさらに細かく要望される場合は次回ループで分割可能です。
