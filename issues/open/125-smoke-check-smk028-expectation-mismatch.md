# smoke_check.md SMK-028 期待文言が state-management.md に未定義（docs 不整合）

## 発見日
2026-04-20

## カテゴリ
docs / consistency / test-specification

## 影響度
低（docs 整合性の問題。実装の正本は state-management.md）

## 発見経緯

Step 11-A ローカル動作確認 SMK-028（ネットワーク断絶時の挙動）の再検証中、smoke_check.md の期待文言「通信エラーが発生しました。再試行してください」および「再試行ボタン」が、state-management.md や画面設計書のどこにも定義されていないことを確認。

併せて SMK-028 の検証手順「ページ更新」が曖昧（F5 解釈だとブラウザネイティブのオフライン画面が出て SPA 挙動を検証できない）ことも判明。

SMK-024 / SMK-026 と同種の smoke_check.md 不整合（issue 122 / 123 の姉妹 issue）。

## 関連ステップ
Step 6-D（テスト設計: smoke_check.md 作成）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### smoke_check.md SMK-028

```
| SMK-028 | ネットワーク断絶時の挙動 | 任意操作 | なし | Member | レポート一覧 | 1. DevTools でオフライン化 2. ページ更新 | 「通信エラーが発生しました。再試行してください」とエラー表示し、再試行ボタンが表示される |
```

### state-management.md（正本）

- `SERVER_ERROR_MESSAGES` マップには「通信エラーが発生しました」という文言は存在しない
- 500 `INTERNAL_ERROR` の文言は「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」
- 「再試行ボタン」に相当するコンポーネント / 挙動は state-management.md にも `common-components.md` にも定義なし
- エラー時の UI は Snackbar（一過性）として規定されており、永続表示の再試行ボタンは設計外

### 検証手順の曖昧性

- 「ページ更新」= F5 リロード解釈だと、オフライン状態でブラウザが index.html を取得できずブラウザネイティブのオフライン画面が表示される
- この状態では SPA 自体が起動せず、SPA 層のエラーハンドリングを検証できない
- DevTools Offline は fetch を即 reject せず pending で保留する挙動のため、SPA 内操作でもエラーを引き出せない（検証ツールのアーティファクト）

### 実環境でエラーを引き出す正しい手順

- バックエンド停止（`docker compose stop api`）により Vite proxy 経由で 500 が返る
- ただしこの場合は「ネットワーク断絶」ではなく「バックエンド停止」になり、観点がずれる
- 真の「ネットワーク断絶」を検証する手段は SPA 層では限定的。SMK-028 の観点自体を見直す必要があるかも

## 修正方針

### 1. SMK-028 の期待文言を state-management.md に揃える

500 系エラーを検証する観点に切り替える:

```
- 1. DevTools でオフライン化 2. ページ更新 → 「通信エラーが発生しました。再試行してください」とエラー表示し、再試行ボタンが表示される
+ 1. ターミナルで `docker compose stop api` で API を停止 2. SPA 内で任意の操作（ダッシュボード遷移等）を行う → Snackbar（エラー）に「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」が表示される
```

### 2. SMK-028 観点の再定義を議論

「ネットワーク断絶時の挙動」を MVP で検証する意義があるか、検証方法があるかを議論。検証不能・MVP 対象外とするなら SMK-028 を廃止 or post-MVP 送りにする。

## 修正対象ファイル

- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md` SMK-028

## 対応条件

- SMK-028 の期待文言が state-management.md と整合
- SMK-028 の検証手順が実環境で実行可能な内容になっている
- （任意）SMK-028 観点の見直し議論が記録されている

## 関連

- issue 122: SMK-024 由来の smoke_check 不整合（post-MVP の UX 改善と一緒に管理）
- issue 123: SMK-026 由来の smoke_check 不整合（docs のみ）
- issue 124: API クライアントで `SERVER_ERROR_MESSAGES` マッピングが未適用（SMK-028 が本 issue とセットで発覚させた）
- 将来同種の不整合を追加発見した場合は都度単独 issue 起票（11-A 運用ルール）
