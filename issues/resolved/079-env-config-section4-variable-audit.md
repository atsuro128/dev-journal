# env_config.md §4.x 全変数の棚卸し

## 発見日
2026-04-13

## カテゴリ
detail-design

## 影響度
中

## 発見経緯
escalation

## 関連ステップ
Step 7（運用設計）、Step 10（機能実装）、Step 11（システムテスト）

## ブロッカー
なし

## 問題

`dev-journal/deliverables/docs/70_operations/env_config.md §4.x` の環境変数一覧と実装・上流設計書の間で連続して誤記が発覚している。運用設計新設時（`0b13c33 refactor: ops-040 Phase 4 成果物修正 — ... 70_operations/ 新設`、2026-03-26）に環境変数を統一フォーマットで網羅列挙した際、以下のような不整合が混入した可能性が高い。

### 既に判明している誤記

| 変数名 | 不整合の種類 | 顕在化した経緯 | 対応 |
|-------|------------|--------------|------|
| `S3_BUCKET_NAME` | 実装は `S3_BUCKET` を参照 | PR #44 (Step 8-11) レビュー時 | PR #44 で修正済み |
| `S3_REGION` | 実装は `AWS_REGION` を参照（AWS SDK v2 標準） | PR #44 レビュー時 | issue 078 で resolved |
| `S3_PRESIGNED_URL_EXPIRY` | 要件 ATT-012 で 15 分固定、環境変数化の意思決定なし | issue 077 対応時に codex 指摘 | issue 077 で §4.4 から削除 |

§4.4（S3 / オブジェクトストレージ）だけで 3 件の誤記が発覚しており、**他の §4.x セクション（DB / JWT / CORS / レート制限 / Argon2id 等）にも類似の誤記が潜在する可能性が高い**。

## 影響

- 運用設計書を参照して環境変数を設定しようとすると、実装と乖離して動作しない変数がある
- 本番デプロイ時に設計書通りに環境変数を設定しても、実装が別の変数名を参照していて起動しないリスク
- Step 11-E デプロイ・スモークテストで顕在化する可能性が高い
- 将来の保守・改修時に、設計書を信頼できない状態が続くと開発速度が低下する

## 提案

### 作業内容: env_config.md §4.x 全変数の棚卸し

`env_config.md §4.x`（§4.1 アプリケーション基本 / §4.2 データベース / §4.3 JWT 認証 / §4.4 S3 / §4.5 CORS / §4.6 レート制限 / §4.7 パスワードハッシュ）で定義されている**全環境変数**について、以下を検証する:

1. **実装参照の有無**: `expense-saas/` 配下を grep して、変数名が実装コードから参照されているか確認
2. **要件の固定性**: 対応する要件 ID がある場合、要件レベルで固定値か可変値か確認
3. **環境ごとの差分の実態**: dev/stg/prod で実際に異なる値が設定されるか、それとも全環境同じ値（= 固定値と扱うべき）か
4. **名称の整合性**: 実装の変数名と設計書の変数名が一致しているか
5. **デフォルト値の整合性**: 実装のデフォルト値と設計書の定義値が一致しているか

### 検証後の対応

各変数について以下のいずれかに分類:

- **A. 整合している**: 何もしない
- **B. 名称乖離**: 実装と設計書のどちらを正とするか判断し、片側を修正
- **C. 誤って環境変数化されている（要件で固定値）**: §4.x から削除（または「環境変数化しない、固定値」と注記）
- **D. 実装で未参照だが本来必要**: 実装を追加（要件が可変性を求めている場合のみ）
- **E. 実装のみで §4.x に記載なし**: §4.x に追記

### 担当

- designer エージェント（§4.x 全変数の調査 + 修正案作成）
- reviewer エージェント（調査結果の妥当性検証）
- codex レビュー（最終検証）

### スコープ外

- `§5 シークレット管理方針`、`§6 環境差分` など §4.x 以外のセクションは対象外（本 issue のスコープを限定するため）
- 必要なら別 issue として切り出す

## 関連

- 本 issue の起票経緯: issue 077（S3_PRESIGNED_URL_EXPIRY 誤起票）の解決過程で `env_config.md §4.4` の信頼性疑義が浮上
- PR #44: Step 8-11 ローカル開発環境統合（`S3_BUCKET_NAME` 誤記を顕在化）
- issue 078: `S3_REGION` vs `AWS_REGION` 乖離（resolved）
- issue 077: `S3_PRESIGNED_URL_EXPIRY` の誤起票と巻き戻し（resolved）

---

## 調査結果（2026-04-13）

Explore エージェントで §4.1〜§4.7 の全 28 変数を網羅的に調査した結果、**16 変数（57%）に不整合**が確認された。issue 079 起票時の懸念（「§4.4 以外にも類似の誤記が潜在する可能性」）は**妥当**であった。

### 全変数の整合性検証結果

| § | 変数名 | 実装参照 | 実装の参照名 | 要件 ID | 判定 |
|---|--------|---------|-----------|---------|------|
| 4.1 | `PORT` | ✅ 有 | — | — | ✅ OK |
| 4.1 | `ENV` | ❌ 無 | — | — | ⚠ 実装未参照 |
| 4.1 | `LOG_LEVEL` | ✅ 有 | — | — | ✅ OK |
| 4.2 | `DATABASE_URL` | ✅ 有 | — | — | ✅ OK |
| 4.2 | `DATABASE_APP_URL` | ⚠ 乖離 | `APP_DATABASE_URL` | — | ⚠ 名称乖離 → Phase 0 |
| 4.2 | `DB_MAX_OPEN_CONNS` | ❌ 無 | — | — | ⚠ 実装未参照 |
| 4.2 | `DB_MAX_IDLE_CONNS` | ❌ 無 | — | — | ⚠ 実装未参照 |
| 4.2 | `DB_CONN_MAX_LIFETIME` | ❌ 無 | — | — | ⚠ 実装未参照 |
| 4.3 | `JWT_PRIVATE_KEY` | ⚠ 部分 | `JWT_PRIVATE_KEY_PATH` | SEC-004 | ⚠ 名称乖離 |
| 4.3 | `JWT_PUBLIC_KEY` | ⚠ 部分 | `JWT_PUBLIC_KEY_PATH` | SEC-004 | ⚠ 名称乖離 |
| 4.3 | `JWT_ACCESS_TOKEN_EXPIRY` | ❌ 無 | ハードコード 15m | SEC-003 | ⚠ 要件固定値 |
| 4.3 | `JWT_REFRESH_TOKEN_EXPIRY` | ❌ 無 | ハードコード 7d | SEC-003 | ⚠ 要件固定値 |
| 4.3 | `JWT_ISSUER` | ❌ 無 | ハードコード | SEC-004 | ⚠ 要件固定値 |
| 4.4 | `S3_BUCKET` | ✅ 有 | — | ATT-001 | ✅ OK |
| 4.4 | `AWS_REGION` | ✅ 有 | — | — | ✅ OK（issue 078 で resolved） |
| 4.4 | `S3_ENDPOINT` | ✅ 有 | — | — | ✅ OK |
| 4.4 | `AWS_ACCESS_KEY_ID` | ✅ 有 | — | — | ✅ OK |
| 4.4 | `AWS_SECRET_ACCESS_KEY` | ✅ 有 | — | — | ✅ OK |
| 4.5 | `CORS_ALLOWED_ORIGINS` | ✅ 有 | — | SEC-013 | ✅ OK |
| 4.6 | `RATE_LIMIT_AUTHENTICATED` | ❌ 無 | ハードコード 100 | — | ⚠ 実装未参照 |
| 4.6 | `RATE_LIMIT_UNAUTHENTICATED` | ❌ 無 | ハードコード 20 | — | ⚠ 実装未参照 |
| 4.6 | `RATE_LIMIT_LOGIN` | ❌ 無 | ハードコード 5 | — | ⚠ 実装未参照 |
| 4.6 | `RATE_LIMIT_UPLOAD` | ❌ 無 | ハードコード 10 | — | ⚠ 実装未参照 |
| 4.7 | `ARGON2_MEMORY` | ❌ 無 | ハードコード 65536 | SEC-002 | ⚠ 要件固定値 |
| 4.7 | `ARGON2_ITERATIONS` | ❌ 無 | ハードコード 3 | SEC-002 | ⚠ 要件固定値 |
| 4.7 | `ARGON2_PARALLELISM` | ❌ 無 | ハードコード 4 | SEC-002 | ⚠ 実装未参照 |

### 追加で判明した残存誤記

調査過程で、**issue 077 対応時の私のミス**が発覚:
- `env_config.md §4.4 L117` から `S3_PRESIGNED_URL_EXPIRY` を削除したが、**同じファイルの §5.3 ECS タスク定義例 L228 と §7.2 dev セットアップ .env サンプル L343** に同変数が残存していた
- 原因: L117 のテーブル行だけを grep で特定して削除し、ファイル全文への全文 grep を怠った
- 同様のミスを防ぐため `feedback_search_before_delete.md` を memory に追加した

---

## 対応方針（Phase 分割）

§4 全 28 変数中 16 変数（57%）に不整合があり、全問題を一気に解決するのは修正規模と設計判断の複雑さから現実的でない。以下の 3 Phase に分割し、**本 issue 内で段階的に対応する**（別 issue には分割しない）。

### Phase 0: 機械的修正（本セッションで対応）

修正規模: 小、設計判断: 不要

対応内容:
1. **D-1 残存誤記削除**: `env_config.md §5.3 L228`（ECS タスク定義例）と `§7.2 L343`（dev .env サンプル）から `S3_PRESIGNED_URL_EXPIRY` を削除。issue 077 の後処理
2. **A-1 名称乖離修正**: `env_config.md` の全箇所（§4.2 L95 テーブル行 + §5.1 L153 シークレット分類 + §5.2 L171 JSON 例 + §5.3 L199/200 ECS 定義 + §6.2 L288 ローテーション例 + §7.2 L326 dev .env サンプル、計 7 箇所）で `DATABASE_APP_URL` → `APP_DATABASE_URL` に統一。実装・`.env.example`・`docker-compose.yml` は既に `APP_DATABASE_URL` を使用しており、設計書側が乖離していた

### Phase 1: JWT 鍵注入方式の統一（本セッション外、別タイミングで対応）

修正規模: 中、設計判断: 必要

対応内容:
- `env_config.md §4.3` の `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY`（PEM 鍵そのものを想定）と実装の `JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH`（ファイルパス）の乖離を解消する
- dev（ローカルファイル）と stg/prod（ECS Secrets Manager から鍵値注入）で異なる注入方式をとっている現状をどう統一するか、セキュリティモデルの再検討を含む

判断が必要な論点:
- 実装を ECS secrets セクション対応に修正（= Secrets Manager から鍵値を直接環境変数として受け取る方式）
- 設計書を `_PATH` サフィックスで統一（= ECS 上でも一時ファイルに書き出す中間ステップを入れる）
- 両方式を併存させる設計にする（現状相当、ただし明示的に文書化）

### Phase 2: 実装未参照 14 変数の整理（本セッション外、別タイミングで対応）

修正規模: 中〜大、設計判断: 必要

対応内容:
- 以下 14 変数について「設計書から削除」または「実装に環境変数参照を追加」を判断する
- 要件固定値（SEC-002/003/004）に該当する変数は issue 077 と同パターンで「誤って環境変数化された」可能性が高い

対象変数:
| 変数名 | ハードコード値 | 要件 ID | 判断候補 |
|-------|-------------|---------|---------|
| `ENV` | — | — | 環境識別ロジックが必要か検討 |
| `DB_MAX_OPEN_CONNS` | 10〜20 | — | 本番チューニング可変性を残すか |
| `DB_MAX_IDLE_CONNS` | 5〜10 | — | 同上 |
| `DB_CONN_MAX_LIFETIME` | 30m | — | 同上 |
| `JWT_ACCESS_TOKEN_EXPIRY` | 15m | SEC-003 | 要件固定 → 削除推奨 |
| `JWT_REFRESH_TOKEN_EXPIRY` | 7d | SEC-003 | 要件固定 → 削除推奨 |
| `JWT_ISSUER` | expense-saas | SEC-004 | システム固定 → 削除推奨 |
| `RATE_LIMIT_AUTHENTICATED` | 100 | — | 運用調整の余地を残すか |
| `RATE_LIMIT_UNAUTHENTICATED` | 20 | — | 同上 |
| `RATE_LIMIT_LOGIN` | 5 | — | 同上 |
| `RATE_LIMIT_UPLOAD` | 10 | — | 同上 |
| `ARGON2_MEMORY` | 65536 | SEC-002 | 要件固定 → 削除推奨 |
| `ARGON2_ITERATIONS` | 3 | SEC-002 | 要件固定 → 削除推奨 |
| `ARGON2_PARALLELISM` | 4 | — | 実装にはあるが要件 ID なし |

---

## 対応状況

| Phase | 内容 | 状態 | 完了日 | 備考 |
|-------|------|------|-------|------|
| Phase 0 | D-1 残存誤記削除 + A-1 名称乖離修正 | ✅ 完了 | 2026-04-13 | commit で対応 |
| Phase 1 | JWT 鍵注入方式の統一 | ✅ 完了 | 2026-04-13 | commit 7381f2e |
| Phase 2 | 実装未参照 14 変数の整理 | ✅ 完了 | 2026-04-13 | commit 7381f2e |

全 Phase 完了。resolved に移動。

---

## 解決内容

### Phase 0 実施内容（2026-04-13）

#### D-1: S3_PRESIGNED_URL_EXPIRY 残存削除
- `env_config.md §5.3 L228` の ECS タスク定義例 JSON から行を削除
- `env_config.md §7.2 L343` の dev .env サンプルから行を削除

#### A-1: DATABASE_APP_URL → APP_DATABASE_URL 統一
- `env_config.md` 全体の `DATABASE_APP_URL` を `APP_DATABASE_URL` に一括置換（7 箇所、§4.2 / §5.1 / §5.2 / §5.3 / §6.2 / §7.2）
- 実装・`.env.example`・`docker-compose.yml` は既に `APP_DATABASE_URL` を使用しているため変更不要

### Phase 1 実施内容（2026-04-13）

#### JWT 鍵注入方式の統一（設計書を実装に寄せる）
- `env_config.md` §4.3 / §5.1 / §5.3 / §7.1: `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` → `JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH` に変更
- `security.md` §2.1 RS256 鍵管理テーブル: 同上、Secrets Manager 連携の説明追加
- `env_config.md` §5.3 ECS タスク定義例: PEM 値を `JWT_*_PEM` として注入 → エントリーポイントでファイル書き出し → `_PATH` で参照する方式に更新

#### Phase 2 実施内容（2026-04-13）

##### 要件固定値の削除（5変数）
- `JWT_ACCESS_TOKEN_EXPIRY` / `JWT_REFRESH_TOKEN_EXPIRY`（SEC-003）、`JWT_ISSUER`（security.md §2.1）、`ARGON2_MEMORY` / `ARGON2_ITERATIONS`（SEC-002）を §4.x テーブルから削除、ハードコードの注記に置換

##### 実装未参照変数の削除（9変数）
- `ENV`、`DB_MAX_OPEN_CONNS` / `DB_MAX_IDLE_CONNS` / `DB_CONN_MAX_LIFETIME`、`RATE_LIMIT_*` x4、`ARGON2_PARALLELISM` を §4.x テーブルから削除、実装の実態に即した注記に置換

##### 連動修正
- §5.3 ECS タスク定義例・§7.1 .env サンプルから削除した 14 変数を除去
- §3.2 アプリケーション設定差分: ログレベルを小文字に修正、レート制限 dev 値を実装と一致させる
- `monitoring.md` §2.3 コード例・§3.1 本文: LOG_LEVEL 設定値を小文字に修正
- `ADR-0005` L60: LOG_LEVEL 設定値を小文字に修正
- §4.2 DB プール注記: オーナープール MaxConns=5 ハードコードを明記
- §5.1 JWT シークレット分類: PEM 値がシークレット対象であると明記

## 解決日
2026-04-13
