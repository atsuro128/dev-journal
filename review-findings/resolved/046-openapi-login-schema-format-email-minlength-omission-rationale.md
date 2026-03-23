# 046: openapi.yaml login スキーマから format: email / minLength: 8 を削除した妥当性

## 指摘概要
codex レビューにて、`POST /api/auth/login` の request schema から `format: email` と `minLength: 8` を削除したことで、スペック駆動の消費者（SDK 生成・ミドルウェア等）が入力制約を検証しなくなるとの指摘。

## 根拠
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml` L148-155（login request schema）

## 判定
重大度: 低
分類: 意図的な設計判断

## 対応方針: 対応不要（意図的省略）

### 理由

1. **SEC-011 との整合性**: OpenAPI schema に `format: email` を残すと、Go の構造体タグ生成（`go-playground/validator`）で `email` バリデーションタグが自動付与される。ハンドラ層の共通バリデーションミドルウェアがスキーマ制約違反を `400 VALIDATION_ERROR` として返し、認証処理に到達する前にメール形式不正が区別可能になる。これは SEC-011（認証失敗の原因を秘匿する）に直接違反する。

2. **auth-login.md §4 の制約はクライアントサイド**: 「有効なメール形式」「8文字以上（SEC-010）」は画面入力項目の制約であり、フロントエンドバリデーションで実施する。サーバー側 API 契約とは抽象度が異なる。

3. **代替措置**: openapi.yaml の login endpoint description に「SEC-011 に基づきスキーマ制約を意図的に省略」と明記済み。実装者が意図を理解できる。

4. **影響範囲の限定**: `format: email` / `minLength: 8` の削除は login エンドポイントのみ。signup（L98, L104）、password-reset-request（L301）、password-reset（L348）では維持されている。
