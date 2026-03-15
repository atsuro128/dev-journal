# マルチテナント型 経費精算SaaS — 全体ステップ資料

この資料は、進め方（フェーズ）と各フェーズの成果物を整理した"全体の地図"です。
各ステップの詳細は `work-breakdown/` 配下のファイルを参照してください。

---

## 最終的な成果物（ゴール）

- [ ] **GitHub 公開リポジトリ**（履歴が分かるコミットメッセージ）
- [ ] **デプロイ済み URL**（動作確認可能）
- [ ] **README（英語＋日本語）**
  - 概要 / セットアップ / アーキテクチャ / 技術選定理由 / 制約
- [ ] **アーキテクチャ図**（システム構成・データフロー）
- [ ] （推奨）**運用・監視の記載**（デプロイ、ヘルスチェック、ログ、アラート方針）
- [ ] （推奨）**テナント分離・権限のテスト**（README or CIで言及）
- [ ] （推奨）**添付ファイルの権限制御の明記**（署名付きURL発行前の認可チェック）

---

## 全体ステップ一覧

> **設計の核を先に固めること**：Step 0〜3 は実装の土台。特に「状態遷移」「テナント分離ルール」「RBAC」「ADR」は後から変えるとコスト大。

| Step | 名称 | 目的 | 作業分解 |
|------|------|------|----------|
| 0 | 事前準備 | 変更に強い"前提"を定義 | [step0-preparation.md](work-breakdown/step0-preparation.md) |
| 1 | 要件定義 | 業務を理解し作るものを言語化 | [step1-requirements.md](work-breakdown/step1-requirements.md) |
| 2 | ドメイン設計 | データ構造と不変条件を固める | [step2-domain.md](work-breakdown/step2-domain.md) |
| 3 | アーキテクチャ設計 | 技術選定と全体構成を決定 | [step3-architecture.md](work-breakdown/step3-architecture.md) |
| 4+5 | 基本設計 + 詳細設計 | 画面・API・DB・認可・セキュリティを機能単位で設計 | [step4-5-design.md](work-breakdown/step4-5-design.md) |
| 6 | テスト設計 | 重要領域のテスト戦略を策定 | [step6-testing.md](work-breakdown/step6-testing.md) |
| 7 | 実装・運用 | 動くものを出し継続改善可能に | [step7-implementation.md](work-breakdown/step7-implementation.md) |

---

## 各ステップ共通ワークフロー

全ステップ（Step 0〜7）は以下のワークフローに従って進める。
`progress.md` はこのワークフローに基づいてステータスを管理する。

```
1. 成果物作成（Claude）
   └─ 問題発見時: issue 起票（発見経緯: proactive）
   ↓
2. レビュー依頼
   ↓
3. レビュー実施（ユーザー + codex）
   ↓
4. 指摘対応（Claude）
   ↓  ※指摘がなくなるまで 3→4 を繰り返す
5. 完了条件確認
   ↓
6. 完了（progress.md 更新）
```

### 各フェーズの詳細

| # | フェーズ | 担当 | 内容 |
|---|---------|------|------|
| 1 | 成果物作成 | Claude | 各ステップの成果物と【root-project 整備】を作成・コミット。問題発見時は issue 起票（Issue 発掘規約に従う） |
| 2 | レビュー依頼 | Claude | 成果物コミット後、`ai-dev-framework/rules/codex-review.md` に従い codex を自動実行 |
| 3 | レビュー実施 | ユーザー + codex | `ai-dev-framework/agents/review-procedure.md` に従いレビュー。指摘は `dev-journal/review-findings/open/` に起票 |
| 4 | 指摘対応 | Claude | `dev-journal/review-findings/open/` の指摘に対応し、`pending-review/` に移動。再レビューは `agents/re-review-procedure.md` に従う |
| 5 | 完了条件確認 | ユーザー | 各ステップの完了条件を全て満たしていることを確認 |
| 6 | 完了 | Claude | `progress.md` のマイルストーン・タスクを更新 |

### progress.md でのトラッキング

| ステータス | 意味 |
|-----------|------|
| 未着手 | まだ開始していない |
| 進行中（成果物作成） | Claude が成果物を作成中 |
| レビュー待ち | 成果物作成完了。レビュー依頼済み |
| 指摘対応中 | レビュー指摘への対応中 |
| 完了 | 完了条件を満たし、レビュー指摘も全て解消 |

### レビュー関連の参照先

| ドキュメント | パス | 用途 |
|------------|------|------|
| 初回レビュー手順 | `ai-dev-framework/agents/review-procedure.md` | レビュー観点・起票ルール |
| 再レビュー手順 | `ai-dev-framework/agents/re-review-procedure.md` | 指摘対応後の再レビュー |
| 指摘管理 | `ai-dev-framework/review-findings/` | open → pending-review → resolved |
