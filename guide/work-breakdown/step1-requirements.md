# Step 1: 要件定義（業務理解 → ユースケース化） — 作業分解

## 概要

経費精算の"業務"を理解し、作るものを言語化する。

## 成果物（出力）

| 成果物 | パス |
|--------|------|
| 機能要件・非機能要件 | `deliverables/docs/10_requirements/requirements.md` |
| ユースケース一覧 | `deliverables/docs/10_requirements/usecases.md` |
| 状態遷移 | `deliverables/docs/10_requirements/workflow.md` |
| ロール/権限要件 | `deliverables/docs/10_requirements/rbac.md` |

## 成果物の詳細

### requirements.md

- コア機能：認証・RBAC・経費申請・承認フロー・添付ファイル
- 付随機能：招待フロー・アプリ内通知・監査ログ・CSVエクスポート
- 非機能要件：レスポンスタイム・可用性・レート制限・ファイルサイズ制限

### usecases.md

- 申請者・承認者・Accounting・Adminそれぞれの操作シナリオを網羅
- 招待フロー・通知受信・監査ログ閲覧も含める

### workflow.md

- 申請〜承認の状態遷移（Mermaid推奨）

### rbac.md

- ロール/権限の要件

## 最低限のMVP業務フロー例

- 申請 → 承認 → 差戻し → 再申請 → 承認 → 完了

## 【root-project 整備】

- `prompts/requirement.md`：要件整理に使う指示文テンプレートを整備（ユースケース抽出・状態遷移整理などの型）
- `rules/security-policy.md`：セキュリティ要件の基礎方針（認証方式・レート制限・ファイルサイズ制限の方針）を記載

## 完了条件

- 「誰が」「何を」「どの順で」するか説明できる
- 例外（却下/取消など）を"やる/やらない"で決めている
- 招待フロー・通知・監査ログ・CSVエクスポートの扱いが明記されている
