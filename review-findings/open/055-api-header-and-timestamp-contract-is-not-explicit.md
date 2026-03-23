# 055: API ヘッダー規約と日時フォーマットが API 契約として閉じていない

## 指摘概要
ヘッダー規約と日時フォーマットの一部は `security.md` やスキーマの型記述に分散しており、`openapi.yaml` / `architecture.md` だけを見ても API 契約として確定しない。

- 設計書のどこにギャップがあるか:
  - `X-Request-ID`、セキュリティヘッダー、CORS 公開ヘッダー、レート制限ヘッダーが「どのレスポンスで保証されるか」が `openapi.yaml` で共通ヘッダーとして定義されていない
  - `architecture.md` はミドルウェアの存在を述べるだけで、API 利用者向けのヘッダー契約になっていない
  - `openapi.yaml` の日時項目は `format: date-time` までで、UTC かつ ISO 8601 `Z` 形式で返す規約が明文化されていない
- そのギャップがないと実装者が何に困るか:
  - フロントエンドは `X-Request-ID` やレート制限ヘッダーを常に読める前提で実装してよいか判断できない
  - バックエンド実装者は成功時/失敗時のどのレスポンスに何のヘッダーを必ず付けるかを文書だけでは揃えにくい
  - 日時のシリアライズで `+09:00` オフセット付きと `Z` 付き UTC のどちらを正とするかがぶれる

## 根拠
- `dev-journal/deliverables/docs/30_arch/architecture.md:125`
  - CORS / SecurityHeaders / RequestID ミドルウェアの存在のみ記載
- `dev-journal/deliverables/docs/50_detail_design/security.md:423`
  - API レスポンスヘッダーとして `X-Request-ID` などを定義
- `dev-journal/deliverables/docs/50_detail_design/security.md:361`
  - CORS の公開ヘッダーを別途定義
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:2277`
  - `429` に対するレート制限ヘッダーはあるが、共通ヘッダー化されていない
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1668`
  - `created_at` は `format: date-time` のみ
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml:1754`
  - `updated_at` も同様に `format: date-time` のみ

## 判定
中 / api-conventions の契約不足

## 修正方針案
- 修正対象は **Step 5 詳細設計書のみ**（`architecture.md` は上流の設計判断記録であり、実装者向け仕様書ではないため修正しない）
- `openapi.yaml` に共通レスポンスヘッダーを `components/headers` として定義し、`X-Request-ID` とレート制限ヘッダーの適用範囲を明記する
- セキュリティヘッダーと CORS は `security.md` に定義済みのため、`openapi.yaml` の説明文に「全 API レスポンスで付与」と記載して参照を閉じる
- 日時項目は `format: date-time` に加えて「ISO 8601 UTC (`...Z`)」を明記する

## 再検証結果（2026-03-23）

### 1. ギャップは実在するか

元指摘のうち、ヘッダー規約の未定義という主張は過大。Step 5 だけを見ても、ヘッダー契約は主に `security.md` と `monitoring.md` で読める。

- `security.md` 5.1 に CORS の公開ヘッダーがある
- `security.md` 6.1/6.3 に全レスポンス向けセキュリティヘッダーと `/api/*` の API レスポンスヘッダーがある
- `security.md` 4.3 と `openapi.yaml` の `429` にレート制限ヘッダーがある
- `monitoring.md` にも `X-Request-ID` をレスポンスヘッダーへ設定する運用がある

一方で、API レスポンス本文に含まれる日時フィールドの表現が「UTC の `Z` 形式」であることは、指定された参照資料内では API 契約として明示されていない。`db_schema.md` の UTC 保存や `monitoring.md` のログ時刻例はあるが、API payload の返却形式そのものは `openapi.yaml` の `format: date-time` に留まる。

### 2. 実装者に実害を与えるか

ヘッダー部分は、Step 5 成果物内で実質的にカバーされているため、独立したレビュー指摘としては弱い。日時部分は軽微だが実害がありうる。

- ヘッダー実装は `security.md` を読めば決定できる
- 日時は `Z` 固定かオフセット付き許容かが未確定で、バックエンドのシリアライズとフロントエンドのテスト期待値がぶれる余地がある

### 3. Step 5 のどのファイルをどう修正すべきか

`open/` 維持。ただし論点は「日時フォーマットの API 契約未明記」に絞るべき。

- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml`
  - `format: date-time` の各日時項目説明に「ISO 8601 UTC（例: `2026-03-22T10:30:00.123Z`）」を追記する
  - `info.description` にも API 全体の日時表現ルールとして同内容を 1 行追記するとよい
- `dev-journal/deliverables/docs/50_detail_design/security.md`
  - 6.3 付近または API 共通規約の節に、API payload の日時も UTC `Z` 形式で返す旨を追記して `openapi.yaml` と揃える
