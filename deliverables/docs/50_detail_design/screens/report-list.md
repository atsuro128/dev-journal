# SCR-RPT-001: レポート一覧（自分）

## 1. 基本情報

| 項目 | 内容 |
|------|------|
| 画面ID | SCR-RPT-001 |
| 画面名 | レポート一覧（自分） |
| URLパス | `/reports` |
| 対応UC | UC-M08（レポートの状況を確認する） |
| 対応ロール | Member, Approver, Admin, Accounting |
| 使用API | GET /api/reports |
| 目的 | 自分が作成した経費レポートの一覧を表示し、ステータスや期間で絞り込む |

### 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `40_basic_design/screens.md` | 画面一覧・共通UIパターン |
| `40_basic_design/ui_flow.md` | 画面遷移図 |
| `10_requirements/usecases.md` | UC-M08 |
| `10_requirements/requirements.md` | RPT-F01 ~ F07 |
| `10_requirements/rbac.md` | 権限マトリクス |
| `references/glossary.md` | 用語集 |

---

## 2. レイアウト

```
+------------------------------------------------------+
| [共通ヘッダー]                                         |
+----------+-------------------------------------------+
|          | ページタイトル: マイレポート                   |
|  サイド   |                                             |
|  ナビ     | [フィルタエリア]                              |
|          | ステータス: [全て v]  期間: [開始日] ~ [終了日]|
|          |                                             |
|          | [+ レポート作成]  <-- 右上ボタン              |
|          |                                             |
|          | [レポートテーブル]                             |
|          | +------+------+------+--------+------+      |
|          | |タイトル|対象期間|合計金額|ステータス|作成日 |      |
|          | +------+------+------+--------+------+      |
|          | | ...  | ...  | ...  | badge  | ...  |      |
|          | +------+------+------+--------+------+      |
|          |                                             |
|          | [さらに読み込む]  <-- has_more 時のみ表示     |
+----------+-------------------------------------------+
```

---

## 3. 表示項目

### レポートテーブル

| # | 項目 | 表示形式 | ソートキー |
|---|------|---------|-----------|
| 1 | タイトル | テキスト。クリックで SCR-RPT-004（report-detail.md）に遷移 | - |
| 2 | 対象期間 | `YYYY/MM/DD ~ YYYY/MM/DD` | - |
| 3 | 合計金額 | `¥` プレフィックス + 3桁カンマ区切り（例: ¥12,500） | - |
| 4 | ステータス | ステータスバッジ（screens.md 4.8 準拠） | - |
| 5 | 作成日 | `YYYY/MM/DD` | デフォルト降順 |

### ステータスバッジ色（screens.md 4.8 準拠）

| ステータス | 日本語表記 | バッジ色 |
|-----------|-----------|---------|
| draft | 下書き | グレー |
| submitted | 提出済み | 青 |
| approved | 承認済み | 緑 |
| rejected | 却下 | 赤 |
| paid | 支払済み | 紫 |

---

## 4. フィルタ

| # | フィルタ項目 | 入力形式 | 初期値 | 説明 |
|---|------------|---------|--------|------|
| 1 | ステータス | ドロップダウン（単一選択） | 全て | 選択肢: 全て / 下書き / 提出済み / 承認済み / 却下 / 支払済み |
| 2 | 対象期間（開始） | 日付ピッカー | 空（未指定） | 指定した日付以降の period_start を持つレポートを表示 |
| 3 | 対象期間（終了） | 日付ピッカー | 空（未指定） | 指定した日付以前の period_end を持つレポートを表示 |

- フィルタ変更時にリストを即時更新する（デバウンス処理は日付入力のみ適用）
- フィルタ条件は URL のクエリパラメータに反映し、ブラウザバック時に復元可能とする

---

## 5. ページネーション

- カーソルベース、デフォルト 20件/ページ（screens.md 4.9 準拠）
- 一覧末尾に「さらに読み込む」ボタンを配置
- `has_more: true` の場合のみボタンを表示
- ボタン押下で次ページを取得し、既存リストの末尾に追加

---

## 6. 操作

| # | 操作 | トリガー | 遷移先 |
|---|------|---------|--------|
| 1 | レポート作成 | 右上の「+ レポート作成」ボタン | SCR-RPT-002（report-create.md） |
| 2 | レポート詳細表示 | テーブル行クリック | SCR-RPT-004（report-detail.md）`/reports/:id` |

---

## 7. 空状態

データが存在しない場合（screens.md 4.7 準拠）:

> 「経費レポートはまだありません。レポートを作成して経費精算を始めましょう。」

- 「レポート作成」ボタンも空状態メッセージの下に表示する

---

## 8. ローディング

- 初回読み込み: テーブル行のスケルトンUI（screens.md 4.5 準拠）
- 追加読み込み: 「さらに読み込む」ボタンにスピナーを表示

---

## 9. エラーハンドリング

| エラー | 表示方法 |
|--------|---------|
| API通信エラー（500系） | トーストで「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」 |
| 認証エラー（401） | ログイン画面にリダイレクト |

---

## 10. ロール別表示差異

本画面はロールによる表示差異はない。全ロール共通で自分のレポートのみ表示する。

---

## 11. 画面遷移

| # | 遷移元 | トリガー | 遷移先 |
|---|--------|---------|--------|
| 1 | SCR-RPT-001 | テーブル行クリック | SCR-RPT-004（report-detail.md） |
| 2 | SCR-RPT-001 | 「+ レポート作成」ボタン | SCR-RPT-002（report-create.md） |
| 3 | SCR-RPT-002（report-create.md） | キャンセル | SCR-RPT-001 |
| 4 | SCR-RPT-004（report-detail.md） | レポート削除成功 | SCR-RPT-001 |
| 5 | SCR-DASH-001（dashboard.md） | マイレポート | SCR-RPT-001 |

---

## 12. API リクエスト/レスポンス

### GET /api/reports

自分が作成した経費レポートの一覧を取得する。

#### リクエスト

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| status | String | 任意 | ステータスフィルタ（`draft`, `submitted`, `approved`, `rejected`, `paid`） |
| from | String (date) | 任意 | 対象期間の開始日（`YYYY-MM-DD`） |
| to | String (date) | 任意 | 対象期間の終了日（`YYYY-MM-DD`） |
| cursor | String | 任意 | 前回レスポンスの next_cursor |
| limit | Integer | 任意 | 取得件数（デフォルト 20、最大 100） |

#### レスポンス（200 OK）

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "2026年3月 営業経費",
      "period_start": "2026-03-01",
      "period_end": "2026-03-31",
      "status": "submitted",
      "total_amount": 12500,
      "submitted_at": "2026-03-15T10:30:00Z",
      "created_at": "2026-03-10T09:00:00Z",
      "updated_at": "2026-03-15T10:30:00Z"
    }
  ],
  "pagination": {
    "next_cursor": "...",
    "has_more": true
  }
}
```

#### テーブル表示項目とレスポンスフィールドのマッピング

| テーブルカラム | レスポンスフィールド | 表示変換 |
|-------------|-------------------|---------|
| タイトル | `title` | そのまま表示 |
| 対象期間 | `period_start`, `period_end` | `YYYY/MM/DD ~ YYYY/MM/DD` 形式 |
| 合計金額 | `total_amount` | `¥` プレフィックス + 3桁カンマ区切り |
| ステータス | `status` | ステータスバッジ（screens.md 4.8 準拠） |
| 作成日 | `created_at` | `YYYY/MM/DD` 形式 |

#### エラーレスポンス

| HTTP ステータス | エラーコード | 説明 |
|---------------|------------|------|
| 401 | UNAUTHORIZED | 認証エラー。ログイン画面にリダイレクト |

---

## 13. 処理シーケンス

### レポート一覧取得

```mermaid
sequenceDiagram
    participant F as フロント
    participant H as ハンドラ
    participant S as サービス
    participant R as リポジトリ
    participant DB as DB

    F->>H: GET /api/reports?status=...&from=...&to=...&cursor=...&limit=20
    Note right of F: ページ表示時 / フィルタ変更時に呼び出し
    Note right of H: JWT検証（Authミドルウェア）<br/>TenantContext設定（RLS）
    H->>H: クエリパラメータのバリデーション<br/>status: enum / from,to: date / limit: 1-100
    H->>S: ListReports(tenantID, userID, filters)
    S->>R: FindReports(tenantID, userID, filters, cursor, limit+1)
    R->>DB: SELECT id, title, period_start, period_end,<br/>status, total_amount, submitted_at,<br/>created_at, updated_at<br/>FROM expense_reports<br/>WHERE tenant_id=? AND user_id=?<br/>AND deleted_at IS NULL<br/>[AND status=?]<br/>[AND period_start >= ?]<br/>[AND period_end <= ?]<br/>[AND (updated_at, id) < (cursor_ts, cursor_id)]<br/>ORDER BY updated_at DESC, id DESC<br/>LIMIT 21
    Note right of DB: LIMIT N+1 で取得し<br/>N+1件目の有無で<br/>has_more を判定
    DB-->>R: rows（最大21件）
    R->>R: len(rows) > limit の場合<br/>has_more=true, 末尾行を除外<br/>最終行の (updated_at, id) を next_cursor に設定
    R-->>S: reports + pagination
    S-->>H: ListReportsResponse
    H-->>F: 200 OK
    Note left of F: has_more=true の場合<br/>「さらに読み込む」ボタンを表示<br/>押下時に next_cursor を付与して再リクエスト
```

---

## 14. 品質チェック

- [x] UC-M08 の全表示項目が定義されているか
- [x] フィルタ・ページネーション・空状態・ローディングが定義されているか
- [x] ステータスバッジ色が screens.md 4.8 に準拠しているか
- [x] 用語が glossary.md に準拠しているか
- [x] MVP スコープ外の機能を含めていないか
