# Accounting ダッシュボードの表示内容が上流定義と不整合

## 発見日
2026-03-19

## 発見経緯
codex レビュー（review-findings/open/034-step4-accounting-dashboard-mismatch.md）

## 関連ステップ
Step 4（screens.md, ui_flow.md）。024 の解決方針に依存。

## カテゴリ
basic-design

## 問題
上流の usecases.md（UC-SYS04）と requirements.md（DASH-003）では Accounting のダッシュボードに「Member 向け情報 + 支払待ち件数」を表示する定義だが、screens.md では「Member と同じ件数情報は非表示。支払待ちレポート数を表示」と定義しており不整合。

024（Accounting の申請権限）が解決されて Accounting がレポート作成可能になる場合、上流定義が正しくなる（自分のレポートの状態を確認する必要がある）。

## 影響
上流成果物と基本設計の間で Accounting のダッシュボード仕様が矛盾している。

## 提案
024 の解決後に screens.md / ui_flow.md を上流定義に合わせて修正する。
- Accounting のダッシュボードに Member 相当の件数情報を追加
- 支払待ちレポート数も引き続き表示

## 解決内容

issue 024（Accounting 申請権限付与）の修正過程で解消。screens.md の Accounting ダッシュボード行に Member 向け情報（下書きレポート数、提出中レポート数、却下レポート数、直近レポート一覧）を追加し、上流定義（UC-SYS04, DASH-003）と整合させた。ui_flow.md も同様に修正済み。

## 解決日
2026-03-20
