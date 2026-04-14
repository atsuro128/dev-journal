# 明細スライドパネルが「スライドパネル」として実装されていない（単なる inline 表示）

## 発見日
2026-04-14

## カテゴリ
implementation / frontend / ui / design-gap

## 影響度
高（UX 上の設計意図から明確に乖離。明細編集体験全体の品質問題）

## 発見経緯
Step 11-A ローカル動作確認 SMK-012 実施中、ユーザーから「スライドパネルって何？スライドされない」との指摘。実装を確認した結果、設計書の「画面右側からスライドイン」仕様に対して、実装は単なる `display: block/none` の inline 表示となっていることが判明

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10（機能実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし（インラインでも form としては動作する。ただし UX・設計整合性の観点で重大）

## 概要

設計書 `report-detail.md` §6 L233-235 は明細編集 UI を「画面右側からスライドインし、メインコンテンツと同時に表示する**スライドパネル**」として定義している。しかし実装は MUI Drawer 等のスライドパネルコンポーネントを使わず、単なる `<div style={{ display: open ? 'block' : 'none' }}>` によるインライン表示になっている。

結果、明細行をクリックしたり「+ 明細追加」を押したりすると、フォームが **明細一覧の下にインライン表示** される。スライドアニメーションも右側配置もなく、「スライドパネル」とは呼べない。

## 事実

### 設計書

`dev-journal/deliverables/docs/50_detail_design/screens/report-detail.md` §6 L233-275

- L235: 「明細の追加・編集は**スライドパネル**で行う（モーダルや別ページではない）。画面右側からスライドインし、メインコンテンツと同時に表示する」
- L247-275: パネルレイアウト図あり（右側パネル形式）

### 実装

`expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx` L102-130

```tsx
return (
  <div
    data-testid="item-slide-panel"
    style={{ display: open ? 'block' : 'none' }}
  >
    <div>
      <h2>{title}</h2>
      <Button variant="text" size="small" onClick={onClose}>
        閉じる
      </Button>
    </div>
    <ItemForm ... />
    <AttachmentArea ... />
  </div>
);
```

- 単なる `<div>` に `display: block/none` の切替
- MUI Drawer / Slide 等は未使用
- position: absolute / fixed / sticky なし
- `ReportDetailPage.tsx:516-535` の JSX 順序上、`ItemListSection` の直後に配置されているため、開くと明細一覧の下にインラインで現れる

## 根本原因

Step 5.5 UI コンポーネント設計では「スライドパネル」という要件が記載されたが、Step 10 実装時に MUI Drawer 等の具体的なコンポーネント選定が行われず、最小実装として `display` 切替の div に落ち込んだ可能性が高い。

## 修正方針

**案 A: MUI Drawer を採用**
- `@mui/material/Drawer` で `anchor="right"` を指定
- `open` / `onClose` を既存 props と繋ぐ
- スライドアニメーションは Drawer が標準で提供

**案 B: MUI Slide + 独自ラッパーで右側固定**
- より軽量だが、オーバーレイ・スクロール抑制・ESC クローズ等の挙動を自前実装する必要があり高コスト

**案 C: 設計書を実装に合わせて修正（インライン表示を正とする）**
- UX 品質を下げる選択。ポートフォリオとしての見栄え悪化
- スライドパネル要件を放棄する理由がない限り非推奨

### 推奨

**案 A（MUI Drawer）** を強く推奨。理由:
- 設計書に沿った実装が最小工数で実現可能
- MUI 公式コンポーネントなのでアクセシビリティ・キーボード操作・フォーカストラップ・ESC クローズ等が標準装備
- ポートフォリオとしての見栄え・UX 品質が大幅向上

## 修正対象ファイル

- `expense-saas/frontend/src/pages/reports/ItemSlidePanel.tsx` — Drawer ラップに書き換え
- `expense-saas/frontend/src/pages/reports/__tests__/ItemSlidePanel.test.tsx` — Drawer 対応のテスト修正
- `expense-saas/frontend/src/pages/reports/ReportDetailPage.tsx` — 変更不要の見込み（props インターフェース維持）

## 完了条件

- 明細行クリック / 「+ 明細追加」押下 / 「編集」ボタン押下のいずれかでスライドパネルが **画面右側から** スライドインする
- スライドパネル表示中もメインコンテンツが表示される（完全オーバーレイで背景を隠さない、または半透明 Backdrop のみ）
- 「閉じる」「ESC」「Backdrop クリック」いずれかで閉じられる
- 設計書 `report-detail.md` §6 と実装が一致する
- 既存テストが通過し、Drawer 対応テストが追加されて通過する

## 関連 issue

- 089（カテゴリラベル重複）— 同じ `ItemForm.tsx` 周辺
- 090（明細編集時プリフィルなし）— 同じスライドパネル配下
- 091（明細行クリック時挙動未定義）— スライドパネル構造改修と合わせて設計整理すべき
