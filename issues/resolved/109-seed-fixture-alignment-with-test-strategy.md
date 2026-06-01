# seed.go を業務的に妥当 + 設計書整合な状態に再整備

## 発見日
2026-05-29

## カテゴリ
testing

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
post-MVP（Step 6 テスト設計 / Step 8 seed 実装）

## ブロッカー
なし

## 問題

seed.go のフィクスチャ定義に以下の問題がある。

### 1. 設計書との乖離（確認済み箇所）

- `test_strategy.md §4.2` では「テナントBのユーザーは Member 1名のみ」と定義されているが、`seed.go` ではテナントBに Member（test-member-b）+ Approver（test-approver-b）の 2 名が投入されている
  - test-approver-b は seed.go のコメントで「SMK-104 用」（スモークチェックでテナント B の承認操作を試すため）と追加された経緯あり。test_strategy.md §4.2 への反映が漏れた
- テナント A の第二 Approver（test-approver2、UserApprover2ID）も seed.go のコメントで「SMK-105 用」と追加されているが、test_strategy.md §4.2 のテナント A ユーザー定義には未記載

### 2. 業務モデルとしての不自然さ

- テナントBに Admin が存在しない。SaaS の業務モデル上、テナントには Admin が必ず存在するのが自然
- 採用担当者・他開発者が README のテストアカウント一覧を見たときに「テナントBには Admin がいないけど業務的に大丈夫？」と疑問を持つ

### 3. 他フィクスチャでも同様の問題がある可能性（未調査）

- 経費レポート（テナントA 8件 + テナントB 3件）
- 経費明細
- 添付ファイル
- カテゴリ

これらでも設計書と実装の乖離・業務不整合が潜んでいる可能性がある。

## 影響

- **採用担当者・他開発者の混乱**: README のテストアカウント表で「Admin 不在」「設計書通りでない」と感じる
- **設計書を正本として開発する人の混乱**: test_strategy.md §4.2 を読んで開発しようとすると、seed.go の実態と異なるためテストが落ちる・期待値が合わない
- **後続フィクスチャ追加時の判断混乱**: 「設計書を直すべきか seed を直すべきか」の判断軸が不明確なまま追加が進む可能性

## 提案

### ゴール

「**妥当な seed**」= 業務モデルとして整合 かつ 設計書と実装が一致 な状態にする。

### 対応手順

1. **乖離・不整合の洗い出し**
   - `seed.go` 全項目を `test_strategy.md §4.2`〜`§4.5` と突合
   - 業務モデル整合性（各テナントに必要なロール構成等）を確認
   - 乖離・不整合の一覧を作成

2. **「妥当な seed」の再定義**
   - 洗い出し結果を元に、テナントごとの理想的なフィクスチャ構成を設計
   - 例: テナントBにも Admin / Approver / Member / Accounting を揃えるか、最小構成（Member + Approver）に留めるか
   - 「テスト用」と「業務モデル妥当性」のバランスをどう取るかを決める

3. **設計書 + 実装の同期更新**
   - `test_strategy.md §4.2`〜`§4.5` と `seed.go` を一致させる
   - 設計書を実態に合わせるパターンと、seed を設計書に合わせるパターンを項目ごとに判断

4. **テスト影響確認**
   - 既存テストコード（unit / integration / E2E）の前提が変わらないか確認
   - フィクスチャを直接参照しているテストがあれば更新
   - SMK-104 / SMK-105 のテストケースが新しいフィクスチャで継続実施可能か確認

5. **README 更新**
   - `expense-saas/README.md` のテストアカウント表を最終的な seed に合わせる

### 想定される波及範囲

| ファイル | 影響内容 |
|---|---|
| `expense-saas/internal/seed/seed.go` | フィクスチャ追加・削除・修正 |
| `dev-journal/deliverables/docs/60_test/test_strategy.md` §4.2〜§4.5 | フィクスチャ定義の更新 |
| `expense-saas/internal/seed/seed_test.go` | テスト前提の更新 |
| `expense-saas/internal/testutil/fixture.go` | testutil で再エクスポートしている fixture の更新 |
| `expense-saas/e2e/` | フィクスチャ参照箇所の更新（特に SMK-104 / SMK-105 関連） |
| `expense-saas/README.md` | テストアカウント表の更新 |

### 留意事項

- UAT 完了後の seed 構成を変更するため、変更後にローカルでの seed 投入 → 動作確認を実施する必要あり
- AWS 本番環境にも seed を投入済み（UAT クリーンアップ時）。変更を本番に反映するかは別途判断

---

## 解決内容

前セッションの architect 調査 + reviewer 検証で「seed.go に誤った値はなく、乖離はすべて『seed が先行 + 設計書の追従漏れ』。唯一の業務モデル不整合はテナント B の Admin 不在」と確定。**案 4（テナント B に Admin のみ追加 + 設計書を seed に同期）** を採用し、以下 4 ステップで対応した。

### ステップ1: テナント B に Admin 追加（PR #157、master `2ae2f5c`）
- `internal/seed/seed.go` に `UserAdminBID = bbbbbbbb-1111-1111-1111-000000000011`（email `test-admin-b@example.com` / RoleAdmin）を user + membership として冪等追加。`internal/testutil/fixture.go` 再エクスポート、`README.md` テストアカウント表に Admin B 行 + テナント B「クロステナント検証用・Accounting なし」注記を追加
- テナント A は不変。reviewer / codex とも PASS（指摘ゼロ）。マージ後の BE full test（lint/unit/integration）全 PASS

### ステップ2: test_strategy.md §4 を seed 最終形に同期（dev-journal `e1f29b5` + `dbaf6ef`）
- §4.2（テナント A 第二 Approver、テナント B Admin/Approver 追記・見出しロール数訂正）、§4.3（レポート 9 件・明細 7 件化 + 承認者/支払者列）、§4.5（テナント B 提出者/承認者列）、§4.6（添付フィクスチャ章新設）を seed.go に一致させた。改訂履歴 1.1 追記
- reviewer PASS → codex 指摘 125（Test Member Empty の用途誤記）/ 126（添付の再 seed 復元説明の誤記）を修正 → codex 再レビューで両件 resolved

### ステップ3: seed.go コメントの実態同期（PR #158、master `b6fdaa6`）
- パッケージコメントの準拠範囲を §4.2/4.3/4.4 → §4.2〜4.6 に修正。添付投入コメントの「再 seed で復元できる」誤記を訂正（ON CONFLICT DO NOTHING のため論理削除済み行は復元されない旨）。コメントのみの簡略 PR

### ステップ4: AWS 本番（公開デモ RDS）への反映
- 本番イメージの seed が旧版の可能性があるため、直接 SQL INSERT 方式を採用（イメージ再ビルド不要・最小・冪等）
- 事前に RDS スナップショット `expense-saas-pre-admin-b-20260601-165039` 取得 → SSM 経由で psql により Admin B の user（password_hash は既存 test-member-b から複製し TestPass1! を保証）+ membership を `ON CONFLICT DO NOTHING` で投入 → 検証 SELECT 1 行・本番ログイン HTTP 200 を確認
- 本番識別子は `expense-saas-portfolio-db`（RDS）/ `expense-saas-portfolio-app`（EC2）

## 解決日
2026-06-01
