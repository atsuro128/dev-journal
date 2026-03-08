# 016: Phase 3「テナント設定画面」が requirements.md に未記載

## 指摘概要
`02_scope.md` にある Phase 3 機能「テナント設定画面」が、`requirements.md` の Phase 3 セクションに存在しない。

## 根拠
- `02_scope.md`:
  - テナント設定画面（会社情報編集、Adminのみ）
  - `deliverables/docs/02_scope.md:74`
- `requirements.md`:
  - Phase 3 は 3.1〜3.5 まで（招待/通知/監査ログ/CSV/メンバー管理）
  - `deliverables/docs/10_requirements/requirements.md:296`
  - `deliverables/docs/10_requirements/requirements.md:336`

## 判定
スコープ文書との整合不備（中）。

## 修正方針案
`requirements.md` にテナント設定画面の節を追加するか、意図的除外なら `02_scope.md` 側を更新して判断を確定する。

