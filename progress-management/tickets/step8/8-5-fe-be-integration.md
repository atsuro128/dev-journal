# FE-BE 連携

- 担当: frontend-developer
- 依存: 8-3, 8-4
- ブランチ: main 直接（済）
- 出力先: expense-saas/frontend/src/api/, expense-saas/frontend/vite.config.ts
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| アーキテクチャ設計 | deliverables/docs/30_arch/architecture.md | §SPA 配信方式 |
| セキュリティ設計 | deliverables/docs/50_detail_design/security.md | 全体 |

## 責務

- 開発時プロキシ設定（vite.config.ts で /api/* → バックエンドに転送）
- API クライアント基盤（fetch ラッパー、認証ヘッダー自動付与）
- トークンリフレッシュ → リトライ（401受信時）
- 共通エラーハンドリング
- 含めない: 個別 API の呼び出し実装（Step 10）

## 完了条件

- フロントエンドからヘルスチェックエンドポイントを呼び出してバックエンドと疎通できる
- 401 受信時にリフレッシュ → リトライが動作する
