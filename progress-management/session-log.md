# セッションログ

## セッション: 2026-03-22 18:40

### ゴール
- Issue 036 解消 → Phase 3 着手（時間次第で Phase 4 は見送り）

### 作業ログ
- Issue 036（S3 キー命名規則 `file_id` → `attachment_id`）の修正
  - security-policy.md, requirements.md, business-rules.md の上流修正
  - security.md, files.md の「読み替え」注記を削除（上流修正済みのため不要）
  - codex レビュー: 2件指摘（ATT-014 文言「ファイル名→S3パス」、implementation-guide.md 旧パス残存）→ 修正 → 再レビュー LGTM
  - issue 036 を resolved に移動
- **画面詳細仕様を `40_basic_design/screens/` → `50_detail_design/screens/` に移動**
  - ユーザー指摘: Step 5 で作成した詳細設計成果物が `40_basic_design` にあるのは不整合
  - 13ファイル移動 + 参照パス6箇所更新
- **ui_flow.md を Step 5 の成果物・完了条件から除外**
  - ユーザー指摘: ui_flow.md の「最終版」は成果物ではなく、レビュー観点の話
  - Phase 4 レビュー観点に「ui_flow.md と screens/*.md の整合性」として移動
- **全 Step（0〜7）の work-breakdown にレビュー観点を追加**
  - planner エージェントで過去 review-findings 43件を分析し、具体的チェック項目を逆引き
  - 8 Step 並列で書き込み
  - 品質チェック（作成者セルフチェック）とレビュー観点（reviewer 検証用）の役割分担を明確化
- **reviewer エージェント5つに work-breakdown への参照を追加**
  - ユーザー指摘: reviewer が guide を見る導線がなかった
- **codex review-procedure.md のステップ別観点を work-breakdown に一本化**
  - 二重管理を解消。共通観点は維持、ステップ別は work-breakdown 参照テーブルに置換
- **廃止済み `project_steps.md` への参照を cleanup**
  - AGENTS.md, design-architect.md, impl-architect.md, review-procedure.md, directory-structures, ops-032 を修正
  - 日報・ログ・resolved issue は歴史的記録として変更なし

### 未完了
- Phase 3（authz.md）未着手

### ブロッカー
- なし

### 次にやること
1. Phase 3（authz.md）に着手
   - 作業計画: `task-plans/step5-detail-design.md` の Phase 3 セクション
   - T3-1: authz.md のみ（ui_flow.md は除外済み）
2. Phase 4（最終レビュー）
3. Step 5 完了宣言

### 学び・気づき
- codex レビューで LGTM を得る前にコミット提案してしまった。ユーザーから「再レビューが必要というか、必ず LGTM を得ること。ルールになかったっけ」と指摘。ルールにある手順を飛ばした
- reviewer エージェントに work-breakdown への参照がなかった。レビュー観点を定義しても、reviewer に導線がなければ参照されない。観点の定義と参照導線はセットで整備すべき
- codex にも独自のレビュー観点があり二重管理になっていた。観点の正規化先を1箇所（work-breakdown）に決め、他は参照にする

### 意思決定ログ
- 画面詳細仕様の配置: `50_detail_design/screens/` に移動。ステップと配置先を一致させる原則
- ui_flow.md: Step 5 の成果物ではなくレビュー観点。Phase 4 cross-reviewer が整合性を検証し、指摘が出れば修正する
- レビュー観点の管理: work-breakdown ファイルが正規化先（Single Source of Truth）。reviewer エージェント・codex review-procedure は参照のみ
- 品質チェック vs レビュー観点: 品質チェック=作成者セルフチェック（成果物内に残す）、レビュー観点=reviewer 検証用（work-breakdown に定義）。別物として維持

---

## セッション: 2026-03-22 15:28（前回）

### ゴール
- Step 5 Phase 1 codex レビュー → Phase 2 openapi.yaml → codex レビュー完了まで
- 品質基準の改定
- 画面詳細仕様の1画面1ファイル分割

### 作業ログ
- Phase 1 codex レビュー: 3件の指摘（039 カテゴリ一意制約、040 トークンライフサイクル、041 RLS バイパス）→ 修正 → 再レビュー resolved
- Phase 2 計画立案（Plan エージェント）: 31エンドポイント + 共通スキーマの整理
- Phase 2 成果物作成（openapi.yaml）: detail-designer で生成
- Phase 2 内部レビュー: blocker 4件（B1-B4 フィールド名不一致・上流未追記）+ warning 4件（W3/W4/W6/W7）
  - ユーザー指摘で warning も修正すべきと判断（「実装者が困るなら修正」→「下流が困るなら修正」に拡大）
  - 申請者フィールドを全スキーマで `submitter` に統一
- Phase 2 codex レビュー: 2件（042 RBAC 認可条件未記載、043 tenant/members 権限不整合）→ 修正 → 再レビュー resolved
- **品質ゲート判定基準を改定**: blocker/warning ラベル → 「下流工程が曖昧さや矛盾なく作業できるか」を唯一の基準に
- Phase 1 を新品質基準で内部レビュー再実施: 5件検出（F1 フィールド名不一致、F2 再申請フロー矛盾、F3 API 未記載、F4 除外ルール未記載、F6 Approver 閲覧権限の設計欠陥）
- F6 修正: Approver の閲覧範囲を「自分 + submitted + 自分が承認/却下したレポート」に拡大
- F2 確定: 再申請はパターン B（作成画面経由）に統一
- **画面詳細仕様を1画面1ファイルに分割**: 5ファイル → 13ファイル（dashboard は変更なし）
- 分割後の内部レビュー: 5件（旧ファイル名参照、API 形式不足、誤記）→ 修正 → 再レビュー PASS
- 分割後の codex レビュー: 4件（アクセス制御矛盾、再申請フロー整合、旧ファイル名参照）→ 修正 → 再レビュー LGTM
- タスク計画を13タスクに分割（T1-01〜T1-13 画面 + T1-14〜T1-17 横断設計）
- テンプレート改定: 「1成果物 = 1タスク」粒度原則、品質基準に下流作業可能性を追加
- work-breakdown に成果物粒度原則を追加

### 未完了
- なし

### ブロッカー
- なし

### 次にやること
1. Phase 3（authz.md + ui_flow.md 最終版）に着手
   - 作業計画: `task-plans/step5-detail-design.md` の Phase 3 セクション
2. Phase 4（最終レビュー）
3. Step 5 完了宣言

### 学び・気づき
- warning を「PASS with NOTE」で素通りさせた結果、下流が困る問題（W3/W4/W6/W7）を見逃した。severity ラベルではなく「下流が困るか」で判定すべき。ユーザーの「品質基準ってそこだろうよ」「下流が困るということ」で本質を理解
- Plan エージェントが screens.md のカテゴリ構造をそのままタスク粒度に採用した。上流の「グルーピング」と成果物の「粒度」は別概念。指揮役がこの粒度をユーザーに確認せず進めたのが根本原因
- 1画面1ファイルの原則は AI でも変わらない。AI はファイル数が多くても苦にならず、小さい単位の並列処理が得意。「AI と人間は違う」は粒度を粗くする理由にならない
- タスク計画の成果物列にファイル名を羅列しても、実際に1エージェントで作ったならタスク構造と乖離する。計画は実行単位を反映すべき

### 意思決定ログ
- 品質ゲート判定基準: 「この成果物の下流工程が、曖昧さや矛盾なく作業できるか」が唯一の基準。reviewer のラベルではなく指揮役が最終判定
- 成果物粒度原則: 1画面（1機能）= 1ファイル = 1タスク = 1エージェント。下流がパイプラインを回せる粒度
- 再申請フロー: パターン B（作成画面 SCR-RPT-002 経由）に統一。A4 は遷移のみ、POST は作成画面で実行。state_machine.md のドメイン層コピーは POST 時に発動
- Approver 閲覧権限: 「自分のレポート + submitted + 自分が承認/却下したレポート」。承認直後に 403 になる設計欠陥を解消
- 差し戻しと却下: このシステムでは統一（rejected のみ、終端状態）。再申請は新規レポート作成（reference_report_id で参照）
