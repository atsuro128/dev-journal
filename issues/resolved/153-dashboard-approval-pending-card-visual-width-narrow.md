# ダッシュボード「承認待ち」カードの見た目幅が他のカードより狭く見える（領域幅は同じ、表示のみ縮む）

## 発見日
2026-04-28

## カテゴリ
implementation / frontend / ui

## 影響度
中（機能影響なし。ビジュアルの不整合で Approver の主要動線が視覚的に弱く見える）

## 発見経緯
user-report / Step 11-A SMK-006 後の Approver ダッシュボード目視確認時に、ユーザーが「承認待ちカードの幅が他のカードに比べて狭い」と報告。再確認で「カードに割り当てられている領域は他カードと同じだが、カード本体の見た目だけが狭く描画されている」と判明

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現象

- Approver でダッシュボードを開くと、承認待ちカードの**カード本体の幅**が、Member 系の下書き / 提出中 / 却下カードよりも狭く描画される
- Grid 領域（割り当てられた幅）は他カードと同じ（`size={{ xs: 12, sm: 4 }}`）
- カード自体だけが領域内で左寄せされ、内部コンテンツの幅にフィットしているように見える

### 該当箇所

- `expense-saas/frontend/src/components/dashboard/CountCard.tsx:64-73`
- `expense-saas/frontend/src/pages/dashboard/DashboardPage.tsx:119-131`（承認待ち）/ `:133-146`（支払待ち、未確認だが同事象が発生する可能性あり）

### 推定原因（要 DOM 検証で確定）

`CountCard.tsx` で `showBadge && count >= 1` のとき、Card を MUI `<Badge>` で wrap している:

```tsx
<Badge
  color="error"
  variant="dot"
  anchorOrigin={{ vertical: 'top', horizontal: 'right' }}
  aria-label="要対応あり"
  sx={{ width: '100%' }}
>
  {cardContent}
</Badge>
```

MUI `Badge` のデフォルト `display` は `inline-flex` で、`sx={{ width: '100%' }}` を指定しても親要素（Grid item）の幅まで広がらないケースがある。結果として Badge 内側の Card がコンテンツ幅まで縮み、視覚的に「カードが狭い」と見える。

一方、Member 系カード（下書き / 提出中 / 却下）は `showBadge` 未指定 → Badge wrapper なし → Card が Grid item 幅まで広がる。これが「狭く見える / 見えない」の差異と整合する。

**注**: 上記は実装コードからの推測であり、DOM Inspector による computed style 検証で確定が必要。実際は Badge wrapper の display 以外の要因（Badge 内部の transform / padding / 別の sx 衝突）が原因の可能性もある。

### 関連

支払待ちカード（Accounting ロール、`DashboardPage.tsx:133-146`）も同じ実装パターン（`showBadge={true}` + Badge wrapper）のため、同事象が発生する可能性が高い。Phase 4 Accounting 検証時に併せて確認すべき。

## 影響

- Approver の主要動線である「承認待ち」カードが視覚的に弱く見え、UX 上の重要度と表示の重みが一致しない
- Accounting の「支払待ち」カードも同様に縮んでいる可能性が高く、同種の視覚不整合がある

## 提案

### Step 1: DOM 検証で原因確定

ブラウザ DevTools で承認待ちカードの DOM 構造を確認:

- `<span class="MuiBadge-root">` の computed style `display`, `width`
- Badge 内側の `<Card>` の computed style `width`
- 同画面内の Member 系カード（下書き等）の Card との比較

### Step 2: 原因に応じた修正

**ケースA: Badge の display が `inline-flex` で width 100% が効かない場合**

CountCard の Badge に `display: 'block'` を追加するか、Badge wrapper を Box で囲む構造に変更:

```tsx
<Badge
  color="error"
  variant="dot"
  anchorOrigin={{ vertical: 'top', horizontal: 'right' }}
  aria-label="要対応あり"
  sx={{ display: 'block', width: '100%' }}
>
  {cardContent}
</Badge>
```

**ケースB: 別の要因（transform / padding / 他 sx 衝突）の場合**

DOM 検証結果に応じて修正案を確定する。

### 注意点

issue #152（Badge dot の意図伝達不足）で showBadge 削除の方針が採用される場合、本 issue は **副次解消**（Badge wrapper 自体がなくなる）する可能性がある。両 issue の対応方針を相互参照しながら進めること。

## 修正対象ファイル（推定）

| ファイル | 変更種別 |
|---|---|
| `expense-saas/frontend/src/components/dashboard/CountCard.tsx` | Badge wrapper の display / 構造修正 |
| `expense-saas/frontend/src/components/dashboard/__tests__/CountCard.test.tsx` | 修正後のレイアウト検証テスト追加（既存テストに影響あれば更新） |

## 完了条件

- 承認待ちカード（および支払待ちカード）の見た目幅が他のカードと一致する
- DOM 検証で根本原因が特定され、修正内容が原因に対応している
- 既存テストが通る + 必要に応じて新規テスト追加

## ラベル

- type: bug / ui
- area: frontend

## 関連

- issue #152（Badge dot の意図伝達不足）— 同じ実装箇所、対応方針が連動する可能性
- 関連 SMK: SMK-006（Approver グローバルナビ確認）の副次発見

## MVP 区分

MVP（UX 違和感、ユーザーが直接目にする）

---

## 解決内容

**副次解消**: issue #152 で CountCard の `showBadge` プロパティと Badge wrapper を削除した結果、Badge wrapper の `display: inline-flex` が原因と推定されていた幅縮みも自動的に解消した。

**実装** (PR #108, commit 6916ed0、issue #152 と同一コミット):
- CountCard.tsx の Badge wrapper 撤去によりカード本体が直接 Grid item の幅まで広がる構造になった
- 単独対応コミットなし（#152 で副次解消）

**検証必要 SMK**:
- SMK-006（Approver ダッシュボード）で承認待ちカードの幅が他カードと一致することを目視確認
- Phase 4 Accounting 検証時に支払待ちカードも同基準で確認

## 解決日

2026-04-30
