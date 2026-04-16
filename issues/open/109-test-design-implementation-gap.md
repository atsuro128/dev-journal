# テスト設計書とテスト実装の乖離（ID 未反映・重複・未実装）

## 発見日
2026-04-16

## カテゴリ
testing

## 影響度
中（テストの traceability が崩れており、カバレッジ管理・回帰テスト追跡が困難）

## 発見経緯
proactive — issue 100 対応中にテスト ID 衝突（ATT-FE-029〜031）が発覚。横断的に調査したところ、設計書と実装の間に広範な乖離が存在することが判明。codex による網羅的な突き合わせ調査を実施（2026-04-16）。

## 関連ステップ
Step 6（テスト設計）/ Step 9（テストコード実装）/ Step 10（機能実装）/ Step 11-A

## ブロッカー
なし（後追い改善事項。ただし Step 11-D 横断レビュー前に整理が望ましい）

## 問題

issue 修正（PR #54〜#58 等）でテストケースが追加・変更された際、テスト設計書（`test_cases/*.md`）への反映が行われていなかった。codex 調査により以下の 3 カテゴリの乖離が確認された。

### 1. 実装にあるが設計書にないテストケース（22 ID / 34 行）

issue 修正で追加されたテストが設計書に未反映:

| ID 群 | 由来 | 内容 |
|---|---|---|
| APR-FE-001〜004 | issue 106 / PR #54 | ApprovalListPage ロールチェック + 403 |
| PAY-FE-001〜005 | issue 106 / PR #54 | PaymentListPage ロールチェック + 403 |
| ATT-FE-007b, 035b, 042b | 添付系 issue | AttachmentList/useAttachmentDownload/useDeleteAttachment |
| DSH-FE-003b, 004b, 012b, 012c | ダッシュボード修正 | DashboardPage Grid / CountCard Badge |
| ITM-FE-091-A/B, 092-A/B | issue 098 / PR #56 | ItemSlidePanel サブテスト |
| ITM-FE-098-1〜8 | issue 098 / PR #56 | ItemForm/ItemSlidePanel/ReportDetailPage |
| PRT-001〜003 | Step 10-A | PrivateRoute テスト（設計書にカテゴリ自体がない） |
| RPT-FE-090-A | issue 修正 | ReportDetailPage 明細行クリック |

### 2. 設計書にあるが実装にないテストケース（99 件）

| 区分 | 件数 | 状態 |
|---|---|---|
| CRS-001〜088（横断テスト・E2E） | 83 件 | Step 11-B/C スコープ。**未着手で想定通り** |
| RPT-FE-011〜014（レポートフィルタ） | 4 件 | 実装漏れの可能性 |
| WFL-FE-011/012/014/015/020/021/040/041/043/044/049/050 | 12 件 | 実装漏れの可能性 |
| ATT-FE-051〜053 | 3 件 | PR #58 に実装済み（マージ前） |

### 3. ID 重複・衝突（14 ID）

同一 ID が異なるテストファイルで別のテストに使用されている:

| ID | 衝突ファイル 1 | 衝突ファイル 2 |
|---|---|---|
| ATT-FE-007 | AttachmentArea.test.tsx（削除ダイアログ表示） | AttachmentList.test.tsx（ファイル名表示） |
| ATT-FE-008 | AttachmentArea.test.tsx（キャンセル時API未呼出） | AttachmentList.test.tsx（空状態メッセージ） |
| ATT-FE-009 | AttachmentArea.test.tsx（削除API呼出） | AttachmentList.test.tsx（3件表示） |
| DSH-FE-008 | CountCard.test.tsx（label/count表示） | DashboardPage.test.tsx（トースト表示） |
| DSH-FE-020 | TenantStatusCards.test.tsx（2箇所で重複） | — |
| ITM-FE-091 | ItemSlidePanel.test.tsx（-A/-Bサブテスト） | — |
| ITM-FE-092 | ItemSlidePanel.test.tsx（-A/-Bサブテスト） | — |
| ITM-FE-098 | ItemForm.test.tsx（複数サブテスト） | ItemSlidePanel.test.tsx / ReportDetailPage.test.tsx |
| RPT-FE-090 | OwnerActions.test.tsx（再申請ボタン） | ReportDetailPage.test.tsx（明細行クリック） |
| TNT-FE-008 | TenantInfoCard.test.tsx（会社名表示） | TenantPage.test.tsx（ロール不一致リダイレクト） |
| TNT-FE-009 | TenantInfoCard.test.tsx（スケルトン） | TenantPage.test.tsx（403リダイレクト） |
| TNT-FE-024 | AllReportsFilterBar.test.tsx（フィルタ描画） | AllReportsPage.test.tsx（ロール不一致リダイレクト） |
| TNT-FE-025 | AllReportsFilterBar.test.tsx（ステータス選択） | AllReportsPage.test.tsx（403リダイレクト） |
| WFL-014 | report_handler_test.go（Submit NoApprover） | workflow_handler_test.go（Approve Draft） |

## 影響

- テスト traceability が崩壊（設計書を見てもどのテストが実装済みかわからない）
- ID 重複によりテスト結果の追跡が不正確
- Step 11-D 横断レビューで「テスト網羅性」を判定する際の基準資料が信頼できない

## 提案

### 優先度順の対応

1. **セクション 3（ID 重複解消）** — 最優先。衝突している 14 ID を一意に振り直す（設計書 + 実装の両方を更新）
2. **セクション 1（設計書への反映）** — 22 ID を対応する test_cases/*.md に追記
3. **セクション 2（FE フィルタ 16 件）** — RPT-FE-011〜014 と WFL-FE-011〜050 の実装漏れを確認し、必要なら実装追加

### 対応方針
- セクション 3 と 1 は designer（設計書更新）+ 指揮役（実装側 ID 修正）の協調作業
- セクション 2 は test-implementer で実装（設計書に記載済みなので設計書更新不要）
- CRS-001〜088 は Step 11-B/C で対応するため本 issue のスコープ外

### 再発防止
- issue 修正でテストを追加した際は、テスト設計書への反映を PR チェックリストに含める
- workflow.md またはスキルテンプレートに「テスト追加時は test_cases/*.md を更新」を追記

## 完了条件

- 14 ID の重複が全て解消されている（設計書・実装ともに一意）
- 22 ID（実装済み・設計書未反映）が test_cases/*.md に追記されている
- RPT-FE-011〜014 / WFL-FE-011〜050 の実装状況が確認され、漏れがあれば実装されている
- 再発防止策が workflow.md または該当スキルに反映されている

## 関連
- 100: ATT-FE-029〜031 の ID 衝突（本 issue の発端）
- 104: UI カバレッジ監査（テスト網羅性の別軸。本 issue は ID 管理の問題）

---

## 解決内容

## 解決日
