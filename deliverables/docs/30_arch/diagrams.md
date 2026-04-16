# 構成図・データフロー図

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | architecture.md の内容を図で表現する |
| 正本情報 | システム構成図、リクエスト処理フロー、デプロイ/配信の概要図 |
| 扱わない内容 | 図だけでは伝わらない長文仕様 |
| 主な参照元 | `./architecture.md`, `./adr/*.md` |
| 主な参照先 | `../40_basic_design/*`, `../50_detail_design/*`, `../70_operations/release.md` |

## 1. システム構成図

> 対応: [architecture.md](architecture.md) 2 システム全体構成

```mermaid
graph TB
    subgraph Client["クライアント"]
        Browser["ブラウザ<br/>React + TypeScript + Vite"]
    end

    subgraph AWS["AWS"]
        ALB["ALB<br/>TLS 終端 / ヘルスチェック"]

        subgraph ECS["ECS Fargate"]
            Task1["Go API サーバー<br/>タスク 1"]
            Task2["Go API サーバー<br/>タスク 2"]
        end

        subgraph Storage["ストレージ"]
            RDS["RDS PostgreSQL<br/>+ RLS"]
            S3["S3<br/>領収書ファイル"]
        end

        CWLogs["CloudWatch Logs"]
        CWMetrics["CloudWatch Metrics"]
        SNS["SNS<br/>アラート通知"]
    end

    Browser -->|HTTPS| ALB
    ALB -->|/api/*, /health| Task1
    ALB -->|/api/*, /health| Task2
    ALB -->|/*| Task1
    ALB -->|/*| Task2
    Note over Task1,Task2: Go embed で SPA 静的ファイルも配信
    Task1 --> RDS
    Task2 --> RDS
    Task1 -->|署名付きURL発行| S3
    Task2 -->|署名付きURL発行| S3
    Browser -->|署名付きURL| S3
    Task1 -->|stdout JSON| CWLogs
    Task2 -->|stdout JSON| CWLogs
    RDS --> CWMetrics
    ECS --> CWMetrics
    CWMetrics -->|アラーム| SNS
```

---

## 2. リクエスト処理フロー

> 対応: [architecture.md](architecture.md) 3.2 ミドルウェアチェーン

```mermaid
sequenceDiagram
    participant C as クライアント
    participant ALB
    participant MW as ミドルウェア
    participant H as ハンドラ
    participant S as サービス
    participant D as ドメイン
    participant R as リポジトリ
    participant DB as PostgreSQL

    C->>ALB: HTTPS リクエスト
    ALB->>MW: HTTP リクエスト

    Note over MW: [1] CORS チェック
    Note over MW: [2] セキュリティヘッダー付与
    Note over MW: [3] RequestID 生成
    Note over MW: [4] ログ記録開始
    Note over MW: [5] レート制限チェック
    Note over MW: [6] JWT 検証（RS256）
    Note over MW: [7] Acquire conn + BEGIN
    MW->>DB: SET LOCAL app.current_tenant = 'tenant-uuid'
    Note over MW: [8] RBAC ロール検証

    MW->>H: 認証済みリクエスト（conn を context 経由で伝播）
    H->>S: ユースケース実行
    S->>D: ドメインロジック実行
    D-->>S: 結果 / ドメインエラー
    S->>R: データアクセス
    R->>DB: SQL クエリ（同一 conn、WHERE tenant_id + RLS）
    DB-->>R: 結果セット
    R-->>S: エンティティ
    S-->>H: レスポンスデータ
    H-->>MW: HTTP レスポンス

    MW->>DB: COMMIT（SET LOCAL 自動リセット）
    Note over MW: Release conn + ログ記録完了（duration_ms）

    MW-->>ALB: HTTP レスポンス
    ALB-->>C: HTTPS レスポンス
```

---

## 3. 認証フロー

> 対応: [architecture.md](architecture.md) 3.3 認証フロー

```mermaid
sequenceDiagram
    participant C as クライアント
    participant API as Go API
    participant DB as PostgreSQL

    rect rgb(240, 248, 255)
        Note over C,DB: ログイン（オーナーロール接続 — RLS バイパス）
        C->>API: POST /api/auth/login {email, password}
        API->>DB: SELECT * FROM users WHERE email = ?
        DB-->>API: user (password_hash)
        Note over API: Argon2id 検証
        API->>DB: SELECT role, tenant_id FROM tenant_memberships WHERE user_id = ?
        Note over DB: ※ オーナーロール接続のため RLS 非適用
        DB-->>API: membership
        Note over API: JWT 生成<br/>access_token (15min)<br/>refresh_token (7day)
        API-->>C: {access_token, refresh_token}
    end

    rect rgb(255, 248, 240)
        Note over C,DB: 認証付きリクエスト（業務ロール接続 — RLS 適用）
        C->>API: GET /api/reports<br/>Authorization: Bearer {access_token}
        Note over API: JWT RS256 検証
        Note over API: Acquire conn + BEGIN<br/>SET LOCAL app.current_tenant
        API->>DB: SELECT ... (RLS 適用、同一コネクション)
        DB-->>API: テナント内データのみ
        Note over API: COMMIT（SET LOCAL 自動リセット）
        API-->>C: {data: [...]}
    end

    rect rgb(248, 255, 240)
        Note over C,DB: トークンリフレッシュ
        C->>API: POST /api/auth/refresh {refresh_token}
        Note over API: refresh_token 検証
        Note over API: 新しい access_token 生成
        API-->>C: {access_token}
    end
```

---

## 4. テナント分離フロー

> 対応: [architecture.md](architecture.md) 3.4 テナント分離の実行フロー

```mermaid
graph TD
    subgraph Request["リクエスト処理"]
        A["JWT から tenant_id 取得"] --> B["Acquire conn + BEGIN<br/>SET LOCAL app.current_tenant"]
        B --> C{"データアクセス（同一 conn）"}
    end

    subgraph DoubleGuard["二重保証"]
        C --> D["アプリ層: WHERE tenant_id = ?<br/>（リポジトリ層で強制）"]
        C --> E["DB 層: RLS ポリシー<br/>tenant_id = current_setting(...)"]
        D --> F["結果セット"]
        E --> F
    end

    subgraph ErrorHandling["エラー時"]
        F --> G{"自テナントの<br/>データ?"}
        G -->|Yes| H["正常レスポンス"]
        G -->|No| I["404 Not Found<br/>（存在漏洩防止）"]
    end

    subgraph Cleanup["クリーンアップ"]
        H --> J["COMMIT<br/>（SET LOCAL 自動リセット）"]
        I --> J
        J --> K["Release conn"]
    end
```

---

## 5. 状態遷移図

状態遷移図の正本は [state_machine.md](../20_domain/state_machine.md) を参照。

---

## 6. デプロイパイプライン（計画）

> 対応: [architecture.md](architecture.md) 4.0 SPA 配信方式（ビルド・配信の概要）
>

```mermaid
graph LR
    subgraph LocalCI["ローカルテスト（devcontainer + ホスト側 Docker）"]
        A["PR 作成"] --> B["Lint<br/>golangci-lint<br/>ESLint"]
        B --> C["Test<br/>Go test<br/>Vitest"]
        C --> D["Build<br/>Vite build → dist/<br/>Go build (embed dist/)"]
    end

    subgraph Merge["PR マージ"]
        D --> E["PR body に<br/>テスト結果記載"]
        E --> F["手動確認 →<br/>マージ"]
    end

    subgraph Deploy["手動デプロイ"]
        F --> G["Docker Build<br/>& Push to ECR"]
        G --> H["ECS<br/>ローリングアップデート"]
        H --> I["ヘルスチェック<br/>確認"]
    end
```

---

## 7. ローカル開発環境

> 対応: architecture.md では本図を参照（ローカル環境の詳細は本図が正本）

```mermaid
graph TB
    subgraph DockerCompose["Docker Compose"]
        API["Go API<br/>:8080"]
        FE["Vite Dev Server<br/>:5173"]
        PG["PostgreSQL<br/>:5432"]
        MinIO["MinIO<br/>:9000<br/>（S3 互換）"]
    end

    Developer["開発者"] --> FE
    FE -->|proxy| API
    API --> PG
    API --> MinIO
```
