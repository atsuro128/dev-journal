# 手動チェックリスト定義

- 担当: designer
- 依存: 6-C（トレーサビリティ定義）
- ブランチ: なし（dev-journal 配下のため設計成果物フロー）
- 出力先: `deliverables/docs/60_test/manual_checklists/`
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| テスト戦略 | deliverables/docs/60_test/test_strategy.md | 保証種別・カバレッジ基準 |
| トレーサビリティ | deliverables/docs/60_test/traceability.md | 自動テストカバー状況の正本。重複排除の判定軸 |
| 全テストケース | deliverables/docs/60_test/test_cases/*.md | 自動テストの網羅範囲 |
| 画面仕様 | deliverables/docs/50_detail_design/screens/*.md | 対象画面・操作手順 |
| ユースケース | deliverables/docs/10_requirements/usecases.md | UAT の受け入れ確認対象 |
| 機能要求 | deliverables/docs/10_requirements/requirements.md | 要件ID の参照元 |
| ポリシー定義 | deliverables/docs/10_requirements/policies.md | ポリシーID の参照元 |

## 責務

自動テストでは検証できない、または自動テストを補完する必要がある手動確認観点を抽出し、Step 11（システムテスト・UAT）の実施者が迷わず作業できる手動チェックリストを作成する。

### 重複排除ルール

traceability.md の実際の列は `要件ID / 要件概要 / 設計反映先 / BE テスト反映先 / FE テスト反映先 / 備考` である。以下のルールで手動確認が必要な要件を特定する:

- BE テスト反映先 と FE テスト反映先 の両方が `-`（自動テスト非該当）の要件 → 自動テストで担保されていない。手動チェック候補
- 備考欄に「自動テスト対象外」「運用確認項目」等の注記がある要件 → 自動テストでは補いきれない。手動チェック候補
- 上記以外（テスト反映先が埋まっており備考に特記なし） → 自動テストで担保済み。手動チェックに含めない

### 責務境界

- **smoke_check.md（開発者視点のスモーク確認、Step 11-A で使用）**:
  - ブラウザ実挙動・表示崩れ・文言・体感導線に限定
  - **業務フロー成立性の検証は含めない**（E2E 自動テストの領域）
  - 観点: ロール別導線、通信中の挙動、エラー表示、添付ファイルのブラウザ実挙動、レスポンシブ、日本語 UI、タイムスタンプ、ページネーション、キャッシュ整合性
- **uat_check.md（ユーザー視点の受け入れ確認、Step 11-F で使用）**:
  - 業務要件の妥当性（各ユースケースが実際の業務で達成できるか）
  - 各ロール（Member / Approver / Accounting / Admin）の主要業務フロー
  - UX の妥当性（操作感・用語・画面の分かりやすさ）

### 標準列（共通）

- チェックID（SMK-XXX / UAT-XXX）
- 確認観点
- 対象画面・対象機能
- 前提データ（投入すべきフィクスチャ・初期状態）
- 利用アカウント（ロール）
- 開始状態（操作開始画面）
- 操作手順
- 期待結果
- 確認者（開発者 / ユーザー）

### uat_check.md のみ追加列

- 対応ユースケースID（`usecases.md` の UC-XXX）
- 対応要件ID（`requirements.md` または `policies.md` の ID）

## 完了条件

- `deliverables/docs/60_test/manual_checklists/smoke_check.md` が作成され、上記9観点が網羅されている
- `deliverables/docs/60_test/manual_checklists/uat_check.md` が作成され、ユースケース・ロール別フロー・UX が網羅されている
- 各項目に共通の標準列（チェックID・確認観点・対象・前提データ・利用アカウント・開始状態・操作手順・期待結果・確認者）が全て記載されている
- uat_check.md の各項目に対応ユースケースID／要件ID が記載されている
- traceability.md を参照した結果として、自動テストで既にカバーされている観点との重複が排除されている
- smoke_check.md が業務フロー成立性の検証（E2E 自動テストの領域）を含んでいない
