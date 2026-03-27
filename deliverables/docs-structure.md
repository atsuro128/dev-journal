# deliverables/docs 構成ガイド

```text
deliverables/
  docs/
    00_goals.md                    # プロジェクトの狙い・評価ポイント・スコープ方針
    01_glossary.md                 # 用語集（申請/承認/差戻し等の定義）
    02_scope.md                    # MVP / 非MVP / やらないことの境界

    10_requirements/
      requirements.md              # 機能要件（コア・付随）・非機能要件
      usecases.md                  # 全ロールのユースケース（招待・通知・監査ログ含む）
      policies.md                  # ルールID体系、業務ルール、RBAC、状態遷移ルール、エラー使い分け方針

    pre_step/                      # 要件定義の中間成果物（業務分析）
      01_business-overview.md      #   業務概要の整理
      02_actor-analysis.md         #   アクター分析
      03_business-flow.md          #   業務フロー分析
      04_business-rules.md         #   業務ルール整理
      project_summary.md           #   プロジェクト概要

    20_domain/
      domain_model.md              # 主要エンティティ・集約・不変条件（audit_logs INSERT ONLY 含む）
      state_machine.md             # 状態遷移の詳細・禁止操作

    30_arch/
      architecture.md              # システム構成・レイヤー構成・責務分担
      diagrams.md                  # 構成図・データフロー図（Mermaid）
      adr/
        0001-tech-stack.md         # Go / React / PostgreSQL 選定理由
        0002-multi-tenant.md       # shared DB + tenant_id 方式の選定理由
        0003-rls-tenant-isolation.md # PostgreSQL RLS による二重保証の設計判断
        0004-infra.md              # AWS ECS Fargate / Terraform 等の選定理由

    40_basic_design/
      screens.md                   # 画面一覧・画面仕様（ロール別の表示差異）
      ui_flow.md                   # 全体画面遷移図（ロール別遷移パス）

    50_detail_design/
      db_schema.md                 # テーブル定義・RLS ポリシー・インデックス方針
      openapi.yaml                 # 全エンドポイント定義（OpenAPI 機械可読）
      authz.md                     # RBAC + テナント分離 + RLS 設定 + 通知・監査ログの権限
      files.md                     # 署名付き URL・MIME バリデーション・発行前認可チェック
      security.md                  # レート制限・CORS・セキュリティヘッダー・govulncheck 方針

    60_test/
      test_strategy.md             # 単体 / 統合 / E2E (Playwright) の層別方針
      traceability.md              # 要件 → テストケースのトレーサビリティマトリクス
      test_cases/                  # テストケース詳細（機能別）
        auth.md                    #   認証・認可
        workflow.md                #   申請ワークフロー
        items.md                   #   経費明細
        attachments.md             #   添付ファイル
        reports.md                 #   経費報告書
        dashboard.md               #   ダッシュボード
        tenant.md                  #   テナント管理
        cross-cutting.md           #   横断的テスト観点

    70_operations/
      runbook.md                   # 監視アラート一次対応・定常確認・エスカレーション手順
      release.md                   # リリース前後チェックリスト・デプロイ手順・ロールバック条件
      backup_restore.md            # バックアップ対象・取得方式・リストア手順・復旧確認
      env_config.md                # 環境差分・環境変数一覧・シークレット管理方針
```
