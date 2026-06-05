# 127: ADR-0004 に旧 Free Tier 750時間/月の前提が残っている

## 指摘概要
issue #197 の正本では、対象 AWS アカウントは 2026-05-16 作成の新方式クレジット制 Free Tier であり、旧来の「EC2/RDS 750h/月無料」は無いことを lean 化の発見経緯・コスト前提として確定している。

しかし `ADR-0004` のポートフォリオ対応表には、EC2 t3.micro と RDS db.t3.micro の理由として「無料枠（12ヶ月 750時間/月）」が残っている。直前の lean コスト節では EC2/RDS が定価課金され、stop/start で稼働時間分のみ削減できると説明しているため、同一 ADR 内でコスト前提が矛盾している。

## 根拠
- `dev-journal/issues/open/197-aws-demo-alb-removal-lean-cost-optimization.md`
  - AWS アカウントは新方式クレジット制 Free Tier で、旧来の「750h無料」が無く定価課金
  - グロス稼働コストは RDS 約 $20/月、EC2 約 $10/月
- `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:110-123`
  - lean 構成後のコストとして、EC2/RDS は stop 比率に応じて稼働時間分のみ削減可と記載
- `dev-journal/deliverables/docs/30_arch/adr/0004-infra.md:126-131`
  - 「AWS 無料枠内に収める」「EC2/RDS は無料枠（12ヶ月 750時間/月）」と記載

## 判定
重大度: 中
分類: 上流方針との不整合 / コスト設計の前提矛盾

issue #197 の主要目的は AWS デモの実コストを正しく抑えることなので、設計成果物に旧 Free Tier 前提が残ると、後続の運用判断で「常時稼働でも無料」と誤読される。特に ADR-0004 はインフラ構成とコスト判断の正本であり、EventBridge stop/start の必要性を説明する根拠が弱くなる。

## 修正方針案
`ADR-0004` のポートフォリオ対応表を新方式クレジット制前提に更新する。

例:

- EC2 t3.micro: 「低コスト・単一インスタンスで十分。新方式 Free Tier では旧750h無料なしのため、EventBridge stop/start で稼働時間課金を抑制」
- RDS db.t3.micro: 「MVP 最小構成。旧750h無料は前提にせず、深夜 stop/start と長期不在時 destroy でコストを抑制」

あわせて `ADR-0004` 内の「AWS 無料枠内に収める」は、現状の「クレジットで相殺しつつグロスコストを lean 化する」方針と矛盾しない表現に直す。

## 対応（2026-06-05）

`ADR-0004`（`deliverables/docs/30_arch/adr/0004-infra.md`）のポートフォリオ対応セクションを新方式クレジット制前提に更新した:

- 総論: 「AWS 無料枠内に収める」→「新方式クレジット制 Free Tier（2026-05-16 作成・750h 無料なし・定価課金）でクレジット相殺しつつ、ALB 除去・EventBridge stop/start・長期不在時 destroy で lean 化」
- コンピュート EC2 t3.micro: 「無料枠（12ヶ月 750時間/月）」→「新方式 Free Tier では旧 750h 無料が無いため EventBridge stop/start で稼働時間課金を抑制」
- DB RDS db.t3.micro: 「無料枠（12ヶ月 750時間/月）」→「旧 750h 無料を前提にせず、深夜 stop/start と長期不在時 destroy でコストを抑制」
- ストレージ S3: 「無料枠（5GB）」→「最小構成・クレジット相殺で実費僅少」

これにより lean コスト節（110-122 行）・EventBridge stop/start 運用の根拠と整合し、同一 ADR 内のコスト前提矛盾を解消した。codex 再レビューを依頼する。
