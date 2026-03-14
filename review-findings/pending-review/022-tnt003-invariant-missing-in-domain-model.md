---
step: 2
severity: low
status: pending-review
---

# 022: TNT-003 不変条件が domain_model.md に未定義

## 指摘内容

`domain_model.md` Section 7（実装責務の層別マッピング）のリポジトリ層コメントには「TNT-002, TNT-003」と記載されているが、Section 6.1（テナント分離の不変条件テーブル）に TNT-003 の定義が存在しない。

**該当箇所**（domain_model.md Section 7）:
```
│  - tenant_id フィルタの強制（TNT-002, TNT-003）
```

Section 6.1 の不変条件テーブルには TNT-001, TNT-002, TNT-004, TNT-005 のみが定義されており、TNT-003 が抜けている。

## 影響

- 読み手が TNT-003 の内容を参照できない
- `10_requirements/preliminary/04_business-rules.md` には定義があると思われるが、domain_model.md 単体では意味が追えない

## 対応案

Section 6.1 の不変条件テーブルに TNT-003 を追記する、または参照先を明記する。

## 対応内容

domain_model.md Section 6.1 の不変条件テーブルに TNT-003（`tenant_id` の付与・検証はリポジトリ層で強制）を追記した。
