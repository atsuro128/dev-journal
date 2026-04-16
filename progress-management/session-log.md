# 引き継ぎメモ

## セッション: 2026-04-16 21:30〜2026-04-17 00:43

### ゴール
- devcontainer rebuild 動作確認（golangci-lint v2）
- `/test backend` で BE フルテスト実行
- 104-B 残件: smoke_check.md に SMK-095〜102 追加

### 作業ログ

#### ゴール 1: devcontainer rebuild 確認
1. `golangci-lint version` → v2.11.4 確認 → 完了

#### ゴール 2: BE フルテスト → ops-107 問題発覚・方針変更
2. db-test 起動を依頼 → ポート 5433 が VS Code ポートフォワーディングに占有されていた → 解除して起動
3. devcontainer から db-test への接続テスト → Connection refused（127.0.0.1 バインドのため host gateway 経由で到達不可）
4. codex に状況分析を依頼（初回は関連ファイル不足 → ユーザーから指摘 → ops-107 issue を渡して再依頼）
5. codex 分析結果: ops-107 の実装に 2 つの矛盾
   - docker-compose.yml の全 ports を 127.0.0.1 バインドにしたため host gateway 経由で到達不可
   - Docker Desktop on WSL2 では host gateway IP（172.17.0.1）に published port の応答主体がなく、host.docker.internal（192.168.65.254）を使う必要がある
6. 修正案を検討:
   - A: ホスト側で実行に割り切る
   - B: devcontainer を multi-container 化して db-test を sidecar にする
   - C: init-firewall.sh に host.docker.internal を追加
7. codex セキュリティ評価:
   - docker-compose.yml の 0.0.0.0 バインドは LAN 露出リスク（非推奨）
   - init-firewall.sh の host.docker.internal 追加は条件付き許容
   - devcontainer sidecar はリスクの置き換え（純粋な改善ではない）
8. ユーザーから「107 は結局ホスト操作が必要で無駄」→ A案に決定
9. docker.sock マウント（DooD）も検討 → codex 分析で不採用維持（セキュリティ説明可能性を優先）

#### ops-107 対応実施
10. ops-107 初回実装をリバート（root-project: `095b2d8`、expense-saas: リベース後リバート）
11. docker-compose.yml に test-be サービス追加（golangci-lint:v2.11.4 イメージ、profiles: [test]）
12. .vscode/tasks.json に BE テスト 4 タスク追加（lint / unit / integration / full）
13. .vscode/launch.json にワンクリック実行エントリ追加
14. テスト結果を `dev-journal/logs/test-results/` に tee で出力する設定
15. ホスト側でユニットテスト実行 → DB テーブル不在エラー（マイグレーション未適用）
16. ホスト側で integration テスト実行 → FK 違反・デッドロック（並列実行問題）→ `-p 1` 追加で対策
17. /test スキルを VS Code タスク + 結果ファイル読み取り方式に更新
18. devcontainer-design.md のセクション 8.2 を outbound から BE テスト実行方式に書き換え
19. ops-107 issue ファイルに初回方式の廃止理由と最終方式を記録

### 今セッションで作成したコミット一覧

#### root-project
| コミット | 内容 |
|---------|------|
| `095b2d8` | Revert: host gateway outbound ルール（init-firewall.sh / SECURITY.md / devcontainer.json） |
| `2dadcb6` | feat: BE テスト用 VS Code タスク追加（tasks.json） |
| `49c6b04` | fix: BE テスト結果を dev-journal/logs/test-results/ に出力 |
| `e3bda3a` | fix: /test スキルを VS Code タスク + 結果ファイル方式に変更 |

#### expense-saas
| コミット | 内容 |
|---------|------|
| リバート | Revert: docker-compose.yml の 127.0.0.1 バインド |
| `9ec5405` | feat: BE テスト用 docker compose サービス追加（test-be） |

#### dev-journal
| コミット | 内容 |
|---------|------|
| `1005aab` | fix: ops-107 方式変更に伴う設計文書・issue 更新 |

### 未完了

- **BE integration テスト PASS 確認**: マイグレーション適用後に `-p 1` で実行したが結果未確認（FK 違反が残る可能性）
- **104-B 残件**: smoke_check.md に SMK-095〜102 の 8 項目を追加する作業が未実施
- **issue 102 BE/FE 実装**: 設計書修正済み、BE/FE 実装は未着手
- **issue 108**: 方針未決定（動作確認後に判断）

### ブロッカー
なし

### 次にやること

#### 優先度 1: BE テスト PASS 確認
1. ホスト側で VS Code タスク「BE: full test」を実行
2. 結果ファイル（dev-journal/logs/test-results/）を確認
3. 失敗があれば修正

#### 優先度 2: 104-B 残件（smoke_check.md 追加）
4. SMK-095〜102 の 8 項目を smoke_check.md に追加
5. 内部レビュー → codex レビュー → コミット

#### 優先度 3: Step 11-A ローカル動作確認
6. docker compose up → smoke_check.md の全項目をブラウザで実施
7. 発見した問題を issue 起票

#### 優先度 4: issue 102 BE/FE 実装
8. stacked PR 方式で BE → FE の順に実装

### 学び・気づき

- **codex に聞くときは元の issue を渡す** — 状況説明だけ書いて関連ファイルを渡さないと、codex が設計意図を理解できず表面的な分析になる。ops-107 の issue ファイル + devcontainer 設計資料を渡して初めて「設計と実装の矛盾」を正確に指摘できた
- **変更の経緯を辿ってから対応する** — DB 接続問題の原因を「127.0.0.1 バインド」だと即断したが、実際には ops-107 で host gateway outbound 方式を採用した際の設計自体が Docker Desktop on WSL2 では機能しなかった。git log で変更の経緯を確認してから対応すべきだった
- **ops-107 の初回実装は動作確認なしで resolved にされていた** — devcontainer → db-test の接続を 1 回も試さずに完了扱いになっていた。接続確認していれば即座に気づけた問題

### 意思決定ログ

#### ops-107 方式変更: host gateway outbound → ホスト側 docker compose run
- **廃止理由**: Docker Desktop on WSL2 では host gateway IP に published port の応答主体がなく、host.docker.internal を使う必要がある。さらに 127.0.0.1 バインドとの矛盾、ホスト操作が結局必要な点で利便性のメリットが薄い
- **最終方式**: docker-compose.yml に test-be サービス（golangci-lint:v2.11.4）を追加、VS Code タスクでワンクリック実行、結果は tee でファイル出力して devcontainer から読み取り

#### docker.sock（DooD）不採用維持
- docker.sock は Docker daemon への管理権限委譲に等しく、sibling container 起動で egress 制御を完全に回避可能
- ポートフォリオとしては「便利だから開ける」より「脅威モデルに合わないから避ける」の方が説明力が高い
- 運用カバー（CLI 未インストール、hooks での制限）は「突破可能という本質」を変えない

## 前回セッション

前回セッション（2026-04-16 16:30〜21:20）の詳細は `dev-journal/archives/session-logs/2026-04-16.md` を参照。
