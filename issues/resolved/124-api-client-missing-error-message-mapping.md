# API クライアントで SERVER_ERROR_MESSAGES マッピングが未適用（共通層の実装不備）

## 発見日
2026-04-20

## カテゴリ
implementation / frontend / error-handling / design-consistency

## 影響度
中（全 API エラーでユーザーが raw な英語メッセージを見る可能性があり、設計で定義した日本語メッセージ UX が機能していない）

## 発見経緯

Step 11-A ローカル動作確認 SMK-028（ネットワーク断絶時の挙動）の再検証で、`docker compose stop api` により API を停止した状態でダッシュボードに遷移したところ、トーストに **raw な英語メッセージ「Internal Server Error」** が表示された。

設計（`55_ui_component/state-management.md` §6.5）では 500 は `SERVER_ERROR_MESSAGES.INTERNAL_ERROR` = 「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」が Snackbar に表示されることになっており、実装との乖離を確認。

## 関連ステップ
Step 5（詳細設計: state-management.md）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（致命的な機能障害ではないが、エラー時の UX が設計を満たしていない）

## 問題

### 設計

`dev-journal/deliverables/docs/55_ui_component/state-management.md:462-501` に `SERVER_ERROR_MESSAGES` マップが定義されており、各エラーコード（`INTERNAL_ERROR`, `RATE_LIMIT_EXCEEDED`, `RESOURCE_NOT_FOUND`, `FORBIDDEN`, `CONFLICT` など）に対応する日本語メッセージが対応付けられている。

同 §6.3 エラー分類テーブル（L375-384）の方針:
- 500 サーバーエラー → グローバルハンドラ → Snackbar（エラー）
- 429 レート制限 → Snackbar（警告）
- 等々

これらの Snackbar 表示時には **エラーコードから `SERVER_ERROR_MESSAGES` マップを引いた日本語メッセージ** が表示されることが期待される。

### 実装

`expense-saas/frontend/src/api/client.ts:122-135`:

```ts
async function handleErrorResponse(res: Response): Promise<never> {
  let code = 'UNKNOWN_ERROR';
  let message = res.statusText;  // ← デフォルトは res.statusText（英語 HTTP status text）
  let details: ValidationError[] | undefined;
  try {
    const body = (await res.json()) as ApiError;
    code = body.error.code;
    message = body.error.message;  // ← サーバーが返す message をそのまま使う
    details = body.error.details;
  } catch {
    // レスポンスボディが JSON でない場合はデフォルト値を使用
  }
  throw new ApiClientError(message, res.status, code, details);
}
```

**問題点**:

1. `code` は抽出しているが **`SERVER_ERROR_MESSAGES[code]` でフロント側マッピングを引いていない**
2. レスポンス JSON パースに失敗すると `res.statusText`（英語の HTTP ステータステキスト、例: `"Internal Server Error"`, `"Bad Gateway"`）がそのまま `ApiClientError.message` に格納される
3. ユーザーが見るメッセージはサーバー実装 / プロキシ実装に依存してしまう（Vite dev proxy は非 JSON の 500 を返すため英語 statusText が露出する）

### 観察された挙動

SMK-028 再テスト時:
- `docker compose stop api` で Go 側 API 停止
- Vite dev server の proxy が `500 Internal Server Error` を非 JSON で返却
- `apiClient` が JSON パース失敗 → `message = res.statusText = "Internal Server Error"`
- `DashboardPage` が `<AppToast message="Internal Server Error" />` を表示

設計の期待:
- Snackbar に「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」が表示されるべき

## 修正方針

### 案A（推奨）: `apiClient` でコード → 日本語メッセージのマッピング

`handleErrorResponse` でエラーコードを判定し、`SERVER_ERROR_MESSAGES` からマッピングして `ApiClientError.message` に代入する:

```ts
import { SERVER_ERROR_MESSAGES } from '../lib/error-messages';

async function handleErrorResponse(res: Response): Promise<never> {
  let code = 'UNKNOWN_ERROR';
  let serverMessage: string | undefined;
  let details: ValidationError[] | undefined;
  try {
    const body = (await res.json()) as ApiError;
    code = body.error.code;
    serverMessage = body.error.message;
    details = body.error.details;
  } catch {
    // JSON でない場合はコードをステータスから推定
    code = inferCodeFromStatus(res.status);  // 500→INTERNAL_ERROR, 502/503→INTERNAL_ERROR 等
  }
  const message = SERVER_ERROR_MESSAGES[code] ?? serverMessage ?? DEFAULT_MESSAGE;
  throw new ApiClientError(message, res.status, code, details);
}
```

- フロント側の日本語マッピングを優先
- サーバー側の message はフォールバック（422 の field メッセージ等で有用）
- 最終フォールバックとして `DEFAULT_MESSAGE`（「予期しないエラーが発生しました」等）

### 案B: グローバルエラーハンドラで画面層側でマッピング

各画面 / Hook の `onError` でコードを判定して Snackbar 文言を決める方式。共通層は raw のまま。

- 分散するので NG。案 A を採用すべき

## 修正対象ファイル

- `expense-saas/frontend/src/api/client.ts`（`handleErrorResponse` 改修）
- `expense-saas/frontend/src/lib/error-messages.ts` 等（新規: `SERVER_ERROR_MESSAGES` マップと `inferCodeFromStatus` ユーティリティ）
  - state-management.md の `SERVER_ERROR_MESSAGES` を実装に落とし込む
- 各 Hook / 画面のエラー表示テスト（既存テストで raw message を検証している箇所の確認・更新）

## 対応条件

- 500 系エラー時にトーストで日本語メッセージが表示される（例: `INTERNAL_ERROR` → 「サーバーとの通信に失敗しました…」）
- Vite proxy 経由の非 JSON 500 でも日本語メッセージが表示される（ステータスコードから推定）
- 既存の 422 `VALIDATION_ERROR` 等、サーバー側メッセージを優先すべきコードは従来の挙動を維持
- テストで各エラーコード → メッセージ対応を検証

## 関連

- SMK-028 再テストで発覚
- issue 125: smoke_check.md SMK-028 期待文言と state-management.md の不整合（docs 側）
- state-management.md §6.3, §6.5: 設計の正本
