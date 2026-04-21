# 明細詳細パネルを閉じる瞬間、スライドアニメーション中に新規追加モードのレイアウトが一瞬表示される

## 発見日
2026-04-21

## カテゴリ
implementation

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 11-A（ローカル動作確認、SMK-031 実施中の副次発見）

## ブロッカー
なし

## 問題

明細一覧から明細をクリックして明細詳細（`ItemSlidePanel` mode='view'）を開き、閉じるボタンで閉じた際、
**スライドアウトアニメーション中（パネルが画面右外へ移動する 200ms 前後）に、mode='view' ではなく mode='add' 相当のレイアウト（保存ボタン・添付アップロード UI など）が一瞬フラッシュ表示される**。

### 再現手順
1. Member でログイン → draft レポート詳細を開く
2. 明細一覧から任意の明細行をクリックして明細詳細（view モード）を開く
3. 「閉じる」ボタン（×）を押下
4. パネルがスライドアウトする間の画面右端を目視 → 保存ボタン・アップロード UI が瞬間的に表示される

### 原因推定

`ItemSlidePanel.tsx` のモード切り替え処理で以下のいずれかが発生している:

1. **親コンポーネント側で onClose の前に mode が 'add' にリセットされている**
   - `ReportDetail` 側で `selectedItemId=null` → `mode='add'` として先に state 更新 → その後 `open=false` で Drawer を閉じると、閉じるアニメーション中の DOM は既に mode='add' でレンダリングされている
2. **Drawer の条件レンダリングではなく transitionDuration 中に子要素が mode に従って再レンダリング**
   - MUI Drawer は `open=false` 後も transition 完了まで DOM を保持する設計。その間に mode='add' に props 変更されるとレイアウトがフラッシュする

### 影響

- **UX バグ**: ユーザーに「閉じる操作なのに編集画面に変わった」という誤認知を与える
- **視覚的品質低下**: ポートフォリオ品質としてはスライドアウトがクリーンに見えない
- **設計整合性**: screens/report-detail.md のモード別レイアウト仕様からの逸脱（閉じるアニメーション中は view レイアウトを保持すべき）

実害（データ・認可・ロジック）はない。

## 影響

- UX: 中（閉じる瞬間のちらつきで視覚的品質を損なう）
- セキュリティ: なし
- データ整合: なし

## 提案

### 調査タスク
1. `ReportDetail` の `ItemSlidePanel` 起動側で、パネル閉じ時に mode と selectedItemId をどの順序で更新しているか特定する
2. MUI Drawer の transitionDuration 中のレンダリング挙動を確認する

### 修正案候補

- **案 A**: 親コンポーネントで `open=false` 後、Drawer の `SlideProps.onExited` コールバックで mode/selectedItemId を reset する（アニメーション完了後に state 更新）
- **案 B**: ItemSlidePanel 内部で `open=false` の間は直前の mode を保持する（ref で previous mode をキャプチャ）
- **案 C**: Drawer を `keepMounted={false}` + 条件レンダリングで閉じる瞬間に DOM から完全に除去（アニメーション犠牲）

**案 A が推奨**（最もクリーン、アニメーション維持）。

### テスト追加
- `ItemSlidePanel.integration.test.tsx` に「view モードを閉じる時に mode='add' の DOM が表示されない」回帰テストを追加

---

## 解決内容

## 解決日
