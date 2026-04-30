# テナント情報画面で MUI Card コンポーネントが未使用、設計書 admin-tenant.md §5 と乖離

## 発見日
2026-04-29

## カテゴリ
implementation / frontend / ui

## 影響度
中（機能影響なし。設計書で「Card 表示」と明記されているのに実装は素の div + dt/dd で、視覚的に「普通の文字」として表示される設計乖離）

## 発見経緯
user-report / Step 11-A SMK-102（スマホ幅でのテナント情報カード表示）検証時に、ユーザーから「会社名は表示されているが、カードではない。普通の文字」との報告。実装確認の結果、MUI `Card` コンポーネントが使われておらず、設計書規定との乖離が判明

## 関連ステップ
Step 5（画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 設計書の規定

**1. `dev-journal/deliverables/docs/50_detail_design/screens/admin-tenant.md`**

- L54-56（画面イメージ）:
  ```
  │  ナビ     │ [テナント情報カード]                                │
  │          │ │ 会社名                                        │ │
  ```
- L75: 「### テナント情報カード」セクション明記
- L91: 「初回読み込み時: テナント情報カードのスケルトンUI」

**2. `dev-journal/deliverables/docs/55_ui_component/screens/admin-tenant.md`**

- L63-66:
  > ### TenantInfoCard
  > - 配置: `pages/admin/TenantInfoCard.tsx`
  > - 責務: テナント情報を **Card コンポーネントで表示する**。ローディング中は PageSkeleton を表示し、データ取得後は TenantInfoField で会社名を読み取り専用で表示する。404 エラー時はエラーメッセージを表示する

設計書側は「**Card コンポーネントでの表示**」を明確に規定。

### 実装の状態

**`expense-saas/frontend/src/pages/admin/TenantInfoCard.tsx:41-45`**:

```tsx
return (
  <div>
    <TenantInfoField label="会社名" value={tenant.name} />
  </div>
);
```

→ コンポーネント名は `TenantInfoCard` だが、MUI `Card` / `CardContent` を使っておらず、`<div>` を直接返している。

**`expense-saas/frontend/src/pages/admin/TenantInfoField.tsx:15-22`**:

```tsx
return (
  <div>
    <dt>{label}</dt>
    <dd>{value}</dd>
  </div>
);
```

→ 平文の `<dt>` / `<dd>` のみ。スタイル（パディング・余白・タイポグラフィ）一切なし。

### 視覚的影響

- 画面上で「テナント情報」がカード形式（背景・枠線・余白付き）で目立つはずが、ただの dt/dd 平文として表示される
- ラベル「会社名」と値が縦に並ぶだけで、視覚的階層・領域分離がない
- スマホ幅でも「カードが画面幅 100% に収まる」という SMK-102 期待結果が満たせない（そもそもカードが存在しないため）

### SMK-102 期待結果との照合

`smoke_check.md:133`:
> | SMK-102 | スマホ幅でのテナント情報カード表示 | カードが画面幅 100% に収まり、テキストが途切れない。ハンバーガーメニューが表示される |

実装はカード自体が存在しないため、期待結果と乖離。

## 影響

- 設計書規定との乖離（実装が設計書の意図を反映していない）
- ユーザーが「テナント情報」セクションを認識しづらい（視覚的階層なし）
- ポートフォリオデモとして見た時、設計書通りに作られていない印象

## 提案

### 修正案: TenantInfoCard を MUI Card で実装

```tsx
// TenantInfoCard.tsx
import Card from '@mui/material/Card';
import CardContent from '@mui/material/CardContent';
import Typography from '@mui/material/Typography';
import TenantInfoField from './TenantInfoField';

export default function TenantInfoCard({ tenant, loading, error }: TenantInfoCardProps) {
  if (loading) {
    return <div data-testid="page-skeleton-card" aria-label="読み込み中" />;
  }
  if (error?.status === 404) {
    return (
      <Card>
        <CardContent>
          <Typography color="text.secondary">テナント情報が見つかりません。</Typography>
        </CardContent>
      </Card>
    );
  }
  if (!tenant) return null;
  return (
    <Card>
      <CardContent>
        <TenantInfoField label="会社名" value={tenant.name} />
      </CardContent>
    </Card>
  );
}
```

```tsx
// TenantInfoField.tsx
import Box from '@mui/material/Box';
import Typography from '@mui/material/Typography';

export default function TenantInfoField({ label, value }: TenantInfoFieldProps) {
  return (
    <Box sx={{ display: 'flex', flexDirection: 'column', gap: 0.5 }}>
      <Typography variant="caption" color="text.secondary" component="dt">{label}</Typography>
      <Typography variant="body1" component="dd" sx={{ m: 0 }}>{value}</Typography>
    </Box>
  );
}
```

メリット: 設計書通りの Card 形式、MUI 標準の見た目で他のカード系画面と一貫性が取れる
デメリット: 既存テストがあれば描画 DOM が変わるので一部更新が必要

### 修正対象ファイル

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/pages/admin/TenantInfoCard.tsx` | MUI Card / CardContent でラップ |
| `expense-saas/frontend/src/pages/admin/TenantInfoField.tsx` | Box + Typography でスタイル付き表示に変更 |
| `expense-saas/frontend/src/pages/admin/__tests__/TenantInfoCard.test.tsx` | Card レンダリング検証テスト更新 |
| `expense-saas/frontend/src/pages/admin/__tests__/TenantInfoField.test.tsx` | Typography レンダリング検証テスト更新 |

## 完了条件

- テナント情報画面で MUI Card 形式（背景・枠線・余白）で会社名が表示される
- 設計書 `admin-tenant.md` §5 規定との整合
- スマホ幅（375px）で SMK-102 期待結果を満たす（カードが画面幅 100% 収まり、テキスト途切れなし）
- 既存テスト通過 + 必要に応じてテスト更新

## ラベル

- type: bug / ui
- area: frontend / docs（設計書整合）

## 関連

- 設計書: `50_detail_design/screens/admin-tenant.md` §5（テナント情報カード）/ `55_ui_component/screens/admin-tenant.md` §TenantInfoCard
- 関連 SMK: SMK-102（スマホ幅でのテナント情報カード表示）

## MVP 区分

MVP（設計書通りの実装、UX 影響）

---

## 解決内容

**採用方針**: 設計書通り MUI `Card` / `CardContent` でラップ（案 X: PageSkeleton 共通化、案 A: dt/dd 廃止）

**実装** (PR #106, commit 4225877):
- `TenantInfoCard.tsx`: 通常表示 / 404 / loading の各ケースで `<Card><CardContent>` 構造に置換
  - 通常表示: `<Card><CardContent><TenantInfoField /></CardContent></Card>`
  - 404 エラー: `<Card><CardContent><Typography color="text.secondary">テナント情報が見つかりません。</Typography></CardContent></Card>`
  - loading: 共通 `<PageSkeleton variant="card" />`（素 div + data-testid="page-skeleton-card" を撤去、AppLayout / DashboardPage / ReportDetailPage と同一パターン）
- `TenantInfoField.tsx`: `<div><dt><dd>` を `<Box><Typography variant="caption" color="text.secondary"><Typography variant="body1">` に置換（MVP は会社名 1 項目のみで dl リストの意味が薄いため dt/dd 廃止、純粋 Box ベース）

**テスト更新**:
- TNT-FE-009 (TenantInfoCard loading): `getByTestId('page-skeleton-card')` を `getByTestId('page-skeleton')` + `toHaveAttribute('data-variant', 'card')` に更新
- TNT-FE-007 (TenantPage loading): 同上
- TNT-FE-008/010/011: getByText ベースのため変更不要

**設計書改訂**: 不要（既存の `50_detail_design/screens/admin-tenant.md` §5 / `55_ui_component/screens/admin-tenant.md` §TenantInfoCard が既に「MUI Card で表示」と規定しており、本対応で実装が設計書に追従）

## 解決日

2026-04-30
