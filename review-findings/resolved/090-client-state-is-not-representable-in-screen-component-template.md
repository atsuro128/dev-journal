# 090: 画面別コンポーネント設計テンプレートでクライアント状態を表現できない

## 指摘概要
Step 5.5 では状態カテゴリとして「サーバー状態 / クライアント状態 / UI 状態」を定義していますが、`component-design-template.md` の状態管理表は `{サーバー/UI/フォーム}` しか想定していません。認証トークン、ユーザー情報、テナント文脈などのグローバル store 由来の状態を画面単位で記述しづらく、`state-management.md` と `screens/*.md` の整合確認が弱くなります。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:13`
  - Step 5.5 で「ローカル state / カスタム Hook / グローバル store の使い分け」を決めるとしている。
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:80`-`84`
  - `state-management.md` に含めるべき内容として状態カテゴリ、グローバル store、フォーム状態管理を定義している。
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:252`-`255`
  - 5.5-B の作業内容で「クライアント状態（認証トークン・ユーザー情報）」を明示している。
- `ai-dev-framework/templates/docs-v2/55_ui_component/screens/component-design-template.md:80`-`84`
  - 状態管理表のカテゴリ例が `{サーバー/UI/フォーム}` となっており、`クライアント` が抜けている。

## 判定
- 重大度: 中
- 分類: FIX

## 修正方針案
- `component-design-template.md` の状態管理表に `クライアント` を含める。
- あわせて、状態の取得元を `store / custom hook / local state` で区別できる列を追加し、`state-management.md` のどの定義に紐づくか参照できるようにする。

## 再レビュー結果（2026-04-06）

対応妥当（クローズ）。

### 確認内容
- `ai-dev-framework/templates/docs-v2/55_ui_component/screens/component-design-template.md` の §5 状態管理表で、カテゴリ例が `{サーバー/クライアント/UI/フォーム}` に更新されている
- これにより、`state-management.md` で定義するクライアント状態を `screens/*.md` 側でも明示できる

### 判定理由
- 元指摘の必須論点だった「クライアント状態を表現できない」問題は解消された
- 追加提案だった取得元専用列は未追加だが、今回の FIX 判定に必要な最小要件はカテゴリ例の欠落解消で満たしている
