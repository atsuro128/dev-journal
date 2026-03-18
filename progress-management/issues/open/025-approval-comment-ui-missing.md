# 承認コメントの入力UIが基本設計に存在しない

## 発見日
2026-03-19

## 発見経緯
codex レビュー（review-findings/open/033-step4-approval-comment-ui-missing.md）

## 関連ステップ
Step 4（screens.md, ui_flow.md）

## カテゴリ
basic-design

## 問題
上流の usecases.md（UC-A02 代替系）、requirements.md（WFL-F01: 承認コメント任意、0〜1000文字）、state_machine.md（approval_comment）で承認時の任意コメント入力が定義されているが、screens.md の承認ダイアログには「このレポートを承認しますか？」の確認のみでコメント入力欄が存在しない。

市場調査でも全8製品が承認時のコメント入力機能を提供している。

## 影響
Step 5 以降で approval_comment を扱う画面仕様に落とし込めない。

## 提案
1. screens.md の確認ダイアログ（4.6節）の承認行に「承認コメント入力フォーム付き（任意、0〜1000文字）」を追記
2. ui_flow.md の Approver フローに「承認コメント入力（任意）」を反映

## 解決内容


## 解決日

