# メンバー管理のPhase境界が「Phase 3固定」と「MVP先行可能」で曖昧

## 発見日
2026-03-05

## カテゴリ
requirements

## 影響度
中

## 関連ステップ
Step 1（要件定義）

## 問題
`02_scope.md` では「メンバー管理画面」は Phase 3 追加機能だが、`requirements.md` では「メンバー一覧APIはMVP先行実装の可能性あり」と記載され、境界が曖昧。

## 影響
- Step2以降で API 設計・画面設計・テスト範囲が分岐する
- MVP工数見積もりと完了条件判定がぶれる

## 提案
- Step1時点で「MVPに含める/含めない」を確定し、`02_scope.md` と `requirements.md` を同一表現に統一
- もし未確定で残す場合は、決定期限と責任者を明記

## 参照
- `review-findings/012-member-list-api-phase-boundary-ambiguous.md`

---

## 解決内容
- `requirements.md` のメンバー管理画面「MVP への影響」を「MVP 対象外。Phase 3 で実装」に修正し、`02_scope.md` と統一

## 解決日
2026-03-05

