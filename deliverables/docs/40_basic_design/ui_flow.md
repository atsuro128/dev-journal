# 画面遷移図

## この文書の役割

| 項目 | 内容 |
|------|------|
| 目的 | 画面遷移と認証状態・ロールによる導線を定義する |
| 正本情報 | 遷移パターン、認証状態別分岐、ロール別導線 |
| 扱わない内容 | 画面詳細仕様、API エラー詳細 |
| 主な参照元 | `10_requirements/usecases.md`, `40_basic_design/screens.md` |
| 主な参照先 | `50_detail_design/screens/*.md`, `60_test/test_cases/*.md` |

## 1. 概要

本書は、経費精算SaaS MVP の画面遷移を Mermaid 図で定義する。
ロール別の遷移パスと主要フロー（提出 → 承認 → 支払完了）を明確にする。

### 参照ドキュメント

| ドキュメント | 役割 |
|------------|------|
| `40_basic_design/screens.md` | 画面一覧・画面ID |
| `10_requirements/usecases.md` | ユースケース |
| `10_requirements/policies.md` | 状態遷移（SS4）、RBAC 権限マトリクス（SS3） |

---

## 2. 全体画面遷移図

```mermaid
graph TD
    subgraph 未認証
        AUTH001[SCR-AUTH-001<br/>サインアップ]
        AUTH002[SCR-AUTH-002<br/>ログイン]
        AUTH003[SCR-AUTH-003<br/>パスワードリセット要求]
        AUTH004[SCR-AUTH-004<br/>パスワードリセット実行]
    end

    subgraph 認証済み共通
        DASH001[SCR-DASH-001<br/>ダッシュボード]
    end

    subgraph 経費レポート
        RPT001[SCR-RPT-001<br/>レポート一覧 - 自分]
        RPT002[SCR-RPT-002<br/>レポート作成]
        RPT003[SCR-RPT-003<br/>レポート編集]
        RPT004[SCR-RPT-004<br/>レポート詳細]
    end

    subgraph ワークフロー
        WFL001[SCR-WFL-001<br/>承認待ち一覧]
        WFL002[SCR-WFL-002<br/>支払待ち一覧]
    end

    subgraph 管理
        ADM001[SCR-ADM-001<br/>テナント全レポート一覧]
        ADM002[SCR-ADM-002<br/>テナント情報]
    end

    %% 未認証フロー
    AUTH001 -->|サインアップ成功| DASH001
    AUTH001 -->|ログインリンク| AUTH002
    AUTH002 -->|ログイン成功| DASH001
    AUTH002 -->|サインアップリンク| AUTH001
    AUTH002 -->|パスワードを忘れた方| AUTH003
    AUTH003 -->|メール送信完了| AUTH002
    AUTH004 -->|リセット完了| AUTH002

    %% ダッシュボードからの遷移
    DASH001 -->|マイレポート| RPT001
    DASH001 -->|レポート作成| RPT002
    DASH001 -->|承認待ちバッジ| WFL001
    DASH001 -->|支払待ちバッジ| WFL002
    DASH001 -->|直近レポート選択| RPT004
    DASH001 -->|テナント全レポート| ADM001
    DASH001 -->|テナント情報| ADM002

    %% レポート系の遷移
    RPT001 -->|レポート選択| RPT004
    RPT001 -->|新規作成| RPT002
    RPT002 -->|作成完了| RPT004
    RPT003 -->|保存完了| RPT004
    RPT004 -->|編集ボタン| RPT003
    RPT004 -->|再申請| RPT002

    %% ワークフロー系の遷移
    WFL001 -->|レポート選択| RPT004
    WFL002 -->|レポート選択| RPT004

    %% 管理系の遷移
    ADM001 -->|レポート選択| RPT004

    %% ログアウト（全認証済み画面から）
    DASH001 -.->|ログアウト| AUTH002
```

> ※ ログアウトは全認証済み画面の共通ヘッダーから実行可能（SCR-AUTH-002 に遷移）。遷移図では代表として SCR-DASH-001 からの点線のみ描画。

---

## 3. ロール別の遷移パス

### 3.1 Member の遷移パス

Member は経費レポートの作成・編集・提出・状況確認を行う。

```mermaid
graph TD
    AUTH002[ログイン] --> DASH001[ダッシュボード<br/>下書き数/提出中数/却下数<br/>直近レポート一覧]

    DASH001 --> RPT001[レポート一覧 - 自分]
    DASH001 --> RPT002[レポート作成]

    RPT001 --> RPT004[レポート詳細]
    RPT002 -->|作成完了| RPT004

    RPT004 -->|編集| RPT003[レポート編集]
    RPT003 -->|保存| RPT004

    RPT004 -->|明細追加/編集/削除| RPT004
    RPT004 -->|領収書添付/削除| RPT004
    RPT004 -->|提出| RPT004
    RPT004 -->|削除| RPT001

    RPT004 -->|再申請<br/>rejected時| RPT002

    style DASH001 fill:#e3f2fd
    style RPT001 fill:#e3f2fd
    style RPT002 fill:#e3f2fd
    style RPT003 fill:#e3f2fd
    style RPT004 fill:#e3f2fd
```

**主要フロー: レポート作成 → 提出**

```
ダッシュボード → レポート作成 → レポート詳細（明細追加・領収書添付） → 提出
```

**却下後の再申請フロー:**

```
レポート詳細（rejected） → 却下理由確認 → 再申請ボタン → レポート作成（内容コピー） → レポート詳細 → 提出
```

### 3.2 Approver の遷移パス

Approver は Member としての操作に加え、承認・却下を行う。

```mermaid
graph TD
    AUTH002[ログイン] --> DASH001[ダッシュボード<br/>Member情報 + 承認待ち数<br/>月別支出サマリー]

    %% Member としての操作
    DASH001 --> RPT001[レポート一覧 - 自分]
    DASH001 --> RPT002[レポート作成]
    RPT001 --> RPT004[レポート詳細]
    RPT002 -->|作成完了| RPT004

    %% 承認者としての操作
    DASH001 -->|承認待ちバッジ| WFL001[承認待ち一覧]
    WFL001 --> RPT004

    RPT004 -->|承認<br/>コメント任意| RPT004
    RPT004 -->|却下<br/>理由入力| RPT004

    style DASH001 fill:#e8f5e9
    style WFL001 fill:#e8f5e9
    style RPT001 fill:#e3f2fd
    style RPT002 fill:#e3f2fd
    style RPT004 fill:#e3f2fd
```

**主要フロー: 承認**

```
ダッシュボード → 承認待ち一覧 → レポート詳細（明細・領収書確認） → 承認（コメント任意）
```

**却下フロー:**

```
ダッシュボード → 承認待ち一覧 → レポート詳細 → 却下（理由入力）
```

> 自分のレポートの承認・却下はできない（自己承認禁止）。レポート詳細画面で承認・却下ボタンは非表示になる。

### 3.3 Accounting の遷移パス

Accounting は Member としての経費申請と、支払完了の記録を行う。

```mermaid
graph TD
    AUTH002[ログイン] --> DASH001[ダッシュボード<br/>Member情報 + 支払待ち数<br/>月別支出サマリー]

    %% Member としての操作（経費申請）
    DASH001 --> RPT001[レポート一覧 - 自分]
    DASH001 --> RPT002[レポート作成]
    RPT001 --> RPT004[レポート詳細]
    RPT002 -->|作成完了| RPT004

    RPT004 -->|編集| RPT003[レポート編集]
    RPT003 -->|保存| RPT004

    RPT004 -->|明細追加/編集/削除| RPT004
    RPT004 -->|領収書添付/削除| RPT004
    RPT004 -->|提出| RPT004
    RPT004 -->|削除| RPT001
    RPT004 -->|再申請<br/>rejected時| RPT002

    %% 経理担当としての操作（支払処理）
    DASH001 -->|支払待ちバッジ| WFL002[支払待ち一覧]
    DASH001 --> ADM001[テナント全レポート一覧]

    WFL002 --> RPT004
    ADM001 --> RPT004

    RPT004 -->|支払完了<br/>自分のレポート以外| RPT004

    style DASH001 fill:#fce4ec
    style WFL002 fill:#fce4ec
    style ADM001 fill:#fce4ec
    style RPT001 fill:#e3f2fd
    style RPT002 fill:#e3f2fd
    style RPT003 fill:#e3f2fd
    style RPT004 fill:#e3f2fd
```

**主要フロー: 経費申請（Member 操作）**

```
ダッシュボード → レポート作成 → レポート詳細（明細追加・領収書添付） → 提出
```

**主要フロー: 支払完了**

```
ダッシュボード → 支払待ち一覧 → レポート詳細（明細・金額確認） → 支払完了
```

> Accounting は Member としての経費申請も行うため、サイドナビゲーションに「マイレポート」「レポート作成」を表示する。
> 自分が作成したレポートの支払完了は記録できない（自己処理禁止）。レポート詳細画面で該当レポートの支払完了ボタンは非表示になる。

### 3.4 Admin の遷移パス

Admin はテナント管理と全レポート閲覧、および自分の経費提出を行う。

```mermaid
graph TD
    AUTH002[ログイン] --> DASH001[ダッシュボード<br/>テナント全体の概況<br/>月別支出サマリー]

    %% 管理者としての操作
    DASH001 --> ADM001[テナント全レポート一覧]
    DASH001 --> ADM002[テナント情報]
    ADM001 --> RPT004[レポート詳細]

    %% 申請者としての操作（自分のレポートのみ）
    DASH001 --> RPT001[レポート一覧 - 自分]
    DASH001 --> RPT002[レポート作成]
    RPT001 --> RPT004
    RPT002 -->|作成完了| RPT004

    style DASH001 fill:#fff9c4
    style ADM001 fill:#fff9c4
    style ADM002 fill:#fff9c4
    style RPT001 fill:#e3f2fd
    style RPT002 fill:#e3f2fd
    style RPT004 fill:#e3f2fd
```

**管理フロー:**

```
ダッシュボード → テナント全レポート一覧 → レポート詳細（閲覧のみ）
```

> Admin は他者のレポートは閲覧のみ。編集・承認・却下は不可（rbac.md 4.3 特殊ルール準拠）。

---

## 4. 主要業務フローと画面遷移の対応

### 4.1 正常系（Happy Path）: 提出 → 承認 → 支払完了

```mermaid
sequenceDiagram
    participant M as Member
    participant A as Approver
    participant AC as Accounting

    Note over M: SCR-DASH-001 ダッシュボード
    M->>M: SCR-RPT-002 レポート作成
    M->>M: SCR-RPT-004 レポート詳細（明細追加・領収書添付）
    M->>M: SCR-RPT-004 提出操作（確認ダイアログ）
    Note over M: draft → submitted

    Note over A: SCR-DASH-001 ダッシュボード（承認待ちバッジ）
    A->>A: SCR-WFL-001 承認待ち一覧
    A->>A: SCR-RPT-004 レポート詳細（明細・領収書確認）
    A->>A: SCR-RPT-004 承認操作（確認ダイアログ・コメント任意）
    Note over A: submitted → approved

    Note over AC: SCR-DASH-001 ダッシュボード（支払待ちバッジ）
    AC->>AC: SCR-WFL-002 支払待ち一覧
    AC->>AC: SCR-RPT-004 レポート詳細（金額確認）
    AC->>AC: SCR-RPT-004 支払完了操作（確認ダイアログ）
    Note over AC: approved → paid
```

### 4.2 却下 → 再申請フロー

```mermaid
sequenceDiagram
    participant M as Member
    participant A as Approver

    Note over M: レポート提出済み（submitted）
    Note over A: SCR-WFL-001 承認待ち一覧
    A->>A: SCR-RPT-004 レポート詳細
    A->>A: 却下操作（理由入力 + 確認ダイアログ）
    Note over A: submitted → rejected

    Note over M: SCR-RPT-001 レポート一覧（rejectedバッジ表示）
    M->>M: SCR-RPT-004 レポート詳細（却下理由を確認）
    M->>M: 「再申請」ボタン押下
    M->>M: SCR-RPT-002 レポート作成（元レポートの内容がコピーされた状態）
    M->>M: SCR-RPT-004 レポート詳細（内容修正・領収書再添付）
    M->>M: 提出操作
    Note over M: 新規レポート: draft → submitted
```

---

## 5. 認証状態による遷移制御

### 5.1 未認証ユーザーのガード

認証が必要な画面（SCR-DASH-001 以降の全画面）に未認証状態でアクセスした場合、ログイン画面（SCR-AUTH-002）にリダイレクトする。

```mermaid
graph LR
    A[未認証で /dashboard にアクセス] --> B{JWTトークン<br/>存在するか?}
    B -->|なし| C[SCR-AUTH-002 ログインにリダイレクト]
    B -->|あり| D{トークン<br/>有効か?}
    D -->|有効| E[画面を表示]
    D -->|期限切れ| F{リフレッシュ<br/>トークン}
    F -->|成功| E
    F -->|失敗| C
```

### 5.2 認証済みユーザーのガード

認証済みユーザーが未認証画面（ログイン・サインアップ）にアクセスした場合、ダッシュボード（SCR-DASH-001）にリダイレクトする。

### 5.3 ロールによるアクセス制御

権限のない画面にアクセスした場合の挙動:

| 状況 | 挙動 |
|------|------|
| Member が /approvals にアクセス | ダッシュボードにリダイレクト |
| Member が /payments にアクセス | ダッシュボードにリダイレクト |
| Member が /reports/all にアクセス | ダッシュボードにリダイレクト |
| Member が /settings/tenant にアクセス | ダッシュボードにリダイレクト |
| Accounting が /approvals にアクセス | ダッシュボードにリダイレクト |
| Approver が /payments にアクセス | ダッシュボードにリダイレクト |

> 権限のないページへのアクセスは、403画面を表示するのではなく、ダッシュボードにリダイレクトする方針とする。これは、サイドナビゲーションでメニュー自体が非表示であるため、URL直接入力でのみ発生するケースであり、ユーザー体験を考慮した判断である。

---

## 6. 品質チェック

- [x] 主要フロー（提出 → 承認 → 支払完了）が画面遷移で表現されているか
- [x] 却下 → 再申請フローが画面遷移で表現されているか
- [x] 4ロール全ての遷移パスが定義されているか
- [x] 未認証/認証済みの遷移制御が定義されているか
- [x] ロール別のアクセス制御が定義されているか
- [x] screens.md の画面IDと一致しているか
- [x] 用語が glossary.md に準拠しているか（提出/却下/再申請/支払完了）
