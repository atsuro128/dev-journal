# smoke_check.md SMK-026 期待文言が state-management.md と不整合（404 Not Found）

## 発見日
2026-04-19

## カテゴリ
docs / consistency / test-specification

## 影響度
低（docs 整合性の問題。実装は正本（state-management.md）準拠で動作済み）

## 発見経緯
Step 11-A ローカル動作確認 SMK-026（404 存在しないリソース）の検証中、実装が表示したメッセージ「指定されたデータが見つかりません。」が smoke_check.md の期待文言「お探しのレポートが見つかりません」と一致しないことを観察。実装は `state-management.md` の `RESOURCE_NOT_FOUND` マッピングおよび `report-detail.md §11` の `EmptyState` 設計に準拠しており、smoke_check.md 側が古い文言のまま取り残されていた。

同種の不整合は SMK-024（401 セッション切れ）で発覚済み（issue 122 に内包）。smoke_check.md に他の同類不整合が残っている可能性がある。

## 関連ステップ
Step 6-D（テスト設計: smoke_check.md 作成）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書間の不整合

**state-management.md（正本）**

- `55_ui_component/state-management.md:464` — `SERVER_ERROR_MESSAGES.RESOURCE_NOT_FOUND: '指定されたデータが見つかりません。'`
- `55_ui_component/state-management.md:520` — エラーメッセージ一覧で同上を再掲
- `55_ui_component/state-management.md:378` — 404 は「404 画面 or Snackbar」として `EmptyState` 対応

**report-detail.md（正本、画面別表現）**

- `55_ui_component/screens/report-detail.md:766` — `EmptyState` message: `'指定されたデータが見つかりません。'`, action: `{ label: 'レポート一覧に戻る', onClick: navigate('/reports') }`
- `55_ui_component/screens/report-detail.md:782` — 「メインコンテンツ領域に『指定されたデータが見つかりません。』メッセージとレポート一覧へのリンクを表示」

**smoke_check.md（不整合側）**

- `60_test/manual_checklists/smoke_check.md` SMK-026 期待結果:
  > 「お探しのレポートが見つかりません」と画面内エラー表示。トップへの戻り導線あり

→ メッセージ文言・戻り導線ラベル・遷移先の全てが乖離している。

### 実装確認

SMK-026 実施時の観察:
- 表示メッセージ: 「指定されたデータが見つかりません。」（state-management.md 準拠）
- 戻り導線: 「レポート一覧」リンク（report-detail.md §11 準拠）
- 画面内 EmptyState 表示、他コンテンツ非表示（設計準拠）

→ 実装は正本準拠で動作。不整合は smoke_check.md 単独。

## 修正方針

### 1. SMK-026 の期待結果を正本に揃える

```
- 「お探しのレポートが見つかりません」と画面内エラー表示。トップへの戻り導線あり
+ 画面内に EmptyState で「指定されたデータが見つかりません。」と表示、「レポート一覧に戻る」リンクが表示される
```

### 2. smoke_check.md 全体の期待文言監査

SMK-024（issue 122 で対応予定）、SMK-026（本 issue）以外にも state-management.md / 画面設計書と文言が乖離する項目が残っている可能性がある。以下の観点で全文突合する:

- 4.3 エラー表示（SMK-020〜029）: 全 SERVER_ERROR_MESSAGES との突合
- 4.6 日本語 UI（SMK-050〜053）: ラベル・ボタン・トースト・状態ラベルの正本突合
- その他、設計書参照を含む期待結果

監査中に追加の不整合を発見した場合は、**本 issue に追記せず都度単独 issue として起票**する（11-A 運用ルール）。

## 修正対象ファイル

- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`
  - SMK-026 期待結果の修正
  - 全体の文言監査と必要に応じた修正（追加の不整合は別 issue 起票）

## 対応条件

- SMK-026 の期待結果が state-management.md / report-detail.md と一致
- smoke_check.md 全体の期待文言が正本設計書と整合（追加発見分は別 issue）

## 関連

- SMK-024 由来の同種不整合は issue 122 で記載済み（smoke_check.md SMK-024 の期待結果修正を含む）
- 将来同種の不整合を追加発見した場合の起票方針: 本 issue に追記せず都度単独 issue
