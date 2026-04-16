# 引き継ぎメモ

## セッション: 2026-04-16 11:00〜11:53

### ゴール
- ops-107 方針決定（GHA 使用上限時のローカル CI 運用戦略）
- issue 109 対応（テスト設計書・実装間の乖離）
- issue 104-A 着手（UI RBAC カバレッジマトリクス）

### 作業ログ

#### ops-107 方針決定
1. `/session-start` で状態確認 → Step 11 進行中、全チケット未着手
2. ユーザーから「issue 対応が先」と指摘 → 前回セッションの優先順位（ops-107 → 109 → 104-A）に従う方針に修正
3. ops-107 issue の内容をユーザーに説明（GHA とは何か、何が問題か、何を決める必要があるか）
4. ユーザー方針: 「GHA 有料利用は不適切。ローカルテストをメインに。GHA は ci.yml をデモとして残す」
5. ユーザーから「devcontainer に Docker-in-Docker を入れられないか？ コンテナ内フルスタック起動 → ホストブラウザ確認は可能か？」の質問
6. codex に DinD 導入の分析・計画策定を依頼 + architect にも並行で分析を依頼
7. codex / architect の結論が一致: **DinD も DooD も不採用**
   - DinD: `--privileged` 必要、no-new-privileges / seccomp と矛盾、firewall FORWARD DROP と競合
   - DooD: docker.sock = ホスト Docker への root 相当アクセス、issue 061 と矛盾、Squid バイパス
8. 代替案: `HOST_GATEWAY_OUTBOUND_TCP_PORTS` で devcontainer → ホスト DB 接続を許可する方式
9. codex にセキュリティレビュー依頼 → PASS with NOTE（5433 ポートの用途固定を明記すべき）
10. 実装・ドキュメント更新・コミット完了

### 今セッションで作成したコミット一覧

#### root-project
- `c00652f` feat: host gateway outbound ルールを追加し DinD/DooD を不採用とする

#### expense-saas
- `4e612df` security: docker-compose.yml の全 ports を 127.0.0.1 バインドに変更

#### dev-journal
- `9ab91fb` docs: ops-107 解決内容記入 + devcontainer-design.md に outbound 設計追加

### 未完了

- **Phase 3（実機検証）**: devcontainer rebuild 後に以下を確認する必要がある
  1. `verify-egress.sh` が PASS（既存セキュリティが壊れていないこと）
  2. devcontainer から host gateway の 5433 に接続できること（`psql -h <gateway_ip> -p 5433`）
  3. 許可していないポートがブロックされること
  4. ホスト側で `docker compose up db-test` → `ss -tlnp` で 127.0.0.1 バインド確認
- **ops-107 issue ファイルの移動**: Phase 3 完了後に `open/` → `resolved/` に移動
- **progress.md の更新**: ops-107 解決を反映
- **issue 109 対応**: 未着手（ops-107 に時間を使った）
- **issue 104-A 着手**: 未着手

### ブロッカー

- **devcontainer rebuild が必要**: init-firewall.sh の変更は rebuild しないと反映されない。ユーザーが rebuild 後にセッションに再接続する予定

### 次にやること

#### 優先度 1: Phase 3 実機検証（rebuild 後）
1. verify-egress.sh の PASS 確認
2. `psql -h <gateway_ip> -p 5433 -U testuser -d expense_test` で接続確認
3. 許可外ポートのブロック確認
4. ホスト側で docker compose up → 127.0.0.1 バインド確認
5. 全 PASS なら ops-107 を resolved に移動、progress.md 更新

#### 優先度 2: issue 109 対応（テスト設計書・実装間の乖離）
6. ID 重複 14 件の解消（最優先）
7. 未反映 22 ID の設計書追記
8. FE フィルタ 16 件の実装漏れ確認

#### 優先度 3: issue 104-A 着手
9. designer で screens/*.md からマトリクス抽出 → ui_coverage_matrix.md 作成

### 学び・気づき

- **ユーザーへの説明を省略しない** — ops-107 の内容をユーザーが把握していないのに方針提案に走り、2回指摘された。issue の内容説明から始めるべきだった
- **codex のコンテキストは持ち越されない** — codex は毎回ゼロからの一度きりセッション。「前回の分析でコンテキストを持っている」という理由は成立しない
- **DinD は現行 devcontainer のセキュリティモデルと根本的に両立しない** — no-new-privileges + NET_ADMIN のみ + seccomp/apparmor デフォルトという厳格な設計では、Docker デーモンを内部で動かすことは不可能
- **host gateway outbound は最小限の穴** — DinD/DooD のような根本的なセキュリティ破壊なしに、特定ポートへの経路を1本追加するだけで Backend テスト環境を確保できる

### 意思決定ログ

#### GHA の位置づけ
- ポートフォリオでの有料 GHA 利用は不適切
- ci.yml は「こういうこともできます」のデモとして残す
- ローカルテストをメインとする

#### DinD / DooD 不採用の根拠
- codex + architect の独立分析で一致した結論
- 詳細は `.devcontainer/SECURITY.md` と `ops-107` issue ファイルに記載

#### host gateway outbound 方式
- `HOST_GATEWAY_OUTBOUND_TCP_PORTS` 環境変数で OUTPUT ルールを制御
- デフォルト: 5433（テスト DB のみ）
- 残余リスク: host の 5433 に proxy/tunnel を立てると Squid バイパスの踏み台になりうる → SECURITY.md に明記済み
- docker-compose.yml の全 ports を 127.0.0.1 バインドに変更（ローカルネットワーク公開防止）

## 前回セッション

前回セッション（2026-04-16 09:50〜10:56）の詳細は `dev-journal/archives/session-logs/2026-04-16.md` を参照。
