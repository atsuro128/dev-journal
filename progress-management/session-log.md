# 引き継ぎメモ

## セッション: 2026-03-30 23:04

### ゴール
- 運用プロセスの整備・重複排除・ルール統一

### 作業ログ
- **CLAUDE.md 整備**
  - メモリパス追記（autoMemoryDirectory のバグ回避）
  - 参照先セクション削除（他ファイルで十分参照されている）
- **project-rules.md と workflow.md の重複排除**
  - ブランチ運用セクションを project-rules.md から削除し workflow.md に一本化
- **成果物作成フローの分離**
  - 設計成果物フロー / PR フローをリポジトリ別で判定するルールを追加（dev-journal → 設計成果物、expense-saas → PR）
  - PR フローに codex レビューステップを追加
- **ブランチ戦略の統一**
  - 「1チケット = 1ブランチ = 1PR = スカッシュマージ」に全 Step で統一
  - main 直接コミットの選択肢を廃止（step8, step11 含む）
  - branch-strategy.md、ticket-template.md、各 work-breakdown を更新
  - Step 8 チケット10件のブランチ記載を更新（完了済み: main 直接（済）、未着手: step8/{ID}-{概要}）
- **review-findings 072/073 解消**
  - 072: env_config.md から JWT_PUBLIC_KEY_PREVIOUS の言及を2箇所削除
  - 073: 対応済み確認のみ
  - 両件を resolved に移動
- **計画策定ルールの明文化**
  - feedback_planning_delegation メモリの内容を workflow.md にルール化
- **セッションログアーカイブのフラット化**
  - 日付フォルダ (YYYY-MM-DD/session-log.md) → YYYY-MM-DD.md に変更
  - session-log-archive.md を日付別ファイルに分割して削除
  - 関連スキル4件（session-log, daily-report, self-review, analyze）のアーカイブ参照を更新
- **スキル整理**
  - review スキル削除（未使用）
  - review-findings スキルを最新化（codex 再レビュー依頼・コミット手順・PASS ループ明示）
  - workflow.md のステップ5を /review-findings 参照に簡略化

### 未完了
- なし

### ブロッカー
- なし

### 次にやること
1. 8-7（テスト基盤）着手 — 8-6 完了で依存解消
2. 8-8（CI/CD パイプライン）— 8-2, 8-3 完了で依存解消
3. 8-9（開発者ツール）— 8-2, 8-3 完了で依存解消
4. 8-10（整理）— 依存なし、最後に実施
5. ops-036（LSP連携）/ ops-047（work-breakdown 分割）— 低優先

### 学び・気づき
- **ルールの分岐を避ける**: Step 別・作業特性別に分岐するルールは守り切れない。「全 Step 共通: 1チケット=1ブランチ=1PR=スカッシュマージ」のように統一ルールにする
- **変更の波及先を網羅的に確認する**: ブランチ戦略変更時、workflow.md だけでなく branch-strategy.md・ticket-template.md・各 work-breakdown・チケットまで波及した。ユーザーに指摘されて気づいた箇所もあった
- **フレームワークの汎用性を意識する**: 特定 Step 専用の記載は NG。リポジトリ別判定のように汎用的な基準を設ける

### 意思決定ログ
- **フロー判定はリポジトリ別**: 「設計 or 実装」の判断を指揮役に委ねず、編集対象リポジトリ（dev-journal / expense-saas）で機械的に決まるようにした
- **スカッシュマージに統一**: 1ブランチ連続 PR（マージコミット）も検討したが、ルールの分岐が増える。1チケット1ブランチなら全 Step でスカッシュマージが使える
- **autoMemoryDirectory のバグ回避**: システムプロンプトにデフォルトパスが表示されるバグがある。CLAUDE.md に正しいパスを明記して回避
- **review-findings スキルの存続**: workflow のステップ5だけスキル化するのは一貫性がないが、セッションをまたぐ独立エントリーポイントとして機能するため残置

---

## セッション: 2026-03-30 19:55（前回）

### ゴール
- Step 8 基盤構築の 8-2 〜 8-6 を可能な限り進める

### 作業ログ
- **8-2（バックエンド初期化）**
  - architect で計画策定 → backend-developer で実装
  - 内部レビュー FIX: pool.Close() 二重呼び出し、/server .gitignore 漏れ → 修正 → PASS
  - codex レビュー FIX: 075（JWT鍵供給方法が未固定 — volumes 削除で keys/ がコンテナに届かない）→ docker-compose.yml に keys/ read-only マウント追加 + JWT パスをコンテナ内パスに修正 → .env.example も合わせて修正（1回差し戻し）→ PASS
  - codex 076（go.mod 依存不足）→ 対応不要理由をファイルに記載し pending-review → codex 差し戻し（チケットと成果物の不一致未解消）→ チケット責務欄を修正して委譲先を明示 → resolved
  - 8-2 完了
- **8-3（フロントエンド初期化）**
  - architect で計画策定（8-2 実装中に並行）→ frontend-developer で実装
  - 内部レビュー PASS（warning: vite-env.d.ts 未作成、info: tsbuildinfo gitignore）
  - codex レビュー PASS（同じ2点、品質ゲートに影響なし）
  - vite-env.d.ts 追加 + .gitignore に *.tsbuildinfo 追加（レビュー指摘の軽微対応）
  - 8-3 完了
- **8-4（共通ミドルウェア + ヘルスチェック）**
  - architect で計画策定 → backend-developer で実装（12ファイル）
  - 内部レビュー FIX: blocker 1件（SQL インジェクション — tenant.go で fmt.Sprintf）+ warning 4件 → 修正 → PASS
  - codex レビュー FAIL: blocker 2件
    - 認証必須ルートに Auth 前のレート制限なし → RateLimitByIP をグローバルチェーンに移動
    - Logger が auth/tenant の tenant_id/user_id を取得できない → mutable RequestInfo パターン導入
  - codex 再レビュー（1回目は expense-saas/ にアクセスできず失敗、再実行で成功）→ PASS
  - 8-4 完了
- **8-5（FE-BE連携）**
  - architect で計画策定 → frontend-developer で実装
  - 内部レビュー FIX: blocker 3件（login レスポンスの data ラッパー未考慮、doRefresh も同様、AuthUser 型が openapi と不一致）+ warning 1件（204 No Content 未対応）→ 修正 → PASS
  - codex レビュー FIX: 077（FormData stringify）、078（useAuth 非リアクティブ）、079（ERROR_CODES 不完全）
    - 077: api.post/put で FormData チェック追加 → resolved
    - 078: 対応不要（API クライアント基盤は正常動作。React リアクティブ性は Step 10 の UI 責務）→ codex が妥当と判断 → resolved
    - 079: security.md 全エラーコード追加 → resolved
  - 8-5 完了
- **8-6（コード生成・スケルトン）**
  - architect で計画策定（5フェーズ、約40ファイル）
  - backend-developer で実装（1回目がトークン上限で停止、Phase A + Phase B 途中まで。2回目で残り全て完了）
  - 内部レビュー FIX: blocker 2件（sqlcクエリフィルタ不足、楽観的ロック未反映）+ warning 5件 → 修正 → 再レビューで B-1a（submitter_id フィルタ漏れ）+ W-6（from/to が created_at ベース）残存 → 修正 → PASS
  - codex レビュー FIX: 080（Repository が TenantContext 接続を使わず RLS バイパス）、081（Service interface フィルタ未露出）、082（Workflow updated_at なし）、083（applicant_name フィルタ未反映）→ 修正 → PASS
  - 8-6 完了
- **メモリ保存先の修正**
  - /home/node/.claude/ → /root-project/.claude/memory/ に移動（コンテナリビルドで消えるため）
  - settings.local.json に autoMemoryDirectory 設定追加

### 未完了
- なし

### ブロッカー
- なし

### 次にやること
1. 8-7（テスト基盤）着手 — 8-6 完了で依存解消
2. 8-8（CI/CD パイプライン）— 8-2, 8-3 完了で依存解消
3. 8-9（開発者ツール）— 8-2, 8-3 完了で依存解消
4. 8-10（整理）— 依存なし、最後に実施
5. ops-036（LSP連携）/ ops-047（work-breakdown 分割）— 低優先

### 学び・気づき
- **計画策定は architect に委譲する**: 指揮役が入力資料を読んで理解しようとすると対話が止まる
- **review-findings は起票者（codex/reviewer）が最終判定する**: 指揮役が独断でクローズしない。対応不要でも理由を記載 → pending-review → 同じレビュー主体に判断を委ねる
- **codex の指摘を品質ゲート基準で批判的に評価する**: 形式的な指摘（チケット記述と成果物の不一致だが下流に影響なし）には押し返すべきだった。076 は余計なチケット修正を生んだ
- **環境変数を変更したら全定義箇所を同時更新**: docker-compose.yml の JWT パスを修正したが .env.example を忘れ codex に差し戻し
- **codex の実行環境**: dev-journal/ から実行すると expense-saas/ が見えないことがある。root-project/ から実行すること
- **メモリはプロジェクトディレクトリに保存**: DevContainer ではコンテナリビルドで /home/node/ が消える。settings.local.json の autoMemoryDirectory で設定

### 意思決定ログ
- **8-2〜8-6 は直列実行**: work-breakdown の「逐次作業」方針に従い、main 直接コミットで直列に進めた。依存関係上は一部並行可能だが、docker-compose.yml 等の共有ファイル競合を回避
- **076 対応（go.mod 依存不足）**: 対応不要が妥当だったが、codex に差し戻されてチケット修正で対応。Go の仕組み上 import しない依存は go mod tidy で消えるため、後続タスクで自然に追加される
- **078 対応（useAuth 非リアクティブ）**: 対応不要。API クライアント基盤のトークン管理は正常動作しており、React UI のリアクティブ性は Step 10 の UI 実装責務
- **RateLimitByIP のグローバルチェーン配置**: architecture.md §3.2 の [5] の位置と異なるが、security.md §4.4 に準拠。全リクエストに IP ベース制限を適用し、Auth 前の無効 JWT 大量送信を防止
- **Repository の TenantContext 接続使用**: queries(ctx, pool) ヘルパーで context からコネクション取得、なければ pool フォールバック。認証不要エンドポイント（ヘルスチェック等）では TenantContext が設定されないため
