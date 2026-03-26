# 069: Step7 のバックアップ・リストア成果物が MVP スコープ定義と衝突している

## 指摘概要
Step 7 の正本は `backup_restore.md` に手動リストア手順まで含めることを完了条件としているが、上流の `02_scope.md` と `requirements.md` は「手動リストア手順は対象外」と定義している。現状のままでは Step 7 成果物が Step 7 の正本には適合しても、MVP スコープおよび可用性要件には適合しないため、レビュー基準が二重化している。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step7-operations.md:67-76,127-134`
  - `backup_restore.md` に「復旧手順」「復旧確認手順」「RPO / RTO」の記載を要求し、完了条件にも含めている。
- `dev-journal/deliverables/docs/02_scope.md:89-102`
  - 「データのバックアップ・リストア手順」はスコープ外であり、「手動リストア手順は文書化しない」と定義している。
- `dev-journal/deliverables/docs/10_requirements/requirements.md:442-444`
  - `NFR-AVAIL-003` は「RDS 自動バックアップ（7日間保持）」を要求しつつ、備考で「手動リストア手順は対象外」としている。
- `dev-journal/deliverables/docs/70_operations/backup_restore.md:104-205`
  - PITR / スナップショット復元の手動手順を詳細に定義している。

## 判定
高 / 上流整合性欠落（要方針判断）

## 修正方針案
- `02_scope.md` と `requirements.md` を更新し、Step 7 で手動リストア手順を扱う方針に正式に揃える。
- もし手動リストア手順を本当に MVP 対象外にするなら、`step7-operations.md` の完了条件と `backup_restore.md` の責務を縮小する。
- いずれにせよ、MVP 境界の正本を 1 つに揃えたうえで Step 7 を再判定する必要がある。

## 再レビュー結果（2026-03-26）

- **解消**
  - `ai-dev-framework/guide/work-breakdown/step7-operations.md` の `backup_restore.md` 責務・MVP 境界・完了条件が縮小され、手動リストア手順を要求しない契約に修正された
  - `dev-journal/deliverables/docs/70_operations/backup_restore.md` は手動リストア手順を削除し、RDS 自動バックアップ前提の復旧方針と AWS 公式ドキュメント参照に責務を限定している
  - `dev-journal/deliverables/docs/02_scope.md` と `dev-journal/deliverables/docs/10_requirements/requirements.md` の「手動リストア手順は対象外」という上流定義と整合している

- **判定**
  - クローズ可。元指摘の「Step 7 正本が MVP スコープと衝突している」状態は解消された
