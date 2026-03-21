# 038: preliminary business overview still excludes Accounting self-application

## 指摘概要
Issue 024 では Accounting に Member 相当の申請権限を付与する方針へ更新されたが、Step 1 の上流整理文書である `preliminary/01_business-overview.md` だけが Accounting を「承認済み経費の確認・支払処理」の役割として据えたままで、自己申請可能なロールとして説明されていない。`rbac.md`・`requirements.md`・`usecases.md` ではすでに「自分の経費申請が可能」で統一されており、Step 1 内のトレーサビリティが崩れている。

## 根拠
- [01_business-overview.md](/root-project/dev-journal/deliverables/docs/10_requirements/preliminary/01_business-overview.md#L155) Accounting の主な責務が「承認済みの経費を確認し、支払処理を行い、会計システムに記録する」となっており、自己申請可能な説明がない
- [01_business-overview.md](/root-project/dev-journal/deliverables/docs/10_requirements/preliminary/01_business-overview.md#L173) ロール間の情報の流れでも Accounting は承認後の支払処理担当としてのみ登場している
- [rbac.md](/root-project/dev-journal/deliverables/docs/10_requirements/rbac.md#L43) Accounting は「経費の申請・支払処理」を担い、自分のレポートの作成・編集・削除・提出が可能と定義されている
- [requirements.md](/root-project/dev-journal/deliverables/docs/10_requirements/requirements.md#L91) Accounting のロール定義は「経費の申請・確認・支払処理」に更新済み

## 判定
低 / Step 1 文書内の整合性不備

## 修正方針案
`preliminary/01_business-overview.md` の Accounting 説明と業務フローを、最終要件と同じく「自分の経費申請も行うが、自分のレポートの支払完了は記録できない」前提へ更新する。
