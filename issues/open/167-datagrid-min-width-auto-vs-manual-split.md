# DataGrid 列の自動幅 minWidth と手動リサイズ minWidth の分離（post-MVP）

## 発見日
2026-05-03

## カテゴリ
ui-design / post-mvp

## 影響度
低（MVP の機能要件・読み取り可能性には影響なし。列幅ユーザーカスタマイズ時の利便性改善）

## 発見経緯
issue #160 再対応レビュー中、ユーザーから「現状の minWidth は MUI X DataGrid 仕様で flex sizing と resize handle の両方に適用される。手動で列幅をドラッグ調整した場合は、設定した minWidth より小さくできた方が便利」との改善要望が寄せられた

## 関連ステップ
Step 5.5（UI コンポーネント設計）/ Step 10-B（レポート UI 実装）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外として扱う。MVP リリース前には対応しない。管理方式は **ops-080** で検討中。

## 問題

### MUI X DataGrid 標準仕様

MUI X DataGrid の `minWidth` プロパティは flex sizing（container resize / 自動幅計算）と resize handle（ユーザーによる列ヘッダードラッグ）の**両方**に一律適用される。Community / Pro どちらも、自動幅調整時の最小値とユーザー手動リサイズ時の最小値を分離する props は標準では存在しない。

### 現状の挙動

issue #160 対応で各列の `minWidth` を引き上げたことにより、コンテナリサイズ時の ellipsis は抑止できている。一方、ユーザーが列ヘッダーをドラッグして列幅を縮小する際も同じ `minWidth` が下限として機能するため、意図的に列を狭めて他の列を広く見たい場合に制約が強すぎる。

### ユーザー要望

- 自動幅調整時（container resize / flex 計算）: 現状の `minWidth` で打ち止め（ellipsis を防ぐ）
- ユーザーが列ヘッダーをドラッグして手動リサイズ: 設定 `minWidth` より小さくできる（ユーザー責任で省略表示を許容）

## 提案

post-MVP 着手時に以下の案を検討する。

### 案 A: onColumnResize / onColumnWidthChange でカスタムロジック

リサイズイベントを捕捉し、手動リサイズ操作時に限り `minWidth` 制約を迂回する独自処理を追加する。

- 利点: DataGrid の列定義を大きく変えずに対応できる可能性がある
- 課題: MUI X 内部の resize 制約ロジックと整合させる難易度が高い。バージョンアップで挙動が変わるリスク

### 案 B: minWidth を低めに設定 + 別の仕組みで auto-fit 最小値を担保

columnDef の `minWidth` は手動リサイズ用の最小値（例: 50px）に設定し、`useResizeObserver` 等でコンテナ幅を監視して auto-fit 時は CSS で見かけ上の最小幅を当てる。

- 利点: 手動リサイズの自由度が高まる
- 課題: MUI X の内部レイアウトロジックと干渉するリスク。実装複雑度が上がる

### 案 C: 列ヘッダーをカスタム実装して MUI X の制御を迂回

DataGrid の resize ハンドラを自前実装し、`minWidth` 制約を完全に独自管理する。

- 利点: 自動幅と手動リサイズを完全に分離できる
- 課題: メンテコスト大、MUI X バージョンアップへの追従コスト大

### 推奨判断

案 A から検討する。着手前に MUI X 公式 GitHub の関連 issue・Stack Overflow・公式ドキュメントを再調査し、標準的なアプローチが出ていないか確認してから方針を確定する。

## MVP との関係

- MVP では issue #160 で引き上げた `minWidth` 値により、読み取り可能性は担保済み
- 本要望は「列幅ユーザーカスタマイズの利便性」改善であり、MVP の機能要件には含まれない
- 着手タイミング: post-MVP のフロントエンド UX 改善フェーズ

## 関連

- **#160**（resolved）: DataGrid mobile 横スクロール + 列見切れ対応。本 issue の起点となった minWidth 引き上げ対応
- **設計書** `55_ui_component/common-components.md §AppDataGrid`: DataGrid 共通コンポーネント設計
- **#081**（open, post-MVP）: MVP スコープ外のテスト観点一括管理。post-MVP 管理方式の参考
- **ops-080**（open）: MVP スコープ外事項の管理方式決定

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
