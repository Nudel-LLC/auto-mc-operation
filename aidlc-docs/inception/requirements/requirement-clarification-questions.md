# 要件確認 — 追加質問・矛盾解消・推奨案の提示

最初の 22 件のご回答を分析した結果、以下の **矛盾** と **追加検討が必要な点** が見つかりました。
回答内容と推奨案を提示しますので、各 `[Answer]:` をお願いします。

---

## Cl-1: 【矛盾】Q12 (UI なし) と Q19 (PR文の元データに B = 手動登録 DB を含む) の整合性

**ご回答**:
- Q12 (UI): **A) UI なし(LINE 通知 + メール下書きのみで完結)**
- Q19 (PR文の元データ): **D) A + B のハイブリッド** (= 過去エントリーメール学習 + 手動登録された実績DB)

**矛盾点**:
B「ユーザーが手動で登録した実績データベース」を成立させるには、**実績を入力する UI(または取り込み手段)が必要**です。
Q12 で UI を持たない選択をしたため、B の実現方法が不明確です。

### Question Cl-1
PR 文の元データはどう扱いますか?

A) Q19 を **A のみ(過去エントリーメールから自動学習)** に変更し、UI なし方針を維持
B) Q12 を **B(シンプルな Web ダッシュボード)** に変更し、Phase 1 から最小限の実績登録 UI を持つ
C) UI なしのまま、実績は **Google スプレッドシート連携(Q19 の C と同等)** で取り込む。スプレッドシートをユーザーが手動更新する運用
D) MVP は A のみ、Phase 2 で UI を追加して B にスコープ拡張する(段階的アプローチ)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Cl-2: 【MVP スコープ再確認】Q2 で A のみ選択された件

**ご回答**: Q2 (MVP 機能) = **A) Gmail メール自動分類のみ**

**論点**:
A だけだと「分類はされるがその後の業務(空き確認・エントリー下書き・LINE 通知・カレンダー反映)はすべて手動」となり、当初の業務負担削減目的(課題2.3「ウェットなコミュニケーションコスト」)を MVP では達成しづらくなります。

最小限の **エンドツーエンド価値** を出すには、最低限以下が揃うのが一般的です:
- A: 分類(案件募集を見つける)
- B: 情報抽出(日程・場所・報酬を構造化)
- C: カレンダー空き確認(エントリー判断材料)
- F: LINE 通知(ユーザーの導線)

### Question Cl-2
MVP のスコープを再選択してください。

A) **当初通り A のみ** で MVP リリース(他機能は Phase 2)。意図的な小さな MVP
B) **A + B + F**(分類 + 情報抽出 + LINE 通知)— ユーザーは LINE で案件サマリを受け取る
C) **A + B + C + F**(分類 + 情報抽出 + 空き確認 + LINE 通知)— 空き状況も LINE に含めて意思決定を支援
D) **A + B + C + D + F**(さらに下書き作成も含む)— Q3 (C: 下書きのみ Gmail 保存) と Q4 (B: 半自動辞退) を含む完全パイプライン
E) **全機能(A〜G)を MVP に含める**
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Cl-3: 【相談】Q9 技術スタック — 推奨案の提示

**ご回答(要約)**: 動的型付け不採用、Rust など型制約の強い言語が AI 開発との相性に有利と想定。全体構成を踏まえた相談希望。

**調査結果サマリ**(Cloudflare Workers MVP → GCP 拡張 を前提に比較):

| 観点 | TypeScript (strict) | Rust (WASM) | Go |
|------|---------------------|-------------|----|
| 型安全性 | 高 (strict + Zod) | 最高 (代数的データ型・所有権) | 中 (zero-value/nil) |
| Workers 対応 | ネイティブ・最速 | workers-rs で WASM 対応・サイズ制約あり | **実質非対応** |
| GCP 移行容易性 | Cloud Run on Node/Bun で容易 | 単一バイナリで容易 | Cloud Run 一級 |
| SDK エコシステム (Gmail/Calendar/LINE/Anthropic) | 公式 SDK 完備 | ほぼ自前(reqwest + 型定義)、Anthropic は非公式 | 公式 SDK あり、LINE は薄い |
| AI エージェント親和性 | 学習データ最多・修正容易 | 借用/lifetime で生成失敗率高、ただしコンパイラ厳格は利点 | 中庸 |

**Primary 推奨**: **TypeScript (strict) + Hono + Zod + Drizzle ORM**
- Cloudflare Workers ネイティブ実行で性能・コスト最適
- Drizzle は D1(SQLite)→ Cloud SQL(PostgreSQL)のスキーマ移行をほぼ無改修で実現
- Anthropic / Google API の公式 TS SDK が揃う
- AI コーディングエージェントの生成精度・修正容易性が最高
- プロパティベーステストは `fast-check`、セキュリティは Hono middleware で標準化

**Alternative**: **Rust (workers-rs) + axum(Cloud Run 拡張時)**
- 型安全性・性能・単一バイナリ移植性で最強
- 公式 Anthropic/LINE SDK 不在、AI エージェントの借用エラー修正コストが高い
- 開発速度より長期運用コスト最小化を最優先する場合に推奨

(Go は Cloudflare Workers 非対応のため MVP 制約に反し除外)

### Question Cl-3
バックエンド言語・フレームワークの最終決定:

A) **Primary 推奨を採用**(TypeScript strict + Hono + Zod + Drizzle ORM)
B) **Alternative を採用**(Rust workers-rs + axum)
C) **MVP は TypeScript、拡張時に Rust への部分書き換えを検討**(段階的)
X) Other (please describe after [Answer]: tag below)

[Answer]: X Java も選択肢に加えてください。また Rust は最近流行りの言語ですが、本当に SDK はあまりないのですか?SDK がないなら REST API の利用などになりますかね?それもそれでシンプルそうですが。 → **Cl-3-followup を参照**

---

## Cl-4: 【相談】Q11 データ永続化 — 推奨案の提示

**ご回答**: 「データベース設計によって A か B か決定したい」

**論点**:
本ドメインは **案件・エントリー・カレンダー・実績PR・事務所** の間に明確な参照関係(FK)があり、月次集計(請求 CSV)・重複検知などのクエリも必要。リレーショナルが自然です。

**Cloudflare MVP 推奨**: **D1 (SQLite at edge)**
- リレーショナル + トランザクション + SQL クエリで上記用途すべてに対応
- KV は補助(セッション・冪等性キー)、R2 はメール原文/添付の保管に使用
- Durable Objects + SQLite は per-user 強整合が必要な箇所(Gmail watch 状態管理)に限定使用

**GCP 拡張時推奨**: **Cloud SQL (PostgreSQL)**
- D1 → Postgres は SQL 方言差が小さく Drizzle ORM でほぼ無改修移行可能
- Firestore はリレーション表現に不向きで本ドメインに適さない

### Question Cl-4
DB 構成の最終決定:

A) **推奨案を採用**(MVP: D1 + KV(補助) + R2(メール原文) → 拡張時 Cloud SQL PostgreSQL)
B) **MVP から PostgreSQL に統一**(Cloudflare D1 ではなく、Hyperdrive 経由で Neon/Supabase の Postgres を最初から使用)
C) **Firestore など NoSQL を採用**(リレーションは ORM 層で擬似的に表現)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Cl-5: 【スケール想定】Q14 処理量の段階的目標

**ご回答**: 「1ユーザーあたり 20件/日。サービス拡大に応じて増える予定」

**論点**:
非機能要件・コスト試算・アーキテクチャ判断のため、段階的なスケール想定を確定したいです。

### Question Cl-5
将来の規模想定の上限値はどれくらいですか?

A) **小規模 SaaS**: ピーク時 100 ユーザー × 20件/日 = 2,000 件/日
B) **中規模 SaaS**: ピーク時 500 ユーザー × 20件/日 = 10,000 件/日
C) **大規模 SaaS**: ピーク時 5,000 ユーザー × 20件/日 = 100,000 件/日
D) **未定**(MVP では 1ユーザー × 20件/日のみ確実、上限は走りながら判断)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Cl-6: 【法律ガイダンス】Q16 データ保存と日本の通信法・個情法について

**ご質問**: 「A や B(永続/期限付き保存)だと、日本の通信法などに抵触しないか?」

**一般的ガイダンス**(法的助言ではなく、設計時の考慮点として):

1. **電気通信事業法「通信の秘密」** (法 4 条):
   - 「通信の媒介」を業として行う者(電気通信事業者)に課される義務。
   - **本サービスは「ユーザー本人が受信したメールを、本人の同意のもとに本人のために処理代行する」位置付け** であり、第三者の通信を媒介していないため、電気通信事業者には**該当しない**(あるいは届出不要の利用者として整理される)というのが一般的解釈です。
   - 総務省ガイドライン上、**ユーザー本人の同意取得**(明確な利用目的の説明・同意ボタンなど)が前提となります。

2. **個人情報保護法**:
   - メール本文に含まれる **第三者(事務所担当者・顧客)の氏名・連絡先** は個人情報。
   - 必要な対応:
     - **利用目的の特定・通知**(プライバシーポリシー)
     - **安全管理措置**(暗号化保存、アクセス制御、Security Baseline 拡張で担保)
     - **第三者提供禁止**(LLM への送信は「業務委託」整理。Anthropic は SOC2/ISO27001 取得済で米国保管。海外移転に該当するため**ユーザー同意必須**)
     - **保存期間の明示**

3. **データ保存方針の設計上の落としどころ**(よく採られる構成):
   - **メール本文(原文)** は短期(例: 7〜30日)で自動削除 — 通信内容そのものの長期保管はリスク。
   - **抽出された構造化データ**(日程・場所・報酬等)は業務遂行に必要な期間 + α 保管。
   - **PR 文学習用データ**(過去エントリー文)は本人作成の文章=本人帰属、保管 OK。
   - **監査ログ・処理メタデータ**(分類結果・送信履歴)は短〜中期保管。

**結論**: A/B も法律上は **ユーザー同意+適切な管理** で可能ですが、リスク最小化の観点では **「原文は短期、構造化データは中期、PR文学習データは長期」のハイブリッド** が一般的です。

### Question Cl-6
データ保存方針の最終決定:

A) **完全揮発(C)を維持**: 処理後はメタデータのみ残し、原文・抽出データも即削除
B) **ハイブリッド**: メール原文 30日 / 構造化データ 24ヶ月 / PR文学習データ 永続(本人帰属) / 監査ログ 12ヶ月
C) **長期保管(A)**: すべて永続保存(精度向上を最優先、ユーザー同意とプライバシーポリシーで担保)
D) **直近 N ヶ月のみ(B 元案)**: N の値を以下に明記
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Cl-7: 【明確化】Q17 抽出項目で D/E/G の扱い

**ご回答**: A, B, C, F, X(案件名 / PR要素の有無 / その他条件)

**論点**: 元の選択肢のうち以下が含まれていません:
- D) 案件種別(MC / コンパニオン / 司会 / ナレーション 等)
- E) 衣装・持ち物指定
- G) 締切日時(エントリー期限)

特に **G) 締切日時** はエントリー判断・通知タイミングに重要なため、抽出するかは早期確定したいです。

### Question Cl-7
D / E / G の扱い:

A) すべて **抽出対象に追加**(精度より網羅性優先)
B) **G(締切日時)のみ追加**(D・E は不要)
C) **D(案件種別)と G(締切日時)を追加**(E は不要)
D) **追加せず A,B,C,F + X のみ**(意図的に絞る)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Cl-3-followup: 【Cl-3 への回答 + 追加検討】Java の選択肢追加と Rust SDK の実態

**ご質問への回答**:

### A. Cloudflare Workers での Java サポート(2026年5月時点)
- **ネイティブサポートは無し**。Workers は V8 isolate ベースで JVM を起動できません。
- TeaVM / GraalVM Native Image での WASM 化は理論上可能ですが、Workers の WASM 制約(サイズ上限、起動時間、GC 制限)で**実用困難**です。Kotlin/JS や Scala.js は動きますが Java 資産流用に限界あり。
- **Java を採用するなら Cloudflare ではなく Cloud Run / GKE / VPS を Day 1 から使う必要があります**(MVP=Cloudflare の方針と両立しません)。

### B. Rust 向け SDK の実態
3 つとも **公式 SDK は無し** (2026年5月時点) です。

| API | 公式 Rust SDK | 代替手段 | 実装コスト |
|-----|---------------|----------|-----------|
| Anthropic (Claude) | ❌ | 非公式 `anthropic-sdk` 系 / `reqwest` 直叩き | **低**(数十行で済む、JSON 構造単純) |
| Google APIs (Gmail / Calendar) | ❌ | コミュニティ `google-apis-rs`(網羅的だが巨大)/ `yup-oauth2`(OAuth2) | **中**(OAuth2 は SDK 利用推奨) |
| LINE Messaging | ❌ | 非公式 `line-bot-sdk-rust` / REST 直叩き | **低〜中**(署名検証含めて 100〜200 行) |

**結論**: `reqwest` + `serde` で REST 直叩きは現実的(Google OAuth のみ `yup-oauth2` 推奨)。コード量は増えますが、依存が少なく、APIの仕様変更に柔軟に追従できます。「SDK の薄さ = シンプル」というご指摘の通りの側面もあります。

### C. 改めて選択肢を整理

| 選択肢 | 言語 | MVP プラットフォーム | 拡張時 | コメント |
|--------|------|---------------------|--------|---------|
| **a** | **TypeScript (strict)** | Cloudflare Workers | Cloud Run (Node/Bun) | 公式 SDK 完備・AI エージェント親和性最高 |
| **b** | **Rust** | Cloudflare Workers (workers-rs) | Cloud Run | 型安全性最高・REST 直叩き、AI エージェントは借用エラー修正がやや重い |
| **c** | **Java (Spring Boot / Quarkus)** | **Cloud Run(Cloudflare 不可)** | Cloud Run / GKE | 公式 SDK 全揃い・成熟・ただし MVP のホスティング方針変更が必要 |
| **d** | TypeScript MVP → Rust 部分書き換え | Cloudflare Workers | Cloud Run | 段階的移行 |

---

### Question Cl-3-followup
バックエンド言語・プラットフォームの最終決定:

A) **TypeScript (strict)** + Cloudflare Workers MVP → Cloud Run 拡張(初回提案の Primary 案)
B) **Rust** + Cloudflare Workers (workers-rs) → Cloud Run 拡張(REST 直叩き許容)
C) **Java** + Cloud Run from Day 1(MVP ホスティング方針を Cloudflare → Cloud Run に変更する)
D) **TypeScript MVP → Rust 部分書き換え**(段階的)
X) Other (please describe after [Answer]: tag below)

[Answer]: 

**注**: C を選ぶ場合、Cl-4 (DB) の推奨も Cloud SQL (PostgreSQL) を MVP から採用する形に変わります(D1 不採用)。

---

## 回答後

Cl-3-followup の `[Answer]:` を埋めたら、PR にコメントいただくか、チャットで「**回答完了**」または「**done**」とお知らせください。

このフォローアップで論点が解消したら `aidlc-docs/inception/requirements/requirements.md` を作成します。
