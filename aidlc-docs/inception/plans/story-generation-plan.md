# User Stories — Story Generation Plan(Part 1: Planning)

このドキュメントは User Stories ステージの **Part 1: Planning** の成果物です。
質問への回答後、Part 2: Generation で `stories.md` と `personas.md` を生成します。

---

## 0. 前提資料
- 要件定義書: `aidlc-docs/inception/requirements/requirements.md`
- アセスメント: `aidlc-docs/inception/plans/user-stories-assessment.md`(User Stories 実行は Yes)

---

## 1. Story 開発の方法論(plan の概要)
以下のステップで User Stories を作成します。各ステップに **チェックボックス** を付けて Part 2 で進捗管理します。

- [ ] **S-1**: 要件定義書から **ペルソナ** を抽出・記述する(`personas.md`)
- [ ] **S-2**: 業務フローを論理的なエピックに分解する
- [ ] **S-3**: 各エピック配下に「As a... I want... so that...」形式の **User Story** を作成する
- [ ] **S-4**: 各 Story に **INVEST 原則**(Independent / Negotiable / Valuable / Estimable / Small / Testable)を満たすよう調整する
- [ ] **S-5**: 各 Story に **受入条件**(Acceptance Criteria)を付与する
- [ ] **S-6**: ペルソナと Story のマッピング表を作成する
- [ ] **S-7**: エッジケース・エラーシナリオを別 Story として明示する(必要箇所)
- [ ] **S-8**: 要件(FR-1〜FR-8 / NFR-1〜NFR-8)と Story のトレーサビリティ表を作成する
- [ ] **S-9**: 成果物 `personas.md` と `stories.md` を `aidlc-docs/inception/user-stories/` に保存する

---

## 2. 必須成果物(plan に含めること)
- [ ] `aidlc-docs/inception/user-stories/personas.md` を生成
- [ ] `aidlc-docs/inception/user-stories/stories.md` を生成
- [ ] 各 Story が INVEST 原則に準拠
- [ ] 各 Story に受入条件(Acceptance Criteria)を付与
- [ ] ペルソナと Story のマッピング
- [ ] FR/NFR と Story のトレーサビリティ

---

## 3. Story Breakdown のアプローチ候補
以下から採用するアプローチを Q4 でご選択ください。

| アプローチ | 説明 | この案件での適合度 |
|-----------|------|------------------|
| **A. User Journey-Based** | ユーザーの業務フロー(受信 → 分類 → 抽出 → 空き確認 → 下書き → 通知 → 管理 → 辞退 → 請求)に沿って Story を並べる | ◎ 業務フロー横断のため自然 |
| **B. Feature-Based** | F1〜F8 の機能ごとに Story を整理 | ○ 要件と直結し追跡しやすい |
| **C. Persona-Based** | ペルソナごとに Story をグループ化 | △ プライマリペルソナがほぼ 1 種で恩恵小 |
| **D. Domain-Based** | ドメイン(メール / カレンダー / 通知 / 学習 / 請求)単位で整理 | ○ 設計時のコンポーネント境界に近い |
| **E. Epic-Based** | 大きなエピックを階層化、配下に Story を配置 | ◎ MVP→Phase 2→Phase 3 を明示でき拡張時の見通し良 |

→ ハイブリッド(例: Epic + User Journey)も可

---

## 4. 質問(回答必須)

### Question Q1: ペルソナのスコープ

`personas.md` に含めるペルソナはどの範囲まで?

A) **プライマリのみ**: MC / コンパニオン本人のみ詳述
B) **プライマリ + 周辺関係者**: MC + 事務所担当者(送信元としての関係者)
C) **プライマリ + 周辺関係者 + サービス運営者**: B + 開発者・運営側(SaaS 提供者)
D) **プライマリ + サービス運営者のみ**: A + サービス運営者(事務所は省略)
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q2: Story の粒度

各 Story の粒度はどれを採用しますか?

A) **細粒度**: 1 Story = 1 受入条件、データベース操作・API 呼び出しレベルまで分解(数十ストーリー)
B) **中粒度**: 1 Story = 1 ユーザーアクション(例: 「LINE で案件サマリを受け取り、承認/却下する」) — 推奨、INVEST に最も合致
C) **大粒度**: 1 Story = 1 機能領域(F1〜F8 を 1 ストーリーずつ)
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q3: 受入条件のフォーマット

受入条件はどのフォーマットで記述しますか?

A) **Given-When-Then(Gherkin)形式**: テスト自動化と相性が良い、PBT/E2E テストに転用しやすい
B) **箇条書きチェックリスト**: 軽量、レビューが速い
C) **両併記**: Given-When-Then + 補足箇条書き(冗長だが網羅性高)
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q4: Story Breakdown のアプローチ

セクション 3 のいずれを採用しますか?

A) **A. User Journey-Based のみ**
B) **B. Feature-Based のみ**(F1〜F8 を軸)
C) **D. Domain-Based のみ**(メール / カレンダー / 通知 / 学習 / 請求)
D) **E. Epic-Based + A(User Journey)**: MVP / Phase 2 / Phase 3 を Epic、配下を業務フロー順 — 推奨
E) **B + E のハイブリッド**: F1〜F8 を Feature Epic、配下に User Journey 順の Story
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q5: MVP / Phase 区分

`stories.md` で MVP と将来 Phase をどこまで明記しますか?

A) **MVP のみ**: Phase 2/3 の Story は記述しない(後で別途追加)
B) **MVP + Phase 2 概略**: MVP は完全記述、Phase 2 はタイトル+概要のみ
C) **MVP + Phase 2 + Phase 3 の全体俯瞰**: 全 Phase を記述、Phase 2/3 は粒度低め
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q6: エラー / エッジケースの扱い

メール抽出失敗・LLM タイムアウト・OAuth 失効・部分重複の境界等のエッジケースは?

A) **正常系 Story の受入条件内に併記**(別 Story にしない)
B) **「異常系/エッジケース Story」として独立**(可視性高、テスト管理しやすい)
C) **重要度の高いものだけ独立 Story に**、軽微なものは正常系の受入条件に併記
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q7: Story の優先度・サイズ表記

Story に優先度やサイズを付けますか?

A) **優先度のみ**(MUST / SHOULD / COULD = MoSCoW)
B) **優先度 + サイズ**(MoSCoW + S/M/L または Story Points)
C) **付けない**(Workflow Planning 段階で別途扱う)
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

### Question Q8: 要件トレーサビリティの粒度

Story と要件の対応表はどこまで詳細化しますか?

A) **Story → FR/NFR の 1 対多マッピング**(MVP に十分)
B) **Story → FR/NFR の 1 対多 + 受入条件 → 個別 NFR(例: SECURITY-XX)もマップ**
C) **付けない**(Application Design で別途整理)
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## 5. 回答後

すべての `[Answer]:` を埋めたら、PR にコメントいただくか、チャットで「**回答完了**」または「**done**」とお知らせください。

回答内容を分析(Step 9: ANALYZE ANSWERS)し、矛盾・曖昧があればフォローアップ質問を作成します。問題なければプランの承認をお願いし、その後 Part 2: Generation で `personas.md` と `stories.md` を作成します。
