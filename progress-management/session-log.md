# 引き継ぎメモ

## セッション: 2026-05-04 〜 2026-05-05 13:00

### ゴール

- 11-A ticket の残 FAIL 項目を実機再検証 + PASS 反映（前セッションの優先度 1 を遂行）
- resolved 済 issue 紐付け FAIL を全件再検証して 11-A の SMK 結果テーブルを完成させる

### 作業ログ

#### 1. 残 FAIL 12 件の実機再検証 → 全件 PASS（前半フェーズ）

resolved 済 issue 紐付け FAIL を ticket §4 順序で連続再検証。Member ロール中心。各 SMK ごとに smoke_check.md から手順抜粋 → ユーザー実機操作 → 結果報告 → 即 ticket 更新。

- SMK-007（#088 / PR #47）: `/settings/tenant` 直接アクセス → トースト + ダッシュボードリダイレクト
- SMK-013（#116 / PR #74）: テーブル領域のみスケルトン化、ヘッダー・フィルタ表示保持
- SMK-021（#118 / PR #76）: 金額 0 入力 → ブラー時に赤字エラー表示
- SMK-022（#119,#120 / PR #75）: 開始日/終了日 blur 双方で両フィールドエラー同時表示
- SMK-023（#121 / PR #82）: Console 経由 fetch で `amount: 100.5` 送信 → 422 + details 配列確認（手順は smoke_check.md の DevTools 改変方式から「Network タブで POST を Copy as fetch → Console でボディ書き換え」に置換。Chrome の `allow pasting` 警告も含めユーザー操作を案内）
- SMK-028（#124 / PR #70）: `docker compose stop api` で SPA 操作 → 日本語トースト「サーバーとの通信に失敗しました。」表示確認。最初 `docker compose down`（全停止）でやって fetch pending 保留現象を踏んだため、`docker compose stop api`（API のみ停止）で再検証して PASS 化
- SMK-033 / SMK-035（#131 / PR #85）: 拡張子・サイズエラーが MUI Alert で文言整合表示
- SMK-036（#134 / PR #84）: MIME 偽装 → BE 422 → トースト「JPEG, PNG, PDF のみアップロード可能です」表示。「クライアント検出=MUI Alert / サーバー検出=トースト」の表示形式不一致は #131 設計上の分担（意図通り）として PASS 扱い
- SMK-050（#139 / PR #92）: Member 主要画面巡回でラベル・ボタン全日本語、status 英語露出なし
- SMK-061（#137 / PR #92, #126/#131）: iPhone SE 375px でハンバーガー収納・テーブル単体横スクロール
- SMK-062（#138 / PR #89）: 期間フィールド縦並びで 375px 内収まり

→ サマリ更新 PASS 45→57 / FAIL 16→4 + コミット (`e5c2652`)。

#### 2. 「Docker: 起動 + ブラウザ」タスクの故障判明

ユーザーから .vscode/launch.json の「Docker: 起動 + ブラウザ」が壊れていてフルリセットしか使えないと報告。本セッションのゴール外なので修正は別タスクへ送り、実検証中は `cd expense-saas && docker compose up -d --build` 等のコマンド直打ちで凌ぐ運用に切替。

#### 3. 残 FAIL 4 件は対応 issue が実は resolved 済と判明 → 再検証可能と判断

ticket・progress.md 上「起票のみ」と記載されていた #141/#143/#159/#161 がすべて `issues/resolved/` にあることを発見。「未着手 issue の実装着手」ではなく「実機再検証で PASS 化可能」に方針変更。

- SMK-051（#141 / PR #95）: 日付論理 V5 で両フィールド発火 + フィールド別文言、未操作側の必須エラー先出し回避を確認
- SMK-052（#143 / PR #96）: 新規追加モードの添付トースト発火 + リスト UI 編集モード同型を確認
- SMK-096（#159 / PR #109、PR #110 で #162 regression 解消済）: ダイアログ初期表示エラーなし・空押下/blur で赤字エラー表示・却下実行
- SMK-102（#161 / PR #106）: スマホ幅 MUI Card 適用、画面幅収まり

→ サマリ PASS 57→61 / **FAIL 0** / SKIP 1 / 未実施 0、**Step 11-A 完全クローズ可能**。

#### 4. SMK-052 副次発見: 明細二重登録バグ → #170 起票（blocker）

ユーザー追加報告: 新規レポート作成中の明細追加モーダルで「保存して続けて追加」ボタン押下すると同じ明細が 2 件登録される。100% 再現、トーストは 1 回だがリロード後も DB に 2 件残存（FE onSuccess は 1 回だが API リクエストが 2 回飛んでいる推定）。データ整合性 blocker として #170 起票。

#### 5. 発行 issue テーブルの表記更新（軽作業）

ticket 発行 issue テーブルで「起票のみ」表記の 8 件（#140/#144/#152/#153/#154/#155/#156/#158）が実は resolved 済みと判明。再検証は副次観点で SMK 主観点に影響しないため不要だが、表記の正確性のため「解決済み」に更新（PR 番号は本セッションでは付与せず、別タスクで完全更新する選択肢を残す）。#157/#160/#161 の表記漏れも合わせて修正。

→ コミット (`cdc48b2`)。

### 未完了

- **#170 対応**（明細二重登録、blocker）— 次セッション最優先
- issue #165（マイレポートフィルタ mobile）対応
- 「Docker: 起動 + ブラウザ」タスク故障の issue 起票
- Step 11-B 横断テスト Go / Step 11-C E2E Playwright 着手判断
- dev-journal master の push（前セッション 4 + 本セッション 2 で 6 commit ahead 想定）
- 発行 issue テーブルの PR 番号付与（本セッションで「解決済み」だけ書いた 9 件分）

### ブロッカー

なし

### 次にやること

#### 優先度 1: #170 対応（明細二重登録 blocker）

データ整合性に関わる blocker。issue ファイル（`issues/open/170-save-and-add-another-creates-duplicate-item.md`）に推定原因と修正方針案を記載済み。`/issue 対応` で着手し、`step11/170-...` ブランチで PR フロー。

#### 優先度 2: Step 11-B / 11-C 着手判断

11-A 完全クローズしたので 11-B（横断テスト Go）と 11-C（E2E Playwright）が次の主軸。並列可。本格着手前に work-breakdown を再確認して見積もり。

#### 優先度 3: issue #165 対応

マイレポートフィルタの mobile レスポンシブ化（Stack direction xs:column → sm:row）。小規模 PR。

#### 優先度 4: dev-journal の push

複数コミット ahead。`git -C /root-project/dev-journal push` でリモート反映。

#### 優先度 5（軽作業・任意）: 「Docker: 起動 + ブラウザ」タスク修正の issue 起票

本セッション内では起票せず先送りした保留事項。

### 学び・気づき

#### smoke_check.md の手順は実装ベースで陳腐化することがある

SMK-023 で smoke_check.md は「DevTools でフォーム改変」を指示していたが、実際には「Network タブで Copy as fetch → Console で書き換え」の方が確実かつ過去の検証実績と一致した（issue #121 にも fetch 直接送信の記述あり）。**SMK 手順は smoke_check.md を正本としつつ、実装の挙動と過去の判定根拠を確認して必要なら手順を読み替える**判断が必要。

#### 「全停止」と「API のみ停止」は別物

SMK-028 を `docker compose down`（全停止）で実施したら FE も停止して fetch が pending 保留になり、過去 ticket 備考と同じ「DevTools Offline 検証不能」状態に陥った。`docker compose stop api`（API のみ停止）で Vite プロキシが生きた状態にして再検証で PASS。**「ネットワーク断絶」のテストは FE 経路を残したまま BE のみ落とす**のが正しい再現条件。

#### progress 記載と実態の乖離はリスクシグナル

ticket・progress 上「起票のみ」だった issue が実は全件 resolved 済みだった。状態反映の運用が形骸化していた疑い。本セッションでは表記更新（再検証不要分は「解決済み」とだけ）まで実施したが、**完了後の状態反映は機械的にやらないと乖離が膨らむ**。

#### サマリ件数の一括書き換えで桁ズレを防ぐ

12 件 + 4 件で計 16 件の PASS 反映だったが、サマリ件数（PASS / FAIL）の更新を 1 コミットでまとめてやるか段階的にやるかで管理コストが変わる。本セッションは 2 段階に分けてサマリ更新したので結果整合は取れたが、機械的な集計を ticket に組み込んだ方が良い余地あり（post-MVP）。

### 意思決定ログ

#### SMK-023 の判定スコープを縮小

- 案 A1（採用）: 確認スコープを「BE が details 配列を返すこと」に限定し、Network タブで目視確認 → details 含まれていれば PASS
- 案 A2（不採用）: smoke_check.md の期待結果（FE field-level UI 表示）まで満たすことを要求
- 採用理由: PR #82 は BE の details 配列対応のみで、FE field-level UI 表示は post-MVP 扱い。過去 FAIL の根拠（details 配列なし・メッセージ汎用）が解消されていれば SMK の主観点は満たすと判断。ticket 備考に観点縮小を明記。

#### SMK-036 の表示形式不一致を「意図通り」と判定

- クライアント検出（拡張子・サイズ）= MUI Alert / サーバー検出（MIME 偽装）= トースト という形式不一致を発見
- issue #131 本文に「Alert インライン vs Snackbar トースト」の論点があり、クライアント検出は MUI Alert・サーバー検出は API エラー一般経路（トースト）という設計上の分担になっている
- ユーザー判断「起票不要、PASS 扱い」で確定。UX 一貫性 issue は post-MVP の余地として記録のみ。

#### 「起票のみ」表記の 9 件を「解決済み」だけ書く（PR 番号付与せず）

- 案 A（採用）: 表記更新だけ実施（「解決済み」とだけ書く、PR 番号確認なし）
- 案 B（不採用）: PR 番号も含めて完全更新
- 採用理由: 本セッションのスコープは 11-A クローズで、表記の完全更新は脱線。次タスクで一括整理する選択肢を残す。

#### #170 は本セッション内で対応せず次セッション送り

- データ整合性 blocker だが、本セッションは 11-A クローズが主目的。スコープ拡大を避け issue 起票のみ。

### PR / コミット要約

**expense-saas**: マージ済み PR なし（本セッションは ticket・issue 操作のみ）

**dev-journal**（前セッション 4 commit ahead + 本セッション 2 commit、計 6 commit ahead）:
- 前セッション分 4 件: `d32c7eb` / `7ec1445` / `f3b9190` / `88d8ce8` / `85c96f7`（5 件）
- 本セッション分 2 件:
  - `e5c2652` docs(11-A): resolved 済 issue 紐付け FAIL 12 件を実機再検証 → PASS 反映
  - `cdc48b2` docs(11-A): 残 FAIL 4 件再検証 PASS で 11-A 完全クローズ + #170 起票

**root-project**: 変更なし

**ai-dev-framework**: 変更なし
