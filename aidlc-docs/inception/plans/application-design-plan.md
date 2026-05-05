# Application Design — Plan & Questions

このドキュメントは Application Design ステージの **プラン + ユーザー質問** です。
回答後、`aidlc-docs/inception/application-design/` 配下に各設計成果物を生成します。

---

## 0. 前提資料
- 要件定義書: `aidlc-docs/inception/requirements/requirements.md`
- ペルソナ: `aidlc-docs/inception/user-stories/personas.md`(P1: AI 慣れ MC / P2: AI 未経験 MC)
- ストーリー: `aidlc-docs/inception/user-stories/stories.md`(58 Story)
- 実行計画: `aidlc-docs/inception/plans/execution-plan.md`(7 ユニット分解)

## 1. Application Design 方法論

以下のステップで設計成果物を作成します。各ステップに **チェックボックス** を付け、Step 10 の Generation で進捗管理します。

- [ ] **D-1**: 機能領域(Use Case 1〜7)とコンポーネントの対応マップを作成
- [ ] **D-2**: DDD レイヤ別にコンポーネントの責務・インターフェース(トレイト)を定義
- [ ] **D-3**: 各コンポーネントのメソッドシグネチャ(入出力型・エラー型)を定義(詳細な業務ロジックは Functional Design)
- [ ] **D-4**: サービス層(`application` レイヤのユースケース)を定義(オーケストレーションパターン)
- [ ] **D-5**: コンポーネント依存関係マトリクス + 通信パターン(同期 / 非同期 / Queue 経由)を作成
- [ ] **D-6**: データモデル(D1 スキーマ)の高レベル設計 — テーブル一覧 + 主要列 + 外部キー(詳細インデックスは Functional Design)
- [ ] **D-7**: API エンドポイント設計 — URL 体系 / 認証 / レート制限 / バージョニング
- [ ] **D-8**: C4 モデル(Context / Container / Component)の Mermaid 図を作成
- [ ] **D-9**: 5 つの設計成果物を `aidlc-docs/inception/application-design/` に保存
  - `components.md`(コンポーネント一覧と責務)
  - `component-methods.md`(メソッドシグネチャ)
  - `services.md`(サービス層オーケストレーション)
  - `component-dependency.md`(依存マトリクス + 通信パターン)
  - `application-design.md`(統合ドキュメント、データモデル + API 設計 + C4 図を含む)

## 2. 必須成果物
- [ ] `components.md` 生成
- [ ] `component-methods.md` 生成
- [ ] `services.md` 生成
- [ ] `component-dependency.md` 生成
- [ ] `application-design.md` 生成(統合ドキュメント、本ステージの主成果物)

---

## 3. 質問(回答必須)

これまでの成果物で多くの設計判断が確定しているため、**残された設計判断ポイントのみ** に絞って質問します(8 問)。

### Question Q1: コンポーネント粒度

`components.md` に列挙するコンポーネントの粒度はどれを採用しますか?

A) **粗粒度**: 7 ユニット = 7 コンポーネント程度(ユニットごとに 1 コンポーネント)
B) **中粒度**: 各ユニット内に 2〜4 個のコンポーネント = 全体で 15〜20 個 — 推奨
C) **細粒度**: 各 Story にほぼ対応するコンポーネント = 全体で 30〜40 個
X) Other (please describe after [Answer]: tag below)

[Answer]: B ユニットに合わせて柔軟にコンポーネント数は調整(1つでも可)。ただし、C のように多くなりすぎないほうがよい

---

### Question Q2: 非同期通信パターン(Queues 利用方針)

メール取込 → 分類 → 抽出 → 通知 のパイプラインで Cloudflare Queues をどう使いますか?

A) **ステップごとに別 Queue**(`classify_queue` / `extract_queue` / `notify_queue` ...)— 各 Consumer を独立したワーカーで動かし、再試行・DLQ もステップ単位で管理
B) **1 つの Queue + メッセージ種別で分岐**(`pipeline_queue` 1 本、メッセージに `step: "classify" | "extract" | ...` を持たせる)
C) **同期処理にする**(Queues を使わず、Webhook 受信時に分類 → 抽出 → 通知 まで直列実行) — Cloudflare Workers の CPU 制限で困難な可能性
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question Q3: エラー型設計

Rust のエラー型はどう構成しますか?

A) **ドメイン別の独立エラー enum**(`MailError` / `CalendarError` / `LineError` ...)を定義し、上位レイヤで `thiserror` の `#[from]` でラップ — モジュール独立性高
B) **統一エラー enum**(`AppError` 1 つにすべての variants を含める)— シンプルだが境界が曖昧
C) **U2-EC-04 共通失敗ハンドリング基盤と統合**(`Transient` / `Recoverable` / `DataIssue` / `Permanent` を最上位、その下にドメイン別をネスト) — 推奨
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question Q4: API エンドポイント設計の詳細度

`application-design.md` に書く API エンドポイント設計はどの粒度?

A) **URL + HTTP メソッド一覧のみ**(詳細なリクエスト/レスポンスは Functional Design 段階で記述)
B) **URL + メソッド + リクエスト / レスポンスのスキーマ概要**(JSON 構造)
C) **URL + メソッド + 認証ヘッダ + レート制限 + バージョニング戦略 まで含めた完全な API 仕様** — 推奨(F-04 AC-2 で「Application Design ステージで実施」と委譲済み)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question Q5: D1 スキーマ設計の詳細度

データモデル(D1 テーブル)はどこまで詳細化?

A) **テーブル一覧 + 主要列 + 外部キーのみ**(列の制約・インデックス設計は Functional Design)
B) **全列 + 制約 + 主要インデックス + マイグレーション順序**(本格的な ER 図) — 推奨
C) **追加で**: シードデータ・ビュー・トリガー(SQLite で対応する範囲)まで定義
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question Q6: 認証・認可の実装方針

API エンドポイントの認証・認可はどう実装しますか?(F-03 OAuth とは別の話、内部認可)

A) **すべての内部 API は無認証**(Cloudflare Workers の内部呼び出しのみで外部公開なし、Webhook は署名検証で十分) — MVP で簡素
B) **Webhook は署名検証、管理 API は Bearer トークン、ユーザー API は今後 LIFF + JWT**(MVP には管理 API のみ追加、ユーザー API は Phase 2)
C) **すべて Cloudflare Zero Trust(SSO 連携)で統一** — 開発者・運用者向けには有効、MVP には過剰
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question Q7: サービス層のトランザクション境界

「分類 → 抽出 → カレンダー判定 → 下書き作成 → 通知」の一連の処理を、どこまでトランザクション境界として扱う?

A) **各ステップを独立(eventually consistent)**: 各ステップで失敗しても他のステップに影響しない、補償処理(saga)で整合性を取る — 推奨(Queues + DLQ と整合)
B) **Use Case 全体を 1 トランザクション**: D1 トランザクション内で全ステップを実行、途中失敗でロールバック — Workers の CPU 制限で困難
C) **ハイブリッド**: D1 操作はトランザクション、外部 API 呼び出しは独立(失敗時の補償ハンドラを Story 別に定義)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question Q8: テストダブル(モック)の境界

ユニットテスト・結合テストでモック / フェイクをどこに作る?

A) **F-11 抽象化レイヤのトレイト境界でモック**(`MailRepository` / `CalendarRepository` / `NotificationChannel` / `LlmClient` のモック実装を `crates/shared/test_support` に置く)— 推奨
B) **各ストアごとにモック**(D1 / KV / R2 / Anthropic / Gmail / Calendar / LINE 全部別に)
C) **ステージング環境を再現**(Wrangler Miniflare + WireMock 等で実物に近い動作)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 4. 回答後の流れ

すべての `[Answer]:` を埋めたら、PR にコメントいただくか、チャットで「**回答完了**」または「**done**」とお知らせください。

回答後:
1. 回答の矛盾・曖昧チェック(必要ならフォローアップ質問)
2. プラン承認
3. **Step 10 で 5 つの設計成果物を生成**(components.md / component-methods.md / services.md / component-dependency.md / application-design.md)
4. 承認 → Units Generation へ進行
