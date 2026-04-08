# DevContainer の mount と secret 注入が必要最小限になっていない

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

`.devcontainer/devcontainer.json` では、workspace に加えて Claude/Codex 設定 volume、`gh` 設定 volume、ホスト `.gitconfig` の bind mount、upstream proxy 用の認証関連環境変数をコンテナへ注入している。

現状の利用前提は「個人専用で、他人の repo は扱わない」ため直ちに危険とは言えないが、DevContainer 内から参照可能な高価値情報が開発に必要な最小範囲を超えている。

特に次の点は見直し余地がある。

- `gh-config-*` を常設 mount している
- ホスト `.gitconfig` を bind mount している
- `UPSTREAM_PROXY_PASSWORD` を環境変数で注入できる
- `.claude` / `.codex` の永続 volume を通常用途と高警戒用途で分離していない

## 影響

- DevContainer 内で参照可能な秘密情報の範囲が広がる
- firewall / allowlist を厳しくしても、コンテナに持ち込んだ認証情報自体は保護対象外になる
- 将来、検証用 repo や一時的な外部コードを扱う際に、現在の利便性寄り設定をそのまま流用してしまうリスクがある
- `UPSTREAM_PROXY_PASSWORD` を環境変数で扱うことで、secret の露出経路が増える

## 提案

mount と secret 注入を「常設するもの」と「必要時のみ持ち込むもの」に分離する。

最低限、次を検討する。

1. `UPSTREAM_PROXY_PASSWORD` を廃止し、`UPSTREAM_PROXY_PASSWORD_FILE` に統一する
2. ホスト `.gitconfig` の bind mount をやめ、必要最小限の Git 設定のみ `postCreateCommand` で生成する
3. `gh-config-*` は常設 mount ではなく、GitHub CLI が必要なときだけ有効化する
4. `.claude` / `.codex` は現行の常設プロファイルに加えて、認証情報を分離した高警戒プロファイルの要否を判断する

判断後は次の資料を同期する。

- `.devcontainer/devcontainer.json`
- `.devcontainer/SECURITY.md`
- `dev-journal/references/devcontainer-docs/devcontainer-design.md`

補足:
既存 issue `060-devcontainer-egress-allowlist-strictness-and-rationale-gap` は egress allowlist の妥当性と検証方針の問題であり、本 issue の mount / secret 最小化とは論点と対処が異なるため統合しない。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
