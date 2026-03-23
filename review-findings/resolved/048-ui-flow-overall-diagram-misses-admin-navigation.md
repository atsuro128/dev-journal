# 048: `ui_flow.md` の全体画面遷移図が Admin/Accounting の管理画面遷移を欠落させている

## 指摘概要
`ui_flow.md` の全体画面遷移図には `SCR-ADM-001` と `SCR-ADM-002` のノードは存在するが、ダッシュボードからそれらへ遷移するエッジが描かれていない。一方で同ファイルのロール別遷移図と `screens.md` では、Admin / Accounting に管理メニューが存在することが定義されている。全体図が現行仕様を網羅できておらず、Step 5 Phase 3 で求められる `ui_flow.md` 最終版として内部整合性が崩れている。

## 根拠
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:47`
  > ADM001[SCR-ADM-001<br/>テナント全レポート一覧]
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:48`
  > ADM002[SCR-ADM-002<br/>テナント情報]
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:61`
  > DASH001 -->|マイレポート| RPT001
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:65`
  > DASH001 -->|直近レポート選択| RPT004
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:80`
  > ADM001 -->|レポート選択| RPT004
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:242`
  > DASH001 --> ADM001[テナント全レポート一覧]
- `dev-journal/deliverables/docs/40_basic_design/ui_flow.md:243`
  > DASH001 --> ADM002[テナント情報]
- `dev-journal/deliverables/docs/40_basic_design/screens.md:176`
  > 全レポート | SCR-ADM-001 | - | - | ○ | ○
- `dev-journal/deliverables/docs/40_basic_design/screens.md:177`
  > テナント情報 | SCR-ADM-002 | - | - | - | ○

## 判定
- 重大度: 中
- 分類: 内部整合性 / 画面遷移

## 修正方針案
- `ui_flow.md` の全体画面遷移図に `DASH001 -> ADM001` と `DASH001 -> ADM002` を追加する。
- あわせて、全体図を「全画面の代表遷移を最低1本は含む」方針で見直し、ロール別遷移図との差分が残らないようにする。
