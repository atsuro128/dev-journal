# 099: Step 9 の FE 仕様テストが未実装ページスタブ破損に依存している

## 指摘概要
Step 9 の品質ゲートは「失敗しているテストが未実装による期待どおりの失敗であり、環境不備・初期化漏れ・テスト破損による失敗ではないこと」を要求している。しかし現行の主要 FE ページは単なる `<div>` スタブのままで、対応するページ仕様テストが UI 契約を一切観測できずに壊れている。これは仕様を表現する失敗ではなく、テスト対象コンポーネント自体が置き換わっていないことによる破損であり、品質ゲートを満たさない。

## 根拠
- 品質ゲートは `ai-dev-framework/guide/work-breakdown/step9-test-implementation.md` で「失敗しているテストが『未実装による期待どおりの失敗』であり、環境不備・初期化漏れ・テスト破損による失敗ではないこと」を PASS 条件としている。
- 現行のページ実装は以下の通り、いずれも仕様コンポーネントを組み立てておらず単純文字列を返すだけである。
  - `expense-saas/frontend/src/pages/ReportListPage.tsx:1`
  - `expense-saas/frontend/src/pages/ReportCreatePage.tsx:1`
  - `expense-saas/frontend/src/pages/ReportDetailPage.tsx:1`
  - `expense-saas/frontend/src/pages/ReportEditPage.tsx:1`
- その結果、`npx vitest run` では以下のように、設計された UI 契約ではなくスタブ描画に起因する失敗が発生している。
  - `expense-saas/frontend/src/pages/__tests__/ReportListPage.test.tsx` は `report-list-header` などの要素を取得できず全件失敗
  - `expense-saas/frontend/src/pages/__tests__/ReportCreatePage.test.tsx` は「タイトル」入力や「キャンセル」ボタンを取得できず全件失敗
  - `expense-saas/frontend/src/pages/__tests__/ReportDetailPage.test.tsx` は `report-info-card` や確認ダイアログ関連要素を取得できず全件失敗
  - `expense-saas/frontend/src/pages/__tests__/ReportEditPage.test.tsx` はフォーム要素や遷移条件を確認できず全件失敗
- これらのテストは `expense-saas/frontend/src/pages/__tests__/ReportListPage.test.tsx:3`、`expense-saas/frontend/src/pages/__tests__/ReportCreatePage.test.tsx:3`、`expense-saas/frontend/src/pages/__tests__/ReportDetailPage.test.tsx:3`、`expense-saas/frontend/src/pages/__tests__/ReportEditPage.test.tsx:3` でそれぞれ画面仕様テストとして定義されているため、現状の失敗は「仕様未実装を示す赤テスト」ではなく、テスト対象をスタブのまま放置した破損状態に近い。

## 判定
高。品質ゲート違反、下流作業可能性なし。

## 修正方針案
- Step 8 のスケルトン責務に沿って、少なくともページが設計済み子コンポーネントと Hook を接続する最小構造まで置き換え、テストが DOM 契約を観測できる状態にする。
- 失敗させる場合も、`ErrNotImplemented` 相当のダミー結果や未接続ミューテーションを通じて仕様差分が読めるようにし、単純スタブ描画による「要素が存在しない」失敗から脱却する。
- 修正で閉じるべき指摘。Step 10 着手前に FE 赤テストの意味を回復する必要がある。
