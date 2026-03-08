# 011 Self Approval/Rejection Rule Mismatch

## 指摘概要
Approver の自己操作禁止ルールが、文書ごとに「承認のみ」と「承認/却下」の解釈で揺れている。

## 根拠
- `rbac.md` は「Approver が自分のレポートを承認/却下できない」と明記  
  `deliverables/docs/10_requirements/rbac.md:143`
- `workflow.md` は自己禁止の対象説明が「承認する操作」かつ「承認API実行時」のみ  
  `deliverables/docs/10_requirements/workflow.md:182`  
  `deliverables/docs/10_requirements/workflow.md:184`
- `workflow.md` の T3（却下遷移）には自己操作禁止条件が書かれていない  
  `deliverables/docs/10_requirements/workflow.md:76`

## 期待仕様
- 自己操作禁止は `approve` と `reject` の両方で同一ルールとして扱う

## 改善案
- `workflow.md` に「自己操作禁止（approve/reject 共通）」を明文化
- T2/T3 の事前条件に `report.created_by != current_user.id` を追加

## 影響度判定
高（認可の実装差分がセキュリティ仕様の欠落につながるため）

