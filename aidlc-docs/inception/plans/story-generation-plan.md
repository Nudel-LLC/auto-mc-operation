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

[Answer]: A

---

### Question Q2: Story の粒度

各 Story の粒度はどれを採用しますか?

A) **細粒度**: 1 Story = 1 受入条件、データベース操作・API 呼び出しレベルまで分解(数十ストーリー)
B) **中粒度**: 1 Story = 1 ユーザーアクション(例: 「LINE で案件サマリを受け取り、承認/却下する」) — 推奨、INVEST に最も合致
C) **大粒度**: 1 Story = 1 機能領域(F1〜F8 を 1 ストーリーずつ)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question Q3: 受入条件のフォーマット

受入条件はどのフォーマットで記述しますか?

A) **Given-When-Then(Gherkin)形式**: テスト自動化と相性が良い、PBT/E2E テストに転用しやすい
B) **箇条書きチェックリスト**: 軽量、レビューが速い
C) **両併記**: Given-When-Then + 補足箇条書き(冗長だが網羅性高)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question Q4: Story Breakdown のアプローチ

セクション 3 のいずれを採用しますか?

A) **A. User Journey-Based のみ**
B) **B. Feature-Based のみ**(F1〜F8 を軸)
C) **D. Domain-Based のみ**(メール / カレンダー / 通知 / 学習 / 請求)
D) **E. Epic-Based + A(User Journey)**: MVP / Phase 2 / Phase 3 を Epic、配下を業務フロー順 — 推奨
E) **B + E のハイブリッド**: F1〜F8 を Feature Epic、配下に User Journey 順の Story
X) Other (please describe after [Answer]: tag below)

[Answer]: D, A, E ディレクトリ構成や開発ルール、インフラとか基盤部分は先に行い、その後はユースケースごとに開発していく

---

### Question Q5: MVP / Phase 区分

`stories.md` で MVP と将来 Phase をどこまで明記しますか?

A) **MVP のみ**: Phase 2/3 の Story は記述しない(後で別途追加)
B) **MVP + Phase 2 概略**: MVP は完全記述、Phase 2 はタイトル+概要のみ
C) **MVP + Phase 2 + Phase 3 の全体俯瞰**: 全 Phase を記述、Phase 2/3 は粒度低め
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question Q6: エラー / エッジケースの扱い

メール抽出失敗・LLM タイムアウト・OAuth 失効・部分重複の境界等のエッジケースは?

A) **正常系 Story の受入条件内に併記**(別 Story にしない)
B) **「異常系/エッジケース Story」として独立**(可視性高、テスト管理しやすい)
C) **重要度の高いものだけ独立 Story に**、軽微なものは正常系の受入条件に併記
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question Q7: Story の優先度・サイズ表記

Story に優先度やサイズを付けますか?

A) **優先度のみ**(MUST / SHOULD / COULD = MoSCoW)
B) **優先度 + サイズ**(MoSCoW + S/M/L または Story Points)
C) **付けない**(Workflow Planning 段階で別途扱う)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question Q8: 要件トレーサビリティの粒度

Story と要件の対応表はどこまで詳細化しますか?

A) **Story → FR/NFR の 1 対多マッピング**(MVP に十分)
B) **Story → FR/NFR の 1 対多 + 受入条件 → 個別 NFR(例: SECURITY-XX)もマップ**
C) **付けない**(Application Design で別途整理)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## 5. 回答分析(Step 9: ANALYZE ANSWERS)

| Q | 回答 | 分析 |
|---|------|------|
| Q1 | A: プライマリのみ | ✅ 明確 |
| Q2 | A: 細粒度 | ⚠️ 私の選択肢文「1 Story = 1 受入条件」は不正確だった(通常 1 Story には複数の受入条件)。実意は「最小ユーザーアクション = 1 Story」と解釈。Cl-S2 で確認 |
| Q3 | A: Given-When-Then | ✅ 明確 |
| Q4 | D, A, E + コメント「基盤先行→ユースケース別」 | ⚠️ 複数選択は意図不明だが、コメントから方針は明確: **Foundation Epic + Use Case Epics 階層**、各 Epic 配下を業務フロー順 |
| Q5 | B: MVP + Phase 2 概略 | ✅ 明確 |
| Q6 | B: エッジケース独立 Story | ✅ Q2(細粒度)と整合 |
| Q7 | B: MoSCoW + サイズ | ✅ 明確 |
| Q8 | B: Story → FR/NFR + 受入条件 → 個別 NFR | ✅ Q2(細粒度)と整合 |

---

## 6. Decided Approach(統合的な構成方針)

ご回答を踏まえた `stories.md` の構成方針:

### 6.1 Epic 階層
```
└── Foundation Epic (基盤・最初に開発)
    ├── Story F-01: プロジェクトのディレクトリ構成・開発ルール整備(Enabler)
    ├── Story F-02: Cloudflare Workers + D1 + KV + R2 + Queues の基盤セットアップ(Enabler)
    ├── Story F-03: Google OAuth(Gmail/Calendar)認可フロー(Enabler/User)
    ├── Story F-04: LINE Messaging API Bot 連携セットアップ(Enabler)
    ├── Story F-05: Anthropic API クライアント整備(Enabler)
    └── ...
└── MVP Epic — Use Case 1: メール受信〜分類
    ├── Feature Sub-Epic: F1 (Gmail 自動分類)
    │   ├── Story U1-01: メールが Gmail に着信したら Push 通知が Workers に到達する
    │   ├── Story U1-02: 受信メールが Claude Haiku で分類される
    │   ├── Story U1-03: [異常系] 分類失敗時にフォールバック
    │   └── ...
└── MVP Epic — Use Case 2: 案件情報抽出
    ├── Feature Sub-Epic: F2 (情報抽出)
    │   └── ...
└── MVP Epic — Use Case 3: カレンダー空き確認
└── MVP Epic — Use Case 4: エントリー下書き作成
└── MVP Epic — Use Case 5: LINE 通知
└── MVP Epic — Use Case 6: カレンダー自動管理
└── MVP Epic — Use Case 7: 辞退連絡(半自動)
└── MVP Epic — Use Case 8: 請求 CSV エクスポート
└── Phase 2 Epic(タイトル+概要のみ)
    ├── Story P2-01: マルチテナント認証
    ├── Story P2-02: Web ダッシュボード(実績登録 UI)
    └── ...
```

### 6.2 Story 構造(各 Story のフォーマット)
```markdown
### Story <ID>: <タイトル>
**As a** MC / コンパニオン (プライマリペルソナ),
**I want** [何をしたい],
**so that** [なぜ・どんな価値].

**Priority**: MUST / SHOULD / COULD
**Size**: S / M / L
**Type**: User Story / Enabler Story / Edge Case

**Acceptance Criteria** (Given-When-Then):
- **AC-1**:
  - **Given** [前提条件]
  - **When** [アクション]
  - **Then** [期待結果]
- **AC-2**: ...

**Traceability**:
- Implements: FR-X, NFR-Y(個別 NFR まで, 例: SECURITY-05)
- Covers: <要件サマリ>
```

### 6.3 Story 数の目安
- Foundation Epic: 5〜10 Story(Enabler 中心)
- MVP Use Case Epics 8 個 × 6〜10 Story = 50〜80 Story
- 異常系/エッジケース独立 Story(Q6=B): 各 Use Case Epic 内に 1〜3 Story
- **合計: 60〜100 Story**(細粒度=Q2 A、独立エッジケース=Q6 B の組み合わせ)

### 6.4 必須成果物との対応
- ✅ INVEST 原則準拠(各 Story が Independent / Negotiable / Valuable / Estimable / Small / Testable)
- ✅ 受入条件: Given-When-Then 形式
- ✅ ペルソナ: プライマリ MC のみ
- ✅ マッピング: Story → FR/NFR (個別 NFR まで)
- ✅ Phase: MVP フル + Phase 2 概略

---

## 7. フォローアップ確認質問(Cl-S シリーズ)

### Question Cl-S1: Story 構成方針の承認

セクション 6 の Decided Approach (Foundation Epic + MVP Use Case Epics + Phase 2 概略 / Story 数 60〜100) で進めて良いですか?

A) **OK、その方針で進める**
B) **Story 数が多すぎる → 中粒度 (Q2 を B 相当) に変更し、合計 30〜50 程度に絞る**
C) **Foundation Epic を含めない**(基盤系は Workflow Planning / Application Design で別管理)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question Cl-S2: Q2「細粒度」の意図確認

私の Q2 選択肢「**1 Story = 1 受入条件**」は不正確な表現でした(通常 1 Story には複数の受入条件が付きます)。
ご意図を再確認させてください:

A) **最小ユーザーアクション単位で 1 Story、各 Story には複数の受入条件**(Given-When-Then を 3〜5 個程度)
B) **本当に 1 Story = 1 受入条件**(極端に細かい、推奨しない)
C) **1 Story = 1 ユーザーアクション、受入条件は 1 〜 2 個に絞る**(中間)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 8. Final Plan(承認用サマリ)

すべての質問が解消したため、最終的な User Stories Generation Plan を以下に確定します。

### 8.1 ペルソナ
- **プライマリのみ**: MC / コンパニオン本人(Q1 = A)

### 8.2 Story 構成
- **Foundation Epic**(基盤・最初に開発): 5〜10 Story(Enabler Stories 中心)
  - F-01: ディレクトリ構成・開発ルール整備
  - F-02: Cloudflare 基盤セットアップ(Workers / D1 / KV / R2 / Queues / Cron / Durable Objects)
  - F-03: Google OAuth(Gmail / Calendar)認可フロー
  - F-04: LINE Messaging API Bot 連携セットアップ
  - F-05: Anthropic API クライアント整備
  - F-06: 構造化ロギング・監査ログ基盤(NFR-4 SECURITY-03 / NFR-8 対応)
  - F-07: プライバシーポリシー・同意管理基盤(NFR-5 対応)
- **MVP Use Case Epics**(F1〜F8 の Feature Sub-Epic、配下を業務フロー順): 50〜80 Story
  - Use Case 1: メール受信〜分類(F1)
  - Use Case 2: 案件情報抽出(F2)
  - Use Case 3: カレンダー空き確認(F3)
  - Use Case 4: エントリー下書き作成(F4)
  - Use Case 5: LINE 通知(F5)
  - Use Case 6: カレンダー自動管理(F6)
  - Use Case 7: 辞退連絡 半自動(F7)
  - Use Case 8: 請求 CSV エクスポート(F8)
- **Phase 2 Epic**(タイトル + 概要のみ、Q5 = B): 5〜10 Story
- **合計**: 60〜100 Story

### 8.3 Story 粒度・フォーマット
- **粒度**(Cl-S2 = A): **最小ユーザーアクション = 1 Story**、各 Story に **複数の受入条件**(3〜5 個)
- **フォーマット**:
  ```markdown
  ### Story <ID>: <タイトル>
  **As a** MC / コンパニオン (プライマリペルソナ)
  **I want** [何をしたい]
  **so that** [なぜ・どんな価値]

  **Priority**: MUST / SHOULD / COULD (MoSCoW)
  **Size**: S / M / L
  **Type**: User Story / Enabler Story / Edge Case

  **Acceptance Criteria** (Given-When-Then):
  - **AC-1**:
    - **Given** [前提条件]
    - **When** [アクション]
    - **Then** [期待結果]
  - **AC-2**: ...

  **Traceability**:
  - Implements: FR-X, NFR-Y(個別 NFR まで, 例: SECURITY-05)
  - Covers: <要件サマリ>
  ```

### 8.4 エッジケース(Q6 = B)
- 各 Use Case Epic 内で **独立した Story** として記述
- 例: 「U2-EC-01: 案件情報抽出が必須項目欠落で失敗した場合、ユーザーに LINE で警告通知される」

### 8.5 トレーサビリティ(Q8 = B)
- Story → FR/NFR の 1 対多マッピング
- 受入条件 → 個別 NFR(SECURITY-XX, etc.)もマップ
- `stories.md` 末尾にトレーサビリティ表を付与

### 8.6 必須成果物
- `aidlc-docs/inception/user-stories/personas.md`
- `aidlc-docs/inception/user-stories/stories.md`
- INVEST 原則準拠 / 受入条件付与 / マッピング / トレーサビリティ

---

## 9. 承認依頼

> **📋 REVIEW REQUIRED:**
> セクション 8 の Final Plan で User Stories の Part 2: Generation を実行して良いかご確認ください。

> **🚀 WHAT'S NEXT?**
>
> **You may:**
>
> 🔧 **Request Changes** — 修正依頼があればお知らせください
> ✅ **Approve & Continue** — 承認後、Part 2 で `personas.md` と `stories.md` を作成します
