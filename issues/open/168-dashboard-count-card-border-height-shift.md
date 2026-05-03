# ダッシュボードカード default accentColor 時に borderTop 不在で 3px 高さズレ

## 発見日
2026-05-03

## カテゴリ
ui-design

## 影響度
中（視覚的整合性、機能影響なし）

## 発見経緯
user-report / ユーザーがダッシュボードのレポートカード視覚確認時に発見。Member ロールの「下書き」「提出中」カードと「却下」カードを並べた際、border ありなしで高さに 3px のズレが生じている。Admin ロールの TenantStatusCards も同様（「全レポート」カードのみ default で border なし）。

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

`frontend/src/components/dashboard/CountCard.tsx:36-41` の sx で `accentColor !== 'default'` の場合のみ `borderTop: 3px solid` を付与しているため、default カードは 3px 短くなる。

### 影響範囲

| 画面 | 影響カード |
|---|---|
| Member ダッシュボード（MyReportCountCards）| 下書き / 提出中 が「却下」より 3px 短い |
| Admin ダッシュボード（TenantStatusCards）| 全レポート が他 4 カードより 3px 短い |

## 影響

- 同じ Grid 行内でカード高さが揃わず、視覚的な不整合が生じる
- ポートフォリオデモで「UI の精度が低い」印象を与える可能性がある
- 機能への影響はなし

## 提案

### 採用方針：案 A（推奨）

borderTop を常時 `'3px solid'` で確保し、default 時は `borderColor: 'transparent'` で透明化する。

```tsx
// CountCard.tsx:36-41
sx={{
  cursor: href ? 'pointer' : 'default',
  borderTop: '3px solid',
  borderColor: accentColor !== 'default' ? `${accentColor}.main` : 'transparent',
}}
```

メリット:
- 1 ファイル 2 行変更で完結
- 視覚的見た目は不変（default は元から色なし、transparent でも同じ）
- DOM 構造維持、テストへの影響最小

### 比較案

- 案 B: default 時に padding-top: 3px で高さ補正 → padding と border の見た目差異リスクあり
- 案 C: 全カードに paddingTop を統一指定し border を装飾扱い → CountCard の責務拡大

→ **案 A 採用**

## 修正対象ファイル

| ファイル | 変更内容 |
|---|---|
| `frontend/src/components/dashboard/CountCard.tsx` L36-41 | borderTop 常時付与 + default 時 borderColor transparent |
| `frontend/src/components/dashboard/__tests__/CountCard.test.tsx` | 必要に応じて高さ・border 関連テスト更新 |
| `frontend/src/components/dashboard/__tests__/MyReportCountCards.test.tsx` | 必要に応じて更新 |
| `frontend/src/components/dashboard/__tests__/TenantStatusCards.test.tsx` | 必要に応じて更新 |

## 完了条件

- 全カードの高さが揃う（同じ Grid 行内で視覚的に整合する）
- DevTools でダッシュボード（Member / Admin）のカード高さが統一されていることを確認
- 既存テストへの影響なし（または適切に更新済み）

## SMK 再検証要項目

- ダッシュボード（Member / Admin）の DevTools 視覚確認: カード高さ統一

## ラベル

- type: bug / ui
- area: frontend

## 関連

- 設計書 `dev-journal/deliverables/docs/55_ui_component/screens/dashboard.md` §CountCard
- 関連 issue: #164（Admin ダッシュボードカード幅問題）
- 関連 PR (TBD): `step11/issue-168-card-border-height`

## MVP 区分

MVP（視覚的整合性、ポートフォリオデモ印象に直結）

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
