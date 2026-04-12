# 設計書で定義された S3_PRESIGNED_URL_EXPIRY が実装で環境変数参照されていない

## 発見日
2026-04-12

## カテゴリ
implementation

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 5（詳細設計）、Step 7（運用設計）、Step 8（基盤構築）、Step 10-G（添付ファイル機能）

## ブロッカー
なし

## 問題（起票時の認識）

`70_operations/env_config.md` §4.4 で環境変数 `S3_PRESIGNED_URL_EXPIRY=15m`（dev/stg/prod 共通）が定義されているが、実装コード側でこの環境変数を参照している箇所が存在しない。

- 実装: `internal/service/attachment_service.go:190` で `s.storage.PresignGetObject(ctx, att.S3Key, att.FileName, string(att.MimeType), attachmentDownloadExpiry)` と呼び出している
- `attachmentDownloadExpiry` は Go 定数（ハードコード値）であり、環境変数から設定できない
- `internal/pkg/s3/client.go` の `PresignGetObject` は `expiry time.Duration` を引数で受けるだけで、`os.Getenv("S3_PRESIGNED_URL_EXPIRY")` を読んでいない

結果として、設計書定義された環境変数が実装で無視されており、運用環境ごとに署名付き URL の有効期限を変更できない。

## 影響（起票時の認識）

- dev/stg/prod で URL 有効期限を環境変数で切り替えられない（現状は全環境で同じ定数）
- 設計書（env_config.md）と実装の乖離が残り、設計書を信頼できなくなる
- 将来、セキュリティ要件で有効期限を短縮する場合、コード変更＋再デプロイが必要

本問題は Step 11-A ローカル動作確認のブロッカーではない（既存の定数ベースで動作する）。Step 8-11 対応 PR #44 のレビュー時に発覚。

---

## 結論: 誤起票（2026-04-13 判明）

本 issue は **誤起票** であり、実装は元々正しかった。誤っていたのは `env_config.md §4.4` の記載のほうだった。

### 根本原因の再調査結果

本 issue の対応着手時に PR #45 を作成したが、codex レビューで **上流の正本が全て「15 分固定」で一貫している** ことが指摘され、調査の結果以下が判明した。

| 層 | ファイル | 記述 |
|----|---------|------|
| Step 1 要件定義 | `requirements.md:315` | **`ATT-012 \| 署名付きURLの有効期限: 15分`** — 要件レベルで固定 |
| Step 1 ポリシー | `policies.md:574` | `ATT-012 \| 署名付きURLの有効期限は短時間とする` |
| pre_step ビジネスルール | `pre_step/04_business-rules.md:96` | `ATT-012 \| 署名付きURLの有効期限は短時間（例: 15分）\| セキュリティ` |
| Step 3 ADR | `30_arch/adr/0004-infra.md:95` | `S3: 署名付きURL（15分有効、ATT-012）によるアクセス制御` |
| Step 5 詳細設計 | `files.md:332` | `\| 有効期限 \| 15分（ATT-012） \| time.Duration(15 * time.Minute) \|` |
| Step 5 詳細設計 | `files.md:598, 657` | 「有効期限は 15 分（ATT-012）」を複数箇所で明記 |
| Step 5 API 契約 | `openapi.yaml:1271, 1288` | `x-requirements: - ATT-012 # 署名付きURL有効期限 15分` |
| Step 5 画面仕様 | `screens/report-detail.md:361` | `\| 有効期限 \| 15分（ATT-012） \|` |
| Step 6 テスト設計 | `test_cases/attachments.md:164` | **`ATT-034 TestGetAttachmentDownload_ExpiresAt_15min`** — `data.expires_at が現在時刻から15分後` を完了条件として固定 |
| Step 10-G 実装 | `attachment_service.go` | 定数 `attachmentDownloadExpiry = 15 * time.Minute` — 上記正本に正しく準拠 |

つまり **Step 1 要件 → Step 3 ADR → Step 5 詳細設計 → Step 6 テスト設計 → Step 10-G 実装まで全て「15 分固定」で完全に一貫していた**。

唯一の逸脱は **Step 7 運用設計の `env_config.md §4.4` のみ**。運用設計新設時（`0b13c33 refactor: ops-040 Phase 4 成果物修正 — ... 70_operations/ 新設: runbook/release/backup_restore/env_config（実運用レベル）`、2026-03-26）に、環境変数一覧を統一フォーマットで網羅列挙する過程で、要件の固定値であることを見落として機械的に `S3_PRESIGNED_URL_EXPIRY=15m` を追加してしまった。dev/stg/prod 全環境で `15m` であることからも、運用設計者自身も可変化の意図は持っていなかったことが窺える。

### 本プロジェクトにおける可変化の実益評価

15 分固定を覆して環境変数化するメリットを検討した結果、本プロジェクトでは実益が薄いと判断した:

- 対象ユーザー: 社内メンバー（Member / Approver / Accounting / Admin）のみ、外部公開なし
- 対象ファイル: 領収書画像（数 MB、ダウンロード数秒で完了）
- 典型的な利用フロー: ユーザーが画面でダウンロードボタンを押してから完了まで 1 分以内
- 15 分は実使用時間に対して 15 倍の余裕があり、短すぎて困るシナリオはない
- 環境ごとに値を変える必要もない（dev/stg/prod 全て同じ）
- 将来セキュリティ要件で短縮が必要になった場合、定数 1 行変更 + 再デプロイ（数分）で対応可能

### 起票ミスの原因

- 起票時（PR #44 レビュー中）に `env_config.md §4.4` の記載と実装コードの乖離だけを見て結論を出した
- 要件 ID `ATT-012` が全文検索すれば特定できる状況だったにもかかわらず、上流の正本（`requirements.md`, `files.md`, `openapi.yaml`, `attachments.md`, ADR）を確認しなかった
- 本 issue の対応着手時にも同じ過ちを繰り返し、Explore エージェントには「実装現状調査」のみ依頼して上流確認を怠った
- PR #45 作成・reviewer 通過後、codex が上流確認したことで初めて誤りに気付いた

## 解決内容

**採用方針**: 方針 C（巻き戻し）

### 実施内容
1. **PR #45 クローズ**: https://github.com/atsuro128/expense-saas/pull/45 を誤起票として close、ブランチ `step10/issue-077-s3-presigned-url-expiry-env` を削除
2. **設計書修正**: `env_config.md §4.4` の `S3_PRESIGNED_URL_EXPIRY` 行を削除（要件 ATT-012 で固定値であり、環境変数化しないことを明確化）
3. **実装**: 無変更（元々正しかった）
4. **memory 追加**: `feedback_issue_upstream_check.md` を新規作成し、issue 起票・対応着手時の上流確認を義務化

### 残存する派生問題
本件で `env_config.md §4.x` の信頼性疑義が強まったため、全変数の棚卸し issue を別途起票する（issue 079）。
- PR #44: `S3_BUCKET_NAME` 誤記
- issue 078: `S3_REGION` vs `AWS_REGION` 乖離（resolved）
- 本 issue 077: `S3_PRESIGNED_URL_EXPIRY` 誤記
- 連続して §4.4 から 3 件の誤記が発覚しており、他変数にも潜在する可能性がある

## 解決日
2026-04-13
