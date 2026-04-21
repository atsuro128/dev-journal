# ログ・エラー出力の言語ポリシー整理（FE/BE 共通、日本語メッセージの棚卸しと削減）

## 発見日
2026-04-21

## カテゴリ
implementation / project-management

## 影響度
低（機能影響なし、運用品質・ポートフォリオ品質の改善）

## 発見経緯
user-report（Step 11-A SMK-036 対応議論中、`AttachmentUploader.tsx:190` の `console.error('ファイルのアップロードに失敗しました:', err)` を発見。ユーザーから「日本語のコンソールログはそもそも不要」「FE/BE 両方で同様の混在がないか確認したい」と指摘）

## 関連ステップ
Step 8（基盤構築）/ Step 10（機能実装）/ Step 11-A

## ブロッカー
なし（後追い品質改善）

## 問題

### 現状の実態

ログ・エラーメッセージに **日本語文言が散在**している。ログは一般に英語が推奨（検索性・grep 容易性・i18n の必要なし・開発者間の認識統一）。

#### 規模（2026-04-21 時点の grep 結果）

| 領域 | 件数 | 対象ファイル（主なもの） |
|------|------|--------------------|
| FE `console.*` with 日本語 | 1 件 | `frontend/src/pages/reports/AttachmentUploader.tsx:190` |
| BE 日本語ログ・エラー包含（slog + fmt.Errorf） | 51 件 | `internal/seed/seed.go` 34 / `cmd/seed/main.go` 12 / `internal/service/auth_service.go` 2 / `cmd/server/main.go` 1 / `internal/handler/workflow.go` 1 / `internal/handler/report.go` 1 |
| BE `slog.*` のみ（本番経路ログ） | 15 件 | `cmd/server/main.go` 1 / `cmd/seed/main.go` 12 / `internal/service/auth_service.go` 2 |

#### 具体例

**FE**:
```ts
// AttachmentUploader.tsx:190
console.error('ファイルのアップロードに失敗しました:', err);
```
- この `console.error` は本 project に error monitoring（Sentry 等、MVP 範囲外）が未導入のため、**本番では誰も見ない**
- 開発時も `err` オブジェクト自体は有用だが、日本語プレフィクスは検索性を下げる
- そもそも既に `onUploadError` で親に通知してトースト表示しているため、このログ自体が冗長

**BE（本番ログ経路の例）**:
```go
// cmd/server/main.go:133
slog.Error("S3 クライアントの初期化に失敗しました", "error", err)

// internal/service/auth_service.go
slog.Warn("認証失敗: xxx")
slog.Error("セッション生成失敗: xxx")
```

**BE（エラー wrap の例）**:
```go
// internal/seed/seed.go（34 箇所）
return fmt.Errorf("seed: パスワードハッシュ生成失敗: %w", err)

// internal/handler/workflow.go:341
return params, details, fmt.Errorf("クエリパラメータにバリデーションエラーがあります")
```

### なぜ問題か

1. **検索性・grep 容易性**: 英語統一されたログは CI ログ・運用時の grep で扱いやすい
2. **ログ aggregation**: 将来的に CloudWatch Logs や Datadog 等に流すとき、英語推奨（ロケール依存の文字化け回避）
3. **開発者認識の統一**: 「コード内コメントは日本語、ログは英語」の明確な分離で読みやすさ向上
4. **ポートフォリオ品質**: 業務で触る OSS / 一般的な Go コードはログが英語、プロジェクト品質の指標として整えておきたい
5. **冗長性**: FE の console.error は error monitoring 未導入のため実質機能していない → 不要

### 関連する設計ルール

`/root-project/.claude/rules/implementation-workflow.md` では以下と定義:

> ## コメント言語
> - コード内コメントは全て日本語で書く（godoc / JSDoc 含む）
> - 識別子（変数名・関数名・型名）は英語のまま

**コメントは日本語、識別子は英語、という規定はあるが、ログ・エラーメッセージの言語指針は定義されていない。本 issue で追加すべき**。

## 影響

- 機能: なし
- UX: なし（BE ログはユーザー目線に出ない / FE console は一般ユーザーは見ない）
- 運用品質: 低〜中（ログ aggregation 時に問題になりうる、ただし MVP では観測不要）
- ポートフォリオ品質: 中（業務コードの作法として整えたい）

## 提案

### フェーズ分け

#### Phase 1: 言語ポリシーの策定（設計タスク）

`/root-project/.claude/rules/implementation-workflow.md`（または `ai-dev-framework/rules/` 配下）に以下を追記:

```
## ログ・エラーメッセージ言語

- ログ出力（slog, logger, console.*）は英語で書く
- エラー wrap（fmt.Errorf, Error オブジェクト）のメッセージは英語で書く
- ユーザー向けメッセージ（UI トースト、API レスポンス body の `message` フィールド）は日本語で書く
- コード内コメント（実装意図・設計根拠）は引き続き日本語で書く
```

### Phase 2: 既存コードの棚卸しと整理

以下を優先度順に対応:

| # | 対象 | 件数 | 判定・対応 |
|---|------|------|---------|
| P1 | 本番経路ログ（`cmd/server/main.go`, `internal/service/auth_service.go`） | 3 件 | 英語化 |
| P2 | FE `console.error`（`AttachmentUploader.tsx:190`） | 1 件 | **削除**（error monitoring 未導入のため出す価値なし、既に親コンポーネントへ通知済み） |
| P3 | BE エラー wrap（`internal/handler/*.go`, `internal/service/*.go`） | 5 件前後 | 英語化 |
| P4 | seed 系（`cmd/seed/main.go`, `internal/seed/seed.go`） | 46 件 | 英語化（開発用ツールだが作法として） |

### Phase 3: ルール化の徹底（予防的対応）

- レビュー観点に「ログ・エラーメッセージの言語」を追加
- 可能なら lint ルールで検出（例: `eslint no-restricted-syntax` で `console.*` 内の非 ASCII 検出、Go の golangci-lint カスタムルールは実装コスト高なので手動レビューで十分）

### 却下案

- **全て日本語に統一**: 既存の暗黙ルール（コメントのみ日本語、識別子は英語）と整合が取れない
- **対応しない**: 件数 50 以上で運用品質に影響するため最低限の整理は必要

## 完了条件

- ログ言語ポリシーが `/root-project/.claude/rules/implementation-workflow.md` に追記されている
- FE: `console.error` 日本語を含む箇所が 0 件（grep で検証可能）
- BE 本番経路（`cmd/server/main.go`, `internal/service/*.go`, `internal/handler/*.go`）のログが英語化されている
- BE seed 系（`cmd/seed/`, `internal/seed/`）のログ・エラー wrap が英語化されている（低優先度、別 PR でも可）
- project-rules.md またはレビューチェックリストにログ言語の観点が追加されている
- 既存テスト通過（文言アサーションはユーザー向けメッセージのみ、ログ文言は検証しない前提）

## 関連

- **#131**（open）: クライアントサイドバリデーションエラー表示の UX 改善。ログとユーザー向けメッセージの扱いを分離する観点で本 issue と論点が接続
- **SMK-036**（Step 11-A、本 issue 起票のきっかけ）: MIME 偽装検出時の BE → FE メッセージ握り潰しの議論中に、FE console.error の不要性が浮上

## 補足: ユーザー向けメッセージのみ日本語とする判断根拠

本 project の UX 観点（SMK-050〜053 で「ラベル・ボタン・エラーメッセージの日本語整合」を保証している）に基づき、**ユーザーに露出するメッセージは日本語維持、内部ログは英語統一** が明確な分離になる。

---

## 解決内容

## 解決日
