# 大規模クロステナント環境での性能テスト計画（post-MVP）

## 発見日
2026-04-25

## カテゴリ
testing / performance / post-mvp

## 影響度
中（実運用想定規模での性能未検証は SaaS としてリスク。インデックス・RLS の効きが大規模で破綻する可能性）

## 発見経緯
Step 11-A SMK-080 検討中、ユーザー指摘「複数テナント × 数百万件以上の現実的データ規模で性能未検証」から既存テスト範囲のギャップが認識された

## 関連ステップ
Step 6（テスト設計）/ Step 11-A（ローカル動作確認）/ Step 11-B（横断テスト Go）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外として扱う。MVP リリース前には対応しない。管理方式は **ops-080** で検討中。

## 問題

### 既存性能テストのカバレッジと現実シナリオのギャップ

| テスト | データ量 | 検証内容 | 実運用との乖離 |
|------|---------|---------|--------------|
| CRS-083 (`GET /api/dashboard`) | 限定的（テストフィクスチャ） | p95 < 500ms（20 回計測） | 集計系クエリが大規模で劣化する可能性未検証 |
| CRS-084 (`GET /api/reports`) | **50 件**（テナント A） | p95 < 500ms | 数百万件超 + 複数テナント混在時の挙動未検証 |
| CRS-085 (`GET /api/reports/all`) | **50 件**（テナント A、Admin 視点） | p95 < 500ms | テナント全体合算後の性能未検証 |
| CRS-086 (`GET /api/reports/{id}`) | 単発取得 | p95 < 500ms | データ量に依存しない |
| CRS-087 (`GET /api/workflow/pending`) | 限定的 | p95 < 500ms | 大規模 submitted 件数時の挙動未検証 |
| CRS-088 (`POST .../attachments` 5MB) | 単発 | アップロード 5 秒以内 | データ量と無関係 |
| SMK-080 | seed 8 件 | 体感 1 秒未満 | 軽量スモーク代替、規模問題は範囲外 |

### 実運用想定リスク

複数テナントを抱える SaaS として、レポート情報テーブルにユーザー全体合算で **数百万〜数千万件** が実用的に存在し得る。各テナントは自分のデータしか見えないが、内部クエリは全行を走査する可能性があるため、`tenant_id` インデックスと RLS の効率が破綻すると単一テナントの利用感が悪化する。

具体的なリスク:

1. **インデックス選択ミス**: PostgreSQL Optimizer が `tenant_id` を含む複合インデックスを正しく使わない場合（統計情報の偏り、プランナの誤判断等）、シーケンススキャンに陥り p95 が劣化
2. **RLS のオーバーヘッド**: 各クエリで `set_config('app.current_tenant_id')` + RLS ポリシ評価が走る。大量同時接続 + 大規模テーブルで競合 / メモリ圧迫が発生する可能性（`test_strategy.md L142` で言及済み）
3. **集計クエリの劣化**: ダッシュボードの月別合計や件数集計（`my_draft_count` 等）が大規模データで劣化
4. **JOIN 系の劣化**: `expense_reports` × `expense_items` × `attachments` の JOIN が大規模でハッシュ JOIN/ネステッドループ JOIN の選択を誤ると劣化

## 既存設計の対策

- **ADR-0002**（マルチテナント方式）: Shared DB + `tenant_id` 方式採用（テナント別 DB 分離は最終手段）
- **ADR-0003**（RLS テナント分離）: PostgreSQL RLS ポリシーで tenant_id 起点フィルタ
- **db_schema.md §8 / 関連箇所**: `tenant_id` を複合インデックスの先頭に配置する設計
- **冗長保持**: `expense_items` / `attachments` は RLS 効率のため `tenant_id` を冗長保持
- **コネクションプール**: スケール時の対応として PgBouncer / RDS Proxy の選択肢を ADR-0003 / 0004 で言及

→ **設計上の対策はあるが、実機での実証試験未実施**

## 提案する検証スコープ（post-MVP 対応時）

### §1 大規模 seed 生成

- 複数テナント（10〜100 程度）× テナント当たり数千〜数十万件の経費レポート + 明細
- 合計目安: `expense_reports` 500 万行、`expense_items` 5000 万行、`attachments` 1 億行
- seed 生成ツール: `cmd/seed-bulk` 等の追加 or `pgbench`/`pg_dump` ベース
- 一意性制約 / FK 制約に違反しないバルクインサート方式（`COPY` 文 + 一時テーブル経由）

### §2 主要クエリの EXPLAIN / 実測

検証対象クエリ（CRS で対象としている範囲を大規模で再実行）:

- `GET /api/reports?status=...&page=...&per_page=...`（CRS-084 大規模版）
- `GET /api/reports/all`（CRS-085 大規模版、Admin）
- `GET /api/dashboard`（CRS-083 大規模版、集計系）
- `GET /api/workflow/pending`（CRS-087 大規模版）

判定:

- `EXPLAIN ANALYZE` で意図したインデックスが選択されているか
- p95 / p99 が NFR-PERF-001 の 500ms 以下を維持できるか
- 劣化があれば PostgreSQL の `auto_explain` で実行時統計を分析

### §3 RLS / set_config の高負荷挙動

- 100 同時接続で繰り返し `set_config('app.current_tenant_id', ...)` を呼ぶシナリオ
- メモリ消費 / セッション接続数 / コネクションプール枯渇の閾値計測
- `pgbouncer` 経由 vs 直接接続の比較

### §4 改善判断材料

実証結果に基づき以下を判断:

- 追加インデックス必要性（カバリングインデックス、部分インデックス等）
- マテリアライズドビュー / 集計テーブル導入要否（特にダッシュボード集計）
- リードレプリカ導入（読み取りスケールアウト）
- PgBouncer / RDS Proxy 必須化
- 最終手段: テナント別 DB 分離（ADR-0002 で「最終手段」と記載済みの選択肢）

## 関連

- **#081**（open, post-MVP）: MVP スコープ外のテスト観点一括管理。本 issue の追加観点として参照価値あり
- **#084**（open, post-MVP）: HttpOnly Cookie 移行。post-MVP 管理方式の参考
- **ADR-0002**（multi-tenant.md）: shared DB + tenant_id 方式の根拠
- **ADR-0003**（rls-tenant-isolation.md）: RLS 設計、スケール時の既知の限界（§71-81）
- **ADR-0004**（infra.md）: インフラ構成（EC2 t3.micro 1 台 + RDS db.t3.micro）
- **NFR-PERF-001/002/003**: パフォーマンス要件
- **CRS-083〜088**: 既存 BE 性能テスト（50 件ベース）
- **`test_strategy.md L142`**: 100 ユーザー負荷テスト MVP スコープ外の判断、RLS の set_config が高負荷時に競合する可能性に言及
- **ops-080**（open）: MVP スコープ外事項の管理方式決定

## 完了条件（post-MVP 対応時）

- 大規模 seed 生成ツールが整備
- §2 主要クエリで EXPLAIN ANALYZE による実測結果が記録される
- p95 / p99 計測値が NFR-PERF-001 の閾値内であることを確認、または閾値超過時の改善案が決定される
- §3 RLS / set_config の高負荷挙動の計測結果が記録される
- §4 改善判断（追加インデックス / マテリアライズドビュー / リードレプリカ / PgBouncer 等）の結論が記録される
- 検証手順 / 結果 / 改善判断のドキュメント化（test_cases/cross-cutting.md または別文書）
