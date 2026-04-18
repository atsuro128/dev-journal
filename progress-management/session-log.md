# 引き継ぎメモ

## セッション: 2026-04-18 10:30〜15:12

### ゴール
- issue 102 結合動作確認（SMK-037 / 認可 / テナント越境 / ポップアップブロック / 署名付き URL）
- 最終 PR（integration → master）マージで issue 102 完了

### 作業ログ

#### SMK-037 動作確認で UX 問題発覚
1. Docker 起動 → Member でログイン → reportSubmitted で SMK-037 確認
2. ↓ アイコン押下時も新タブ open される UX 問題発覚（空白タブが残る）
3. master 旧実装を確認 → C パターン（signed URL を動的 `<a download>` 要素クリック）と判明
4. ユーザーが「D（blob 取得）は設計意図とズレ」の質問 → 私の誤読を訂正（対応前は D ではなく C だった）

#### ダウンロード実装の C パターン化
5. architect に修正計画依頼 → issue 102 本文末尾に「追加対応ログ（2026-04-18）」として計画書追記
6. frontend-developer + designer を並列起動（別リポジトリなので重複なし）:
   - frontend-developer (worktree `fix/102-download-anchor-pattern`): `handleDownload` を C パターンに変更 + テスト修正 → PR #66（`22dcdd3`）
   - designer: 設計書 6 ファイル修正（計画の 5 + 整合性のため `security.md §6.2` も）
7. dev-journal 側 2 コミット（設計書 `8ddf9b9` / 追加対応ログ `1a51197`）
8. reviewer 起動 → PASS（warning 2: W-1 `link.click()` スパイ欠落 / W-2 JSDoc 表現 / info 3）
9. 指揮役判断で W-1 を PR 内対応に変更（reviewer は「別 issue 起票で OK」と提案したが、`feedback_accountability` に従い乖離を残さない方針）
10. frontend-developer 再起動 → W-1/W-2 対応追補コミット `585b2cf`
11. reviewer 再レビュー → PASS（残指摘 0）

#### codex レビューで stdin 問題発覚
12. /codex-review 初回: `echo "" | cd && codex exec` の順序ミスで stdin 無限待機 → pkill
13. 正しい順序（`cd && echo "" | codex exec`）で再実行 → PASS（対象テスト 20/20、lint PASS）
14. /codex-review スキルの 6 箇所コマンド例 + 注記を修正 → `3e62dba`（冗長な誤りパターン解説も削除）

#### PR #66 マージと Docker 再起動トラブル
15. PR #66 を integration に squash マージ（`aeb32d4`）
16. 本体 integration 最新化
17. ユーザーが Docker 再ビルド → worktree 配下の node_modules シンボリックリンクでビルドエラー
18. worktree 2 つ削除 + `.dockerignore` に `.claude` 追加（`046f39b`）

#### CI 運用の根本見直し（本セッションの大きな副産物）
19. ユーザー指摘「CI 通したの？」→ 指揮役が /test スキルを実行していなかったことが判明
20. 原因究明:
    - エージェントの自己申告（lint/test PASS）を指揮役の検証と置き換えていた
    - /test スキル description の「PR 作成前のローカル検証時」が誤解を招く表現
    - workflow.md §2 の指揮役 /test 実行手順を機械的に辿らなかった
    - /implement スキルを使わず Agent tool 直接起動していた（チケット対応スキルという縛りで）
21. memory `feedback_no_local_test_run` を再確認 → **サブエージェントにローカル CI を実行させない**方針が正本
22. 私の誤りは「サブエージェントに CI を実行させた」こと自体（プロンプトに含めた）
23. ユーザーの構造提案「workflow.md に書くとサブエージェントが指示に背く必要がある。指揮役が CI 指示しない運用が根本修正」を採用

#### スキル・定義の整理（5 コミット）
24. `test` スキル description を明確化（`325522f`）: PR 作成後・reviewer 起動前に指揮役が実行、の意を明示
25. `.gitignore` に `logs/test-results/` 追加（`2fac35a`、dev-journal）
26. `codex-review` スキル stdin 処理コマンド例修正（`3e62dba`、既出）
27. サブエージェント定義・ルールから CI 実行コマンド削除（`956ed40`、5 ファイル）
28. `/implement` スキルを issue 対応・PR 追加対応に拡張 + ローカル CI 禁止ブロック追加（`6b68ff2`）
29. `/issue` スキルを起票 / 対応 / 一覧に分離、対応フロー（本文読み → 上流確認 → 方針提案 → /implement 呼び出し → 完了処理）を明確化（`793d4ca`）
30. `workflow.md` PR フロー §1 の /implement 引数を汎用化（`ae1d14b`、ai-dev-framework）

#### SMK-037 再開と残確認
31. Docker 再ビルド成功（`.dockerignore` 効果）
32. SMK-037 PASS（JPEG でファイル名→新タブ inline 表示、↓ アイコン→タブ開かずダウンロード）
33. 別件発覚「明細新規追加時にファイルアップロード不可」→ issue 108 L114 に「新規明細で添付できない」として既に記録されていたことを確認（独立 issue ではなく 108 のサブトピック、保留）
34. 認可マトリクス 4 ロール × 2 操作 = 8 ケース全 PASS
35. テナント越境確認: 初回 401（Cookie 認証ではなく `Authorization: Bearer` だった）→ sessionStorage からトークン取得して再 fetch で 404 PASS
36. ポップアップブロック確認: `window.open = () => null` でシミュレート → プレビューはエラートースト / ダウンロードは影響なし、両方 PASS
37. 署名付き URL 有効期限: 14.99 分 PASS

#### 最終 PR マージ
38. PR #67 作成（`integration/102-attachment-preview → master`）
39. sub PR (#64/#65/#66) で既にレビュー済みのため最終 PR は追加レビューなしで squash merge（`c24bfe0`）
40. 本体 master ff 最新化、integration ローカルブランチ削除、`/tmp/expense-pr66` worktree 削除
41. issue 102 → `resolved/` 移動、progress.md の残存 issue テーブルから #102 削除、push（`4209ee6` in dev-journal）

#### 各リポジトリ push
42. root-project push（`793d4ca`）
43. ai-dev-framework push（`ae1d14b`）

### 未完了
- なし（issue 102 完了）

### ブロッカー
- なし

### 次にやること

#### 優先度 1: Step 11-A ローカル動作確認（smoke_check.md 全 62 項目）
- SMK-037 は issue 102 で済み、残項目（SMK-001〜036、038〜062、101〜102）を実施
- 残 open issue（#104 UI RBAC 可視性監査 / #108 関連の動作）の挙動も同時確認
- 発見問題を issue 起票

#### 優先度 2: issue 108 の分割・方針決定
- issue 108 は現在「並行操作整合性」「即時アップロード UX」「新規明細で添付不可」の 3 サブトピックが混在
- 独立 issue に分割（A, B, C 案）するか、1 つにまとめて対応方針を決めるか判断
- 「新規明細で添付不可」は UX として不自然（ユーザー指摘あり）

#### 優先度 3: 残 open issue の整理・対応
- 運用・基盤系: ops-055 / 060 / 061 / ops-062 / 064 / ops-080 / 081 / 084
- Step 11-A 関連: #104 / #108

### 学び・気づき

#### CI 実行の責任分担を誤った（重大）
サブエージェント（frontend-developer）のプロンプトに「ローカル検証: lint/tsc/test/build」を含めて実行させたのが根本的な誤り。memory `feedback_no_local_test_run` は「サブエージェントにローカル実行させない」を明言していた。エージェントの PASS 報告を指揮役の検証と置き換える行動ルールの逸脱が発生。

**対応**: エージェント定義から CI コマンド削除、/implement スキルに「ローカル CI 禁止ブロック」追加、/test スキル description を「PR 作成後・reviewer 起動前に指揮役が実行」に明確化。将来 GHA CI に移行する場合は workflow.md §2 の 1 行書き換えで済む構造。

#### UX 決定時の挙動検証不足（再発）
前セッションの「React hooks rules 違反」に続き、今回も「Content-Disposition: attachment の URL を新タブで開くと空白タブが残る」UX を合意時に端末で挙動検証しなかった。設計書では「↓ アイコン = window.open(download_url)」と明記していたが、実際の UX 退化を確認せず実装に進んだ。

**教訓**: UI/UX の決定時は、実装に入る前に類似パターンをブラウザで動かしてみる（手元の静的 HTML でも可）。設計書に書いた通り作っても UX が退化するパターンは少なくない。

#### codex exec + run_in_background の stdin 問題
`echo "" | cd /root-project && codex exec` の順は cd が stdin を読まないため codex にパイプが届かず `Reading additional input from stdin...` で無限待機する。正しくは `cd /root-project && echo "" | codex exec`。スキルの例示コマンドが誤っていたのが発端で、スキル経由で実行する際は例示をそのまま使っても動かない状態だった。

#### 指揮役は /implement スキルを使うべき
issue 対応時に Agent tool を直接呼び出してしまい、結果プロンプトに「ローカル検証」を入れ込む隙を作った。/implement スキルを経由していれば「ローカル CI 禁止ブロック」が自動挿入される（今回のスキル拡張後）。今後は issue 対応・PR 追加対応でも必ず /implement を使う。

#### issue 108 の記録構造
ユーザーの「明細新規追加時にアップロード不可」指摘は issue 108 の議論ログ（L114）に「追加発見」として記録されていたが、独立 issue 化されなかったため「埋もれた」状態。スコープ拡大時は独立 issue に分割する必要がある。

### 意思決定ログ

#### C パターン採用
- ダウンロードは master 旧実装と同じ動的 `<a download>` 要素 + click 方式に戻す
- プレビューは window.open 方式維持
- `<a download>` 属性はクロスオリジンでは効かない可能性があるが、BE の `Content-Disposition: attachment` で強制ダウンロード保証

#### 最終 PR の追加レビューなし
- sub PR (#64/#65/#66) で既に reviewer + codex レビュー済み、結合動作確認も全 PASS
- workflow.md PR フロー §6 通り、最終 PR (integration → master) は追加レビューなしで squash merge

#### スキル・ルール修正の構造化方針
- 「サブエージェントに CI 指示しない」は workflow.md（指揮役ルール）ではなく、/implement スキルと各エージェント定義に集約
- 将来 GHA CI 移行時は workflow.md §2 の指揮役手順を 1 行書き換えるだけで済むよう、CI 実施の責任ロジックを集中
- /issue スキルの起票 / 対応 / 一覧 の分離で、スキル発動時のガイドが明確化

#### issue 108 は保留
- 本セッションは issue 102 最終 PR に集中
- issue 108 の 3 サブトピック（並行操作 / 即時アップロード / 新規明細で添付不可）の分割・方針決定は次以降のセッションで対応

## 前回セッション

前回セッション（2026-04-17 13:00〜16:17 および 2026-04-17 17:00〜2026-04-18 00:03）の詳細は `dev-journal/archives/session-logs/2026-04-17.md` を参照。
