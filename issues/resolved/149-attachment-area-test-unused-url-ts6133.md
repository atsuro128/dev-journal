# AttachmentArea.test.tsx の未使用引数 url で TS6133 が発生（master 直接編集による回帰）

## 発見日
2026-04-25

## カテゴリ
implementation / regression / test-quality

## 影響度
中（ローカル CI（`npx tsc --noEmit` / `npm run build`）が失敗。本番 CI（GitHub Actions）が現在停止中のため master HEAD に紛れ込んだまま放置されていた）

## 発見経緯

issue #141 / #144 / #143 の実装 PR（PR #95 / #94 / #96）に対して指揮役がローカル CI を並列実行したところ、3 ブランチすべてで同一の TS6133 エラーが発生。各 PR の差分を調査した結果、master HEAD（`4491c073`）の `AttachmentArea.test.tsx` に既に未使用引数が存在することが判明。

## 関連ステップ

Step 11-A（ローカル動作確認）/ Step 8 / 9（基盤・テストコード実装）

## ブロッカー

`AttachmentArea.test.tsx` を変更する PR（特に PR #96 = #143）で、ローカル CI を完全 PASS させるためには master 側の修正が先行する必要がある。

## 関連 issue

- **#141 / #143 / #144**: 本 issue 修正後にリベースが必要（影響範囲）
- ops-064（MCP ジョブログ proxy allowlist）: 本番 CI 停止の遠因となる egress 問題

## 問題

### 該当箇所

`expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.test.tsx:339`（master HEAD 時点の行番号）

```typescript
globalThis.fetch = vi.fn().mockImplementation((url: string, opts?: RequestInit) => {
  //                                            ^^^^ ← 関数本体で参照されていない
  if ((opts?.method ?? 'GET').toUpperCase() === 'DELETE') {
    return Promise.resolve(deleteResponse);
  }
  // url を一切参照せず opts のみで分岐する
  ...
});
```

### TS6133 の内容

```
src/pages/reports/__tests__/AttachmentArea.test.tsx(339,52): error TS6133: 'url' is declared but its value is never read.
```

TypeScript の `noUnusedParameters` ルール違反。`url: string` 引数が宣言されているが、関数本体で一度も参照されていない。

### なぜ master に紛れ込んだか

- 直近の master 直接編集（PR フローを経由しないコミット）で混入したと推定
- 本番 CI（GitHub Actions）が現在停止中のため、紛れ込みに気付けず放置されていた
- ローカル CI（`npx tsc --noEmit`）でしか検出できない状態

## 対応方針

### 採用方針

**`url` を `_url` にリネーム**する（`noUnusedParameters` 違反を回避するアンダースコア接頭規約に沿う）。

### 却下案

- **削除する案**: `(_, opts?: RequestInit)` のように位置引数を残す方法もあるが、`fetch` シグネチャと比較したときの可読性で `_url` 表記の方が意図（「使わないが将来使う可能性のある第 1 引数」）が伝わるため不採用
- **ESLint disable コメント案**: 1 行のためだけにディレクティブを増やすのは保守コストが見合わない

## 修正対象ファイル

| ファイル | 変更内容 |
|--------|---------|
| `expense-saas/frontend/src/pages/reports/__tests__/AttachmentArea.test.tsx` | 339 行目（または相当箇所）の `url: string` を `_url: string` にリネーム |

## 完了条件

- `npx tsc --noEmit` で TS6133 が解消する
- 既存テスト（AttachmentArea.test.tsx 17 件以上）が PASS する
- master にマージされ、影響を受けた 3 PR（#141 / #143 / #144）のリベースができる状態になる

## 影響範囲

- `AttachmentArea.test.tsx` のテスト 1 ファイルのみ
- 実装側（`AttachmentArea.tsx` / `AttachmentUploader.tsx`）への影響なし
- API 契約・設計書への影響なし

## 再発防止メモ

- master 直接編集を継続せず、PR フローを通すこと（軽微修正でも CI で型エラーが検出されることを確認したい）
- 本番 CI（GitHub Actions）の復旧を別 issue（ops 系）で計画すること（停止状態が長引くと同種の回帰が継続発生する）
