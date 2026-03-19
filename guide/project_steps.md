# マルチテナント型 経費精算SaaS — 全体ステップ資料

この資料は、進め方（フェーズ）と各フェーズの成果物を整理した"全体の地図"です。
各ステップの詳細は `work-breakdown/` 配下のファイルを参照してください。

---

## 全体ステップ一覧

> **設計の核を先に固めること**：Step 0〜3 は実装の土台。特に「状態遷移」「テナント分離ルール」「RBAC」「ADR」は後から変えるとコスト大。

| Step | 名称 | 目的 | 作業分解 |
|------|------|------|----------|
| 0 | 事前準備 | 変更に強い"前提"を定義 | [step0-preparation.md](work-breakdown/step0-preparation.md) |
| 1 | 要件定義 | 業務を理解し作るものを言語化 | [step1-requirements.md](work-breakdown/step1-requirements.md) |
| 2 | ドメイン設計 | データ構造と不変条件を固める | [step2-domain.md](work-breakdown/step2-domain.md) |
| 3 | アーキテクチャ設計 | 技術選定と全体構成を決定 | [step3-architecture.md](work-breakdown/step3-architecture.md) |
| 4 | 基本設計 | 画面一覧・画面遷移・共通UIパターンを確定 | [step4-basic-design.md](work-breakdown/step4-basic-design.md) |
| 5 | 詳細設計 | 機能別画面詳細・API・DB・認可・セキュリティを設計 | [step5-detail-design.md](work-breakdown/step5-detail-design.md) |
| 6 | テスト設計 | 重要領域のテスト戦略を策定 | [step6-testing.md](work-breakdown/step6-testing.md) |
| 7 | 実装・運用 | 動くものを出し継続改善可能に | [step7-implementation.md](work-breakdown/step7-implementation.md) |

---

## 各ステップ共通ワークフロー

レビューフローの詳細は `.claude/rules/workflow.md` を参照。

### progress.md でのトラッキング

| ステータス | 意味 |
|-----------|------|
| 未着手 | まだ開始していない |
| 進行中（成果物作成） | Claude が成果物を作成中 |
| レビュー待ち | 成果物作成完了。レビュー依頼済み |
| 指摘対応中 | レビュー指摘への対応中 |
| 完了 | 完了条件を満たし、レビュー指摘も全て解消 |

