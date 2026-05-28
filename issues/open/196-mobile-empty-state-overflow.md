# マイレポート画面で 0 件時のテーブル空状態表示が見切れる（post-MVP）

## 発見日
2026-05-28

## カテゴリ
ux / responsive

## 影響度
低（業務継続可能、見切れた状態でも操作不能ではない）

## 発見経緯
UAT-044（スマホでの利用感）

## 関連ステップ
Step 11-F UAT / 関連設計: `dev-journal/deliverables/docs/50_detail_design/screens/report-list.md` / `dev-journal/deliverables/docs/55_ui_component/common-components.md` AppDataGrid (`noRowsOverlay`)

## ブロッカー
なし

## 関連 issue

- **#195**: スマホ幅でレポート詳細の明細一覧テーブルの列幅が縦書き化（同じく UAT-044 で発見、別現象として分離追跡）
- **#167**（post-MVP / 起票のみ）: DataGrid 列の自動幅 minWidth と手動リサイズ minWidth の分離
  - → 本 issue とは異なる原因（noRowsOverlay 系）だが、AppDataGrid 共通コンポーネントの修正単位として**対応時に参照**
- **#165**（resolved）: マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える（PR #135-137 で対応済み）

## 問題

### マイレポート画面で 0 件時のテーブル内空状態メッセージが見切れる

**発生条件**:
- スマホ幅でマイレポート画面（report-list）を表示
- 一覧に**レポートが 0 件**の状態（例: test-member-empty@example.com でログイン、または UAT-009 で全削除後）

**現象**:
- 空状態のメッセージ（「データがありません」「No data」「申請するレポートがありません」等）が**テーブル枠内に収まらず見切れる**
- スマホ幅でメッセージが画面外に出る or 折り返されない

**期待挙動**:
- 空状態メッセージは画面幅に応じて折り返し or 中央配置で完全に表示されるべき
- AppDataGrid の `noRowsOverlay` 等の機構が機能していない可能性

## 影響

- 初回利用者（レポート 0 件）の最初の体験で表示崩れが発生
- 業務継続には支障なし（「新規作成」導線は別途存在する想定）
- post-MVP の UX 改善として優先度低

## 提案

- AppDataGrid の `slots.noRowsOverlay` の実装を確認、スマホ幅で枠内に収まるよう調整
- 空状態メッセージのコンテナを `max-width: 100%` + `text-align: center` で中央配置
- 文言自体もスマホ幅で 2 行折り返し前提に設計

## 着手タイミング

**post-MVP**。MVP リリース後の UX 改善として対応。**#195 と同時期に AppDataGrid 共通コンポーネントの修正で着手すれば効率的**。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
