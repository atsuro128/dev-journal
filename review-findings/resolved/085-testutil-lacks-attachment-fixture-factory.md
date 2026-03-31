# 085: （タイトル：テスト基盤に添付ファイルのフィクスチャファクトリがない）

## 指摘概要
Step 8-7 の責務には添付ファイルを含むフィクスチャファクトリが明記されているが、`internal/testutil` にあるのは tenant / user / membership / report / item までで、attachment を生成するファクトリが存在しない。Step 9 の添付 API テストは共通基盤だけで着手できず、チケット責務と受け渡し契約を満たしていない。

## 根拠
- `dev-journal/progress-management/tickets/step8/8-7-test-infra.md:18-20`
  - フィクスチャファクトリの対象として「テナント、ユーザー、経費レポート、明細、添付ファイル」を要求している。
- `ai-dev-framework/guide/work-breakdown/step8-foundation.md:197-199`
  - Step 9 への受け渡しとして「動作するテスト基盤（…フィクスチャファクトリ、HTTP テストヘルパー）」を要求している。
- `expense-saas/internal/testutil/fixture.go:231`
  - `CreateTenant` が実装されている。
- `expense-saas/internal/testutil/fixture.go:275`
  - `CreateUser` が実装されている。
- `expense-saas/internal/testutil/fixture.go:318`
  - `CreateMembership` が実装されている。
- `expense-saas/internal/testutil/fixture.go:342`
  - `CreateReport` が実装されている。
- `expense-saas/internal/testutil/fixture.go:392`
  - `CreateItem` が実装されている。
- `expense-saas/internal/testutil` 配下を確認しても `CreateAttachment` 相当の実装は存在しない。

## 判定
中 / FIX

## 修正方針案
- `internal/testutil/fixture.go` などに attachment 用の factory / option を追加する。
- 標準 seed にも添付ありケースが必要なら追加し、添付 API の統合テストが即座に書ける状態にする。

## 解決内容
- `internal/testutil/fixture.go` に `CreateAttachment` ファクトリと `AttachmentOption` 型を追加
- WithAttachmentFileName, WithAttachmentFileSize, WithAttachmentMimeType, WithAttachmentS3Key オプションを実装

## 解決日
2026-03-31
