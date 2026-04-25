# ReportInfoCard 常時表示 4 項目のラベル欠落 + 作成日フォーマット設計乖離

## 発見日
2026-04-24

## カテゴリ
implementation / ui-design

## 影響度
中（視覚的情報設計の欠落。機能動作に影響なし、ユーザーが「この日付は何？」と判断できない認知負荷問題）

## 発見経緯
Step 11-A SMK-070（JST 表示・提出日時）実施中、ユーザー指摘「Test Member の下の日付は何？」から ReportInfoCard の常時表示 4 項目でラベル欠落が組織的に発生していることが判明

## 関連ステップ
Step 5（API 詳細設計 / 画面詳細設計）/ Step 5.5（UI コンポーネント設計）/ Step 10-B（レポート UI 実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 関連 issue

- **#139**（resolved, PR #92）: レポート一覧のステータス列英語露出。同様の設計 vs 実装乖離パターン
- **#140**（open）: フォーム必須マーカー `*` 不整合。UI 表現の統一課題として関連
- SMK-050（§4.6 日本語 UI）: 本 issue の観察視点が欠落していた SMK。再発防止の参考

## 問題

### 実装 vs 設計の差異（`ReportInfoCard.tsx:55-71` と `50_detail_design/screens/report-detail.md §2 / §3`）

設計書 §2 レイアウトモック（L49-53）:

```
| タイトル: XXXXXX           [ステータスバッジ]  |
| 対象期間: YYYY/MM/DD ~ YYYY/MM/DD            |
| 合計金額: \XX,XXX                            |
| 作成者: XXXX       作成日: YYYY/MM/DD        |
```

実装（`ReportInfoCard.tsx:62-70`）:

```tsx
<p>
  {formatDateSlash(report.period_start)} 〜 {formatDateSlash(report.period_end)}
</p>
<p data-testid="total-amount">¥{report.total_amount.toLocaleString()}</p>
<p>{report.submitter?.name ?? ''}</p>
<p>{formatDateTimeJa(report.created_at)}</p>
```

→ **4 項目すべてでラベルが欠落**（値だけが羅列される状態）。

ラベル有無マトリクス:

| 項目 | 設計書（ラベル） | 実装 | 判定 |
|------|---------------|------|------|
| タイトル | （見出し、ラベル不要） | `<h2>` | ✓ |
| ステータス | （バッジ、ラベル不要） | `<StatusChip>` | ✓ |
| 対象期間 | 「対象期間:」 | ラベルなし | ❌ 欠落 |
| 合計金額 | 「合計金額:」 | ラベルなし | ❌ 欠落 |
| 作成者 | 「作成者:」 | ラベルなし | ❌ 欠落 |
| 作成日 | 「作成日:」 | ラベルなし + フォーマット乖離 | ❌ 欠落 + 乖離 |
| 提出日 | 「提出日:」 | ラベルあり | ✓ |
| 承認者 | 「承認者:」 | ラベルあり | ✓ |
| 承認日 | 「承認日:」 | ラベルあり | ✓ |
| 支払処理者 | 「支払処理者:」 | ラベルあり | ✓ |
| 支払日 | 「支払日:」 | ラベルあり | ✓ |

→ **常時表示 4 項目だけが組織的に欠落**。条件付き表示（ワークフロー系）は設計通り動作。

### 作成日のフォーマット乖離

- 設計書 `50_detail_design/screens/report-detail.md:53, 99`: `YYYY/MM/DD`（日付のみ）
- 実装（`formatDateTimeJa`）: `YYYY/MM/DD HH:mm`（時刻付き、例: `2026/04/23 15:56`）

## 対応方針（ユーザー承認済み、案 B）

**案 B**: 実装側（時刻付き）に合わせて設計書を改訂。ワークフロー系の日時（提出日・承認日・却下日・支払日）と統一されたフォーマットで監査目線でも情報量が担保される。

却下案:
- **案 A**: 設計書通り `YYYY/MM/DD` に実装を戻す → ワークフロー系と不整合・情報量低下で却下
- **案 C**: ラベルのみ修正、フォーマットは後回し → 追加 issue 起票が必要で非効率のため却下

## 根本原因と再発防止

- SMK-050 実施時、Member 画面全体を巡回したが「ラベル有無の網羅確認」の観点が弱かった（ユーザーの「これは何？」気づきで初めて露出）
- ReportInfoCard の Vitest でラベル文字列を検査していれば捕まった可能性あり
- Step 11-D（横断レビュー）で codex に「設計 vs 実装の乖離監査」を依頼予定（session-log.md 2026-04-22 セッション記録済み）

## 提案

### 実装

ファイル: `expense-saas/frontend/src/pages/reports/ReportInfoCard.tsx` の §常時表示セクション（L62-70）

- L62-64（対象期間）: `対象期間: YYYY/MM/DD 〜 YYYY/MM/DD` 形式に変更
- L66（合計金額）: `合計金額: ¥XX,XXX` 形式に変更
- L68（作成者）: `作成者: XXXX` 形式に変更
- L70（作成日）: `作成日: YYYY/MM/DD HH:mm` 形式に変更（案 B、`formatDateTimeJa` を維持）

ラベル表現は設計書モック（L49-53）の `XXX: ` 形式を採用。実装時はレイアウト維持（モバイル幅で折返し等の挙動に注意）。

### 設計書改訂

1. `50_detail_design/screens/report-detail.md`
   - L99（§3 常時表示項目表）: `作成日 | YYYY/MM/DD` → `作成日 | YYYY/MM/DD HH:mm`
   - L53（§2 レイアウトモック内の `作成日: YYYY/MM/DD`）: `作成日: YYYY/MM/DD HH:mm` に更新
   - 表記用語の統一: 「作成日」で統一するか「作成日時」に変更するかは本 issue 対応者が判断。実装のラベル文言と設計書の項目名を一致させること（ワークフロー系は「提出日」「承認日」「支払日」で時刻付きだが「日」とだけ命名されているため、同じ命名規則に従い「作成日」で統一する方針を推奨）。

2. `55_ui_component/screens/report-detail.md`
   - L98, L115, L128 周辺の `createdAt` 関連箇所を、実装 props のフォーマットと矛盾しない記述に更新
   - 具体的には L115「作成日」と L128「作成日 | report.created_at」の表現を改変不要（型定義の範囲内）。フォーマット仕様は `50_detail_design` 側に一本化する方針で OK

### テスト

1. `expense-saas/frontend/src/pages/reports/__tests__/ReportInfoCard.test.tsx`: 以下のアサーション追加
   - 「対象期間:」ラベル表示
   - 「合計金額:」ラベル表示
   - 「作成者:」ラベル表示
   - 「作成日:」ラベル表示
   - 作成日の時刻付きフォーマット（例: `2026/04/23 15:56`）

2. `dev-journal/deliverables/docs/60_test/test_cases/reports.md`
   - RPT-FE 系に新規テストケース（ラベル + フォーマットの検証）を追加
   - ID 採番: `reports.md` 内の既存最大 ID +1（既存は RPT-FE-107 を本セッションで予約済みのため、本 issue では RPT-FE-108 以降を使用）

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ReportInfoCard.tsx`
- `expense-saas/frontend/src/pages/reports/__tests__/ReportInfoCard.test.tsx`
- `dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md`
- （必要に応じて）`dev-journal/deliverables/docs/55_ui_component/screens/report-detail.md`
- `dev-journal/deliverables/docs/60_test/test_cases/reports.md`

## 完了条件

- ReportInfoCard の常時表示 4 項目（対象期間・合計金額・作成者・作成日）にラベルが表示される
- 作成日が `YYYY/MM/DD HH:mm` 形式で表示される
- 設計書が新仕様を反映（`50_detail_design/screens/report-detail.md` §2 L53 と §3 L99）
- Vitest ケースがラベル + フォーマットを検証
- test_cases/reports.md に新規 ID が追記
