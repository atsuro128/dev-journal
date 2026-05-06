# 横断テスト（Go）

- 担当: test-implementer
- 依存: 11-A
- ブランチ: `step11/11-B-cross-cutting-test`
- 出力先: expense-saas/internal/handler/cross_cutting_test.go, rate_limit_test.go
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テスト戦略 | deliverables/docs/60_test/test_strategy.md | §4.5（テナントBフィクスチャ） |
| 横断テストケース | deliverables/docs/60_test/test_cases/cross-cutting.md | §1（CRS-001〜CRS-016 テナント分離、CRS-010b 含む）、§2（CRS-021〜CRS-054 RBAC、添付 preview エンドポイント含む）、§4（CRS-076〜CRS-088 非機能） |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | RBAC マトリクス |
| セキュリティ設計 | deliverables/docs/50_detail_design/security.md | レート制限 |
| 全機能の実装コード | expense-saas/internal/ | ハンドラ・サービス・リポジトリ |

## 責務

- テナント分離テスト（CRS-001〜CRS-016、CRS-010b 含む）: 全リソース x テナント越境 = 404 の検証。添付エンドポイントは download / preview の 2 本（CRS-010 / CRS-010b）に分かれる点に注意
- RBAC マトリクステスト（CRS-021〜CRS-054）: 全エンドポイント x 全ロールの認可検証。機能別テストと重複するケースはクロス参照のみ（cross-cutting.md 実装ガイド参照）
  - 添付 preview エンドポイント（`getAttachmentPreview`）は cross-cutting.md §2 RBAC マトリクスに記載済みで、`getAttachmentDownload` と同一の認可ルール（authz.md §10 準拠）
- 非機能テスト（CRS-076〜CRS-088）: レート制限・レスポンスタイムの検証（ローカル実行、E2E と同タイミング）
- テナントBフィクスチャの投入（test_strategy.md §4.5 参照）
- 含めない: E2E テスト（11-C で対応）、機能別テストの修正

## 進め方

1. `cross-cutting.md` の CRS-001〜CRS-016（CRS-010b 含む）/ CRS-021〜CRS-054 / CRS-076〜CRS-088 を読み、既存の機能別テストでカバー済みのケースを洗い出す
2. テナントA・テナントB・各ロールのテストフィクスチャを確認し、不足があれば追加する
3. テナント分離テストを先に実装し、他テナントリソース参照が 404 になることを確認する
4. RBAC テストを実装し、許可・禁止の期待ステータスが `authz.md` と一致することを確認する
5. レート制限・レスポンスタイムなどの非機能テストを実装する
6. Go テストをローカルで実行し、失敗・未実装・仕様不一致を分類して記録する

## 実施ログ

| 区分 | 対象 | 結果 | メモ |
|------|------|------|------|
| テナント分離 | CRS-001〜CRS-016（CRS-010b 含む） | 未実施 | 添付は download / preview の 2 件 |
| RBAC | CRS-021〜CRS-054 | 未実施 | 機能別テストとの重複はクロス参照。preview エンドポイントは download と同一認可ルール |
| 非機能 | CRS-076〜CRS-088 | 未実施 | ローカル実行（E2E と同タイミング） |

## 11-D への引き継ぎ

- 機能別テストでカバー済みとして 11-B では実装しなかった CRS ID
- 設計と実装で期待ステータスが一致しなかった箇所
- セキュリティ上の判断が必要な箇所（404 / 403 の使い分け、レート制限条件など）
- テスト実装時に仕様の曖昧さが残った箇所

## 完了条件

- CRS-001〜CRS-016（テナント分離。CRS-010b 添付プレビュー越境テストを含む）の全テストケースが実装されローカルで通過している
- CRS-021〜CRS-054（RBAC）のうち、機能別テストでカバーされないケースが実装されローカルで通過している（添付 preview エンドポイントの RBAC は cross-cutting.md §2 マトリクスに従う）
- CRS-076〜CRS-088（非機能）の全テストケースが実装されローカルで通過している（レート制限・レスポンスタイムは E2E と同タイミングでローカル実行）

## 更新履歴

- 2026-04-12: 初版起票（commit `a2d642d`）
- 2026-05-06: 上流成果物（cross-cutting.md / authz.md）の更新に追従（commit `a2a3b47`）
  - 添付エンドポイント分割（download / preview）に伴う CRS-010b 追加を反映
  - 添付 preview エンドポイントの RBAC マトリクス記載が cross-cutting.md §2 に存在することを確認しクロス参照のみとした（チケット側に追加実装ガイドの記述は不要）
  - 非機能テストの実行環境を「CI」→「ローカル実行（E2E と同タイミング）」に統一
