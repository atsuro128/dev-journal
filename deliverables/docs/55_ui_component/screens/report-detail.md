# レポート詳細 -- コンポーネント設計

## この文書の役割

| 項目 | 内容 |
|---|---|
| 目的 | 1 画面のコンポーネントツリー・Props 型定義・データフローを定義する |
| 正本情報 | コンポーネント分割・Props 型・データフロー・状態管理 |
| 扱わない内容 | 明細・添付・ワークフローの詳細ロジック（50_detail_design/screens/report-detail.md）、共通コンポーネント詳細（common-components.md） |
| 主な参照元 | `50_detail_design/screens/report-detail.md`, `55_ui_component/common-components.md`, `55_ui_component/state-management.md` |
| 主な参照先 | `60_test/test_cases/*.md` |

---

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 画面 ID | `SCR-RPT-004` |
| 画面名 | レポート詳細 |
| URL パス | `/reports/:id` |
| 画面詳細仕様 | `50_detail_design/screens/report-detail.md` |

---

## 2. コンポーネントツリー

```
ReportDetailPage
└── AppLayout（← common-components.md）
    ├── AppHeader（← common-components.md）
    ├── AppSidebar（← common-components.md）
    └── [メインコンテンツ]
        ├── AppBreadcrumbs（← common-components.md）
        ├── PageSkeleton（← common-components.md）[データ読み込み中]
        ├── EmptyState（← common-components.md）[404 Not Found 時]
        ├── ReportInfoCard
        │   ├── ReportBasicInfo
        │   │   └── StatusChip（← common-components.md）
        │   ├── ReportWorkflowInfo
        │   └── ReportReferenceLink
        ├── ReportActionBar
        │   ├── OwnerActions
        │   └── WorkflowActions
        ├── ItemListSection
        │   ├── ItemListHeader
        │   ├── ItemTable
        │   │   └── AppDataGrid（← common-components.md）
        │   └── EmptyState（← common-components.md）[明細0件時]
        ├── ItemSlidePanel
        │   ├── ItemForm
        │   │   ├── FormAlert（← common-components.md）[apiError 表示]
        │   │   ├── AppDatePicker（← common-components.md）[日付]
        │   │   ├── AppTextField（← common-components.md）[金額]
        │   │   ├── AppSelect（← common-components.md）[カテゴリ]
        │   │   └── AppTextField（← common-components.md）[摘要]
        │   └── AttachmentArea
        │       ├── AttachmentList
        │       │   └── AttachmentItemRow × N（per-item で useAttachmentDownloadUrl / useAttachmentPreviewUrl を保持）
        │       └── AttachmentUploader
        └── ConfirmDialog（← common-components.md）
```

---

## 3. コンポーネント定義

### ReportDetailPage

- 配置: `pages/reports/ReportDetailPage.tsx`
- 責務: レポート詳細画面のページコンポーネント。URL パスパラメータ `:id` からレポート ID を取得し、useReport Hook でデータを読み込む。レポートが存在しない場合はメインコンテンツに not found メッセージ（「指定されたデータが見つかりません。」）とレポート一覧へのリンクを表示する。useCurrentUser で取得した現在のユーザー情報とレポートデータを組み合わせて、所有権・ロールに基づく表示制御を子コンポーネントに伝播する。全ワークフロー操作（提出・削除・承認・却下・支払完了）の確認ダイアログ制御と、明細スライドパネルの開閉制御を管理する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;1, &sect;8, &sect;11, &sect;12

```typescript
// Props なし（ページコンポーネント）
```

### ReportInfoCard

- 配置: `pages/reports/ReportInfoCard.tsx`
- 責務: レポートの基本情報と状態遷移関連情報（承認者・却下理由・支払処理者等）をカード形式で表示する。常時表示項目と条件付き表示項目の出し分けを行う
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;3

```typescript
interface ReportInfoCardProps {
  /** レポート詳細データ */
  report: ExpenseReportDetail;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `report` | `ExpenseReportDetail` | Yes | レポート詳細データ | useReport Hook の data |

### ReportBasicInfo

- 配置: `pages/reports/ReportBasicInfo.tsx`
- 責務: レポートの常時表示項目（タイトル・ステータスバッジ・対象期間・合計金額・作成者・作成日）を表示する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;3（常時表示項目 #1-#6）

```typescript
interface ReportBasicInfoProps {
  /** レポートタイトル */
  title: string;
  /** レポートステータス */
  status: ReportStatus;
  /** 対象期間開始日 */
  periodStart: string;
  /** 対象期間終了日 */
  periodEnd: string;
  /** 合計金額 */
  totalAmount: number;
  /** 作成者名 */
  submitterName: string;
  /** 作成日 */
  createdAt: string;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `title` | `string` | Yes | レポートタイトル | report.title |
| `status` | `ReportStatus` | Yes | レポートステータス | report.status |
| `periodStart` | `string` | Yes | 対象期間開始日 | report.period_start |
| `periodEnd` | `string` | Yes | 対象期間終了日 | report.period_end |
| `totalAmount` | `number` | Yes | 合計金額 | report.total_amount |
| `submitterName` | `string` | Yes | 作成者名 | report.submitter.name |
| `createdAt` | `string` | Yes | 作成日 | report.created_at |

### ReportWorkflowInfo

- 配置: `pages/reports/ReportWorkflowInfo.tsx`
- 責務: レポートの状態遷移に応じた条件付き表示項目（提出日・承認者名・承認日・承認コメント・却下者名・却下日・却下理由・支払処理者名・支払完了日）を表示する。却下理由は赤色の背景付きで目立たせる
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;3（条件付き表示項目 #8-#16）

```typescript
interface ReportWorkflowInfoProps {
  /** レポートステータス */
  status: ReportStatus;
  /** 提出日（submitted 以降の状態で表示） */
  submittedAt: string | null;
  /** 承認者名（approved / paid 状態で表示） */
  approverName: string | null;
  /** 承認日（approved / paid 状態で表示） */
  approvedAt: string | null;
  /** 承認コメント（approved / paid 状態かつ存在する場合に表示） */
  approvalComment: string | null;
  /** 却下者名（rejected 状態で表示） */
  rejectorName: string | null;
  /** 却下日（rejected 状態で表示） */
  rejectedAt: string | null;
  /** 却下理由（rejected 状態で表示、赤色背景） */
  rejectionReason: string | null;
  /** 支払処理者名（paid 状態で表示） */
  paidByName: string | null;
  /** 支払完了日（paid 状態で表示） */
  paidAt: string | null;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `status` | `ReportStatus` | Yes | 表示切り替え用ステータス | report.status |
| `submittedAt` | `string \| null` | Yes | 提出日 | report.submitted_at |
| `approverName` | `string \| null` | Yes | 承認者名 | report.approved_by?.name |
| `approvedAt` | `string \| null` | Yes | 承認日 | report.approved_at |
| `approvalComment` | `string \| null` | Yes | 承認コメント | report.approval_comment |
| `rejectorName` | `string \| null` | Yes | 却下者名 | report.rejected_by?.name |
| `rejectedAt` | `string \| null` | Yes | 却下日 | report.rejected_at |
| `rejectionReason` | `string \| null` | Yes | 却下理由 | report.rejection_reason |
| `paidByName` | `string \| null` | Yes | 支払処理者名 | report.paid_by?.name |
| `paidAt` | `string \| null` | Yes | 支払完了日 | report.paid_at |

### ReportReferenceLink

- 配置: `pages/reports/ReportReferenceLink.tsx`
- 責務: 再申請元レポートへのリンクを表示する。reference_report_id が存在しない場合は非表示
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;3（条件付き表示項目 #7）

```typescript
interface ReportReferenceLinkProps {
  /** 再申請元レポート ID（null の場合は非表示） */
  referenceReportId: string | null;
  /** 再申請元レポートのタイトル */
  referenceReportTitle: string | null;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `referenceReportId` | `string \| null` | Yes | 再申請元レポート ID | report.reference_report_id |
| `referenceReportTitle` | `string \| null` | Yes | 再申請元レポートのタイトル | ReportDetailPage が useReport(referenceReportId) で別途取得。openapi.yaml の ExpenseReportDetail には未定義のため、ページコンポーネントの責務として取得する |

### ReportActionBar

- 配置: `pages/reports/ReportActionBar.tsx`
- 責務: レポートの状態・所有権・ロールに基づいて表示すべきアクションボタンを判定し、OwnerActions と WorkflowActions に振り分ける。ボタンの表示/非表示ロジックの中核コンポーネント
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;4

```typescript
interface ReportActionBarProps {
  /** レポートステータス */
  status: ReportStatus;
  /** 現在のユーザーがレポート所有者か */
  isOwner: boolean;
  /** 現在のユーザーのロール */
  currentUserRole: 'admin' | 'approver' | 'member' | 'accounting';
  /** 明細件数（提出ボタンの活性/非活性判定に使用） */
  itemCount: number;
  /** 編集ボタン押下時のコールバック */
  onEdit: () => void;
  /** 提出確認ダイアログ表示コールバック */
  onSubmitReport: () => void;
  /** 削除確認ダイアログ表示コールバック */
  onDelete: () => void;
  /** 再申請コールバック（作成画面に遷移） */
  onResubmit: () => void;
  /** 承認確認ダイアログ表示コールバック */
  onApprove: () => void;
  /** 却下確認ダイアログ表示コールバック */
  onReject: () => void;
  /** 支払完了確認ダイアログ表示コールバック */
  onMarkAsPaid: () => void;
  /** 各ミューテーションの実行中フラグ（実行中のボタンを disabled にする） */
  pendingAction: 'submit' | 'delete' | 'approve' | 'reject' | 'pay' | null;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `status` | `ReportStatus` | Yes | レポートステータス | report.status |
| `isOwner` | `boolean` | Yes | 所有者フラグ | report.submitter.id === currentUser.id |
| `currentUserRole` | `string` | Yes | 現在のユーザーのロール | useCurrentUser の data.role |
| `itemCount` | `number` | Yes | 明細件数 | report.items.length |
| `onEdit` | `() => void` | Yes | 編集ボタンコールバック | ReportDetailPage（navigate） |
| `onSubmitReport` | `() => void` | Yes | 提出確認ダイアログ表示 | ReportDetailPage |
| `onDelete` | `() => void` | Yes | 削除確認ダイアログ表示 | ReportDetailPage |
| `onResubmit` | `() => void` | Yes | 再申請コールバック | ReportDetailPage（navigate） |
| `onApprove` | `() => void` | Yes | 承認確認ダイアログ表示 | ReportDetailPage |
| `onReject` | `() => void` | Yes | 却下確認ダイアログ表示 | ReportDetailPage |
| `onMarkAsPaid` | `() => void` | Yes | 支払完了確認ダイアログ表示 | ReportDetailPage |
| `pendingAction` | `string \| null` | Yes | 実行中アクション識別子 | ReportDetailPage |

### OwnerActions

- 配置: `pages/reports/OwnerActions.tsx`
- 責務: 所有者が操作可能なボタン群（編集・提出・削除・再申請）を表示する。提出ボタンは明細 0 件の場合に disabled + ツールチップを表示する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;4（A1-A4）

```typescript
interface OwnerActionsProps {
  /** レポートステータス */
  status: ReportStatus;
  /** 明細件数（提出ボタンの活性/非活性判定） */
  itemCount: number;
  /** 編集ボタン押下コールバック */
  onEdit: () => void;
  /** 提出ボタン押下コールバック */
  onSubmitReport: () => void;
  /** 削除ボタン押下コールバック */
  onDelete: () => void;
  /** 再申請ボタン押下コールバック */
  onResubmit: () => void;
  /** 実行中アクション識別子 */
  pendingAction: 'submit' | 'delete' | null;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `status` | `ReportStatus` | Yes | ボタン表示条件の判定 | 親 Props |
| `itemCount` | `number` | Yes | 提出ボタン活性判定 | 親 Props |
| `onEdit` | `() => void` | Yes | 編集遷移 | 親 Props |
| `onSubmitReport` | `() => void` | Yes | 提出確認ダイアログ | 親 Props |
| `onDelete` | `() => void` | Yes | 削除確認ダイアログ | 親 Props |
| `onResubmit` | `() => void` | Yes | 再申請遷移 | 親 Props |
| `pendingAction` | `'submit' \| 'delete' \| null` | Yes | 実行中アクション | 親 Props |

### WorkflowActions

- 配置: `pages/reports/WorkflowActions.tsx`
- 責務: 承認者・経理担当が操作可能なボタン群（承認・却下・支払完了）を表示する。自己承認禁止（RBC-016）・自己支払処理禁止（RBC-012）の制約上、所有者の場合はこれらのボタンを表示しない
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;4（A5-A7）

```typescript
interface WorkflowActionsProps {
  /** レポートステータス */
  status: ReportStatus;
  /** 現在のユーザーのロール */
  currentUserRole: 'admin' | 'approver' | 'member' | 'accounting';
  /** 承認ボタン押下コールバック */
  onApprove: () => void;
  /** 却下ボタン押下コールバック */
  onReject: () => void;
  /** 支払完了ボタン押下コールバック */
  onMarkAsPaid: () => void;
  /** 実行中アクション識別子 */
  pendingAction: 'approve' | 'reject' | 'pay' | null;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `status` | `ReportStatus` | Yes | ボタン表示条件の判定 | 親 Props |
| `currentUserRole` | `string` | Yes | ロール判定 | 親 Props |
| `onApprove` | `() => void` | Yes | 承認確認ダイアログ | 親 Props |
| `onReject` | `() => void` | Yes | 却下確認ダイアログ | 親 Props |
| `onMarkAsPaid` | `() => void` | Yes | 支払完了確認ダイアログ | 親 Props |
| `pendingAction` | `'approve' \| 'reject' \| 'pay' \| null` | Yes | 実行中アクション | 親 Props |

### ItemListSection

- 配置: `pages/reports/ItemListSection.tsx`
- 責務: 経費明細一覧セクションのコンテナ。セクションヘッダー（明細件数・明細追加ボタン）とテーブルを統合する。明細 0 件時は EmptyState を表示する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;5

```typescript
interface ItemListSectionProps {
  /** 明細データ配列 */
  items: ExpenseItem[];
  /** 所有者フラグ */
  isOwner: boolean;
  /** レポートステータス */
  status: ReportStatus;
  /** 明細追加ボタン押下コールバック */
  onAddItem: () => void;
  /** 明細行クリックコールバック（閲覧モードでスライドパネルを開く） */
  onItemClick: (itemId: string) => void;
  /** 明細編集ボタン押下コールバック */
  onEditItem: (itemId: string) => void;
  /** 明細削除ボタン押下コールバック */
  onDeleteItem: (itemId: string) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `items` | `ExpenseItem[]` | Yes | 明細データ | report.items |
| `isOwner` | `boolean` | Yes | 操作ボタン表示判定 | 親 Props |
| `status` | `ReportStatus` | Yes | 操作ボタン表示判定（draft のみ） | report.status |
| `onAddItem` | `() => void` | Yes | 明細追加パネル表示 | ReportDetailPage |
| `onItemClick` | `(itemId: string) => void` | Yes | 明細詳細パネル表示 | ReportDetailPage |
| `onEditItem` | `(itemId: string) => void` | Yes | 明細編集パネル表示 | ReportDetailPage |
| `onDeleteItem` | `(itemId: string) => void` | Yes | 明細削除確認ダイアログ表示 | ReportDetailPage |

### ItemListHeader

- 配置: `pages/reports/ItemListHeader.tsx`
- 責務: 明細セクションのヘッダー。「明細一覧（N件）」の見出しと「+ 明細追加」ボタンを横並びに配置する。明細追加ボタンは所有者 AND draft 状態の場合のみ表示する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;5

```typescript
interface ItemListHeaderProps {
  /** 明細件数 */
  itemCount: number;
  /** 明細追加ボタンを表示するか */
  canAddItem: boolean;
  /** 明細追加ボタン押下コールバック */
  onAddItem: () => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `itemCount` | `number` | Yes | セクションタイトルの件数表示 | items.length |
| `canAddItem` | `boolean` | Yes | 明細追加ボタンの表示制御 | isOwner AND status === 'draft' |
| `onAddItem` | `() => void` | Yes | 明細追加コールバック | 親 Props |

### ItemTable

- 配置: `pages/reports/ItemTable.tsx`
- 責務: 明細データをテーブル形式で表示する。AppDataGrid をラップし、カラム定義（日付・金額・カテゴリ・摘要・添付数・操作）を設定する。操作列（編集・削除ボタン）は所有者 AND draft 状態の場合のみ表示する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;5

```typescript
interface ItemTableProps {
  /** 明細データ配列 */
  items: ExpenseItem[];
  /** 操作ボタンを表示するか（所有者 AND draft のみ） */
  canEditItems: boolean;
  /** 行クリックコールバック */
  onItemClick: (itemId: string) => void;
  /** 編集ボタン押下コールバック */
  onEditItem: (itemId: string) => void;
  /** 削除ボタン押下コールバック */
  onDeleteItem: (itemId: string) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `items` | `ExpenseItem[]` | Yes | 明細データ | 親 Props |
| `canEditItems` | `boolean` | Yes | 操作列の表示制御 | isOwner AND status === 'draft' |
| `onItemClick` | `(itemId: string) => void` | Yes | 行クリック（閲覧パネル） | 親 Props |
| `onEditItem` | `(itemId: string) => void` | Yes | 編集ボタン | 親 Props |
| `onDeleteItem` | `(itemId: string) => void` | Yes | 削除ボタン | 親 Props |

### ItemSlidePanel

- 配置: `pages/reports/ItemSlidePanel.tsx`
- 責務: 明細の追加・編集・閲覧をスライドパネルで提供する。画面右側からスライドインし、パネルモード（追加・編集・閲覧）に応じてフォームの入力可否を制御する。パネル内に ItemForm と AttachmentArea を配置する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;6

```typescript
type PanelMode = 'add' | 'edit' | 'view';

interface ItemSlidePanelProps {
  /** パネルの開閉状態 */
  open: boolean;
  /** パネルモード */
  mode: PanelMode;
  /** レポート ID（API 呼び出しに使用） */
  reportId: string;
  /** 編集/閲覧時の明細データ（追加モードでは null） */
  item: ExpenseItem | null;
  /** レポートステータス（添付ファイル操作の表示制御に使用） */
  reportStatus: ReportStatus;
  /** 所有者フラグ（添付ファイル操作の表示制御に使用） */
  isOwner: boolean;
  /** レポートの対象期間開始日（ITM-007 期間外警告判定のため ItemForm に渡す） */
  reportPeriodStart: string;
  /** レポートの対象期間終了日（ITM-007 期間外警告判定のため ItemForm に渡す） */
  reportPeriodEnd: string;
  /** パネルを閉じるコールバック */
  onClose: () => void;
  /** 明細保存成功時のコールバック */
  onSaveSuccess: () => void;
  /** 「保存して続けて追加」成功時のコールバック */
  onSaveAndContinue: () => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `open` | `boolean` | Yes | パネル開閉状態 | ReportDetailPage |
| `mode` | `PanelMode` | Yes | パネルモード | ReportDetailPage |
| `reportId` | `string` | Yes | レポート ID | URL パスパラメータ |
| `item` | `ExpenseItem \| null` | Yes | 明細データ | report.items から選択 |
| `reportStatus` | `ReportStatus` | Yes | 添付操作の表示制御 | report.status |
| `isOwner` | `boolean` | Yes | 添付操作の表示制御 | 親 Props |
| `reportPeriodStart` | `string` | Yes | 対象期間開始日（ITM-007 警告判定で ItemForm に渡す） | report.period_start |
| `reportPeriodEnd` | `string` | Yes | 対象期間終了日（ITM-007 警告判定で ItemForm に渡す） | report.period_end |
| `onClose` | `() => void` | Yes | パネル閉じるコールバック | ReportDetailPage |
| `onSaveSuccess` | `() => void` | Yes | 保存成功コールバック | ReportDetailPage |
| `onSaveAndContinue` | `() => void` | Yes | 保存して続けて追加コールバック | ReportDetailPage |

### ItemForm

- 配置: `pages/reports/ItemForm.tsx`
- 責務: 明細の入力フォーム。React Hook Form + Zod（itemCreateSchema / itemUpdateSchema）でバリデーションを行う。閲覧モードでは全フィールドを readonly にする。追加モードでは「保存して続けて追加」ボタンも表示する。API エラー（useCreateItem / useUpdateItem の error）はフォーム上部の FormAlert コンポーネント（common-components.md §FormAlert、`components/ui/FormAlert.tsx`）で表示する。保存ボタン押下時に `expenseDate` がレポートの対象期間外である場合、ConfirmDialog で警告を表示してからユーザー確認後に `onSubmit` / `onSaveAndContinue` を呼び出す（ITM-007、`50_detail_design/screens/report-detail.md` §6「保存時の期間外警告」参照）
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;6

```typescript
interface ItemFormValues {
  /** 支出日（YYYY-MM-DD 形式） */
  expenseDate: string;
  /** 金額（円、正の整数） */
  amount: number;
  /** カテゴリ ID */
  categoryId: string;
  /** 摘要 */
  description: string;
}

interface ItemFormProps {
  /** パネルモード */
  mode: PanelMode;
  /** フォーム送信コールバック */
  onSubmit: (data: ItemFormValues) => void;
  /** 「保存して続けて追加」コールバック（追加モードのみ） */
  onSaveAndContinue?: (data: ItemFormValues) => void;
  /** キャンセルコールバック */
  onCancel: () => void;
  /** カテゴリ一覧（ドロップダウン選択肢） */
  categories: Array<{ value: string; label: string }>;
  /** API エラーメッセージ */
  apiError: string | null;
  /** 送信中フラグ */
  isPending: boolean;
  /** 編集/閲覧時の初期値 */
  defaultValues?: ItemFormValues;
  /** レポートの対象期間開始日（ITM-007 期間外警告判定に使用、YYYY-MM-DD） */
  reportPeriodStart: string;
  /** レポートの対象期間終了日（ITM-007 期間外警告判定に使用、YYYY-MM-DD） */
  reportPeriodEnd: string;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `mode` | `PanelMode` | Yes | 閲覧モードで readonly 制御 | ItemSlidePanel |
| `onSubmit` | `(data: ItemFormValues) => void` | Yes | フォーム送信 | ItemSlidePanel |
| `onSaveAndContinue` | `(data: ItemFormValues) => void` | No | 保存して続けて追加（追加モードのみ） | ItemSlidePanel |
| `onCancel` | `() => void` | Yes | キャンセル | ItemSlidePanel |
| `categories` | `Array<{ value: string; label: string }>` | Yes | カテゴリ選択肢 | useCategories Hook |
| `apiError` | `string \| null` | Yes | API エラーメッセージ | useCreateItem / useUpdateItem の error |
| `isPending` | `boolean` | Yes | 送信中フラグ | useCreateItem / useUpdateItem の isPending |
| `defaultValues` | `ItemFormValues` | No | 編集/閲覧時の初期値 | 親 Props |
| `reportPeriodStart` | `string` | Yes | レポート対象期間の開始日。保存時の期間外判定（ITM-007）に使用 | ReportDetailPage → ItemSlidePanel 経由で report.period_start |
| `reportPeriodEnd` | `string` | Yes | レポート対象期間の終了日。保存時の期間外判定（ITM-007）に使用 | ReportDetailPage → ItemSlidePanel 経由で report.period_end |

### AttachmentArea

- 配置: `pages/reports/AttachmentArea.tsx`
- 責務: 明細スライドパネル内の添付ファイル管理領域。添付ファイル一覧（AttachmentList）とアップロード機能（AttachmentUploader）を統合する orchestration 層。一覧取得（`useAttachments`）、アップロード（`useUploadAttachment`）、削除（`useDeleteAttachment`）、削除確認ダイアログの制御、トースト表示（成功・失敗）を担当する。プレビュー / ダウンロードの署名付き URL 取得と `window.open` 操作は AttachmentList 内部の AttachmentItemRow に委譲し、発生したエラー通知を `onError` コールバックで受け取ってトースト表示する。操作可否は所有者 AND draft 状態で判定する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;7

```typescript
interface AttachmentAreaProps {
  /** レポート ID */
  reportId: string;
  /** 明細 ID（明細保存後に設定される。未保存の追加モードでは null） */
  itemId: string | null;
  /** アップロード・削除操作が可能か */
  canModify: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reportId` | `string` | Yes | レポート ID | 親 Props |
| `itemId` | `string \| null` | Yes | 明細 ID | 親 Props |
| `canModify` | `boolean` | Yes | アップロード・削除操作の表示制御 | isOwner AND status === 'draft' |

### AttachmentList

- 配置: `pages/reports/AttachmentList.tsx`
- 責務: 添付ファイル一覧のレンダリングと、**添付 1 件ごとの署名付き URL 取得 Hook（`useAttachmentDownloadUrl` / `useAttachmentPreviewUrl`）のオーケストレーション**を担う層。内部で行単位の描画コンポーネント `AttachmentItemRow` を使用し、`AttachmentItemRow` が per-item で Hook を保持する。プレビューはクリック同期 `window.open('about:blank')` → `refetch` → `newWindow.location.href` 差し替えパターン、ダウンロードは `refetch` → 動的生成 `<a download>` 要素のクリック → 要素除去パターン（`50_detail_design/files.md` §4.5）を実装する。各行にはファイル名（クリックでプレビュー）、↓ アイコン（クリックでダウンロード）、ファイルサイズ、削除ボタンを表示する。エラー通知は親（AttachmentArea）にコールバックで委譲する
- 構造に関する補足: React hooks rules（条件分岐・ループ内での Hook 呼び出し禁止）を遵守しつつ、添付ごとに独立した署名付き URL 取得を実現するため、行単位に内部コンポーネント `AttachmentItemRow` を切り出し、そのトップレベルで Hook を呼ぶ構造とする。結果として `AttachmentList` は純粋な presentational ではなく **per-item rendering + per-item hook orchestration 層**となる
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;7、`50_detail_design/files.md` &sect;4.5

```typescript
interface AttachmentListProps {
  /** レポート ID（AttachmentItemRow の per-item Hook に渡す） */
  reportId: string;
  /** 明細 ID（AttachmentItemRow の per-item Hook に渡す） */
  itemId: string;
  /** 添付ファイルデータ配列 */
  attachments: Attachment[];
  /** 削除ボタンを表示するか */
  canDelete: boolean;
  /** 削除ボタン押下コールバック */
  onDelete: (attachmentId: string) => void;
  /** 削除処理中の添付 ID（グレーアウト対象） */
  deletingId: string | null;
  /** プレビュー・ダウンロード失敗時のエラー通知コールバック（親 AttachmentArea がトースト表示） */
  onError: (message: string) => void;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reportId` | `string` | Yes | レポート ID（per-item Hook に渡す） | AttachmentArea |
| `itemId` | `string` | Yes | 明細 ID（per-item Hook に渡す） | AttachmentArea |
| `attachments` | `Attachment[]` | Yes | 添付ファイルデータ | useAttachments Hook の data |
| `canDelete` | `boolean` | Yes | 削除ボタンの表示制御 | canModify |
| `onDelete` | `(attachmentId: string) => void` | Yes | 削除コールバック | AttachmentArea |
| `deletingId` | `string \| null` | Yes | 削除処理中の添付 ID | AttachmentArea |
| `onError` | `(message: string) => void` | Yes | エラー通知コールバック（トースト表示を親に委譲） | AttachmentArea |

#### AttachmentItemRow（AttachmentList 内部コンポーネント）

- 配置: `pages/reports/AttachmentList.tsx` 内（export しない内部コンポーネント）
- 責務: 添付 1 件分の行を描画し、クリック時に `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` の `refetch()` を起動する。プレビューはファイル名クリックで `files.md` §4.5 の「クリック同期で `window.open('about:blank')` → 非同期に URL を取得 → `newWindow.location.href` に差し替え、失敗時は `newWindow.close()` して `onError` で親に通知」パターンをこのコンポーネント内に実装する。ダウンロードは ↓ アイコンクリックで `files.md` §4.5 の「`refetch()` 成功後に動的生成した `<a>` 要素（`href` に署名付き URL、`download` 属性にファイル名）を `document.body.appendChild` → `click()` → `removeChild` の順で操作してダウンロードを起動する。失敗時は `onError` で親に通知（DOM 操作なし）」パターンを実装する
- 構造上の位置付け: hooks rules 遵守のため、`attachments.map(...)` でループ内に Hook を書けない制約から分離された per-item Hook ホルダー。プレビュー / ダウンロードの `onPreview` / `onDownload` 外部コールバックは持たない（オーケストレーションを内包するため不要）
- 主な Props（内部インターフェース）: `reportId`, `itemId`, `attachment`, `canDelete`, `deletingId`, `onDelete`, `onError`

### AttachmentUploader

- 配置: `pages/reports/AttachmentUploader.tsx`
- 責務: 「+ ファイルを追加」ボタンとドラッグ&ドロップによるファイルアップロード機能を提供する。ファイル形式（JPEG, PNG, PDF）とサイズ（5MB）のクライアントサイドバリデーションを行う。アップロード中はプログレスバーを表示する
- 対応セクション: `50_detail_design/screens/report-detail.md` &sect;7

```typescript
interface AttachmentUploaderProps {
  /** レポート ID */
  reportId: string;
  /** 明細 ID */
  itemId: string;
  /** アップロード成功時のコールバック */
  onUploadSuccess: () => void;
  /** アップロード中フラグ */
  isUploading: boolean;
}
```

| Props | 型 | 必須 | 説明 | データソース |
|-------|---|------|------|------------|
| `reportId` | `string` | Yes | レポート ID | 親 Props |
| `itemId` | `string` | Yes | 明細 ID | 親 Props |
| `onUploadSuccess` | `() => void` | Yes | アップロード成功コールバック | AttachmentArea |
| `isUploading` | `boolean` | Yes | アップロード中フラグ | useUploadAttachment の isPending |

---

## 4. データフロー

```
GET /api/reports/:id
  → useReport(reportId)（← state-management.md §3 データフェッチ系）
    → ReportDetailPage
      → ReportInfoCard (props: report)
        → ReportBasicInfo (props: title, status, period, totalAmount, submitter, createdAt)
          → StatusChip (props: status)
        → ReportWorkflowInfo (props: status, submitted/approved/rejected/paid 情報)
        → ReportReferenceLink (props: referenceReportId, referenceReportTitle)
      → ReportActionBar (props: status, isOwner, currentUserRole, itemCount, アクションコールバック群)
        → OwnerActions (props: status, itemCount, onEdit, onSubmitReport, onDelete, onResubmit)
        → WorkflowActions (props: status, currentUserRole, onApprove, onReject, onMarkAsPaid)
      → ItemListSection (props: items, isOwner, status, 明細操作コールバック群)
        → ItemListHeader (props: itemCount, canAddItem, onAddItem)
        → ItemTable (props: items, canEditItems, onItemClick, onEditItem, onDeleteItem)
          → AppDataGrid
        → EmptyState (props: message, action) [明細0件時]
      → ItemSlidePanel (props: open, mode, reportId, item, reportStatus, isOwner, reportPeriodStart, reportPeriodEnd, パネル操作コールバック群)
        → ItemForm (props: mode, onSubmit, categories, apiError, isPending, defaultValues, reportPeriodStart, reportPeriodEnd)
        → ConfirmDialog（ITM-007 期間外警告、ItemForm 内部で制御）
        → AttachmentArea (props: reportId, itemId, canModify)
          → AttachmentList (props: reportId, itemId, attachments, canDelete, onDelete, deletingId, onError)
            → AttachmentItemRow × N (per-item で useAttachmentDownloadUrl / useAttachmentPreviewUrl を呼び、クリック同期 window.open パターンを実行)
          → AttachmentUploader (props: reportId, itemId, onUploadSuccess)
      → ConfirmDialog (props: open, title, message, confirmLabel, ...)

GET /api/categories
  → useCategories()（← state-management.md §3 データフェッチ系）
    → ItemForm (props: categories)
```

### ワークフロー操作のデータフロー

```
[提出]
ReportDetailPage → ConfirmDialog（確認） → useSubmitReport.mutate({ id, updated_at })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', id], ['reports', 'mine'], ['dashboard'], ['workflow', 'pending'])
         → ReportDetailPage が最新データで再描画 + AppToast で成功通知
  → 失敗: AppToast でエラー表示（409 Conflict / 422 InvalidStateTransition 等）

[削除]
ReportDetailPage → ConfirmDialog（確認） → useDeleteReport.mutate(id)
  → 成功: queryClient.invalidateQueries(['reports', 'mine'], ['dashboard'])
         → navigate('/reports')（一覧にリダイレクト）+ AppToast で成功通知
  → 失敗: AppToast でエラー表示

[承認]
ReportDetailPage → ConfirmDialog（確認、任意コメント入力） → useApproveReport.mutate({ id, comment?, updated_at })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', id], ['workflow', 'pending'], ['workflow', 'payable'], ['dashboard'])
         → ReportDetailPage が最新データで再描画 + AppToast で成功通知
  → 失敗: AppToast でエラー表示（403 SelfApprovalNotAllowed / 409 / 422 等）

[却下]
ReportDetailPage → ConfirmDialog（確認、必須理由入力） → useRejectReport.mutate({ id, reason, updated_at })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', id], ['workflow', 'pending'], ['dashboard'])
         → ReportDetailPage が最新データで再描画 + AppToast で成功通知
  → 失敗: AppToast でエラー表示（403 SelfApprovalNotAllowed / 422 MissingRejectionReason 等）

[支払完了]
ReportDetailPage → ConfirmDialog（確認） → useMarkAsPaid.mutate({ id, updated_at })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', id], ['workflow', 'payable'], ['dashboard'], ['reports', 'all'])
         → ReportDetailPage が最新データで再描画 + AppToast で成功通知
  → 失敗: AppToast でエラー表示（403 SelfPaymentNotAllowed / 409 / 422 等）
```

### 明細 CRUD のデータフロー

```
[明細追加]
ReportDetailPage → ItemSlidePanel (mode='add') → ItemForm → useCreateItem.mutate({ reportId, ...formData })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', reportId])
         → onSaveSuccess: パネルクローズ + AppToast で成功通知
         → onSaveAndContinue: フォームリセット（パネルは開いたまま）+ AppToast で成功通知
  → 失敗: FormAlert で apiError を表示（ItemForm 内フォーム上部）

[明細編集]
ReportDetailPage → ItemSlidePanel (mode='edit') → ItemForm → useUpdateItem.mutate({ reportId, itemId, ...formData })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', reportId])
         → onSaveSuccess: パネルクローズ + AppToast で成功通知
  → 失敗: FormAlert で apiError を表示（ItemForm 内フォーム上部）

[明細削除]
ReportDetailPage → ConfirmDialog（確認） → useDeleteItem.mutate({ reportId, itemId })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', reportId])
         → AppToast で成功通知
  → 失敗: AppToast でエラー表示
```

### 添付ファイル操作のデータフロー

AttachmentArea は一覧取得・アップロード・削除・トースト管理を担当し、プレビューのクリック同期 `window.open` パターンとダウンロードの動的 `<a download>` 要素パターン（`50_detail_design/files.md` §4.5）は `AttachmentList` 内部の `AttachmentItemRow` が per-item で実行する。これは `attachments.map(...)` のループ内では `useAttachmentDownloadUrl` / `useAttachmentPreviewUrl` を呼べない（hooks rules 違反となる）ため、添付 1 件ごとにコンポーネントを分割してトップレベルで Hook を呼ぶ構造としたためである。エラーはトースト表示の一元管理のため `AttachmentItemRow.onError → AttachmentList.onError → AttachmentArea` の順に委譲する。

```
[添付一覧取得]
GET /api/reports/:id/items/:itemId/attachments
  → useAttachments({ reportId, itemId })（← state-management.md §3 データフェッチ系）
    → AttachmentArea
      → AttachmentList (props: reportId, itemId, attachments, canDelete, onDelete, deletingId, onError)
        → AttachmentItemRow × N

[添付プレビュー]
AttachmentItemRow（ファイル名クリック）
  → window.open('about:blank') をクリック同期で実行
  → useAttachmentPreviewUrl.refetch()
    → 成功: 署名付き URL（Content-Disposition: inline）を取得 → newWindow.location.href = url
    → 失敗: newWindow.close() + onError(message) → AttachmentList → AttachmentArea がエラートースト表示

[添付ダウンロード]
AttachmentItemRow（↓ アイコンクリック）
  → useAttachmentDownloadUrl.refetch()
    → 成功: 署名付き URL（Content-Disposition: attachment）を取得
           → document.createElement('a') で <a> 要素生成（href = url, download = file_name）
           → document.body.appendChild → link.click() → document.body.removeChild（同期的に連続実行）
           → タブを開かずブラウザのダウンロードが起動
    → 失敗: onError(message) → AttachmentList → AttachmentArea がエラートースト表示

[添付アップロード]
AttachmentArea → AttachmentUploader → useUploadAttachment.mutate({ reportId, itemId, file })
  → 成功: queryClient.invalidateQueries(['reports', 'detail', reportId])
         → onUploadSuccess: 添付一覧を再取得 + AppToast で成功通知
  → 失敗: AppToast でエラー表示（422 InvalidFileType / 413 FileTooLarge 等）

[添付削除]
AttachmentItemRow（削除ボタンクリック） → AttachmentList（onDelete 経由） → AttachmentArea
  → useDeleteAttachment.mutate({ reportId, itemId, attId })
    → 成功: queryClient.invalidateQueries(['reports', 'detail', reportId])
           → 添付一覧を再取得 + AppToast で成功通知
    → 失敗: AppToast でエラー表示
```

### 使用する Hook

| Hook | 用途 | 定義元 |
|------|------|-------|
| `useReport` | レポート詳細データの取得 | `state-management.md §3 データフェッチ系` |
| `useCategories` | カテゴリマスタデータの取得（明細フォームのドロップダウン） | `state-management.md §3 データフェッチ系` |
| `useCurrentUser` | 現在のユーザー情報（所有権判定・ロール判定） | `state-management.md §3 認証系` |
| `useSubmitReport` | レポート提出のミューテーション | `state-management.md §3 ミューテーション系` |
| `useDeleteReport` | レポート削除のミューテーション | `state-management.md §3 ミューテーション系` |
| `useApproveReport` | レポート承認のミューテーション | `state-management.md §3 ミューテーション系` |
| `useRejectReport` | レポート却下のミューテーション | `state-management.md §3 ミューテーション系` |
| `useMarkAsPaid` | レポート支払完了のミューテーション | `state-management.md §3 ミューテーション系` |
| `useCreateItem` | 明細追加のミューテーション | `state-management.md §3 ミューテーション系` |
| `useUpdateItem` | 明細編集のミューテーション | `state-management.md §3 ミューテーション系` |
| `useDeleteItem` | 明細削除のミューテーション | `state-management.md §3 ミューテーション系` |
| `useAttachments` | 添付ファイル一覧の取得 | `state-management.md §3 データフェッチ系` |
| `useAttachmentDownloadUrl` | 添付ファイルのダウンロード用署名付き URL 取得 | `state-management.md §3 データフェッチ系` |
| `useAttachmentPreviewUrl` | 添付ファイルのプレビュー用署名付き URL 取得 | `state-management.md §3 データフェッチ系` |
| `useUploadAttachment` | 添付ファイルのアップロード | `state-management.md §3 ミューテーション系` |
| `useDeleteAttachment` | 添付ファイルの削除 | `state-management.md §3 ミューテーション系` |

---

## 5. 状態管理

| 状態 | カテゴリ | 管理方法 | スコープ |
|------|---------|---------|---------|
| レポート詳細データ | サーバー | useReport Hook（TanStack Query useQuery） | ReportDetailPage |
| カテゴリマスタデータ | サーバー | useCategories Hook（TanStack Query useQuery、staleTime: Infinity） | ItemSlidePanel |
| 添付ファイル一覧 | サーバー | useAttachments Hook（TanStack Query useQuery） | AttachmentArea |
| 各ミューテーション状態 | サーバー | 各 useMutation Hook（isPending, error） | ReportDetailPage |
| 確認ダイアログの開閉状態・種別 | UI | useState | ReportDetailPage |
| スライドパネルの開閉状態 | UI | useState | ReportDetailPage |
| スライドパネルのモード（add / edit / view） | UI | useState | ReportDetailPage |
| 選択中の明細 ID | UI | useState | ReportDetailPage |
| 実行中アクション識別子（pendingAction） | UI | 各 useMutation の isPending から導出 | ReportDetailPage |
| 楽観的ロック用 updated_at | UI | useState（useReport で取得した updated_at を保持） | ReportDetailPage |
| 明細フォーム入力値 | フォーム | React Hook Form + Zod（itemCreateSchema / itemUpdateSchema） | ItemForm 内 |
| 削除処理中の添付 ID | UI | useState | AttachmentArea |

---

## 6. 共通コンポーネント使用箇所

| 共通コンポーネント | 使用箇所 | カスタマイズ |
|------------------|---------|------------|
| `AppLayout`（← common-components.md） | ReportDetailPage のルートラッパー | children にメインコンテンツを配置 |
| `AppBreadcrumbs`（← common-components.md） | メインコンテンツ上部 | items: [{ label: 'マイレポート', href: '/reports' }, { label: レポートタイトル }] |
| `StatusChip`（← common-components.md） | ReportBasicInfo 内のステータスバッジ | status: レポートの status 値 |
| `AppDataGrid`（← common-components.md） | ItemTable 内の明細テーブル | columns: 日付・金額・カテゴリ・摘要・添付数・操作 |
| `EmptyState`（← common-components.md） | ItemListSection 内の明細 0 件時 | message: 所有者 AND draft の場合「明細はまだ追加されていません。「明細追加」から経費を登録してください。」、それ以外「明細はまだ追加されていません。」 |
| `EmptyState`（← common-components.md） | ReportDetailPage 内の 404 Not Found 時 | message: 「指定されたデータが見つかりません。」、action: { label: 'レポート一覧に戻る', onClick: navigate('/reports') }。他コンテンツ（ReportInfoCard 等）は非表示 |
| `FormAlert`（← common-components.md） | ItemForm 内の API エラー表示 | message: apiError（useCreateItem / useUpdateItem の error）、severity: 'error'。フォーム上部に配置 |
| `ConfirmDialog`（← common-components.md） | レポートの提出・削除・承認・却下・支払完了の各確認ダイアログ | open, title, message, confirmLabel, confirmColor, inputField（却下時: 必須テキストエリア、承認時: 任意テキストエリア） |
| `ConfirmDialog`（← common-components.md） | ItemForm 保存時の期間外警告（ITM-007） | title: 「入力内容の確認」、message: `WARNING_MESSAGES.ITEM_DATE_OUTSIDE_PERIOD_WARNING`、confirmLabel: 「保存する」、confirmColor: primary、inputField なし |
| `AppTextField`（← common-components.md） | ItemForm 内の金額・摘要入力 | 金額: type="number"、摘要: multiline |
| `AppSelect`（← common-components.md） | ItemForm 内のカテゴリ選択 | options: useCategories から取得した 6 カテゴリ |
| `AppDatePicker`（← common-components.md） | ItemForm 内の日付入力 | label: 日付、required: true |
| `PageSkeleton`（← common-components.md） | 初回読み込み時のスケルトン UI | variant: 'card' |
| `AppToast`（← common-components.md） | 操作成功・エラー時のトースト通知 | 成功: severity 'success'、エラー: severity 'error' |

---

## 7. 表示条件・認可

| コンポーネント | 表示条件 | 操作可否条件 | 根拠 |
|--------------|---------|-------------|------|
| `ReportDetailPage` | 認証済みユーザー。閲覧範囲はロール別: Member は自分のみ、Approver は submitted + 自分が承認/却下したレポート、Accounting は approved + 自分のレポート、Admin は全レポート閲覧可 | - | `authz.md` &sect;6.3, `screens/report-detail.md` &sect;8 |
| `EmptyState`（404 Not Found） | useReport が 404 を返した場合に表示。メインコンテンツ領域に「指定されたデータが見つかりません。」メッセージとレポート一覧へのリンクを表示する。他のコンテンツ（ReportInfoCard, ReportActionBar, ItemListSection 等）は非表示 | - | `screens/report-detail.md` &sect;11（404 Not Found） |
| `OwnerActions`（編集 A1） | 所有者 AND status === draft | 常時有効 | `screens/report-detail.md` &sect;4, `state_machine.md` &sect;5.1 |
| `OwnerActions`（提出 A2） | 所有者 AND status === draft | 明細 1 件以上で有効、0 件で disabled + ツールチップ | `screens/report-detail.md` &sect;4, `state_machine.md` T1 |
| `OwnerActions`（削除 A3） | 所有者 AND status === draft | 常時有効 | `screens/report-detail.md` &sect;4, `state_machine.md` T5 |
| `OwnerActions`（再申請 A4） | 所有者 AND status === rejected | 常時有効 | `screens/report-detail.md` &sect;4, `state_machine.md` &sect;6 |
| `WorkflowActions`（承認 A5） | Approver AND status === submitted AND 非所有者 | 常時有効 | `screens/report-detail.md` &sect;4, `authz.md` &sect;6.6（RBC-016） |
| `WorkflowActions`（却下 A6） | Approver AND status === submitted AND 非所有者 | 常時有効 | `screens/report-detail.md` &sect;4, `authz.md` &sect;6.6（RBC-016） |
| `WorkflowActions`（支払完了 A7） | Accounting AND status === approved AND 非所有者 | 常時有効 | `screens/report-detail.md` &sect;4, `authz.md` &sect;6.6（RBC-012） |
| `ItemListHeader`（明細追加） | 所有者 AND status === draft | 常時有効 | `screens/report-detail.md` &sect;5 |
| `ItemTable`（編集・削除列） | 所有者 AND status === draft | 常時有効 | `screens/report-detail.md` &sect;5 |
| `AttachmentUploader` | 所有者 AND status === draft | 常時有効 | `screens/report-detail.md` &sect;7 |
| `AttachmentList`（削除ボタン） | 所有者 AND status === draft | 常時有効 | `screens/report-detail.md` &sect;7 |
| `AttachmentList`（プレビュー） | 全ロール、全状態 | 常時有効 | `screens/report-detail.md` &sect;7, `authz.md` &sect;6.5 |
| `AttachmentList`（ダウンロード） | 全ロール、全状態 | 常時有効 | `screens/report-detail.md` &sect;7, `authz.md` &sect;6.5 |

---

## 8. 画面詳細仕様との対応

| 50_detail_design セクション | 対応コンポーネント | 備考 |
|---------------------------|------------------|------|
| &sect;1 基本情報 | ReportDetailPage | URL パス、対応 API 群 |
| &sect;2 レイアウト | AppLayout, AppBreadcrumbs, ReportInfoCard, ReportActionBar, ItemListSection, ItemSlidePanel | 全体構成 |
| &sect;3 レポート基本情報カード | ReportInfoCard, ReportBasicInfo, ReportWorkflowInfo, ReportReferenceLink | 常時表示項目 + 条件付き表示項目 |
| &sect;4 アクションボタンエリア | ReportActionBar, OwnerActions, WorkflowActions, ConfirmDialog | ボタン表示条件マトリクス（状態 x ロール x 所有権）、確認ダイアログ（D1-D5） |
| &sect;5 経費明細一覧セクション | ItemListSection, ItemListHeader, ItemTable, EmptyState | 明細テーブル、明細操作（追加・編集・削除）、空状態 |
| &sect;6 明細スライドパネル | ItemSlidePanel, ItemForm | パネルモード（追加・編集・閲覧）、入力項目（V1-V7）、保存して続けて追加 |
| &sect;7 添付ファイル管理 | AttachmentArea, AttachmentList, AttachmentUploader | 一覧・アップロード・ダウンロード・削除 |
| &sect;8 アクセス制御 | ReportDetailPage | 存在チェック、テナント境界、ロール別閲覧範囲 |
| &sect;9 ローディング | PageSkeleton, OwnerActions/WorkflowActions（pendingAction）, ItemForm, AttachmentUploader | スケルトン、ボタン disabled + スピナー、プログレスバー |
| &sect;10 ロール別表示差異 | ReportActionBar, ItemListSection | アクションボタン・明細操作の表示マトリクス |
| &sect;11 エラーハンドリング | AppToast, FormAlert, EmptyState（404） | 500/403/401/409/422: AppToast、404: EmptyState（メインコンテンツに not found メッセージ + 一覧リンク）、明細フォーム API エラー: FormAlert |
| &sect;12 画面間遷移まとめ | ReportDetailPage | 編集・再申請・削除後の遷移 |
| &sect;14 処理シーケンス | 各 Hook | 提出・承認・却下・支払完了・明細追加のシーケンス |

---

## 9. テスト追跡用 設計識別子

**コンポーネント参照形式**: `55_ui_component/screens/report-detail.md §{ComponentName}`

| 識別子 | 種別 | テスト対象の概要 |
|--------|------|----------------|
| `55_ui_component/screens/report-detail.md §ReportDetailPage` | コンポーネント単体テスト | データ取得、アクセス制御判定、確認ダイアログ制御、スライドパネル開閉制御、各操作成功時の画面更新 |
| `55_ui_component/screens/report-detail.md §ReportInfoCard` | コンポーネント単体テスト | レポート基本情報の描画 |
| `55_ui_component/screens/report-detail.md §ReportBasicInfo` | コンポーネント単体テスト | タイトル・ステータスバッジ・対象期間・合計金額・作成者・作成日の描画 |
| `55_ui_component/screens/report-detail.md §ReportWorkflowInfo` | コンポーネント単体テスト | 状態別の条件付き表示（提出日、承認情報、却下情報、支払情報）、却下理由の赤色背景表示 |
| `55_ui_component/screens/report-detail.md §ReportReferenceLink` | コンポーネント単体テスト | 再申請元リンクの表示/非表示、リンク先の正確性 |
| `55_ui_component/screens/report-detail.md §ReportActionBar` | コンポーネント単体テスト | 状態 x ロール x 所有権の全組み合わせでのボタン表示/非表示 |
| `55_ui_component/screens/report-detail.md §OwnerActions` | コンポーネント単体テスト | 編集・提出・削除・再申請ボタンの表示条件、提出ボタンの disabled（明細 0 件）+ ツールチップ |
| `55_ui_component/screens/report-detail.md §WorkflowActions` | コンポーネント単体テスト | 承認・却下・支払完了ボタンの表示条件（ロール + 状態 + 非所有者） |
| `55_ui_component/screens/report-detail.md §ItemListSection` | コンポーネント単体テスト | 明細一覧セクション全体の描画、空状態表示 |
| `55_ui_component/screens/report-detail.md §ItemListHeader` | コンポーネント単体テスト | 明細件数表示、明細追加ボタンの表示/非表示 |
| `55_ui_component/screens/report-detail.md §ItemTable` | コンポーネント単体テスト | 明細テーブルのカラム描画、金額フォーマット、操作列の表示/非表示、行クリック |
| `55_ui_component/screens/report-detail.md §ItemSlidePanel` | コンポーネント単体テスト | パネル開閉、モード切替（追加・編集・閲覧）、閲覧モードの readonly 制御 |
| `55_ui_component/screens/report-detail.md §ItemForm` | コンポーネント単体テスト | フォーム入力・バリデーション（V1-V7）・送信・エラー表示・「保存して続けて追加」ボタン |
| `55_ui_component/screens/report-detail.md §AttachmentArea` | コンポーネント単体テスト | 添付ファイル領域全体の描画、操作可否の表示制御 |
| `55_ui_component/screens/report-detail.md §AttachmentList` | コンポーネント単体テスト | 添付ファイル一覧の描画、ダウンロードリンク、削除ボタンの表示/非表示 |
| `55_ui_component/screens/report-detail.md §AttachmentUploader` | コンポーネント単体テスト | ファイル選択・ドラッグ&ドロップ、形式/サイズバリデーション、プログレスバー表示 |
| `55_ui_component/state-management.md §useReport` | Hook 単体テスト | API 呼び出し・レポート詳細データ取得 |
| `55_ui_component/state-management.md §useSubmitReport` | Hook 単体テスト | 提出 API 呼び出し・updated_at 付き・キャッシュ無効化 |
| `55_ui_component/state-management.md §useDeleteReport` | Hook 単体テスト | 削除 API 呼び出し・キャッシュ無効化 |
| `55_ui_component/state-management.md §useApproveReport` | Hook 単体テスト | 承認 API 呼び出し・コメント付き・キャッシュ無効化 |
| `55_ui_component/state-management.md §useRejectReport` | Hook 単体テスト | 却下 API 呼び出し・理由付き・キャッシュ無効化 |
| `55_ui_component/state-management.md §useMarkAsPaid` | Hook 単体テスト | 支払完了 API 呼び出し・キャッシュ無効化 |
| `55_ui_component/state-management.md §useCreateItem` | Hook 単体テスト | 明細追加 API 呼び出し・キャッシュ無効化 |
| `55_ui_component/state-management.md §useUpdateItem` | Hook 単体テスト | 明細編集 API 呼び出し・キャッシュ無効化 |
| `55_ui_component/state-management.md §useDeleteItem` | Hook 単体テスト | 明細削除 API 呼び出し・キャッシュ無効化 |
