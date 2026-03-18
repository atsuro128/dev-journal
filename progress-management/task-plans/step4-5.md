# Step 4+5 基本設計 + 詳細設計 タスク実行計画

作成: design-architect / 作成日: 2026-03-18 / 更新日: 2026-03-18

## 概要

基本設計（画面・UI フロー）と詳細設計（API・DB・認可・セキュリティ・監視）を機能単位の縦割りで進める。画面設計と API 設計は密結合（画面の入出力 = API の入出力）のため、1 つの機能について画面から API まで一貫して設計する。

---

## 上流成果物の分析結果

### 整合性の確認

上流成果物（Step 0〜3）を横断的に検証した結果、以下を確認した。

| 検証項目 | 結果 |
|---------|------|
| domain_model.md のエンティティと requirements.md の機能要件の対応 | 整合。全機能要件のデータ構造がエンティティで表現されている |
| state_machine.md と workflow.md の遷移定義 | 整合。T1〜T5, X1〜X10 が一致 |
| rbac.md の権限マトリクスと requirements.md のアクター定義 | 整合。全機能のアクター・権限が一致 |
| architecture.md 5.1 の API 一覧と requirements.md の機能 | 整合。全機能要件に対応するエンドポイントが存在 |
| architecture.md のレイヤー構成と domain_model.md の責務マッピング | 整合。ハンドラ/サービス/ドメイン/リポジトリの責務が一致 |

### 検出されたリスク

| # | リスク | 影響度 | 対処方針 |
|---|--------|--------|---------|
| R1 | ダッシュボード API（GET /api/dashboard）の画面仕様がユースケース（UC-SYS04）にしか記載されておらず、requirements.md 2.7 の DASH-001〜004 のロール別表示の詳細化が必要 | 低 | Wave 2 の画面一覧で詳細化する（ダッシュボードは経費 CRUD に含めて設計） |
| R2 | 再申請 API（UC-M09）に対応する明示的なエンドポイントが architecture.md 5.1 にない | 低 | 再申請は「レポート作成 + reference_report_id の設定」で実現。API 設計時に明記する |
| R3 | パスワードリセットのメール送信は MVP スコープだが、メール送信基盤の設計が未定義 | 低 | security.md でパスワードリセットのトークン生成・検証フローを定義。メール送信の実装方式は実装フェーズで決定（環境変数でログ出力 or SES） |

---

## work-breakdown からの変更点

| # | 変更内容 | 理由 |
|---|---------|------|
| C1 | `screens.md` を画面一覧（全体俯瞰）と機能別詳細ファイルに分離 | 抽象度が異なる成果物の混在を防ぐ。画面一覧は全画面の俯瞰、機能別ファイルは入力項目・バリデーション等の詳細仕様。詳細は後述の「成果物ファイルの分割方針」参照 |
| C2 | openapi.yaml は最初から 1 ファイルに追記する方式を採用 | 機能別に分割してマージする方式はコンフリクト・重複定義のリスクが高い。セクション（tags）で機能を分離し、直列的に追記することで統合コストをゼロにする。詳細は後述の「共有ファイル調整」参照 |
| C3 | Wave 4（4+5-G）の作業内容を具体化 | 「最終統合」が曖昧だったため、作業内容・検証項目・成果物を明確に定義。詳細は受け入れ基準セクション参照 |
| C4 | Wave 2 にダッシュボード画面仕様を追加（4+5-D-1 のスコープに含める） | ダッシュボードは経費レポートの状態集計であり、経費 CRUD の画面設計と密接に関連する。独立タスクにするほどの規模ではないため、経費画面詳細化に含める |
| C5 | Wave 1 の detail-designer タスクを直列化（セキュリティ → 監視の順） | 同一エージェント（detail-designer）に 2 タスクを並列で割り当てると、実際にはどちらかを先にやることになる。監視設計はセキュリティ設計を参照する場面があるため（ログのセキュリティ要件等）、直列化を明示する。ただし db-designer / basic-designer とは引き続き並列 |

---

## 成果物ファイルの分割方針

### screens.md の分離

**採用（提案を一部修正して採用）**

| ファイル | パス | 作成 Wave | 内容 |
|---------|------|----------|------|
| 画面一覧 | `40_basic_design/screens.md` | Wave 1 (4+5-I) | 全画面の一覧（画面ID・画面名・目的・対応ロール・遷移先）、共通 UI パターン |
| 認証画面詳細 | `40_basic_design/screens/auth.md` | Wave 2 (4+5-C-1) | ログイン・サインアップ・パスワードリセットの入力項目・バリデーション・エラー表示 |
| 経費画面詳細 | `40_basic_design/screens/expenses.md` | Wave 2 (4+5-D-1) | レポート一覧・作成・編集・詳細・明細追加/編集、ダッシュボードの画面仕様 |
| 添付画面詳細 | `40_basic_design/screens/attachments.md` | Wave 3 (4+5-E-1) | アップロード・プレビュー・削除の画面仕様 |
| 承認画面詳細 | `40_basic_design/screens/approvals.md` | Wave 3 (4+5-F-1) | 承認キュー・承認/却下アクション・支払待ち/支払完了の画面仕様 |

**修正点**: 提案案ではダッシュボードの位置づけが不明確だったため、`expenses.md` に含める。ダッシュボードは経費レポートの状態集計であり、経費画面の一部として扱う。

### openapi.yaml の扱い

**1 ファイル追記方式を採用**

- Wave 2 の 4+5-C-2 で `openapi.yaml` を新規作成（共通定義 + 認証エンドポイント）
- 以降のタスク（D-2, E-2, F-2）が順次 tags セクションを追記
- 機能間でエンドポイントの出力先が重複しないため、セクション単位での追記で衝突は発生しない
- 共通スキーマ（Error, Pagination 等）は 4+5-C-2 で初回定義。後続タスクは共通スキーマを参照し、不足があれば追加

**追記順の制約**: openapi.yaml に複数タスクが書き込むため、同一 Wave 内では直列実行を保証する（Wave 2: C-2 → D-2、Wave 3: E-2 → F-2）。これは work-breakdown の機能内直列ルール（画面 → API）に加え、API 定義間でも直列を保証するもの。

---

## タスク一覧

| Wave | ID | タスク | エージェント | 状態 | セッション | 成果物パス |
|------|----|--------|------------|------|-----------|-----------|
| 1 | 4+5-A | DB スキーマ設計（全体） | db-designer | 未着手 | - | `50_detail_design/db_schema.md` |
| 1 | 4+5-B | セキュリティ設計（横断） | detail-designer | 未着手 | - | `50_detail_design/security.md` |
| 1 | 4+5-H | 監視・ログ詳細設計（横断） | detail-designer | 未着手 | - | `50_detail_design/monitoring.md` |
| 1 | 4+5-I | 画面一覧・遷移の全体設計 | basic-designer | 未着手 | - | `40_basic_design/screens.md`, `40_basic_design/ui_flow.md` |
| 2 | 4+5-C-1 | 認証・ユーザー管理（画面詳細化） | basic-designer | 未着手 | - | `40_basic_design/screens/auth.md` |
| 2 | 4+5-C-2 | 認証・ユーザー管理（API 定義） | detail-designer | 未着手 | - | `50_detail_design/openapi.yaml` |
| 2 | 4+5-D-1 | 経費レポート CRUD + ダッシュボード（画面詳細化） | basic-designer | 未着手 | - | `40_basic_design/screens/expenses.md` |
| 2 | 4+5-D-2 | 経費レポート CRUD + ダッシュボード（API 定義） | detail-designer | 未着手 | - | `50_detail_design/openapi.yaml` |
| 3 | 4+5-E-1 | 添付ファイル（画面詳細化） | basic-designer | 未着手 | - | `40_basic_design/screens/attachments.md` |
| 3 | 4+5-E-2 | 添付ファイル（API + S3 設計） | detail-designer | 未着手 | - | `50_detail_design/openapi.yaml`, `50_detail_design/files.md` |
| 3 | 4+5-F-1 | 承認フロー（画面詳細化） | basic-designer | 未着手 | - | `40_basic_design/screens/approvals.md` |
| 3 | 4+5-F-2 | 承認フロー（API 定義） | detail-designer | 未着手 | - | `50_detail_design/openapi.yaml` |
| 4 | 4+5-G | 認可設計 + 最終統合 | design-architect | 未着手 | - | `50_detail_design/authz.md`, `40_basic_design/ui_flow.md` |

### 状態値

- `未着手`: まだ開始していない
- `実行中`: エージェントが作業中（セッション列にターミナル識別子を記入）
- `レビュー中`: 成果物完成、reviewer が検証中
- `完了`: レビュー通過、コミット済み
- `ブロック`: 依存タスクの未完了や blocker issue により着手不可

---

## 依存グラフ

```
Phase 0: design-architect → task-plans/step4-5.md [4+5-0] ← 完了（本ファイル）
    ↓ ユーザー合意
Wave 1（並列 — 出力先独立）:
    ┌─ db-designer ───────→ db_schema.md           [4+5-A]
    ├─ detail-designer ───→ security.md            [4+5-B]
    │    ↓（完了後）
    │  detail-designer ───→ monitoring.md           [4+5-H]
    └─ basic-designer ────→ screens.md + ui_flow.md [4+5-I]
    ↓ Wave 1 レビュー
Wave 2（機能間並列・機能内直列）:
    ┌─ [認証] basic-designer → screens/auth.md      [4+5-C-1]
    │           ↓（完了後）
    │         detail-designer → openapi.yaml §認証   [4+5-C-2]
    │
    └─ [経費] basic-designer → screens/expenses.md   [4+5-D-1]
               ↓（完了後）
             detail-designer → openapi.yaml §経費    [4+5-D-2]
    ※ openapi.yaml 追記: C-2 が先、D-2 が後（初回作成の制約）
    ↓ Wave 2 レビュー
Wave 3（機能間並列・機能内直列）:
    ┌─ [添付] basic-designer → screens/attachments.md [4+5-E-1]
    │           ↓（完了後）
    │         detail-designer → openapi.yaml §添付 + files.md [4+5-E-2]
    │
    └─ [承認] basic-designer → screens/approvals.md   [4+5-F-1]
               ↓（完了後）
             detail-designer → openapi.yaml §承認     [4+5-F-2]
    ※ openapi.yaml 追記: E-2 と F-2 はどちらが先でも可（tags が独立）
    ↓ Wave 3 レビュー
Wave 4（直列）:
    design-architect → authz.md + ui_flow.md（最終版） [4+5-G]
    ↓ Wave 4 レビュー（横断レビュー）
    ↓ /codex-review
```

### クリティカルパス

```
4+5-I → 4+5-C-1 → 4+5-C-2 → 4+5-D-2 → (Wave 2 レビュー)
  → 4+5-E-1 → 4+5-E-2 → (Wave 3 レビュー) → 4+5-G
```

openapi.yaml の初回作成が C-2 に依存するため、C-2 の完了が D-2 の着手条件となる。これがクリティカルパスの律速要因。

---

## 実行順序

### Wave 1: 基盤設計

**実行パターン**: db-designer と basic-designer は並列。detail-designer は B → H の直列。全体としてはパターン A（並列）+ 部分直列。

```
┌─ db-designer ────────→ db_schema.md           [4+5-A]
│
├─ detail-designer ────→ security.md            [4+5-B]
│    ↓（完了後）
│  detail-designer ────→ monitoring.md           [4+5-H]
│
└─ basic-designer ─────→ screens.md + ui_flow.md [4+5-I]
```

直列にする理由（B → H）: 同一エージェントであり、かつ監視設計はセキュリティ設計のログ・メトリクス要件を参照する。

### Wave 1 完了後のレビュー

```
design-unit-reviewer x 各成果物（並列）: 基盤成果物の上流整合性スポットチェック
  - db_schema.md: domain_model.md との対応、RLS ポリシー、インデックス方針
  - security.md: requirements.md 4.2 との対応、architecture.md 6.x との整合
  - monitoring.md: ADR-0005 との対応、requirements.md 4.1/4.3 との整合
  - screens.md + ui_flow.md: usecases.md の全 UC カバー、rbac.md との整合
```

---

### Wave 2: 機能設計 1 -- 認証・経費CRUD

**実行パターン**: 機能単位並列、機能内は basic-designer → detail-designer の直列。ただし openapi.yaml は C-2 → D-2 の順で追記。

```
┌─ [認証] basic-designer → screens/auth.md       [4+5-C-1]
│           ↓（完了後）
│         detail-designer → openapi.yaml §認証    [4+5-C-2]
│
└─ [経費] basic-designer → screens/expenses.md    [4+5-D-1]
            ↓（完了後かつ C-2 完了後）
          detail-designer → openapi.yaml §経費     [4+5-D-2]
```

直列にする理由:
- 画面 → API: API 設計は画面仕様（入力項目・バリデーション・表示条件）を参照するため
- C-2 → D-2: openapi.yaml の初回作成（共通定義 + 認証エンドポイント）が C-2 で行われるため、D-2 はそれを前提とする

**実行の最適化**: C-1 と D-1 は並列起動できる（出力ファイルが独立）。C-1 完了後すぐに C-2 を起動し、D-1 完了 + C-2 完了後に D-2 を起動する。

### Wave 2 完了後のレビュー

```
design-unit-reviewer x 2（並列）:
  - 認証機能: screens/auth.md x openapi.yaml §認証 x db_schema.md x rbac.md §3.1
  - 経費CRUD機能: screens/expenses.md x openapi.yaml §経費 x db_schema.md x rbac.md §3.2
```

---

### Wave 3: 機能設計 2 -- 添付・承認

**実行パターン**: 機能単位並列、機能内は basic-designer → detail-designer の直列。

```
┌─ [添付] basic-designer → screens/attachments.md  [4+5-E-1]
│           ↓（完了後）
│         detail-designer → openapi.yaml §添付 + files.md [4+5-E-2]
│
└─ [承認] basic-designer → screens/approvals.md     [4+5-F-1]
            ↓（完了後）
          detail-designer → openapi.yaml §承認      [4+5-F-2]
```

直列にする理由: Wave 2 と同様。API 設計は画面仕様を参照するため。

**openapi.yaml の追記順**: E-2 と F-2 はそれぞれ独立した tags セクションに書き込むため、どちらが先でもよい。ただし同時書き込みは避け、いずれか先に完了した方から追記する。

### Wave 3 完了後のレビュー

```
design-unit-reviewer x 2（並列）:
  - 添付ファイル機能: screens/attachments.md x openapi.yaml §添付 x files.md x db_schema.md x rbac.md §3.4
  - 承認フロー機能: screens/approvals.md x openapi.yaml §承認 x db_schema.md x state_machine.md x rbac.md §3.3
```

---

### Wave 4: 認可設計 + 最終統合

**実行パターン**: 直列（design-architect → design-cross-reviewer）

```
design-architect → authz.md + ui_flow.md（最終版）  [4+5-G]
  ↓（完了後）
design-cross-reviewer: 全機能横断レビュー
```

### Wave 4 完了後のレビュー

```
design-cross-reviewer: 全機能横断レビュー
  - 機能間整合性（画面ID → APIパス → DBテーブル → 認可ルール の一貫性）
  - 用語一貫性（glossary.md 準拠）
  - RBAC 完全性（全エンドポイント x 全ロールの網羅）
  - 上流トレーサビリティ（requirements.md の全機能 → 設計成果物の対応）
→ /codex-review（Step 成果物完成後の外部レビュー）
```

---

## 共有ファイル調整

| ファイル | 初回作成（Wave / タスクID） | 追記（Wave / タスクID） | 調整方法 |
|---------|--------------------------|----------------------|---------|
| `openapi.yaml` | Wave 2 / 4+5-C-2 | Wave 2 / 4+5-D-2、Wave 3 / 4+5-E-2, 4+5-F-2 | 直列追記。C-2 で共通定義（info, servers, securitySchemes, 共通 schemas, Error/Pagination）+ 認証エンドポイントを作成。以降は tags セクションごとに追記。同一 Wave 内で openapi.yaml への書き込みが重ならないよう、指揮役が制御する |
| `ui_flow.md` | Wave 1 / 4+5-I（初版） | Wave 4 / 4+5-G（最終版） | Wave 1 で全体構造を作成。Wave 2-3 では更新しない。Wave 4 で機能別画面詳細化の結果を反映して最終版に更新 |

---

## 受け入れ基準

### 4+5-A: DB スキーマ設計（全体）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| ドメインモデル | `20_domain/domain_model.md` | 3〜3.6（全エンティティ）, 4（値オブジェクト） |
| 状態遷移 | `20_domain/state_machine.md` | 2.1（状態一覧 — DB値）, 7.3（楽観的ロック） |
| ADR 0002 | `30_arch/adr/0002-multi-tenant.md` | Shared DB + tenant_id 方式 |
| ADR 0003 | `30_arch/adr/0003-rls-tenant-isolation.md` | RLS ポリシー設計 |
| アーキテクチャ | `30_arch/architecture.md` | 3.1（ディレクトリ構成 — db/ セクション） |
| 要件定義 | `10_requirements/requirements.md` | 4.4（データ要件: 論理削除, タイムスタンプ, 通貨） |

**出力**:

| 成果物 | パス |
|--------|------|
| DB スキーマ設計 | `deliverables/docs/50_detail_design/db_schema.md` |

**完了条件**:

- [ ] 全主要テーブル（tenants, users, tenant_memberships, expense_reports, expense_items, attachments）の DDL（概要レベル）が定義されている
- [ ] RLS ポリシーの適用対象テーブルと具体的なポリシー定義が記載されている
- [ ] インデックス方針（tenant_id 複合インデックス、ステータス検索用等）が記載されている
- [ ] マイグレーション方針（golang-migrate）が記載されている
- [ ] audit_logs テーブル（Phase 3 事前設計、INSERT ONLY 制約）が記載されている
- [ ] 論理削除（deleted_at）、タイムスタンプ（created_at, updated_at）の方針が全テーブルで統一されている
- [ ] domain_model.md の全エンティティ・全属性がテーブル定義に対応している

**作業上の注意点**:

- tenant_id の冗長保持（expense_items, attachments）は domain_model.md で設計判断済み。そのまま反映する
- User テーブルには tenant_id を持たない（TenantMembership 経由）。domain_model.md の設計判断に従う
- 楽観的ロック用の updated_at の利用方法を明記する（state_machine.md 7.3）
- Phase 3 エンティティ（notifications, invitations）は概要レベルで触れる程度でよい。audit_logs のみ INSERT ONLY 制約を詳細に定義する

---

### 4+5-B: セキュリティ設計（横断）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 4.2（セキュリティ要件）, SEC-001〜014 |
| アーキテクチャ | `30_arch/architecture.md` | 3.2（ミドルウェアチェーン）, 3.3（認証フロー）, 6（セキュリティアーキテクチャ） |
| ADR 0001 | `30_arch/adr/0001-tech-stack.md` | JWT ライブラリ等の選定 |

**出力**:

| 成果物 | パス |
|--------|------|
| セキュリティ設計 | `deliverables/docs/50_detail_design/security.md` |

**完了条件**:

- [ ] レート制限の具体的な数値と実装方式（ライブラリ選定含む）が決まっている
- [ ] CORS 方針（許可オリジン・メソッド・ヘッダー・credentials）が定義されている
- [ ] セキュリティヘッダー一覧（HSTS, X-Content-Type-Options, X-Frame-Options 等）が定義されている
- [ ] JWT 検証フロー（RS256、鍵管理、トークンローテーション方針）が定義されている
- [ ] パスワードリセットのトークン生成・検証・有効期限のフローが定義されている
- [ ] govulncheck / npm audit の CI 組み込み方針が記載されている

**作業上の注意点**:

- architecture.md 6 に大枠が定義済み。詳細設計では「具体的な実装方式」まで落とす
- リフレッシュトークンの保存方式（DB テーブル設計は 4+5-A で対応。ここではフローを定義）
- パスワードリセットメールの送信方式は「MVP ではログ出力 or 環境変数で切替」程度の記載でよい

---

### 4+5-H: 監視・ログ詳細設計（横断）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| ADR 0005 | `30_arch/adr/0005-monitoring-logging.md` | 監視・ログ方針全体 |
| アーキテクチャ | `30_arch/architecture.md` | 7（監視・ログ方針） |
| 要件定義 | `10_requirements/requirements.md` | 4.1（パフォーマンス要件）, 4.3（可用性要件） |

**出力**:

| 成果物 | パス |
|--------|------|
| 監視・ログ設計 | `deliverables/docs/50_detail_design/monitoring.md` |

**完了条件**:

- [ ] ログフォーマット定義（構造化ログのフィールド: timestamp, level, request_id, tenant_id, user_id, message 等）が定義されている
- [ ] ログレベル運用基準（ERROR / WARN / INFO / DEBUG の使い分け）が定義されている
- [ ] MVP で計測するメトリクス一覧とアラート閾値（p95 レスポンスタイム > 500ms、エラーレート > 1% 等）が定義されている
- [ ] ヘルスチェックエンドポイント仕様（`/health` のレスポンス形式、チェック項目）が定義されている
- [ ] CloudWatch Logs / Metrics / Alarms の具体的な設定方針が記載されている

**作業上の注意点**:

- ADR-0005 で大枠が決定済み。詳細設計では「具体的なフィールド定義」「閾値」「アラート条件」まで落とす
- security.md（4+5-B）のセキュリティ関連ログ要件を参照するため、4+5-B 完了後に着手する

---

### 4+5-I: 画面一覧・遷移の全体設計

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| ユースケース | `10_requirements/usecases.md` | 2〜6（全ユースケース） |
| 要件定義 | `10_requirements/requirements.md` | 2（全機能要件）, 4.5（ユーザビリティ） |
| RBAC | `10_requirements/rbac.md` | 3（権限マトリクス）, 2（ロール定義） |
| ワークフロー | `10_requirements/workflow.md` | 9（業務フロー全体像） |
| アーキテクチャ | `30_arch/architecture.md` | 4.3（ページとロールの対応）, 5.1（API 一覧） |

**出力**:

| 成果物 | パス |
|--------|------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` |
| 画面遷移図（初版） | `deliverables/docs/40_basic_design/ui_flow.md` |

**完了条件**:

- [ ] 全画面の一覧（画面ID・画面名・目的・対応ロール）が定義されている
- [ ] 画面遷移図（Mermaid）が全画面を網羅している
- [ ] ロール別の遷移パスが明示されている
- [ ] 共通 UI パターン（ヘッダー、ナビゲーション、エラー表示、ローディング、確認ダイアログ）が定義されている
- [ ] architecture.md 4.3 のページ一覧と整合している
- [ ] 後続の機能タスク（C〜F）が画面一覧を参照して詳細化できる状態になっている（画面IDが付与されている）

**作業上の注意点**:

- ここでは「全体の俯瞰」を作る。各画面の入力項目やバリデーションの詳細は Wave 2-3 の機能タスクで定義
- architecture.md 4.3 にページ一覧の叩き台がある。これをベースに、ユースケースから漏れている画面がないか確認
- 画面IDの命名規則を定義し、後続タスクで参照可能にする

---

### 4+5-C-1: 認証・ユーザー管理（画面詳細化）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.1（認証・ユーザー管理）, バリデーション表 |
| RBAC | `10_requirements/rbac.md` | 3.1（認証関連権限） |
| ユースケース | `10_requirements/usecases.md` | UC-SYS01〜03, UC-SYS05 |
| 画面一覧 | `40_basic_design/screens.md` | 認証関連画面（4+5-I で作成） |

**出力**:

| 成果物 | パス |
|--------|------|
| 認証画面詳細 | `deliverables/docs/40_basic_design/screens/auth.md` |

**完了条件**:

- [ ] ログイン画面の仕様（入力項目・バリデーション・エラー表示・遷移先）が定義されている
- [ ] サインアップ画面の仕様（入力項目・バリデーション・エラー表示）が定義されている
- [ ] パスワードリセット画面（要求・実行の2画面）の仕様が定義されている
- [ ] 画面一覧（screens.md）で定義した画面IDと対応している

**作業上の注意点**:

- SEC-011（認証失敗時のユーザー存在漏洩防止）をエラー表示に反映する
- パスワードの入力制約（8文字以上）をバリデーションに含める

---

### 4+5-C-2: 認証・ユーザー管理（API 定義）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.1（認証機能一覧・入出力） |
| RBAC | `10_requirements/rbac.md` | 3.1（認証関連権限マトリクス） |
| アーキテクチャ | `30_arch/architecture.md` | 5.1（認証 API 一覧）, 5.2（共通レスポンス形式）, 5.3（ページネーション） |
| DB スキーマ | `50_detail_design/db_schema.md` | users, tenant_memberships テーブル（4+5-A） |
| 認証画面詳細 | `40_basic_design/screens/auth.md` | 全体（4+5-C-1） |

**出力**:

| 成果物 | パス |
|--------|------|
| OpenAPI 定義 | `deliverables/docs/50_detail_design/openapi.yaml` |

**完了条件**:

- [ ] openapi.yaml の共通部分（info, servers, securitySchemes, 共通 schemas）が定義されている
- [ ] 認証エンドポイント（signup, login, refresh, logout, me, password-reset/*）が定義されている
- [ ] 各エンドポイントのリクエスト/レスポンス/エラーが定義されている
- [ ] rbac.md 3.1 および architecture.md 5.1 の認証エンドポイントとの差分が記録されている（過不足をコメントまたは別セクションに記載）

**作業上の注意点**:

- **openapi.yaml の初回作成タスク**。共通定義（Error schema, Pagination schema, securitySchemes）を含めること
- architecture.md 5.2 の共通レスポンス形式に準拠する
- 後続タスク（D-2, E-2, F-2）が追記する前提で、構造を拡張しやすく設計する

---

### 4+5-D-1: 経費レポート CRUD + ダッシュボード（画面詳細化）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.4（経費レポート CRUD）, 2.7（ダッシュボード）, カテゴリ一覧 |
| ドメインモデル | `20_domain/domain_model.md` | 3.4（ExpenseReport）, 3.5（ExpenseItem）, 4.3（Category） |
| ユースケース | `10_requirements/usecases.md` | UC-M01〜M09, UC-SYS04, UC-AD01, UC-AD02, UC-AC03 |
| RBAC | `10_requirements/rbac.md` | 3.2（経費レポート権限） |
| 画面一覧 | `40_basic_design/screens.md` | 経費関連画面（4+5-I で作成） |

**出力**:

| 成果物 | パス |
|--------|------|
| 経費画面詳細 | `deliverables/docs/40_basic_design/screens/expenses.md` |

**完了条件**:

- [ ] レポート一覧画面の仕様（フィルタ・ソート・ページネーション・ロール別表示差異）が定義されている
- [ ] レポート作成/編集画面の仕様（入力項目・バリデーション）が定義されている
- [ ] レポート詳細画面の仕様（明細一覧・状態表示・アクションボタンの出しわけ）が定義されている
- [ ] 明細追加/編集の仕様（入力項目・カテゴリ選択・バリデーション）が定義されている
- [ ] ダッシュボード画面の仕様（ロール別表示内容: DASH-001〜004）が定義されている
- [ ] テナント全レポート一覧（Admin, Accounting）の仕様が定義されている
- [ ] ロール別の表示・操作制限が明記されている（Member: 自分のみ、Admin: 全件閲覧、Accounting: approved 以降）
- [ ] 再申請フロー（UC-M09）の画面操作が定義されている

**作業上の注意点**:

- ダッシュボードは requirements.md 2.7 の DASH-001〜004 のロール別表示が仕様の核。各ロールで見えるカード/数値/リストを具体的に定義する
- テナント情報表示（UC-AD01）は Admin のみ。設定画面または管理画面の一部として扱う
- 再申請（UC-M09）は「却下レポート詳細画面の再申請ボタン → レポート作成画面（内容コピー済み）」のフローを定義

---

### 4+5-D-2: 経費レポート CRUD + ダッシュボード（API 定義）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.4（経費レポート CRUD）, 2.7（ダッシュボード） |
| ドメインモデル | `20_domain/domain_model.md` | 3.4, 3.5（エンティティ属性） |
| RBAC | `10_requirements/rbac.md` | 3.2（経費レポート権限マトリクス） |
| アーキテクチャ | `30_arch/architecture.md` | 5.1（経費 API 一覧）, 5.3（ページネーション） |
| DB スキーマ | `50_detail_design/db_schema.md` | expense_reports, expense_items テーブル（4+5-A） |
| 経費画面詳細 | `40_basic_design/screens/expenses.md` | 全体（4+5-D-1） |
| OpenAPI 定義 | `50_detail_design/openapi.yaml` | 共通定義部分（4+5-C-2） |

**出力**:

| 成果物 | パス |
|--------|------|
| OpenAPI 定義（追記） | `deliverables/docs/50_detail_design/openapi.yaml` |

**完了条件**:

- [ ] 経費レポート CRUD エンドポイント（GET/POST/PUT/DELETE /reports, GET /reports/all）が定義されている
- [ ] 経費明細 CRUD エンドポイント（POST/PUT/DELETE /reports/:id/items）が定義されている
- [ ] ダッシュボードエンドポイント（GET /dashboard）が定義されている
- [ ] テナント情報取得エンドポイント（GET /tenant）が定義されている
- [ ] ページネーション・フィルタリング・ソートのクエリパラメータが定義されている
- [ ] rbac.md 3.2 および architecture.md 5.1 の経費エンドポイントとの差分が記録されている

**作業上の注意点**:

- openapi.yaml に **追記** するタスク。4+5-C-2 で作成した共通スキーマを参照し、経費固有のスキーマを追加する
- レポート作成 API に `reference_report_id`（再申請元レポートID）をオプションパラメータとして含める
- ダッシュボード API のレスポンスはロール別に返却内容が異なる。API 仕様でそれを明記する

---

### 4+5-E-1: 添付ファイル（画面詳細化）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.6（添付ファイル） |
| ドメインモデル | `20_domain/domain_model.md` | 3.6（Attachment）, 4.4（MimeType） |
| ユースケース | `10_requirements/usecases.md` | UC-M03, UC-M03a |
| 経費画面詳細 | `40_basic_design/screens/expenses.md` | レポート詳細・明細画面（4+5-D-1） |

**出力**:

| 成果物 | パス |
|--------|------|
| 添付画面詳細 | `deliverables/docs/40_basic_design/screens/attachments.md` |

**完了条件**:

- [ ] 添付ファイルのアップロード UI 仕様（ファイル選択・ドラッグ&ドロップ・プログレス表示）が定義されている
- [ ] ファイル形式制限（JPEG, PNG, PDF）とサイズ制限（5MB）のバリデーション表示が定義されている
- [ ] 添付ファイルのプレビュー・ダウンロード操作が定義されている
- [ ] 添付ファイルの削除操作（確認ダイアログ含む）が定義されている
- [ ] draft 以外の状態ではアップロード・削除ボタンが非表示/無効になることが明記されている

**作業上の注意点**:

- 添付ファイルは経費明細（ExpenseItem）に紐づく。レポート詳細画面内の明細セクションに統合される UI を想定
- 経費画面詳細（expenses.md）で定義した明細表示との連携を意識する

---

### 4+5-E-2: 添付ファイル（API + S3 設計）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.6（添付ファイル）, ATT-002〜ATT-020 |
| ドメインモデル | `20_domain/domain_model.md` | 3.6（Attachment エンティティ） |
| RBAC | `10_requirements/rbac.md` | 3.4（明細・添付の権限マトリクス） |
| アーキテクチャ | `30_arch/architecture.md` | 5.1（添付 API 一覧） |
| DB スキーマ | `50_detail_design/db_schema.md` | attachments テーブル（4+5-A） |
| 添付画面詳細 | `40_basic_design/screens/attachments.md` | 全体（4+5-E-1） |
| OpenAPI 定義 | `50_detail_design/openapi.yaml` | 共通定義（4+5-C-2） |

**出力**:

| 成果物 | パス |
|--------|------|
| OpenAPI 定義（追記） | `deliverables/docs/50_detail_design/openapi.yaml` |
| 添付ファイル設計 | `deliverables/docs/50_detail_design/files.md` |

**完了条件**:

- [ ] 添付ファイルエンドポイント（POST, GET, GET/:attId, DELETE）が OpenAPI で定義されている
- [ ] 署名付き URL の発行フロー（アップロード用・ダウンロード用）が files.md で図示されている
- [ ] S3 バケット構成とキー命名規則（`{tenant_id}/{report_id}/{attachment_id}`）が定義されている
- [ ] MIME バリデーション（JPEG, PNG, PDF）とサイズ制限（5MB）の実装方式が定義されている
- [ ] 認可チェック（リクエスト者がそのレポートにアクセス可能か）の責務が明記されている
- [ ] rbac.md 3.4 および architecture.md 5.1 の添付エンドポイントとの差分が記録されている

**作業上の注意点**:

- 署名付き URL 方式のフロー（クライアント → API で URL 取得 → クライアント → S3 に直接アップロード）を明確にする
- S3 パスの命名規則は domain_model.md の ATT-014 に従う（ただし `{file_id}` ではなく `{attachment_id}` に統一）

---

### 4+5-F-1: 承認フロー（画面詳細化）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.5（承認フロー） |
| 状態遷移 | `20_domain/state_machine.md` | 3（許可される遷移）, 5（操作可否マトリクス） |
| ワークフロー | `10_requirements/workflow.md` | 4（許可される遷移）, 7（却下と再申請） |
| ユースケース | `10_requirements/usecases.md` | UC-A01〜03, UC-AC01〜02 |
| RBAC | `10_requirements/rbac.md` | 3.3（承認フロー権限） |
| 経費画面詳細 | `40_basic_design/screens/expenses.md` | レポート詳細画面（4+5-D-1） |

**出力**:

| 成果物 | パス |
|--------|------|
| 承認画面詳細 | `deliverables/docs/40_basic_design/screens/approvals.md` |

**完了条件**:

- [ ] 承認待ち一覧画面の仕様（Approver 向け: フィルタ・ソート・申請者名・合計金額表示）が定義されている
- [ ] 承認/却下操作の画面仕様（確認ダイアログ・コメント入力・理由入力）が定義されている
- [ ] 支払待ち一覧画面の仕様（Accounting 向け）が定義されている
- [ ] 支払完了操作の画面仕様（確認ダイアログ）が定義されている
- [ ] 自己承認禁止の UI 表現（自分のレポートには承認/却下ボタンを非表示 or 無効化）が定義されている
- [ ] 承認/却下後の画面遷移が定義されている

**作業上の注意点**:

- 承認操作はレポート詳細画面の延長として行われる。expenses.md のレポート詳細画面との連携を意識
- 却下理由は必須入力（WFL-012）。UI で空送信を防ぐバリデーションを含める

---

### 4+5-F-2: 承認フロー（API 定義）

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| 要件定義 | `10_requirements/requirements.md` | 2.5（承認フロー機能一覧） |
| 状態遷移 | `20_domain/state_machine.md` | 3（T1〜T4 の事前条件・事後処理）, 4（禁止遷移） |
| ワークフロー | `10_requirements/workflow.md` | 4（許可される遷移） |
| RBAC | `10_requirements/rbac.md` | 3.3（承認フロー権限マトリクス） |
| アーキテクチャ | `30_arch/architecture.md` | 5.1（承認 API 一覧） |
| DB スキーマ | `50_detail_design/db_schema.md` | expense_reports テーブル（4+5-A） |
| 承認画面詳細 | `40_basic_design/screens/approvals.md` | 全体（4+5-F-1） |
| OpenAPI 定義 | `50_detail_design/openapi.yaml` | 共通定義 + 経費スキーマ（4+5-C-2, D-2） |

**出力**:

| 成果物 | パス |
|--------|------|
| OpenAPI 定義（追記） | `deliverables/docs/50_detail_design/openapi.yaml` |

**完了条件**:

- [ ] 承認フローエンドポイント（submit, approve, reject, mark-paid, pending, payable）が定義されている
- [ ] 各エンドポイントの事前条件（状態制約、ロール制約、自己承認禁止）がレスポンスに反映されている
- [ ] 不正遷移時のエラーレスポンス（422 INVALID_STATE_TRANSITION 等）が定義されている
- [ ] rbac.md 3.3 および architecture.md 5.1 の承認エンドポイントとの差分が記録されている

**作業上の注意点**:

- submit エンドポイントは `/api/reports/:id/submit` で経費 CRUD タグに含まれる可能性があるが、状態遷移の一環として workflow タグに含めるのが自然。API 設計時に整理する
- 楽観的ロック（409 Conflict）のエラーレスポンスを含める

---

### 4+5-G: 認可設計 + 最終統合

**入力（参照すべき上流成果物）**:

| 成果物 | パス | 該当セクション |
|--------|------|---------------|
| RBAC | `10_requirements/rbac.md` | 3〜4（権限マトリクス全体、アクセス制御ルール） |
| セキュリティ設計 | `50_detail_design/security.md` | 全体（4+5-B） |
| アーキテクチャ | `30_arch/architecture.md` | 5.1（API 一覧全体） |
| OpenAPI 定義 | `50_detail_design/openapi.yaml` | 全エンドポイント（4+5-C-2〜F-2） |
| 全画面詳細 | `40_basic_design/screens/*.md` | 全機能の画面仕様（4+5-C-1〜F-1） |
| 画面一覧 | `40_basic_design/screens.md` | 全体（4+5-I） |
| 画面遷移図（初版） | `40_basic_design/ui_flow.md` | 全体（4+5-I） |
| DB スキーマ | `50_detail_design/db_schema.md` | 全体（4+5-A） |

**出力**:

| 成果物 | パス |
|--------|------|
| 認可設計 | `deliverables/docs/50_detail_design/authz.md` |
| 画面遷移図（最終版） | `deliverables/docs/40_basic_design/ui_flow.md` |

**完了条件**:

- [ ] 全エンドポイント x 全ロール（Admin, Approver, Member, Accounting, 未認証）の認可マトリクスが一覧化されている
- [ ] 認可チェックの実装場所（ミドルウェア / ハンドラ / ドメイン層 / リポジトリ）がエンドポイントごとに明確
- [ ] テナント分離の認可チェック（アプリ層 + RLS の二重保証）の設計が明記されている
- [ ] リソースオーナーシップチェック（自分のレポートのみ操作可能）の実装方針が明記されている
- [ ] 自己承認禁止の実装方針が明記されている
- [ ] 画面遷移図が全機能の画面詳細化結果を反映し、最終版として整合している
- [ ] 全 API エンドポイントの最終一覧と上流（rbac.md, architecture.md 5.1）との差分が記録されている
- [ ] 画面ID → API パス → DB テーブル → 認可ルール の一貫性が検証されている

**作業上の注意点**:

- authz.md は rbac.md の要件レベルの権限マトリクスを「実装レベル」に落とすもの。具体的には:
  - rbac.md: 「レポート編集は ※（所有者のみ）」
  - authz.md: 「PUT /api/reports/:id → RBAC ミドルウェア: [Admin, Approver, Member] → ハンドラ: report.user_id == current_user.id → リポジトリ: WHERE tenant_id = ?」
- ui_flow.md の最終更新では、各機能タスクで追加・変更された画面（確認ダイアログ等）を反映する
- 上流 API 一覧との差分記録: 各機能タスク（C-2〜F-2）で記録された差分を集約し、最終的な差分一覧を作成する

---

## リスク・注意事項

### 並列実行時のリスク

| # | リスク | 対策 |
|---|--------|------|
| 1 | openapi.yaml への同時書き込み | Wave 内での追記順を制御。同一 Wave で openapi.yaml に書き込むタスクが 2 つある場合、指揮役が直列実行を保証する |
| 2 | detail-designer の過負荷（Wave 1 で B+H、Wave 2 で C-2+D-2） | B → H を直列化。Wave 2 では C-2 → D-2 を直列化（openapi.yaml 制約と一致）。実質的に detail-designer は常に 1 タスクずつ処理する |
| 3 | basic-designer の過負荷（Wave 2 で C-1+D-1 並列） | C-1 と D-1 は出力ファイルが独立（screens/auth.md vs screens/expenses.md）のため並列実行可能。ただし同一エージェントの場合は指揮役がいずれか先に完了した方から次のタスクを起動する |

### 上流成果物のリスク

| # | リスク | 対策 |
|---|--------|------|
| 4 | architecture.md 5.1 の API 一覧と設計成果物の乖離 | 各機能タスク（C-2〜F-2）で差分を記録。Wave 4 で最終差分一覧を作成し、必要に応じて issue を起票 |
| 5 | ダッシュボード API のレスポンス仕様が上流で未詳細 | 4+5-D-1（画面）と 4+5-D-2（API）で requirements.md 2.7 の DASH-001〜004 を基に詳細化する |

### スコープに関する注意

- メンバー管理画面・招待フローは Phase 3。設計しない
- テナント設定画面は Phase 3。テナント情報の**取得** API のみ MVP 対象
- CSV エクスポートは Phase 3。設計しない
- 監査ログ閲覧画面は Phase 3。audit_logs テーブルの DDL のみ 4+5-A で事前設計する

---

## 作業ログ

| 日時 | セッション | Wave | 内容 |
|------|-----------|------|------|
| 2026-03-18 | - | Phase 0 | タスク実行計画を作成 |
