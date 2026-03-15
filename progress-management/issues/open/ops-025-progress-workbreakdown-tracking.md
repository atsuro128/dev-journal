# progress.md と work-breakdown の進捗管理方針の策定

## 発見日
2026-03-15

## カテゴリ
project-management

## 影響度
中

## 発見経緯
proactive

## 関連ステップ
全 Step（横断）

## 問題

progress.md は Step 単位で進捗管理している。work-breakdown でタスク単位に分解したことで、タスクレベルの進捗をどこで管理するかが未定義。work-breakdown 内の「状態」列と progress.md の二重管理になるリスクがある。

### 選択肢（未決定）

- A: progress.md にタスク行を追加（一元管理、ただし肥大化リスク）
- B: progress.md は Step 単位のまま、タスク進捗は work-breakdown 内で管理（二重管理だがコンパクト）

## 影響

- 方針未定のまま作業を進めると、進捗管理が属人的になる
- Step 7 のようにタスク数が多い Step では管理破綻の可能性

## 提案

Step 3 で小規模に試し、運用実績を踏まえて方針を確定する。

---

## 解決内容

## 解決日
