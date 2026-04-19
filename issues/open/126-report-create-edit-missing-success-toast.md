# レポート作成・編集画面で成功トーストが表示されない（実装漏れ）

## 発見日
2026-04-20

## カテゴリ
implementation / frontend / ui / design-consistency

## 影響度
低〜中（機能動作には影響なし。成功フィードバックがなく、操作完了の視認性が劣化）

## 発見経緯

Step 11-A ローカル動作確認 SMK-029（エラートースト/フォームエラーの自動消去）の成功トースト側を検証するため、Member でレポート新規作成を実施した際、正常に作成・詳細画面へ遷移したにも関わらず **成功トーストが一切表示されない** ことを観察。

比較として:
- 明細追加（useCreateItem）: 成功トースト表示あり
- レポート提出（useSubmitReport + ReportDetailPage）: 成功トースト表示あり

→ アプリ内で成功トーストの実装パターン自体は存在するが、**レポート作成・編集画面のみで適用漏れ**。

## 関連ステップ
Step 5（詳細設計: state-management.md）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計

`dev-journal/deliverables/docs/55_ui_component/state-management.md:579`:

> | 成功メッセージ | 「レポートを作成しました」「承認しました」 | 各 Hook の `onSuccess` 内の定数オブジェクト |

設計上、成功操作には Snackbar で日本語成功メッセージを表示することが規定されている。例示として「レポートを作成しました」「承認しました」が挙げられている。

### 実装

**レポート作成**: `expense-saas/frontend/src/pages/reports/ReportCreatePage.tsx:60-62`

```tsx
onSuccess: (result) => {
  navigate(`/reports/${result.id}`);
},
```

→ navigate のみ。成功トースト表示なし。

**レポート編集**: `expense-saas/frontend/src/pages/reports/ReportEditPage.tsx:103-105`

```tsx
onSuccess: () => {
  navigate(`/reports/${id}`);
},
```

→ 同じく navigate のみ。成功トースト表示なし。

**対照: Hook レベル**: `expense-saas/frontend/src/hooks/useReports.ts`

`useCreateReport` / `useUpdateReport` / `useSubmitReport` / `useDeleteReport` の `onSuccess` は全て **キャッシュ invalidate のみ**。Hook レベルでは成功トーストを表示していない（ページ側で個別表示する設計）。

### 観察された挙動

| 操作 | 実装 | 成功トースト | 備考 |
|------|------|------------|------|
| レポート作成 | ReportCreatePage | **なし** | 本 issue で起票 |
| レポート編集 | ReportEditPage | **なし** | 本 issue で起票（未検証だが onSuccess コード上なし） |
| 明細追加 | （Item 系ページ） | あり | 設計通り |
| レポート提出 | ReportDetailPage | あり | 設計通り |

## 修正方針

### ReportCreatePage

```tsx
// NG（現状）
onSuccess: (result) => {
  navigate(`/reports/${result.id}`);
},

// OK（修正案）
onSuccess: (result) => {
  showSuccessToast('レポートを作成しました');  // SnackbarContext or Body 系のパターンに従う
  navigate(`/reports/${result.id}`);
},
```

トースト表示の具体的パターンは、既存で成功トーストが動作している画面（明細追加・レポート提出）のコードを参照し、同じ仕組みを流用する。SnackbarContext / `showSnackbar` / BodyToast 等、プロジェクトで採用されているパターンに統一。

### ReportEditPage

同様に onSuccess に成功トーストを追加。文言は「レポートを更新しました」または「保存しました」（既存画面の文言と整合を取る）。

### 再申請フロー（`?ref=:id` 付きの ReportCreatePage）

再申請も同じ画面を使うため、文言を「レポートを作成しました」と汎用的にする or 「新しいレポートを作成しました」のように区別する判断が必要。設計書を参照。

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ReportCreatePage.tsx`
- `expense-saas/frontend/src/pages/reports/ReportEditPage.tsx`
- 各ページのテスト（`__tests__/ReportCreatePage.test.tsx`, `__tests__/ReportEditPage.test.tsx`）
  - 「成功時にトーストが表示される」観点を追加

## 対応条件

- レポート作成成功時に「レポートを作成しました」等の成功トーストが表示される
- レポート編集成功時に同等の成功トーストが表示される
- 既存の成功トーストパターン（明細追加・レポート提出等）と表示方法・デザインが一貫している
- トーストは state-management.md §6.4 の Snackbar 仕様（自動消去・severity: 'success' 等）に従う
- テストで「成功時にトーストが表示される」が検証される

## 監査の推奨

本 issue では検証できていないが、**他の mutation 系 Hook 利用ページでも同様の漏れが残っている可能性**がある。代表的な候補:

- レポート削除（`useDeleteReport` 利用ページ）
- 承認・却下（Approver 画面）
- 支払完了（Accounting 画面）
- 明細編集・削除
- 添付ファイル削除

本 issue の対応時に合わせて上記の成功トースト有無を確認し、漏れがあれば同一 issue に追加 or 別 issue 起票を判断する。

## 関連

- state-management.md §6.4（SnackbarContext）/ §8（ハードコード文字列一覧）: 設計の正本
- issue 124: API クライアントで SERVER_ERROR_MESSAGES マッピングが未適用（エラー側の兄弟 issue）
