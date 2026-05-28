# draft レポートの Admin / Accounting 閲覧可否の見直し検討（post-MVP）

## 発見日
2026-05-27

## カテゴリ
authorization / ux

## 影響度
低（業務継続可能、現状は MVP 仕様通り）

## 発見経緯
UAT-020（Admin 業務シナリオ — 全レポート俯瞰）

## 関連ステップ
Step 11-F UAT / 設計成果物: `dev-journal/deliverables/docs/50_detail_design/authz.md` §10.1 / `dev-journal/deliverables/docs/10_requirements/policies.md` RBC-013

## ブロッカー
なし（MVP 仕様として意図された設計、機能としては正常動作）

## 問題

Admin / Accounting ロールが、同テナントの**他ユーザーが作成した draft（下書き）レポート**を閲覧できる仕様について、業務上の違和感が UAT で報告された。

### 設計の現状（`authz.md` §10.1 レポート閲覧権限）

| ロール | 閲覧範囲 | 根拠 |
|---|---|---|
| Member | 自分のレポートのみ | RBC-010 |
| Approver | 自分 + submitted のみのテナント全レポート + 自分が承認/却下した分 | RBC-011, RBC-015 |
| Accounting | 自分 + テナント内全レポート（**draft 含む全ステータス**） | RBC-012 関連、UC-AC03 |
| Admin | テナント内全レポート（**draft 含む全ステータス**） | RBC-013 |

`GET /api/reports/all` で draft も返却され、`SCR-ADMIN-002`（全レポート画面）に表示される。

### 業務上の違和感

- draft（下書き）は申請者の**私的な作業中データ**であり、以下のような性質を持つ:
  - 金額・摘要が未確定（仮置きされた値が他者に見える）
  - 後で削除される可能性が高い試案
  - 提出されないまま放置される個人メモ的なレポート
- 提出前の draft を Admin / Accounting が見られると、申請者の心理的安全性に影響（仮入力を見られる気まずさ、削除前提のテストデータが監査対象に映る等）
- 一方で、Approver の閲覧範囲は **submitted 以降に限定**されており、draft は対象外。Approver と Admin/Accounting で粒度が揃っていない

### 監査・統制とのバランス

- RBC-013（Admin 全レポート閲覧）/ UC-AC03（経理の月次経費俯瞰）の意図は**監査・統制**目的であり、提出済みレポートが対象として自然
- draft を含める必要性は限定的（「Member が長期間 draft で放置している」を Admin が検知するユースケースは MVP 対象外）

## 影響

- UAT-020 で UX 観点の違和感として顕在化
- 機能としては設計通り動作しており、ブロッカーではない
- ただし、Member の心理的安全性 + Approver との一貫性の観点で post-MVP 検討の価値あり

## 提案

### 案 A: Admin / Accounting の閲覧範囲から draft を除外
- `GET /api/reports/all` のクエリ条件に `status != 'draft'` を追加
- `authz.md` §10.1 を更新（Admin / Accounting の閲覧範囲を「submitted 以降」に変更）
- メリット: Approver と粒度が揃う、申請者の心理的安全性が確保される
- デメリット: 「draft で長期放置」を検知するユースケースが消失（ただし MVP 対象外）

### 案 B: 設定可能なオプションとして提供（テナント設定）
- テナント設定画面（Phase 3 SCR-ADM-012）に「draft 閲覧範囲」設定を追加
- デフォルト: draft 非公開（作成者のみ）
- 設定により Admin / Accounting に開放可能
- メリット: 組織のポリシー次第で切り替えられる
- デメリット: 設定 UI と DB スキーマ拡張が必要

### 案 C: 現状維持（MVP 仕様通り）
- RBC-013 / UC-AC03 の意図通り、Admin / Accounting に全レポート閲覧権限を維持
- メリット: 監査機能が完全
- デメリット: UAT で指摘された違和感が残る

### 推奨

**案 A**（draft を Admin/Accounting の閲覧範囲から除外）を post-MVP の早期に検討。Approver の挙動（submitted 以降のみ）との一貫性が取れ、Member の心理的安全性も確保される。監査用途では draft を含めない方が自然（draft は申請者の意思決定前のデータ）。

## 着手タイミング

**post-MVP**。MVP リリース後に UX + 認可設計の見直しとして検討する。

---

## 解決内容
<!-- pending-review へ移動する前に記入 -->

## 解決日
<!-- YYYY-MM-DD -->
