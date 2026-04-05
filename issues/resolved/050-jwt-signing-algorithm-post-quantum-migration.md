# JWT 署名アルゴリズムを耐量子計算機方式へ移行

## 発見日
2026-04-02

## カテゴリ
security

## 影響度
中

## 発見経緯
user-report

## 関連ステップ
Step 5（詳細設計）、Step 8（基盤構築）

## 問題

現在 JWT 署名に RS256（RSA 2048bit + SHA-256）を採用しているが、NIST は RSA 2048bit を 2030 年以降非推奨とする方針を示している（SP 800-131A Rev.2）。実務レベルのセキュリティを実現する上で、非推奨予定のアルゴリズムを採用していることはマイナス評価となる。

EdDSA（Ed25519）は RSA より高速・短鍵だが、楕円曲線ベースのため量子コンピュータに対する耐性がない。

## 影響

### 設計書（修正必要）
- `dev-journal/deliverables/docs/50_detail_design/security.md` — RS256 鍵管理セクション、JWT 検証フロー、品質チェック（6箇所）
- `dev-journal/deliverables/docs/50_detail_design/openapi.yaml` — セキュリティスキーム定義（2箇所）
- `dev-journal/deliverables/docs/50_detail_design/authz.md` — JWT 検証説明（間接的言及）

### コード（修正必要）
- `expense-saas/internal/pkg/jwt/jwt.go` — Verifier の RSA 固定実装

## 提案

1. **調査**: Go の JWT ライブラリにおける耐量子署名アルゴリズムの成熟度を調査
   - ML-DSA（CRYSTALS-Dilithium, FIPS 204）— NIST 標準化済み
   - Go 実装の有無と安定性（`circl` 等）
   - `golang-jwt` にカスタム SigningMethod として登録する接続コードの実装可否
2. **方針決定**: 調査結果に基づき、以下のいずれかを選択
   - A. ML-DSA を採用（耐量子を全面アピール）
   - B. EdDSA を採用し、耐量子対応は将来課題として明記（実用性優先）
   - C. ハイブリッド方式（従来 + PQC の二重署名）

### 署名サイズのトレードオフ

JWT は毎リクエスト Authorization ヘッダーに載せて送るため、署名サイズが通信量に直結する。

| 方式 | 署名サイズ | JWT 全体の目安 | RS256 比 |
|------|-----------|---------------|----------|
| RS256（現行） | 256 bytes | 約 800 bytes | 1x |
| EdDSA | 64 bytes | 約 600 bytes | 0.75x |
| ML-DSA-44（最小） | 約 2.4 KB | 約 3.5 KB | 4.4x |
| ML-DSA-65 | 約 3.3 KB | 約 4.5 KB | 5.6x |

- HTTP ヘッダー上限（一般的に 8KB）には収まる
- モバイル回線では毎リクエスト数 KB 増加が気になる可能性あり
- Cookie 方式だと 4KB 制限に抵触するが、本プロジェクトは Authorization ヘッダー方式のため問題なし
- 方針決定時にこのサイズ増加を許容するか判断が必要
3. **設計書修正**: 選択した方式に合わせて security.md、openapi.yaml を更新
4. **コード修正**: `jwt.go` の署名・検証ロジックを変更

---

## 解決内容

RS256 を維持する。詳細は ADR-0006（`deliverables/docs/30_arch/adr/0006-jwt-signing-algorithm.md`）を参照。

- EdDSA: 量子耐性がない点は RS256 と同じ。移行の論理的根拠なし
- ML-DSA: Go エコシステム未成熟、署名サイズ 4〜6倍増
- RS256: NIST 2030年非推奨は予防的ガイドライン。本プロジェクトの想定ライフサイクルで実リスクなし

なお、本 issue の出典「SP 800-131A Rev.2」は不正確。同文書は RSA 2048 を "acceptable" としている。正しい出典は NIST IR 8547（2024年11月ドラフト）。

## 解決日
2026-04-05
