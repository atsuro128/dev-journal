# 差戻し機能が存在しない（却下のみ）

## 発見日
2026-03-19

## 発見経緯
proactive（市場調査による発見）

## 関連ステップ
Step 1（usecases.md, workflow.md, requirements.md）、Step 2（domain_model.md, state_machine.md）、Step 4（screens.md, ui_flow.md）

## カテゴリ
requirements / workflow

## 問題
現行設計では「却下（rejected）」のみで「差戻し（returned）」が存在しない。却下後の再申請は新規レポートとして作成する仕様になっている。

市場調査の結果、調査した全8製品（MFクラウド経費、楽楽精算、ジョブカン、HRMOS経費、freee、Expensify、SAP Concur、Brex）が差戻しと却下を明確に区別している。

- **差戻し**: 軽微な不備（入力ミス、添付漏れ、カテゴリ間違い）→ 同一レポートを修正して再提出
- **却下**: 経費として認められない内容 → 申請終了（再申請は新規レポート）

## 影響
- ユーザビリティに深刻な影響。カテゴリ間違いの修正だけでもレポート全体を再作成（明細再入力、添付再アップロード）が必要
- 修正履歴の追跡が困難（元レポートとの関連が参照のみ）
- 業界標準から大きく乖離しており、製品としての実用性に疑問

## 提案
1. 状態遷移に `submitted → returned` を追加
2. returned 状態では draft と同様に編集可能とし、再提出で `returned → submitted` に遷移
3. 差戻し理由コメントは必須
4. rejected は「経費として不適切」な場合に限定し、再申請は新規レポート（現行仕様を維持）

### 影響する上流成果物
- `10_requirements/usecases.md` — 差戻し UC の追加
- `10_requirements/workflow.md` — 状態遷移の追加
- `10_requirements/requirements.md` — WFL 機能追加
- `20_domain/domain_model.md` — ReportStatus に returned 追加
- `20_domain/state_machine.md` — 遷移ルール追加
- `40_basic_design/screens.md` — 差戻しボタン・差戻し理由表示の追加
- `40_basic_design/ui_flow.md` — 差戻しフローの追加

## 解決内容

対応不要。草案（PROJECT_SUMMARY.md）の時点で差戻しは意図的にスコープ外とされており、5状態モデル（draft/submitted/approved/rejected/paid）で設計されている。本プロジェクトはポートフォリオ用途であり、却下→新規再申請のフローで状態遷移の設計力は十分に示せる。市場調査に基づく業界標準との比較はプロダクション基準の指摘であり、ポートフォリオのスコープには該当しない。

## 解決日
2026-03-20
