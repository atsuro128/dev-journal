# ダッシュボード「承認待ち」カード右上の赤点 dot バッジの意図伝達が不十分（UX 改善）

## 発見日
2026-04-28

## カテゴリ
ui-design / frontend

## 影響度
中（機能影響なし。ユーザーが「謎の赤点」と感じる UX 課題。設計通りの実装だが意図が伝わらない）

## 発見経緯
user-report / Step 11-A SMK-006 後の Approver ダッシュボード目視確認時に、ユーザーが「承認待ちカード右上に謎の赤点がある」と報告

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書側の規定（実装は設計通り）

- `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md:114`: `showBadge` プロパティは「count >= 1 のとき右上に赤点（dot）バッジを表示する。**表示の意図は要対応件数の可視化**であり、`anchorOrigin={{ vertical: 'top', horizontal: 'right' }}` + `aria-label="要対応あり"` で実装する」と規定
- 同 L365-366: 「承認待ちカウントカード」「支払待ちカウントカード」は showBadge 付きと明記
- `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md:110, 120`: 「件数が 1 以上の場合、要対応バッジ（赤色の丸バッジ）を表示する」

### 実装

- `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx:125`（承認待ち）/ `DashboardPage.tsx:140`（支払待ち）で `showBadge={true}` 指定 — 設計通り
- `expense-saas/frontend/src/components/dashboard/CountCard.tsx:64-73` で `<Badge color="error" variant="dot" anchorOrigin={{ vertical: 'top', horizontal: 'right' }} aria-label="要対応あり">` を表示 — 設計通り

### UX 課題

- 設計書では「要対応件数の可視化」が意図だが、画面上には dot のみで意図を示す凡例・tooltip・ラベルが存在しない
- ユーザー視点では「これがバグなのか仕様なのか判別できない」状態
- カードのラベル「承認待ち」自体が既に「要対応」を示しており、追加で dot を出す必然性が薄い
- `aria-label="要対応あり"` はスクリーンリーダー向け。視覚ユーザーには情報が届かない

### 過去経緯

- issue **#085**（ダッシュボード「却下」カード右下の Badge dot が意図不明）が PR #52 で解決済み。却下カードでは「自分の却下件数 = 要対応ではない」として dot を削除した経緯あり
- 承認待ち / 支払待ちカードは「仕様通り」として残置されているが、視覚的伝達不足は #085 と同種の課題

## 影響

- ユーザーが赤点の意味を判別できず、UI バグと誤認する
- 実際にユーザー（指揮役へのレビュー報告として）からも「謎の赤点」として報告された

## 提案

**対応方針（要選択）**:

1. **dot 削除案**: showBadge プロパティ自体を廃止し、Badge を出さない（#085 と整合した方針）。カードラベル「承認待ち」自体が要対応の意味を内包しているため冗長性は最小
2. **意図明示案**: dot を残し、tooltip（hover で「要対応の件数があります」表示）または件数ラベル付き badge（dot ではなく「N件 要対応」のテキストバッジ）に変更
3. **件数強調案**: dot をやめ、件数 0 と件数 ≥1 でカードのアクセントカラー強度を変える等の代替表現

### 推奨

設計書 `55_ui_component/screens/dashboard.md` で `showBadge` 仕様を見直し、上記いずれかに収束させる。
個人的には **案1（削除）** が最小変更で UX 課題が解消する。承認待ちカードのアクセントカラー（`accentColor="info"` の info ボーダー）で要対応性は十分視覚化されている。

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/dashboard/CountCard.tsx` | showBadge プロパティ削除 or 仕様変更 |
| `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx` | 承認待ち / 支払待ちの `showBadge` 指定削除 or 維持 |
| `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` | showBadge 仕様の改訂（案1〜3いずれかへ） |
| `dev-journal/deliverables/docs/50_detail_design/screens/dashboard.md` | §4.2 / §4.3 の「要対応バッジ」記述の改訂 |
| `expense-saas/frontend/src/components/dashboard/__tests__/CountCard.test.tsx` | showBadge 関連テスト更新 |

## ラベル

- type: ux / ui
- area: frontend / docs（設計改訂）

## 関連

- 過去 issue: #085（却下カードの Badge dot 削除、PR #52）
- 関連 SMK: SMK-006（Approver グローバルナビ確認）の副次発見

## MVP 区分

MVP（UX 違和感、ユーザーが直接目にする）

---

## 解決内容

**採用方針**: 案 1（dot 削除）

**設計判断根拠**: 件数がカード本体に既に大きく表示されており、Badge dot は情報的に冗長。要対応性は accentColor (info / warning ボーダー) で十分視覚化されている。バッジが価値を出すのは「情報が隠れている対象（アイコン / メニュー項目）」であり、件数が見えているカウントカードでは冗長になり「謎の赤点」と感じられる。#085（却下バッジ削除）も結果的にこの設計原則に整合していた。

**実装** (PR #108, commit 6916ed0):
- `CountCard.tsx` から `showBadge` プロパティと MUI `Badge` wrapper を削除（import も削除）
- `DashboardPage.tsx` の承認待ち / 支払待ちカードの `showBadge={true}` 指定を削除
- テスト DSH-FE-012 / 012b / 012c / 013 の 4 ケース削除、DSH-FE-014 のコメント整理

**設計書改訂** (commit d3c201a):
- `55_ui_component/screens/dashboard.md`: showBadge プロパティ・要対応バッジ記述を全削除（Props 表 / データフロー / 対応表 / テスト識別子）。コンポーネントツリー図の「ActionCountCard」サブ型呼称も削除（show Badge 削除でサブ型化の意義消失）
- `50_detail_design/screens/dashboard.md`: §4.2/§4.3 の「件数が 1 以上の場合、要対応バッジ（赤色の丸バッジ）を表示する」記述を削除、§9.2/§9.3 ASCII 図の「N 件 ●」末尾の `●` も削除
- `40_basic_design/screens.md`: 3.2 Approver 行の「（要対応バッジ）」記述削除
- `60_test/test_cases/dashboard.md`: DSH-FE-012/012b/012c/013 削除、優先度サマリー表からも削除
- `60_test/traceability.md`: DSH-FE 範囲記述を「001〜039（一部欠番あり、サブID DSH-FE-003b/004b を含む）」に補正

**スコープ外（記録のみ）**: ナビメニュー側のバッジ追加議論は本対応のスコープ外、検討せず。

## 解決日

2026-04-30
