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

## 7. Phase 1 調査結果

### 7.1 仮説の検証

| 仮説 | 判定 | 根拠 |
|------|------|------|
| A: 設計書曖昧 | **部分成立（実態は「曖昧」ではなく「古い前提のまま更新されていない」）** | `55_ui_component/state-management.md` §6「エラーハンドリングの状態管理パターン」と §6.5「エラーメッセージ管理方針」は明文化されていた（`ERROR_MESSAGES` マッピング定義、`getErrorMessage(error)` ヘルパー、画面固有上書きルール §6.5.7 等）。ただし設計では **各 onError で `getErrorMessage(error)` を呼び出す** 前提になっており、**`ApiClientError.message` が既にマッピング済みである**現行実装（#124 対応）への追従がされていない。さらに `50_detail_design/screens/report-detail.md` §11 は HTTP エラー種別ごとの日本語文言を表形式で列挙しており（例: 「サーバーとの通信に失敗しました…」「この操作を行う権限がありません。」）、実装者がこの表を「コンポーネントにハードコードせよ」と解釈できる書き方になっている。実際 `ReportDetailPage.tsx:220/227/214` のハードコード文言は同表の文字列と完全一致する |
| B: マッピング未共有 | **成立** | `.claude/rules/implementation-workflow.md` は FE 側のエラーハンドリング方針に一切触れていない（「必須参照」に `state-management.md` / `common-components.md` も含まれていない）。`error-messages.ts` と `client.ts` の `handleErrorResponse` 層マッピングは **issue #124 対応・PR #70（`dc94d5` 2026-04-20）で初めて導入された** ため、Step 10-B（2026-04-09 の `3c375e3a`）/ 10-G（2026-04-18 の `c24bfe0c`）時点では **そもそも `client.ts` 層のマッピングは存在しなかった**。当時 `err.message` は BE の英語 raw メッセージのままで、日本語文言をコンポーネントで組み立てる実装は当時の実装者にとって合理的だった。その後 #124 で前提が変わったが、既存の onError ハンドラは更新されず取り残された |
| C: チケット指示具体すぎ | **不成立** | `progress-management/tickets/step10/10-B-report.md` / `10-G-attachments.md` を grep しても「エラー」「error」「失敗」「onError」等のキーワードは **一切ヒットしない**。チケット本文で onError 文言を具体的に指示した形跡はない。実装者は設計書（screens/report-detail.md §11）の文言表を参照して判断したと推察される（→ 仮説 A の実態） |
| D: レビュー観点欠落 | **成立** | `work-breakdown/step10-feature-implementation/review.md` の FE レビュー観点（L77-81）は「API クライアント、認証トークン管理、共通エラーハンドリングの利用方式が Step 8 基盤と一致しているか」という抽象レベルにとどまり、「onError で `err.message` をそのまま使っているか」「ハードコード文言で上書きしていないか」「`SERVER_ERROR_MESSAGES` と同一文字列がコンポーネント内に重複していないか」等の具体観点は存在しない。`work-breakdown/step8-foundation/review.md` 側も同様。結果として PR #58・#68・#70 等のマージ時に重複・上書きが見逃された |

### 7.2 主因の特定

**主因は A と B の複合（時系列ズレによる設計と実装の乖離）、D（レビュー観点欠落）が再発増幅要因**:

1. **Step 10-B / 10-G 実装時点（2026-04-09 〜 04-18）**: `client.ts` 層のマッピングがまだ存在せず、`screens/*.md` §11 の日本語文言表を参照して各 onError に日本語をハードコードする実装は当時合理的だった。`ReportListPage.tsx` のみ `error instanceof Error ? error.message : ...` パターンを採用していたが、これは例外的で、同 PR（`3c375e3a`）の `ReportDetailPage.tsx` では §11 表と一致するハードコード文言が主流だった
2. **issue #124 対応時点（2026-04-20、PR #70 / `dc94d54`）**: `handleErrorResponse` で `SERVER_ERROR_MESSAGES` マッピングが導入され、**`ApiClientError.message` は throw 時点で既に日本語マッピング済み** に仕様変更された。ただしこの変更は `client.ts` と `error-messages.ts` に閉じて完結し、**既存コンポーネントの onError ハンドラを書き換える作業は含まれていなかった**。上流設計書（`state-management.md` §6.5.3 の `getErrorMessage(error)` ヘルパー、`screens/report-detail.md` §11 の文言表）も更新されなかった
3. **レビューでの見逃し（仮説 D）**: #124 以降の PR（#68, #71, #72 等）でも既存の onError パターンを踏襲したコードが追加され続け、reviewer・codex レビューともに「新しい設計前提（`err.message` がマッピング済み）」を適用したチェックが行われなかった

設計書が「曖昧」だったのではなく、**マッピングの担当レイヤーが途中で移動した（コンポーネント層 → API クライアント層）のに、上流ドキュメント・既存コード・レビュー観点のいずれもその移動を完全にフォローしきれていない**、というのが実態。

### 7.3 実装時系列（抜粋）

| 時期 | コミット / PR | 対象 | 内容 |
|------|-------------|------|------|
| 2026-04-08 | `653f84a3` / PR #19 | `ReportListPage.tsx:141` | 「正しいパターン」 `error instanceof Error ? error.message : 'データの取得に失敗しました'` がここで導入（fallback 付きで `err.message` 優先） |
| 2026-04-09 | `3c375e3a` / Step 10-B | `ReportDetailPage.tsx:193-230` `handleActionError` | 409 / 422 / 403 / その他 の 4 分岐。**422 のみ `err.message \|\| fallbackMsg`、他はハードコード**。L220/227 の 500系文言は `error-messages.ts:INTERNAL_ERROR` と完全一致、L214 の 403 文言は `FORBIDDEN` と完全一致 |
| 2026-04-09 | `3c375e3a` / Step 10-B | `ReportDetailPage.tsx:429/454/473/501` | `onError: () => setItemApiError('明細の XX に失敗しました')` が 4 箇所で追加。**err パラメータを受け取らず**、常に固定文言 |
| 2026-04-09 | `3c375e3a` / Step 10-B | `AttachmentList.tsx:168`（現在位置） | `showToast('error', '削除に失敗しました')` — 添付削除 onError のハードコード文言 |
| 2026-04-11 | `73730549` / 10-X | `AttachmentUploader.tsx:190-191` | `console.error('ファイルのアップロードに失敗しました:', err)` + `const message = 'ファイルのアップロードに失敗しました。もう一度お試しください。'` を onError に追加 |
| 2026-04-18 | `c24bfe0c` / PR #67（issue 102） | `AttachmentList.tsx:103, 108, 131, 135` | 添付プレビュー・ダウンロードの catch で `onError('プレビューの取得に失敗しました')` / `onError('ダウンロードの取得に失敗しました')` を追加。err を受け取らず固定文言 |
| 2026-04-20 | `d6c1c94d` / PR #68（issue #108） | `ItemSlidePanel.tsx:336` | catch 文で `createErr.message` を使うパターン（部分的に正しい）だが、fallback 文言として `'明細の保存に失敗しました。もう一度お試しください。'` をハードコード |
| **2026-04-20** | **`dc94d54` / PR #70（issue #124）** | **`client.ts:124-150`, `error-messages.ts` 新規作成** | **`handleErrorResponse` で `SERVER_ERROR_MESSAGES` マッピングを導入。以降 `ApiClientError.message` は throw 時点で日本語マッピング済みになる。本件バグの前提条件が切り替わった分水嶺** |
| 2026-04-20 以降 | — | 既存 onError ハンドラ | 上記分水嶺以降も **書き換えられず、設計書 `state-management.md` §6.5.3 の `getErrorMessage(error)` ヘルパー前提とも、新実装 `err.message` そのまま利用前提とも乖離した状態** |

### 7.4 再発防止のターゲット確定

主因（A 実態 = 古い前提残留、B 共有不足、D レビュー観点欠落）を踏まえ、Phase 2-4 で手を入れるべき箇所を確定:

#### Phase 2（方針策定）の追記先

- **`.claude/rules/implementation-workflow.md`**（高優先度）: 現状 FE エラーハンドリング方針は一切未記載。新規セクション「FE エラーハンドリング」を追加し、`err.message` そのまま使用の原則・推奨パターン・アンチパターンを明文化する。「必須参照」にも `55_ui_component/state-management.md` と `55_ui_component/common-components.md` を追加する
- **`55_ui_component/state-management.md` §6.5**（必須更新）: `ApiClientError.message` が `client.ts` 層でマッピング済みになった事実（#124）を前提に更新。§6.5.3 `getErrorMessage(error)` ヘルパーの位置付けを「廃止」または「画面固有上書きのみで使用」に改訂。§6.5.7 画面固有上書きのパターンも `ApiClientError.code` 分岐ベースに簡素化
- **`50_detail_design/screens/report-detail.md` §11**（および他 screens）: エラー種別ごとの文言表は残しつつ、冒頭に「表内の文言は `SERVER_ERROR_MESSAGES` マッピングの期待値であり、コンポーネントにハードコードするものではない」旨の注記を追加。文言出典を `error-messages.ts` にリンクする
- **`55_ui_component/common-components.md`** の `AppToast` / `FormAlert` セクション: message prop の推奨ソース（`err.message`）を明記

#### Phase 4（work-breakdown テンプレート）の追記箇所

- **`step10-feature-implementation/review.md`** L77-81（フロントエンドレビュー観点）に以下を追加:
  - onError / catch で `ApiClientError.message` をそのまま使用しているか（ハードコード文言で上書きしていないか）
  - onError ハンドラが `err` パラメータを受け取っているか（`onError: () => ...` の無視パターンがないか）
  - コンポーネント内の日本語エラー文言が `SERVER_ERROR_MESSAGES` と重複していないか
- **`step8-foundation/review.md`** §8-7（共通 UI）にも同等の観点を追加（AppToast 利用側での文言ハードコード禁止）
- **`step10-feature-implementation/main.md`** の機能実装タスクの「完了条件」共通項に「サーバーエラー時に `ApiClientError.message` を表示し、ハードコード文言で上書きしないこと（fallback 文言は許容）」を追加

#### Phase 4（レビュー観点）の追加内容

- reviewer / codex レビュー向け: 上記 3 観点をコードレビューチェックリストに明記
- `screens/*.md` §11 の文言表を更新または変更した場合は、`error-messages.ts` との整合を必ず確認する手順を追加

#### Phase 4（lint）の導入可否判断

- **方針（推奨）**: 本格的な ESLint カスタムルール開発は実装コストに対する再発防止効果が限定的なため見送る。代わりに以下の軽量策を推奨:
  - **CI grep チェック**: `expense-saas/frontend/src/pages/` 配下で「に失敗しました」「できません」「行えません」等のハードコード日本語エラー文言を検出し、許可リスト（`error-messages.ts` 本体・fallback 用途・空状態文言）と照合するシェルスクリプトを CI に追加
  - **根拠**: Phase 3 で既存バグを全修正したあとは、新規コミットでハードコード文言が混入しない限り再発しない。grep ベースの軽量チェックで十分カバーでき、ESLint AST パースまで必要としない
  - **見送る場合の補完**: レビュー観点（Phase 4）に明示することで人によるチェックを担保。lint 導入は将来の余力判断とする

---

## 6. 関連

- **#131**（open）: AttachmentUploader のクライアントサイドバリデーションエラー表示 UX 改善。本 issue と経路は違うが同ファイルを触るため実装時の衝突に注意
- **#133**（open）: FE/BE 日本語ログの棚卸し。`console.error('ファイルのアップロードに失敗しました:', err)` の削除は #133 で扱う
- **SMK-036**（Step 11-A、FAIL）: 本 issue の修正で SMK 期待結果を画面上で満たすようになる
- `frontend/src/lib/error-messages.ts`: FE 側のコード → 日本語マッピング定義（既存）
- `frontend/src/api/client.ts:124-150`: `handleErrorResponse` によるマッピング適用（既存）

---

## 8. Phase 4 判断記録

### 8.1 lint ルール導入: 見送り判断

**判断**: ESLint AST カスタムルールおよび CI grep チェックは**本 issue では導入しない**。

**理由**:
- **ESLint AST カスタムルール**: `onError` / catch ブロック内でのハードコード文言検出は AST パターンが複雑（引数なし `onError: () => ...` と引数あり `onError: (err) => ...` の両方を正確に区別する必要あり）。実装・メンテナンスコストに対して再発防止効果が限定的
- **CI grep チェック**: 「に失敗しました」等のキーワードで検出するアプローチは誤検知（fallback 文言・確認ダイアログ文言・成功メッセージ等）が多く、許可リスト管理の運用コストが発生する。本プロジェクトの規模・運用体制に馴染まない
- **補完策**: Phase 2 でルール文書（`.claude/rules/implementation-workflow.md`）および設計書（`state-management.md` §6.5.3、`screens/*.md` §11 注記）に明文化済み。Phase 4 でレビュー観点チェックリスト（`step10-feature-implementation/review.md` §5 / `step8-foundation/review.md` §5.5）にも追記済みであり、人によるチェックで十分カバーできる
- **将来**: FE コードベースが拡大し、lint による自動検出のコスト効果が改善したタイミングで改めて検討する

## 解決内容

PR #84（commit 2ab3926）にて対応。Phase 1（原因究明）〜 Phase 4（再発防止）を完遂。FE の `AttachmentUploader.tsx` / `ItemSlidePanel.tsx` / `AttachmentList.tsx` / `AttachmentArea.tsx` / `ReportDetailPage.tsx` 計 15 箇所の onError ハンドラを `err.message` ベースに修正。`.claude/rules/implementation-workflow.md` へ FE エラーハンドリング方針を追記し、`step10-feature-implementation/review.md` にレビュー観点を追加。SMK-036 再検証にて画面トーストに正しい日本語マッピング文言が表示されることを確認。

## 解決日

2026-04-22
