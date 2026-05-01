# `/review` コマンド 使い方ガイド

## 概要

`/review` はPRの変更内容を**5つの視点**から自動レビューするClaude Codeスラッシュコマンドです。

AI支援開発（Claude Code等）で生成されたコードに特有のリスク（ハルシネーション・セキュリティ不備・ビジネス意図の誤解）を含め、多角的にレビューします。

---

## 設計の参考元

| 出典 | 採用した観点 |
|---|---|
| [Google Engineering Practices](https://google.github.io/eng-practices/review/) | 設計・複雑性・命名・テスト・一貫性の基準 |
| [mgreiler/code-review-checklist](https://github.com/mgreiler/code-review-checklist) | 12カテゴリの包括的チェックリスト構造 |
| [axolo-co/developer-resources](https://github.com/axolo-co/developer-resources/blob/main/code-review-guideline-template/code-review-guideline-template.md) | ガイドラインテンプレートのフレームワーク |
| [Vibe Coding Security - Checkmarx](https://checkmarx.com/blog/security-in-vibe-coding/) | AI生成コード特有の脆弱性パターン |
| [AI-Generated Code Vulnerabilities - Vidoc Security Lab](https://blog.vidocsecurity.com/blog/vibe-coding-security-vulnerabilities) | SQLi・XSS・認証不備など9種の具体的脆弱性 |
| [AI Code Review Best Practices - Graphite](https://graphite.com/guides/ai-code-review-implementation-best-practices) | AI開発フロー向けレビュー実装指針 |
| [Establishing Standards for AI-Generated Code - MetaCTO](https://www.metacto.com/blogs/establishing-code-review-standards-for-ai-generated-code) | AI生成コード専用のレビュー基準 |

---

## 使い方

### 基本的な使い方

PRブランチにチェックアウトした状態で、Claude Codeのチャットに入力します:

```
/review
```

Claude が `git diff main...HEAD` で変更差分を取得し、5視点のレビューを自動実行します。

### 特定の視点だけ依頼する場合

```
/review セキュリティ視点だけレビューしてください
```

```
/review 開発者視点とQA視点でレビューしてください
```

### 特定ファイルを指定する場合

```
/review src/gmail/client.py のレビューをしてください
```

---

## 5つのレビュー視点

### 👔 プロダクトオーナー視点

ビジネス価値と要件の整合性を確認します。

**主な確認内容**:
- 実装が要件・Issueの意図を正確に実現しているか
- スコープクリープ（要求外の変更）がないか
- ユーザーへの影響が明確か
- AIがビジネス意図を誤解した実装をしていないか

---

### 🧪 品質保証 (QA) 視点

テスト品質とリグレッションリスクを確認します。

**主な確認内容**:
- ユニットテスト・統合テストが追加・更新されているか
- エッジケース（null・境界値・タイムアウト等）がカバーされているか
- **AIハルシネーション検出**: 「それらしいが間違った」ロジックがないか
- 既存テストへの影響

---

### 💻 開発者視点

コード品質・設計・保守性を確認します。[Google Engineering Practices](https://google.github.io/eng-practices/review/reviewer/looking-for.html) の10カテゴリ（Design / Functionality / Complexity / Tests / Naming / Comments / Style / Consistency / Documentation / Context）に準拠します。

**主な確認内容**:
- 設計の適切さ（過剰な抽象化・AIによる不要コードの排除）
- 単一責任・DRY原則
- 命名・コメントの質
- エラーハンドリング・パフォーマンス
- コミットメッセージ規約の遵守

---

### 🔒 セキュリティ視点

AI生成コード特有のリスクを含むセキュリティ脆弱性を確認します。

> **背景**: Veracode 2025レポートによると、AI生成コードは人間のコードより **2.74倍** 脆弱性が多く、検証した200以上のAI開発アプリの **91.5%** にハルシネーション由来の脆弱性が存在しました。

**主な確認内容**:
- シークレット・APIキーのハードコード
- 認証・認可ロジックの正確性（Broken Object-Level Authorization）
- インジェクション（SQL・OS・テンプレート）
- **Gmail等OAuth連携のスコープが最小権限か**
- **依存パッケージのハルシネーション**（存在しないパッケージを参照していないか）

---

### 📦 依存・保守視点

依存関係の健全性と長期的な保守性を確認します。

**主な確認内容**:
- 新規パッケージ追加の正当性
- ロックファイルの更新状況
- 既知の脆弱性（npm audit / pip audit等）
- 環境変数追加等の設定変更のドキュメント反映

---

## 重要度レベルの読み方

| アイコン | レベル | 対応方針 |
|---|---|---|
| 🔴 | **Critical** | マージ前に必ず修正。Claudeは `Request Changes` 判定を出します |
| 🟡 | **Warning** | 修正を強く推奨。理由があれば許容されます |
| 🟢 | **Suggestion** | 改善提案。対応は任意です |
| 💡 | **Note** | 情報共有・補足。対応不要です |

---

## 出力例

```
### 👔 プロダクトオーナー視点 レビュー

✅ 問題なし

---

### 🧪 品質保証 (QA) 視点 レビュー

#### 🟡 エッジケースのテストが不足しています
**ファイル**: `src/gmail/client.py` (L45)
**指摘**: Gmail APIがレート制限エラーを返した場合のテストがありません
**提案**: `RateLimitError` を mock して retry 処理が動作することを確認するテストを追加してください

---

### 🔒 セキュリティ視点 レビュー

#### 🔴 APIキーがハードコードされています
**ファイル**: `config/settings.py` (L12)
**指摘**: `GMAIL_API_KEY = "AIza..."` が直接記述されています
**提案**: 環境変数または Secret Manager から読み込む実装に変更してください

---

## 総合評価

**判定**: 🔄 Request Changes

**サマリー**:
- 🔴 Critical: 1件
- 🟡 Warning: 1件
- 🟢 Suggestion: 0件

**優先対応項目**:
- `config/settings.py` のAPIキーをシークレット管理に移行してください
```

---

## ファイル構成

```
.claude/
└── commands/
    └── review.md   # /review コマンドの定義（Claudeへの指示）

docs/
└── review-guide.md  # このドキュメント
```

---

## カスタマイズ

`.claude/commands/review.md` を編集することでレビュー観点を調整できます。

**例: プロジェクト固有の観点を追加する場合**

```markdown
## 視点6: 🗓️ MC業務固有視点

**確認項目**:
- [ ] Gmail送信処理に誤送信防止の確認ステップがあるか
- [ ] 司会スクリプトの文言に問題がないか
```
