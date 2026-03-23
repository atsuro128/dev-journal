---
step: 5
severity: medium
status: open
---

# 051: 同一テナント内の権限不足エラー契約が上流要件と Step 5 で再び分岐している

## 指摘概要
Step 5 の `authz.md` / `security.md` / `openapi.yaml` は、同一テナント内での所有権不足・リソースアクセス権なしを `403 PERMISSION_DENIED` として定義している。一方、上流の `requirements.md` と `rbac.md` は同じケースを `403 Forbidden` と定義したままで、レスポンス例も旧形式のまま残っている。  
この状態では、Step 5 の API 契約は内部では整合しているが、上流要件に対するトレーサビリティが切れており、実装者・テスターがどの 403 エラーコードを正とすべきか文書だけでは確定できない。

## 根拠
- [requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L100)
  - `RBC-004 | 同一テナント内の権限不足時は 403 Forbidden を返却`
- [rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L209)
  - `403 Forbidden | 所有権不足`
- [authz.md](/root-project/dev-journal/deliverables/docs/50_detail_design/authz.md#L660)
  - `403 FORBIDDEN` はロール不足、`403 PERMISSION_DENIED` は所有権不足・リソースアクセス権なしに分離
- [security.md](/root-project/dev-journal/deliverables/docs/50_detail_design/security.md#L539)
  - `FORBIDDEN` / `PERMISSION_DENIED` を別コードとして定義
- [openapi.yaml](/root-project/dev-journal/deliverables/docs/50_detail_design/openapi.yaml#L625)
  - `GET /api/reports/{id}` で同一テナント内の権限不足を `403 PERMISSION_DENIED` と定義

## 判定
中 / 上流整合性の不備

## 修正方針案
以下のいずれかに統一する。

1. Step 5 を正本とし、`requirements.md` と `rbac.md` を `FORBIDDEN = ロール不足`、`PERMISSION_DENIED = 所有権不足・リソース権限不足` に更新する
2. 上流を正本とし、Step 5 側の `PERMISSION_DENIED` を廃止して `FORBIDDEN` に寄せる

現状の Step 5 文脈では 403 の内訳を分けているため、上流文書を Step 5 に合わせて更新する方が自然。
