# AI-DLC State Tracking

## Project Information
- **Project Name**: MC・コンパニオン エントリー業務自動化システム (auto-mc-operation)
- **Project Type**: Greenfield
- **Start Date**: 2026-05-01T00:00:00Z
- **Current Stage**: INCEPTION - Application Design(プラン作成完了 = ユーザー回答待ち)

## Workspace State
- **Existing Code**: No
- **Programming Languages**: Rust(確定 — Cl-3-followup B)
- **Build System**: Cargo(確定)
- **Project Structure**: Empty
- **Reverse Engineering Needed**: No (Greenfield)
- **Workspace Root**: /home/user/auto-mc-operation

## Code Location Rules
- **Application Code**: ワークスペースルート (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ 配下のみ
- **Structure patterns**: code-generation.md Critical Rules を参照

## Extension Configuration

| Extension | Enabled | Decided At | Notes |
|-----------|---------|------------|-------|
| security-baseline | **Yes** | Requirements Analysis (Q21) | 全 SECURITY ルールを Blocking 制約として強制 |
| property-based-testing | **Yes** | Requirements Analysis (Q22) | 全 PBT ルールを Blocking 制約として強制 |

## Confirmed Tech Stack
- **言語**: Rust
- **MVP プラットフォーム**: Cloudflare Workers (workers-rs WASM)
- **拡張時プラットフォーム**: GCP Cloud Run (axum + tokio)
- **DB**: D1 (SQLite) + KV + R2 → Cloud SQL (PostgreSQL)
- **HTTP クライアント**: reqwest(Anthropic / Google / LINE はすべて REST 直叩き)
- **OAuth2**: yup-oauth2(Google API のみ)
- **LLM**: Claude Haiku
- **テスト**: proptest(プロパティベース)+ 標準テスト

## Stage Progress

### INCEPTION PHASE
- [x] Workspace Detection (ALWAYS)
- [ ] Reverse Engineering (SKIP — Greenfield)
- [x] Requirements Analysis (ALWAYS) — 2026-05-01 ユーザー承認取得
- [x] User Stories (CONDITIONAL) — 2026-05-02 ユーザー承認取得(58 Story)
- [x] Workflow Planning (ALWAYS) — 2026-05-02 ユーザー承認取得
- [ ] Application Design (CONDITIONAL) — **EXECUTE** プラン作成完了、ユーザー回答待ち
- [ ] Units Generation (CONDITIONAL) — **EXECUTE**(7 ユニットに分解予定)

### CONSTRUCTION PHASE
- [ ] Per-Unit Loop
- [ ] Build and Test (ALWAYS)

### OPERATIONS PHASE
- [ ] Operations (PLACEHOLDER)
