# Step 11-A SMK-104/105 検証用フィクスチャ不足（テナント B Approver / テナント A 第二 Approver）

## 発見日
2026-05-02

## カテゴリ
testing / infrastructure

## 影響度
中（機能影響なし。Step 11-A の SMK-104（テナント分離）/ SMK-105（同テナント別 Approver 処理分の非表示）の検証が前提データ不足で実施不能）

## 発見経緯
user-report / Step 11-A SMK-104 検証時、smoke_check.md の前提条件「テナント B にも別途処理済みレポート（テナント B 側の Approver が処理）」を満たすために `internal/seed/seed.go` を確認したところ、**テナント B には Member のみ存在し Approver が定義されていない**ことが判明。SMK-105 の前提条件「同テナント内の別 Approver が承認したレポートあり」も同様に**テナント A の Approver が 1 人のみのため再現不可能**であることが副次発見された。

SMK-103〜105 は issue #158（Approver 処理済み一覧 SCR-WFL-003 新規）対応時に smoke_check.md §4.12 として新規追加されたが、SMK 設計時にフィクスチャと突き合わせて前提条件の整合を確認していなかった。

## 関連ステップ
Step 6（テスト設計）/ Step 8（基盤構築・seed）/ Step 11-A（ローカル動作確認）

## ブロッカー
なし

## 問題

### 現状フィクスチャ（`internal/seed/seed.go`）

| アイテム | 状態 |
|---|---|
| テナント B Approver アカウント | **未定義** |
| テナント B のレポート | Draft / Submitted / Approved の 3 件（Member B 作成） |
| `ReportTenantBApprovedID` の `approved_by` | INSERT 文で未指定（NULL の可能性） |
| テナント A Approver | UserApproverID 1 人のみ |
| テナント A 第二 Approver | **未定義** |

### SMK-104（テナント分離）の前提条件不足

> 前提データ: テナント A の Approver、テナント B にも別途処理済みレポート（テナント B 側の Approver が処理）

→ テナント B Approver が存在しないため、テナント B 側で「Approver が処理した」状態のレポートを DB 上に置けない。SMK-104 の検証用前提が満たせない。

### SMK-105（同テナント内別 Approver 処理分の非表示）の前提条件不足

> 前提データ: 同テナント内の別 Approver が承認したレポートあり、自分は未処理

→ テナント A に第二 Approver が存在しないため、「別 Approver が処理したレポート」を作成できない。SMK-105 の検証用前提が満たせない。

## 影響

- Step 11-A の SMK-104 / SMK-105 を実施できないため、Step 11-A クローズ条件を満たせない
- テナント分離 / RBAC の自動テスト（CRS-001〜054）でカバーされる責務だが、UI レベルでの「処理済み一覧画面のテナント越境/別 Approver 越境フィルタ」確認が SMK で抜ける

## 提案

### 修正内容（`internal/seed/seed.go`）

#### 1. テナント B Approver の追加

```go
// 定数追加
UserApproverBID = "bbbbbbbb-2222-2222-2222-000000000022"

// users テーブル INSERT
{UserApproverBID, "test-approver-b@example.com", "Test Approver B"}

// tenant_memberships INSERT
{TenantBID, UserApproverBID, domain.RoleApprover}
```

#### 2. テナント B の処理済みレポートの approved_by 設定

既存の `ReportTenantBApprovedID` の INSERT で `approved_by` / `approved_at` カラムを `UserApproverBID` / `now` に設定。schema を確認のうえ、必要なら別途 UPDATE 文を加える。SMK-104 では「処理日 DESC で表示される 1 件」が見えるよう、approver_b が approved した状態で十分。

#### 3. テナント A 第二 Approver の追加

```go
// 定数追加
UserApprover2ID = "aaaaaaaa-2222-2222-2222-000000000023"

// users テーブル INSERT
{UserApprover2ID, "test-approver2@example.com", "Test Approver Two"}

// tenant_memberships INSERT
{TenantAID, UserApprover2ID, domain.RoleApprover}
```

#### 4. テナント A 第二 Approver が処理したレポート追加

SMK-105 用に「テナント A 内で第二 Approver (UserApprover2ID) が承認したレポート」を 1 件追加。既存 ReportApprovedID は UserApproverID（既存 Approver）が処理したものとして区別する形にする（既存のテストで参照されていれば現状維持、新規レポート ID で別途追加）。

```go
// 定数追加
ReportApprovedByApprover2ID = "cccccccc-9999-9999-9999-000000000099"

// expense_reports INSERT（テナント A、Member 作成、Approver2 が承認した状態）
```

### 既存テストへの波及確認

- `cross_cutting_test.go`（11-B で実装予定、現時点では未実装）
- `internal/handler/*_test.go`（テナント分離・RBAC マトリクス検証で UserApproverID / UserMemberBID を参照している箇所）
- `internal/seed/seed_test.go`（フィクスチャ件数を hardcode していないか）

新規定数追加と新規レポート追加は基本的に下位互換だが、件数を expect しているテストがあれば調整する。

## 修正対象ファイル（推定）

| ファイル | 変更内容 |
|---|---|
| `expense-saas/internal/seed/seed.go` | 上記 1〜4 の追加 |
| `expense-saas/internal/seed/seed_test.go` | 件数アサーションがあれば更新 |
| 既存 BE テスト群 | フィクスチャ件数依存のテストを調整（必要に応じて） |

## 完了条件

- `seed.go` にテナント B Approver / テナント A 第二 Approver / Approver2 処理済みレポートが追加されている
- `docker compose up` で seed が成功し、追加フィクスチャが DB に投入される
- 既存 BE テスト（unit + integration）が通過する
- ユーザーが SMK-104 / SMK-105 を実施可能になる

## ラベル

- type: testing / infrastructure
- area: backend / fixture

## 関連

- 関連 issue: #158（Approver 処理済み一覧 SCR-WFL-003 新規、本 issue は #158 の SMK 検証用前提整備）
- 関連 SMK: SMK-104 / SMK-105

## MVP 区分

MVP（Step 11-A クローズ条件）
