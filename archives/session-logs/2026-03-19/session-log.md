## 21:36 セッション
- 作業: 解決済み issue のクローズ
  - ops-034（マイルストーン表の分離）: progress.md から既に削除済みのため closed に移動
  - ops-035（機能別進捗テーブルの分離）: progress.md から既に削除済みのため closed に移動

## 22:09 セッション
- 作業: issue 028（ダッシュボードの月別支出サマリー欠落）の対応
  - 業界調査を実施（Expensify, Concur, Zoho, freee, マネーフォワード, 楽楽精算等）
  - 調査結果に基づき DASH-005 の設計を業界標準に合わせて修正
    - Member: 月別支出サマリーを表示しない（ステータス確認が主目的）
    - Approver / Accounting / Admin: テナント全体の月別支出サマリーを表示
    - Approver の部下スコープ限定は Phase 3 に先送り（MVP は部門紐づきなし）
  - 修正対象: requirements.md (DASH-005), screens.md (SCR-DASH-001), ui_flow.md, usecases.md (UC-SYS04)
  - issue 028 を closed に移動
- 作業: Accounting の経費申請可否の調査・issue 確認
  - 業界調査の結果、経理担当も経費申請できるのが標準と判明
  - 既存 issue 024 がこの論点をカバー済み（重複起票した 036 は削除）
