# 422 レスポンスに details 配列が欠落し、フィールド単位のエラー情報が返らない（OpenAPI 契約違反）

## 発見日
2026-04-19

## カテゴリ
implementation / backend / api-contract / error-response

## 影響度
中（通常操作で踏む経路は限定的だが、OpenAPI 契約違反 + 将来の UI field-level error 表示ができない）

## 発見経緯
Step 11-A ローカル動作確認（SMK-023 422 サーバーサイドバリデーション）で、DevTools Console から `/api/reports/{reportId}/items` に `amount: 100.5`（小数）を含むリクエストを送信。サーバーは正しく 422 を返したが、レスポンス body に OpenAPI 契約で定義されている `details` 配列（field 単位のエラー情報）が含まれていないことを発見。

## 関連ステップ
Step 5（API 詳細設計 openapi.yaml）/ Step 8-4（middleware / エラー応答共通化）/ Step 10（各機能ハンドラー実装）/ Step 11-A

## ブロッカー
なし（フロントバリデーションが先に弾くため通常操作で踏まない）

## 問題

### OpenAPI 契約（正本）

`50_detail_design/openapi.yaml:1856-1870`:

```yaml
Error:
  properties:
    error:
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: array
          description: 400/422 のバリデーションエラー時のみ付与。フィールド単位のエラー情報を配列で返却する
          items:
            $ref: "#/components/schemas/ValidationError"

ValidationError:
  required: [field, message]
  properties:
    field:
      type: string
      example: title
    message:
      type: string
```

`openapi.yaml:2602-2614` 例:
```yaml
ValidationFailed:
  example:
    error:
      code: VALIDATION_ERROR
      message: "Input validation failed"
      details:
        - field: title
          message: "Title is required"
```

### 実装の挙動

明細作成 API (`POST /api/reports/{reportId}/items`) に小数 `amount: 100.5` を送信した際のレスポンス:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "invalid request body"
  }
}
```

`details` 配列が存在せず、`message` も汎用的な「invalid request body」のみ。どのフィールドが不正か、どう直せばよいかが分からない。

### 根本原因

ハンドラー実装が `json.NewDecoder(r.Body).Decode(&req)` の失敗を単純に汎用メッセージで返している:

`expense-saas/internal/handler/report.go:211-214` 等:
```go
var req createReportRequest
if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
    middleware.RespondError(w, http.StatusUnprocessableEntity, "VALIDATION_ERROR", "invalid request body")
    return
}
```

`amount int` と `amount float64` の型ミスマッチが Go の json パッケージで検出された時点でエラーが返されるが、どのフィールドで失敗したかの情報は捨てられている。また `RespondError` 自体がおそらく `details` 引数を受け取らない設計（要確認）。

### 影響範囲

- 明細作成 / 更新 ハンドラー（`item.go`）
- レポート作成 / 更新 ハンドラー（`report.go`）
- 他の POST/PUT ハンドラー全般（`auth.go`, `workflow.go`, `tenant.go` 等）
- 共通エラー応答ヘルパー（`middleware.RespondError`）

## 影響

### 実害

- 通常操作ではフロントバリデーション（react-hook-form + Zod）が先に弾くため 422 到達経路はほぼない
- DevTools による改変・curl/外部クライアント等の非正規経路でのみ実害が出る

### 仕様遵守・保守性

- OpenAPI 契約違反（`details` は 400/422 時に「付与する」と定義）
- 外部クライアント（API 利用者）が自動生成コードで `details` を期待している場合にエラーになる
- Post-MVP で 422 UI field-level 表示を実装する場合、このバックエンド改修がブロッカーになる

## 修正方針

### 案A（推奨）: 構造体タグバリデーション + details 生成

- 入力 struct に `go-playground/validator` 等のバリデーションタグを付与
- JSON デコード後に `validator.Struct(&req)` を呼ぶ
- バリデーション失敗時、`ValidationErrors` を field/message ペアの配列に変換
- `middleware.RespondError` を `RespondValidationError(w, code, msg, details)` 経由で拡張

### 案B: 手書きの個別バリデーションと details 組み立て

- 各 handler で field 単位に検査し、失敗時に `details` にエントリ追加
- 実装量が増えるが、既存の愚直な validation ロジックと相性が良い

### 案C: 最小限対応（JSON decode エラーを details なし、他の手動 validation のみ details 付き）

- JSON decode エラーは現状維持（汎用メッセージ）
- 型 decode 通過後の手動 validation（title empty, period_start format 等）のみ details 付きで返す
- 実装量最小、ただし「型ミスマッチ」系は依然として field 情報が落ちる

## 修正対象ファイル

- `expense-saas/internal/middleware/` のエラー応答ヘルパー（`RespondError` 拡張）
- `expense-saas/internal/handler/` 配下の全 POST/PUT ハンドラー
- `expense-saas/internal/handler/*_test.go` のエラーレスポンス形式検証テスト

## 完了条件

- 400/422 レスポンスで `details` 配列が付与され、各 field の失敗理由が返る
- 上記 example と同形式でフロントが field-level エラー表示を行える
- OpenAPI 契約（`openapi.yaml`）と実装レスポンス形式が一致する
- 関連テストが details 配列の内容を検証する

## 関連 SMK

- SMK-023: 422 サーバーサイドバリデーション — 本 issue により UI 反映まで含めた完全な検証は不可能。本 issue 解消後に再検証可能になる

## 次セッション着手手順

1. 方針選定（案A / 案B / 案C）をユーザーと合意
2. ブランチ作成: `fix/121-422-details-array`
3. `middleware/RespondError` の拡張 or `RespondValidationError` 新設
4. 全 POST/PUT ハンドラーの JSON decode + validation 経路を改修
5. テスト更新（details 配列の形式・内容を検証）
6. フロント側の API 型定義（`types.ts` の ErrorResponse）も必要に応じて更新
7. PR 作成
