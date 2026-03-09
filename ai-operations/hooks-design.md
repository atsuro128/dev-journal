# Hooks 設計資料

## 概要

Claude Code の hooks 機能を利用して、プロジェクトルールの遵守を支援する仕組み。
ブロック（exit 2）は最小限にとどめ、基本は警告（exit 0）+ ログ記録で運用する。

---

## Hooks ライフサイクル（公式仕様）

### イベント一覧

| イベント | 発火タイミング | matcher対象 |
|---|---|---|
| `SessionStart` | セッション開始・再開時 | 開始方法（startup, resume, clear, compact） |
| `SessionEnd` | セッション終了時 | 終了理由 |
| `UserPromptSubmit` | ユーザーがプロンプト送信した直後 | なし |
| `InstructionsLoaded` | CLAUDE.md 等が読み込まれた時 | なし |
| `PreToolUse` | ツール実行前（ブロック可能） | ツール名 |
| `PostToolUse` | ツール実行成功後 | ツール名 |
| `PostToolUseFailure` | ツール実行失敗後 | ツール名 |
| `PermissionRequest` | パーミッション確認ダイアログ表示時 | ツール名 |
| `Stop` | Claude が応答を完了した時 | なし |
| `SubagentStart` / `SubagentStop` | サブエージェント起動・終了時 | エージェント型 |
| `Notification` | 通知送信時 | 通知型 |
| `PreCompact` | コンテキスト圧縮前 | トリガー（manual, auto） |

### Exit code

| Code | 意味 | 動作 |
|---|---|---|
| 0 | 成功 | stdout の JSON を処理。stderr は詳細ログに表示 |
| 2 | ブロッキングエラー | stderr が Claude へフィードバックされる |
| その他 | 非ブロッキングエラー | stderr は詳細ログに表示、実行は継続 |

### Exit 2 のイベント別効果

| イベント | 効果 |
|---|---|
| `PreToolUse` | ツール呼び出しをブロック |
| `Stop` | Claude の停止を防止（継続させる） |
| `UserPromptSubmit` | プロンプト処理をブロック＆削除 |
| `PostToolUse` | stderr を Claude に表示（ツールは実行済み） |

### 入力データ（stdin JSON）

全イベント共通:
```json
{
  "session_id": "string",
  "transcript_path": "string",
  "cwd": "string",
  "hook_event_name": "string"
}
```

PreToolUse / PostToolUse 追加フィールド:
```json
{
  "tool_name": "string",
  "tool_input": { ... },
  "tool_use_id": "string"
}
```

### 出力（stdout / stderr）

- **stdout**: exit 0 時のみ処理される。JSON で構造化出力が可能
- **stderr**: exit 2 時は Claude へのフィードバック。それ以外は詳細ログ（Ctrl+O）に表示

### Matcher

正規表現でツール名等をフィルタリングする。

```json
"matcher": ""            // 空文字列 = 全イベント対象
"matcher": "Bash"        // 完全一致
"matcher": "Edit|Write"  // OR マッチ
"matcher": "mcp__.*"     // 正規表現パターン
```

`Stop`, `UserPromptSubmit` 等は matcher をサポートしない（常に発火）。

---

## 本プロジェクトの Hook 構成

設定ファイル: `.claude/settings.json` の `hooks` セクション
スクリプト格納先: `.claude/hooks/`

### 1. edit-scope-check（PreToolUse）

| 項目 | 内容 |
|---|---|
| matcher | `Edit\|Write` |
| 目的 | ソースコードが expense-saas/ 外で編集されることを検知 |
| 動作 | 警告表示 + ログ記録（exit 0） |
| 対象拡張子 | .rs, .ts, .tsx, .js, .jsx, .css, .scss, .html, .sql |
| ログ出力先 | `dev-journal/logs/hooks/hook-warnings.log` |

**設計意図**: CLAUDE.md で既にスコープ制限を指示しているため、hookはセーフティネット。
ブロックではなく警告にすることで、無限ループリスクを回避する。

### 2. commit-session-log-check（PreToolUse）

| 項目 | 内容 |
|---|---|
| matcher | `Bash` |
| 目的 | git commit 前にセッションログの存在を確認 |
| 動作 | ログ未記録時はブロック（exit 2） |
| ログ記録 | なし |

**設計意図**: コミット前のセッションログ記録は必須ルール。exit 2 でブロックしても、
「ログを作成する → 再コミット」で条件が解消される収束型のフローであり、無限ループにならない。

注意: matcher が `Bash` のため全 Bash コマンドで発火するが、スクリプト内で
`git commit` を含まないコマンドは早期 return している。

### 3. stop-check（Stop）

| 項目 | 内容 |
|---|---|
| matcher | なし（Stop イベントは matcher 非対応） |
| 目的 | セッション終了時に未コミット変更の存在を通知 |
| 動作 | 警告表示 + ログ記録（exit 0） |
| 確認対象 | root-project, expense-saas, ai-dev-framework, dev-journal |
| ログ出力先 | `dev-journal/logs/hooks/hook-warnings.log` |

**設計意図**: 未コミット変更の見落としを防ぐ。exit 2 にすると「作業指示 → 新たな変更 →
再度ブロック」の発散型無限ループが発生するため、警告のみとする。

---

## 設計原則

### 警告（exit 0）を基本とする

- exit 2（ブロック）は、1回の対応で条件が解消される**収束型**フローの場合のみ使用する
- ブロックのフィードバックメッセージが新たな作業を生む可能性がある場合は、exit 0 にする
- 警告のログは `dev-journal/logs/hooks/hook-warnings.log` に記録し、後から確認可能にする

### 無限ループの回避

Stop hook で exit 2 を使うと以下の循環が発生しうる:

```
Claude 停止試行 → exit 2 ブロック
  → 「未コミット変更があります、コミットしてください」
  → Claude がコミット作業を実行
  → 新たな変更が発生
  → Claude 停止試行 → exit 2 ブロック → ...
```

この問題は exit 0（警告のみ）にすることで解決した。

### Bash matcher の制約

Bash hook の matcher はコマンド内容ではなくツール名（`Bash`）でマッチする。
特定のコマンドだけに反応させたい場合は、スクリプト内で `tool_input.command` を検査して
早期 return する必要がある。
