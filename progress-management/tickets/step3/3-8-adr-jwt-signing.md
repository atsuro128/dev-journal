# ADR: JWT 署名アルゴリズム選定

- 担当: architect
- 依存: なし
- ブランチ: なし（master 直接コミット）
- 出力先: `deliverables/docs/30_arch/adr/0006-jwt-signing-algorithm.md`
- テンプレート: `ai-dev-framework/templates/docs-v2/30_arch/adr/adr-template.md`

## 入力

| 資料 | パス | 参照箇所 |
|------|------|----------|
| issue 050 | `dev-journal/issues/open/050-jwt-signing-algorithm-post-quantum-migration.md` | 全体 |
| セキュリティ設計 | `dev-journal/deliverables/docs/50_detail_design/security.md` | §2.1 JWT、RS256 鍵管理 |
| 技術スタック ADR | `dev-journal/deliverables/docs/30_arch/adr/0001-tech-stack.md` | golang-jwt 選定 |
| JWT 実装 | `expense-saas/internal/pkg/jwt/jwt.go` | RS256 固定実装 |

## 責務

- JWT 署名アルゴリズムとして RS256 を維持する判断根拠を記録する
- 検討した選択肢（RS256, EdDSA, ML-DSA）の比較と、各方式を採用しなかった理由を記載する
- NIST IR 8547 の 2030 年非推奨ガイドラインに対する本プロジェクトの立場を明記する
- 含めないこと: 実装詳細（security.md に委譲）、図（diagrams.md に委譲）

## 完了条件

- 「なぜ RS256 を維持するか」が判断根拠として読める
- EdDSA, ML-DSA を不採用とした理由が明記されている
- 末尾に「反映先: architecture.md §X」を記載している
- issue 050 のクローズ根拠として参照可能である
