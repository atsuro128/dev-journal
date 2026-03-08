# 012 Member List API Phase Boundary Ambiguous

## 指摘概要
メンバー管理関連のスコープ境界が、MVP/Phase 3 の間で揺れている。

## 根拠
- `02_scope.md` は「メンバー管理画面」を Phase 3 追加機能に分類  
  `deliverables/docs/02_scope.md:73`
- `requirements.md` は「メンバー一覧APIは MVP で先行実装の可能性あり」と記載  
  `deliverables/docs/10_requirements/requirements.md:342`

## 期待仕様
- Step1完了時点で MVP 境界が一意に読めること

## 改善案
- `requirements.md` の記述を「Phase 3 固定」または「MVP先行実装確定」に変更
- 未確定のまま残すなら、決定日と責任者を併記

## 影響度判定
中（次フェーズの設計範囲に直接影響するため）

