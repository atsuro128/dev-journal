# 引き継ぎメモ

## セッション: 2026-05-02 10:00 〜 2026-05-03 01:43

### ゴール

- Step 11-A クローズ向けに残 SMK 検証 + #160 真因再分析を並行で進める
- 全 PASS なら 6 件 issue を resolved 移動 + Step 11-A 完了化
- このセッションは PR マージまでで切り上げ（SMK 再検証は次セッションへ持越）

### 作業ログ

#### 1. #160 真因再分析の架空仮説 → 実機計測で否定 → 真の真因確定

architect レポート委譲で「flex の min violation 後の再フローで hasScrollX 未発火」仮説を取得。代替案（B: columnVisibilityModel / C: カードビュー / E: 案 B+保険等）を提示したが、ユーザー指摘「テーブル構造を変えず横スクロール発火が本来要件」で方向修正。

実機計測 1 巡目で「Box (`css-d9bexz`) 726px / 親 (`css-k008qs`) 375px / 親 display=flex」を確認 → 「親 flex 内で子 Box の min-width:auto 罠」と仮定 → PR #122 で AppDataGrid Box に minWidth:0 + overflowX:auto 追加。

PR #122 マージ後の SMK-101 再検証で **依然 FAIL**。再計測で `css-d9bexz` の computed style を確認すると `flex-grow: 1` のみ → AppDataGrid Box ではないことが判明。HTML 階層を読み直して **`<main>` 要素そのもの**だったと特定。

真因確定: AppLayout の `<Box component="main">` が `flexGrow: 1` + `min-width: auto` で content (DataGrid 内部) に追従して 726px に膨張していた。PR #124 で AppLayout に minWidth:0 追加。

#### 2. #162 却下ダイアログ regression 再対応 (autoFocus 削除)

SMK-011 #4 検証で「却下理由欄を開いた瞬間からバリデーションエラー」を確認。前回 PR #113 の修正は disabled 解消は OK だが、touched=true が初期から立つ現象が残存。

真因: TextField の `autoFocus` + MUI Dialog FocusTrap の連鎖で blur 発火 → `setTouched(true)`。jsdom では再現できないため自動テストでは検出不可。

ユーザー指摘「autoFocus 切れないの？」で案 D（TransitionProps.onEntered）より案 A（autoFocus 削除）を採用。PR #121 で 1 行削除 + focus spy パターンのテスト追加。codex 指摘で当初の `not.toHaveAttribute('autofocus')` テストが gard として機能しないことが判明 → focus spy パターンに書き換え（autoFocus を戻すと FAIL することを実証済み）。

#### 3. #166 SMK-104/105 用フィクスチャ追加

SMK-104 検証時に「テナント B Approver」「テナント A 第二 Approver」「Approver2 処理レポート」が seed.go に存在しないことを発見。新規 issue #166 起票 → PR #123 で seed.go 追加 + dashboard_handler_test.go の期待値更新（DSH-011/014/015）。

reviewer 1 回目: DSH-011 / DSH-015 漏れ指摘 → 修正 (commit 22e3598)。
reviewer 2 回目: DSH-014 漏れ追加指摘 → 指揮役直接修正 (commit cf48722)。
reviewer 3 回目: PASS。

codex 1 回目: ReportTenantBApprovedID の補完 UPDATE が必要と指摘（既存 DB seed 再実行で approved_by が NULL のまま残る問題） → backend-developer が DO NOTHING + 補完 UPDATE 採用 (commit 1aeb73d)。
codex 2 回目: regression テスト追加要求が未対応と指摘 → backend-developer が seed_test.go に TestSeed_ReportTenantBApproved_BackfillExistingRow 追加 (commit d449603)。
codex 3 回目: PASS。

#### 4. #164 Admin カード幅 再対応

DevTools 視覚確認で「Admin だけ 5 カードが 1 行に詰まっている」を確認。他ロール (Member/Approver/Accounting) は 1 行目 3 カード (1/3 幅) なのに、Admin だけ 1 行 5 カード (1/5 幅 = `md:2.4`) で視覚的に細い。

PR #125 で TenantStatusCards の Grid を `md:2.4` → `md:4` に変更し、5 カード × 1/3 幅 = 3+2 の 2 行レイアウトに統一。reviewer 1 回目で詳細設計書 (50_detail_design/screens/dashboard.md) の改訂漏れ 3 箇所指摘 → 指揮役直接修正 (dev-journal commit 9884c5d)。reviewer 再レビュー / codex 共に PASS。

#### 5. #165 マイレポートフィルタエリア mobile overflow を新規起票

#160 真因調査中の副次発見。ReportListTable のフィルタエリア（ステータス / 開始日 / 終了日）が mobile 幅で横一列のまま画面幅を超える。新規 issue #165 として起票（修正案 A: Stack direction レスポンシブ化、未着手）。

#### 6. PR マージ完了

5 PR マージ完了:
- PR #121 (#162 autoFocus 削除) → master `3e81f077`
- PR #122 (#160 案 F: AppDataGrid Box minWidth:0) → master `1d586e28`
- PR #123 (#166 fixture 追加) → master `b4c577c7`
- PR #124 (#160 案 F': AppLayout main minWidth:0) → master `1b3c587c`
- PR #125 (#164 TenantStatusCards Grid 3+2) → master `80962cd8`

### 未完了

- **ユーザー実施事項（次セッション着手時、必須）**:
  - docker compose リビルド（`docker compose build web && docker compose up -d web`）
  - SMK-101 再検証（5 画面 mobile 375px で横スクロール発火 + ellipsis 解消）
  - DevTools 視覚確認 #164（4 ロール ダッシュボード PC 幅でカード幅統一）
- 全 PASS なら 6 件 issue (#157 / #158 / #160 / #162 / #164 / #166) を `pending-review` or `open` から `resolved/` 移動 → **Step 11-A クローズ**
- issue #165（マイレポートフィルタエリア mobile）対応
- BE: full test 実行（ユーザー方針: マージ後にユーザー実施。PR #123 の seed.go / fixture.go / handler テスト変更の整合性検証）

### ブロッカー

なし（5 PR 全マージ完了、SMK 残検証フェーズ）

### 次にやること

#### 優先度 1: docker compose リビルド + SMK-101 / #164 視覚確認

ユーザー側で docker compose リビルド → 5 画面 mobile 横スクロール発火 + 4 ロール カード幅統一を確認。FAIL なら再対応、PASS なら優先度 2 へ。

#### 優先度 2: BE: full test 実行

ユーザーが VS Code タスク「BE: full test」を実行 → PR #123 の影響で TestListProcessedReports 系 / dashboard handler 系 / tenant handler 系の整合性確認。FAIL があれば即対応。

#### 優先度 3: 6 件 issue resolved 移動 + Step 11-A クローズ

`#157 / #158 / #160 / #162 / #164 / #166` を `resolved/` 移動。progress.md の Step 11-A 状態を「完了」に更新。

#### 優先度 4: issue #165 対応

マイレポートフィルタエリアの mobile レスポンシブ化（Stack direction xs:column → sm:row）。

#### 優先度 5: Step 11-B / 11-C 着手

11-B 横断テスト Go (cross_cutting_test.go / rate_limit_test.go) と 11-C E2E Playwright を並列着手判断。

### 学び・気づき

#### DevTools 計測対象の取り違え（最大の手戻り原因）

実機計測で「Box (css-d9bexz) 726px = AppDataGrid Box ラッパー」と決めつけて PR #122 を実装 → マージ後 SMK-101 再 FAIL → 再計測で `<main>` 要素だったと判明 → PR #124 で外側の真因を修正。**DOM 階層を最初に確認してからクラス名を識別すべき。`MuiBox-root` は MUI で多用されるため、computed style だけでなく HTML タグ (`<main>` / `<div>`) と DOM 親子関係を必ず確認する**。

#### architect レポートが「テーブル構造を変える別解」に偏った

architect 委譲時に「過去 PR が空振りした事実」を強調しすぎたため、案 B (columnVisibilityModel) / 案 C (カードビュー) など「テーブル内横スクロールは諦めて別解」に偏った提案が返ってきた。ユーザー指摘「画面全体としての幅は収まる、テーブル部分だけ横スクロール、これが本来要件」で軌道修正。**架空の代替案を増やす前に、本来の要件を再確認 + 実機計測で真因を実証する流れを優先する**。

#### reviewer の見落としを codex がカバーした

PR #123 の reviewer 内部レビューで DSH-014 を見落とし → 再レビューで指摘。さらに codex で UPSERT / regression テストの 2 段階指摘。**reviewer 単独では網羅性に限界がある。重要な変更（特に SQL や fixture 系）は codex にも投げて二段構えにする**（C 方針の正しさを再確認）。

#### 私の指示漏れ（PR #123 codex フィードバック）

backend-developer への差し戻し指示で codex 前回コメントの「regression テスト追加」要求を拾い損ねた。**指摘対応を再委譲する際は、過去レビューコメント全文を読み直して要求を漏らさず転記する**。

#### TenantStatusCards / MyReportCountCards の責務分割の限界

ユーザー指摘「本来はカード表示はコンポーネント化して、ドメインによって表示するカードを変える作りならこんなこと起こらなかった」。MyReportCountCards / TenantStatusCards という別コンポーネント分割が今回の不整合の温床。MVP では現アーキテクチャ維持、Phase 2 以降のリファクタリングで「共通カードコンテナ + props で表示内容を差別化」する設計を検討（post-MVP）。

#### dev-journal の同期更新を frontend-developer に明示する必要

PR #125 で `55_ui_component/screens/dashboard.md` は更新したが `50_detail_design/screens/dashboard.md` の同種記述を漏らした。**設計書改訂を指示するときは、関連ファイルを grep で全件特定してから漏れなく指示する**。

### 意思決定ログ

#### #160 真因の最終確定（案 F'）

- 真因: AppLayout の `<Box component="main">` が flex item として `min-width: auto` で content 追従して膨張
- 修正: AppLayout.tsx の sx に `minWidth: 0` 追加（PR #124）
- PR #122（AppDataGrid Box の minWidth:0 + overflowX:auto）は内部スクロール発火の「最後の砦」として維持
- 両 PR 揃って初めて mobile 横スクロールが発火する想定（実機検証は次セッション）

#### #162 autoFocus 削除（案 A）採用

- 真因: autoFocus → FocusTrap → blur 連鎖で touched=true
- 案 D（TransitionProps.onEntered + useRef）より案 A（autoFocus 削除）を採用
- 理由: 最小変更、UX デメリット軽微（Approver の意識的操作で TextField を 1 クリックする手間は許容）、v8 transition タイミングのエッジケースリスクなし

#### #166 fixture 補完で別案（DO NOTHING + 補完 UPDATE）採用

- backend-developer 判断: seed.go 全 INSERT が `ON CONFLICT DO NOTHING` を一貫採用しているため、一貫性を保ちつつ INSERT 直後に WHERE 句付き UPDATE で冪等性確保
- 推奨案（ON CONFLICT DO UPDATE）よりシンプルさより一貫性重視

#### #164 TenantStatusCards Grid 3+2 採用

- 真因: 他ロール 1/3 幅基準と整合せず、Admin だけ 1/5 幅で視覚的に細い
- 修正: `md:2.4` → `md:4` で 3+2 の 2 行レイアウトに変更
- カード数の差（5 vs 3）から来る幅不整合を、レイアウト変更で吸収（カード数を減らす案 / カード幅を変える案より自然）

### PR / コミット要約

**expense-saas**（5 PR、全マージ済み）:
- PR #121 (#162 autoFocus 削除、commits c402d65 / 51cac99) → master `3e81f077`
- PR #122 (#160 AppDataGrid minWidth:0、commit 4722b35) → master `1d586e28`
- PR #123 (#166 fixture + 期待値修正、commits a56362f / 22e3598 / cf48722 / 1aeb73d / d449603) → master `b4c577c7`
- PR #124 (#160 AppLayout minWidth:0、commit a8175305) → master `1b3c587c`
- PR #125 (#164 TenantStatusCards Grid 3+2、commit adcad2f) → master `80962cd8`

**dev-journal**（master、複数コミット）:
- 大量の設計書改訂・issue 更新・progress.md 更新を実施
- 直近: `9884c5d` docs(dashboard): #164 再対応の設計書同期更新

**root-project**: 変更なし

**ai-dev-framework**: 変更なし
