# Step1 要件定義レビュー指摘（2026-03-05）

## 対象
- `deliverables/docs/10_requirements/` 配下の8ファイル
  - `requirements.md`
  - `usecases.md`
  - `workflow.md`
  - `rbac.md`
  - `preliminary/01_business-overview.md`
  - `preliminary/02_actor-analysis.md`
  - `preliminary/03_business-flow.md`
  - `preliminary/04_business-rules.md`

## レビュー観点
- `.claude/commands/requirement.md` の観点
  - アクター
  - ユースケース（正常系/代替系/例外系）
  - 入出力
  - ビジネスルール
  - 状態遷移
  - 非機能（性能/セキュリティ/データ/ユーザビリティ）
  - MVPスコープ境界
- 追加観点（本レビュー）
  - 仕様間の整合性（矛盾・重複）
  - 検証可能性（テストに落とせる粒度か）
  - トレーサビリティ（ルールID参照の完全性）
  - 用語統一（glossary準拠）

## 指摘一覧（重要度順）

### 1. 重大: RBAC仕様が文書間で相互矛盾している
- 根拠:
  - `requirements.md` では経費レポートCRUDのアクターを「Memberが作成/編集/削除/提出、Approver/Admin/Accountingは閲覧」と定義
    - `deliverables/docs/10_requirements/requirements.md:128`
  - 同ファイルのルール `RPT-012` は「所有者のみ可能（Adminを除く）」となっており、Admin例外を許容
    - `deliverables/docs/10_requirements/requirements.md:191`
  - `rbac.md` では「Approver は Member 操作可能」「レポート作成は Admin/Approver/Member が可能」
    - `deliverables/docs/10_requirements/rbac.md:52`
    - `deliverables/docs/10_requirements/rbac.md:73`
  - `preliminary/04_business-rules.md` では「作成は Member/Admin（Approver不可）」かつ「編集はAdmin可」
    - `deliverables/docs/10_requirements/preliminary/04_business-rules.md:35`
    - `deliverables/docs/10_requirements/preliminary/04_business-rules.md:156`
    - `deliverables/docs/10_requirements/preliminary/04_business-rules.md:159`
- 影響:
  - API認可設計、テストケース、UI表示制御が確定できない
- 改善提案:
  - まず `rbac.md` を唯一の正として権限ポリシーを確定
  - その確定値を `requirements.md / usecases.md / preliminary/04_business-rules.md` に同期

### 2. 重大: 役割定義とワークフロー定義が接続不整合
- 根拠:
  - `workflow.md` の提出遷移は「Member（所有者）」限定
    - `deliverables/docs/10_requirements/workflow.md:81`
  - 一方で `rbac.md` は Approver/Admin のレポート作成を許可
    - `deliverables/docs/10_requirements/rbac.md:73`
  - `usecases.md` はレポート作成を Memberユースケースとしてのみ定義
    - `deliverables/docs/10_requirements/usecases.md:22`
- 影響:
  - 「Approver/Adminが作成したレポートを誰が提出できるか」が不明で実装不能
- 改善提案:
  - 「作成可能ロール」と「提出可能ロール」の関係を明文化（例: 自分が作成したレポートはロールに関わらず提出可能、など）
  - ユースケースにも Approver/Admin の自己申請フロー有無を反映

### 3. 中: 提出取消の扱いが「未決/決定済み」で混在している
- 根拠:
  - `preliminary/03_business-flow.md` に「判断が必要な点」として残存
    - `deliverables/docs/10_requirements/preliminary/03_business-flow.md:210`
    - `deliverables/docs/10_requirements/preliminary/03_business-flow.md:229`
  - 同じ文書内で「推奨: MVP対象外」、まとめでも「MVP対象外」と記載
    - `deliverables/docs/10_requirements/preliminary/03_business-flow.md:232`
    - `deliverables/docs/10_requirements/preliminary/03_business-flow.md:367`
  - `preliminary/04_business-rules.md` でもMVP対象外と記載済み
    - `deliverables/docs/10_requirements/preliminary/04_business-rules.md:260`
- 影響:
  - Step1完了後の資料として意思決定状態が曖昧に見える
- 改善提案:
  - 「判断が必要」節を「決定事項」に書き換え、意思決定日・理由を併記

### 4. 中: 状態モデル（5状態）と削除表現が混在し、誤解を誘発する
- 根拠:
  - 状態一覧は5状態（draft/submitted/approved/rejected/paid）
    - `deliverables/docs/10_requirements/workflow.md:10`
  - 状態遷移図では `deleted` を疑似状態として記載
    - `deliverables/docs/10_requirements/workflow.md:29`
    - `deliverables/docs/10_requirements/workflow.md:36`
- 影響:
  - 実装時に `status=deleted` を追加すべきか誤解される可能性
- 改善提案:
  - 削除は「状態遷移」ではなく「論理削除イベント（deleted_at更新）」として図と表現を統一

### 5. 中: SECルールID参照のトレーサビリティ欠落
- 根拠:
  - `requirements.md` のセキュリティ要件で `SEC-012/013/014` を参照
    - `deliverables/docs/10_requirements/requirements.md:365`
    - `deliverables/docs/10_requirements/requirements.md:366`
    - `deliverables/docs/10_requirements/requirements.md:367`
  - 同ファイル内の明示定義は `SEC-001〜005, 010, 011` まで
    - `deliverables/docs/10_requirements/requirements.md:50`
    - `deliverables/docs/10_requirements/requirements.md:56`
- 影響:
  - 参照先が同一文書内で追えず、レビュー/テスト時の追跡性が落ちる
- 改善提案:
  - `requirements.md` 内に `SEC-012/013/014` の定義表を追加するか、参照元を明示リンク化

### 6. 軽微: 用語統一ルールと操作表現に一部揺れがある
- 根拠:
  - 用語集では「提出（submit）」を推奨、「申請/送信」は非推奨
    - `references/glossary.md:40`
  - 用語集では「却下（reject）」を推奨、「差戻し」は非推奨
    - `references/glossary.md:37`
  - `usecases.md` で「精算申請」「差し戻す」を使用
    - `deliverables/docs/10_requirements/usecases.md:27`
    - `deliverables/docs/10_requirements/usecases.md:274`
- 影響:
  - 仕様・UI文言・API説明の用語がブレる
- 改善提案:
  - 操作語は glossary に合わせて「提出」「却下」に統一（名詞としての「申請者」は許容範囲として扱う等、運用ルールを明記）

## 総評
- Step1成果物として、機能・業務・状態遷移・非機能の土台は十分整理されている。
- 一方で、RBACの中核ルールに重大な矛盾が残っており、Step2（ドメイン設計）着手前に解消が必要。
