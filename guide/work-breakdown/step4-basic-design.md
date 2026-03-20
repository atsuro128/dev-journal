# Step 4: 基本設計（画面・画面遷移・UX）

## 目的

画面一覧と画面遷移を確定する。「何の画面が必要か」を全体俯瞰レベルで決め、Step 5 の機能別詳細設計の土台を作る。

## 上流成果物（入力）

| 成果物 | パス |
|--------|------|
| ユースケース | `deliverables/docs/10_requirements/usecases.md` |
| 要件定義 | `deliverables/docs/10_requirements/requirements.md` |
| RBAC | `deliverables/docs/10_requirements/rbac.md` |
| ワークフロー | `deliverables/docs/10_requirements/workflow.md` |
| アーキテクチャ | `deliverables/docs/30_arch/architecture.md` |

## 成果物

| 成果物 | パス |
|--------|------|
| 画面一覧 | `deliverables/docs/40_basic_design/screens.md` |
| 画面遷移図 | `deliverables/docs/40_basic_design/ui_flow.md` |

### screens.md の内容

- 全画面の一覧（画面ID・画面名・目的・主要表示項目・対応ロール）
- 画面IDの命名規則
- 共通 UI パターン（ヘッダー、ナビゲーション、エラー表示、ローディング、確認ダイアログ）

粒度は粗めでよい。各画面の入力項目・バリデーション等の詳細は Step 5 で定義する。

### ui_flow.md の内容

- 画面遷移図（Mermaid）
- ロール別の遷移パス

## プロセス

```
Lead
 1. basic-designer を起動: screens.md + ui_flow.md を作成
    入力: usecases.md, requirements.md, rbac.md, workflow.md, architecture.md
    出力: 40_basic_design/screens.md, 40_basic_design/ui_flow.md
    粒度: 画面一覧レベルの俯瞰。入力項目・バリデーション等の詳細は Step 5
 2. design-unit-reviewer を起動: 成果物レビュー
    チェック: 上流成果物との整合性、全UCの画面カバー率、ロール別遷移パスの網羅性
 3. 品質ゲート判定（.claude/rules/workflow.md 基準）
 4. ユーザーにコミット確認を依頼
```

## 完了条件

- 主要画面（申請、承認キュー、管理画面）が揃っている
- 権限で操作可否が変わる点が明記されている
- 画面遷移図がある
- 後続の Step 5 で機能別に詳細化できる状態になっている（画面IDが付与されている）
