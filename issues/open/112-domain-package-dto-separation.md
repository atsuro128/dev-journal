---
# domain パッケージに DTO が混在している

## 発見日
2026-04-16

## カテゴリ
architecture

## 影響度
中（現時点で動作に問題はないが、放置すると domain 層の責務が曖昧になり、無関係な型が追加される温床になる）

## 発見経緯
user-report — issue 110 対応中に domain/ の構成を確認した際、ユーザーが指摘。

## 関連ステップ
Step 3（アーキテクチャ設計）/ Step 8-2（バックエンド初期化）/ Step 10（機能実装）

## ブロッカー
なし

## 問題

`expense-saas/internal/domain/` にエンティティ・値オブジェクト・ドメインエラー・リポジトリインターフェースと並んで `dto.go`（レイヤー間転送オブジェクト）が配置されている。

DTO はアプリケーション層（service）やインターフェース層（handler）の関心事であり、ドメイン層の責務ではない。DDD の観点からは domain パッケージに含めるべきではない。

### 現状の domain/ 構成

```
internal/domain/
├── entity.go        # User, Tenant, TenantMembership（エンティティ）
├── report.go        # ExpenseReport 集約ルート（エンティティ）
├── item.go          # ExpenseItem（エンティティ）
├── auth.go          # RefreshToken 等（エンティティ）
├── actor.go         # Actor 認可主体（値オブジェクト）
├── enum.go          # ReportStatus, Role, Category（値オブジェクト）
├── errors.go        # ドメインエラー
├── repository.go    # リポジトリインターフェース
├── dto.go           # ← ここが問題
├── model/           # 空ディレクトリ（.gitkeep のみ、Step 8-2 の残骸）
└── *_test.go        # ドメイン層テスト
```

### domain に妥当なもの

エンティティ、値オブジェクト、ドメインエラー、リポジトリインターフェース — これらは DDD でドメイン層に属する。

### domain に不適切なもの

- `dto.go` — レイヤー間転送オブジェクト。service → handler の変換用であり、ドメインの関心事ではない
- `model/.gitkeep` — 未使用の空ディレクトリ（Step 8-2 スキャフォールドの残骸）

## 影響

- domain パッケージの責務が曖昧になる
- 今後の開発で「とりあえず domain に置く」が常態化するリスク
- DTO の変更がドメイン層のテストに波及する可能性
- Go のパッケージ循環参照回避のために domain に寄せた経緯があり、分離時に import 関係の整理が必要

## 提案

### dto.go の移動先候補

- **案 A**: `internal/service/dto.go` — service 層が主な利用者なので自然
- **案 B**: `internal/dto/dto.go` — 独立パッケージ化。handler と service の両方から参照可能
- **案 C**: `internal/handler/response.go` 等に分割 — handler のレスポンス型として配置

### model/.gitkeep の削除

空ディレクトリの残骸なので単純に削除。

### 影響範囲の見積もり

dto.go を移動すると、以下の import 修正が必要:
- `internal/service/*.go` — DTO を生成・返却している
- `internal/handler/*.go` — DTO を JSON レスポンスに変換している
- `internal/handler/*_test.go` — テストで DTO を検証している

## 早期対応 vs Post-MVP の判断材料

### 早期対応が望ましい理由
- 今なら DTO の利用箇所が限定的（handler + service のみ）
- Step 11-B/C でテストコードが増える前に移動した方が影響範囲が小さい
- 102（添付プレビュー）で新しい DTO（AttachmentAccess）を追加する予定があり、追加前に整理すべき

### Post-MVP でもよい理由
- 現時点で動作に問題はない
- MVP のスコープを広げることになる
- import 修正は機械的だがテスト全通しが必要

## 完了条件

- dto.go が domain パッケージから適切な場所に移動されている
- model/.gitkeep が削除されている
- 全テスト（unit + integration）が通過している
- architecture.md のディレクトリツリーが更新されている

## 関連

- 110: architecture.md ディレクトリツリー乖離（本 issue の発見契機）
- 102: 添付プレビュー（新しい DTO 追加予定、本 issue の対応タイミングに影響）
- Step 8-2: バックエンド初期化（model/.gitkeep の作成元）

---

## 解決内容

## 解決日
