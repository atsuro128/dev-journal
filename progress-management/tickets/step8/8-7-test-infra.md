# テスト基盤

- 担当: backend-developer
- 依存: 8-6
- 出力先: expense-saas/internal/testutil/
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | テスト基盤の方針 |
| DB スキーマ | deliverables/docs/50_detail_design/db_schema.md | テストデータ構造 |
| テスト戦略 | deliverables/docs/60_testing/ | テスト方針 |

## 責務

- テスト用 DB セットアップ/クリーンアップ関数
- フィクスチャファクトリ（テナント、ユーザー、経費レポート、明細、添付ファイル）
- HTTP テストヘルパー（認証付きリクエスト発行、レスポンス検証）
- 含めない: 個別機能のテストケース（Step 9）

## 完了条件

- テスト基盤のコードがコンパイルできる
- Step 9 がテスト基盤を使って即座にテストコード実装を開始できる
