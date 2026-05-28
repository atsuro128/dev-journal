# 解決済み issue アーカイブ

progress.md から退避した解決済み issue のテーブルを保持する。

## 解決済み（2026-04-22）
| ID | タイトル | 解決 PR | マージ commit |
|----|---------|---------|--------------|
| 134 | FE エラーハンドラのハードコード文言を err.message ベースに統一し、原因究明と再発防止を行う | PR #84 | 2ab3926 |
| 131 | AttachmentUploader のクライアントサイドバリデーションエラー表示を整理（MUI Alert + 文言整合） | PR #85 | e06a521 |
| 130 | 明細詳細パネル閉じ時のスライドアニメーション中にレイアウトが切替 | PR #86 | e73a659 |
| 129 | 新規追加モードの添付 UX を編集モードと同等にする（プレビュー対応 + ラベル整理） | PR #87 | 8dd13f5 |
| 132 | 明細保存成功後も beforeunload リスナが残存し、F5 でリロード確認ダイアログが表示される | PR #88 | e7b8a59 |
| 138 | スマホ幅（375px）でレポート作成フォームが横幅に収まらない（ReportPeriodField の flex 横並び） | PR #89 | 1369e50 |
| 135 | ReportDetailPage 削除/提出/承認等アクション系エラーが画面に表示されない（itemApiError の表示経路の欠落） | PR #90 | 72f0df5 |
| 136 | 明細編集の変更破棄ダイアログが ConfirmDialog 未使用で UI 不統一（共通コンポーネントに寄せる） | PR #91（dev-journal: 049c128） | 2d1b1ed |
| 137 | スマホ幅（375px）でレポート一覧・明細一覧ページがページ全体横スクロールになる（TableContainer 未使用） | PR #92（#137 + #139 合算） | b7f4430 |
| 139 | レポート一覧が設計済み ReportListTable / StatusChip を使わず status 英語生値を露出（SCR-RPT-001 実装乖離） | PR #92（#137 + #139 合算） | b7f4430 |

## 解決済み（2026-04-16）
| ID | タイトル | 解決 PR |
|----|---------|---------|
| 113 | architecture.md ディレクトリツリー粒度不適切 | dev-journal コミット |
| 104-B | UI カバレッジギャップのテスト追加 | PR #63 |
| 101 | SMK-037 文言不一致 | 既存コミットで対応済み |
| 105 | smoke_check URL パス不一致 | 既存コミットで対応済み |
| 111 | devcontainer golangci-lint v1→v2 不整合 | root-project コミット |
| 109 | テスト設計書・実装間の乖離 | PR #60 + #61 |
| 110 | architecture.md ディレクトリツリー陳腐化 | dev-journal コミット |
| 112 | domain パッケージ DTO 分離 | PR #62 |
| 104-A | UI カバレッジリサーチ | dev-journal コミット |
| 096 | キャッチオール 404 ルート未実装 | PR #59 |
| 100 | AttachmentUploader UI 課題（重複ボタン + スピナー + DnD 視覚化） | PR #58 |

## 解決済み（2026-04-15）
| ID | タイトル | 解決 PR |
|----|---------|---------|
| 097 | AppSelect の outlined 切り欠きと InputLabel 位置不整合 | PR #55 |
| 098 | ItemSlidePanel UX 不整合 4 件（幅・閉じるボタン・アニメ・閲覧モード） | PR #56 |
| 099 | useDeleteAttachment の invalidation が useAttachments を対象外 | PR #57 |
| 103 | 添付削除時の確認ダイアログ欠落 | PR #57 |
| 106 | Workflow 2 ページの同期ロールチェック欠落（ロール変更時フラッシュ表示）| PR #54 |

## 解決済み（2026-04-14）
| ID | タイトル | 解決 PR |
|----|---------|---------|
| 085 | ダッシュボード「却下」カード Badge dot 意図不明 | PR #52 |
| 086 | 承認待ち/支払待ちカードのスタイル不統一 | PR #52 |
| 087 | Admin ダッシュボード月別合計 0 円（seed データ 3 重欠陥） | PR #50 |
| 088 | 403 認可エラー時のフィードバック欠落（SMK-007/025 FAIL） | PR #53 |
| 089 | 明細フォーム カテゴリラベル重複 | PR #51 |
| 090 | 明細フォーム プリフィル未実装（blocker） | PR #51 |
| 091 | 明細行クリック挙動未定義（閲覧モード時の添付操作禁止） | PR #51 |
| 092 | ItemSlidePanel が Drawer 未使用（スライドしない） | PR #51 |
| 093 | 添付一覧のサムネ/アイコン要件が smoke_check.md のみ存在 | PR #52（smoke_check.md 修正） |
| 094 | 添付ファイルサイズが生バイト数表示 | PR #52 |
| 095 | S3 署名付き URL の minio 内部ホスト名問題（blocker） | PR #49 |

## 解決済み（2026-04-13）
| ID | タイトル | 解決 PR |
|----|---------|---------|
| 083 | 認証トークンを sessionStorage に永続化（F5 リロードでログアウトしないように） | PR #48 |
| 082 | 認証フロー UX バグ 2 件（AppLayout 未適用 + ログイン 401 リダイレクト誤動作） | PR #47 |

## 解決済み（2026-04-12）
| ID | タイトル | 解決 PR |
|----|---------|---------|
| 070 | App.tsx の LoginPage import がスケルトンを参照 | PR #42 |
| 071 | pages ディレクトリ構成が3パターン混在 | PR #42 |
| 072 | App.tsx に管理系画面のルートが未登録 | PR #43 |
| 074 | 11-A ローカル動作確認のチェックリストが存在しない | Step 6-D 追加 |
| 065 | 認証系エンドポイントで expense_owner 専用 DB 接続が未使用 | PR #35 |
| 066 | フィルタ UI で AppSelect が使われていない | PR #37 |
| 067 | auth store に currentUser を保持しており state-management.md と乖離 | PR #36 |
| 068 | 一部ページで素の HTML 要素（button / table）が残存 | PR #34 |
| 069 | 添付ファイルアップロード後の invalidation が設計テーブルと乖離 | — |
| 076 | Step 8 ローカル開発環境統合の漏れ（シード・migration・MinIO・README） | PR #44（8-11） |

## Step 11 関連（resolved 退避、2026-05-28 アーカイブ）

| ID | タイトル | 解決 | 解決日 |
|----|---------|------|-------|
| 165 | マイレポート画面のフィルタエリアがスマホ幅で横一列のまま画面幅を超える | PR #135 + #136 + #137 マージ済み、レイアウト統一 + 改行位置調整 + デグレ修正 | 2026-05-06 |
| 170 | 「保存して続けて追加」ボタン押下で明細が 2 件登録される（データ整合性 blocker） | PR #132 マージ済み | 2026-05-05 |
| 171 | 認証フォームのメールアドレス欄に autocomplete 属性が欠落 | PR #133 + PR #134 マージ済み、Edge での手動確認 PASS | 2026-05-05 |
| 172 | ログインエンドポイントのレート制限値を環境変数で上書き可能にする（11-C E2E テストブロッカー） | PR #140 マージ済み、本番値 5/20 維持・dev/E2E 100/200 緩和 | 2026-05-06 |
| 173 | JWT 検証にクロックスキュー許容（leeway）を追加（CRS-066 真因 = Docker クロックドリフト） | PR #141 マージ済みで domain.JWTVerifier に leeway 適用、PR #143 マージ済みで pkg/jwt.Verifier (middleware 経路) にも適用、両系統を 11-D 再レビューで確認 | 2026-05-07 |
| 175 | deploy.yml の e2e ジョブが Playwright プロジェクト構成と不整合（5 件、11-E 着手前修正） | PR #144 マージ済み、5 件不整合 + healthz→health bug の 2 コミットで対応、codex 再レビュー PASS | 2026-05-08 |
| 181 | RDS backup_retention_period が Free Tier 制約により NFR-AVAIL-003 から逸脱（7 日 → 1 日） | PR #146 + PR #147 マージ済み、設計修正 dev-journal f10f88a + 解決移動 d175c83 で NFR を portfolio 仕様の 1 日保持に緩和、案 3 採用 | 2026-05-19 |
| 183 | SPA embed 配信が未実装（architecture.md §4.0 と実装の乖離、Phase 7 ブロッカー） | PR #148 マージ済み、A 案 go:embed 方式で実装。Dockerfile に node build stage 追加 + cmd/server/embed.go + internal/spa パッケージ。内部レビュー 3 ラウンドで blocker 3 件解消 + codex レビュー PASS | 2026-05-20 |
| 184 | SPA fallback ハンドラが HEAD リクエストで 405 を返す（GET のみ登録） | PR #150 マージ済み、master dd480b5。SPA fallback を GET+HEAD 登録、HEAD /health はヘルスハンドラ経由化し index.html 誤返却を回避、回帰テスト 5 件追加。内部レビュー + codex PASS | 2026-05-20 |
| 185 | ALB が HTTP 平文で HTTPS 未対応（security.md §11 と乖離） | Step 11-E 実 apply で T4（CORS/env 値反映 → CloudFront ドメイン）+ T6（疎通・CORS・HSTS・レート制限・B-1-b 2 層目）全消化。本 issue 派生で副次 #189 起票 → PR #154 で fix | 2026-05-26 |
| 186 | 設計書のコンピュート層表記が ECS Fargate のまま（実装は EC2 t3.micro 単一） | 案D「EC2 を正本、Fargate は ADR-0004 だけに残す」で対応完了。Phase 0 設計確定 + Phase 1-4 designer 5 並列 + Phase 4 diagrams シリアル + codex 2 ラウンド経て PASS。11 ファイル EC2 化、ユーザー判断 UD-1〜7 確定 | 2026-05-25 |
| 187 | terraform/ec2.tf: SSM Session Manager + SSM Parameter Store 対応 | PR #152 マージ済み squash `445240b`。AWS 実 apply 系の受け入れ基準は Step 11-E 実 apply セッションで全消化 | 2026-05-25 |
| 188 | 監視設計のロググループ命名残置をリネーム（monitoring.md + 実装側 docker awslogs-group 設定の同期更新） | PR #153 マージ済み squash `420f0ea`。当初「リネームのみ」から awslogs 完全実装スコープに拡大、AWS 実環境疎通検証は Step 11-E で消化 | 2026-05-25 |
| 190 | cloudfront.tf: default_root_object="index.html" が SPA fallback と組み合わさり / で無限リダイレクトループ（11-F UAT 着手ブロッカー） | PR #155 マージ済み squash `51f93fb`、reviewer + codex PASS。原因: CloudFront default_root_object が `/index.html` にリライト → Go FileServer が canonical 化で 301 → 無限ループ。修正: `default_root_object` 削除（1 行）、terraform apply で CloudFront in-place 変更 | 2026-05-26 |
