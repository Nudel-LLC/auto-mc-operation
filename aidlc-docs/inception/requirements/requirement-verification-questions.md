# 要件確認 Questions

要件分析を完成させるため、以下の質問にご回答ください。
各質問の `[Answer]:` タグの後に **A〜X のレター** を記入してください。
選択肢に該当しない場合は **X) Other** を選び、説明を追記してください。

---

## カテゴリ A: プロダクトのスコープと利用者像

### Question 1: 利用ユーザーの規模感
このシステムは誰が利用しますか?

A) 開発者本人(または特定の知人) 1名のみが使う「個人ツール」
B) 限定公開の数名〜数十名のクローズドβ版
C) 複数のMC/コンパニオンが利用する SaaS(マルチテナント)
D) まずは1名で MVP を出し、後にマルチテナント SaaS へ拡張
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

### Question 2: MVP として優先度の高い機能はどれですか?
複数選択可。優先度の高いものを **カンマ区切り** で記入してください(例: `A, C, D`)。
それ以外を Phase 2 以降に回します。

A) Gmailメール自動分類(案件募集 / 決定連絡 / その他)
B) 案件情報抽出(日程・場所・報酬)
C) Googleカレンダーとの空き確認・重複検知
D) エントリー下書き自動作成(PR文生成含む)
E) カレンダー自動管理(仮予定登録/確定更新/削除)
F) LINE通知
G) 請求データ CSV エクスポート
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 3: エントリー送信(返信メール送信)の自動化レベル
エントリー希望の返信メールはどこまで自動化しますか?

A) 完全自動送信(下書き生成→確認なしで自動送信)
B) 半自動(下書き生成→ユーザーがLINE等で承認したら送信)
C) 下書きのみ生成し、Gmailの下書きフォルダに保存(送信はユーザーが手動)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 4: 辞退メール送信の自動化レベル
決定案件と重複した仮案件への辞退メールはどこまで自動化しますか?

A) 完全自動送信(検出→定型文で自動送信)
B) 半自動(検出→ユーザー承認後に送信)
C) 通知のみ(検出してLINEで知らせるが送信はユーザーが手動)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## カテゴリ B: 外部サービス連携

### Question 5: Gmail との連携方式
Gmail からのメール取得はどの方式を想定していますか?

A) Gmail API + Push通知(Pub/Sub Watch、ほぼリアルタイム)
B) Gmail API + 定期ポーリング(例: 5分間隔)
C) IMAP接続による取得
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 6: 認証/認可(OAuth スコープ)
Google系API(Gmail, Calendar)のOAuth認可はどのスコープを想定していますか?

A) 読み取り+書き込み(メール送信・カレンダー編集を含むフルアクセス)
B) 読み取りのみ(送信・編集はユーザーが手動)
C) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 7: LINE 通知の方式
LINE通知はどの方式を使いますか?(LINE Notify は2025年3月で終了予定のため代替検討が必要)

A) LINE Messaging API(公式Botを作成して個人/グループに送信)
B) LINE Notify(終了予定だが既存トークンを利用)
C) 通知は別の手段(メール、Slack、Webプッシュ等)に変更
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 8: AI/LLM の利用
メール分類・情報抽出・PR文生成に使うLLMは?

A) Claude Haiku のみ(コスト最適化)
B) Claude Haiku(分類)+ Claude Sonnet(複雑な抽出/生成)の使い分け
C) Claude Opus を含む全モデル使い分け(品質重視)
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## カテゴリ C: 技術スタック & デプロイ

### Question 9: バックエンドの実装言語/フレームワーク
バックエンドはどの言語・フレームワークで実装しますか?

A) Python(FastAPI など)
B) TypeScript / Node.js(NestJS / Express など)
C) Go
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: X 全体構成を踏まえた上で相談したい。動的型付け言語は採用しない予定。Rust など言語的に制約の多い方が AI 開発(ハルシネーション等によるバグの発生懸念)との相性が良いのではないかと考えている。

---

### Question 10: デプロイ先
本番運用のホスティング先は?

A) AWS(Lambda + EventBridge + DynamoDB / RDS など サーバレス中心)
B) Google Cloud(Cloud Run + Cloud Scheduler + Firestore など)
C) Vercel / Cloudflare Workers などの PaaS
D) 自宅サーバ / VPS(常時稼働の Linux サーバ)
E) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: X MVP の段階では Cloudflare の予定。拡張の際は GCP を採用する予定。

---

### Question 11: データ永続化(DB)
案件・エントリー・実績PRなどのデータをどこに保存しますか?

A) リレーショナル DB(PostgreSQL / MySQL / RDS / Cloud SQL)
B) NoSQL ドキュメント DB(DynamoDB / Firestore / MongoDB)
C) ファイル(SQLite / JSON / CSV のローカル保存)
D) Google スプレッドシート連携(GUI で確認できるように)
E) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: X データベース設計によって A か B か決定したい。

---

### Question 12: ユーザー向け UI
ユーザーが操作する UI は何を作りますか?

A) UI なし(LINE通知 + メール下書きのみで完結)
B) シンプルな Web ダッシュボード(案件一覧、設定変更、CSV出力)
C) スマホネイティブアプリ
D) LINE 内のリッチメニュー/LIFF アプリ
E) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## カテゴリ D: 非機能要件

### Question 13: メール処理のレイテンシ要件
案件募集メールを受信してから LINE 通知が届くまでの目標時間は?

A) ほぼリアルタイム(1分以内)
B) 数分以内(5分以内)
C) 1時間以内
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 14: 想定処理量(1日あたり)
1日あたりに処理する案件募集メールはどれくらいを想定しますか?

A) 〜20件/日
B) 20〜100件/日
C) 100〜500件/日
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: X 1ユーザーあたりは 20件以下だが、サービス拡大を狙っているので、規模の拡大に応じて増える予定。

---

### Question 15: 月額運用コストの目安
LLM API + ホスティング + 各種APIを含めた月額の許容コストは?

A) 〜1,000円/月
B) 1,000〜5,000円/月
C) 5,000〜20,000円/月
D) コストよりも品質・速度を優先する
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

### Question 16: データ保存期間とプライバシー
案件メール・カレンダー情報の取り扱いは?

A) すべての処理結果を永続保存(分析・PR文の精度向上に利用)
B) 直近Nヶ月のみ保存し、それ以前は自動削除
C) 揮発的に処理し、最低限のメタデータのみ保存
X) Other (please describe after [Answer]: tag below)

[Answer]: C 逆に A や B だと、日本の通信法などに抵触しないか、教えてください。

---

## カテゴリ E: 業務ロジックの細部

### Question 17: 案件情報抽出の必須項目
メールから抽出して構造化したい項目は?(複数選択可、カンマ区切り)

A) 日程(開始日時・終了日時)
B) 場所(地名・会場名・住所)
C) 報酬(金額・支払条件)
D) 案件種別(MC/コンパニオン/司会/ナレーション 等)
E) 衣装・持ち物指定
F) 事務所名(送信元から自動推定)
G) 締切日時(エントリー期限)
X) Other (please describe after [Answer]: tag below)

[Answer]: A,B,C,F,X 案件名、PR要素の有無(自己PR記載が求められているか)、その他条件

---

### Question 18: スケジュール重複の判定ロジック
重複判定はどこまで厳密に行いますか?

A) 完全重複のみ(時間帯が同一)
B) 部分重複も検知(時間帯の一部が被る)
C) 部分重複 + 移動時間バッファ(例: 前後1時間)も考慮
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 19: PR文生成の元データ
過去実績を元にPR文を生成する際の入力ソースは?

A) 過去にユーザーが書いたエントリーメールから自動学習
B) ユーザーが手動で登録した実績データベース(別途UIで入力)
C) Google スプレッドシート等の既存実績シートを取り込み
D) A + B のハイブリッド
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

### Question 20: CSV エクスポートの内容
請求用CSVに含めたい項目は?(複数選択可、カンマ区切り)

A) 月、事務所名、案件名、日程、場所、報酬
B) 上記 + ステータス(確定/辞退/未決定)
C) 上記 + 源泉徴収/消費税/振込手数料
D) 任意 — 開発者(あなた)に決めてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## カテゴリ F: 拡張機能(opt-in 確認)

### Question 21: Security Baseline 拡張
セキュリティ拡張ルールをこのプロジェクトで強制(blocking 制約)しますか?

A) Yes — すべての SECURITY ルールをブロッキング制約として強制(本番運用するアプリケーションに推奨)
B) No — SECURITY ルールはスキップ(PoC・プロトタイプ・実験用に適切)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 22: Property-Based Testing 拡張
プロパティベーステスト(PBT)ルールをこのプロジェクトで強制しますか?

A) Yes — すべての PBT ルールをブロッキング制約として強制(ビジネスロジック・データ変換・シリアライズ・ステートフルコンポーネントを含むプロジェクトに推奨)
B) Partial — pure function とシリアライズ往復のみ PBT を強制(アルゴリズム複雑度が限定的なプロジェクトに適切)
C) No — PBT ルールはスキップ(シンプルな CRUD アプリ・UI のみのプロジェクト・薄い統合層に適切)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 回答後

すべての `[Answer]:` を埋めたら、チャットで「**回答完了**」または「**done**」とお知らせください。
回答内容を分析し、矛盾や曖昧さがあればフォローアップ質問を作成します。問題なければ要件定義書 `requirements.md` の作成に進みます。
