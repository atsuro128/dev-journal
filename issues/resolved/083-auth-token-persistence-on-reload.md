# 認証トークンを sessionStorage に永続化（F5 リロードでログアウトしないように）

## 発見日
2026-04-13

## カテゴリ
implementation / frontend / design-correction

## 影響度
高

## 発見経緯
Step 11-A ローカル動作確認で、ログイン状態のままダッシュボードで F5 を押すとログイン画面に戻ることを発見

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
**あり** — Step 11-A の SMK-094 が機能不能。SMK 全項目の実施で F5 ログアウトが頻発しブロッカー化

## 解決
- 解決日: 2026-04-13
- 解決 PR: #48（マージコミット f1ca9cf）
- 対応概要:
  - §1: `state-management.md` L38 を「sessionStorage 永続化」に書き換え、dangling reference 削除、コードサンプル更新、issue 084 への参照追加
  - §2: `frontend/src/stores/auth.ts` を sessionStorage 併用構造に変更（メモリキャッシュ + sessionStorage、try/catch ガード、モジュール初回ロード時に復元）
  - 新規テスト: `frontend/src/stores/__tests__/auth.test.ts`（7 ケース）
- レビュー: 内部レビュー PASS（blocker / warning なし）、codex APPROVE 相当・指摘なし
- 関連 issue: 084（post-MVP の httpOnly Cookie 移行）を別途起票

## 概要

認証トークンが `frontend/src/stores/auth.ts` のモジュール変数（メモリ）にしか保持されておらず、F5 リロードで全状態が消えてログイン画面にリダイレクトされる。これは Step 11-A スモークチェックの SMK-094 期待結果と直接矛盾しており、実機デモでも UX 上の致命的欠陥となる。

設計成果物自体に矛盾があり、`state-management.md` は「メモリのみ・リロードでログアウト」と明記しているが `smoke_check.md` SMK-094 は「F5 でログイン画面に戻らない」を期待結果としている。

ユーザー判断により、**smoke_check.md SMK-094 を正とし、実装と state-management.md を sessionStorage 永続化方式に修正**する方針を採用する。

---

## §1 設計矛盾の解消

### 事実
- `dev-journal/deliverables/docs/55_ui_component/state-management.md` L38: 「永続化: 不要（メモリのみ。ページリロードでログアウト。architecture.md SS4.2 準拠）」
- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-094: 「F5 で再読込 → 表示中の URL でレポート詳細が再描画される（404 やログイン画面に戻らない）」
- 引用先の `architecture.md SS4.2` は **実在しない**（architecture.md は SS6 セキュリティのみ）。dangling reference

### 修正方針
1. `state-management.md` §AuthStore の永続化記述を **「sessionStorage に保存。タブを閉じると破棄。XSS リスクは React 自動エスケープと CSP で軽減。本番運用への移行で httpOnly Cookie 化を検討する」** に書き換える
2. `architecture.md SS4.2` への dangling reference を削除（または存在する別節への正しい参照に置換）
3. ADR を新規追加（`30_arch/adr/0009-auth-token-storage.md` 等）して MVP の判断根拠を残すことを検討（任意。スコープに応じて判断）

---

## §2 実装修正

### 事実
- `expense-saas/frontend/src/stores/auth.ts` L1-2: `let accessToken: string | null = null` のモジュール変数のみ
- F5 で JavaScript モジュールが再評価されトークン変数が `null` 初期化される
- `useAuth.ts` が `getAccessToken() !== null` で `isAuthenticated` を判定するため false になる
- `PrivateRoute` が `/login` リダイレクト → ユーザー観測上「F5 でログアウト」

### 修正方針
1. **トークンの永続化先**: `sessionStorage`（localStorage ではなく）
   - 理由: タブを閉じれば自動消去、他タブとの共有なし、XSS 影響範囲が狭い
2. **`stores/auth.ts` の API は維持**:
   - `setTokens(access, refresh)` → sessionStorage に書き込み + メモリキャッシュ更新
   - `getAccessToken() / getRefreshToken()` → メモリキャッシュ優先、未ロード時は sessionStorage から復元
   - `clearTokens()` → sessionStorage 削除 + メモリキャッシュクリア
3. **アプリ起動時の復元**:
   - モジュール初回ロード時に sessionStorage からトークンを読み戻し、メモリ変数に格納
   - `useAuth()` は引き続き `getAccessToken() !== null` で判定する（API 変更なし）
4. **キー名**: `auth.access_token`, `auth.refresh_token`（プロジェクト内で衝突しない命名）
5. **エラー処理**: sessionStorage アクセス失敗（プライベートモード等）は try/catch でメモリ専用にフォールバック

### 既存テストへの影響
- `frontend/src/api/__tests__/client.test.ts` — `stores/auth` モックの整合性を確認
- `frontend/src/components/auth/__tests__/PrivateRoute.test.tsx` — モック方式は維持可能
- 新規テスト追加:
  - `stores/auth.test.ts`: setTokens → sessionStorage 書き込み、getAccessToken の sessionStorage 復元、clearTokens の削除、sessionStorage 例外時のフォールバック

---

## 修正対象ファイル（推定）

- `expense-saas/frontend/src/stores/auth.ts` — sessionStorage 永続化
- `expense-saas/frontend/src/stores/__tests__/auth.test.ts` — 新規テスト
- `dev-journal/deliverables/docs/55_ui_component/state-management.md` — §AuthStore の永続化記述修正、コードサンプル更新、dangling reference 修正

## 完了条件

- ログイン後の任意の認証必須画面で F5 を押しても URL が維持され、ログイン画面に飛ばない
- ブラウザのタブを閉じて再度開くと未ログイン状態になる（sessionStorage の自然消去）
- `stores/auth.ts` のユニットテストが追加され通過する
- 既存 unit/integration テストが通る
- `state-management.md` の AuthStore 記述が実装と一致する
- SMK-094 の期待結果と実装挙動が一致する

## 関連 issue
- 082（認証フロー UX バグ）— PR #47 で AppLayout / PrivateRoute を追加。本 issue はその延長で発見された残課題
