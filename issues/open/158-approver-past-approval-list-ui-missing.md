# Approver が過去に承認/却下したレポートを UI から探す動線が存在しない（API は対応済み、画面側のみ未対応）

## 発見日
2026-04-28

## カテゴリ
ui-design / scope

## 影響度
中（API は仕様通り。UI 動線欠如のため、Approver が承認後のレポートを業務追跡できない。設計書規定と UI 実装の乖離）

## 発見経緯
user-report / Step 11-A SMK-095 検証後、ユーザーから「承認したレポートを閲覧できない」との指摘。確認したところ、詳細 URL 直打ちでは閲覧可能（API は仕様通り）だが、一覧から探す動線が UI にないことが判明

## 関連ステップ
Step 1（要件定義）/ Step 4（基本設計）/ Step 5（詳細設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書の規定

`dev-journal/deliverables/docs/50_detail_design/authz.md:585-597` で Approver の閲覧範囲が以下の通り規定されている:

| API | 可視範囲 | 説明 |
|-----|---------|------|
| `GET /api/reports`（自分のレポート一覧） | 自分が作成した全レポート（全状態） | 申請者としての閲覧 |
| `GET /api/workflow/pending`（承認待ち一覧） | テナント内 submitted レポート全件 | 承認者としての閲覧。自分のレポート含む |
| `GET /api/reports/{id}`（詳細） | 自分のレポート + submitted + **過去に自分が承認/却下したレポート** | 承認者が過去の判断結果を確認可能 |

設計書本文（同 L597）:
> 「これにより、Approver は過去に承認/却下したレポートのその後の状態（paid など）を追跡できる」

### 現状

- **API**: 設計通り。詳細 URL 直打ち（`GET /api/reports/{id}`）で過去承認/却下分も閲覧可能（ユーザー手動検証で確認済み）
- **UI**: 一覧から探す動線なし

### 画面一覧（`40_basic_design/screens.md`）の整理

| 画面 | URL | Approver 対象 | Approver から見た過去承認分の可視性 |
|------|-----|--------------|------------------------------|
| SCR-RPT-001 レポート一覧（自分） | `/reports` | ✓ | ✗ 自分作成のみ → 過去承認分は出ない |
| SCR-WFL-001 承認待ち一覧 | `/approvals` | ✓ | ✗ submitted のみ → 承認後は消える |
| SCR-ADM-001 テナント全レポート一覧 | `/reports/all` | **✗（Admin / Accounting のみ）** | — |

→ Approver が過去承認/却下したレポートを **UI 上で一覧表示する経路が皆無**。詳細 URL を別途記録しておかなければ業務追跡不可能。

## 影響

- **業務影響**: Approver が「自分が承認したレポートのその後の状態（approved → paid に進んだか等）」を追跡する手段が UI で提供されていない。設計意図と UI 実装の乖離
- **UX 影響**: 「承認したレポートが消えて見つからない」という UX 違和感。SMK 検証時にユーザーが即座に気付いた
- **設計書整合**: 設計書 `authz.md` で「Approver は過去の判断結果を確認可能」と謳っているのに、UI で実現されていない

## 提案

### 対応方針（要選択）

**案1: SCR-ADM-001 を Approver にも開放（推奨、最小変更）**

- `dev-journal/deliverables/docs/40_basic_design/screens.md` の SCR-ADM-001 対応ロール欄に Approver を追加
- API 側の visibility scope に Approver 用の分岐追加:
  - Approver の場合: `WHERE tenant_id = ? AND (user_id = actor.user_id OR status = 'submitted' OR approved_by = actor.user_id OR rejected_by = actor.user_id)`
- フロントエンド側: ナビメニューに「全レポート」を Approver でも表示。フィルタ UI に「自分が承認したもの」「自分が却下したもの」のステータス選択肢を追加
- メリット: 既存画面の流用、新規画面不要
- デメリット: フィルタ UI 拡張 + Authorizer の visibility scope 変更が必要

**案2: Approver 用に「承認/却下履歴」一覧画面を新規追加（SCR-WFL-003 等）**

- 専用画面として `/approvals/history` などを新規追加
- 表示内容: 自分が approved_by / rejected_by に記録されているレポート一覧（status 不問）
- メリット: 動線が独立。Approver の業務文脈に最適化
- デメリット: 新規画面分の設計・実装コスト大

**案3: ダッシュボードに「最近承認したレポート」セクション追加**

- ダッシュボードに直近 N 件のみ表示
- 詳細は URL 直打ち継続
- メリット: 軽量
- デメリット: 一覧で全件閲覧する手段は依然なし。ダッシュボード情報量が増える

**案4: 承認待ち一覧にタブ切替（pending / approved-by-me）追加**

- `/approvals` 画面内でタブ切替
- メリット: 動線が承認業務の文脈に統合される
- デメリット: 承認待ち一覧の責務が拡大

### 推奨

**案1（SCR-ADM-001 Approver 開放）** が最小変更で本質解決。

理由:
- 既存の `/reports/all` 画面に「テナント全レポート閲覧」の枠組みが既にある
- Accounting も類似の業務（過去支払対象を追跡）で同画面を使っており、Approver の業務とも親和性が高い
- 新規画面 / ダッシュボード拡張なしで動線を提供できる

ただし、Approver の閲覧範囲を Admin / Accounting と同等にするのは設計上不適切（Approver は自分が関わっていないレポートを横断閲覧する権限はない想定）。**API 側で Approver 用の visibility scope を適用**することが必須。

## 修正対象ファイル（推定、案1 採用時）

| ファイル | 変更種別 |
|---|---|
| `dev-journal/deliverables/docs/40_basic_design/screens.md` | SCR-ADM-001 対応ロールに Approver 追加 |
| `dev-journal/deliverables/docs/50_detail_design/authz.md` | Approver の visibility scope 規定追加（一覧取得時） |
| `dev-journal/deliverables/docs/50_detail_design/screens/admin-all-reports.md` | Approver 表示時の UX 規定追加（フィルタ初期値・項目構成） |
| `expense-saas/internal/service/report_service.go`（または同等） | List API の visibility scope に Approver 分岐追加 |
| `expense-saas/internal/repository/postgres/sqlcgen/expense_reports.sql.go` | Approver 用クエリ追加（`approved_by = ? OR rejected_by = ?` フィルタ） |
| `expense-saas/frontend/src/pages/admin/AllReportsPage.tsx` | Approver でアクセス可能に + フィルタ拡張 |
| `expense-saas/frontend/src/components/layout/AppLayout.tsx`（ナビ） | Approver にも「全レポート」メニュー表示 |
| 各種テスト | 認可テスト・E2E テスト追加 |

## 完了条件

- Approver で `/reports/all` にアクセスでき、自分が承認/却下したレポートが一覧表示される
- Approver の visibility scope（自分作成 + submitted + approved_by/rejected_by に自分が記録）が API 側で正しく適用されている
- 設計書（authz.md / screens.md）に Approver の一覧取得時 scope が規定されている
- 既存テスト（テナント分離 / RBAC）が引き続き通る + 新規テスト追加

## ラベル

- type: gap / scope / ui
- area: frontend / backend / docs

## 関連

- 設計書: `50_detail_design/authz.md` §10.2（Approver の閲覧範囲）/ §10.3（Visibility Scope の組み立て方式）
- 関連 SMK: SMK-095（承認確認ダイアログ）の副次発見

## MVP 区分

MVP（業務上必要な動線、UX 影響大、設計書規定と UI 実装の乖離）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
