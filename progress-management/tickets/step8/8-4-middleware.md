# 共通ミドルウェア + ヘルスチェック

- 担当: backend-developer
- 依存: 8-2
- ブランチ: main 直接（済）
- 出力先: expense-saas/internal/middleware/, expense-saas/internal/handler/health.go
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | §3.2 ミドルウェアチェーン |
| 認可設計 | deliverables/docs/50_detail_design/authz.md | §5.1 ルートグループ定義 |
| セキュリティ設計 | deliverables/docs/50_detail_design/security.md | セキュリティヘッダー、認証フロー |
| 監視設計 | deliverables/docs/50_detail_design/monitoring.md | §2 構造化ログ必須フィールド |

## 責務

- ミドルウェアチェーン実装（architecture.md §3.2 の順序）:
  [1] CORS、[2] SecurityHeaders、[3] RequestID、[4] Logger、[5] RateLimit、[6] Auth（JWT検証）、[7] TenantContext（RLS設定）、[8] RBAC
- API エラーレスポンス形式の確定（architecture.md §3.5）
- ヘルスチェックエンドポイント（DB接続ステータス含む）
- 認証不要/認証必須のルートグループ定義（authz.md §5.1）
- 含めない: 個別機能のハンドラ（8-6）

## 完了条件

- ミドルウェアが architecture.md §3.2 の順序で適用される
- ヘルスチェックが DB ステータスを返す
- 認証不要ルート（login, signup, health 等）がスキップされる
