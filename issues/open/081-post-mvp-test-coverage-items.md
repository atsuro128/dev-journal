# MVP スコープ外のテスト観点一括管理

## 発見日
2026-04-13

## カテゴリ
testing

## 影響度
中

## 発見経緯
escalation

## 関連ステップ
Step 6（テスト設計）/ Step 11（システムテスト・UAT）

## ブロッカー
なし

## ⚠ MVP スコープ外

本 issue は MVP スコープ外のテスト観点を一括記録するものであり、MVP リリース前には対応しない。管理方式は **`ops-080`** で検討中。決定後、本 issue の §1〜§5 は適切な管理先へ移動または分割される。

## 概要

issue 073（work-breakdown の保証種別 5 分類 → 8 分類統一）の対応過程で、`traceability.md` に「カバー済み」と記載されているが実ケースが不足している観点、および要件自体が存在しない観点が複数発覚した。これらを「MVP スコープ外」と判断したが、忘却防止のため本 issue で詳細を一括記録する。

各観点は章立て（§1〜§5）で管理し、要件 ID 状態・現状カバー状況・検証コスト・優先度・要件遡及の必要性を明示する。

---

## §1 CORS / セキュリティヘッダ

### 要件 ID
- `NFR-SEC-006` (CORS: 許可オリジンを明示指定)
- `NFR-SEC-007` (セキュリティヘッダ: HSTS, X-Content-Type-Options, X-Frame-Options)

### 現状
- **実装**: BE middleware で対応済み（`internal/middleware/` 配下）
- **テストケース**: `cross-cutting.md` には定義されていない
- **traceability.md の従前記載**: `CRS-076〜088` で「カバー済み」扱い（実態は暗黙的カバーの虚偽標記）
- **手動補強**: なし

### 提案内容
- `cross-cutting.md` に以下のテストケース（または同等内容）を追加:
  - `CRS-089: TestCORS_AllowedOrigin_Success` — 許可オリジンからのリクエストが CORS ヘッダを正しく返す
  - `CRS-090: TestCORS_DisallowedOrigin_Blocked` — 不許可オリジンからのリクエストがブロックされる
  - `CRS-091: TestSecurityHeaders_HSTS` — HSTS ヘッダが付与される
  - `CRS-092: TestSecurityHeaders_XContentTypeOptions` — `X-Content-Type-Options: nosniff`
  - `CRS-093: TestSecurityHeaders_XFrameOptions` — `X-Frame-Options: DENY`
- 実装は `net/http/httptest` で軽量に検証可能

### 検証コスト
**小**（middleware 層のレスポンスヘッダ確認のみ、DB 不要）

### 優先度
**高**（セキュリティ要件の保証、実装済み機能の検証）

### 要件遡及
**不要**（NFR-SEC-006/007 が既存）

---

## §2 タイムゾーン（UTC 保存 / JST 表示）

### 要件 ID
- `NFR-DATA-007` (タイムゾーン: UTC 保存、表示時 JST 変換)

### 現状
- **実装**: BE は UTC で DB 保存、FE は `formatJST` ユーティリティで JST 表示
- **テストケース**: `cross-cutting.md` には定義されていない
- **traceability.md の従前記載**: `- | - | 自動テスト対象外（アプリケーション実装規約で保証。Step 9 以降でタイムスタンプ形式の検証を含める）`
- **手動補強**: `smoke_check.md SMK-070/071` で実施予定

### 提案内容
- BE: レポート提出時刻・承認時刻等が `YYYY-MM-DDTHH:MM:SSZ` 形式（UTC）で返ることの検証
- FE: `formatJST` ヘルパーのユニットテスト + コンポーネントテストで JST 表示が正しいことの検証

### 検証コスト
**中**（BE + FE の両方で必要、時刻データの前提準備）

### 優先度
**中**（タイムゾーン処理のバグは運用で気付きにくいため実益あり）

### 要件遡及
**不要**（NFR-DATA-007 が既存、ただし「自動テスト対象外」と記載されていた前提を覆す場合は traceability.md の意思決定を変更する必要がある）

---

## §3 /health エンドポイント

### 要件 ID
- `NFR-AVAIL-002` (ヘルスチェックエンドポイント `/health`)

### 現状
- **実装**: 実装済み（200 OK を返す単純なエンドポイント）
- **テストケース**: `cross-cutting.md` には定義されていない
- **traceability.md の従前記載**: `- | - | Step 9 以降で実装時にヘルスチェックテストを追加`
- **手動補強**: なし

### 提案内容
- `cross-cutting.md` に 1 ケース追加:
  - `CRS-094: TestHealth_Returns_200_OK` — GET `/health` が 200 OK を返し、認証不要でアクセスできる

### 検証コスト
**極小**（1 ケース、依存ゼロ）

### 優先度
**小**（実装済み機能の検証、コストがほぼゼロなので拒む理由なし）

### 要件遡及
**不要**（NFR-AVAIL-002 が既存）

---

## §4 真の競合シナリオ（楽観ロック 409 レース再現）

### 要件 ID
**未定** — 要件遡及が必要

現状 `requirements.md` / `policies.md` で「楽観ロックによる 409 返却」は記載があるが、「**同時実行シナリオでの挙動**」は要件レベルで明確化されていない。

### 現状
- **実装**: 楽観ロック機構は BE の各サービス層で実装済み（`version` 列 + `WHERE version = ?` + 影響行数チェック）
- **既存テスト**: 単一リクエストでの 409 テスト（例: version 不一致で 409 を返す）は存在する
- **未実施**: 並行リクエストでの実レース再現（goroutine + sync.WaitGroup）
- **traceability.md の従前記載**: 要件 ID が存在しないため未記載

### 提案内容
1. 要件定義への遡及:
   - `requirements.md` または `policies.md` に「競合シナリオでの楽観ロック動作保証」要件を追加（例: `NFR-DATA-008`）
   - `test_strategy.md` に並行実行テストの方針を追加
2. テストケース:
   - `CRS-095: TestConcurrentApproval_ReturnsOneSuccess_OneConflict` — 同一レポートへの同時承認要求で、1 つが成功、1 つが 409 を返す

### 検証コスト
**大**（goroutine + sync.WaitGroup、DB トランザクション境界の制御、**flaky test リスクあり**）

### 優先度
**低**（flaky test リスクと検証コストに対して実益が限定的）

### 要件遡及
**必要**（要件 ID の新設）

---

## §5 二重送信防止

### 要件 ID
**不在** — 要件定義から遡及必須

`requirements.md` / `policies.md` / `pre_step/04_business-rules.md` を全文検索しても「二重送信防止」「idempotency key」「重複送信」に該当する要件・ポリシーの記載は見つからない。

### 現状
- **実装**: FE / BE のどちらにも明示的な二重送信防止機構の仕様がない
  - FE: ボタンクリック後の disable 処理は一部のコンポーネントで実装されているが、設計書に規定なし
  - BE: idempotency key によるリクエスト重複排除は未実装
- **実質的なリスク**: ユーザーがレポート提出ボタンを連打すると、複数の提出リクエストが発生する可能性
- **smoke_check.md / uat_check.md**: 二重送信防止の確認項目なし

### 提案内容
1. 要件定義への遡及:
   - `requirements.md` に「二重送信防止」要件を新設（例: `NFR-UX-NNN`）
   - 「楽観ロック + UI disable」か「idempotency key」のどちらを採用するか設計判断
2. 設計:
   - 採用方針に応じて BE / FE の設計書を更新
3. 実装:
   - BE: idempotency key 採用時はミドルウェア層で実装
   - FE: 既存の disable 処理を state-management.md の共通パターンとして規定
4. テストケース:
   - 要件・設計確定後に cross-cutting.md に定義

### 検証コスト
**中**（要件定義 → 設計 → 実装 → テストまでの新規作業）

### 優先度
**中**（UX バグ防止、セキュリティ観点でもあり）

### 要件遡及
**必要**（要件自体が不在）

---

## 関連

- **issue 073**: 本 issue の起票契機（`traceability.md` の虚偽カバー標記整理の過程で顕在化）
- **ops-080**: MVP スコープ外事項の管理方式決定。本 issue はその検討対象
- **Step 11-B**: 横断テスト実装タスク。ops-080 の決定次第では、本 issue の §1〜§3（要件既存）を Step 11-B に統合する選択肢もある
- **requirements.md / policies.md**: §4 §5 の要件遡及対象

## traceability.md からの参照

本 issue 起票時点で、`traceability.md` の以下の行で本 issue を参照する:

| 要件 ID | 参照先 |
|---------|--------|
| NFR-SEC-006 (CORS) | `issue 081 §1` で追跡、管理方式は ops-080 で検討中 |
| NFR-SEC-007 (セキュリティヘッダ) | `issue 081 §1` で追跡、管理方式は ops-080 で検討中 |
| NFR-DATA-007 (タイムゾーン) | `issue 081 §2` で追跡、管理方式は ops-080 で検討中 |
| NFR-AVAIL-002 (/health) | `issue 081 §3` で追跡、管理方式は ops-080 で検討中 |
| (要件ID 未定、競合シナリオ) | `issue 081 §4` で追跡、要件遡及必要、管理方式は ops-080 で検討中 |
| (要件ID 不在、二重送信防止) | `issue 081 §5` で追跡、要件遡及必要、管理方式は ops-080 で検討中 |

---

## 解決内容

本 issue は ops-080 の決定後に移動または分割される。現時点では「情報保持のための記録」が主目的であり、個別観点の実装・要件遡及は ops-080 決定後に各セクションごとに着手する。

## 解決日
