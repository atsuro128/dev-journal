# 引き継ぎメモ

## セッション: 2026-03-23 15:25

### ゴール
- Step 5 完了を目指す（review-finding 048 対応 → Phase 4 最終レビュー → 完了宣言）

### 作業ログ
- **review-finding 048 対応**（ui_flow.md 全体図の遷移欠落）
  - ui_flow.md の全体画面遷移図に `DASH001 -> ADM001`, `DASH001 -> ADM002` エッジ追加
  - review-finding 048 を resolved に移動
- **Phase 4 最終レビュー実施**（unit x4 + cross x1 = 5エージェント並列）
  - Auth 4画面: LGTM
  - Dashboard + Workflow 3画面: LGTM（info 1件 — Phase 3 対応で十分）
  - Admin 2画面: warning 2件（ソートキー不一致、期間フィルタ曖昧）
  - Report 4画面: blocker 1件（ソートキー updated_at vs created_at）
  - 横断レビュー: LGTM（warning 2件 — エラーコード略記、submitter 用語 → PASS）
- **Phase 4 指摘修正**
  - RPT-001: シーケンス図 `ORDER BY updated_at` → `created_at` に修正
  - ADM-001: ソートキー `updated_at` → `submitted_at DESC NULLS LAST` に修正 + 期間フィルタのセマンティクス明確化
  - ADM-001: draft ソート順の矛盾修正（`created_at DESC` 追加）
  - 再レビュー: RPT-001 LGTM、ADM-001 LGTM
- **codex レビュー（Phase 4 後）**
  - review-finding 051 起票: PERMISSION_DENIED が上流（RBC-004）と乖離
- **review-finding 051 対応**（PERMISSION_DENIED → FORBIDDEN 統一）
  - ユーザー指摘: なぜ上流と違う設計になったのかをはっきりすべき
  - 経緯調査: Step 5 authz.md 作成時に独断導入された概念。上流に定義なし
  - codex に追加情報を踏まえた判断を依頼 → 方針 C（MVP では FORBIDDEN に統一、将来分離可能と注記）
  - 9ファイルを修正（authz.md, openapi.yaml, security.md, architecture.md, domain_model.md, files.md, report-create/edit/detail.md）
  - 内部レビュー: LGTM
  - codex レビュー: 051 resolved、新規指摘なし
- **Step 5 完了宣言**
  - task-plan Phase 3/4 ステータスを完了に更新
  - progress.md を完了（2026-03-23）に更新

### 未完了
- なし（Step 5 完了）

### ブロッカー
- なし

### 次にやること
1. Step 6（テスト設計）に着手
   - まず `dev-journal/guide/work-breakdown/step6-testing.md` を確認
   - open issues のうち Step 6 に関連するもの（ops-028）を確認
2. Step 6 の作業計画を立案（Plan エージェント）

### 学び・気づき
- review-finding 048 のレビューを内部レビューに回してしまったが、codex からの指摘は codex に再レビューを委譲すべき。正しい流れ: 修正 → コミット → `/codex-review`
- review-finding の修正後は `pending-review/` に移動してからコミット。`resolved/` に移動するのは codex が解決確認した後
- PERMISSION_DENIED の導入は上流に定義がない概念の独断導入だった。「なぜ上流と違うのか」を問うユーザーの視点が、スコープ逸脱を防ぐ重要なチェックポイント
- codex に判断を求める際、追加の文脈情報（MVP での実益、プロジェクトルール等）を添えると、より妥当な回答が得られる

### 意思決定ログ
- PERMISSION_DENIED 廃止: 方針 C を採用。MVP の外部契約は上流 RBC-004 準拠で FORBIDDEN に統一。将来分離は authz.md に注記として残す。理由: MVP で分離の実益なし、独断導入は「設計判断を独断しない」に反する
- ソートキー統一: RPT-001 は `created_at`（DB インデックスと整合）、ADM-001 は `submitted_at DESC NULLS LAST`（画面仕様「提出日の降順」と整合）+ draft 間は `created_at DESC` で補完
- Phase 4 横断レビュー warning の PASS 判定: openapi.yaml のエラーコード略記（description 内、実害なし）と submitter 用語（動詞と名詞の区別、glossary.md の意図に反しない）は下流に支障なしと判断

---

## セッション: 2026-03-23 14:10（前回）

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
