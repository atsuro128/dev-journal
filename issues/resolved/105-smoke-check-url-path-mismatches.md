# smoke_check.md SMK-007 / SMK-025 の URL パス表記が実装と乖離

## 発見日
2026-04-15

## カテゴリ
documentation / testing / smoke-check

## 影響度
中（SMK 検証時に存在しないパスを叩いてしまい、catch-all なし issue 096 と相まって真っ白な画面になる。検証者が「PR #53 が壊れた」と誤認するリスク）

## 発見経緯
Step 11-A Phase 3 SMK 再検証 SMK-007（Member の /admin/tenant 直接アクセス）を実施したところ真っ白な画面になり、PR #53 のリグレッションを疑った。実装（`App.tsx`）を確認した結果、smoke_check.md が記載しているパスが実装上存在せず、issue 096（catch-all 未実装）と相まってブランク画面に到達していたことが判明。

前セッション（2026-04-14）の session-log にも「issue 088 の再現手順と SMK-025 の乖離が判明」とあり、Group C 対応で実装側は修正したが**smoke_check.md のパス表記自体は修正されないまま放置されていた**。

## 関連ステップ
Step 6-D（手動チェックリスト定義）/ Step 11-A

## ブロッカー
なし（後追い改善事項。ただし Step 11-A の SMK 検証作業に直接影響するため早めに修正したい）

## 事実

### 実装側のパス（`App.tsx` で確認）

```
| ページ | 実装上のパス |
|---|---|
| TenantPage         | /settings/tenant |
| AllReportsPage     | /reports/all     |
| ApprovalListPage   | /approvals       |
| PaymentListPage    | /payments        |
```

### smoke_check.md の現状表記

`dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`:

- L81 SMK-007:
  ```
  1. ブラウザで `/admin/tenant` を直接開く
  ```
  → 実装は `/settings/tenant`

- L104 SMK-025:
  ```
  1. `/workflow/pending` を直接開く
  ```
  → 実装は `/approvals`

### 結果として何が起きるか

存在しないパスにアクセス → React Router が catch-all を持たない（issue 096）→ どのルートにもマッチせず**真っ白な画面**

これにより:
- SMK-007 / SMK-025 が常に FAIL の結果になる
- 検証者は PR #53 のリグレッションを疑うが、実際は SMK 表記の問題で検証自体が成立していないだけ
- 設計書と実装が乖離している印象を与え、品質判断を誤らせる

### 他にも誤記がある可能性

smoke_check.md には他にも URL 表記が存在する可能性があるため、合わせて棚卸しが必要。

## 修正方針

### 1. SMK-007 / SMK-025 のパス修正

```diff
- 1. ブラウザで `/admin/tenant` を直接開く
+ 1. ブラウザで `/settings/tenant` を直接開く

- 1. `/workflow/pending` を直接開く
+ 1. `/approvals` を直接開く
```

### 2. smoke_check.md 全体の URL 表記棚卸し

`grep -n '/'` でパス表記を全て抽出し、`App.tsx` のルート定義と突き合わせて他の誤記がないか確認する。発見されたものは合わせて修正。

### 3. 設計書側の URL 表記との整合確認

`dev-journal/deliverables/docs/50_detail_design/screens/*.md` および `screens.md` で URL パスが定義されていれば、それも実装と一致しているか合わせて確認する。乖離があれば設計書か実装のどちらが正かを判断して修正。

### 4. 再発防止

URL パスの表記揺れを防ぐため、以下のいずれかを検討:
- 案 A: 設計書 `screens.md` に「URL 一覧表」を新設し、smoke_check.md / SMK / 実装はそこを参照
- 案 B: 実装側にルート定数を集約（例: `routes.ts`）し、テストやドキュメントから参照可能にする
- 案 C: 何もしない（手作業注意）

→ 推奨は **案 A**（ドキュメント正本の一元化）。本 issue とは別に検討。

## 修正対象ファイル

- `dev-journal/deliverables/docs/60_test/manual_checklists/smoke_check.md`
- 必要に応じて `dev-journal/deliverables/docs/50_detail_design/screens/*.md`

## 完了条件

- SMK-007 のパスが `/settings/tenant` に修正されている
- SMK-025 のパスが `/approvals` に修正されている
- smoke_check.md 全体の URL 表記が棚卸しされ、他の誤記が修正されている
- 修正後の SMK-007 / SMK-025 を実機で確認し、トースト + リダイレクトの期待挙動が確認できる

## 関連
- 096: catch-all ルート未実装 — 本 issue の症状（真っ白画面）の根本原因のひとつ。catch-all があれば誤記でも 404 ページに到達するため検証者の誤認を防げる
- 088 / PR #53: 認可エラー UX 改善 — 本 issue の修正後にようやく検証可能になる
- 101: SMK-037 文言誤記 — 同じく smoke_check.md の表記不正系
- 104: UI 層 RBAC カバレッジ監査 — 本 issue で発見した「SMK 表記が実装に追従していない」問題の根っこ。re-用紙化の動機の一部
- 前セッション session-log 2026-04-14: 「issue 088 の再現手順と SMK-025 の乖離が判明」の記述あり

---

## 解決内容
smoke_check.md の SMK-007 パスを `/settings/tenant`、SMK-025 パスを `/approvals` に修正済み（Group E 対応時に実施）。catch-all ルートも PR #59 で実装済みのため、URL 誤記による真っ白画面の問題も解消。

## 解決日
2026-04-15
