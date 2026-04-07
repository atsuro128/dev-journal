# 086: （タイトル：マージ後ワークフローに E2E / スモークの枠がない）

## 指摘概要
Step 8-9 は「デプロイ後のワークフロー定義（E2E/スモークは枠のみ）」を成果物に含める契約だが、`deploy.yml` には lint / test / build と Docker/ECS のコメントアウト枠しかなく、main マージ後に走る E2E / レスポンスタイムスモークのジョブ定義が存在しない。テスト戦略とチケット責務が CI 実装に反映されていない。

## 根拠
- `dev-journal/progress-management/tickets/step8/8-9-cicd.md:17-19`
  - 「デプロイ後のワークフロー定義（E2E/スモークは枠のみ）」を責務に含めている。
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md:179`
  - レビュー観点として「デプロイ後のワークフロー定義が存在するか（E2E/スモークは枠のみで可）」を要求している。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:218-221`
  - main ブランチへのマージ後に「E2E テスト + レスポンスタイム スモークテスト」を実行し、その YAML 具体設計は Step 8 で行うとしている。
- `expense-saas/.github/workflows/deploy.yml:1-158`
  - 定義されているのは lint / test / build と Docker/ECS のコメントアウト枠であり、E2E / smoke に対応するジョブ名やコメント枠が存在しない。

## 判定
中 / FIX

## 修正方針案
- `deploy.yml` に、main マージ後フローの末尾として `e2e` / `smoke` のプレースホルダジョブを追加する。
- まだ実装しない場合でも、トリガー条件・依存関係・将来の実行コマンドが分かる枠を YAML 上に固定する。

## 解決内容
- `deploy.yml` に `e2e`（Playwright）と `smoke`（レスポンスタイム）のプレースホルダジョブを `if: false` で追加
- コメントアウトではなく実ジョブとして定義し、有効化時に `if: false` を削除するだけで動作する設計

## 解決日
2026-03-31
