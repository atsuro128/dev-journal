# 017: RBC-010 の文言が所有権チェック対象を曖昧にしている

## 指摘概要
`RBC-010` の主語が Member のみで記述されており、Approver/Admin を含む所有権ルールの適用範囲が曖昧。

## 根拠
- `rbac.md`:
  - ApproverはMember操作可能
  - `deliverables/docs/10_requirements/rbac.md:52`
  - レポート作成は Admin/Approver/Member 許可（所有権注記あり）
  - `deliverables/docs/10_requirements/rbac.md:73`
  - `RBC-010` は Member のみを主語にしている
  - `deliverables/docs/10_requirements/rbac.md:134`

## 判定
ルール文言の曖昧性（低）。

## 修正方針案
`RBC-010` を申請系ロール横断の表現に修正し、所有権ルールを明確化する。

