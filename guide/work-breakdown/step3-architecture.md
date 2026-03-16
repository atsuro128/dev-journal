# Step 3: アーキテクチャ設計 — 作業分解

## 概要

技術選定と全体構成を決定する。ADR で判断を記録し、architecture.md / diagrams.md で全体像をまとめる。

## 前提

### 上流成果物（入力）

| 成果物 | パス | 用途 |
|--------|------|------|
| ドメインモデル | `deliverables/docs/20_domain/domain_model.md` | エンティティ構造・集約・不変条件 |
| 状態遷移 | `deliverables/docs/20_domain/state_machine.md` | 状態遷移の詳細・禁止操作 |
| 要件定義 | `deliverables/docs/10_requirements/requirements.md` | 非機能要件（レスポンスタイム・可用性等） |
| RBAC | `deliverables/docs/10_requirements/rbac.md` | ロール・権限構造 |
| スコープ | `deliverables/docs/02_scope.md` | MVP / Phase 3 の境界 |

### 成果物（出力）

| 成果物 | パス |
|--------|------|
| ADR 0001 | `deliverables/docs/30_arch/adr/0001-tech-stack.md` |
| ADR 0002 | `deliverables/docs/30_arch/adr/0002-multi-tenant.md` |
| ADR 0003 | `deliverables/docs/30_arch/adr/0003-rls-tenant-isolation.md` |
| ADR 0004 | `deliverables/docs/30_arch/adr/0004-infra.md` |
| ADR 0005 | `deliverables/docs/30_arch/adr/0005-monitoring-logging.md` |
| アーキテクチャ | `deliverables/docs/30_arch/architecture.md` |
| 構成図 | `deliverables/docs/30_arch/diagrams.md` |

### 完了条件

- 「なぜその技術か」を ADR で短く言える
- 図（構成図・データフロー）が1枚以上ある
- テナント分離の二重保証（アプリ層 + RLS）の方針が決まっている
- 監視・ログの方針（何を監視し、どう記録するか）が決まっている

---

## タスク一覧

| ID | タスク | 入力 | 出力 | 依存 | 並列可 | 状態 |
|----|--------|------|------|------|--------|------|
| 3-1 | ADR: 技術スタック選定 | requirements.md §4.1, §4.2 | adr/0001-tech-stack.md | - | Yes | 未着手 |
| 3-2 | ADR: マルチテナント方式 | domain_model.md §2, §3, §5 | adr/0002-multi-tenant.md | - | Yes | 未着手 |
| 3-3 | ADR: RLS テナント分離 | domain_model.md §3, §6, requirements.md §4.2, ADR 0002 | adr/0003-rls-tenant-isolation.md | 3-2 | No | 未着手 |
| 3-4 | ADR: インフラ選定 | requirements.md §4.1, §4.3, §4.4 | adr/0004-infra.md | - | Yes | 未着手 |
| 3-5 | アーキテクチャ設計 | 3-1〜3-7 の全 ADR | architecture.md | 3-1,3-2,3-3,3-4,3-7 | No | 未着手 |
| 3-6 | 構成図・データフロー図 | architecture.md | diagrams.md | 3-5 | No | 未着手 |
| 3-7 | ADR: 監視・ログ戦略 | requirements.md（非機能要件）, 3-4 の結論 | adr/0005-monitoring-logging.md | 3-4 | No | 未着手 |

### 依存グラフ

```
3-1 (tech-stack) ─────────────────┐
3-2 (multi-tenant) ─→ 3-3 (RLS) ─→ 3-5 (architecture) → 3-6 (diagrams)
3-4 (infra) ─→ 3-7 (monitoring) ─┘
```

**並列実行パターン**: 3-1, 3-2, 3-4 を同時実行 → 3-3, 3-7 を同時実行 → 3-5 → 3-6

---

## タスク詳細

### 3-1: ADR — 技術スタック選定

- **判断対象**: Go / React (TypeScript, Vite) / PostgreSQL / その他ライブラリ
- **書くべきこと**: 候補の比較、選定理由、トレードオフ
- **参照先**:
  - CLAUDE.md §技術スタック（既に決定済みだが、「なぜ」を記録する）
  - requirements.md §4.1（パフォーマンス目標 — 技術選定の制約）
  - requirements.md §4.2（セキュリティ要件 — JWT RS256、Argon2id 等）
- **完了条件**: 各技術について「なぜ選んだか」「何を不採用にしたか」が ADR として読める

### 3-2: ADR — マルチテナント方式

- **判断対象**: shared DB + tenant_id vs DB per tenant vs schema per tenant
- **書くべきこと**: 3方式の比較、shared DB を選んだ理由、リスクと緩和策
- **参照先**:
  - domain_model.md §2（ER図 — Tenant の関係構造）
  - domain_model.md §3（エンティティ定義 — Tenant / TenantMembership の属性・不変条件）
  - domain_model.md §5（集約設計 — テナント境界の扱い）
- **完了条件**: テナント分離方式の判断根拠が明確

### 3-3: ADR — RLS テナント分離

- **判断対象**: RLS の適用範囲、SET app.current_tenant の方式、アプリ層との二重保証
- **書くべきこと**: RLS で何を保証するか、アプリ層の WHERE tenant_id との棲み分け
- **依存**: 3-2 のマルチテナント方式が確定していること
- **参照先**:
  - domain_model.md §3（エンティティ定義 — 各エンティティの tenant_id 保持構造）
  - domain_model.md §6（不変条件 — テナント分離に関するドメインルール）
  - requirements.md §4.2（セキュリティ要件 — TNT-001〜005）
  - ADR 0002（3-2 の結論）
- **完了条件**: RLS ポリシーの方針（どのテーブルに適用するか）が決まっている

### 3-4: ADR — インフラ選定

- **判断対象**: AWS ECS Fargate / RDS / S3 / IaC (Terraform or CDK)
- **書くべきこと**: 選定理由、コスト考慮、デプロイパイプライン方針
- **参照先**:
  - requirements.md §4.1（パフォーマンス — 同時接続100ユーザー等の規模感）
  - requirements.md §4.3（可用性 — 99.5%、ヘルスチェック、DBバックアップ）
  - requirements.md §4.4（データ — バックアップ・タイムゾーン等）
- **完了条件**: インフラ構成の全体像が決まっている

### 3-5: アーキテクチャ設計

- **作業内容**: ADR の結論を統合し、システム全体のレイヤー構成・通信フロー・責務分担をまとめる
- **含めるべき内容**:
  - バックエンドのレイヤー構成（Handler → Service → Repository）
  - フロントエンドの構成（ページ / コンポーネント / API クライアント）
  - 認証フロー（JWT 発行 → 検証 → RLS 設定の流れ）
  - テナント分離の実行フロー
  - 監視・ログの全体方針（ADR 0005 の結論を反映）
- **完了条件**: システム全体の責務分担が1つの文書で把握できる

### 3-6: 構成図・データフロー図

- **作業内容**: architecture.md の内容を Mermaid で図にする
- **最低限の図**:
  - システム構成図（フロント → API → DB / S3 の関係）
  - リクエスト処理フロー（認証 → 認可 → RLS → ビジネスロジック）
- **完了条件**: 図が1枚以上あり、architecture.md と整合している

### 3-7: ADR — 監視・ログ戦略

- **判断対象**: ログフレームワーク、構造化ログ方針、ヘルスチェック、メトリクス収集、アラート戦略
- **書くべきこと**:
  - ログ: 構造化ログライブラリの選定（slog / zerolog 等）、ログレベル方針、出力先
  - ヘルスチェック: `/health` エンドポイントの設計（DB接続確認を含むか等）
  - メトリクス: 収集方式の選定（CloudWatch / Prometheus 等）、MVP で計測すべき指標
  - アラート: 閾値の方針（p95 レスポンスタイム、エラーレート等）
- **依存**: 3-4 のインフラ選定が確定していること（監視基盤はインフラに依存）
- **参照先**: requirements.md §4.1（パフォーマンス目標）, §4.3（可用性・ヘルスチェック）
- **完了条件**: 「何を監視し、どう記録するか」の方針が ADR として読める
