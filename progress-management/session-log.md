# 引き継ぎメモ

## セッション: 2026-04-17 13:00〜16:17

### ゴール
- BE テスト PASS 確認（前回の宿題）
- 104-B 残件: smoke_check.md に SMK-095〜102 追加
- Step 11-A ローカル動作確認着手（時間があれば）

### 作業ログ

#### ゴール 1: BE テスト PASS 確認
1. ユーザーに VS Code タスク「BE: full test」を依頼 → lint 0 issues / unit 全 PASS / integration 全 PASS → 完了

#### ゴール 2: 104-B 残件 → 成果物監査・整合性修正に発展
2. 104-B の smoke_check.md 追加に着手しようとしたところ、ユーザーから「ui_coverage_matrix.md は work-breakdown に定義されていない」と指摘
3. 成果物監査を実施 → 3 件の不整合を発見:
   - Step 3: ADR 0006（JWT 署名アルゴリズム）が work-breakdown 未定義
   - Step 6: ui_coverage_matrix.md が work-breakdown 未定義、テンプレートなし
   - Step 8: directory_structure.md が work-breakdown 未定義
   - Step 6: smoke_check.md / uat_check.md のテンプレートなし
4. work-breakdown 更新 + テンプレート作成（Step 3, 6, 8）
5. codex レビュー → FIX 4 件:
   - 3-8 の入力が下流成果物（security.md）を参照 → requirements.md に修正
   - 6-E のフィードバックループ未定義 → 完了条件に追加
   - 正本テーブルに ADR 0006 未追加 → 追加
   - 8-12 タスク詳細に directory_structure.md 未反映 → 追加
6. ユーザーから「ui_coverage_matrix.md は下流で使われていない中間成果物。work-breakdown に載せるべきでない」→ 6-E 関連を全て取り消し
7. ui_coverage_matrix.md を deliverables/ から削除、private-materials/ に退避
8. issue 110 ファイルの U+FFFD 文字化けを発見 → 根本原因調査
   - Write/Edit ツールのマルチバイト UTF-8 文字化け（Claude Code プラットフォーム由来）
   - PostToolUse hook で Write|Edit 後に U+FFFD を自動検出する仕組みを追加
9. smoke_check.md に SMK-095〜102 を追加（designer → reviewer PASS → codex PASS）
   - §4.10 確認ダイアログ操作（SMK-095〜098）
   - §4.11 画面内ナビゲーションリンク（SMK-099〜100）
   - §4.5 レスポンシブにSMK-101〜102 追加
10. 11-A チケットに実施結果セクションを統合、11-A-smoke-results.md を廃止
11. workflow.md の終了時手順を /session-log に一本化
12. session-log スキルに progress.md 更新・アーカイブ手順を追加

### 今セッションで作成したコミット一覧

#### ai-dev-framework
| コミット | 内容 |
|---------|------|
| `c2661df` | fix: work-breakdown とテンプレートの整合性修正（Step 3/6/8） |
| `a6f20b0` | fix: セッション終了時の progress.md 更新を /session-log に統合 |

#### dev-journal
| コミット | 内容 |
|---------|------|
| `d1f4fbf` | refactor: ui_coverage_matrix.md を成果物から除外 |
| `e427f15` | feat: smoke_check.md に SMK-095〜102 追加、11-A チケット統合 |

#### root-project
| コミット | 内容 |
|---------|------|
| `703b027` | feat: /session-log に progress.md 更新・アーカイブ手順を追加 |

### 未完了

- **Step 11-A ローカル動作確認**: 未着手（本セッションでは到達できず）
- **issue 102 BE/FE 実装**: 設計書修正済み、BE/FE 実装は未着手
- **issue 108**: 方針未決定（動作確認後に判断）

### ブロッカー
なし

### 次にやること

#### 優先度 1: Step 11-A ローカル動作確認
1. docker compose up → smoke_check.md の全 62 項目をブラウザで実施（Phase 2 SMK-012 から再開）
2. 発見した問題を issue 起票

#### 優先度 2: issue 102 BE/FE 実装
3. 添付プレビュー機能の BE → FE 実装

### 学び・気づき

- **中間成果物を成果物ディレクトリに置かない** — ui_coverage_matrix.md は issue 104 の作業中間成果物だったが deliverables/ に置いたことで「正式成果物なのに work-breakdown 未定義」という不整合を生み、テンプレート作成・work-breakdown 更新・codex レビューと大幅な手戻りになった。下流タスクの正式入力にならないファイルは private-materials/ 等に置くべき
- **work-breakdown は手順書であり修正背景は不要** — タスク詳細に「Step 5 で追加判断が必要となり起票された」等の経緯を書いたが、ユーザーから「手順書に背景は不要」と指摘。再現可能な手順だけ書く
- **管理ファイルの分散を避ける** — 11-A-smoke-results.md がチケットとは別の例外的な管理ファイルになっていた。チケットに統合して 1 ファイルに収めた

### 意思決定ログ

#### ui_coverage_matrix.md の位置付け
- work-breakdown の成果物ではなく、issue 104 の中間作業成果物
- deliverables/ から削除し private-materials/ に退避
- 理由: 下流タスクの正式入力として使用されていない

#### U+FFFD 文字化け対策
- Claude Code の Write/Edit ツールにマルチバイト UTF-8 文字化けのバグがある（プラットフォーム由来、プロジェクト側で修正不可）
- PostToolUse hook で Write|Edit 後に自動検出する仕組みを settings.local.json に追加

#### progress.md アーカイブ運用
- 完了 Step チケット一覧 → archives/progress/steps.md
- 解決済み issue テーブル → archives/progress/issues.md
- 運用手順は /session-log スキルに統合（workflow.md から progress.md 更新の個別記述を削除）

#### 11-A 実施結果のチケット統合
- 11-A-smoke-results.md を廃止し、チケット（11-A-local-verification.md）に実施結果セクションを統合
- 理由: 例外的な管理ファイルの排除。チケットが「そのタスクに必要な情報の 1 ファイル集約」場所

## 前回セッション

前回セッション（2026-04-16 21:30〜2026-04-17 00:43）の詳細は `dev-journal/archives/session-logs/2026-04-16.md` を参照。
