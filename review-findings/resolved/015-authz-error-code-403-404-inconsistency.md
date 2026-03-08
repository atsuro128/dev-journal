# 015: 認可エラーの 403/404 仕様が文書間で不整合

## 指摘概要
認可失敗時のHTTPステータス定義が `requirements.md` と `rbac.md` / `security-policy.md` で一致していない。

## 根拠
- `requirements.md`:
  - `RBC-004` 権限不足時は 403
  - `deliverables/docs/10_requirements/requirements.md:96`
- `rbac.md`:
  - 他テナントリソースは 404 Not Found
  - `deliverables/docs/10_requirements/rbac.md:194`
- `rules/security-policy.md`:
  - 権限不足: 403
  - 他テナントリソース: 404
  - `rules/security-policy.md:39`
  - `rules/security-policy.md:40`

## 判定
セキュリティ要件の不整合（中）。

## 修正方針案
`requirements.md` にケース別の返却コード（403/404）を明記し、RBACルールIDを分離して統一する。

---

## 再レビュー結果（2026-03-05）
対応不十分（差し戻し）。

### 確認内容
- `requirements.md` の RBAC ルールは引き続き `RBC-004: 権限不足時は 403 Forbidden` のみ
  - `deliverables/docs/10_requirements/requirements.md:96`
- `requirements.md` 内に「テナント境界越えアクセスは 404」を明示した記述は未追加
- `rbac.md` / `rules/security-policy.md` 側の 404 方針は維持

### 差し戻し理由
認可失敗時のケース分岐（同一テナント内の権限不足=403、テナント境界越え=404）が `requirements.md` で明文化されておらず、文書間で解釈差が残るため。

### 対応内容に対する見解（追記）
`progress-management/pending-review/015-authz-error-code-403-404-inconsistency.md` の「対応不要」判断は確認済み。  
そのうえで、以下の理由により差し戻し判定を維持する。

- 「`requirements.md` はロールベース認可の403のみを定義し、404は `rbac.md` 側に委譲」という主張:
  - `requirements.md` の `RBC-004` は「権限不足時は 403」と包括表現であり、同一テナント内に限定する注記がない。
  - そのため、読者によってはテナント境界越えも 403 対象と解釈しうる。
- 「詳細は `rbac.md` 参照のため不整合ではない」という主張:
  - 参照委譲自体は妥当だが、参照元（`requirements.md`）にある規範文が参照先の例外（404）を包含してしまっている点が問題。
  - 最低限、`requirements.md` 側に「テナント境界越えは 404（存在漏洩防止）」の一文が必要。

結論: 層別分離の意図は理解できるが、文言上の曖昧性が残るため「対応完了」とは判定できない。

---

## 再レビュー結果（2026-03-05 / 2回目）
対応妥当（クローズ）。

### 確認内容
- `requirements.md` の `RBC-004` が「同一テナント内の権限不足時は 403」に修正済み
  - `deliverables/docs/10_requirements/requirements.md:96`
- `requirements.md` に「テナント境界越えアクセスは 404」を `TNT-006` として追加済み
  - `deliverables/docs/10_requirements/requirements.md:123`
- `rbac.md` / `rules/security-policy.md` の 404 方針と整合

### 判定理由
当初の曖昧性（403の適用範囲未限定）が解消され、要件本体で 403/404 のケース分岐が明文化されたため。
