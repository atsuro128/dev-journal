# サブエージェント・オーケストレーション・ワークフロー

## 概要

親エージェント（Claude Code のメインセッション）が**指揮役**として 15 体のサブエージェントを呼び出し、成果物を作り上げるワークフロー。
指揮役はタスクの分配・品質ゲートの判定・ユーザーへの確認を担い、自身では成果物を直接書かない。

### 指揮役の責務

| 責務 | 内容 |
|------|------|
| 計画 | architect エージェントの分析結果を元にタスク実行計画を立てる |
| 分配 | Wave ごとにサブエージェントを並列起動する |
| 品質ゲート | reviewer エージェントの結果を判定し、次 Wave に進むか差し戻すか決める |
| 共有ファイル調整 | 複数エージェントが同一ファイルに書き込む場合、順序を制御する |
| ユーザー連携 | 成果物のコミット確認、blocker の判断確認を行う |
| 進捗管理 | `progress.md` の更新、issue 起票の判断 |

### 指揮役がやらないこと

- 成果物ファイルの直接編集（設計者・実装者エージェントに委譲）
- レビュー（reviewer エージェントに委譲）
- 設計判断（architect エージェントの分析を元にユーザーに確認）

---

## 共通プロトコル

### セッション開始

```
1. progress.md を確認 → 現在の Step を把握
2. task-plans/ のタスク実行計画を確認 → 各タスクの状態・担当セッションを把握
   - 計画ファイルがない場合: Phase 0（architect による計画策定）から開始
   - 計画ファイルがある場合: 未着手タスクの中から次に実行可能なものを特定
3. issues/open/ のブロッカーを確認 → 未解決ブロッカーがあれば先に対応
4. ユーザーに作業計画を提示して合意を得る
```

### サブエージェント呼び出しの基本形

```
Agent(
  subagent_type="<agent-name>",
  prompt="<タスク指示 + 入力ファイル + 出力先 + 完了条件>"
)
```

model と isolation はエージェント定義のフロントマターで設定済み。呼び出し時の指定は不要。

**プロンプトに必ず含める情報:**
- 何を作るか（タスク ID と成果物名）
- 入力ファイルのパス
- 出力ファイルのパスとセクション（共有ファイルの場合）
- 完了条件（step4-5-design.md から引用）
- 先行タスクの成果物で参照すべきポイント

### 並列実行のルール

- 成果物を作成するサブエージェントは `isolation: "worktree"` で起動し、worktree 上で作業・コミット・PR 作成する
- 並列実行時は `Agent` ツールを 1 メッセージ内に複数記述して同時起動
- 同一ファイルへの並列書き込みも worktree 隔離により安全に実行可能（マージ時に競合があれば指揮役が解決）
- 詳細は `ai-dev-framework/rules/branching.md` を参照

### 品質ゲートの判定基準

reviewer エージェントのレポートに基づき判定:

| 重大度 | 判定 | アクション |
|--------|------|-----------|
| blocker なし | **PASS** | 次 Wave に進む |
| blocker あり（設計者で修正可能） | **FIX** | 該当設計者エージェントに修正指示 → 再レビュー |
| blocker あり（上流の問題） | **ESCALATE** | `/issue 起票` → ユーザーに判断を求める |
| warning のみ | **PASS with NOTE** | warning を記録して次 Wave に進む |

### 共有ファイルのマージ順序

複数エージェントが同一ファイルを編集する場合、各エージェントは worktree 上で独立して作業し PR を作成する。指揮役がマージ順序を制御する。

| ファイル | 初回作成（先にマージ） | 追記（後からマージ） |
|---------|---------------------|-------------------|
| `screens.md` | basic-designer（4+5-I: 全体構成） | basic-designer + detail-designer（機能別に PR） |
| `openapi.yaml` | detail-designer（4+5-C: 認証） | detail-designer（機能別に PR） |
| `ui_flow.md` | basic-designer（4+5-I: 初版） | design-architect（4+5-G: 最終統合） |

**原則**: 初回作成の PR を先にマージし、追記の PR を後からマージする。機能別セクションが分かれているため、通常は git の 3-way merge で自動解決される。競合が発生した場合は指揮役が解決する。

### コミットタイミング

```
各エージェント: worktree 上でコミット → push → PR 作成
Wave 完了 → reviewer PASS → ユーザーに PR マージ確認 → マージ
```

各サブエージェントが worktree 上で機能単位にコミット・PR 作成する。マージはユーザー承認後に Wave 単位で行う。

---

## タスク実行計画

### 保存先

```
dev-journal/progress-management/task-plans/
├── step4-5.md    # design-architect が作成
└── step6-7.md    # impl-architect が作成
```

### 目的

1. **セッション間引き継ぎ**: 別セッションの指揮役がファイルを読むだけで現在地を把握できる
2. **並列指揮役の調整**: 複数ターミナルの指揮役が同じファイルを参照し、タスクの重複を防ぐ
3. **進捗の可視化**: ユーザーが直接読んで全体の進行状況を確認できる

### ライフサイクル

```
architect がタスク計画を作成（Phase 0）
  ↓
指揮役がタスク着手時にステータスを「実行中」に更新
  ↓
Wave 完了・レビュー通過後にステータスを「完了」に更新
  ↓
全タスク完了 → progress.md を更新
```

### セッション引き継ぎ

セッションが途中で終了しても、タスク計画ファイルに状態が残る。
新セッションの指揮役は以下の手順で復帰する:

```
1. task-plans/step4-5.md を読む
2. 「実行中」のタスクがあれば、成果物ファイルの存在を確認
   - ファイルが存在し内容がある → 「レビュー中」に進める
   - ファイルが存在しない or 不完全 → 「未着手」に戻してやり直し
3. 次に実行可能なタスク（依存が解消済み + 未着手）を特定
4. ユーザーに現状と次のアクションを提示
```

### 並列指揮役（複数ターミナル）

複数の指揮役が同時に動く場合のルール:

- タスク着手前に計画ファイルを読み、「実行中」でないタスクを選ぶ
- 着手時に計画ファイルのステータスを「実行中」+ ターミナル識別子（T1, T2 等）に更新
- **同一 Wave 内の独立タスク**は異なるターミナルで並列実行可能
- **Wave をまたぐ実行**は禁止（Wave N の全タスク完了前に Wave N+1 に着手しない）
- コミットはターミナルごとに独立して行う（Wave 内の自分の担当分のみ）

---

## Step 4+5: 設計フェーズ・ワークフロー

### Phase 0: 計画

```
指揮役:
  1. progress.md, issues/open/ を確認
  2. task-plans/step4-5.md の有無を確認
     - 存在する → 読んで現在地を把握、Phase 0 スキップ
     - 存在しない → design-architect を起動
  3. design-architect を起動 ──→ タスク実行計画ファイルを作成
  4. 作成された計画をユーザーに提示 → 合意
```

**呼び出し例:**

```
Agent(
  subagent_type="design-architect",
  prompt="
    Step 4+5 のタスク実行計画を作成してください。
    1. step4-5-design.md の Wave 構成・依存グラフを読み込む
    2. 上流成果物（Step 0〜3）の状態を確認する
    3. 各タスクの入力・出力・受け入れ基準を整理する
    4. リスク（上流成果物の不整合等）があれば報告する
    5. 結果を dev-journal/progress-management/task-plans/step4-5.md に書き出す
  "
)
```

### Phase 1: Wave 1 — 基盤設計（並列）

**タスク**: 4+5-A, 4+5-B, 4+5-H, 4+5-I（全て独立ファイルに出力）

```
指揮役: 4 エージェントを同時起動

  ┌─ db-designer ──────→ db_schema.md       [4+5-A]
  ├─ detail-designer ──→ security.md        [4+5-B]
  ├─ detail-designer ──→ monitoring.md      [4+5-H]
  └─ basic-designer ───→ screens.md + ui_flow.md [4+5-I]
```

**呼び出し例（1 メッセージで 4 つ同時起動）:**

```
Agent(subagent_type="db-designer", prompt="
  タスク 4+5-A: DB スキーマ設計（全体）
  入力: domain_model.md, state_machine.md, ADR 0002, ADR 0003
  出力: dev-journal/deliverables/docs/50_detail_design/db_schema.md
  完了条件: 全主要テーブル DDL、RLS ポリシー、インデックス方針
")

Agent(subagent_type="detail-designer", prompt="
  タスク 4+5-B: セキュリティ設計（横断）
  入力: requirements.md（非機能要件）, architecture.md
  出力: dev-journal/deliverables/docs/50_detail_design/security.md
  完了条件: レート制限の具体値、セキュリティヘッダー一覧
")

Agent(subagent_type="detail-designer", prompt="
  タスク 4+5-H: 監視・ログ詳細設計（横断）
  入力: adr/0005-monitoring-logging.md, architecture.md, requirements.md §4.1 §4.3
  出力: dev-journal/deliverables/docs/50_detail_design/monitoring.md
  完了条件: ログフィールド定義、メトリクス一覧、アラート閾値、ヘルスチェック仕様
")

Agent(subagent_type="basic-designer", prompt="
  タスク 4+5-I: 画面一覧・遷移の全体設計
  入力: usecases.md, requirements.md, rbac.md, workflow.md, architecture.md §5.1
  出力: dev-journal/deliverables/docs/40_basic_design/screens.md（画面一覧）
         dev-journal/deliverables/docs/40_basic_design/ui_flow.md（画面遷移図）
  完了条件: 全画面一覧、画面遷移図、共通UIパターン
")
```

**Wave 1 完了後:**

```
指揮役:
  1. 各エージェントの PR を確認（ファイル存在・基本的な完全性）
  2. design-unit-reviewer で基盤成果物の上流整合性をスポットチェック
  3. ユーザーに PR マージ確認 → マージ（Wave 1）
```

### Phase 2: Wave 2 — 機能設計 1（並列）

**タスク**: 4+5-C, 4+5-D

**並列・直列ルール**: 機能間は並列、機能内は直列（basic → detail の順）。
API 設計は画面仕様を参照するため、画面設計の完了を待つ。

```
指揮役: 2 機能を並列起動。各機能内は basic → detail の直列。

  ┌─ [4+5-C 認証] basic-designer → screens.md §認証
  │               ↓（完了後）
  │              detail-designer → openapi.yaml §認証
  │
  └─ [4+5-D 経費] basic-designer → screens.md §経費
                   ↓（完了後）
                  detail-designer → openapi.yaml §経費
```

**共有ファイルの衝突防止**: 各エージェントは worktree 上で作業し PR を作成する。
screens.md / openapi.yaml は機能別セクションが明確に分かれるため、通常は自動マージ可能。

**Wave 2 完了後:**

```
指揮役:
  1. design-unit-reviewer × 2 を並列起動
     - 「認証機能」の設計整合性レビュー
     - 「経費CRUD機能」の設計整合性レビュー
  2. 品質ゲート判定（blocker 有無）
  3. ユーザーに PR マージ確認 → マージ（Wave 2）
```

### Phase 3: Wave 3 — 機能設計 2（並列）

**タスク**: 4+5-E, 4+5-F（4+5-D に依存）

```
指揮役: 2 機能を並列起動。各機能内は basic → detail の直列。

  ┌─ [4+5-E 添付] basic-designer → screens.md §添付
  │               ↓（完了後）
  │              detail-designer → openapi.yaml §添付 + files.md
  │
  └─ [4+5-F 承認] basic-designer → screens.md §承認
                   ↓（完了後）
                  detail-designer → openapi.yaml §承認
```

**Wave 3 完了後:**

```
指揮役:
  1. design-unit-reviewer × 2 を並列起動
     - 「添付ファイル機能」の設計整合性レビュー
     - 「承認フロー機能」の設計整合性レビュー
  2. 品質ゲート判定
  3. ユーザーに PR マージ確認 → マージ（Wave 3）
```

### Phase 4: Wave 4 — 統合 + 横断レビュー

**タスク**: 4+5-G（認可設計 + 最終統合）

Wave 1〜3 で各設計者が作った「部品」を、design-architect が全体視点で組み合わせる工程。

- **authz.md**: 全エンドポイント × 全ロールの認可マトリクス（全機能の API が揃わないと作れない）
- **ui_flow.md 最終版**: 各機能タスクで追加・変更された画面を反映
- **整合性確認**: 画面ID → APIパス → DBテーブル → 認可ルール の一貫性検証

```
指揮役:

  design-architect → authz.md 作成 + ui_flow.md 最終更新 [4+5-G]
    ↓
  design-cross-reviewer → 全機能横断レビュー（パターン 4）
    ↓
  品質ゲート判定
    ↓
  ユーザーに PR マージ確認 → マージ（Wave 4）
    ↓
  /codex-review（Step 成果物完成後の外部レビュー）
```

**横断レビューの呼び出し例:**

```
Agent(
  subagent_type="design-cross-reviewer",
  prompt="
    Step 4+5 の全設計成果物を横断レビューしてください（パターン 4）。
    チェック対象:
    - dev-journal/deliverables/docs/40_basic_design/ 全ファイル
    - dev-journal/deliverables/docs/50_detail_design/ 全ファイル
    上流:
    - dev-journal/deliverables/docs/10_requirements/ 全ファイル
    - dev-journal/deliverables/docs/20_domain/ 全ファイル
    - dev-journal/deliverables/docs/30_arch/ 全ファイル（ADR含む）
    重点: 機能間整合性、用語一貫性、RBAC完全性、上流トレーサビリティ
  "
)
```

### Phase 5: レビュー指摘対応

```
指揮役:
  cross-reviewer / codex-review の指摘を分類:
    blocker → 該当設計者エージェントに修正指示
    warning → ユーザーに判断を求める or 記録して次へ
    info → 記録のみ

  修正完了後:
    design-unit-reviewer で修正箇所を再レビュー
    全指摘解消 → ユーザーにコミット確認 → /commit
    progress.md を更新
```

---

## Step 6+: 実装フェーズ・ワークフロー

### Phase 0: 計画

```
指揮役:
  1. progress.md, issues/open/ を確認
  2. impl-architect を起動 ──→ 実装タスク分解・依存関係・受け入れ基準
  3. 分析結果をユーザーに提示 → 合意
```

### Phase 1: テスト設計

```
指揮役:
  test-designer を起動 → test_strategy.md, test_cases.md
    ↓
  test-reviewer でテスト設計をレビュー（テスト網羅性の事前確認）
    ↓
  ユーザーにコミット確認 → /commit
```

### Phase 2: 基盤構築

```
指揮役:
  platform-builder を起動 → ディレクトリ構造、ミドルウェア、Docker、CI
    ↓
  impl-unit-reviewer で基盤をレビュー（アーキテクチャ準拠、ビルド成功）
    ↓
  ユーザーにコミット確認 → /commit
```

### Phase 3: 機能実装（機能単位で繰り返し）

各機能（認証 → 経費CRUD → 添付 → 承認）について以下を繰り返す:

```
指揮役: 1 機能について以下を実行

  ┌─ backend-developer ──→ API ハンドラ・サービス・ドメイン・リポジトリ
  └─ frontend-developer ─→ 画面コンポーネント・API クライアント
    ↓（両方完了後）
  test-implementer ──→ テストコード
    ↓
  impl-unit-reviewer ──→ フロント/バック横断 + 設計トレーサビリティ
    ↓
  品質ゲート判定
    ↓
  ユーザーにコミット確認 → /commit（機能単位）
```

**バック/フロント並列実行の呼び出し例:**

```
Agent(subagent_type="backend-developer", prompt="
  認証機能の API を実装してください。
  設計: openapi.yaml §認証エンドポイント, db_schema.md, authz.md
  出力先: expense-saas/apps/api/
  基盤（platform-builder 作成済み）のミドルウェアを使用すること。
")

Agent(subagent_type="frontend-developer", prompt="
  認証機能の画面を実装してください。
  設計: screens.md §認証画面, openapi.yaml §認証エンドポイント
  出力先: expense-saas/apps/web/
  基盤（platform-builder 作成済み）の API クライアントを使用すること。
")
```

### Phase 4: 横断レビュー

```
指揮役:
  全機能の実装完了後:

  ┌─ test-reviewer ────────→ テスト網羅性・品質チェック
  └─ impl-cross-reviewer ─→ 全機能横断 + セキュリティ + 設計整合性
    ↓
  品質ゲート判定（blocker は必ず修正）
    ↓
  ユーザーにコミット確認 → /commit
    ↓
  /codex-review（Step 成果物完成後の外部レビュー）
```

### Phase 5: レビュー指摘対応

設計フェーズと同様のフロー。
実装の修正は該当する developer エージェントに委譲。

---

## エラーハンドリング

### サブエージェントが失敗した場合

| 状況 | 対応 |
|------|------|
| エージェントが入力ファイルを見つけられない | 指揮役がパスを確認し、正しいパスで再起動 |
| エージェントがスコープ外の作業をした | 出力を破棄し、制約を明示して再起動 |
| エージェントの出力が不完全 | 不足部分を明示して同じエージェントを再起動 |
| 共有ファイルの競合 | 指揮役が競合を解消し、後続エージェントに修正後ファイルを渡す |

### reviewer が blocker を検出した場合

```
1. blocker の内容を分析
2. 修正が設計者/実装者の範囲内 → 該当エージェントに修正指示
3. 上流成果物の問題 → /issue 起票 → ユーザーに判断を求める
4. 修正完了後、同じ reviewer で再レビュー
5. 3 回修正しても解消しない → ユーザーにエスカレート
```

### 上流成果物との矛盾を発見した場合

```
1. /issue 起票（発見経緯: proactive）
2. 影響範囲を分析（設計者エージェントまたは architect に依頼）
3. ユーザーに判断を求める:
   a. 上流を修正する → 該当 Step に戻る
   b. 下流で吸収する → 差分を記録して続行
```

---

## 状態遷移（Wave レベル）

```
未着手 ──→ 実行中 ──→ レビュー中 ──→ コミット待ち ──→ 完了
              │            │              │
              │         blocker          ユーザー
              │            │            却下
              ▼            ▼              │
           失敗 ──→  修正中 ───→ 再レビュー  │
                                         ▼
                                      調整中
```

各 Wave がこの状態遷移を辿る。指揮役は現在の Wave の状態を把握し、次のアクションを決定する。

---

## ユーザーへの確認ポイント一覧

| タイミング | 確認内容 |
|-----------|---------|
| Phase 0 完了後 | 作業計画の合意 |
| 各 Wave 完了後 | コミットの承認 |
| blocker 検出時 | 修正方針の判断 |
| 上流矛盾発見時 | 対応方針の判断（上流修正 or 下流吸収） |
| 横断レビュー完了後 | 最終コミットの承認 |
| Step 全体完了時 | progress.md 更新の確認 |
