# requirements.md にMVP閲覧要件の定義漏れがある

## 発見日
2026-03-05

## 関連ステップ
Step 1（要件定義）

## カテゴリ
requirements

## 影響度
中

## 問題
`requirements.md` では、レポート機能の要件が「自分のレポート一覧/詳細」「承認待ち一覧」「支払待ち一覧」までしか定義されていない。  
一方で `usecases.md` と `rbac.md` では、MVPとして以下が定義済み:

- Accounting の「経費一覧を閲覧する（UC-AC03）」
- Admin の「テナント情報を確認する（UC-AD01）」
- Admin の「全レポートを閲覧する（UC-AD02）」

要件IDとの対応関係が欠けており、トレーサビリティが崩れている。

## 影響
- 詳細設計（API一覧）で対象機能の取りこぼしが発生する
- テスト設計で「どの要件IDを検証しているか」を示せない
- Step 2以降で文書間の整合確認コストが上がる

## 提案
- `requirements.md` に Admin/Accounting の閲覧系を要件ID付きで追加する
  - 例: `RPT-F07` テナント全体レポート一覧取得
  - 例: `RPT-F08` テナント全体レポート詳細取得
  - 例: `ADM-F01` テナント情報取得
- `usecases.md` / `rbac.md` の該当箇所から新規要件IDを参照できるようにする

## 参照
- `review-findings/014-requirements-mvp-view-coverage-gap.md`

---

## 解決内容

- `requirements.md` に以下の要件IDを追加:
  - `RPT-F07`: テナント全レポート一覧取得（Admin/Accounting 用）
  - `ADM-F01`: テナント情報取得
- RPT-F02 の説明を「自分のレポート一覧」と明確化
- 既に rbac.md 権限マトリクス・PROJECT_SUMMARY に定義済みの内容を requirements.md にもトレース可能にした

## 解決日
2026-03-05
