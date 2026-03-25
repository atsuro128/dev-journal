# Step 6 テスト設計 — 作業計画

## 判断ポイント（確定済み）

| # | 論点 | 決定 |
|---|------|------|
| 1 | カバレッジ基準 | `.claude/rules/testing.md` の既定値（ドメイン層80%、その他ベストエフォート）を採用 |
| 2 | CI/CD パイプライン設計の扱い | テスト戦略では「方針」のみ記載（PR時に単体+統合、マージ時にE2E）。パイプラインYAML設計は Step 7 |
| 3 | レビュー単位 | Phase ごと（Step 5 と同様） |
| 4 | E2E テスト粒度 | 主要フローのみ（申請→承認→支払完了、却下→再申請） |
| 5 | フィクスチャ設計 | test_strategy.md で標準フィクスチャを一元定義。各テストケースはそれを参照 |

## Phase 構成

### 各 Phase の進め方

**実行前:**
1. 並列タスクがある場合、共通ルール・判断ポイントの適用箇所・タスクごとの要点を整理する
2. 整理した内容をサブエージェントのプロンプトに含めて委譲する

**実行後:**
1. **内部レビュー**: Phase 成果物の整合性チェック
   - 下流が困る問題があれば修正 → 再レビュー（PASS まで繰り返す）
2. **ユーザーにコミットを提案**
3. **codex レビュー**（/codex-review）: コミット後に実行
   - 指摘があれば対応 → 再レビュー（LGTM まで繰り返す）
4. 次の Phase に進む

### Phase 1（逐次）: テスト戦略

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 6-A: テスト戦略策定 | `test_strategy.md` | authz.md, db_schema.md, openapi.yaml, state_machine.md, security.md, rbac.md, files.md, `.claude/rules/testing.md` | test-designer | 未着手 |
| 6-A-R1: 内部レビュー | — | — | test-reviewer | 未着手 |
| 6-A-R2: codex レビュー | — | — | codex | 未着手 |

成果物配置先: `dev-journal/deliverables/docs/60_test/`

### Phase 2（7タスク並列）: 機能別テストケース

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 6-B-1: 認証TC | `test_cases/auth.md` | test_strategy.md, openapi.yaml §/api/auth/*, security.md, authz.md §認証 | test-designer | 未着手 |
| 6-B-2: レポートTC | `test_cases/reports.md` | test_strategy.md, openapi.yaml §/api/reports/*, state_machine.md, authz.md §レポート | test-designer | 未着手 |
| 6-B-3: 明細TC | `test_cases/items.md` | test_strategy.md, openapi.yaml §/api/reports/{id}/items/*, authz.md §明細 | test-designer | 未着手 |
| 6-B-4: 添付TC | `test_cases/attachments.md` | test_strategy.md, openapi.yaml §attachments, files.md, authz.md §添付 | test-designer | 未着手 |
| 6-B-5: ワークフローTC | `test_cases/workflow.md` | test_strategy.md, openapi.yaml §/api/workflow/*, state_machine.md, authz.md §承認 | test-designer | 未着手 |
| 6-B-6: ダッシュボード・カテゴリTC | `test_cases/dashboard.md` | test_strategy.md, openapi.yaml §/api/dashboard, §/api/categories | test-designer | 未着手 |
| 6-B-7: テナント管理TC | `test_cases/tenant.md` | test_strategy.md, openapi.yaml §/api/tenant/*, authz.md §テナント管理 | test-designer | 未着手 |
| 6-B-R1: 内部レビュー | — | — | test-reviewer | 未着手 |
| 6-B-R2: codex レビュー | — | — | codex | 未着手 |

成果物配置先: `dev-journal/deliverables/docs/60_test/test_cases/`

### Phase 3（逐次）: 横断テストケース

| タスク | 成果物 | 入力（主要） | エージェント | 状態 |
|--------|--------|-------------|------------|------|
| 6-B-8: 横断TC | `test_cases/cross-cutting.md` | 6-B-1〜7 の全テストケースファイル, authz.md, rbac.md | test-designer | 未着手 |
| 6-B-8-R1: 内部レビュー | — | — | test-reviewer | 未着手 |
| 6-B-8-R2: codex レビュー | — | — | codex | 未着手 |

成果物配置先: `dev-journal/deliverables/docs/60_test/test_cases/`

## 依存グラフ

```
Step 5 完了（確認済み）
  └→ Phase 1: 6-A (テスト戦略)
       └→ Phase 2 (7タスク並列):
            ├→ 6-B-1 (認証TC)          ─┐
            ├→ 6-B-2 (レポートTC)      ─┤
            ├→ 6-B-3 (明細TC)          ─┤
            ├→ 6-B-4 (添付TC)          ─┤→ Phase 3: 6-B-8 (横断TC)
            ├→ 6-B-5 (ワークフローTC)  ─┤
            ├→ 6-B-6 (ダッシュボードTC) ─┤
            └→ 6-B-7 (テナント管理TC)  ─┘
```

## 品質基準

- **下流作業可能性**: Step 7 実装者が各テストケースを迷わずコードに落とし込める粒度で書かれているか（唯一の判定基準）
- **上流整合性**: openapi.yaml の全エンドポイント（31 operationId、health 除く30）、state_machine.md の遷移ルール（T1〜T5, X1〜X10）、rbac.md の権限マトリクスと一致しているか
- **タスク間整合性**: テストID の命名規則が統一され、重複がないか。機能別ファイルと cross-cutting.md の責務境界が守られているか
- **MVP スコープ**: `deliverables/docs/02_scope.md` の範囲内か
- **用語集準拠**: `dev-journal/deliverables/docs/01_glossary.md` の用語を使用しているか
- **完了条件**: work-breakdown に定義された完了条件を全て満たしているか
