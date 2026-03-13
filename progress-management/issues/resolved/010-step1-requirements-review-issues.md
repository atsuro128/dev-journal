# Issue: Step1要件定義レビューでの仕様不整合（RBAC中心）

- 発生日: 2026-03-05
- 解決日: 2026-03-05
- カテゴリ: requirements
- ステータス: 解決済み
- 影響度: 高

## 概要
Step1要件定義成果物（8ファイル）レビューの結果、RBAC・操作権限・状態遷移の接続に複数の矛盾が確認された。
この状態では Step2 以降の設計判断（ドメイン責務/API責務/テスト仕様）に分岐が生じる。

## 主な不整合
1. レポート作成可能ロールが文書間で不一致（Memberのみ / Member+Admin / Member+Admin+Approver）
2. Adminの編集可否が不一致（不可 / 可）
3. 「作成可能ロール」と「提出可能ロール（Member限定）」の接続が未定義
4. 提出取消の記述が「判断中」と「決定済み（対象外）」で混在

## 参照
- `review-findings/2026-03-05_step1-requirements-review.md`

---

## 解決内容（2026-03-05）

### 採用方針
`rbac.md` を RBAC 認可仕様の**唯一の正本（Single Source of Truth）**として固定。
他ドキュメントを rbac.md の確定値に同期した。

### 確定した RBAC 仕様（rbac.md 準拠）

| 操作 | Admin | Approver | Member | Accounting |
|------|-------|----------|--------|------------|
| レポート作成 | ○（自分のみ） | ○（自分のみ） | ○（自分のみ） | × |
| レポート編集 | ○（自分のdraftのみ） | ○（自分のdraftのみ） | ○（自分のdraftのみ） | × |
| レポート削除 | ○（自分のdraftのみ） | ○（自分のdraftのみ） | ○（自分のdraftのみ） | × |
| レポート提出 | ○（自分のdraftのみ） | ○（自分のdraftのみ） | ○（自分のdraftのみ） | × |
| 承認 | × | ○（他人のレポートのみ） | × | × |
| 却下 | × | ○（他人のレポートのみ） | × | × |

**ポイント**: ロールに関わらず他者のレポートの編集・提出は不可。自己承認も不可（workflow.md section 8）。

### 修正ファイル一覧

| ファイル | 修正内容 | 対応 Issue |
|---------|---------|-----------|
| `requirements.md` | アクター記述を rbac.md に同期（Approver/Admin の自己申請を明記） | Issue 1, 2 |
| `requirements.md` | RPT-012 の "Admin を除く" を廃止し「ロール問わず他者のレポート編集不可」に統一 | Issue 1 |
| `requirements.md` | ロール定義の Member "申請" を "提出" に修正 | Issue 6 |
| `requirements.md` | SEC-012/013/014 の定義表を 4.2 セキュリティ節に追加 | Issue 5 |
| `workflow.md` | 状態遷移図の `deleted` 疑似状態を廃止、`[*]` 直接遷移に変更（論理削除の明示） | Issue 4 |
| `workflow.md` | 遷移表 T1/T5 の実行者を "Member（レポート所有者）" → "所有者（Member/Approver/Admin）" に修正 | Issue 1, 2 |
| `usecases.md` | UC-M01/06/07 のアクターに Approver/Admin も実行可能な旨を追記 | Issue 2 |
| `usecases.md` | UC-A03 目的の "差し戻す" を削除（glossary 準拠） | Issue 6 |
| `usecases.md` | UC-M01 目的の "精算申請する" を "精算する" に修正（glossary 準拠） | Issue 6 |
| `preliminary/04_business-rules.md` | RPT-010 の作成可能ロールに Approver を追加 | Issue 1 |
| `preliminary/04_business-rules.md` | RPT-012 を requirements.md と同一文言に統一 | Issue 1 |
| `preliminary/04_business-rules.md` | RBAC マトリクスを全面修正（Approver 自己申請権限追加・Admin 提出権限追加・承認欄に「他人のみ」注記追加） | Issue 1, 2 |
| `preliminary/03_business-flow.md` | 提出取消の「判断が必要な点」を「決定事項（2026-03-05）: MVP 対象外」に書き換え | Issue 3 |

### レビューでの指摘に対する補足（AIレビューより）

- **RPT-012 の "Admin を除く" の解釈**: "Admin を除く" は「Admin は所有権制限の例外＝Admin が他人のレポートを編集できる」とも読めるが、rbac.md 2.3 および 3.2 では Admin も自分のレポートのみ編集可としており矛盾していた。rbac.md を正本として「ロール問わず他者のレポート編集不可」に確定。

- **対応案への補足**: 当初の対応案は「rbac.md を正本にして同期」としていたが、**同期後の確定値を明示しないと実行者が再判断を迫られる**との指摘を受け、上記テーブルの形で確定値を本 resolved ファイルに明記した。
