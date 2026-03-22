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

## レビュー観点

成果物内の「品質チェック」は作成者のセルフチェック。以下は reviewer が検証する観点。

### 1. 上流整合性（Step 1 との整合）
- `usecases.md` の全 MVP ユースケースに対応する画面が `screens.md` に定義されているか（カバー率 100%）
- `rbac.md` のロール別権限が画面アクセス制御・操作ボタン表示に反映されているか
- `workflow.md` の全状態遷移が画面遷移図に表現されているか（提出・承認・却下・再申請・支払完了）
- `requirements.md` のロール定義と、サイドナビ・画面のロール別表示制御が一致しているか
- 全ロール（特に Accounting の Member 相当の申請権限）の画面アクセスが上流のロール定義と整合しているか

### 2. 内部整合性（Step 4 成果物間）
- `screens.md` の全画面IDが `ui_flow.md` の遷移図に登場しているか
- `ui_flow.md` の遷移先画面が `screens.md` に定義されているか（孤立画面・未定義画面がないか）
- ログアウト遷移が定義されているか

### 3. 完全性
- 共通 UI パターン（ヘッダー、ナビゲーション、エラー表示、ローディング、確認ダイアログ）が定義されているか
- 画面IDの命名規則が定義されており、全画面に適用されているか
- ロール別の遷移パスが全4ロール分定義されているか
- 未認証/認証済みの遷移制御が定義されているか

### 4. 用語準拠
- 画面名・ボタン名・ラベルが `glossary.md` の操作用語に従っているか

### 5. スコープ準拠
- Phase 3 固有の画面（メンバー管理・招待・通知一覧等）が MVP 画面一覧に混入していないか

### 6. 下流作業可能性
- Step 5（詳細設計）の作業者が、各画面IDをもとに「どの画面の詳細仕様を書けばよいか」を迷わず判断できるか
- 画面一覧の粒度が「1画面=1機能別詳細仕様ファイル」の原則に合致しているか
