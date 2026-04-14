# 添付ファイル一覧のサムネ/アイコン表示要件が smoke_check.md と設計書・実装で不整合

## 発見日
2026-04-14

## カテゴリ
design / test-spec-inconsistency

## 影響度
中（要仕様整理）

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 / SMK-030〜032 実施準備中に、SMK-030 の期待結果「サムネイル or アイコンが表示される」の根拠を確認したところ、設計書 `report-detail.md` §7.1 添付ファイル一覧に該当記載がなく、実装 `AttachmentList.tsx` にもサムネ/ファイル種別アイコン描画がないことを確認

## 関連ステップ
Step 5（詳細設計）/ Step 6（テスト設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（SMK 判定の基準が曖昧になるため要整理）

## 概要

`smoke_check.md` SMK-030 / SMK-031 / SMK-032 の期待結果は「一覧に追加され、**サムネイル or アイコンが表示される**」だが、設計書および実装のどちらにもサムネ・ファイル種別アイコン表示の定義/実装がない。

test 設計と詳細設計の仕様乖離であり、いずれかを正として他方を合わせる必要がある。

## 事実

### smoke_check.md

`dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` L114-116:
- SMK-030: 「一覧に追加され、サムネイル or アイコンが表示される」
- SMK-031: 「一覧に追加される」（SMK-030 と異なり、サムネ/アイコンへの言及なし）
- SMK-032: 「一覧に追加される」（同上）

→ SMK-030（JPEG）のみサムネ/アイコンを期待しており、SMK-031（PNG）/ SMK-032（PDF）は素の「一覧追加」のみ。内部整合性も曖昧

### 設計書

`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §7.1 L321-328

| # | 項目 | 表示形式 |
|---|------|---------|
| 1 | ファイル名 | テキスト。クリックでダウンロード |
| 2 | ファイルサイズ | `XX KB` / `X.X MB` |
| 3 | 削除ボタン | `[x]` アイコン。status == draft かつ所有者の場合のみ表示 |

→ ファイル自体のサムネ・ファイル種別アイコンの記載なし（削除ボタンの `[x]` アイコンのみ）

### 実装

`expense-saas/frontend/src/pages/reports/AttachmentList.tsx`:

```tsx
<li key={att.id} data-testid={`attachment-item-${att.id}`}>
  <Button onClick={() => onDownload(att.id)}>
    {att.file_name}
  </Button>
  <span>{att.file_size}</span>
  {canDelete && (
    <Button color="error" onClick={() => onDelete(att.id)}>削除</Button>
  )}
</li>
```

→ サムネ・ファイル種別アイコンの描画なし

## 判断の選択肢

### 案 A: smoke_check.md を正とし、設計 + 実装にサムネ/アイコンを追加

- JPEG/PNG はサムネイル（S3 署名付き URL で img 要素埋め込み）
- PDF はファイル種別アイコン（MUI Icon など）
- 設計書 §7.1 に列追加、AttachmentList.tsx に img/Icon 追加
- **工数大** — サムネ画像の取得経路が認可チェック必須、キャッシュ戦略、表示サイズ、リストパフォーマンス等の設計事項が多数発生
- ポートフォリオ MVP スコープとしては over-engineering の可能性

### 案 B: 設計書を正とし、smoke_check.md から「サムネイル or アイコン」を削除

- smoke_check.md SMK-030 の期待結果を「一覧に追加される」に修正
- SMK-031 / SMK-032 と整合
- 設計書・実装に変更なし
- **工数小**、MVP スコープとして妥当

### 推奨

**案 B**。理由:
- サムネ機能はポートフォリオ MVP の範疇で over-engineering
- 既に SMK-031 / SMK-032 は「一覧追加」のみの判定なので、SMK-030 だけ別要件は内部整合性を欠く
- 設計書が正なら `smoke_check.md` 側を修正するのが最小変更

## 修正対象ファイル（案 B 採用時）

- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` L114 SMK-030 の期待結果修正

## 完了条件

- 案 A / 案 B のいずれを採用するかユーザー判断がなされている
- 採用案に沿って設計書・実装・smoke_check.md が整合している
- SMK-030 が PASS 判定できる明確な基準を持つ
