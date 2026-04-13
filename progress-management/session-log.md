# 引き継ぎメモ

## セッション: 2026-04-13 14:00頃〜20:50

### ゴール
- Step 11-A ローカル動作確認の着手 — smoke_check.md 全 54 項目を手動実施し、発見した問題を issue 起票して解消

### 作業ログ

#### Phase 1: session-start と着手判断
1. `/session-start` 実行、progress.md / session-log.md / 7 open issue（ops-055, 060, 061, ops-062, 064, ops-080, 081）を確認、未解決レビュー指摘なしを確認
2. ユーザー意向「11-A 着手」で合意
3. 11-A チケット `11-A-local-verification.md` と smoke_check.md 全 54 項目を読み込み
4. 作業計画策定（環境起動準備 → ブラウザ疎通 → 全 54 項目手動実施 → 結果記録 → issue 起票 → 完了処理）

#### Phase 2: プラン確定と結果ファイル初期化
5. ユーザーから「1件ずつ具体的手順で順次提示してほしい」と指示
6. Plan mode で進行プラン作成（`cuddly-knitting-gosling.md`）、ExitPlanMode で承認
7. `dev-journal/progress-management/11-A-smoke-results.md` を新規作成（SMK-001〜094 の結果テーブル + 進捗管理 + FAIL ログ + UI 観察メモ）

#### Phase 3: SMK-001 → 認証バグ発見（issue 082）
8. SMK-001 提示（Member `test-member@example.com` / `TestPass1!`）
9. FAIL: ログイン時に一瞬エラー → ログイン画面に戻る（シード未投入が root cause → `make seed` で復旧後は SMK-001 PASS）
10. 復旧過程で **2 つの実装バグを発見**:
    - **§1 AppLayout 未適用**: `App.tsx` のルートがほとんど `AppLayout` でラップされていない（ヘッダー・サイドバー・PrivateRoute 全欠落）→ SMK-002 以降ブロック
    - **§2 ログイン 401 リダイレクト誤動作**: `api/client.ts` L77-101 が `/api/auth/login` の 401 にも refresh フローを走らせ、`window.location.href='/login'` で強制リロード → エラーメッセージが一瞬で消える
11. Issue **082**（§1 + §2）起票、frontend-developer エージェントに修正委譲（ブランチ `fix/082-auth-flow-ux-bugs`、worktree 隔離、background）

#### Phase 4: PR #47 レビュー・マージ
12. frontend-developer 完了 → `PR #47` 作成（`PrivateRoute.tsx` / `AppLayoutOutlet.tsx` 新規、App.tsx ネストルート化、client.ts 401 バイパス）
13. CI 全 6 ジョブ SUCCESS
14. 内部レビュー（reviewer エージェント） → **FIX**: blocker 1（PrivateRoute/client.ts テストゼロ） + warning 2（死にモック、useAuth 未使用）
15. 指摘評価: blocker は正当（認証クリティカルパス）、warning も正当 → 修正委譲
16. 追加コミット `8dc4f8f`（PRT-001〜003、CLT-401-001〜004 追加、useAuth 置換、死にモック削除）
17. 再レビュー → **PASS**
18. codex レビュー → **APPROVE 相当・指摘なし**
19. `gh pr merge 47 --squash` → マージコミット `e05ddf9`、worktree クリーンアップ
20. Issue 082 を `resolved/` に移動、progress.md 更新

#### Phase 5: SMK-001 / 002 / 003 / 004 / 005 実施
21. ホスト側で `git pull && docker compose down/up -d --build` 完了
22. SMK-001 PASS（Member 入口、下書き/提出中/却下カード確認）
23. SMK-002 PASS（Approver 入口、承認待ちカード追加確認）
24. SMK-003 PASS（Accounting 入口、支払待ちカード追加確認）
25. SMK-004 PASS（Admin 入口、テナント全体集計確認）— 観察: 月別合計が 2026-03 のみ 0 円表示（seed データの分布起因の可能性）
26. SMK-005 PASS（Member ナビ、ダッシュボード/マイレポート/レポート作成のみ表示）
27. UI 観察バッファ 2 件追記: 却下カード右下の赤点 Badge、承認待ち/支払待ちカードのスタイル不統一

#### Phase 6: F5 リロード問題発見（issue 083 + 084）
28. ユーザー報告「ログイン状態で F5 を押すとログイン画面に戻る」
29. 調査: `stores/auth.ts` がモジュール変数のみ（永続化なし）。F5 でトークン消失
30. **設計矛盾発見**:
    - `state-management.md L38`: 「永続化: 不要（メモリのみ。ページリロードでログアウト）」
    - `smoke_check.md SMK-094`: 「F5 で再読込 → ログイン画面に戻らない」
    - → 直接矛盾
31. さらに引用先 `architecture.md SS4.2` は存在しない dangling reference
32. ユーザー判断: **方針 B（smoke_check.md を正とし実装と state-management.md を修正）**。sessionStorage 採用
33. ユーザー追加指示: XSS リスク説明を求める → 将来の httpOnly Cookie 移行を別 issue として起票
34. **Issue 083** 起票（§1 設計矛盾解消 + §2 実装 sessionStorage 化）
35. **Issue 084** 起票（post-MVP の httpOnly Cookie 移行、ops-080 紐付け、081 と同様）

#### Phase 7: PR #48 レビュー・マージ
36. frontend-developer に 083 を委譲、`fix/083-auth-token-persistence` ブランチ・worktree 隔離・background 起動
37. 並行して SMK-002 PASS 記録（このタイミングで Phase 5 の続きを進めた）
38. frontend-developer 完了 → `PR #48` 作成（`stores/auth.ts` sessionStorage 化、7 ケーステスト新規、state-management.md 修正）
39. CI 全 6 ジョブ SUCCESS
40. 内部レビュー → **PASS**（blocker 0 / warning 0 / info 2 のみ）
41. codex レビュー → **APPROVE 相当・指摘なし**
42. ユーザー承認 → `gh pr merge 48 --squash` → マージコミット `f1ca9cf`
43. Issue 083 を `resolved/` に移動、progress.md 更新

#### Phase 8: 実施順序の最適化（Member 先行へ）
44. ホスト側で `git pull && docker compose up -d --build` → F5 リロード維持を確認
45. SMK-003 PASS、SMK-004 PASS、SMK-005 PASS を連続実施
46. ユーザーから「1件ずつアカウント変えるのは非効率」指摘
47. ロール別グルーピングでロール切替 4 回に圧縮する計画を提案
48. さらにユーザーから「セッションまたぎの持続性」指摘 → `smoke_check.md`（設計成果物）ではなく `11-A-local-verification.md`（チケット）に実施順序を追記する方針で合意
49. ユーザーから「Member 先行が直感的」追加指摘 → データ依存（Approver/Accounting がフィクスチャを書き換える）も考慮して Member → Approver → Accounting → 未ログインの順に再設計
50. `11-A-local-verification.md` に「実施順序」セクション追加（Phase 1〜5 + 設計原則）
51. `11-A-smoke-results.md` の「現在地」欄を強化（現在 Phase / 次 SMK / Phase 別進捗テーブル）

#### Phase 9: SMK-007 / 009 / 010 実施
52. SMK-007 **FAIL**: 認可エラーリダイレクトは動作するが、仕様要求の「403 画面 or トースト」が表示されない（`TenantPage.tsx` L30-34 が黙って navigate しているだけ）。非ブロッカーだが UX 問題として FAIL 記録
53. SMK-008 は Phase 5（未ログイン）に送る
54. SMK-009 PASS（ログアウト後の戻るボタンでログイン画面維持、401 は未検証）
55. SMK-010 PASS（reportDraft 提出時のボタン disabled + スピナー + 成功トースト + 提出済みステータス遷移）
56. ユーザー判断「一旦セッション終了」

### 未完了
- **Step 11-A 残り 46 項目**: Phase 2 途中。次は **SMK-012**（添付アップロード中のローディング）
- **UI 観察バッファ 3 件**: 全 SMK 完了後に issue 化（却下カード赤点、承認/支払カードのスタイル不統一、月別合計 0 円表示）
- **SMK-007 FAIL の issue 化**: 認可エラー時のフィードバック欠落。非ブロッカーだが要対応
- **ops-080 対応**: post-MVP スコープ管理方式の決定（前セッションから継続持ち越し）
- **issue 081 / 084 対応**: ops-080 決定後に移動/分割
- **運用系 open issue**: ops-055, 060, 061, ops-062, 064 は長期保留中

### ブロッカー
- なし（issue 082 と 083 で解消済み）

### 次にやること

`/session-start` で状態確認。以下の順で進める:

1. **Step 11-A 続行（Phase 2 Member 連続実施）** — 最優先
   - 開始点: **SMK-012**（添付アップロード中のローディング）
   - 前提データ注意: 本セッションで `reportDraft`（ccc...001）を提出済みにしたため、現在は「提出済み」状態。SMK-012 は reportDraft が必要なので `make seed` で再投入するか、reportDraftEmpty（明細なし）を使える項目から再開するか判断
   - 推奨: `make seed` は冪等（ON CONFLICT DO NOTHING）なので安易に叩いても `reportDraft` が draft に戻らない点に注意。必要なら手動でステータス戻し or DB リセット
2. **UI 観察 3 件 + SMK-007 FAIL の issue 化** — SMK 全項目完了後にまとめて実施
3. **Step 11-B / 11-C 並列起動** — 11-A 完了後、test-implementer に委譲
4. **ops-080 対応** — post-MVP の管理方式決定

### 学び・気づき

- **シード未投入の盲点** — SMK-001 の初回 FAIL は `make seed` 未実行が root cause。ユーザーは「docker compose up -d 済み」と伝えたがシード投入は別ステップ。**環境起動時には compose ps だけでなく seed 完了確認も必要**。次回の 11-A 再開時にも要確認
- **実装バグの連鎖発見** — F5 問題（083）は 082 修正後に初めて気づけた。082 の `AppLayout` / `PrivateRoute` 適用で認証ガードが有効になったため、メモリ専用トークンの弱点が顕在化した。**1 つのバグを潰すと隣接バグが見える** — 環境整備 → 1 機能スモーク → 次のバグ発見、のサイクルは効率的
- **設計矛盾の検出** — `state-management.md` と `smoke_check.md` の矛盾はスモーク実施時にしか発見できない類のバグ。Step 5.5 設計レビューでは検出できなかった。**複数文書の組合わせで矛盾する設計エラー** は、実装後のスモークが唯一の検出機会となる場合がある
- **ロール切替の最適化を最初に設計すべきだった** — 初回 SMK 提示時、smoke_check.md の ID 順で機械的に進めてしまい、切替コストの問題をユーザー指摘まで気づけなかった。**初回着手時に「実施順序の最適化」を考える** べきだった。SMK 数 × ロール数のマトリクス分析は事前にできた
- **持続可能性の観点** — セッション間の引き継ぎを減らすには「次セッションで同じ文脈を再構築できる成果物」を書き残す必要がある。チケット側に手順を書く判断は、単発セッションでの進め方ではなく、プロジェクトのライフサイクル全体を意識した判断
- **Member 先行の理由は「心理」ではなく「データ依存」** — ユーザーは「Member 先行の方が心理的に楽」と指摘したが、実際にはフィクスチャ状態変更の依存関係（Approver が reportSubmitted を承認する、Accounting が reportApproved を支払処理する）でも Member 先行が必須。直感が技術的正解と一致するケース
- **issue 084 を ops-080 紐付けで起票する判断** — sessionStorage 採用は妥当な MVP 判断だが、本番移行時には httpOnly Cookie が正解。この「将来の正解」を忘却しない仕組みとして ops-080 管理が有効。issue 081 と同じパターンで MVP スコープ外事項を明示的に記録
- **codex レビューが両 PR で指摘ゼロ** — 本セッションの 2 PR（#47, #48）は codex が両方とも APPROVE 相当で通った。前セッションの issue 079 で codex が 4 ラウンド指摘を出したのとは対照的。**差分のサイズ・複雑度・文書横断範囲が小さい PR は codex も 1 回で通る**

### 意思決定ログ

- **issue 082 の 2 不具合を 1 PR で修正**: §1 AppLayout 未適用 と §2 ログイン 401 リダイレクトは別々の原因だが、どちらも認証フロー UX に属する。1 PR で修正する方がレビュー・マージコストが安い
- **PrivateRoute の採用と useAuth 経由**: reviewer 指摘で `getAccessToken` 直接呼び出しから `useAuth()` フック経由に変更。認証状態取得経路を一本化し、将来 httpOnly Cookie 移行時の変更点も最小化
- **issue 083 は方針 B（実装修正）を採用**: 方針 A（smoke_check.md 側を修正）は UX が破綻するため不採用。ポートフォリオ MVP として F5 ログアウトは致命的欠陥。sessionStorage を localStorage より優先（タブ閉じで消える、他タブ共有なし）
- **XSS リスクの受容判断**: sessionStorage は XSS があれば読み取り可能だが、React 自動エスケープ + dangerouslySetInnerHTML 不使用 + 将来 CSP 追加で多層防御可能。本番運用時には httpOnly Cookie に移行（issue 084）
- **実施順序はチケット側に追記**: smoke_check.md（テスト設計書）は「何を検証するか」の正本として安定性が必要。「どの順序で実施するか」は作業計画の領域なのでチケット側（11-A-local-verification.md）に追記
- **Member 先行の順序**: データ依存を考慮。Approver SMK-011 が reportSubmitted を承認して状態を変更するため、Member 系で reportSubmitted を参照する項目（SMK-037, 070 等）は Phase 2（Member）で完了させてから Phase 3（Approver）に進む
- **SMK-007 を FAIL と判定**: `feedback_accountability.md` の「安易に PASS しない、仕様との乖離は黙って見逃さない」に従い、リダイレクトは動作するが仕様要求のフィードバック（403 画面 or トースト）が欠落しているため FAIL とした。非ブロッカーだが UX 問題として記録
- **UI 観察は issue 化せずバッファに溜める**: SMK 全項目完了後に分類してまとめて issue 化。粒度が細かすぎる issue は避ける
- **session-log archive 命名**: 前回セッション（11:00〜13:24）は終了日で 2026-04-13.md に追記。本セッション（14:00〜20:50）は次セッション開始時に同 2026-04-13.md へ追記される想定

### 環境再開時の注意

- **docker compose の状態**: ホスト側で `up -d --build` 済み、`make seed` 実行済み
- **フィクスチャ状態**:
  - `reportDraft`（cccccccc-0001-0001-0001-000000000001）: 本セッションの SMK-010 で **提出済み** に変更した
  - その他フィクスチャは seed 初期状態のはず
- **再開時の推奨**:
  - `reportDraft` が提出済みのままだと SMK-012（draft への添付アップロード）以降で使えない
  - 対応: DB リセット（`docker compose down -v && up -d && make seed`）または Accounting で `reportPaid` を使った SMK 項目から再開
  - 判断は次セッション冒頭で

### PR/マージ済み（本セッション）

| PR | タイトル | マージコミット | Issue |
|----|---------|--------------|-------|
| #47 | fix/082: 認証フロー UX バグ修正（AppLayout/PrivateRoute + login 401 リフレッシュ除外） | `e05ddf9` | 082 |
| #48 | fix/083: 認証トークンを sessionStorage 永続化（F5 リロード後もログイン維持） | `f1ca9cf` | 083 |

### Open issue（本セッション追加分）

| ID | タイトル | ステータス |
|----|---------|-----------|
| 084 | 認証トークン保管方式を sessionStorage → httpOnly Cookie へ移行（post-MVP） | 起票のみ、ops-080 紐付け |
