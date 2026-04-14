# ダッシュボード「却下」カード右下の Badge dot が意図不明（CountCard showBadge 矛盾の可能性）

## 発見日
2026-04-13

## カテゴリ
implementation / frontend / ui

## 影響度
中

## 発見経緯
Step 11-A ローカル動作確認（SMK-001〜003）で、Member / Approver / Accounting ロールのダッシュボードを確認した際、「却下」カードの右下に意図不明の赤い小さな点（MUI Badge dot）が表示されていることを観察

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（視覚的違和感のみ）

## 概要

ダッシュボードの「却下」カード右下に、意図不明の赤い小さな点（`MuiBadge-colorError MuiBadge-anchorOriginTopRight` クラス付き）が表示される。クラス名は TopRight だが視覚上は右下に見える。`CountCard` の `showBadge` 設計と矛盾している可能性がある。

## 事実

- 観察箇所: Member / Approver / Accounting ダッシュボードの「却下」カード
- DOM: `MuiBadge-colorError MuiBadge-anchorOriginTopRight`（推定）
- 観察: 赤い dot が右下に表示されている
- 関連設計: `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` の CountCard `showBadge` プロパティ（却下件数 > 0 のときに表示する想定か要確認）

## 要確認事項

1. CountCard の `showBadge` が設計上どの条件で表示されるべきか
2. `anchorOriginTopRight` クラスが付いているのに視覚上右下に見えるのは、親要素のレイアウトに起因するか、transform 指定の問題か
3. 他の Member 系カード（下書き / 提出中）にも同様の Badge が付く条件があるのか整合性確認

## 修正対象ファイル（推定）

- `expense-saas/frontend/src/components/dashboard/CountCard.tsx`（または同等のコンポーネント）
- `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md`（設計と実装の整合性確認）

## 完了条件

- 却下カードの Badge dot 表示条件が設計書と実装で一致している
- 視覚位置（TopRight / BottomRight）の意図が明確化されている
- 意図しない点が表示される場合は修正される
