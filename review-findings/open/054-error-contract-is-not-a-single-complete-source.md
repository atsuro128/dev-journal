# 054: エラー契約が単一の完全仕様になっていない

## 指摘概要
`architecture.md` と `security.md` と `openapi.yaml` でエラー契約の持ち方が揃っておらず、実装者が「どのエラーコードを返し、どの項目を JSON に含めるべきか」を 1 つの仕様として確定できない。

- 設計書のどこにギャップがあるか:
  - `architecture.md` の共通エラーレスポンスは `code` と `message` のみで、`details` を含む正式な構造になっていない
  - `architecture.md` のドメインエラー表はコード一覧として不完全で、`BAD_REQUEST`、`INVALID_TOKEN`、`TOKEN_EXPIRED`、`RATE_LIMIT_EXCEEDED`、`INTERNAL_ERROR`、`INVALID_CREDENTIALS`、`SELF_PAYMENT_NOT_ALLOWED` などが載っていない
  - `security.md` の HTTP ステータス別コード表にも `INVALID_CREDENTIALS` がなく、`openapi.yaml` のログイン API だけが個別例として定義している
- そのギャップがないと実装者が何に困るか:
  - ハンドラ共通実装で採用すべきエラー JSON 構造を文書だけでは決め切れない
  - エラーコードの定数化、共通レスポンス生成、テスト期待値が API ごとにばらつく
  - 後続実装者が `openapi.yaml` の個別例外を見落とすと、ログイン系だけ別コード体系になる

## 根拠
- `dev-journal/deliverables/docs/30_arch/architecture.md:203`
  - 共通エラー例が `code` / `message` のみ
- `dev-journal/deliverables/docs/30_arch/architecture.md:213`
  - ドメインエラー対応表が一部コードのみ
- `dev-journal/deliverables/docs/30_arch/architecture.md:404`
  - 共通レスポンス形式のエラー例にも `details` がない
- `dev-journal/deliverables/docs/50_detail_design/security.md:532`
  - HTTP ステータス別コード表に `INVALID_CREDENTIALS` がない
- `dev-journal/deliverables/docs/50_detail_design/security.md:559`
  - バリデーション時のみ `details` を返す別定義がある
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:8`
  - OpenAPI の説明では `details` を含む形式を採用
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:188`
  - ログイン API が `INVALID_CREDENTIALS` を個別定義

## 判定
中 / error-handling の規約分散・不整合

## 修正方針案
- 修正対象は **Step 5 詳細設計書のみ**（`architecture.md` は上流の設計判断記録であり、実装者向け仕様書ではないため修正しない）
- `security.md` または `openapi.yaml` に「全エラーコード一覧」と「エラーレスポンス JSON 構造（details 含む）」を正本として集約する
- `INVALID_CREDENTIALS` 等、openapi.yaml の個別 API にしか定義されていないコードも正本に含める

## 再検証結果（2026-03-23）

### 1. ギャップは実在するか

実在する。今回の制約で `architecture.md` を除外しても、Step 5 内だけで次の不整合が残っている。

- `openapi.yaml` は冒頭説明と `Error` schema で `details` を含む統一形式を示す一方、`security.md` の「8.2 エラーレスポンス形式」は `code` / `message` の 2 項目だけを正式形式として提示している
- `security.md` の HTTP ステータス別エラーコード表には `INVALID_CREDENTIALS` が載っておらず、ログイン API 固有の注記としてしか現れない
- そのため、実装者が「エラー契約の正本はどこか」「全 API 共通のコード一覧は何か」を Step 5 だけで一意に確定しにくい

### 2. 実装者に実害を与えるか

与える。`openapi.yaml` と `security.md` を併読しても、以下が実装時にぶれうる。

- 共通エラーレスポンス DTO に `details` を常設するか
- エラーコード定数の一覧に `INVALID_CREDENTIALS` を含めるか
- テスト期待値で `details` の有無をどこまで固定するか

`openapi.yaml` の schema は構造の下限を示しているが、「コード一覧の正本」としては閉じていない。`security.md` もコード表を持つが、ログイン固有コードが表外にあるため、実質的に単一の完全仕様にはなっていない。

### 3. Step 5 のどのファイルをどう修正すべきか

`open/` 維持。

- `dev-journal/deliverables/docs/50_detail_design/security.md`
  - 8.2 のエラー形式を `details` を含む正式形に修正し、「`details` は 400/422 のバリデーションエラー時のみ使用」と明記する
  - 8.4 のエラーコード表に `INVALID_CREDENTIALS` を通常行として追加し、ログイン注記に依存しない一覧にする
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml`
  - `info.description` に「エラーコード一覧の正本は security.md 8.4」と参照を書き、仕様の起点を固定する
  - 可能なら `components/schemas/Error` の `details` 説明にも「400/422 のみ」を追記して `security.md` と文言を揃える
