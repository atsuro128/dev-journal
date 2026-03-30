# 8-5 FE-BE 連携レビュー

- レビュー日: 2026-03-30
- レビュー対象: Step 8 タスク 8-5 成果物
- 判定: **FIX** (blocker: 3件, warning: 2件, info: 1件)

---

## 指摘一覧

| # | チェック種別 | 対象ファイル | 問題内容 | 重大度 | 推奨対応 |
|---|------------|------------|---------|--------|---------|
| 1 | 型定義とAPI契約の整合性 | `frontend/src/api/auth.ts` L6-7 | login 関数のレスポンス型が openapi.yaml と不一致 | blocker | 下記詳細参照 |
| 2 | 型定義とAPI契約の整合性 | `frontend/src/api/client.ts` L36-37 | doRefresh のレスポンスパースが openapi.yaml と不一致 | blocker | 下記詳細参照 |
| 3 | 型定義とAPI契約の整合性 | `frontend/src/api/types.ts` L35-42 | AuthUser 型が openapi.yaml UserProfile スキーマと不一致 | blocker | 下記詳細参照 |
| 4 | React 状態管理 | `frontend/src/hooks/useAuth.ts` L3-5 | useAuth が React の再レンダリングをトリガーしない | warning | 下記詳細参照 |
| 5 | 共通エラーハンドリング | `frontend/src/api/client.ts` L97,104 | 204 No Content 等のボディなしレスポンスで res.json() が失敗する | warning | 下記詳細参照 |
| 6 | 未使用コード | `frontend/src/lib/constants.ts` | ERROR_CODES がどこからも import されていない | info | Step 10 で使用予定であれば問題なし。不要であれば削除 |

---

## 詳細

### 1. login 関数のレスポンス型が openapi.yaml と不一致 [blocker]

**対象**: `frontend/src/api/auth.ts` L6-7

```typescript
const res = await api.post<TokenPair>('/api/auth/login', { email, password });
setTokens(res.access_token, res.refresh_token);
```

**問題**: `openapi.yaml` の `POST /api/auth/login` レスポンスは `{data: AuthTokens}` 形式であり、`AuthTokens` スキーマは `{user, tenant, access_token, refresh_token}` の 4 フィールドを持つ（openapi.yaml L1816-1855）。

しかし `api.post<TokenPair>` は `apiClient` を経由し、`apiClient` は `res.json()` の結果をそのまま返す（L104）。つまり返却される値は `{data: {user, tenant, access_token, refresh_token}}` であり、`res.access_token` は `undefined` になる。

また `TokenPair` 型（`types.ts` L30-33）は `{access_token, refresh_token}` のみで、`AuthTokens` の `user` / `tenant` フィールドが欠落している。

**根拠**: openapi.yaml L193-203、architecture.md SS5.2（成功レスポンスは `{data: ...}` ラッパー）

**推奨対応**:
1. `types.ts` に `AuthTokens` 型を openapi.yaml の `AuthTokens` スキーマに合わせて定義する（`user`, `tenant`, `access_token`, `refresh_token`）
2. `auth.ts` の login 関数で `api.post<ApiResponse<AuthTokens>>` を使用し、`res.data.access_token` でトークンを取得する
3. login 成功時に `res.data.user` と `res.data.tenant` の情報を返却できるため、別途 `GET /api/auth/me` を呼ぶ必要がなくなる（L8 の追加リクエストを削除可能）

### 2. doRefresh のレスポンスパースが openapi.yaml と不一致 [blocker]

**対象**: `frontend/src/api/client.ts` L36-37

```typescript
const data = (await res.json()) as TokenPair;
setTokens(data.access_token, data.refresh_token);
```

**問題**: `POST /api/auth/refresh` のレスポンスも `{data: AuthTokens}` 形式である（openapi.yaml L256-265）。`doRefresh` は `fetch` を直接使用しているため、レスポンスボディは `{data: {user, tenant, access_token, refresh_token}}` だが、`data.access_token` でアクセスしようとしている。正しくは `body.data.access_token` である。

**根拠**: openapi.yaml L256-265

**推奨対応**: レスポンスの `data` プロパティを経由してトークンを取得する。

```typescript
const body = (await res.json()) as { data: AuthTokens };
setTokens(body.data.access_token, body.data.refresh_token);
```

### 3. AuthUser 型が openapi.yaml UserProfile スキーマと不一致 [blocker]

**対象**: `frontend/src/api/types.ts` L35-42

```typescript
export interface AuthUser {
  user_id: string;
  email: string;
  name: string;
  tenant_id: string;
  role: 'admin' | 'approver' | 'member' | 'accounting';
}
```

**問題**: openapi.yaml の `UserProfile` スキーマ（L1857-1886）と以下の差異がある。

| フィールド | types.ts AuthUser | openapi.yaml UserProfile |
|-----------|-------------------|--------------------------|
| ID | `user_id: string` | `id: string` (format: uuid) |
| テナント | `tenant_id: string` | `tenant: {id: string, name: string}` (ネストオブジェクト) |

Step 10 でこの型を使って画面を実装する際に、バックエンドレスポンスとフィールド名が一致せず混乱が生じる。

**根拠**: openapi.yaml L1857-1886（UserProfile スキーマ）

**推奨対応**: openapi.yaml の `UserProfile` スキーマに合わせて型を修正する。

```typescript
export interface UserProfile {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'approver' | 'member' | 'accounting';
  tenant: {
    id: string;
    name: string;
  };
}
```

### 4. useAuth が React の再レンダリングをトリガーしない [warning]

**対象**: `frontend/src/hooks/useAuth.ts` L3-5

```typescript
export function useAuth() {
  const isAuthenticated = getAccessToken() !== null;
  return { isAuthenticated };
}
```

**問題**: `getAccessToken()` はモジュールレベル変数を直接読み取るだけであり、React の状態管理（useState, useContext, useSyncExternalStore 等）と連携していない。トークンが `setTokens()` や `clearTokens()` で変更されても、このフックを使用しているコンポーネントは再レンダリングされない。結果として、ログイン後やログアウト後に UI が更新されない。

**推奨対応**: 以下のいずれかの方式で React の再レンダリングと連携させる。
- `useSyncExternalStore` を使用して外部ストア（モジュール変数）の変更を購読する
- `zustand` 等の状態管理ライブラリでトークンを管理する
- React Context + useState でトークン状態を管理する

（注: Step 10 で状態管理が整備される可能性もあるが、認証状態は全画面のルーティングガードに影響するため、基盤フェーズで方式を確定しておくことを推奨する）

### 5. ボディなしレスポンスで res.json() が失敗する [warning]

**対象**: `frontend/src/api/client.ts` L97, L104

```typescript
return retryRes.json() as Promise<T>;
// ...
return res.json() as Promise<T>;
```

**問題**: `apiClient` は成功時に常に `res.json()` を呼び出す。しかし `DELETE` エンドポイントの一部（例: openapi.yaml の添付削除）は `204 No Content` を返す可能性がある。204 の場合 `res.json()` は例外を投げる。

**推奨対応**: レスポンスの Content-Length や status が 204 の場合は JSON パースをスキップする。

```typescript
if (res.status === 204) return undefined as T;
```

---

## レビュー観点チェックリスト

| # | 観点 | 結果 | 備考 |
|---|------|------|------|
| 1 | 開発時プロキシが API パスとヘルスチェックパスをバックエンドに転送するか | OK | vite.config.ts で `/api` と `/health` の両方を `http://api:8080` に転送している |
| 2 | API クライアントが認証ヘッダーを自動付与するか | OK | client.ts L64-67 で `getAccessToken()` を読み取り `Authorization: Bearer` ヘッダーを設定している |
| 3 | 401 受信時にトークンリフレッシュ → リトライが動作するか | NG (blocker #2) | リフレッシュ自体は実装されているが、レスポンスパースの不一致によりトークン取得に失敗する。また並行リクエストのリフレッシュ重複防止（refreshPromise のシングルトン化）は正しく実装されている |
| 4 | 型定義が architecture.md SS5.2 のレスポンス形式と整合しているか | NG (blocker #1, #3) | `ApiResponse<T>`, `ApiListResponse<T>`, `ApiError` の基本構造は SS5.2 と整合しているが、`TokenPair` と `AuthUser` が openapi.yaml の `AuthTokens` / `UserProfile` と不一致 |
| 5 | トークンがメモリのみで保持されているか | OK | stores/auth.ts でモジュールレベル変数に保持。localStorage / sessionStorage の使用なし（Grep で確認済み） |
| 6 | 循環依存がないか | OK | 依存グラフ: `hooks/useAuth.ts` → `stores/auth.ts`（依存なし）、`api/client.ts` → `stores/auth.ts` + `api/types.ts`、`api/auth.ts` → `api/client.ts` + `stores/auth.ts` + `api/types.ts`。循環なし |

---

## 判定

**FIX**: blocker が 3 件あり、いずれもバックエンドとのレスポンス型の不整合に関する問題。これらが未修正のまま Step 10（機能実装）に進んだ場合、ログイン・トークンリフレッシュ・ユーザー情報取得の全てが実行時に失敗する。下流工程が正常に作業を進められない。
