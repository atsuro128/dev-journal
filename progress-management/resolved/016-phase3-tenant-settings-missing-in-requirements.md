# Phase 3「テナント設定画面」が requirements.md に未記載

## 発見日
2026-03-05

## 関連ステップ
Step 1（要件定義）

## カテゴリ
requirements

## 影響度
中

## 問題
`02_scope.md` の Phase 3 追加機能には「テナント設定画面（会社情報編集）」が含まれているが、`requirements.md` の Phase 3 要件には記載がない。

## 影響
- Phase 3計画時に対象機能の認識ズレが起きる
- 画面/API/権限設計で実装漏れ・重複議論が発生する

## 提案
- `requirements.md` の Phase 3 に「テナント設定画面」を追加する
  - 例: `3.6 テナント設定画面`
  - 権限: Admin のみ
  - MVPへの影響: テナント基本情報モデルの拡張性確保
- もし意図的に除外する場合は、`02_scope.md` との変更差分を明記して判断を確定する

## 参照
- `review-findings/016-phase3-tenant-settings-missing-in-requirements.md`

---

## 解決内容

- `requirements.md` Phase 3 セクションに「3.6 テナント設定画面」を追加
  - 概要: 会社情報の編集画面（Admin のみ）
  - MVP への影響: テナント基本情報取得API（ADM-F01）はMVPで提供、編集画面は Phase 3
- `02_scope.md` との整合を確認済み

## 解決日
2026-03-05
