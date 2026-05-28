# UAT

- 担当: ユーザー
- 依存: 11-E（Step 6-D 完了が前提）
- ブランチ: なし
- 出力先: UAT 確認結果（ブロッカー issue の有無）
- テンプレート: なし

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| UAT チェックリスト | deliverables/docs/60_test/manual_checklists/uat_check.md | 全36項目（受け入れ確認の正本） |
| デプロイ済み環境 | — | 11-E の成果物 |
| ユースケース | deliverables/docs/10_requirements/usecases.md | 受け入れ判定の根拠 |
| 画面仕様 | deliverables/docs/50_detail_design/screens/*.md | 全画面 |

## 責務

- **uat_check.md の全36項目を実施**
- 各ロール（Member / Approver / Accounting / Admin）の主要業務フローを確認
- 業務要件（ユースケース）の妥当性確認
- UX の妥当性確認（操作感・用語・画面の分かりやすさ）
- 各項目の対応ユースケースID／要件ID を確認しながら実施
- 含めない: テストコードの実装、インフラ変更

## 進め方

1. 11-E から公開 URL、UAT 用アカウント、既知の非ブロッカーを受け取る
2. `uat_check.md` の全36項目を、ユーザー視点で順番に実施する
3. 各項目で、期待結果・実際の結果・判定・問題分類を記録する
4. ブロッカーが出た場合は UAT を中断し、修正後に該当項目から再確認する
5. 非ブロッカーは公開可否に影響するかを判断し、公開後対応でよいものは残リスクとして記録する
6. 最後に、リリース可否をユーザーが判定する

## 実施ログ

### 実施日

2026-05-27 〜 2026-05-28

### 実施環境

- 公開 URL: `https://djhmwtrr79jdq.cloudfront.net/` （CloudFront 経由）
- EC2 instance: `i-051ca0c9129854b10`
- CloudFront Distribution ID: `EG1AJBSQL6399`
- アカウント（テナントA 6 件 + テナントB 2 件、パスワード共通 `TestPass1!`）:
  - テナントA: test-admin / test-approver / test-approver2 / test-accounting / test-member / test-member-empty
  - テナントB: test-approver-b / test-member-b
  - UAT-021 で追加作成: uat-test-01@example.com（クリーンアップ対象）

### カテゴリ別結果

| カテゴリ | 項目 | 件数 | 結果 |
|------|------|------|------|
| Member | 申請作成・明細追加・添付・編集・提出・一覧・削除 | UAT-001〜009 (9) | 全 PASS |
| 却下・再申請 | 却下理由確認・再申請レポート作成・再提出 | UAT-010〜012 (3) | 全 PASS |
| Approver | 承認待ち一覧・承認・却下 | UAT-013〜015 (3) | 全 PASS |
| Accounting | 支払待ち一覧・支払完了・経費全体俯瞰 | UAT-016〜018 (3) | 全 PASS（#191 起票） |
| Admin | テナント情報・全レポート俯瞰 | UAT-019〜020 (2) | 全 PASS（#192 起票） |
| 認証系 | サインアップ・ログイン/ログアウト・パスワードリセット | UAT-021〜023 (3) | 2 PASS + 1 既知の未実装スキップ（UAT-023 / issue #151） |
| ロール別フロー一気通貫 | ハッピーパス・却下→再申請・4 ロール網羅 | UAT-030〜032 (3) | 全 PASS（#193 起票） |
| ダッシュボード | Member / Approver+Accounting / Admin の妥当性 | UAT-033〜035 (3) | 全 PASS |
| UX 妥当性 | 用語・フィードバック・入力支援・確認ダイアログ・スマホ・エラー継続性・クロステナント分離 | UAT-040〜046 (7) | 全 PASS（#194 / #195 / #196 起票） |
| **合計** | | **36** | **35 PASS + 1 スキップ、ブロッカー 0** |

### UAT 中に発見した issue（全て post-MVP / 非ブロッカー）

| ID | タイトル | カテゴリ | 発見元 UAT |
|---|---|---|---|
| #191 | 全レポート画面: 対象期間フィルタはあるがテーブル列に対象期間カラムが無い | ux | UAT-018 |
| #192 | draft レポートの Admin / Accounting 閲覧可否の見直し検討 | authz / ux | UAT-020 |
| #193 | 一覧↔詳細往復で 100 req/min 到達 + F5 で生 JSON + 回復遅さ | ux / non-functional | UAT-032 自由探索 |
| #194 | 明細金額フィールドの UX 改善（全角 IME 残置 + エラー文言 + 上限要相談） | ux / form-validation | UAT-042 |
| #195 | スマホ幅でレポート詳細明細一覧テーブルの列幅縦書き化 | ux / responsive | UAT-044 |
| #196 | スマホ幅でマイレポート 0 件時の空状態見切れ | ux / responsive | UAT-044 |

### 切り分けで未起票・取り下げた論点

- **#197（取り下げ）**: 「オフライン時 fetch がタイムアウトせずスピナー永続」を起票したが、追加切り分けで Chrome DevTools の Network throttling: Offline モード限定の挙動と判明。実環境のネット切断（Wi-Fi OFF 等）では `net::ERR_NAME_NOT_RESOLVED` が即座に出てエラーメッセージが表示される（NFR-UX-003 を満たす）。MVP 品質として問題なし、issue は削除済み。

## UAT 終了後のクリーンアップ（必須）

UAT 中に作成・変更したデータを初期状態に戻す。**UAT 全項目完了後に必ず実施すること**（忘れると公開デモ環境にテストテナント・破壊された seed データが残る）。

### 対象データ

| データ変更 | UAT 番号 | 影響 |
|---|---|---|
| 新規レポート作成（テスト経費） | UAT-001〜007 | test-member 名義で 1 件追加 |
| 既存 draft 削除 | UAT-009 | seed の draft が消えた可能性 |
| rejected→再申請（新規 draft）→ submitted | UAT-010〜012 | test-member 名義で 1 件追加 |
| submitted → approved | UAT-014 | 再申請分が approved 化 |
| submitted → rejected | UAT-015 | UAT-001〜007 分が rejected 化 |
| approved → paid | UAT-017 | 1 件 paid 化 |
| 新規テナント作成（uat-test-01） | UAT-021 | UAT Test Company 残存 |

### 実施手順（方法 A: 完全初期化、推奨）

1. **DB の業務テーブルを TRUNCATE**:
   - 対象: `tenants, users, memberships, expense_reports, expense_items, attachments, refresh_tokens` 等（CASCADE 含む）
   - 接続: SSM send-command → docker run --rm --env-file /etc/expense-saas/app.env postgres:16-alpine psql で接続
2. **S3 (expense-saas-portfolio-attachments) のテナントプレフィックスを削除**:
   - `aws s3 rm s3://expense-saas-portfolio-attachments/ --recursive`
3. **seed 再投入**:
   - SSM 経由で `docker run --rm --env-file ... <expense-saas image> /app/seed`
   - 結果: テナントA（5 アカウント / 9 レポート / 2 添付）+ テナントB（2 アカウント / 3 レポート）の初期状態に復元

### 実施タイミング

UAT 全 36 項目完了 → 最終判定（リリース可否）の前または直後。指揮役が SSM send-command で実行可能（ユーザー指示時）。

### 状態

- [x] **完了**（2026-05-28 実施）

### 実施結果

- DB TRUNCATE: `attachments / expense_items / expense_reports / tenant_memberships / users / tenants / refresh_tokens / password_reset_tokens` を CASCADE で TRUNCATE（`categories` は CASCADE で巻き込まれたが seed で再投入される設計）
- S3 削除: `s3://expense-saas-portfolio-receipts-d8ed055a/aaaaaaaa-0001-0001-0001-000000000001/` 配下 4 オブジェクト削除（`_temp/` は image 再配備用に残置）
- seed 再投入: docker run --rm --env-file ... expense-saas:portfolio /app/seed 実行、初期状態に復元
- 検証: tenants=2 / users=8 / reports=12 / items=7 / attachments=2 / categories=6（事前確認時と一致）
- API 疎通: test-admin でテナントA reports 9 件取得確認

## 問題分類

| 分類 | 基準 | 対応 |
|------|------|------|
| ブロッカー | 主要業務フローが完了できない、他テナントデータが見える、権限外操作ができる、ログイン不能 | 修正必須。解消までリリース不可 |
| 非ブロッカー | 表示崩れ、軽微な文言、代替操作が可能な使いにくさ | 公開可否をユーザーが判断 |
| 要相談 | 仕様通りだが業務上違和感がある、仕様が曖昧 | 仕様判断後に対応方針を決める |

## 最終判定

- **判定**: **PASS（MVP 完成、リリース可能）**
- **確認日**: 2026-05-27 〜 2026-05-28
- **確認者**: ユーザー（atsuro128）
- **公開 URL**: `https://djhmwtrr79jdq.cloudfront.net/`
- **ブロッカー**: なし
- **非ブロッカー（post-MVP）**: 6 件起票（#191 / #192 / #193 / #194 / #195 / #196）
- **既知の未実装（スキップ）**: 1 件（UAT-023 パスワードリセット = issue #151 post-MVP 明示済み）
- **公開可否**: **公開可**（MVP として完成判定）

### MVP 受け入れ判定の根拠

1. MVP の全ユースケース（UC-M01〜09, M03a, A01〜03, AC01〜03, AD01〜02, SYS01〜04）がカバーされ実機 PASS
2. 全業務フロー（提出 → 承認 → 支払 / 却下 → 再申請 → 承認 → 支払）が業務として自然に完結
3. RBAC・テナント分離が機能（UAT-046 で実存テナントB データへの不可視性確認）
4. ブロッカー issue 0 件
5. post-MVP 改善点は記録済み（#191〜#196）

## 完了条件

- uat_check.md の全36項目が実施され、結果が記録されている
- MVP の全ユースケース（UC-M01〜09, M03a, A01〜03, AC01〜03, AD01〜02, SYS01〜05）がカバーされている
- ブロッカーとなる問題がないこと
