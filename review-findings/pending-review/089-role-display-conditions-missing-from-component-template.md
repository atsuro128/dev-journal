# 089: コンポーネント設計テンプレートに認可・表示条件の記録欄がない

## 指摘概要
`55_ui_component/screens/component-design-template.md` には、コンポーネントごとの表示条件や認可根拠を記録する欄がありません。Step 5.5 の上流入力には `authz.md` が含まれ、レビュー観点でもロール別表示制御との整合が必須ですが、現行テンプレートのままだと designer がその情報を構造化して残せず、画面詳細仕様にあるロール差分がコンポーネント設計に落ちないおそれがあります。

## 根拠
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:24`-`27`
  - 上流成果物に `50_detail_design/authz.md` を含めている。
- `ai-dev-framework/guide/work-breakdown/step5.5-ui-component.md:148`-`151`
  - 「ロール別表示制御が `authz.md` の認可ルールと一致しているか」をレビュー観点にしている。
- `ai-dev-framework/templates/docs-v2/50_detail_design/screens/screen-detail-template.md:75`-`79`
  - 上流の画面詳細テンプレートには「ロール差分 / 状態差分」セクションがある。
- `ai-dev-framework/templates/docs-v2/55_ui_component/screens/component-design-template.md:42`-`100`
  - コンポーネント定義、データフロー、状態管理、画面詳細仕様との対応はあるが、表示条件・認可根拠を記録する欄がない。

## 判定
- 重大度: 中
- 分類: FIX

## 修正方針案
- `component-design-template.md` のコンポーネント定義または別セクションに、少なくとも以下を追加する。
  - 表示条件
  - 操作可否条件
  - 根拠参照（`authz.md` / `50_detail_design/screens/*.md` の該当節）
- 画面詳細仕様の「ロール差分 / 状態差分」をどのコンポーネントに反映したか追跡できる形にする。
