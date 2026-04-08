# DevContainer の egress allowlist 選定根拠と検証厳格性に不整合がある

## 発見日
2026-04-08

## カテゴリ
security

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
DevContainer 運用

## ブロッカー
なし

## 問題

`.devcontainer/proxy-allowlist.txt` には `chatgpt.com`、`platform.claude.com`、`sentry.io`、`statsig.*` などのホストが含まれているが、それらの必要性の強さが文書間で揃っていない。

- `dev-journal/references/devcontainer-docs/devcontainer-design.md` では `chatgpt.com` を OpenAI の必須到達先として扱っている
- `.devcontainer/proxy-allowlist-rationale.md` では `chatgpt.com` と `platform.claude.com` は `要確認`、`sentry.io` と `statsig.*` は telemetry / feature flag の可能性として整理されている
- `.devcontainer/verify-egress.sh` は一時的に一部ホストを optional 扱いに変更されていた

この状態では、allowlist に残す基準と起動時検証の基準が一致せず、「必須だから許可しているのか」「念のため許可しているのか」が曖昧になる。

## 影響

- DevContainer の外向き通信境界に対する説明責任が弱くなる
- 必須通信先と任意通信先の区別が曖昧になり、不要な許可先が残りやすくなる
- 起動時検証を strict にすべきか warn-only にすべきかの判断が属人的になる

## 提案

allowlist 採用基準と verify-egress の判定基準を統一する。候補は以下。

1. allowlist に残すホストはすべて必須扱いとし、到達確認も strict にする
2. 必須と説明できないホストは allowlist から削除し、必要性が確認できたものだけ残す

少なくとも次を対象に再判定する。
- `chatgpt.com`
- `platform.claude.com`
- `sentry.io`
- `statsig.anthropic.com`
- `statsig.com`

判定後は次の 3 点を同期する。
- `.devcontainer/proxy-allowlist.txt`
- `.devcontainer/proxy-allowlist-rationale.md`
- `dev-journal/references/devcontainer-docs/devcontainer-design.md`

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
