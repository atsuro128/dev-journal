# SCR-WFL-001: 承認待ち一覧

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 「承認待ち一覧」画面の詳細仕様を定義する |
| 正本情報 | 一覧項目、承認/却下操作、API 連携、エラー表示 |
| 扱わない内容 | 全画面共通の UI ガイドライン（ui-guidelines.md）、画面間の遷移定義（ui_flow.md）、API 詳細定義（openapi.yaml） |
| 主な参照元 | `40_basic_design/ui_flow.md`, `40_basic_design/screens.md`, `50_detail_design/openapi.yaml`, `50_detail_design/authz.md` |
| 主な参照先 | `60_test/test_cases/workflow.md` |

## 1. 基本情報

| 項目 | 内容 |
|------|------|
| **画面ID** | SCR-WFL-001 |
| **画面名** | 承認待ち一覧 |
| **URL パス** | `/approvals` |
| **対応要件ID** | WFL-F04（承認待ち一覧） |
| **対応 UC** | UC-A01（承認待ちレポートを確認する） |
| **対応 API** | `GET /api/workflow/pending` |
| **アクセス可能ロール** | Approver |
| **アクセス不可ロールの挙動** | ダッシュボード（SCR-DASH-001）にリダイレクト |

## 2. 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `40_basic_design/screens.md` | 画面一覧・画面ID・共通UIパターン |
| `10_requirements/usecases.md` | UC-A01 |
| `10_requirements/policies.md` | 状態遷移定義（SS4）、承認フロー権限（SS3） |
| `20_domain/state_machine.md` | 遷移 T2/T3 の事前条件・事後条件 |
| `deliverables/docs/01_glossary.md` | 用語集 |

## 3. アクションの責務分担（screens/report-detail.md との接点）

| 操作 | 実行画面 | 本画面での役割 |
|------|---------|---------------|
| 承認（approve） | SCR-RPT-004 | SCR-WFL-001 から SCR-RPT-004 へ遷移して実行 |
| 却下（reject） | SCR-RPT-004 | SCR-WFL-001 から SCR-RPT-004 へ遷移して実行 |

本画面は **一覧表示とレポート詳細画面へのナビゲーション** のみを担う。

---

## 4. レイアウト

```
┌─────────────────────────────────────────────────────────┐
│ ヘッダー（共通）                                          │
├──────────┬──────────────────────────────────────────────┤
│          │ ページタイトル: 「承認待ち一覧」                 │
│          │                                              │
│  サイド   │ ┌──────────────────────────────────────────┐ │
│  ナビ     │ │ フィルタエリア                              │ │
│          │ └──────────────────────────────────────────┘ │
│          │                                              │
│          │ ┌──────────────────────────────────────────┐ │
│          │ │ 件数表示: 「N 件の承認待ちレポート」          │ │
│          │ ├──────────────────────────────────────────┤ │
│          │ │ テーブル                                    │ │
│          │ │ ┌──────┬──────┬──────┬──────┬──────┐     │ │
│          │ │ │申請者 │タイトル│合計金額│提出日 │      │     │ │
│          │ │ ├──────┼──────┼──────┼──────┼──────┤     │ │
│          │ │ │ ...  │ ...  │ ...  │ ...  │  →   │     │ │
│          │ │ └──────┴──────┴──────┴──────┴──────┘     │ │
│          │ ├──────────────────────────────────────────┤ │
│          │ │ [ページネーションコントロール]                │ │
│          │ │  < 1 2 3 ... 8 9 10 >                       │ │
│          │ └──────────────────────────────────────────┘ │
│          │                                              │
│          │ ※ 空状態: 「承認待ちのレポートはありません。」    │
└──────────┴──────────────────────────────────────────────┘
```

## 5. 表示項目

### テーブルカラム

| # | カラム名 | データソース | 表示形式 | ソート |
|---|---------|------------|---------|-------|
| 1 | 申請者名 | `submitter.name`（レポート作成者） | テキスト | - |
| 2 | タイトル | `expense_report.title` | テキスト（リンク: SCR-RPT-004 へ遷移） | - |
| 3 | 合計金額 | `expense_report.total_amount` | `¥` プレフィックス + 3桁カンマ区切り（例: ¥12,500） | - |
| 4 | 提出日 | `expense_report.submitted_at` | 日付形式（例: "2026/03/15"） | デフォルト降順（新しい順） |
| 5 | 遷移アイコン | - | 右矢印アイコン（行クリックで遷移可能であることを示す） | - |

### 自己承認禁止の表示ルール

Approver 自身が作成したレポートが submitted 状態で一覧に含まれる場合、以下のルールを適用する。

| ルール | 内容 | 根拠 |
|--------|------|------|
| 一覧での表示 | **一覧に表示する**（除外しない） | Approver は承認待ち全件を俯瞰する必要がある |
| 自己レポート行の識別 | 申請者名の横に「自分」ラベルを表示 | 視覚的に自分のレポートであることを示す |
| 行クリック時の遷移 | SCR-RPT-004 に通常通り遷移する | 詳細画面で承認・却下ボタンが非表示になる（SCR-RPT-004 側で制御） |

> 自己承認禁止はレポート詳細画面（SCR-RPT-004）でボタン非表示により制御される。一覧画面では除外せず表示する方針とする。これにより Approver は承認待ち全件を把握でき、他の Approver が対応すべきレポートとして認識できる。

## 6. フィルタ

| # | フィルタ名 | 入力形式 | 選択肢 / 制約 | デフォルト値 | API パラメータ |
|---|-----------|---------|-------------|------------|---------------|
| 1 | 申請者名 | テキスト入力 | 部分一致検索 | 空（全件） | `applicant_name` |

- フィルタはリアルタイム適用（入力後に自動検索、デバウンス 300ms）
- フィルタリセットボタン: 全フィルタを初期値に戻す
- フィルタ条件は AND 結合

> 承認待ち一覧は submitted 状態のレポートのみが対象であるため、ステータスフィルタは不要。

## 7. ページネーション

| 項目 | 仕様 |
|------|------|
| 方式 | オフセットベースページネーション（screens.md §4.9 準拠） |
| 1ページあたりの件数 | デフォルト 20 件 |
| フッター配置 | **`AppDataGrid` の `slots.footer` プロパティに `AppPaginationFooter` を差し込み、DataGrid フッターコンテナに統合する**（issue #147 再オープン 2026-04-27 確定方針 D-1。クラス名定義は `55_ui_component/common-components.md` §AppDataGrid を参照）。テーブル外側の独立した `<Box>` には配置しない。本画面は `AppDataGrid` を直接利用するため、画面側で `<AppDataGrid slots={{ footer: () => <AppPaginationFooter ... /> }} />` の形で直接 `slots.footer` に渡す（issue #147 再オープン パターン ②a） |
| フッター構成 | 共通 `AppPaginationFooter` を使用。中央: ページ番号（`AppPagination`、現在ページハイライト、省略表示）、右: 表示件数セレクタ（`PageSizeSelector`、標準選択肢 `[10, 20, 50, 100]`、デフォルト 20） |
| レスポンシブ | 375px 等のスマホ幅では `flex-direction: column` で縦並びにフォールバック |
| 表示制御 | **フッターは常時表示**（issue #147 Q3）。`totalPages <= 1` でも非表示にしない。内部 `AppPagination` は `count={Math.max(totalPages, 1)}` でページ番号「1」を常時表示する |
| API パラメータ | `page`（デフォルト 1）、`per_page`（デフォルト 20、最大 100。範囲外は BE バリデーションで 422） |
| URL ⇔ UI 反映 | URL の `?per_page=N` を `PageSizeSelector` に反映（標準外値は動的に選択肢に追加）。セレクタ操作で URL クエリ `per_page` を更新 |
| per_page 変更時 | page=1 にリセット（既存フィルタ変更と同一パターン）。`setSearchParams` は 1 回のコールに集約 |
| FE フォールバック | per_page の NaN/負数 URL 値は FE 側で 20 にフォールバック（issue #147 Q4）。範囲内不正値は BE バリデーションに委ねる |
| ソート順 | 提出日の降順（新しい提出が上位に表示） |
| フィルタ変更時 | page を 1 にリセットする |
| Props 型 | `55_ui_component/common-components.md` §AppPaginationFooter / §PageSizeSelector 参照 |

## 8. 件数表示

テーブル上部に件数を表示する。

| 表示条件 | 表示テキスト |
|---------|------------|
| 1件以上 | 「N 件の承認待ちレポート」 |
| 0件 | 空状態メッセージを表示（§9 参照） |
| フィルタ適用中で0件 | 「条件に一致するレポートはありません。」 |

## 9. 空状態

| 条件 | メッセージ | 補足 |
|------|-----------|------|
| 承認待ちレポートが0件 | 「承認待ちのレポートはありません。」 | screens.md §4.7 準拠 |
| フィルタ適用中に0件 | 「条件に一致するレポートはありません。」 | フィルタリセットボタンを併記 |

## 10. ローディング

| 状態 | 表示 |
|------|------|
| 初回読み込み中 | テーブル行のスケルトン UI（5行分） |
| ページ切替時 | テーブル領域にスケルトンUIを表示 |

## 11. エラー表示

> **実装者注記（issue #134）**: 下表の「メッセージ」欄の日本語文言は `frontend/src/lib/error-messages.ts` の `SERVER_ERROR_MESSAGES` の期待値であり、コンポーネント内でハードコードするものではない。onError ハンドラでは `err.message` をそのまま使うこと（`.claude/rules/implementation-workflow.md` 「FE エラーハンドリング」参照）。

| エラー種別 | HTTP ステータス | 表示方式 | メッセージ |
|-----------|---------------|---------|-----------|
| サーバーエラー | 500 | トースト（画面上部） | 「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」 |
| 認証エラー | 401 | リダイレクト | ログイン画面（SCR-AUTH-002）へリダイレクト |
| 認可エラー | 403 | リダイレクト | ダッシュボード（SCR-DASH-001）へリダイレクト |

## 12. 行クリック時の遷移

| 操作 | 遷移先 | 遷移方法 |
|------|--------|---------|
| テーブル行クリック | SCR-RPT-004（`/reports/:id`） | 画面遷移（ブラウザ履歴に追加） |
| タイトルリンククリック | SCR-RPT-004（`/reports/:id`） | 同上 |

遷移先のレポート詳細画面（SCR-RPT-004）では、ロールと状態に応じて以下のアクションボタンが表示される。

| レポートの状態 | 閲覧者 | 承認ボタン | 却下ボタン | 備考 |
|-------------|--------|----------|----------|------|
| submitted | Approver（自分のレポートでない） | 表示 | 表示 | 承認コメント任意（0〜1000文字）、却下理由必須（1〜1000文字） |
| submitted | Approver（自分のレポート） | **非表示** | **非表示** | 自己承認禁止（RBC-016） |

---

## 13. 共通仕様

### 13.1 データのリアルタイム性

| 項目 | 仕様 |
|------|------|
| データの鮮度 | 画面表示時に API を呼び出して最新データを取得する |
| 自動リフレッシュ | 実装しない（MVP）。手動でブラウザリロードにより最新化 |
| 他ユーザーの操作反映 | 画面表示後に他の Approver が承認/却下した場合、一覧には反映されない。レポート詳細画面で操作しようとした際に楽観的ロックで競合を検知する |

### 13.2 楽観的ロックとの連携

一覧画面自体はリスト表示のみのため楽観的ロックは不要だが、遷移先のレポート詳細画面（SCR-RPT-004）でアクション実行時に楽観的ロック（`updated_at` チェック）が適用される。

| シナリオ | 挙動 |
|---------|------|
| Approver A が承認待ち一覧からレポートを開いている間に、Approver B が同じレポートを承認した | Approver A がレポート詳細画面で承認を実行すると、状態が submitted でなくなっているため `InvalidStateTransition` エラーが返る。「このレポートは既に処理されています。」というエラーメッセージを表示し、一覧に戻す |

### 13.3 テナント分離

| 項目 | 仕様 |
|------|------|
| データスコープ | 同一テナントのレポートのみ表示（API 側で `tenant_id` フィルタ + RLS 二重保証） |
| 他テナントのレポート ID を URL に直接指定した場合 | 404 Not Found を返す（存在漏洩防止） |

---

## 14. SCR-RPT-004 で実行されるワークフローアクション（参照）

以下はレポート詳細画面（SCR-RPT-004、screens/report-detail.md で定義）で実行されるアクションの概要を参照として記載する。承認待ち一覧からの遷移後に実行されるアクションとの整合を示す。

### 14.1 承認（T2: submitted → approved）

| 項目 | 内容 |
|------|------|
| 実行者 | Approver（同テナント） |
| 事前条件 | ステータスが submitted、自己承認でないこと（RBC-016） |
| 確認ダイアログ | 「このレポートを承認しますか?」 |
| 承認コメント | 任意、0〜1000 文字 |
| API | `POST /api/workflow/:id/approve` |
| 成功時 | ステータスが approved に遷移。成功トースト表示 |
| 失敗時（状態不整合） | 「このレポートは既に処理されています。」をトースト表示 |
| 失敗時（自己承認） | 「自分のレポートは承認できません。」をトースト表示 |

### 14.2 却下（T3: submitted → rejected）

| 項目 | 内容 |
|------|------|
| 実行者 | Approver（同テナント） |
| 事前条件 | ステータスが submitted、自己操作でないこと（RBC-016） |
| 確認ダイアログ | 「このレポートを却下しますか?」 |
| 却下理由 | **必須**、1〜1000 文字（WFL-012） |
| API | `POST /api/workflow/:id/reject` |
| 成功時 | ステータスが rejected に遷移。成功トースト表示 |
| 失敗時（却下理由未入力） | フィールドレベルエラー「却下理由を入力してください。」 |
| 失敗時（却下理由超過） | フィールドレベルエラー「却下理由は1000文字以内で入力してください。」 |
| 失敗時（状態不整合） | 「このレポートは既に処理されています。」をトースト表示 |

---

## 15. API リクエスト/レスポンス

### GET /api/workflow/pending

リクエスト:

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| page | Integer | 任意 | ページ番号（デフォルト 1） |
| per_page | Integer | 任意 | 1ページあたりの取得件数（デフォルト 20、最大 100） |
| applicant_name | String | 任意 | 申請者名部分一致フィルタ |

レスポンス:

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "2026年3月 営業経費",
      "total_amount": 12500,
      "submitted_at": "2026-03-15T10:30:00Z",
      "submitter": {
        "id": "uuid",
        "name": "一般 次郎"
      },
      "is_own_report": false
    }
  ],
  "pagination": {
    "current_page": 1,
    "per_page": 20,
    "total_count": 15,
    "total_pages": 1
  }
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| id | UUID | レポート ID（遷移先 URL の構築に使用） |
| title | String | レポートタイトル |
| total_amount | Integer | 合計金額（円） |
| submitted_at | Timestamp | 提出日時 |
| submitter.id | UUID | 申請者 ID |
| submitter.name | String | 申請者名 |
| is_own_report | Boolean | 自分が作成したレポートか（自己承認禁止の「自分」ラベル表示に使用） |

---

## 16. 処理シーケンス

### 承認待ち一覧取得

```mermaid
sequenceDiagram
    participant F as フロント
    participant H as ハンドラ
    participant S as サービス
    participant R as リポジトリ
    participant DB as DB

    F->>H: GET /api/workflow/pending?page=1&per_page=20&applicant_name=...
    Note right of H: JWT検証（Authミドルウェア）<br/>Approverロール検証（RBACミドルウェア）<br/>TenantContext設定（RLS）
    H->>H: クエリパラメータのバリデーション<br/>per_page: 1-100 / page: 1以上 / applicant_name: string
    H->>S: ListPendingReports(tenantID, approverUserID, filters, page, perPage)
    S->>R: FindPendingReports(tenantID, approverUserID, filters, page, perPage)
    R->>DB: SELECT COUNT(*)<br/>FROM expense_reports er JOIN users u ON er.user_id = u.user_id<br/>WHERE er.tenant_id=? AND er.status='submitted'<br/>AND er.deleted_at IS NULL<br/>[AND u.name LIKE ?]
    DB-->>R: total_count
    R->>DB: SELECT er.id, er.title, er.total_amount,<br/>er.submitted_at, u.id, u.name<br/>FROM expense_reports er JOIN users u ON er.user_id = u.user_id<br/>WHERE er.tenant_id=? AND er.status='submitted'<br/>AND er.deleted_at IS NULL<br/>[AND u.name LIKE ?]<br/>ORDER BY er.submitted_at DESC, er.id DESC<br/>LIMIT per_page OFFSET (page-1)*per_page
    Note right of DB: RBC-016: 自己レポートは is_own_report フラグで表示、<br/>承認ボタンを非活性化
    DB-->>R: rows
    R->>R: total_pages = ceil(total_count / per_page)
    R-->>S: reports + pagination{current_page, per_page, total_count, total_pages}
    S->>S: 各レポートに is_own_report フラグを付与<br/>is_own_report: (er.user_id == approverUserID)
    S-->>H: ListPendingReportsResponse
    H-->>F: 200 OK
    Note left of F: ページネーションコントロールを表示<br/>current_page をハイライト<br/>ページ番号クリックで該当ページを再リクエスト
```

---

## 17. 品質チェック

- [x] screens.md §3.4 の画面定義と画面ID・URL・対応UCが一致しているか
- [x] UC-A01（承認待ちレポートを確認する）の正常系フローが画面仕様に反映されているか
- [x] 承認/却下のアクション自体は SCR-RPT-004 で行う旨が明確に記載されているか
- [x] 自己承認禁止の表示ルールが定義されているか（一覧から除外しない + 「自分」ラベル + SCR-RPT-004 でボタン非表示）
- [x] policies.md SS3.5 の権限マトリクスと一致しているか（Approver のみ承認待ち）
- [x] ページネーション仕様が screens.md §4.9 と一致しているか（オフセットベース、20件/ページ、ページネーションコントロール）
- [x] 空状態メッセージが screens.md §4.7 と一致しているか
- [x] エラー表示が screens.md §4.4 と一致しているか
- [x] ローディング仕様が screens.md §4.5 と一致しているか
- [x] state_machine.md の T2/T3 遷移の事前条件が参照セクション（§14）に正しく反映されているか
- [x] 却下理由必須（WFL-012）・承認コメント任意の仕様が反映されているか
- [x] 用語が glossary.md に準拠しているか（提出/却下）
- [x] API エンドポイントが architecture.md §5.1 と一致しているか
- [x] テナント分離のルール（他テナントアクセス時 404）が記載されているか
