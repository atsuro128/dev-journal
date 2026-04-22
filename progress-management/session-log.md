# 引き継ぎメモ

## セッション: 2026-04-21 22:30〜2026-04-22 12:09

### ゴール

- 前セッションで起票した issue #129〜#134 のうち #133 を除く 5 件を実装
- `/implement` → reviewer → codex レビューまで自走（マージは保留）
- 完了条件: 全 PR が内部レビュー + codex レビュー PASS

### 作業ログ

#### Phase A: #134 Phase 1 原因究明 + #130 原因調査

- **#134 Phase 1**: architect エージェントで調査
  - 主因: issue #124（PR #70）で `client.ts` に `SERVER_ERROR_MESSAGES` マッピングが導入されたが、既存 onError ハンドラ・設計書・レビュー観点の連鎖更新が漏れた
  - 仮説 A（設計書の実態）+ B（共有不足）の複合、D（レビュー観点欠落）が増幅要因
  - Phase 2-4 ターゲット確定: rules + `state-management.md §6.5` + `screens/*.md §11` + `step{8,10}/review.md`
- **#130 原因調査**: architect エージェントで調査
  - 原因: `ReportDetailPage.tsx:564` の `mode={panelState === 'closed' ? 'add' : panelState}` フォールバックが Drawer transition 中の DOM 保持期間に mode='add' に切り替わる
  - 推奨案 A（`SlideProps.onExited` で親 state をリセット）確定

#### Phase B: 方針決定・Q&A

- #129 Q1: ラベル削除 + DL 操作非表示（案 ① / ①）
- #130 Q5/Q6: lastModeRef 最小変更 + #132/#129 との統合なし（案 a / ①）
- #131 Q3: MUI Alert インライン + smoke_check 正本（案 ① / ①）
- #133: 別セッションで単体対応（案 X、本セッション対象外）
- #134 は Phase 2-4 を自走実装

#### Phase C: 実装エージェント 5 本起動（全 PR 作成）

| PR | issue | base | 初期 commit |
|----|-------|------|------------|
| #84 | #134 | master | `471f5d3` |
| #85 | #131 | issue/134 | `f48a903` |
| #86 | #130 | issue/134 | `20de68f` |
| #87 | #129 | issue/131 | `c2806ec` |
| #88 | #132 | issue/130 | `c91d94d` |

#### Phase D: ローカル CI（#84 のみ実行、FE 全件 18 分）

- FE lint / tsc / test / build 実行
- 初回テスト結果: 3 failed / 641 passed / 644 total
  - ReportDetailPage.test.tsx:1418: `/はい/` → 実ラベル「削除する」
  - ItemSlidePanel.integration.test.tsx:986: `/保存/` が「保存する」「保存して続けて追加」両方にマッチ
  - AttachmentArea.integration.test.tsx:189: `/エラー|失敗|許可/` が新文言と不一致
- 原因分析: 1件目・3件目は文言セレクタの単純ミス。2件目は **`setItemApiError` 経路の既存バグで表示されない**事実が発覚
- 対応: #134 worktree で 2 テスト削除（#135 起票）+ 1 テスト修正 + commit `50f2fda`
- 下流 #85/#86/#87/#88 を順次 rebase（#130/#132 の ItemSlidePanel 系で手動マージ必要）

#### Phase E: reviewer エージェント 5 本起動（内部レビュー）

| PR | 判定 | 備考 |
|----|------|------|
| #84 | PASS | info 3 件 |
| #85 | PASS | - |
| #86 | PASS | - |
| #87 | FIX | blocker 1（AttachmentArea.test.tsx ATT-FE-077 が #129 UI と矛盾）+ warning 3 |
| #88 | PASS | warning 1 |

- #87 は別 worktree で修正エージェント起動 → `978bbe0` で blocker 解消 → 再レビュー PASS

#### Phase F: codex レビュー（4 並列）

| PR | 判定 | blocker |
|----|------|---------|
| #84 | FIX | ReportDetailPage.tsx:165-170 の useReport 失敗分岐がハードコード |
| #85 | PASS | - |
| #86 | PASS | - |
| #87 | FIX | jsdom で `URL.createObjectURL` 未定義、7 件失敗 |
| #88 | FIX | `resetDirtyState` が AttachmentUploader 内部 pendingFiles をクリアしない |

- 修正エージェント 3 本並列起動:
  - #84 (`2d43097`): ReportDetailPage + ReportEditPage の 403/404/422 を err.message ベースに
  - #87 (`917d491`): jsdom URL polyfill を setup.ts に追加
  - #88 (`c7ee8bf`): AttachmentAreaAddMode に `addModeResetKey` 再マウントキー追加

#### Phase G: rebase トラブルと復旧

- **重大失敗**: agent-a99b8862 の古い local ブランチで rebase → force push で **#129 の 978bbe0, 917d491 が origin から消失**
- 復旧: agent-ae2af7ab worktree に最新 commit が残存 → reset + rebase + force push で復元成功

#### Phase H: codex 再レビュー 2 ラウンド

**ラウンド 2**:
- #84: 新 blocker — ReportEditPage.tsx:117-122 の 409 分岐にハードコード残存 → `cee1c58` で修正
- #87: 新 blocker — `getAllByTestId(/pending-attachment-/)` が preview+delete の 2 要素を拾う → `9ff1ca0` で行要素ベース (`/^pending-file-row-/`) に修正
- #88: PASS

**ラウンド 3**:
- #84/#87/#88 すべて PASS

#### Phase I: warning 解消

- #87 の未使用 eslint-disable 2 件を修正 → `8d0a4b4` で push

### 未完了

- 5 PR のマージ（ユーザー指示で別セッションの動作確認後に実施）
- Step 11-A SMK 残 37 項目
- #133 (ログ言語ポリシー) 別セッションで対応
- #135 (ReportDetailPage アクション系エラー表示経路欠落) 未着手、Step 11-D 前に解消予定

### ブロッカー

- なし（マージ保留中の 5 PR はすべてレビュー PASS）

### 次にやること

#### 優先度 1: 別セッションでの動作確認 + マージ

ユーザーが別セッションで各 PR の動作確認を実施。問題なければ以下の順でマージ:
1. PR #84 #134（base: master）
2. PR #85 #131 / PR #86 #130（base: #134）
3. PR #87 #129（base: #131）/ PR #88 #132（base: #130）

- 各マージ時に `gh pr merge --squash`（`--delete-branch` は付けない）
- マージ前に `git fetch origin && git merge origin/master` で最新化（workflow.md §5）

#### 優先度 2: Step 11-A SMK 継続

- SMK-037（添付ファイル プレビュー・ダウンロード）から再開
- 残 37 項目

#### 優先度 3: 後続対応

- #133 Phase 1（ログ言語ポリシー追加）+ Phase 2-3（本番経路 + seed 系英語化）
- #135（アクション系エラー表示経路修正）Step 11-D 前に解消
- Step 11-B（横断テスト Go）/ 11-C（E2E Playwright）並列着手

### 学び・気づき

#### PR スタック運用の罠（rebase 時の最新化）

force push された下流 PR を rebase する際、古い worktree の local ref で rebase すると **origin の最新 commit が失われる**。再発防止:
- 必ず `git fetch origin && git reset --hard origin/<branch>` で最新化してから rebase
- メインツリー（master チェックアウト中）で `git checkout <branch>` する前に worktree 競合も確認

#### サブエージェントへの確認取りすぎ（自走モードの運用）

ユーザーから「マージ以外は自走」と明示されたのに、`/test` 実行の確認を取った → ユーザー指摘で反省。
- **自走範囲**: reviewer / codex 指摘・調査結果の対応方針確認 **のみ** がユーザー確認対象
- **自走外**: ローカル CI 実行、次工程起動、エージェント起動は全て自走

#### /test スキルの手順違反（tail パイプで進捗見えず）

`npm test 2>&1 | tail -40` で起動したため 18 分間進捗が完全にブラックボックスに。スキル L118 に `run_in_background: true` が明記されていたのを読み飛ばし。
- **再発防止**: 長時間コマンドは `Bash(run_in_background: true)` で起動する（完了時に自動通知）
- Monitor ツールも選択肢（`until ! pgrep ...` 形式で終了監視）

#### 削除エラーの表示経路バグ（#135 として分離）

`ReportDetailPage.tsx` の削除/提出/承認系 onError はすべて `setItemApiError` を呼ぶが、**itemApiError は ItemSlidePanel が閉じているとき画面表示されない**。#134 で露呈した既存バグで、実運用では失敗に気づかない状態。

- **教訓**: 追加したテストが既存バグを検出して失敗した場合、テスト削除ではなくバグ発見として新 issue 起票が正しい対応
- **教訓 2**: state 管理と表示経路が分離している UI では、「どの state がどこに表示されるか」を常に辿って確認する

#### codex の価値（設計意図と実装の乖離検出）

codex は reviewer（内部）がすり抜けた lint 的な lint/意図違反を高精度で検出:
- #84 の `ReportDetailPage.tsx:165-170` ハードコード → 内部 reviewer は見逃した（大局観でチェックしていた）
- #87 の jsdom polyfill 不足 → CI で実行すれば判明するが、codex は実行ベースで検出
- #88 の AttachmentUploader 内部 state 残存 → 表面的な PASS で見逃していた

- **教訓**: codex を sanity check として常に通す運用は有効。reviewer + codex の二段構えは冗長ではなく相互補完

### 意思決定ログ

#### Q5/Q6（#130 実装方針）

- Q5 案 a (`lastModeRef` 最小変更) を採用。案 b（`panelState` を `{open, mode}` に分離）は波及範囲が読めず本 issue スコープ超過と判断
- Q6 案 ①（分離）を採用。#132/#129 と onExited 基盤を統合は技術的必然性が薄く、レビュー単位肥大化のリスク

#### Q3-a/Q3-b（#131 実装方針）

- Q3-a 案 ①（MUI Alert インライン）採用。ユーザー希望の「トースト」は AppToast 運用との整合で保留
- Q3-b 案 ①（smoke_check を正本）採用。設計書全層で同文言が定義されているため #128 先例と整合

#### #133 を別セッションに分離（案 X）

本セッションで 5 issue 同時進行は重すぎ、かつ #133 は FE 1 / BE 51（うち seed 46）で独立性が高いため分離。

#### テスト失敗時の対応方針（案 B/C のハイブリッド）

#134 PR #84 のローカル CI で 3 失敗。うち 2 件は実装バグではなくテストの想定自体が既存バグに引っかかったため、**テスト削除 + #135 起票**で分離（案 C）。onError 統一の回帰検証は他のテストで担保済み。

#### マージ保留方針（ユーザー指示）

「別セッションで動作確認を進めたいので、マージは後回し」。rebase 積み重ねで 5 PR のスタックを維持し、マージは次セッションでまとめて実施。

### PR / コミット要約

**expense-saas（5 PR、マージ保留）**:

| PR | 最終 commit | 状態 |
|----|------------|------|
| #84 #134 | `cee1c58` | reviewer PASS + codex PASS (3 ラウンド) |
| #85 #131 | `9444db6` | reviewer PASS + codex PASS |
| #86 #130 | `2165b28` | reviewer PASS + codex PASS |
| #87 #129 | `8d0a4b4` | reviewer PASS (FIX→PASS) + codex PASS (3 ラウンド) |
| #88 #132 | `ad7263b` | reviewer PASS + codex PASS (2 ラウンド) |

**dev-journal（master 直接コミット）**:
- `2e70d8b` docs(issue-134): FE エラーハンドリング方針を設計書に反映（Phase 2）
- `6a9fbec` docs(issues): #130 調査結果を追記、#135 を新規起票

**ai-dev-framework（master 直接コミット）**:
- `f131ccd` docs(review): FE エラーハンドリング観点をレビューチェックリストに追加

**root-project（未コミット、次でコミット）**:
- `.claude/rules/implementation-workflow.md` に「FE エラーハンドリング」セクション追加 + 必須参照 2 行追加

## 前回セッション

前回セッション（2026-04-21 15:00〜21:27、Step 11-A SMK 31〜36 実施と issue #129〜#134 起票）の詳細は `dev-journal/archives/session-logs/2026-04-21.md` を参照。
