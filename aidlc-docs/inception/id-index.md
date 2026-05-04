# ID Index — auto-mc-operation 横断 ID 一覧

本ドキュメントは、AIDLC ステージ全体で使用されるすべての **識別子(ID プレフィックス)** の正本配置・参照先・命名規則を一元化したインデックスです。

各ドキュメントで `A-3` / `F-09` / `U2-EC-04` 等の ID が登場した際、ここを参照することで **その ID がどこで定義され、どの文書から参照されているか** を辿れます。

---

## 1. ID プレフィックス一覧

| プレフィックス | 例 | 意味 | 正本(定義元) | 主な参照先 |
|--------------|-----|------|---------------|-----------|
| **FR-N** | FR-1〜FR-8 | 機能要件 | `inception/requirements/requirements.md` §5 機能要件 | stories.md トレーサビリティ表、application-design.md §8 完了基準、各設計成果物 |
| **NFR-N** | NFR-1〜NFR-8 | 非機能要件 | `inception/requirements/requirements.md` §6 非機能要件 | 全設計成果物 |
| **SECURITY-NN** | SECURITY-01〜15(全 15 ルール) | セキュリティ拡張ルール(`security-baseline` Extension) | `.aidlc-rule-details/extensions/security/baseline/security-baseline.md` | requirements.md NFR-4(全 15 ルールへ参照)、application-design.md §4 / §4.6 |
| **F-NN** | F-01〜F-14 | **Foundation Story**(基盤・最初に開発) | `inception/user-stories/stories.md` Foundation Epic セクション | application-design.md レビュー反映履歴、components.md `責務` 列の本文記述、各設計成果物 |
| **UN-NN** | U1-01, U2-EC-04 等 | **MVP Use Case Story** | `inception/user-stories/stories.md` MVP Use Case Epic セクション(Use Case 1〜7) | components.md A-N の `責務` 列で言及、services.md オーケストレーション節 |
| **P2-NN** | P2-01〜P2-12 | **Phase 2 Story 概略** | `inception/user-stories/stories.md` Phase 2 Epic セクション | requirements.md §4.2、application-design.md §17、data-model.md(§17 アカウント削除 / §18 保管方針) |
| **P1 / P2(ペルソナ)** | P1, P2 | **ユーザーペルソナ**(P1: AI 慣れ / P2: AI 未経験) | `inception/user-stories/personas.md` | stories.md 構成方針、application-design.md §1.0 |
| **D-NN** | D-1〜D-19 + D-18.5 | **ドメイン層コンポーネント** | `inception/application-design/components.md` §1 ドメイン層 | component-methods.md(シグネチャ)、component-dependency.md(依存マトリクス) |
| **A-N** | A-1〜A-11 | **アプリケーション層ユースケース** | `inception/application-design/components.md` §2 アプリケーション層 | services.md(オーケストレーション)、component-dependency.md、data-model.md |
| **I-NN** | I-1〜I-11 | **インフラストラクチャ層コンポーネント** | `inception/application-design/components.md` §3 インフラ層 | component-methods.md、services.md |
| **P-N** | P-1〜P-7 | **プレゼンテーション層ハンドラ** | `inception/application-design/components.md` §4 プレゼンテーション層 | application-design.md §4.4 API エンドポイント、component-dependency.md |
| **S-N** | S-1〜S-4 | **共有層コンポーネント** | `inception/application-design/components.md` §5 共有層 | application-design.md §1.0(S-2 MessageCatalog 等) |
| **Q-N / Cl-N / Cl-S-N** | Q1-Q22, Cl-1〜Cl-7, Cl-S1〜Cl-S2 | **質問・確認質問**(historical) | 各 `inception/plans/*-questions.md` / `*-clarification-questions.md` | requirements.md トレーサビリティ表 §11、stories.md レビュー反映、各 plan の Final Plan セクション |
| **C/W/S/N + 番号** | C1, W4, S9, N2 等 | **レビュー指摘 ID**(historical) | 各レビュー issue コメント(PR #3 内) | application-design.md §7.1 / §7.2 レビュー反映履歴 |

---

## 2. ID 命名規則

- **数字部の桁**: 2 桁ゼロ埋めを推奨(F-09, D-12, U1-02 など)。1 桁のままでも許容(A-1, P-7)。新規追加時は同レイヤ・同フォルダの既存先例に合わせる
- **EC**: Edge Case を表す中置語。`U1-EC-01`、`U2-EC-04` のように Use Case 番号と通し番号の間に挿入
- **N.5**: 既存番号と既存番号の間に挿入する場合に使う(例: D-18.5)。後続のリファクタで本来の体系に再付番してよい
- **形式**: 大文字プレフィックス + ハイフン + 番号 で統一(`A-1`、`F-09`、`U2-EC-01`)
- **重複禁止**: 同一プレフィックス内で同じ番号は使わない。番号の意味は永久不変

---

## 3. 主要マッピング表(クロスリファレンス)

### 3.1 User Story → Application UseCase の対応

| User Story | 対応する A-N(application-design `services.md`) |
|------------|----------------------------------------------|
| F-01〜F-14(Foundation Story) | 個別 A-N なし(基盤・Enabler 系)。実装は Construction の Foundation ユニット |
| U1-01 / U1-EC-03 / U1-02 / U1-EC-01 / U1-EC-02 | A-2 IngestMail / A-3 ClassifyMail / A-10 RotateGmailWatch |
| U2-00〜U2-EC-04 | A-4 ExtractCase(U2-00 はプロンプト設計の Enabler) |
| U3-01〜U3-EC-01 | A-5 CheckAvailability |
| U4-01〜U4-EC-02 | A-6 ComposeEntryDraft |
| U5-01〜U5-EC-01 | A-9 NotifyUser(LINE 通知 + Postback) |
| U6-01〜U6-EC-01 | A-7 ManageCalendar |
| U7-01〜U7-EC-01 | A-8 DetectAndDeclineConflicts |
| (セットアップ) | A-1 OnboardUser(F-03 OAuth + F-07 同意) |
| Phase 2 P2-01〜P2-08 | 個別 A-N は Phase 2 で再設計 |

### 3.2 FR → A-N → 主要テーブル のトレーサビリティ

| 機能要件 | 対応 A-N | 主要テーブル(`data-model.md`) |
|---------|---------|------------------------------|
| FR-1 メール分類 | A-2 / A-3 | `messages` / `classification_rules` |
| FR-2 案件抽出 | A-4 | `cases` / `schedules` / `offices` / `office_patterns` |
| FR-3 カレンダー空き確認 | A-5 | `cases` / `schedules`(`overlap_status`)/ Calendar API freeBusy |
| FR-4 エントリー下書き作成 | A-6 | `entries` / `pr_corpus` |
| FR-5 LINE 通知 | A-9 | (D1 への永続化なし、`audit_logs` のみ) |
| FR-6 カレンダー自動管理 | A-7 | `calendar_events` |
| FR-7 辞退連絡 半自動 | A-8 | `declines` / `decline_corpus` |
| ~~FR-8~~ 請求 CSV | (Phase 2 P2-08) | (Phase 2 で再設計) |

### 3.3 Foundation Story → 設計成果物の対応

| F-NN | 主に対応する設計箇所 |
|------|--------------------|
| F-01 ディレクトリ・DDD | `components.md` の 5 レイヤ構造、F-01 AC-1 で定義 |
| F-02 Cloudflare 基盤 | `application-design.md` §1.2 Container 図、`data-model.md` 0001〜0010 マイグレーション |
| F-03 Google OAuth | A-1 OnboardUser、I-11 OAuth2Service、D-18.5 OAuthExchanger、`oauth_tokens` テーブル |
| F-04 LINE Bot | A-9 NotifyUser、I-3 LineMessagingAdapter、P-1 LineWebhookHandler |
| F-05 Anthropic クライアント | I-4 AnthropicClient、D-18 LlmClient(ポート) |
| F-06 ロギング・監査 | S-1 Logger、`audit_logs` テーブル |
| F-07 プライバシー同意 | A-1 OnboardUser、`consents` テーブル(append-only) |
| F-08 メッセージング規約 | S-2 MessageCatalog |
| F-09 シークレット管理 | I-10 CryptoService、`oauth_tokens.encrypted_refresh_token` / `key_id` |
| F-10 ドメイン取得・DNS | `application-design.md` §4.1 ベース URL(`<base-domain>`) |
| F-11 抽象化レイヤ | D-15〜D-18.5 ポート群、I-1〜I-4 アダプタが実装 |
| F-12 システム構成図 | `application-design.md` §1.1 / §1.2 C4 図 |
| F-13 テスト戦略・E2E | S-4 TestSupport、`application-design.md` §6 |
| F-14 監視・アラート | F-06 と統合、`application-design.md` §7 完了基準・F-14 Story 内 |

### 3.4 ペルソナ → 設計上の配慮

| ペルソナ | 設計上の主な配慮箇所 |
|---------|--------------------|
| **P1**(AI 慣れ) | 標準フロー(Flex Message ボタン操作、効率重視) |
| **P2**(AI 未経験) | F-08 専門用語禁止、S-2 MessageCatalog による平易な日本語、エラー時の具体ガイド埋め込み(`application-design.md` §1.0) |

---

## 4. ID 体系の使い方ガイド

### ある ID を見たとき

1. プレフィックスから本ドキュメント §1 で「正本」列を確認
2. 正本のドキュメント・セクションを開く
3. その ID の詳細(責務 / メソッド / Implements 等)を読む

### 逆引き(機能要件から実装を辿る)

`requirements.md` の機能要件 FR-X
→ §3.2 の表で対応する A-N を確認
→ `components.md` §2 で A-N の責務を読む
→ `services.md` でオーケストレーションを確認
→ `component-dependency.md` で依存関係(どの D-NN / I-NN を使うか)を確認
→ `data-model.md` で永続化先テーブルを確認

### 設計変更時のチェック

ある ID(例: A-7)を変更する際、本ドキュメント §1 の「主な参照先」列で **影響範囲** を機械的に把握できる。

---

## 5. 注意事項

- 本ドキュメントは **正本ではなくインデックス**。各 ID の意味の正本は §1 「正本(定義元)」列のドキュメントを優先
- ID を新規追加・廃止する際は **本ドキュメントを併せて更新**(将来 CI で自動検証する候補)
- レビュー指摘 ID(C/W/S/N)は時系列でレビュー履歴に保管され、再レビュー後に解消した場合も履歴として残る(削除しない)

---

## 6. Phase 2 / MVP 範囲外項目の管理規約

設計議論やレビューで **「MVP で扱わない」「Phase 2 で〜」「将来拡張」** とした項目は、必ず以下の手順で **stories.md の Story として一元管理** する。設計文書のインライン記述に散逸させない。

### 6.1 必須手順(項目を Phase 2 行きと判断した時点で実施)

1. **既存 Story の検索**: `aidlc-docs/inception/user-stories/stories.md` Phase 2 Epic セクション(`P2-01〜`)に該当するテーマがあるか確認
2. **既存があれば追記**: 既存 `P2-NN` の概要に bullet を追加(タイトルが大幅に変わる場合のみ Story 名を更新)
3. **既存がなければ新規追加**: 末尾に新しい `P2-NN` 行を追加。形式: `| **P2-NN** | タイトル | 概要(MVP 移送理由 + 期待される機能) |`
4. **id-index.md §1 の P2-NN 範囲を同期**: `P2-01〜P2-NN` を最新値に更新
5. **設計文書の記述方法**: data-model.md / application-design.md / components.md / services.md 等から Phase 2 を言及する場合、**インライン詳述は禁止**。代わりに **`[Phase 2: P2-NN]`** の形式で Story ID リンクのみ記述

### 6.2 設計文書での記述パターン

| 用途 | 記述例 |
|------|--------|
| カラムの将来用途を示唆 | `| カラム名 | 型 | 制約 | 用途。\`[Phase 2: P2-08]\` で請求 CSV エクスポートに使用 \|` |
| セクションの拡張予告 | `### 17.6 拡張予定(Phase 2)\n\n本セクションの拡張は \`[Phase 2: P2-09]\` に集約:\n- ...\n- ...` |
| インデックス | data-model.md §19 のように **テーブル名 → P2-NN** マッピング表のみ |

### 6.3 禁止事項(アンチパターン)

- ❌ 設計文書に「Phase 2 で〜を追加する」「Phase 2 で再設計」とだけ書いて Story を作らない
- ❌ Phase 2 機能の詳細仕様を data-model.md / application-design.md にインラインで何段落も書く(stories.md に書く)
- ❌ 同じ Phase 2 アイデアを複数の設計文書に重複記述する(stories.md の P2-NN を単一の正本として参照させる)

### 6.4 AI による執筆時のチェックリスト

設計文書の編集セッションで「Phase 2 で〜」と書きたくなった際、提出前に必ず:

- [ ] 該当する `P2-NN` が stories.md にあるか確認した
- [ ] なければ stories.md に新規 Story を追加した
- [ ] あれば既存 Story の概要に項目を追記した
- [ ] 設計文書のインライン記述は `[Phase 2: P2-NN]` 形式のリンクのみにした
- [ ] id-index.md §1 の P2-NN 範囲が最新値か確認した

**自動検証(将来候補)**: 設計文書(`aidlc-docs/inception/application-design/**/*.md`)に "Phase 2" の文字列が含まれる場合、`[Phase 2: P2-NN]` 形式のリンクが伴っているかを CI でチェック。リンクが無い箇所はビルド失敗。
