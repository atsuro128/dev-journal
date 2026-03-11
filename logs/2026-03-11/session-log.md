# 2026-03-11 セッションログ

## 17:00 セッション
- 作業: Dev Container 設定ファイル3点（devcontainer.json, Dockerfile, init-firewall.sh）を `.devcontainer/` に作成
- 作業: 公式 Claude Code devcontainer をベースに Rust ツールチェーン・PostgreSQL クライアント・Python を追加
- 作業: ファイアウォール許可ドメインに crates.io 関連4ドメインを追加
- 作業: ADR-003（Dev Container 導入の計画・判断記録）を dev-journal に作成
- 作業: MEMORY.md の開発環境セクションを Dev Container 構成に更新
- 判断: `.devcontainer/` は git 追跡対象とする（理由: Dev Container の目的は環境再現性の共有。ポートフォリオとしてもIaCを示せる）

## 17:30 セッション
- 修正: VSCode で devcontainer を開く際に postStartCommand (init-firewall.sh) がエラーで失敗する問題を修正
- 原因: 複数ドメインが同一IPに解決される場合、`ipset add` が重複エラーを出し、`set -e` でスクリプトが即終了していた
- 対応: `ipset add` に `-exist` フラグを追加し、重複時にエラーにならないよう修正（2箇所）

## 18:45 セッション
- 修正: 前回の `-exist` フラグ追加だけでは解決しなかった postStartCommand エラーの再修正
- 原因: marketplace.visualstudio.com が同一IP（150.171.73.16）を複数回返し、ipset v7.17 で `-exist` フラグが期待通り動作せず重複エラーが発生
- 対応1: DNS結果を `sort -u` で重複排除し、`ipset add` に `|| true` を安全策として追加
- 対応2: devcontainer.json の postStartCommand 内の em dash（U+2014）を ASCII `--` に変更（非ASCII文字によるシェルパース問題を排除）

## 19:21 セッション
- 作業: Dev Container 環境の動作確認（DEVCONTAINER=true、各ツールバージョン、サブリポジトリのリモート設定を確認）
- 修正: コンテナ内のワークスペースパスを `/workspace` から `/root-project` に変更（Dockerfile・devcontainer.json）
- 判断: マウント先パスをリポジトリ名と一致させる（理由: `/workspace` は汎用的すぎてプロジェクトの実態と乖離しており、`root-project` に合わせることで一貫性を保つ）

## 20:30 セッション
- 作業: Dev Container 接続不可の調査。コンテナが存在しない状態だったが、イメージは健全。手動起動・ファイアウォールスクリプトとも正常動作を確認
- 議論: 技術スタックの見直し。Rust 未経験 + 1ヶ月目標 + 自社開発 SaaS 狙いの観点から、Rust → Go への変更を検討
- 判断: バックエンドを Rust から Go に変更する方針で合意（理由: 学習コスト、転職市場の需要、ポートフォリオ完成リスクの総合判断）
- 作業: Issue #018（tech-stack-change-rust-to-go）を起票

## 23:00 セッション
- 作業: Issue #018 の実装 — 全リポジトリのバックエンド技術スタック記述を Rust から Go に一括変更
- 作業: root-project — CLAUDE.md, AGENTS.md, Dockerfile（Go 1.24.1 + golangci-lint）, devcontainer.json, init-firewall.sh（proxy.golang.org）, coding-standards.md, security-policy.md, testing.md, settings.json を更新
- 作業: dev-journal — portfolio_project_steps.md, implementation-guide.md（全面書き換え）, dev-commands.md, directory-structures/expense-saas.md, links.md, 01_glossary.md, PROJECT_SUMMARY.md を更新。Issue #018 を resolved/ に移動
- 作業: ai-dev-framework — AGENTS.md, README-template.md を更新
- 作業: expense-saas — .gitignore を Go 向けに変更、Cargo.toml を削除
- 作業: stop-check hook のブロック時にもログファイルへ記録するよう修正
- 作業: Stop hook の動作調査 — hook は正常に発火・ブロックしていることを確認。ループ回避スキップが多発していた原因はブロック後にコミットアクションが取られなかったため
- 判断: 具体的なフレームワーク・ライブラリ選定（Echo/Chi, sqlc/pgx 等）は Step 3 ADR に委ねる（理由: このIssueは「Go への汎用的な更新」がスコープであり、詳細選定は別ステップの責務）
