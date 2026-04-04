# 30_arch テンプレート・ガイドの文書責務境界ルール不足

## 発見日
2026-04-04

## カテゴリ
ai-ops

## 影響度
中

## 発見経緯
review

## 関連ステップ
Step 3（アーキテクチャ設計）

## ブロッカー
なし

## 問題

issue 051 対応時の codex レビューで、30_arch 配下の文書責務逸脱が6件見つかった。調査の結果、5件はテンプレート・ガイドの不足が再発要因であることが判明した。

### 根本原因

step3-architecture.md（work-breakdown）は責務境界を十分に定義しているが、その制約が docs-v2/30_arch のテンプレートに継承されていない。さらに ADR テンプレートが三層構造（ADR-template.md / docs-v2/30_arch/adr/*.md / step3-architecture.md）になっており、境界ルールの伝播漏れが起きている。

### テンプレート起因の指摘一覧

| # | 問題 | 主因 |
|---|------|------|
| 1 | ADR-0003 に設計仕様（適用対象テーブル、SQL、接続フロー）が入っている | テンプレート不足 + 作成判断ミス |
| 2 | architecture.md に代替案比較・採否理由が入っている（L229-233、ADR-0004 と重複） | テンプレート起因 |
| 3 | architecture.md にテナント分離の ASCII 図が埋め込まれている（L168-184、diagrams.md §4 と重複） | ガイド不足（Mermaid のみ禁止、ASCII 図は未禁止） |
| 4 | architecture.md に SPA 配信のブロック図が埋め込まれている（L235-244、diagrams.md §6 と重複） | ガイド不足 |

## 影響

- AI エージェントがテンプレートだけ見て成果物を作成した場合、同じ責務逸脱が再発する
- 文書間で正本が不明になり、メンテナンス時に不整合が生じる

## 提案

### A. テンプレート修正

1. **architecture.md テンプレート**（`templates/docs-v2/30_arch/architecture.md`）
   - 「扱わない内容」に「代替案比較表、採否理由、選定根拠の本文」を追加
   - 「図表現全般（Mermaid、ASCII、ブロック図）は埋め込まない。diagrams.md を参照」を追加

2. **ADR テンプレート**（`templates/ADR-template.md`）
   - 「決定」セクションに「実装 SQL/DDL、処理フロー、接続手順は書かず、反映先に委譲」を追加

3. **docs-v2/30_arch/adr/*.md**
   - ADR-template.md の制約を継承するか、正本を一本化する

4. **diagrams.md テンプレート**（`templates/docs-v2/30_arch/diagrams.md`）
   - 「図の正本は本書のみ。architecture.md には図を再掲しない」を追加

### B. ガイド修正

5. **step3-architecture.md**
   - 「含めない内容」の図禁止を「Mermaid 図」→「図表現全般」に拡張

### C. 成果物修正（上記テンプレート修正後に実施）

6. architecture.md の代替案比較表を ADR-0004 に移動
7. architecture.md の ASCII 図を diagrams.md 参照に差し替え
8. ADR-0003 の設計仕様（適用対象テーブル、SQL、接続フロー）を architecture.md または Step 5 に移動

---

## 解決内容
テンプレート・ガイド・成果物を一括修正:
- ADR テンプレート5ファイル→1ファイルに統合、責務境界ルール追加
- architecture.md/diagrams.md テンプレートを成果物ベースに再整備
- workflow.md に成果物テンプレート参照の共通導線追加
- step3〜6 のテンプレート参照・プロセスセクション整理
- 成果物: ADR-0003 設計仕様移動、比較表・ASCII 図の差し替え、forward reference 修正
- 既存成果物のテンプレート準拠残件は ops-056 に分離

## 解決日
2026-04-04
