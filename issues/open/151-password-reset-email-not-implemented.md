# パスワードリセットのメール送信が未実装で、要件 AUTH-F06 と設計書記述に乖離（post-MVP）

## 発見日
2026-04-28

## カテゴリ
implementation / architecture

## 影響度
中（ポートフォリオ・デモ用途では機能影響なし。本番運用では必須機能の欠落）

## 発見経緯
user-report / Step 11-A SMK-100 検証時にユーザーから「これって本来は本当にメールが届くんですか？」との指摘で発覚

## 関連ステップ
Step 1（要件定義）/ Step 3（アーキテクチャ設計）/ Step 10-B（認証実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書の規定

- `10_requirements/requirements.md:476`: 「認証機能としてのパスワードリセットメール（AUTH-F06）は MVP 対象」
- `50_detail_design/screens/auth-password-reset-request.md:186`: 「メールアドレスが登録済みの場合、サーバー側でリセットトークン（1 時間有効、SEC-006）を生成し、**メールを送信する**」
- `40_basic_design/ui_flow.md:66`: 状態遷移「メール送信完了」を AUTH002 へ遷移条件として定義

### 実装の状態

`expense-saas/internal/service/auth_service.go:416-417`:

```go
// (e) MVP: メール送信はログ出力で代替する。
slog.Info("パスワードリセットトークンを生成しました", "email", email, "token", tokenValue)
```

メール送信は実装されておらず、`slog.Info` で平文トークンをサーバーログに出力するのみ。

### 設計成果物・インフラ側の欠落

- ADR にメール送信プロバイダ（AWS SES / SendGrid 等）の選定なし
- `30_arch/architecture.md` にメール送信経路の記載なし（運用通知の SNS → メールのみ）
- `30_arch/external_systems.md` にメール基盤の記載なし

## 影響

- 本番運用ではユーザーがパスワードリセット要求しても何も届かない
- 平文トークンがサーバーログに出力されるため、ログ閲覧者がリセットトークンを取得可能（セキュリティリスク。本番では監査ログ保護が必要）
- 設計書を読んだ第三者（採用ポートフォリオ閲覧者含む）に「メール送信される」と誤認させる

## 提案

**MVP 区分**: post-MVP（ポートフォリオ・デモ用途では現状の動作で十分。本番運用化時に対応）

**対応内容（post-MVP）**:

1. ADR を新規作成（`adr/000X-email-provider.md`）。AWS SES（または Mailpit などの開発時代替）を選定
2. `30_arch/architecture.md` / `30_arch/external_systems.md` にメール送信経路を追補
3. `internal/service/auth_service.go` にメール送信処理を実装（プロバイダ抽象 + 本番 SES / 開発 Mailpit 切替）
4. `slog.Info` でのトークン平文ログ出力を削除（セキュリティリスク解消）

**当面の MVP 対応（このチケットでは何もしない）**:

- 設計書側に「MVP ではログ出力で代替する」旨の注記を追加するか、現状の乖離を記録のみで残すかは別途判断
- 本 issue は post-MVP として棚上げ

## ラベル

- type: gap / scope
- area: backend / architecture / docs
- mvp: post-mvp

## 関連

- 要件: AUTH-F06（requirements.md:46）
- 関連 SMK: SMK-100（パスワードリセット実行画面のリンク確認 — 機能フロー検証として PASS、メール到達は別観点）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
