# smoke_check.md SMK-037 の操作手順文言が実装・設計と乖離

## 発見日
2026-04-15

## カテゴリ
documentation / testing / smoke-check

## 影響度
低（ドキュメントの文言ミス。実機検証時に「ボタンが見つからない」と誤認する）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-037（添付ダウンロード起動）を実施中、smoke_check.md の手順に従って「ダウンロード」ボタンを探したが見当たらず、実装を確認した結果、添付一覧の操作トリガは「ファイル名クリック」だった。

## 関連ステップ
Step 6-D（手動チェックリスト定義）/ Step 11-A

## ブロッカー
なし（後追い改善事項）

## 事実

### 該当箇所

`dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-037:

```
| SMK-037 | ダウンロード起動 | 添付一覧 | reportSubmitted（添付あり） | Member（所有者） | レポート詳細 | 1. 添付ファイルの「ダウンロード」を押下 | ブラウザのダウンロードが開始され、ファイル名・拡張子が正しい | 開発者 |
```

### 設計書（正本）

`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §7 添付ファイル一覧 L325:

```
| 1 | ファイル名 | テキスト。クリックでダウンロード |
```

同 §7 ダウンロード L358:

```
| 操作方法 | ファイル名クリック |
```

### 実装

`expense-saas/frontend/src/pages/reports/AttachmentList.tsx` L42-51:

```tsx
<li key={att.id} data-testid={`attachment-item-${att.id}`}>
  <Button
    variant="text"
    size="small"
    onClick={() => onDownload(att.id)}
    ...
  >
    {att.file_name}
  </Button>
  ...
```

ファイル名自体がクリック可能なテキストボタンとして実装されており、「ダウンロード」というラベルのボタンは存在しない。

### 結論
- 設計書: 「ファイル名クリック」
- 実装: 「ファイル名クリック」
- SMK: 「ダウンロード」を押下 ← 誤り

設計と実装は一致しているので、SMK 側の文言を修正すべき。

## 修正方針

`smoke_check.md` SMK-037 の操作手順を以下に書き換える:

```
1. 添付ファイルの「ファイル名」テキストを押下
```

合わせて期待結果も「ブラウザのダウンロードが開始され、ファイル名・拡張子が正しい」のままで OK だが、issue 102（プレビュー機能未実装）の対応次第で「新しいタブでファイルが表示される、または保存される」に変わる可能性あり。issue 102 と合わせて再度文言調整する。

## 修正対象ファイル
- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`

## 完了条件
- SMK-037 の操作手順文言が「ファイル名クリック」に修正されている
- 同様に他の SMK チェック項目に同種の誤記がないか合わせて確認する（grep で「ダウンロード」を押下 等の表現を洗い出す）

## 関連
- issue 102: 添付ファイルのプレビュー機能未実装（設計 vs 実装乖離） — 期待結果文言を再調整する場合がある

---

## 解決内容
smoke_check.md の SMK-037 操作手順を「添付ファイルの「ファイル名」テキストを押下」に修正済み（Group E 対応時に実施）。

## 解決日
2026-04-15
