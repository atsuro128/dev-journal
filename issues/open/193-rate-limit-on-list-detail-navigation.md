# レポート一覧↔詳細の往復で 100 req/min に到達 + F5 リロードで生 JSON 表示（post-MVP）

## 発見日
2026-05-27

## カテゴリ
ux / non-functional

## 影響度
中（業務継続可能だが、Accounting/Admin の業務密度では実用上のストレス。F5 時の UX は明確に致命的）

## 発見経緯
UAT-032 進行中の自由探索（全レポート一覧 → レポート詳細 → 戻る → 別レポート選択 を繰り返した）

## 関連ステップ
Step 11-F UAT / 設計成果物: `dev-journal/deliverables/docs/50_detail_design/security.md` §4 / `frontend/src/lib/error-messages.ts`

## ブロッカー
なし（1 分待てば回復、業務継続は可能。ただし MVP リリース基準としての許容度はユーザー判断）

## 問題

### A. 業務密度に対する 100 req/min レート制限の妥当性

**発生条件**: Accounting / Admin の「全レポート一覧 → レポート詳細 → 戻る → 別レポート選択」を短時間で繰り返す（**業務で十分にあり得る密度**）と、認証済みリクエスト 100 req/min の制限に到達する。

**設計（`security.md` §4.1）**:
| 対象 | 制限値 | キー |
|---|---|---|
| 認証済みリクエスト | **100 req/min** | user_id |

**実測のリクエスト数**: 詳細画面 1 回開くごとに `GET /api/reports/{id}` + `GET /api/reports/{id}/items` + 添付一覧 + ダッシュボード件数等で 3-5 API。一覧 ↔ 詳細往復で 20 回程度の操作で容易に到達する。

業務想定（特に Accounting/Admin の月次経費全体確認 UC-AC03 / UC-AD02）では、1 分間に多数のレポート閲覧は十分あり得る。

### B. レート制限超過 → F5 リロードで API URL の生 JSON が表示される

**発生条件**:
1. UAT 中、レート制限超過 → FE はトーストエラーで「レート制限超過」を表示（**ここまでは正常**）
2. ユーザーが F5（リロード）でリトライ
3. **ブラウザに `{"error":{"code":"RATE_LIMIT_EXCEEDED","message":"Too many requests..."}}` の生 JSON が一杯に表示される**

**仮説**:
- リロード時に URL が `/api/reports/{id}` のような API URL に変わっていた可能性（SPA ルーティング異常）
- もしくは、Go SPA fallback handler が `/api/...` を捕まえて index.html を返さず、API ハンドラに転送 → 429 レスポンスの JSON が描画
- 再現条件の確定には追加調査が必要

**設計との乖離**:
- 設計（`screens/admin-all-reports.md` §7）では、API 通信エラーは「トースト」で表示する規定
- 生 JSON がページ本体に表示されるのは設計外の挙動

### C. 回復速度に対する体感の遅さ

**現象**: レート制限超過後、操作再開できるまで「思ったより時間がかかる」とユーザーが UAT 中に体感。

**設計（`security.md` §4.2）**:
- アルゴリズム: **トークンバケット方式**
- ストレージ: インメモリ（`golang.org/x/time/rate` 等）
- ウィンドウ: 1 分間のスライディングウィンドウ

**仮説**:
- トークンバケット方式なら本来、リクエスト終了後すぐにトークンが補充され、徐々に操作可能に戻るはず（補充レート: 100/min = 1.67 req/sec）
- 「スライディングウィンドウ」と「トークンバケット」が混在記述されており、実装次第で挙動が異なる
  - 純粋なスライディングウィンドウ: 過去 1 分間の総数を見るため、超過した瞬間から最古のリクエストが窓外に出るまで待ち
  - トークンバケット: 連続して叩けないが、徐々に補充される
- 実装が固定ウィンドウまたは pure スライディングウィンドウになっている場合、超過後の体感が「すぐ戻らない」になる

**追加調査ポイント**: `internal/middleware/ratelimit.go` 等の実装が `golang.org/x/time/rate.NewLimiter` ベースのトークンバケットになっているか、それとも固定/スライディングウィンドウ実装になっているかを確認

### D. FE レート制限ハンドリングの網羅性

`frontend/src/lib/error-messages.ts:72` で `RATE_LIMIT_EXCEEDED` の汎用メッセージは定義済みだが、専用ハンドリング（`if (err.code === 'RATE_LIMIT_EXCEEDED')`）の実装が認証系 3 画面（Login / Signup / PasswordReset 系）のみ。

レポート系・ワークフロー系のフックには専用ハンドリングがなく、共通の `err.message` 表示のみで対応している。これは設計意図 (`state-management.md` §6.5) 通りだが、状況によっては UX が不十分。

## 影響

- Accounting / Admin の業務効率に影響（1 分のクールダウン待ちが発生する場面がある）
- F5 リロードでの生 JSON 表示は MVP のユーザー体験として明確に不適切（②）
- ただし業務継続自体は可能（1 分待てば回復）

## 提案

### 案 A: B（F5 で生 JSON）を最優先で fix、A（設計値）は post-MVP 検討
- B は SPA fallback / ルーティングのバグ可能性。再現条件を絞り込んで fix
- A は post-MVP で再検討（実運用データを見てから判断）

### 案 B: A + B を MVP リリース前に両方対応
- 設計値を 200 req/min に緩和（または認証済みは制限なしオプション）
- B の SPA fallback バグも fix
- 工数増だが MVP 品質を高める

### 案 C: 全部 post-MVP（UAT-032 OK 記録のまま進める）
- 業務継続は可能、1 分のクールダウンを許容
- F5 で生 JSON は post-MVP の課題として残置

### 追加調査ポイント（B 系統）

- F5 直前の URL は何だったか？（`/reports/{id}` か `/api/reports/{id}` か）
- リロード時のリクエストヘッダー / レスポンスヘッダー
- Go SPA fallback handler の `/api/*` 経路ハンドリング（PR #148 で実装、`internal/spa/handler.go`）
- React Router の useNavigate / Link で誤って API URL に遷移していないか

## 着手タイミング

**B（生 JSON 表示）は post-MVP の早期に再現追跡**（再現できれば fix は容易と推定）
**A（設計値 100 req/min）は MVP リリース後に実運用観察で再評価**

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
