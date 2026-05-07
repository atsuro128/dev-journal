# Step 11-C E2E テスト品質改善（CRS-074 アサーション強化 + カテゴリセレクタ data-testid 化）

## 発見日
2026-05-07

## カテゴリ
test / quality-improvement / post-mvp

## 影響度
低（本番動作に影響なし。E2E テストの将来フレーク化リスクとカバレッジの薄さを改善する）

## 発見経緯

PR #138（Step 11-C E2E テスト）の reviewer レビューで以下 2 件の info 指摘を受けた。本 PR では受容しマージ済みだが、品質改善として post-MVP 対応する:

1. **CRS-074（Admin で全レポート一覧ページにアクセスできる）が URL 確認のみ**
   - 現状: `page.goto('/all-reports')` → `expect(page).toHaveURL(/\/all-reports/)` のみ
   - 不足: AllReportsPage の実データ表示（複数テナントのレポートが見える等）が未検証
   - work-breakdown 11-C はフロー3 を「オプション、ベストエフォート」と明記しているため本 PR では受容

2. **flow1 のカテゴリセレクタが脆弱（将来フレーク化リスク）**
   - 現状: `[aria-label="カテゴリ"]` + fallback の二段構え（DOM walk）
   - リスク: MUI バージョンアップで aria-label 体系が変わるとフレーク化
   - 対応案: 実装側 `frontend/src/pages/reports/ItemForm.tsx` に `data-testid="category-select"` を追加 → テスト側を `[data-testid="category-select"]` に簡略化

## 関連ステップ

- Step 11-C E2E テスト（PR #138 マージ後の品質改善）
- post-MVP 改善対応として扱う

## ⚠ MVP スコープ外

本 issue は MVP スコープ外として扱う。MVP リリース前には対応しない。管理方式は **ops-080** で検討中。

## 問題

### 1. CRS-074 のアサーション弱さ

`expense-saas/e2e/flow3_admin_test.ts` の CRS-074:

```ts
await page.goto('/all-reports');
await expect(page).toHaveURL(/\/all-reports/);
// ※ 実データ（複数テナントのレポート行が表示されること）の検証なし
```

CRS-074 の test_cases/cross-cutting.md §3 上の期待:
- Admin が AllReportsPage で全テナントのレポートを閲覧できること

現状はアクセス可能性のみを確認しており、Admin 専用の「全テナントレポート閲覧」要件が完全には検証されていない。

### 2. カテゴリセレクタの DOM walk 依存

`expense-saas/e2e/flow1_test.ts` CRS-058:

```ts
const categorySelect = page.locator('[aria-label="カテゴリ"]').first();
if (await categorySelect.isVisible()) {
  await categorySelect.click();
} else {
  await page.locator('label', { hasText: 'カテゴリ' })
    .locator('..').locator('select, [role="combobox"]').first().click();
}
```

8 行のセレクタ分岐は MUI の内部 DOM 構造（label の親要素から select/combobox を探す）に依存している。

## 提案

### 1. CRS-074 のアサーション追加

```ts
// AllReportsPage が表示されたことを確認する。
await page.waitForSelector('[data-testid="all-reports-page"]', { timeout: 15_000 });

// 自テナント外のレポートも 1 件以上表示されることを確認する（Admin 専用機能）。
const rowCount = await page.locator('[data-testid="report-row"]').count();
expect(rowCount).toBeGreaterThan(0);
```

前提: `frontend/src/pages/all-reports/AllReportsPage.tsx` に `data-testid="all-reports-page"` および各行に `data-testid="report-row"` が付与されていること（要実装側対応）。

### 2. カテゴリセレクタの data-testid 化

`frontend/src/pages/reports/ItemForm.tsx` のカテゴリ Select に `data-testid="category-select"` を追加。

E2E 側:

```ts
// Before
const categorySelect = page.locator('[aria-label="カテゴリ"]').first();
if (await categorySelect.isVisible()) {
  await categorySelect.click();
} else {
  await page.locator('label', { hasText: 'カテゴリ' })
    .locator('..').locator('select, [role="combobox"]').first().click();
}

// After
await page.locator('[data-testid="category-select"]').click();
```

実装変更（FE）+ E2E テスト変更がセットになる。

## 対応優先度

post-MVP（MVP リリース前は対応しない）。

## 対応方針

- MVP リリース後、テスト品質改善のスプリントで対応
- 1 PR で両方対応（実装変更 + E2E テスト変更）
- 修正後は E2E 全件 PASS を確認

## 関連 issue / PR

- 関連 PR: #138（Step 11-C E2E テスト、本 issue の発見元）
- 関連 reviewer 投稿: PR #138 の info-3 / info-4 指摘
