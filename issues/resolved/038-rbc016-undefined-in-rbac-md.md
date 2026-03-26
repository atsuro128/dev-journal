# rbac.md に RBC-016（自己承認禁止）の定義が存在しない

## 発見日
2026-03-23

## カテゴリ
requirements

## 影響度
高

## 発見経緯
review

## 関連ステップ
Step 1（要件定義）/ Step 5（詳細設計）

## 問題

openapi.yaml の承認 API（`PUT /api/reports/:id/approve`）が `RBC-016`（自己承認禁止）を認可条件として参照しているが、rbac.md にはルール ID が RBC-010〜RBC-013 までしか定義されていない。RBC-016 の正式な定義が上流成果物に存在しない。

- openapi.yaml: `RBC-016: 自己承認禁止` を参照
- rbac.md SS4: RBC-010〜RBC-013 のみ定義
- security.md SS9.1: `RBC-016` を自己承認禁止として参照

## 影響

authz.md（認可設計）の作成時に、RBC-016 の正式な定義・適用範囲が不明確なまま設計を進めることになる。テスト設計・実装でもルール ID の根拠が辿れない。

## 提案

1. rbac.md SS4 に RBC-014〜RBC-016 を正式に追加定義する（欠番がないか確認）
2. requirements.md の RBAC 関連セクションにも必要に応じて反映
3. authz.md 着手前に解決する（ブロッカー）

---

## 解決内容

rbac.md SS4.2 に RBC-014〜016 を正式定義として追加。SS4.3 を「補足・解釈」に縮退し、正式ルールへの参照に書き換え。

1. **RBC-014**: Admin は自分が作成したレポートに限り申請者として操作可能。他者のレポートは閲覧のみ
2. **RBC-015**: MVP では Approver は全件承認型（担当者割当は Phase 3 以降）
3. **RBC-016**: Approver の自己承認・自己却下禁止

下流の参照箇所（openapi.yaml, security.md, screens/*.md, domain_model.md, state_machine.md）は既に正しい ID を使用しており変更不要。

## 解決日

2026-03-23
