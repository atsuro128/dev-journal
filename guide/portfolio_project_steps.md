# マルチテナント型 経費精算SaaS（ポートフォリオ）全体ステップ資料

この資料は、ポートフォリオとして「実務レベル」を示すために、進め方（フェーズ）と各フェーズの成果物を整理したものです。
※詳細は後で変更してOK。ここは"全体の地図"として使います。

---

## 最終的な成果物（ゴール）

- [ ] **GitHub 公開リポジトリ**（履歴が分かるコミットメッセージ）
- [ ] **デプロイ済み URL**（動作確認可能）
- [ ] **README（英語＋日本語）**
  - 概要 / セットアップ / アーキテクチャ / 技術選定理由 / 制約
- [ ] **アーキテクチャ図**（システム構成・データフロー）
- [ ] （推奨）**運用・監視の記載**（デプロイ、ヘルスチェック、ログ、アラート方針）
- [ ] （推奨）**テナント分離・権限のテスト**（README or CIで言及）
- [ ] （推奨）**添付ファイルの権限制御の明記**（署名付きURL発行前の認可チェック）

---

## 全体ステップ（推奨の順番）

> ポイント：**画面より先に「業務フロー（状態遷移）」と「ドメイン（データ＋ルール）」を固める**と、変更に強く迷子になりにくい。

> **設計の核を先に固めること**：Step 1〜3（要件定義・ドメイン設計・アーキテクチャ設計）は実装の土台になる。特に「状態遷移」「テナント分離ルール」「RBACの責務」「ADR」は後から変えるとコスト大。ここが固まる前に実装に進まない。

> **各ステップの構成**：`deliverables/docs/` への成果物作成と、`root-project/` 管理ファイルの整備を並行して進める。`【root-project 整備】` はそのステップで肉付けすべき管理ファイルを示す。

---

## 各ステップ共通ワークフロー

全ステップ（Step 0〜7）は以下のワークフローに従って進める。
`progress.md` はこのワークフローに基づいてステータスを管理する。

```
1. 成果物作成（Claude）
   ├─ 作成中: 重大な問題 → 即 issue 起票（発見経緯: proactive）
   └─ コミット前: 懸念トリアージ → 高影響度は issue 昇格
   ↓
2. レビュー依頼
   ↓
3. レビュー実施（ユーザー + codex）
   ↓
4. 指摘対応（Claude）
   ↓  ※指摘がなくなるまで 3→4 を繰り返す
5. 完了条件確認
   ↓
6. 完了（progress.md 更新）
```

### 各フェーズの詳細

| # | フェーズ | 担当 | 内容 |
|---|---------|------|------|
| 1 | 成果物作成 | Claude | 各ステップの成果物と【root-project 整備】を作成・コミット。Issue 発掘規約に従い、問題の即時起票・懸念の記録とトリアージを実施 |
| 2 | レビュー依頼 | Claude | 成果物コミット後、`ai-dev-framework/rules/codex-review.md` に従い codex を自動実行 |
| 3 | レビュー実施 | ユーザー + codex | `ai-dev-framework/agents/review-procedure.md` に従いレビュー。指摘は `dev-journal/review-findings/open/` に起票 |
| 4 | 指摘対応 | Claude | `dev-journal/review-findings/open/` の指摘に対応し、`pending-review/` に移動。再レビューは `agents/re-review-procedure.md` に従う |
| 5 | 完了条件確認 | ユーザー | 各ステップの完了条件を全て満たしていることを確認 |
| 6 | 完了 | Claude | `progress.md` のマイルストーン・タスクを更新 |

### progress.md でのトラッキング

各ステップのステータスは以下の値を使う:

| ステータス | 意味 |
|-----------|------|
| 未着手 | まだ開始していない |
| 進行中（成果物作成） | Claude が成果物を作成中 |
| レビュー待ち | 成果物作成完了。レビュー依頼済み |
| 指摘対応中 | レビュー指摘への対応中 |
| 完了 | 完了条件を満たし、レビュー指摘も全て解消 |

### レビュー関連の参照先

| ドキュメント | パス | 用途 |
|------------|------|------|
| codex 指示書 | `root-project/AGENTS.md` | codex の作業開始点（移行予定） |
| 初回レビュー手順 | `ai-dev-framework/agents/review-procedure.md` | レビュー観点・起票ルール |
| 再レビュー手順 | `ai-dev-framework/agents/re-review-procedure.md` | 指摘対応後の再レビュー |
| 指摘管理 | `ai-dev-framework/review-findings/` | open → pending-review → resolved |

---

### 0. 事前準備（プロジェクトの土台づくり）
**目的**：変更が入っても崩れない"前提"を定義する
**成果物（例）**
- `deliverables/docs/00_goals.md`（狙い・評価ポイント・スコープ方針）
- `deliverables/docs/01_glossary.md`（用語集）
- `deliverables/docs/02_scope.md`（MVP/非MVP・やらないこと）

**【root-project 整備】**
- `references/glossary.md`：`deliverables/docs/01_glossary.md` の用語を転記・集約（AI参照用の生きた用語集として運用）
- `references/links.md`：参考にした資料・公式ドキュメントのリンクを記録し始める
- `templates/README-template.md`：最終成果物の `project/README.md` 雛形を確認・調整

**完了条件**
- MVPの範囲が一言で説明できる
- 用語のブレを防げる（申請/承認/差戻し等）

---

### 1. 要件定義（業務理解 → ユースケース化）
**目的**：経費精算の"業務"を理解し、作るものを言語化する
**成果物（例）**
- `deliverables/docs/10_requirements/requirements.md`（機能要件・非機能要件）
  - コア機能：認証・RBAC・経費申請・承認フロー・添付ファイル
  - 付随機能：招待フロー・アプリ内通知・監査ログ・CSVエクスポート
  - 非機能要件：レスポンスタイム・可用性・レート制限・ファイルサイズ制限
- `deliverables/docs/10_requirements/usecases.md`（ユースケース一覧）
  - 申請者・承認者・Accounting・Adminそれぞれの操作シナリオを網羅
  - 招待フロー・通知受信・監査ログ閲覧も含める
- `deliverables/docs/10_requirements/workflow.md`（申請〜承認の状態遷移 ※Mermaid推奨）
- `deliverables/docs/10_requirements/rbac.md`（ロール/権限の要件）

**最低限のMVP業務フロー例**
- 申請 → 承認 → 差戻し → 再申請 → 承認 → 完了

**【root-project 整備】**
- `prompts/requirement.md`：要件整理に使う指示文テンプレートを整備（ユースケース抽出・状態遷移整理などの型）
- `rules/security-policy.md`：セキュリティ要件の基礎方針（認証方式・レート制限・ファイルサイズ制限の方針）を記載

**完了条件**
- 「誰が」「何を」「どの順で」するか説明できる
- 例外（却下/取消など）を"やる/やらない"で決めている
- 招待フロー・通知・監査ログ・CSVエクスポートの扱いが明記されている

---

### 2. ドメイン設計（データとルールの核）
**目的**：実装に入る前に、主要データ構造と不変条件（ルール）を固める
**成果物（例）**
- `deliverables/docs/20_domain/domain_model.md`（主要エンティティと関係）
- `deliverables/docs/20_domain/state_machine.md`（状態遷移の詳細・禁止操作）

**重要な不変条件（例）**
- テナント（workspace）を跨ぐ参照は禁止
- 承認中は編集制限（状態によって操作可否が変わる）
- 添付URL発行前に認可チェック必須
- 監査ログは INSERT ONLY（更新・削除不可）
- 不正な状態遷移（例：`draft` → `approved`）はドメイン層で拒否

**【root-project 整備】**
- `references/decisions/`：ドメイン設計上の採用/不採用の判断ログをファイルとして記録
  （例：`20_domain-invariants.md` — テナント分離をアプリ層に寄せる vs RLS に寄せる等の判断）

**完了条件**
- 主要エンティティと責務が説明できる
- 重要ルールが文章化されている
- audit_logs の INSERT ONLY 制約が明記されている

---

### 3. アーキテクチャ設計（技術選定・構成決定）
**目的**：技術選定と全体構成（クラウド含む）を決める
**成果物（例）**
- `deliverables/docs/30_arch/architecture.md`（システム構成）
- `deliverables/docs/30_arch/diagrams.md`（構成図・データフロー図）
- `deliverables/docs/30_arch/adr/`（ADR：技術選定理由のメモ集）
  - `0001-tech-stack.md`（Rust / React / PostgreSQL / SQLx 等の選定理由）
  - `0002-multi-tenant.md`（テナント分離方式：shared DB + tenant_id）
  - `0003-rls-tenant-isolation.md`（PostgreSQL RLS によるアプリ層との二重保証）
  - `0004-infra.md`（AWS ECS Fargate / RDS / S3 / Terraform 等の選定理由）

**このフェーズで決める代表例**
- フロント/バックエンド構成、DB、ORM（SQLx）、認証方式（JWT RS256）、ファイルストレージ（S3）、デプロイ先（AWS ECS Fargate）
- PostgreSQL RLS の適用方針（アプリ層の WHERE tenant_id との二重保証）
- IaC ツール選定（Terraform or CDK）
- ログ/ヘルスチェック/監視の方針（最低限）

**【root-project 整備】**
- `templates/ADR-template.md`：ADR雛形をプロジェクトに合わせて確認・調整（このフェーズで初めて本格利用）
- `references/tech-stack-notes/`：技術比較メモ・候補・選定根拠を記録（ADRの下書き・補足資料として）
- `references/decisions/`：技術選定の採用/不採用理由を追記
- `skills/architecture/`：設計レビュー観点（テナント分離・状態遷移の漏れ確認など）を整備

**完了条件**
- 「なぜその技術か」をADRで短く言える
- 図（構成図・データフロー）が1枚以上ある
- テナント分離の二重保証（アプリ層 + RLS）の方針が決まっている

---

### 4. 基本設計（画面・画面遷移・UX）
**目的**：画面一覧と責務、権限別の見え方、主要な画面遷移を固める
**成果物（例）**
- `deliverables/docs/40_basic_design/screens.md`（画面一覧・画面仕様の粒度は粗めでOK）
- `deliverables/docs/40_basic_design/ui_flow.md`（画面遷移）
- （任意）Figmaリンク / スクショ / 簡易モック

**【root-project 整備】**
- `skills/frontend/`：フロントエンド実装観点（コンポーネント設計・権限別表示の確認項目）を整備

**完了条件**
- 主要画面（申請、承認キュー、管理画面）が揃っている
- 権限で操作可否が変わる点が明記されている

---

### 5. 詳細設計（API / DB / 認可 / 添付 / セキュリティ）
**目的**：実装できるレベルまで仕様を確定する
**成果物（例）**
- `deliverables/docs/50_detail_design/openapi.yaml`（OpenAPI：全エンドポイント・権限・エラー形式）
- `deliverables/docs/50_detail_design/db_schema.md`（ER図、またはスキーマ定義・インデックス方針）
- `deliverables/docs/50_detail_design/authz.md`（認可設計：RBAC + テナント分離 + RLS設定）
  - 通知・監査ログの権限設計もここに含める
- `deliverables/docs/50_detail_design/files.md`（添付：署名付きURL・発行前チェック・MIMEバリデーション）
- `deliverables/docs/50_detail_design/security.md`（セキュリティ設計）
  - レート制限（認証済み: 100 req/min/user、未認証: 20 req/min/IP）
  - CORS方針、セキュリティヘッダー（HSTS, X-Content-Type-Options 等）
  - `cargo audit` / `npm audit` のCI組み込み方針

**【root-project 整備】**
- `rules/coding-standards.md`：コーディング規約を詳細化（Rust: snake_case・unwrap禁止等 / TypeScript: strict mode・any禁止等）
- `rules/data-handling.md`：tenant_id 強制・RLS・監査ログ INSERT ONLY・署名付きURL発行前チェック等のデータ取り扱いルールを記載
- `skills/backend/`：バックエンド実装チェックリスト（tenant_id 漏れ・エラーハンドリング・unwrap 使用箇所等）を整備
- `skills/db/`：DBスキーマ設計・インデックス方針・マイグレーション手順の観点を整備
- `skills/security/`：認証認可チェック・セキュリティヘッダー・CORS・MIME バリデーション等のレビュー観点を整備

**完了条件**
- APIの入出力が決まっている（OpenAPIで機械可読）
- 主要テーブル/関係・インデックス方針が決まっている
- 認可チェックの責務と実装場所が決まっている
- セキュリティ要件の実装方針が明記されている

---

### 6. テスト設計（"重要領域だけ強く"）
**目的**：品質と"実務意識"を見せる。TDDは全面より部分適用が現実的
**成果物（例）**
- `deliverables/docs/60_test/test_strategy.md`（どの層で何をテストするか）
- `deliverables/docs/60_test/test_cases.md`（重要ケース中心）
- CI（GitHub Actions 等）で自動実行

**テストレベルと対象ツール**
- 単体テスト：Rust 標準テスト（ドメインロジック・状態遷移・バリデーション）
- 統合テスト：Rust + テスト用DB（APIエンドポイント・認証認可・DB操作）
- E2Eテスト：Playwright（申請〜承認〜支払いの一連フロー）

**ポートフォリオで評価が上がるテスト（最優先）**
- テナント分離テスト（Tenant A のユーザーが Tenant B のデータにアクセスできないことを全リソースで検証）
- RBAC（権限）テスト（各ロールが許可されていない操作で 403 を返すことを検証）
- ワークフロー状態遷移テスト（不正な遷移が拒否されることを検証）
- 添付URL発行の認可テスト

**【root-project 整備】**
- `rules/review-checklist.md`：コードレビューチェックリストを整備（テスト観点・セキュリティ観点・テナント分離漏れ確認を含む）

**完了条件**
- CIで自動テスト（単体・統合）が走る
- E2EテストのPlaywright設定が整っている
- READMEに「何を担保しているか」説明がある

---

### 7. 実装・運用（テスト→実装→CI→デプロイ）
**目的**：動くものを出し、継続的に改善できる形にする

**実装フェーズ構成**
- Phase 1（基盤）：プロジェクト初期設定・Docker Compose（ローカル PostgreSQL）・DBマイグレーション・認証API・CI基盤（lint / test / build）
- Phase 2（コア機能）：RBACミドルウェア・経費レポートCRUD・状態遷移・添付ファイル・承認フロー・フロントエンド主要画面
- Phase 3（付随機能）：監査ログ・通知・CSVエクスポート・招待フロー・メンバー管理
- Phase 4（デプロイ・仕上げ）：AWSインフラ構築（Terraform or CDK）・staging/production デプロイ・ヘルスチェック・CloudWatchアラート

**成果物**
- 実装コード、コミット履歴、PR運用（任意でも強い）
- IaCコード（Terraform or CDK）
- デプロイURL（検証用アカウント/デモ手順）
- README（日英）、運用メモ（ログ/ヘルスチェック/アラート方針）

**【root-project 整備】**
- `docker-compose.yml`：ローカル開発用の PostgreSQL サービス定義（Dev Container には DB クライアントのみ含まれるため、DB サーバーは Docker Compose で提供する）（Phase 1 着手時に作成）
- `rules/branching.md`：ブランチ戦略（main / develop / feature / hotfix）を確定・記載（Phase 1 着手前に完了）
- `rules/commit-message.md`：Conventional Commits 規約を詳細化（型の一覧・スコープ例・NG例）（Phase 1 着手前に完了）
- `scripts/setup.sh`：ローカル開発環境のセットアップ手順をスクリプト化（Phase 1 完了後）
- `scripts/lint.sh`：`cargo clippy` / `npm run lint` をまとめた lint スクリプト実装（Phase 1 完了後）
- `scripts/db-reset.sh`：DB リセット・シード投入スクリプト実装（Phase 1 完了後）
- `prompts/implementation.md`：実装フェーズで繰り返し使う指示文テンプレートを整備（Phase 2 着手前）
- `prompts/refactor.md`：リファクタリング指示文テンプレートを整備（Phase 2〜3 中に整備）
- `skills/release/`：リリース前チェックリスト・デプロイ手順・ロールバック手順を整備（Phase 4 着手前）
- `deliverables/docs/30_arch/architecture.md` を公開向けに整形（Phase 4）
- `deliverables/docs/50_detail_design/api.md`：openapi.yaml の概要・利用方法・認証方式を記載（Phase 4）
- `references/tech-stack-notes/`：選定理由サマリの整形（Phase 4）
- `deliverables/docs/70_operations/runbook.md`：デプロイ・障害対応・ヘルスチェック手順を記載（Phase 4）

**完了条件**
- デプロイ済みで第三者が触れる
- READMEだけでセットアップ＆概要が伝わる
- ヘルスチェックエンドポイント（`/health`）が存在する

---

## 推奨 deliverables/docs 構成（最小で実務っぽい）

```text
deliverables/
  docs/
    00_goals.md
    01_glossary.md
    02_scope.md
    10_requirements/
      requirements.md        # コア機能・付随機能（招待/通知/監査ログ/CSV）・非機能要件
      usecases.md            # 全ロールのユースケース（招待・通知・監査ログ含む）
      workflow.md
      rbac.md
    20_domain/
      domain_model.md        # 主要エンティティ・audit_logs の INSERT ONLY 制約を含む
      state_machine.md
    30_arch/
      architecture.md
      diagrams.md
      adr/
        0001-tech-stack.md          # Rust / React / PostgreSQL / SQLx 選定理由
        0002-multi-tenant.md        # shared DB + tenant_id 方式の選定理由
        0003-rls-tenant-isolation.md # PostgreSQL RLS による二重保証の設計判断
        0004-infra.md               # AWS ECS Fargate / Terraform 等の選定理由
    40_basic_design/
      screens.md
      ui_flow.md
    50_detail_design/
      db_schema.md
      openapi.yaml
      authz.md               # RBAC + テナント分離 + RLS設定 + 通知・監査ログの権限
      files.md               # 署名付きURL・MIME バリデーション・発行前認可チェック
      security.md            # レート制限・CORS・セキュリティヘッダー・cargo audit 方針
    60_test/
      test_strategy.md       # 単体 / 統合 / E2E (Playwright) の層別方針
      test_cases.md
```

---

## メモ（運用上のコツ）
- 「迷ったら」**MVPフロー（申請→承認→差戻し→再申請→承認→完了）**に戻ってスコープを守る
- 図は凝らなくてOK。**"あること"が重要**（Mermaidで十分）
- 仕様変更が起きても、`scope / workflow / domain` が更新されていれば破綻しにくい
