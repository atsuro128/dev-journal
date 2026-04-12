# UAT

- 担当: ユーザー
- 依存: 11-E（Step 6-D 完了が前提）
- ブランチ: なし
- 出力先: UAT 確認結果（ブロッカー issue の有無）
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| UAT チェックリスト | deliverables/docs/60_test/manual_checklists/uat_check.md | 全36項目（受け入れ確認の正本） |
| デプロイ済み環境 | — | 11-E の成果物 |
| ユースケース | deliverables/docs/10_requirements/usecases.md | 受け入れ判定の根拠 |
| 画面仕様 | deliverables/docs/50_detail_design/screens/*.md | 全画面 |

## 責務

- **uat_check.md の全36項目を実施**
- 各ロール（Member / Approver / Accounting / Admin）の主要業務フローを確認
- 業務要件（ユースケース）の妥当性確認
- UX の妥当性確認（操作感・用語・画面の分かりやすさ）
- 各項目の対応ユースケースID／要件ID を確認しながら実施
- 含めない: テストコードの実装、インフラ変更

## 完了条件

- uat_check.md の全36項目が実施され、結果が記録されている
- MVP の全ユースケース（UC-M01〜09, M03a, A01〜03, AC01〜03, AD01〜02, SYS01〜05）がカバーされている
- ブロッカーとなる問題がないこと
