# 076: 8-2 で要求された基盤依存が go.mod に固定されていない

## 指摘概要
8-2 チケットの責務では `chi, pgx, golang-migrate, golang-jwt, validator` の依存追加が明示されているが、現状の `go.mod` に含まれるのは `chi` と `pgx` だけで、残り 3 つが未追加のままになっている。8-4/8-6 側で都度依存追加から始める必要があり、バックエンド初期化タスクとしての受け渡しが完了していない。

## 根拠
- `dev-journal/progress-management/tickets/step8/8-2-backend-init.md` の責務: 「go.mod 初期化、必要な依存追加（chi, pgx, golang-migrate, golang-jwt, validator）」
- `expense-saas/go.mod:5-17` では `github.com/go-chi/chi/v5` と `github.com/jackc/pgx/v5` しか直接依存に含まれていない。
- work-breakdown `ai-dev-framework/guide/work-breakdown/step8-foundation.md` では 8-2 の出力を「バックエンドコード（基盤）」とし、8-4 はその後続依存として定義している。

## 判定
重大度: 中
分類: チケット完了条件 / 下流準備不足

## 修正方針案
`go.mod` に不足 3 依存を追加し、未使用で直接 import できない段階なら次のいずれかで固定する。
- 実装予定の最小スケルトンを追加して通常 import する。
- `tools.go` などで build tag を使って依存を保持する。
- もし意図的に後続タスクへ移すなら、8-2 チケットの責務記述を修正して委譲先を明示する。

## 対応: チケット責務を修正し委譲先を明示

8-2 チケット（`dev-journal/progress-management/tickets/step8/8-2-backend-init.md`）の責務欄を以下のように修正した:

変更前:
```
- go.mod 初期化、必要な依存追加（chi, pgx, golang-migrate, golang-jwt, validator）
```

変更後:
```
- go.mod 初期化、必要な依存追加（chi, pgx）
  - golang-jwt → 8-4（共通ミドルウェア）で import 時に追加
  - validator → 8-6（スケルトン）で import 時に追加
  - golang-migrate → CLI 利用のため go.mod に不要（Makefile から直接実行）
```

これにより、チケットの責務記述と成果物が一致し、下流タスクへの委譲先が明示された。

## 再レビュー結果
差し戻し（`open/` へ移動）。

### 未解消の論点
- 今回の指摘は「未使用依存を go.mod に残せるか」そのものではなく、`dev-journal/progress-management/tickets/step8/8-2-backend-init.md` の責務に明記した `golang-migrate, golang-jwt, validator` を 8-2 の成果物としてどう扱うかが未整理な点にある。
- 対応不要理由は Go の一般論としては一部妥当だが、責務記述と現成果物の不一致を解消していない。少なくとも、8-2 から下流へ委譲するならチケット本文や受け渡し記録を修正し、委譲先を明示する必要がある。
- `golang-migrate` を CLI 運用する方針だとしても、そうであれば 8-2 チケットの「必要な依存追加」に含め続ける根拠は弱く、責務記述側の修正が必要。

### 追加で確認すべきファイルや観点
- `dev-journal/progress-management/tickets/step8/8-2-backend-init.md` の責務欄を、実際の Go ワークフローと一致する形に修正するか
- 必要なら `ai-dev-framework/guide/work-breakdown/step8-foundation.md` や 8-4 / 8-6 チケット側に、依存追加の担当境界を明示するか
- 8-2 完了時点の受け渡し契約として、下流担当が追加判断なしで着手できる状態になっているか

### このまま進めた場合の下流影響
- 8-2 完了条件の解釈が人によって割れ、再レビューや後続チケットで同じ論点が再燃する
- 8-4 / 8-6 側で「依存追加は前工程の漏れか、現工程の責務か」の判断が必要になり、受け渡しが曖昧なまま進行する
