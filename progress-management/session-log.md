# 引き継ぎメモ

## セッション: 2026-03-23 14:10

### ゴール
- Step 5 完了を目指す（Issue 037 → Phase 3 → Phase 4 → 完了宣言）

### 作業ログ
- **Issue 037 対応**（auth-login フローチャートの API 契約不整合）
  - フローチャートを API 契約（200/400/401/429）に統一
  - E1(422), E3(独立401) を削除、全認証失敗を E2(401 INVALID_CREDENTIALS) に合流
  - openapi.yaml: login の format:email / minLength:8 削除、400 レスポンス追加、SEC-011 意図を description に明記
  - security.md §8.4: login エンドポイントの SEC-011 例外注記を追加
  - codex レビュー: 4回のイテレーション（400 未定義、schema 制約、logout 認証方式、INVALID_TOKEN スコープ）→ LGTM
  - review-finding 046（schema 制約削除の妥当性）を pending-review に記録
  - issue 037 を resolved に移動
- **Issue 038 起票・対応**（rbac.md に RBC-014〜016 の正式定義がない）
  - codex に方針確認: 「上流で正式採番して下流参照を追認する」が妥当
  - rbac.md SS4.2 に RBC-014, RBC-015, RBC-016 を追加
  - SS4.3 を「補足・解釈」に縮退
  - codex レビュー: P3 1件（RBC-010+RBC-011 の誤記）→ 修正 → LGTM
  - issue 038 を resolved に移動
- **Phase 3: authz.md 作成**（認可設計・13セクション・756行）
  - codex に計画レビュー依頼: セクション構成・設計判断・リスクについて意見取得
  - 設計判断確定: 所有権チェック=サービス層 Authorizer、Approver 閲覧=ハイブリッド方式、ロール変更遅延=MVP 許容
  - 内部レビュー: blocker 1件（FORBIDDEN vs PERMISSION_DENIED）+ warning 4件 → 修正 → 再レビュー PASS
  - codex レビュー（/codex-review 正式手順）: 3ラウンド
    - 初回: 047（Approver 閲覧範囲の上流超過）+ 048（ui_flow.md 遷移欠落）
    - 047: rbac.md に追跡閲覧を追記して正式化（コミット 1f4aa58 で既に下流修正済み、上流反映漏れ）→ resolved
    - 2回目: 049（report-detail.md の閲覧範囲記述が古い）→ 修正 → resolved
    - 3回目: 050（openapi.yaml の PERMISSION_DENIED 未反映）→ openapi.yaml に PermissionDenied 追加・11エンドポイント修正 → resolved
  - 048（ui_flow.md 全体図に Admin 遷移欠落）は Phase 4 で対応
- **ワークフロールール改善**
  - codex レビューの指摘対応フローに「LGTM まで繰り返す」を明記
  - Auto Memory ルール変更: 禁止 → `.claude/memory/` に保存（.gitignore 対象、次セッションで自動読み込みされない）
  - `.claude/memory/` ディレクトリ作成

### 未完了
- Phase 4（最終レビュー）未着手
- review-finding 048（ui_flow.md 全体図の Admin/Accounting 遷移欠落）未対応
- progress.md 未更新（Phase 3 完了の反映）

### ブロッカー
- なし

### 次にやること
1. review-finding 048 対応（ui_flow.md 全体図に `DASH001 -> ADM001`, `DASH001 -> ADM002` 追加）
2. Phase 4（最終レビュー・横断）の実施
3. progress.md 更新
4. Step 5 完了宣言

### 学び・気づき
- 内部レビューで LGTM を得る前に codex レビューに持っていった。正しい順序: 内部レビュー → LGTM → コミット → codex レビュー
- codex レビューは `codex exec review --uncommitted` ではなく `/codex-review` スキルの正式手順（コミット後に `codex exec "Step N の初回レビューを実施してください" --full-auto`）で行うべき
- codex レビューは review-findings を起票する前提で動く。内部レビューの指摘は issue 不要だが、codex の指摘はレビュー指摘資料として管理される
- ルールに書いてあることに従えなかった場合、「以後気をつけます」は空約束（セッション間の記憶がない）。仕組み（ルールの明文化）で担保するしかない
- Auto Memory を使うなとルールにあるのにシステムプロンプトに引きずられて書いてしまった。対策: `.claude/memory/` への書き込み先変更で、衝動を制御可能にする

### 意思決定ログ
- Issue 037: フローチャートを API 契約に合わせる方針（SEC-011 に基づき全認証失敗を 401 に統一、リクエストボディ不正は 400）
- Issue 038: 上流（rbac.md）に RBC-014〜016 を正式採番。下流が使っている ID を追認する形。codex の助言「正式ルールと補足説明の境界を整理する」に従い SS4.3 を縮退
- authz.md 設計判断: 所有権チェックはサービス層 Authorizer パターン（architecture.md のハンドラ層から変更、理由を注記）
- FORBIDDEN vs PERMISSION_DENIED: FORBIDDEN=ロール不足（MW層）、PERMISSION_DENIED=所有権不足（Authorizer層）に区別。openapi.yaml にも PermissionDenied レスポンスを追加
- Approver 追跡閲覧: rbac.md に正式化。コミット 1f4aa58 で下流は修正済みだったが上流反映が漏れていた
- Auto Memory 運用: 禁止ではなく `.claude/memory/` に書かせる。次セッションで自動読み込みされないが、git 追跡外で蓄積可能。将来ルール化の種になりうる
- codex レビュー手順: `/codex-review` スキルを使い、コミット後に正式手順で実行する

---

## セッション: 2026-03-22 20:37（前回）

### ゴール
- 成果物の図を Mermaid 形式に統一する
- 詳細設計の図の不足を洗い出し、補強する

### 作業ログ
- **ASCII罫線図 → Mermaid変換**（3ファイル・5箇所）
  - domain_model.md: 集約構成図 + レイヤー責務図
  - architecture.md: システム全体構成図 + ミドルウェアチェーン
  - business-flow.md: 月次業務サイクル図
- **ASCIIシーケンス図 → Mermaid変換**（architecture.md・2箇所）
  - ログイン→JWT発行、認証付きリクエストフロー
- **UIライブラリ変更: shadcn/ui → MUI**
  - ADR-0001 に比較検討がなかった点をユーザーに報告
  - ポートフォリオ用途で「楽にきれいに」→ MUI が最適と判断
  - 4ファイル・5箇所を更新（ADR, architecture, security の Tailwind→Emotion）
- **全13画面に処理シーケンス図を追加**（18図）
  - ユーザー指摘: 詳細設計に「ボタン→API→サービス→ドメイン→DB」の縦断フローがない
  - 内部レビュー: blocker 5件 + warning 10件 → 修正 → 再レビュー LGTM
  - codex レビュー: 3件（OpenAPI項目名不一致、INSERT文カラム名、カーソル条件欠落）→ 修正 → 再レビュー LGTM
- **フローチャート追加**（3ファイル・4図）
  - report-detail.md: 提出バリデーション + 状態遷移操作の権限チェック（統合図）
  - files.md: ファイルアップロードバリデーション
  - auth-login.md: 認証フロー分岐（SEC-011 合流表現）
  - 内部レビュー: blocker 2件（自己承認チェック比較対象、却下フロー自己チェック欠落）→ 修正 → LGTM
  - codex レビュー: 2件 → 045 即修正 + 044 issue 037 起票 → 再レビュー LGTM
- **UIデザインガイドライン作成**（ui-guidelines.md）
  - MUI デフォルトテーマベース、ステータスカラーマッピング、コンポーネント使用方針
  - work-breakdown + task-plan に成果物として追記
- **work-breakdown 改善**
  - 完了条件に「処理シーケンス図」「フローチャート」を追加
  - レビュー観点にも同項目を追加
  - 今後 Planner が自動的にスコープに含めるようになる

### 未完了
- issue 037（auth-login フローチャートの API 契約不整合）未対応

### ブロッカー
- なし

### 次にやること
1. issue 037 対応（auth-login フローチャートを API 契約に合わせるか、API 契約を拡張するか）
2. Phase 3（authz.md）に着手
3. Phase 4（最終レビュー）
4. Step 5 完了宣言

### 学び・気づき
- work-breakdown に書かれていない成果物は作られない。Planner も Designer も work-breakdown に忠実に動く。成果物の「完了条件」にシーケンス図・フローチャートを含めなかったことが、今回の補強作業の根本原因
- ADR にUIライブラリの比較検討がなかった。選定理由が1行では判断根拠が不十分。ただしMVPでは変更コストが低いため実害は小さかった
- codex レビューは内部レビューと異なる視点で指摘する（OpenAPI との項目名一致、DBカラム名の大文字小文字など）。両方実施することで網羅性が上がる

### 意思決定ログ
- UIライブラリ: shadcn/ui → MUI に変更。理由: ポートフォリオ用途で「楽にきれいに」、MUI はテーマで一貫性担保、DataGrid 等の業務コンポーネント充実、Claude の実装力も MUI の方が高い
- UIデザイン方針: MUI デフォルトテーマをそのまま使用。カラー変更は theme.ts 1ファイルで後から可能
- 上流シーケンス図は残す: 詳細設計に縦断フローを追加しても、上流（要件定義・アーキテクチャ）の図は別視点のため削除しない
- work-breakdown 改善: 「完了条件」と「レビュー観点」に図の有無を追加することで、次回以降は Planner が自動的にスコープに含める
