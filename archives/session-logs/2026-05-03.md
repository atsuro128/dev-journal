# 引き継ぎメモ

## セッション: 2026-05-03 09:30 〜 2026-05-04 12:30

### ゴール

- Step 11-A クローズ向けに前セッションマージ済 PR 群の SMK 視覚再検証
- SMK FAIL 残項目への追加対応（PR 起票 → マージ）
- 全 PASS 確認後に 6 + 2 件 issue を resolved 移動 + Step 11-A 完了化

### 作業ログ

#### 1. SMK-101 全 5 画面 mobile 検証 → 5 画面とも FAIL 報告

ユーザー実機検証で「マイレポート / 全レポート / 承認待ち / 支払待ち / 処理済み 5 画面で合計金額・ステータス系列が ellipsis 見切れ」を確認。PR #122/#124 で横スクロール発火は OK だが column の minWidth が小さすぎる真因が確定。

#### 2. PR #126: 5 画面 columnDef minWidth 引き上げ

合計金額 100→130 / ステータス 100→120 / 処理結果 100→120 / 現在ステータス 130→140 を 5 画面に反映。frontend-developer (worktree) で実装、reviewer + codex 共に APPROVE 相当、マージ完了 (`3a3719e`)。

#### 3. PR #127: CountCard 高さズレ解消（#168）

Member / Admin ダッシュボードで「default カードと色付き border カードの高さに 3px ズレ」を発見 → issue #168 起票 + PR #127。`borderTop: '3px solid'` 常時 + `borderColor: 'transparent' or accentColor.main` の修正。reviewer + codex APPROVE、マージ完了 (`df3e8bf`)。

#### 4. issue 対応のチケット押し上げ運用誤りを発見 → 5 件削除

ユーザー指摘で `progress-management/tickets/step11/11-A-issue-XXX.md` 5 件が workflow.md 上根拠のない「issue → ticket 押し上げ」だったと確認。workflow.md PR フロー §0 の「チケットは work-breakdown 由来」+ §2「`/implement {チケット or issue}`」より、issue 対応は issue ファイル自体を渡すのが正規。5 件削除コミット (`a958551`)。

#### 5. PR #128 (#169) → 構造誤り発覚 → revert (#129) → 真の修正 PR #130

ユーザー指摘「Approver/Accounting でカード行間がない」→ 私は「2 つの Grid container 間に Box mt:2」アプローチで PR #128 マージ → 視覚再確認で「Admin と異なる結果」と FAIL 報告。原因: MUI Grid v2 の負マージン (-8px) が Box mt:2 (16px) を相殺し実効ギャップ 8px、Admin の 16px と乖離。revert PR #129 → PR #130 で MyReportCountCards Fragment 化 + DashboardPage 単一 Grid 統合（4 枚 3+1 折り返し、Admin TenantStatusCards と同構造）に再実装。reviewer 1 回目で test_cases.md / screens.md の同期更新指摘 → ops-writer で対応 → 再レビュー PASS、codex APPROVE、マージ完了 (`010edb7`)。

#### 6. PR #131 (#160 追加対応): 処理済み 現在ステータス 140→170

SMK-101 再検証で「処理済みレポート一覧の現在ステータス列のみ FAIL」と判明（ヘッダ「現在ステータス」6 文字 + ソートアイコンで 140 不足）。PR #131 で 140→170 に再引き上げ + 設計書同期。reviewer + codex APPROVE、マージ完了 (`cde1118`)。

#### 7. ローカルテストポリシーの memory 修正

「テストは PR → CI で確認」の memory ルール前提が、GitHub Actions 無料枠制限と CI 未稼働状態で破綻していると指摘されメモリ全面書き直し。**「指揮役の /test ローカル実行が正規ゲート、GitHub Actions は補助扱い」**に変更。

#### 8. SMK 全 PASS → 8 件 issue resolved 移動 + Step 11-A 完了

ユーザー実機で SMK-101（5 画面）/ #168（Member/Admin カード高さ）/ #169（Approver/Accounting 行間）の 3 件全 PASS 確認。8 件 issue (#157/#158/#160/#162/#164/#166/#168/#169) を `resolved/` 移動 + 各ファイルに「解決確認」セクション追記 + progress.md の Step 11-A を「進行中」→「完了 (2026-05-04)」に更新 (`88d8ce8`)。

#### 9. 11-A ticket SMK 結果テーブル更新の運用誤り → 是正

ユーザー指摘「resolved にあるからといって動作確認した証明にならない。FAIL のままのものはちゃんと動作確認する必要がある」。SMK-101 のみ本セッションで実機 PASS 確認したため ticket に PASS 反映 (`85c96f7`)。残 FAIL は次セッション以降の運用とする旨を活動ログに明記。

### 未完了

- 11-A ticket の他 FAIL 項目（SMK-013/021/022/023/028/033/035/036/050/051/052/061/062/096/102 等）の実機再検証 + PASS 反映
- issue #165（マイレポートフィルタエリア mobile）対応
- Step 11-B 横断テスト Go / Step 11-C E2E Playwright 着手判断
- dev-journal master の push（4 commit ahead）

### ブロッカー

なし

### 次にやること

#### 優先度 1: 11-A ticket の残 FAIL 項目の再検証 + 反映

resolved 済 issue に対応する FAIL 項目から順に実機再検証。優先候補:
- SMK-061 / SMK-062（#137 / #138 関連）
- SMK-096（#159 関連、#162 と統合された可能性あり）
- SMK-013 / SMK-021〜SMK-036（多数の resolved/ 済 issue）
- SMK-050〜SMK-052（日本語 UI）

各項目について実機再検証 → PASS なら ticket 更新、FAIL なら追加対応 issue 起票。

#### 優先度 2: issue #165 対応

マイレポートフィルタエリアの mobile レスポンシブ化（Stack direction xs:column → sm:row）。

#### 優先度 3: Step 11-B / 11-C 着手判断

11-B 横断テスト Go (cross_cutting_test.go / rate_limit_test.go) と 11-C E2E Playwright を並列着手 or 順次。本格着手前に work-breakdown を再確認して見積もり。

#### 優先度 4: dev-journal master の push

4 commit ahead 状態。`git -C /root-project/dev-journal push` でリモート反映。

### 学び・気づき

#### 「資料整理 = 動作確認」と誤認した

8 件 issue を resolved 移動した直後、11-A ticket の SMK 結果テーブルも自動的に PASS 更新する案を出したが、ユーザー指摘「resolved だからといって動作確認した証明にならない」で撤回。**resolved 移動と SMK PASS は独立した判断軸**。SMK PASS は実機再検証 + 個別記録が必要。

#### memory の前提が時代遅れだった

「PR → CI で確認」の memory ルールは GitHub Actions が動いている前提だったが、CI が未稼働の現状ではこのルールに従うと品質ゲートが消失する。**memory も陳腐化する**。前提条件が変わったら都度見直し。CI が動いていないことを確認したのはユーザー指摘で、私は最初気づいていなかった。

#### Box mt:2 の負マージン相殺を見落とした

PR #128 で「Box mt:2 で行間 16px 確保」と判断したが、内側 Grid container の `spacing={2}` が負マージン (-8px) を持つため実効ギャップは 8px だった。**MUI Grid v2 の spacing 仕組みを理解せずに視覚的近似で判断した**。視覚 16px ≠ box-model 上の mt 値。MUI コンポーネントの内部実装を踏まえた構造設計が必要。

#### ユーザー指摘の「他のドメイン」の意図を読み違えた

「Approver/Accounting でカード同士の余白がない」というユーザー指摘を、私は「2 つの Grid container 間の余白」と解釈したが、ユーザーは「4 枚カードを単一 Grid 内で 3+1 折り返し（Admin と同構造）」を意図していた。**ユーザーの「同じ」という言葉を浅く受け取らない**。Admin の構造を正しく観察して同じものを作る視点が必要。

#### issue → ticket 押し上げ運用は workflow に存在しない

過去セッションで `progress-management/tickets/step11/11-A-issue-XXX.md` を起票していたが、workflow.md には「issue → ticket 化」のルールはない（チケットは work-breakdown 由来のみ）。重複情報の生成だった。**workflow.md にないことを勝手に運用しない**。

#### vitest 並列実行は CPU 競合で逆に遅い

PR #126/#127 のテストを並列起動したが CPU が両者に分散し各々 26 分かかった（合計約 52 分）。順次なら各 15-20 分（合計 30-40 分）で済んだ可能性。**並列が常に速いとは限らない**。CPU 集中タスクは順次の方が良い場合もある。

### 意思決定ログ

#### #169 真の修正の構造判断

- 案 A: DashboardPage で単一 Grid 化（コンポーネント維持しつつ MyReportCountCards を Fragment 化）→ 採用
- 案 B: MyReportCountCards に extraCard props 追加 → 不採用
- 案 C: 共通カード行コンテナ post-MVP 前倒し → 不採用
- 採用理由: コンポーネント維持の制約下でも構造的整合性（Admin TenantStatusCards と同パターン）を優先

#### case A1 vs A2

- A1: ApproverActionCards / AccountingActionCards をコンポーネント抽出せずインライン → 採用
- A2: 抽出 → 不採用
- 理由: 4 枚目は単一 CountCard でロジックなし、抽出する意義が薄い。MVP の過剰設計回避

#### memory feedback_no_local_test_run.md の改訂方針

- 旧: 「テストフルスイートは CI で回す。サブエージェントにローカル実行させない」
- 新: 「サブエージェントにフルテスト走らせない。指揮役の /test ローカル実行が正規ゲート（GitHub Actions は無料枠制限で補助扱い）」
- 理由: GitHub Actions 無料枠の制約とユーザーの方針（ポートフォリオ用に課金しない）

#### 11-A ticket の FAIL 反映方針

- 案 A: resolved 済 issue 対応 SMK を全件 PASS 反映 → 不採用（ユーザー指摘で撤回）
- 案 B+C: 本セッション実機再検証分のみ PASS（SMK-101）+ 残は次セッション以降の TODO → 採用
- 理由: SMK PASS 判定は実機動作確認が前提、resolved 状態とは独立

### PR / コミット要約

**expense-saas**（マージ済み 6 PR + revert 1）:
- PR #126 (#160 5 画面 minWidth 引き上げ) → master `3a3719e`
- PR #127 (#168 CountCard transparent border) → master `df3e8bf`
- PR #128 (#169 Box mt:2、後に revert) → master `6a23937`
- PR #129 (PR #128 revert) → master `07558ee`
- PR #130 (#169 単一 Grid 統合) → master `010edb7`
- PR #131 (#160 追加対応 現在ステータス 140→170) → master `cde1118`

**dev-journal**（4 commit ahead、push 未実施）:
- `d32c7eb` docs(issues): #169 起票
- `7ec1445` docs(dashboard): #169 真の修正に伴う設計書・テスト仕様書の同期更新
- `f3b9190` docs(reports): #160 再々検証 FAIL の真因記録 + 現在ステータス 140→170
- `88d8ce8` docs(progress): Step 11-A クローズ — 8 件 issue resolved 移動 + progress.md 更新
- `85c96f7` docs(11-A): SMK-101 PASS 反映

**root-project**: memory 1 件更新（feedback_no_local_test_run.md 全面書き直し + MEMORY.md インデックス）

**ai-dev-framework**: 変更なし
