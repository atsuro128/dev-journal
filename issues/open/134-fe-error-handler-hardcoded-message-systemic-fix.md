# FE エラーハンドラのハードコード文言を err.message ベースに統一し、原因究明と再発防止を行う

## 発見日
2026-04-21

## カテゴリ
implementation / project-management

## 影響度
中〜高（サーバーエラー種別がユーザーに伝わらず UX 退化。FE 全体に系統的に散在しており、単なる局所修正では再発する）

## 発見経緯
user-report（Step 11-A SMK-036 実施中、`AttachmentUploader` の `onError` ハンドラがマッピング済み `err.message` を上書きしているバグを発見。調査の結果、同パターンが FE 全体に系統的に散在していることが判明）

## 関連ステップ
Step 8-7（共通 UI / エラーハンドリング基盤）/ Step 10 全般（機能実装）/ Step 11-A

## ブロッカー
なし（Step 11-A のブロッカーではないが、ポートフォリオ品質・UX 品質の観点で Step 11-D（横断レビュー）前に解消したい）

---

## 1. 問題

### 1.1 既存アーキテクチャ（設計意図）

本プロジェクトの FE は **API クライアント層でエラーコード → 日本語メッセージのマッピング** を持っている:

```ts
// frontend/src/lib/error-messages.ts
export const SERVER_ERROR_MESSAGES = {
  INVALID_FILE_TYPE: 'JPEG, PNG, PDF のみアップロード可能です',
  FILE_TOO_LARGE: 'ファイルサイズは5MB以下にしてください',  // ※#131 統合時に文言調整
  CONFLICT: 'このレポートは他のユーザーによって更新されました。ページを再読み込みしてください。',
  FORBIDDEN: 'この操作を行う権限がありません。',
  RESOURCE_NOT_FOUND: '指定されたデータが見つかりません。',
  INTERNAL_ERROR: 'サーバーとの通信に失敗しました。しばらくしてから再度お試しください。',
  // ...
};
```

```ts
// frontend/src/api/client.ts:144-149
const message = code === 'VALIDATION_ERROR'
  ? (serverMessage ?? SERVER_ERROR_MESSAGES.VALIDATION_ERROR)
  : ((SERVER_ERROR_MESSAGES as Record<string, string>)[code] ?? SERVER_ERROR_MESSAGES.INTERNAL_ERROR);
throw new ApiClientError(message, res.status, code, details);
```

`ApiClientError.message` は throw の時点で**既に正しい日本語マッピング済み**。各コンポーネント/フックの onError ハンドラは `err.message` をそのまま使うだけで、エラーコードに応じた適切な日本語文言が表示される設計。

### 1.2 実態（バグ）

FE の onError ハンドラの多くが、`err.message` を無視して**ハードコード文言で上書き**している:

| 箇所 | 実装 | 判定 |
|------|------|------|
| `pages/reports/AttachmentUploader.tsx:190-191` | 「ファイルのアップロードに失敗しました。もう一度お試しください。」 | バグ（本 issue の発見起点） |
| `pages/reports/ItemSlidePanel.tsx:336` | 「明細の保存に失敗しました。もう一度お試しください。」 | バグ |
| `pages/reports/AttachmentList.tsx:103, 108, 131, 135` | 「プレビュー/ダウンロードの取得に失敗しました」 | バグ |
| `pages/reports/AttachmentArea.tsx:168` | `showToast('error', '削除に失敗しました')` | バグ |
| `pages/reports/ReportDetailPage.tsx:429/454/473/501` | `onError: () => setItemApiError('明細のXXに失敗しました')`（err すら受け取らない） | バグ |
| `pages/reports/ReportDetailPage.tsx:193-230` `handleActionError` | 422 のみ `err.message` 使用。403/500系/その他は全てハードコード | 部分的バグ |
| **正しいパターン**: `pages/reports/ReportListPage.tsx:141` | `error instanceof Error ? error.message : 'データの取得に失敗しました'` | OK |

### 1.3 副次的な問題

- **コード重複**: `ReportDetailPage.tsx:220` と `error-messages.ts:77` に**同一文言**「サーバーとの通信に失敗しました。しばらくしてから再度お試しください。」がハードコード。将来 `SERVER_ERROR_MESSAGES.INTERNAL_ERROR` の文言変更があっても追従できない
- **可観測性の低下**: ユーザーが「どのエラーに遭遇したか」を画面から判断できず、サポート・デバッグの手がかりが失われる
- **設計意図と実装の乖離**: `SERVER_ERROR_MESSAGES` マッピングを丁寧に定義した意図が、コンポーネント側で無効化されている

### 1.4 規模感

- 対象ファイル: 5〜6 ファイル
- 対象エラーハンドラ: 15 箇所前後
- 各修正は 1〜5 行

---

## 2. 影響

- **UX**: 中〜高（サーバーエラー種別が画面に伝わらず、ユーザーが原因特定・自己対処できない）
- **ポートフォリオ品質**: 中（設計意図と実装の乖離が広範囲にある、レビューで指摘されうる）
- **SMK 整合**: SMK-036 FAIL の一因。他の SMK でも画面表示文言と期待結果の乖離が発生する可能性
- **データ・セキュリティ**: 実害なし

---

## 3. 原因の究明（第一次仮説）

### 3.1 なぜこの実装になったか

現段階の仮説（実装ログ・過去チケット・PR 履歴の再確認で検証する）:

**仮説 A: 設計書でのエラーハンドリング方針が曖昧**
- `dev-journal/deliverables/docs/50_detail_design/` に「FE が BE エラーコードをどう表示するか」の明文規定が不足していた可能性
- 各コンポーネント実装時に「ユーザーに何を表示するか」を実装者が場当たり的に決めた

**仮説 B: `SERVER_ERROR_MESSAGES` マッピングが存在することが開発者間で共有されていなかった**
- `error-messages.ts` は API クライアント層（`client.ts`）の内部実装として扱われ、コンポーネント側の実装ガイドで言及されていなかった可能性
- `ApiClientError.message` が既にマッピング済みである事実が知られていなかった

**仮説 C: 初期実装時の work-breakdown/チケットで「onError の文言」指示が具体的すぎた**
- チケット文言の期待値（例:「保存失敗時に『保存に失敗しました』と表示する」）を実装者がそのままハードコードした
- 実装者視点では「ハードコードで仕様通り」と認識、レビューでも通過した

**仮説 D: レビュー観点にエラーハンドリングの統一性チェックがなかった**
- reviewer / codex レビューで「コンポーネント内でエラー文言をハードコードしていないか」を見ていなかった
- 個々の PR 単位では小さい逸脱で見逃されやすい

### 3.2 検証タスク（本 issue の Phase 1）

以下を実施して仮説を確定させる:

1. **設計書の確認**: `deliverables/docs/50_detail_design/security.md` / `55_ui_component/state-management.md` / `55_ui_component/screens/*.md` にエラーハンドリング方針の記述があるか
2. **work-breakdown/チケットの確認**: Step 8-7（共通UI）/ Step 10-G（添付）/ Step 10 の機能チケットで onError の指示がどう書かれていたか
3. **PR / コミット履歴の確認**: 上記バグ箇所が追加された PR（reviewer / codex 指摘の有無、マージ時の判断）
4. **`error-messages.ts` の導入経緯**: いつ・誰が・どういう意図で追加したか（git log）

---

## 4. 提案

### 4.1 全体ゴール

**「同じバグが次回以降起きない」状態までを本 issue のゴールとする**。

- Phase 1: 原因究明（3.2 の検証タスク）
- Phase 2: 方針策定（エラーハンドリング共通パターンの明文化）
- Phase 3: 既存コードの修正
- Phase 4: 運用更新（work-breakdown テンプレート、レビュー観点、rules 追記）

### 4.2 Phase 1: 原因究明

3.2 の検証タスクを実施。仮説 A〜D のどれが主因かを特定し、本 issue に記録する。検証完了条件:

- 上流（設計書 / work-breakdown / チケット）にエラーハンドリング方針の有無を明示
- PR / commit を特定し、コードレビューでの見逃し経緯を時系列で記録
- 再発防止のターゲット（設計書? work-breakdown? rules? レビュー観点?）を明確化

### 4.3 Phase 2: 方針策定

エラーハンドリング共通パターンを明文化。以下の設計書・ルールに追記:

#### 4.3.1 実装ルール（`/root-project/.claude/rules/implementation-workflow.md`）

```markdown
## FE エラーハンドリング

### 原則
- `ApiClientError.message` は `client.ts` 層で `SERVER_ERROR_MESSAGES` によりマッピング済みである
- コンポーネント/フックの onError ハンドラでは **`err.message` をそのまま使う** ことを原則とする
- エラーコードごとのユーザー向け文言は `frontend/src/lib/error-messages.ts` の `SERVER_ERROR_MESSAGES` に集約する

### 推奨パターン
```ts
onError: (err) => {
  const message = err instanceof Error ? err.message : 'XX の処理に失敗しました';
  setToast({ open: true, severity: 'error', message });
}
```

### アンチパターン（禁止）
```ts
// ハードコード文言で err.message を上書き
onError: () => setItemApiError('明細のXXに失敗しました')  // ← err を受け取らない
onError: (err) => {
  const message = 'XX に失敗しました。もう一度お試しください。';  // ← err.message を無視
  onXxError(message);
}
```
```

#### 4.3.2 UI 層設計書（`deliverables/docs/55_ui_component/state-management.md` 等）

- 「エラーハンドリング」セクションを追加し、以下を明記:
  - `SERVER_ERROR_MESSAGES` の存在と使用方針
  - 推奨パターン / アンチパターン
  - 新規エラーコード追加時の手順（マッピング定義 → コンポーネント側は err.message そのまま）

### 4.4 Phase 3: 既存コードの修正

以下を対応:

| ファイル | 修正内容 |
|---------|--------|
| `pages/reports/AttachmentUploader.tsx:182-196` | onError で `err.message` を使用 |
| `pages/reports/ItemSlidePanel.tsx:336` 周辺 | catch で `err.message` を使用 |
| `pages/reports/AttachmentList.tsx:103, 108, 131, 135` | refetch エラーで `err.message` を使用（onError コールバック経由で ItemSlidePanel / AttachmentArea に伝播） |
| `pages/reports/AttachmentArea.tsx:168` | `err.message` を使用 |
| `pages/reports/ReportDetailPage.tsx:193-230` `handleActionError` | 403/500系/ApiClientError以外分岐で `err.message ?? fallbackMsg` を使用。重複ハードコード文言を `SERVER_ERROR_MESSAGES.XXX` を参照する形に統一 |
| `pages/reports/ReportDetailPage.tsx:429/454/473/501` | onError で err を受け取り `err.message` を使用 |
| 既存テスト | トースト文言アサーションを `SERVER_ERROR_MESSAGES` マッピング経由の値に更新 |

### 4.5 Phase 4: 運用更新（再発防止）

#### 4.5.1 work-breakdown テンプレート

`ai-dev-framework/guide/work-breakdown/` の該当 Step に「エラーハンドリング実装観点」の明示を追加:

- FE 機能実装タスクの「完了条件」に「サーバーエラー時に `err.message` をそのまま表示し、ハードコード文言で上書きしないこと」を追加

#### 4.5.2 レビュー観点

reviewer / codex レビュー向けのチェック観点を追記:

- onError / catch で `err.message` を無視してハードコード文言を渡していないか
- 新規エラーコード（ApiClientError の code）が `SERVER_ERROR_MESSAGES` に定義されているか
- `error-messages.ts` の文言と同一のハードコードがコンポーネント内に重複していないか

#### 4.5.3 lint ルール（オプション）

可能であれば lint で検出（実装コスト次第）:
- ESLint カスタムルール: `onError` / catch ブロック内で `err.message` を使わずにハードコード文字列を渡す挙動を警告
- 最小案: grep + CI チェックで「に失敗しました」を含むコンポーネント内ハードコードを検出し、許可リスト（fallback 文言のみ）と照合

---

## 5. 完了条件

- **Phase 1**: 原因究明の結果が本 issue に追記されている（仮説 A〜D のどれが主因か、検証ログ付き）
- **Phase 2**: エラーハンドリング方針が `.claude/rules/implementation-workflow.md` と UI 設計書に追記されている
- **Phase 3**:
  - 4.4 の該当ファイルで `err.message` が使用されている
  - SMK-036 を再実施して、MIME 偽装検出時に画面トーストに「JPEG, PNG, PDF のみアップロード可能です」が表示される（SMK 期待結果と一致）
  - `INVALID_FILE_TYPE`, `FILE_TOO_LARGE`, `CONFLICT`, `FORBIDDEN`, `RESOURCE_NOT_FOUND`, `INTERNAL_ERROR` の各コードでマッピング文言がそのままトースト表示される
  - 既存テスト通過 + 新規回帰テスト追加
- **Phase 4**:
  - work-breakdown テンプレートに観点追記
  - レビュー観点チェックリストに項目追加
  - lint ルール導入（実装コスト次第、できない場合は見送り判断を記録）

## 6. 関連

- **#131**（open）: AttachmentUploader のクライアントサイドバリデーションエラー表示 UX 改善。本 issue と経路は違うが同ファイルを触るため実装時の衝突に注意
- **#133**（open）: FE/BE 日本語ログの棚卸し。`console.error('ファイルのアップロードに失敗しました:', err)` の削除は #133 で扱う
- **SMK-036**（Step 11-A、FAIL）: 本 issue の修正で SMK 期待結果を画面上で満たすようになる
- `frontend/src/lib/error-messages.ts`: FE 側のコード → 日本語マッピング定義（既存）
- `frontend/src/api/client.ts:124-150`: `handleErrorResponse` によるマッピング適用（既存）

---

## 解決内容

## 解決日
