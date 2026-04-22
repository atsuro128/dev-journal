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

## 調査結果（2026-04-21）

### 再現箇所の特定

- `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx`
  - L106: `const [panelState, setPanelState] = useState<PanelState>('closed');` — `'closed' | 'add' | 'edit' | 'view'` の 4 値統合 state
  - L561-582: `<ItemSlidePanel ... />` 呼び出し
    - L563: `open={panelState !== 'closed'}`
    - L564: `mode={panelState === 'closed' ? 'add' : panelState}` — **`panelState === 'closed'` のとき mode は `'add'` にフォールバック**
    - L569: `onClose={() => setPanelState('closed')}` — 閉じるボタン押下で panelState を即 `'closed'` に更新
    - L570: `onSaveSuccess={() => setPanelState('closed')}`
- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx`
  - L521-529: `<Drawer anchor="right" open={open} onClose={handleCloseAttempt} PaperProps={{...}} >` — **`SlideProps` / `TransitionProps` / `onExited` / `keepMounted` / `transitionDuration` の指定なし**（MUI Drawer デフォルト挙動：`open=false` 後も Slide transition 完了まで DOM 保持・約 225ms）
  - L154: `const title = mode === 'add' ? '明細追加' : mode === 'edit' ? '明細編集' : '明細詳細';`
  - L269: `const canModify = isOwner && reportStatus === 'draft' && mode !== 'view';` → mode='add' のとき `canModify=true` になり、ItemForm と AttachmentArea の両方で編集 UI が活性化
  - L270: `const formMode = canModify ? mode : 'view';` → mode が 'add' に流れると formMode も 'add' に切り替わる
  - L582: `onPendingFilesChange={mode === 'add' ? handlePendingFilesChange : undefined}` — mode 依存でコールバック切替
- `expense-saas/frontend/src/pages/reports/ItemForm.tsx`
  - L128-129: `const isView = mode === 'view'; const isAdd = mode === 'add';`
  - L320-347: `{!isView && ( ... 保存ボタン・「保存して続けて追加」ボタン ... )}` — **mode が 'view' から 'add' に切り替わると保存ボタン等がレンダリングされる**
  - L328: `{isAdd && onSaveAndContinue && (...)}` — 「保存して続けて追加」は mode='add' でのみ表示
- `expense-saas/frontend/src/pages/reports/AttachmentArea.tsx`
  - L284-293: `if (mode === 'add') { return <AttachmentAreaAddMode ... /> }` — **mode='add' に切り替わると pendingFiles 保留型の AttachmentUploader UI へ分岐**

該当コードスニペット（ReportDetailPage.tsx L561-582, 抜粋）:

```tsx
<ItemSlidePanel
  key={formKey}
  open={panelState !== 'closed'}
  mode={panelState === 'closed' ? 'add' : panelState}  // ← closed の間は mode='add' にフォールバック
  reportId={report.id}
  item={selectedItem}
  reportStatus={report.status}
  isOwner={isOwner}
  onClose={() => setPanelState('closed')}              // ← 即 panelState='closed' に更新
  onSaveSuccess={() => setPanelState('closed')}
  ...
/>
```

ItemSlidePanel.tsx L521-529（Drawer 設定）:

```tsx
<Drawer
  anchor="right"
  open={open}
  onClose={handleCloseAttempt}
  PaperProps={{ 'data-testid': 'item-slide-panel', sx: {...} } as PaperProps}
>
  {/* SlideProps / TransitionProps / onExited / keepMounted / transitionDuration の指定なし */}
```

### 仮説検証

| 仮説 | 判定 | 根拠 |
|------|------|------|
| 1: 親で onClose 前に mode リセット | **成立（主因）** | ReportDetailPage.tsx L564 で `mode={panelState === 'closed' ? 'add' : panelState}` としているため、onClose (L569) で panelState='closed' に更新された瞬間、次レンダリングで ItemSlidePanel の mode prop が 'view' → 'add' に切り替わる。`selectedItem` は L107 の state で保持されているがモード判定は panelState のみに従う |
| 2: MUI Drawer transition 中の props 変更 | **成立（フラッシュが見える条件）** | ItemSlidePanel.tsx L521-529 で Drawer に `open=false` 以外の transition 制御（`keepMounted` / `SlideProps.onExited` / `transitionDuration`）を何も指定していないため、MUI デフォルトの Slide transition（約 225ms）で DOM が保持される。その間に mode='add' への props 変更が反映されて保存ボタン・AttachmentAreaAddMode が再レンダリングされる |

### 原因の結論

**仮説 1 と仮説 2 は単独ではなく合成で発生している。** 直接のトリガーは仮説 1（親 state `panelState='closed'` への遷移が `mode` 派生値を 'add' に落とすこと）、フラッシュが視覚化される条件は仮説 2（Drawer がスライドアウト中も DOM を保持すること）。`panelState` を 'closed' / 'add' / 'edit' / 'view' の単一 state に集約した設計上、`panelState === 'closed'` のとき `mode` に渡すフォールバック値として `'add'` を選択している L564 の式そのものが本質原因。

### 推奨案（案 A/B/C から選択）

**推奨: 案 A（Drawer の `SlideProps.onExited` コールバックで親 state をリセット）**

- 根拠:
  - **実装コスト**: ItemSlidePanel に `onExited` を onExited として親に通知する prop を追加 + ReportDetailPage で onClose を「open=false のみトリガー」と「exited 完了後に selectedItem リセット等の掃除」に分離する、15-25 行規模の変更で済む。現行の `panelState` 統合 state は維持できる（L564 の式は据え置きでよい）
  - **MUI 公式パターンとの整合性**: MUI Drawer は `TransitionProps={{ onExited: ... }}` を公式サポート。`keepMounted` や `transitionDuration: 0` のような副作用のある回避策に比べてクリーン
  - **他 issue との整合性**:
    - #132 (dirty state リセット): 保存成功 → Drawer 閉じ時の state リセット順序を onExited に寄せれば、#132 の「保存成功後の再オープンで dirty が残る」系とも同じ仕掛けで解決可能（共通論点）
    - #129 (pending files): 追加モードの pendingFiles も onExited で確実にクリアできる（現状は破棄ダイアログ経由でしかクリアされない）
  - **アニメーション自然さ**: Drawer の Slide アニメーションは変更せず維持できる
- 想定変更箇所:
  - `ItemSlidePanel.tsx` L521-529: `<Drawer>` に `SlideProps={{ onExited: onExited }}` あるいは `TransitionProps={{ onExited }}` を追加、props interface に `onExited?: () => void` を追加
  - `ReportDetailPage.tsx` L569-582: `onClose={() => setPanelState('closed')}` は維持しつつ、`onExited` で `setSelectedItem(null); setItemApiError(null);` 等を実行する分離を追加。ただし `panelState='closed'` の時点で mode が 'add' に落ちる問題は別途解決が必要 → **L564 の式を `mode={panelState === 'closed' ? (lastModeRef.current ?? 'add') : panelState}` のような「直前モード保持」にするか、あるいは `panelState` を 'closing-view' / 'closing-edit' / 'closing-add' のような中間状態に拡張する**
- 想定変更規模: 20-30 行程度（ItemSlidePanel に onExited prop 追加 + ReportDetailPage に直前モード保持 ref または中間 state + onExited でリセット）
- 他の案を退ける理由:
  - **案 B（ItemSlidePanel 内部で直前 mode を ref 保持）**: 親から流れる mode prop を「内部で書き換える」挙動になり、props の信頼性が下がる。テストで mode を切り替えて検証する際の前提が壊れる。ItemSlidePanel の責務拡大
  - **案 C（`keepMounted={false}` + 条件レンダリング）**: MUI Drawer の `keepMounted` のデフォルトは既に `false` のため、これだけではスライド transition 中の DOM 保持挙動を変えられない。代わりに親で `{panelState !== 'closed' && <ItemSlidePanel ... />}` と条件レンダリングすると Drawer 自体が unmount されてスライドアウトアニメーションが失われる → UX 低下。アニメーション犠牲のため不採用

### 追加論点

- **#132 (dirty state リセット) との相互作用**: 保存成功 → Drawer 閉じ時の state リセット順序は #132 と共通論点。案 A で `onExited` コールバックに「selectedItem リセット」「itemApiError リセット」「formKey インクリメント」をまとめると両 issue を一手で解決できる可能性あり（#132 の実装時に本 issue の onExited 基盤を流用）
- **#129 (pending files) との相互作用**: 追加モードを × で閉じたとき、現状は `isDirty` 判定 → 破棄ダイアログ経由で `setPendingFiles([])` が呼ばれるが、onExited で常時クリアするようにすれば「非 dirty で閉じた後に再度 add で開いた際の pendingFiles 残留」等のエッジも防げる
- **案 A 実装時の注意**: `panelState === 'closed'` のとき mode フォールバックを 'add' のままにすると本 issue は解決しない。`lastModeRef` で直前の非 closed モードを保持し、closed のときはそれを mode に渡す必要がある。あるいは panelState を 'closed' ではなく `{ open: boolean, mode: PanelMode }` のような構造体に分離する設計も検討余地あり（ただし L46 コメントにあるとおり「open 遷移が必ず closed → open になりアニメーションが一貫」させる現行設計を崩さない形で）

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

PR #86（commit e73a659）にて対応。`ItemSlidePanel` の `Drawer` に `TransitionProps={{ onExited }}` を追加し、スライドアウトアニメーション完了後に親 state をリセットする方式（案 A）を採用。アニメーション中は直前モードを保持することでレイアウトフラッシュを解消。

## 解決日

2026-04-22
