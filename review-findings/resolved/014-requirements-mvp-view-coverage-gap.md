# 014: requirements.md にMVP閲覧要件の定義漏れがある

## 指摘概要
MVPで必要な閲覧系ユースケース（Admin/Accounting）の一部が `requirements.md` に要件ID付きで定義されていない。

## 根拠
- `requirements.md` ではレポート要件が「自分の一覧/詳細」「承認待ち」「支払待ち」中心
  - `deliverables/docs/10_requirements/requirements.md:135`
  - `deliverables/docs/10_requirements/requirements.md:214`
  - `deliverables/docs/10_requirements/requirements.md:215`
- `usecases.md` ではMVPとして以下が定義済み
  - `UC-AC03` 経費一覧を閲覧する
  - `UC-AD01` テナント情報を確認する
  - `UC-AD02` 全レポートを閲覧する
  - `deliverables/docs/10_requirements/usecases.md:327`
  - `deliverables/docs/10_requirements/usecases.md:346`
  - `deliverables/docs/10_requirements/usecases.md:361`
- `rbac.md` でも Admin/Accounting の閲覧権限が定義済み
  - `deliverables/docs/10_requirements/rbac.md:76`

## 判定
要件トレーサビリティ不備（中）。

## 修正方針案
`requirements.md` に Admin/Accounting の閲覧機能を要件ID付きで追加し、`usecases.md`/`rbac.md` と相互参照を張る。

