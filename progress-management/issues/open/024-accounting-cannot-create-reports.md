# Accounting がレポート作成不可は業界標準から乖離

## 発見日
2026-03-19

## 発見経緯
proactive（市場調査による発見）

## 関連ステップ
Step 1（rbac.md, usecases.md, requirements.md）、Step 4（screens.md, ui_flow.md）

## カテゴリ
requirements / rbac

## 問題
現行設計では Accounting ロールはレポート作成不可（経費閲覧・支払完了記録のみ）だが、市場調査の結果、調査した全8製品で経理担当者も自分の経費を申請可能。

「経理担当者も従業員である」という基本前提に基づき、自己申請は許可しつつ、自己処理（自分の申請を自分で経理処理）は禁止するのが業界標準。

## 影響
- Accounting ロールのユーザーが自分の経費を精算できない
- 実運用では経理担当者に別途 Member ロールを持たせるなどの回避策が必要になるが、1ユーザー1ロール制約（RBC-002）と矛盾
- 業界標準から乖離しており、製品としての実用性に疑問

## 提案
1. Accounting に Member 相当の経費申請権限を付与（レポート作成・編集・提出・明細追加・添付）
2. 自己処理禁止ルールを追加: Accounting は自分が作成したレポートの支払完了を記録できない
3. Approver と同様の「自分のレポートは自分で処理できない」パターン

### 影響する上流成果物
- `10_requirements/rbac.md` — 権限マトリクス修正（Accounting の申請系操作を ○/※ に変更）
- `10_requirements/usecases.md` — UC-SYS04 ダッシュボードの Accounting 表示修正
- `10_requirements/requirements.md` — DASH-003 の修正
- `40_basic_design/screens.md` — Accounting のサイドナビにマイレポート・レポート作成を追加
- `40_basic_design/ui_flow.md` — Accounting の遷移パスに申請フローを追加

## 解決内容


## 解決日

