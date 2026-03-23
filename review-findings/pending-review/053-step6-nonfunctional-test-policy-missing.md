# 053: 非機能要件のテスト方針を Step 6 で対象外にしており上流要件と矛盾

## 指摘概要
`test_strategy.md` は「レスポンスタイム・レート制限のテストは対象外」と明記しているが、Step 6 のレビュー観点では非機能要件へのテスト方針を含めることが必須であり、要件定義・セキュリティ設計でも具体値まで確定している。方針を Step 7 へ先送りすると、性能・レート制限をどのレベルでどう検証するかが未設計のまま実装に進む。

## 根拠
- `dev-journal/guide/work-breakdown/step6-testing.md:46`-`50`
  - Step 6 reviewer 観点として「非機能要件（レスポンスタイム、レート制限）に対応するテスト方針が含まれているか」を要求している。
- `dev-journal/deliverables/docs/10_requirements/requirements.md:400`-`401`
  - API レスポンスタイム p95 500ms 以下、5MB ファイルアップロード 5 秒以下を非機能要件として定義している。
- `dev-journal/deliverables/docs/10_requirements/requirements.md:411`-`423`
  - レート制限を SEC-012 として要求している。
- `dev-journal/deliverables/docs/50_detail_design/security.md:297`-`351`
  - 認証済み 100 req/min、未認証 20 req/min、ログイン 5 req/min/IP、アップロード 10 req/min/user まで具体設計済みである。
- `dev-journal/deliverables/docs/60_test/test_strategy.md:88`-`90`
  - 「MVP ではレスポンスタイム・レート制限のテストは対象外」としている。

## 判定
- 重大度: 中
- 分類: 上流整合性欠落 / テスト方針不足

## 修正方針案
- Step 6 に非機能テストを実装レベルまで詳細化する必要はないが、少なくともテストレベル・実施タイミング・合否基準は定義する。
- 例として、レート制限は統合テストで 429 とヘッダーを確認し、レスポンスタイムは CI の軽量性能スモークまたは定期ジョブで p95 を監視する、という形で方針を明記する。
- `cross-cutting.md` または `test_strategy.md` に、対象エンドポイント、試験条件、期待値、実施タイミング（PR / main / nightly）を追加する。

## 対応内容

test_strategy.md §2.3 を全面書き換え。「対象外」を削除し、以下を定義:
- レート制限テスト: 統合テスト、PR時実行、security.md の4種制限値（100/20/5/10 req/min）を明記、429+Retry-After確認、記載先は cross-cutting.md
- レスポンスタイムテスト: 統合テスト（軽量スモーク）、mainマージ後実行、requirements.md の期待値（p95 500ms、5MB 5秒）を明記
- CI組み込み方針（§5.1）にも非機能テストのタイミングを追記
