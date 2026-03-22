---
step: 5
severity: medium
status: resolved
---

# 040: パスワードリセットトークンの使用後処理が文書間で矛盾している

## 指摘概要
`security.md` は「使用後に DB から削除」と定義していますが、`db_schema.md` は `used_at` により使用済み管理する設計で、物理削除を前提としていません。どちらを実装基準にするかで SQL、インデックス、再利用防止ロジック、運用バッチの設計が変わるため、現状のままでは実装が分岐します。

## 根拠
- [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L235) では「使用回数: 1回限り。使用後に DB から削除」と定義している
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L360) では `password_reset_tokens` に `used_at` カラムを持たせている
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L370) では「使用済みトークンは `used_at` に使用日時を記録」としており、削除方針と一致していない
- [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L762) のインデックス設計も `used_at IS NULL` 前提になっている

## 判定
中 / 詳細設計の内部不整合

## 修正方針案
トークンの終端処理をどちらかに統一する。
- 監査性や再利用検知を優先するなら `used_at` 管理に統一し、`security.md` の「削除」を修正する
- 物理削除に統一するなら `used_at` と関連インデックスを削除し、削除時点・検索条件・運用上の監査方針を再定義する

## 再レビュー結果（2026-03-22）

対応妥当（クローズ）。

### 確認内容
- `security.md` のパスワードリセット仕様が「使用後に `used_at` を記録して再利用を防止」に更新され、物理削除方針が除去された
  - [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L233)
- `db_schema.md` の `password_reset_tokens` は引き続き `used_at` カラムを保持し、設計判断でも同じ終端処理を明記している
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L364)
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L377)
- 有効トークン向け部分インデックスも `used_at IS NULL` 前提で残っており、本文仕様とインデックス設計が一致している
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L762)
  - [db_schema.md](/root-project/dev-journal/deliverables/docs/50_detail_design/db_schema.md#L784)

### 判定理由
パスワードリセットトークンの使用後処理が `used_at` 管理に統一され、仕様・スキーマ・インデックスの解釈が分岐しない状態になったため。
