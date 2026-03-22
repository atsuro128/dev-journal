# 引き継ぎメモ

## セッション: 2026-03-22 20:37

### ゴール
- 成果物の図を Mermaid 形式に統一する
- 詳細設計の図の不足を洗い出し、補強する

### 作業ログ
- **ASCII罫線図 → Mermaid変換**（3ファイル・5箇所）
  - domain_model.md: 集約構成図 + レイヤー責務図
  - architecture.md: システム全体構成図 + ミドルウェアチェーン
  - business-flow.md: 月次業務サイクル図
- **ASCIIシーケンス図 → Mermaid変換**（architecture.md・2箇所）
  - ログイン→JWT発行、認証付きリクエストフロー
- **UIライブラリ変更: shadcn/ui → MUI**
  - ADR-0001 に比較検討がなかった点をユーザーに報告
  - ポートフォリオ用途で「楽にきれいに」→ MUI が最適と判断
  - 4ファイル・5箇所を更新（ADR, architecture, security の Tailwind→Emotion）
- **全13画面に処理シーケンス図を追加**（18図）
  - ユーザー指摘: 詳細設計に「ボタン→API→サービス→ドメイン→DB」の縦断フローがない
  - 内部レビュー: blocker 5件 + warning 10件 → 修正 → 再レビュー LGTM
  - codex レビュー: 3件（OpenAPI項目名不一致、INSERT文カラム名、カーソル条件欠落）→ 修正 → 再レビュー LGTM
- **フローチャート追加**（3ファイル・4図）
  - report-detail.md: 提出バリデーション + 状態遷移操作の権限チェック（統合図）
  - files.md: ファイルアップロードバリデーション
  - auth-login.md: 認証フロー分岐（SEC-011 合流表現）
  - 内部レビュー: blocker 2件（自己承認チェック比較対象、却下フロー自己チェック欠落）→ 修正 → LGTM
  - codex レビュー: 2件 → 045 即修正 + 044 issue 037 起票 → 再レビュー LGTM
- **UIデザインガイドライン作成**（ui-guidelines.md）
  - MUI デフォルトテーマベース、ステータスカラーマッピング、コンポーネント使用方針
  - work-breakdown + task-plan に成果物として追記
- **work-breakdown 改善**
  - 完了条件に「処理シーケンス図」「フローチャート」を追加
  - レビュー観点にも同項目を追加
  - 今後 Planner が自動的にスコープに含めるようになる

### 未完了
- issue 037（auth-login フローチャートの API 契約不整合）未対応

### ブロッカー
- なし

### 次にやること
1. issue 037 対応（auth-login フローチャートを API 契約に合わせるか、API 契約を拡張するか）
2. Phase 3（authz.md）に着手
3. Phase 4（最終レビュー）
4. Step 5 完了宣言

### 学び・気づき
- work-breakdown に書かれていない成果物は作られない。Planner も Designer も work-breakdown に忠実に動く。成果物の「完了条件」にシーケンス図・フローチャートを含めなかったことが、今回の補強作業の根本原因
- ADR にUIライブラリの比較検討がなかった。選定理由が1行では判断根拠が不十分。ただしMVPでは変更コストが低いため実害は小さかった
- codex レビューは内部レビューと異なる視点で指摘する（OpenAPI との項目名一致、DBカラム名の大文字小文字など）。両方実施することで網羅性が上がる

### 意思決定ログ
- UIライブラリ: shadcn/ui → MUI に変更。理由: ポートフォリオ用途で「楽にきれいに」、MUI はテーマで一貫性担保、DataGrid 等の業務コンポーネント充実、Claude の実装力も MUI の方が高い
- UIデザイン方針: MUI デフォルトテーマをそのまま使用。カラー変更は theme.ts 1ファイルで後から可能
- 上流シーケンス図は残す: 詳細設計に縦断フローを追加しても、上流（要件定義・アーキテクチャ）の図は別視点のため削除しない
- work-breakdown 改善: 「完了条件」と「レビュー観点」に図の有無を追加することで、次回以降は Planner が自動的にスコープに含める

---

## セッション: 2026-03-22 18:40（前回）

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
- **codex review-procedure.md のステップ別観点を work-breakdown に一本化**
- **廃止済み `project_steps.md` への参照を cleanup**

### 未完了
- Phase 3（authz.md）未着手

### ブロッカー
- なし

### 次にやること
1. Phase 3（authz.md）に着手
2. Phase 4（最終レビュー）
3. Step 5 完了宣言

### 学び・気づき
- codex レビューで LGTM を得る前にコミット提案してしまった。ルールにある手順を飛ばした
- reviewer エージェントに work-breakdown への参照がなかった。観点の定義と参照導線はセットで整備すべき
- codex にも独自のレビュー観点があり二重管理になっていた。観点の正規化先を1箇所（work-breakdown）に決め、他は参照にする

### 意思決定ログ
- 画面詳細仕様の配置: `50_detail_design/screens/` に移動。ステップと配置先を一致させる原則
- ui_flow.md: Step 5 の成果物ではなくレビュー観点。Phase 4 cross-reviewer が整合性を検証
- レビュー観点の管理: work-breakdown ファイルが正規化先（Single Source of Truth）
- 品質チェック vs レビュー観点: 品質チェック=作成者セルフチェック、レビュー観点=reviewer 検証用。別物として維持
