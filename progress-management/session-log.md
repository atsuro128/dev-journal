# 引き継ぎメモ

## セッション: 2026-04-05 00:04

### ゴール
- issue 036（LSP サーバー統合）の動作検証・クローズ
- issue 049（コメント言語方針）の方針決定・既存コメント日本語化・クローズ

### 作業ログ
- **issue 036（LSP サーバー統合）** — resolved
  - gopls: documentSymbol, hover 正常動作を確認
  - vtsls: `spawn vtsls ENOENT` — `@vtsls/language-server` が未インストールだった
  - `@vtsls/language-server` をグローバルインストール後、documentSymbol 正常動作を確認
  - Dockerfile を修正: `typescript-language-server` を削除し `@vtsls/language-server` に置き換え
  - issue を resolved に移動、コミット完了
- **issue 049（コメント日本語化）** — resolved
  - ユーザーと方針を合意: 全コメント日本語（godoc/JSDoc 含む）、エラーメッセージ・ログは英語維持
  - コーディング規約の置き場について議論: `.claude/rules/implementation-workflow.md` に追加（`expense-saas/**/*` スコープ）
  - 4つのサブエージェントで並列変換:
    1. Go domain/config/cmd/jwt（9ファイル）
    2. Go handler/middleware（20ファイル）
    3. Go service/repo/testutil（30ファイル）
    4. Frontend TS/TSX（21ファイル中、変更は constants.ts のみ）
  - 自己レビューで jwt.go のエラーメッセージ3箇所と main.go の slog.Warn 2箇所が誤って日本語化されていたのを発見・修正
  - go build / go vet エラーなし確認
  - 3リポジトリにコミット完了
- **ops-056（成果物のテンプレート準拠修正）** — ユーザーが対応済み

### 未完了
- ops-055: work-breakdown テンプレートと実ファイルの構造不整合
- ops-047: work-breakdown ディレクトリ構成
- ops-050: 受け渡し契約の冗長性
- 050: JWT 署名アルゴリズム
- 052: JWT 鍵ファイル必須化

### ブロッカー
なし

### 次にやること
1. Step 9（テストコード実装）着手 — チケット起票から
2. ops issue の優先度判断（055, 047, 050 は Step 9 をブロックしない）
3. issue 050, 052（JWT 関連）は Step 9 or Step 10 で対応可

### 学び・気づき
- **サブエージェントはコメント以外も変える**: エラーメッセージ文字列やログメッセージをコメントと混同して日本語化した。コミット前の diff レビューで `// ` 以外の変更行を抽出するチェックが有効だった
- **コーディング規約の置き場は expense-saas スコープのルールファイル**: ai-dev-framework ではなく `.claude/rules/implementation-workflow.md`（paths: expense-saas/**/*）が正しい

### 意思決定ログ
- **コメント言語は全日本語**: ユーザーは日本企業向けポートフォリオを想定。経験のないプログラム言語でコメントまで英語は負担が大きい
- **エラーメッセージ・ログは英語維持**: コメントとコードリテラルは区別する
- **sqlcgen/ は変換対象外**: sqlc 自動生成コードのため
- **typescript-language-server は不要**: vtsls プラグインが TypeScript LSP を担うため、重複する typescript-language-server を Dockerfile から削除

---

## セッション: 2026-04-04 19:45（前回）

### ゴール
- issue 036（LSP サーバー統合）の動作検証・完了
- issue 051（スケーラビリティトレードオフ未記載）の対応

### 作業ログ
- **issue 036（LSP サーバー統合）** — 着手継続
  - LSP 動作検証: `ENABLE_LSP_TOOL=1` は読み込み済み、gopls/vtsls はインストール済みだが LSP ツールが有効にならず
  - 原因: サードパーティマーケットプレイス（Piebald-AI/claude-code-lsps）からのプラグインインストールが必要だった
  - gopls, vtsls プラグインをインストール、Dockerfile にマーケットプレイス設定を追加
  - 動作検証は次回セッション起動後（プラグインはセッション起動時にロードされるため）
- **issue 051（スケーラビリティトレードオフ）** — resolved
  - ADR-0003 に「スケール時の既知の限界」セクションを追加
  - architecture.md は ADR-0003 への参照リンクのみに修正
  - codex レビューで「トレードオフは ADR に書くべき」と指摘され、architecture.md → ADR-0003 に移動
  - 集計クエリの性能リスクも追記（ユーザーとの議論で追加）
- **ops-054（テンプレート・ガイドの責務境界ルール不足）** — resolved
  - 051 対応中に codex が 30_arch 全体の文書責務逸脱を6件発見
  - テンプレート・ガイドのルール不足が再発要因と判明。対応:
    - ADR テンプレート: 5ファイル→1ファイル（adr-template.md）に統合、責務境界ルール追加
    - architecture.md テンプレート: 成果物ベースに再整備、代替案・図の除外ルール追加
    - diagrams.md テンプレート: 「図の正本は本書のみ」追加
    - workflow.md: 成果物テンプレート参照の共通導線を追加
    - step3: 図禁止を「図表現全般」に拡張
    - step4: プロセスセクション削除（workflow.md の責務）
    - step5, step6: テンプレート直接参照を削除
  - 成果物修正:
    - ADR-0003: 設計仕様を要約+forward reference に置き換え
    - architecture.md: 比較表を ADR-0004 に移動、ASCII 図を diagrams.md 参照に差し替え
    - 全5 ADR: ステータス・日付削除、「背景（ペイン）」→「背景」
    - forward reference の節番号修正（§8→§9、security.md §4→§3 等）
    - security.md: ADR-0003 表記を「判断根拠」に修正
  - codex レビュー計7回実施（forward reference の節番号ずれを段階的に修正）
- **ops-055（work-breakdown テンプレート不整合）** — 起票のみ
- **ops-056（成果物のテンプレート準拠修正）** — 起票のみ

### 未完了
- issue 036: LSP プラグインの動作検証（次回セッション起動後）
- ops-055: work-breakdown テンプレートと実ファイルの構造不整合
- ops-056: 既存成果物のテンプレート準拠修正（ADR-0005 の詳細設計、architecture.md の API 詳細等）
- issue 049: コメント言語方針（ユーザー判断待ち）

### ブロッカー
- 036: LSP 動作検証は次回セッション起動後
- 049: ユーザーの目標設定が未確定のため保留

### 次にやること
1. LSP 動作検証（036 クローズ）
2. ops-056 対応（既存成果物のテンプレート準拠修正）
3. ops-055 対応（work-breakdown テンプレートのリファクタ）
4. Step 9（テストコード実装）着手

### 学び・気づき
- **テンプレートは成果物に合わせるべき**: テンプレートが成果物の後に作られた場合、テンプレートを正として成果物を直すのではなく、成果物の実績をベースにテンプレートを整備するのが正しい
- **ADR-template.md と docs-v2 ADR テンプレートの混同**: step3 の work-breakdown が汎用 ADR-template.md（references/decisions/ 用）を参照しており、成果物 ADR テンプレートとの三層問題が発生していた。用途が違うテンプレートの参照先は明確に分離すべき
- **チケット方式を使わない step のテンプレート導線**: step3 はチケット起票手順がないため、work-breakdown 内にテンプレートパスを明示する必要がある。workflow.md の共通導線だけでは不十分
- **codex レビューの芋づる式**: 1つの修正で節番号がずれ、forward reference の修正が連鎖する。最初から全 forward reference を grep で確認すべきだった
- **「背景（ペイン）」のような格好つけた表記は避ける**: シンプルに「背景」で十分

### 意思決定ログ
- **トレードオフは ADR の責務**: architecture.md は「どう構成するか」、ADR は「なぜ・どんなリスクがあるか」。トレードオフは判断理由の一部なので ADR に書く
- **ADR テンプレートは1ファイルに統合**: 5ファイルとも構造が同じなので、共通テンプレート1つで十分。個別の内容はチケットまたは work-breakdown のタスク一覧から導出
- **成果物 ADR からステータス・日付を削除**: git 履歴で追える情報。references/decisions/ 用の ADR-template.md にはステータス・日付を残す（用途が違う）
- **workflow.md に成果物テンプレート参照を追加**: チケット方式・非チケット方式の両方で docs-v2 テンプレートへの導線を担保
- **jq は環境にインストール済み**: Dockerfile L28 で jq をインストール済み。worktree hooks で使用中。以前「入っていない」と言われたのは誤り
