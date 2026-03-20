# UC-M01〜M07 のアクター定義に Accounting が未記載

## 発見日
2026-03-20

## カテゴリ
requirements

## 影響度
中

## 発見経緯
proactive

## 関連ステップ
Step 1（usecases.md）

## 問題
Issue 024 で Accounting に Member 相当の経費申請権限を付与したが、usecases.md の UC-M01〜M07（レポート作成・明細追加・領収書添付・添付削除・レポート編集・明細編集削除・レポート提出・レポート削除）のアクター定義には Accounting が含まれていない。

UC-M08/M09 は Issue 024 の review-findings 034/035 で修正されたが、UC-M01〜M07 は指摘対象外だったため未修正。

## 影響
- rbac.md の権限マトリクスでは Accounting に申請系操作が許可されているのに、usecases.md のユースケース定義では Member のみとなっている
- Step 1 内の内部不整合（rbac.md ↔ usecases.md）
- 後続の詳細設計・実装で参照するユースケース定義が不完全

## 提案
1. UC-M01〜M07 のアクター定義に Accounting を追加（既存パターン「Approver/Admin も自分のレポートで実行可能」に Accounting を追記）
2. ユースケース一覧サマリ（セクション8）のアクター列も同様に更新
3. UC-M03a（添付ファイル削除）も同様に更新

### 修正対象
- `usecases.md` UC-M01〜M07 + UC-M03a の本文アクター定義（8箇所）
- `usecases.md` ユースケース一覧サマリのアクター列（8行）

---

## 解決内容

## 解決日
