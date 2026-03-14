---
step: 2
severity: medium
status: open
---

# 025: domain_model.md のドメインエラー一覧が state_machine.md を網羅していない

## 指摘概要

`state_machine.md` で定義している違反ケース・競合ケースの一部が、`domain_model.md` のドメインエラー一覧に含まれていない。Step 2 の完了条件である「ドメインエラーが全ての不変条件違反をカバーしているか」を満たしていない。

## 根拠

- `state_machine.md` の T3（却下）では、却下理由が空の場合のエラーとして `MissingRejectionReason` を使っている
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:126`
- しかし `domain_model.md` のドメインエラー一覧には `MissingRejectionReason` が存在しない
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:427`
- `state_machine.md` の競合制御方針では、楽観的ロック失敗時に `ConflictError` を返すとしている
  - `dev-journal/deliverables/docs/20_domain/state_machine.md:333`
- しかし `domain_model.md` のドメインエラー一覧には `ConflictError` も存在しない
  - `dev-journal/deliverables/docs/20_domain/domain_model.md:427`

## 影響

- `WFL-012` 違反時の標準エラー名と HTTP 変換先が未確定のまま残る
- 楽観的ロック競合の扱いがエラー一覧に現れず、詳細設計や API エラー設計に引き継げない
- 文書間でエラー体系が一致しないため、実装とテストの観点がぶれる

## 修正方針案

- `domain_model.md` のドメインエラー一覧に `MissingRejectionReason` と `ConflictError` を追加し、HTTP 変換方針を明記する
- もし `ConflictError` をドメインエラーではなくアプリケーション/リポジトリエラーとして扱うなら、`state_machine.md` 側でその層責務を明示して一覧との整合を取る
