# テストケース設計書の FE テストファイルパスが実装慣習と不一致

## 発見日
2026-04-07

## カテゴリ
testing

## 影響度
低

## 発見経緯
review

## 関連ステップ
Step 9（テストコード実装）

## ブロッカー
なし

## 問題

テストケース設計書（`test_cases/*.md`）に記載された FE テストファイルの配置パスが、プロジェクトで実際に採用されているコロケーション方式と異なる。

### 設計書の記載（例: items.md セクション 12.1）
```
src/__tests__/pages/reports/ItemForm.test.tsx
src/__tests__/hooks/useItems.test.tsx
```

### 実装の慣習（全既存テストファイルが採用）
```
src/pages/reports/__tests__/ItemForm.test.tsx
src/hooks/__tests__/useItems.test.tsx
```

items.md 以外のテストケース設計書（auth.md, reports.md, dashboard.md, tenant.md, workflow.md）にも同様の不一致がある可能性がある。

## 影響

- テストケース設計書を参照して実装する際に、パスの不一致で混乱が生じる
- テストケース ID とファイルパスの追跡可能性が設計書ベースでは成立しない

## 提案

以下のいずれかで統一する（判断が必要）:

1. **設計書を実装に合わせる**: `test_cases/*.md` のファイルパス記載をコロケーション方式に修正
2. **実装を設計書に合わせる**: 全テストファイルを `src/__tests__/` 配下に移動

既存テストファイル（9-A〜9-F で60ファイル以上）が全てコロケーション方式で配置済みのため、選択肢1が現実的と思われるが、判断はユーザーに委ねる。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
