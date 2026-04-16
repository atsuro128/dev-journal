# GHA 使用上限時のローカル CI 運用戦略を確立する

## 発見日
2026-04-15

## カテゴリ
ai-ops

## 影響度
高（PR 検証フロー全体に影響、Step 11-B/C 以降の進行をブロックする可能性）

## 発見経緯
user-report / proactive

Step 11-A の Open issue 10 件（097〜106）への並列 dispatch 計画策定中、ユーザーから「今月の残りは GHA 使用上限に達していて使えないかもしれない」との指摘を受け、直近の PR CI 実行状況を確認したところ、使用上限による即時失敗が確定した。

## 関連ステップ
Step 11-A（対応中）/ Step 11-B（横断テスト Go）/ Step 11-C（E2E Playwright）/ 全 Step 横断

## ブロッカー
なし（即起票対応可能）

## 問題

### 現状
1. **GHA 使用上限ヒット**: 直近の PR CI（run `24400750264` 等）で `Lint Backend / Frontend` ジョブがステップ実行前に failure（4-6 秒）、依存ジョブは skipped。月末まで（または当該月いっぱい）GHA は実行不可能と想定される
2. **ブランチ保護の強制力なし**: expense-saas は private repo で GitHub Pro が未契約のため、branch protection 設定が API でブロックされる（`403 Upgrade to GitHub Pro`）。**required status checks による merge ブロックは事実上機能していない**
3. **devcontainer に docker 無し**: `docker` CLI も `/var/run/docker.sock` マウントも無い → devcontainer 内で `docker compose` 実行不可
4. **ローカル CI 手順が未整備**: 指揮役・サブエージェントのどちらが CI を実行するのか、どのコマンドを使うのかが文書化されていない

### 既存 CI ワークフロー（ci.yml）
6 ジョブ:
- `lint-backend` (golangci-lint)
- `lint-frontend` (ESLint + tsc --noEmit)
- `test-backend` (Go test + `integration` タグ + postgres service)
- `test-frontend` (vitest run)
- `build-backend` (go build)
- `build-frontend` (vite build)

`test-backend` は GHA の `services: postgres` を使っており、docker compose には依存していない。`integration` タグ付き go test は testcontainers ではなく DATABASE_URL 経由の直接接続。

### 今後ブロックする可能性のある箇所
- **Step 11-A 以降の全 PR**: CI が赤のままマージされる（branch protection 無いので事実上マージ可能だが、検証が抜ける）
- **Step 11-B 横断テスト（Go）**: postgres 起動手段が devcontainer に無い
- **Step 11-C E2E（Playwright）**: docker compose + ブラウザ起動が必要、devcontainer では完結しない
- **Step 11-E デプロイ**: deploy.yml も同様に実行不可

## 影響

- **短期（当該月中）**: GHA が使えないため、検証なしでマージされると回帰バグが混入する可能性
- **中期（Step 11-B/C）**: 横断テスト・E2E の検証手段が未確立のままだと、Step 11-D 横断レビューまで辿り着かない
- **長期**: GHA 復旧後も「ローカル CI の選択肢がある」ことは価値があるため、恒久運用として整備する余地

## 提案

決定する必要がある事項:

### 1. 指揮役のローカル CI 実行役割
- **案 A（推奨）**: 指揮役が各 worktree で CI コマンドを順次シリアル実行
  - 利点: サブエージェント権限制約を回避、worktree 並列実行時のリソース競合回避
  - 欠点: 指揮役の作業負荷増
- **案 B**: 各サブエージェントが自分の worktree で CI を実行
  - 利点: 並列実行の速度
  - 欠点: サブエージェント権限で docker が使えない、並列時の CPU/メモリ競合

### 2. Frontend CI コマンド（devcontainer 内で実行可能）
```bash
cd <worktree>/frontend
npm ci                  # 初回のみ
npm run lint            # ESLint
npx tsc --noEmit        # TypeScript 型チェック
npm test                # Vitest
npm run build           # ビルド検証（オプション）
```

### 3. Backend CI コマンド（postgres 必要）
- **案 A**: devcontainer に docker.sock マウント追加 → 内部で `docker run postgres` → go test
- **案 B**: WSL2 ホストで postgres コンテナを常駐 → devcontainer からホスト IP 経由で接続
- **案 C**: testcontainers-go 採用を検討して docker 経由に移行
- **案 D**: 恒久的な postgres コンテナを docker-compose.yml に追加

### 4. Step 11-C E2E 実行戦略
- **案 A**: WSL2 ホスト側で `docker compose up` + `npx playwright test` を実行（devcontainer の外）
- **案 B**: dind 対応した devcontainer に拡張
- 推奨: 案 A（最小改修、既に環境リセット自動化スクリプトがホスト側にある）

### 5. PR マージ条件の一時変更
- GHA 使用上限中は「指揮役のローカル CI PASS ログを PR body に添付」をマージ条件とする
- 具体的には、PR 作成時に以下を含める:
  ```
  ## Local CI
  - [x] `npm run lint` PASS
  - [x] `npx tsc --noEmit` PASS
  - [x] `npm test` PASS
  - 実行環境: devcontainer（Claude Code）
  - 実行者: 指揮役
  ```

### 6. `/test` スキル化の検討
- ローカル CI 実行を Claude Code のスキルとして定義し、`/test frontend`、`/test backend`、`/test frontend AttachmentUploader` のようにスコープ指定で呼び出せるようにする
- `Bash run_in_background: true` + timeout 延長で、メイン会話をブロックせず実行可能
- GHA 復旧後も「PR レビュー前のローカル検証」として恒久的に使える
- 検討事項:
  - FE: `vitest run` は devcontainer 内で完結（2分前後）
  - BE: Docker（DB + MinIO）起動が前提。devcontainer 内の docker 可否（issue 061）に依存
  - 出力: PASS/FAIL サマリ + 失敗テスト詳細を報告するフォーマット
  - PR body への「Local CI 実行結果」自動挿入との連携

### 7. 復旧後の運用戻し
- GHA 使用上限リセット（翌月初日）のタイミングで、PR body 手順を廃止するかどうかを判断
- 恒久化するかは本 issue の議論で決定

### 8. devcontainer の docker 可否
- issue 061（devcontainer mount and secret exposure minimization）と整合性を確認
- docker.sock マウントがセキュリティ issue 061 と矛盾する場合、案 A は却下

## 暫定運用（本 issue 決着前）

Step 11-A の Open issue（097〜106）対応では、以下の暫定運用で進める:
- 指揮役が各 worktree で `npm run lint / tsc / test / build` を実行（frontend のみ）
- PR body に「Local CI 実行結果」セクションを追加
- CI が赤でも PR マージを許可（branch protection 強制なし、手動確認で担保）
- 本 issue の正式決着は Step 11-A 完了後の別セッションで実施

---

## 解決内容

### 方針決定

GHA はポートフォリオでの有料利用が不適切なため、ローカルテストをメインとする。GHA の ci.yml は「こういうこともできます」のデモとして残す。

### DinD / DooD は不採用

codex + architect による分析の結果、以下の理由で不採用。

- **DinD**: `--privileged` が必要で `no-new-privileges`・seccomp・apparmor を全て無効化する。firewall の FORWARD DROP とも競合。セキュリティ方針を根本的に破壊する
- **DooD**: docker.sock マウントはホスト Docker への root 相当アクセス。issue 061（mount 最小化）と矛盾し、Squid proxy/allowlist もバイパスされる

### 採用した方式: ホスト Docker 併用 + host gateway outbound

- **Frontend CI**: devcontainer 内で完結（`npm run lint` / `tsc` / `vitest` / `vite build`）
- **Backend CI**: ホスト側で `docker compose up db-test` → devcontainer から host gateway 経由で接続 → `go test -tags integration`
- **E2E**: ホスト側で `docker compose up` + ブラウザ確認
- **フルスタック起動**: ホスト側で `docker compose up` → ホストブラウザで `localhost:5173` にアクセス

### 実装した変更

1. `init-firewall.sh`: `HOST_GATEWAY_OUTBOUND_TCP_PORTS` 環境変数による OUTPUT ルール追加（コンテナ → ホスト方向の指定ポートを許可）
2. `devcontainer.json`: `HOST_GATEWAY_OUTBOUND_TCP_PORTS=5433` を設定（テスト DB 接続用）
3. `docker-compose.yml`: 全 ports を `127.0.0.1` バインドに変更（ローカルネットワークへの意図しない公開を防止）
4. `SECURITY.md`: DinD/DooD 非採用理由と outbound 設計を追記
5. `devcontainer-design.md`: host gateway 設計にセクション 8.2（outbound）を追加

### 残作業

- `/test` スキル化は別タスクとして実施
- PR body の Local CI テンプレートは `/test` スキル整備時に合わせて策定

## 解決日
2026-04-16
