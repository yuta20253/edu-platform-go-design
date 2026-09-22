# 分析機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒の学習進捗や成績を分析して返却すること。
- システム上の役割
  - 指定された分析タイプに応じて、適切な分析サービスを呼び出す。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|分析取得|学習進捗や成績の分析結果を取得する|

---

# 3. 業務フロー

1. 生徒が分析データを要求する。
2. システムはリクエストの `analytics[type]` を判定する。
3. 対応する分析サービスを生成し、結果を取得する。
4. JSON 形式で返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::AnalyticsController`|`index`|GET|`/api/v1/student/analytics`|分析結果を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::AnalyticsController#index`

### 概要

生徒の学習分析データを取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`analytics[type]`|必須|string|分析タイプ|
|`course_id`|任意|integer|コース指定|
|`unit_id`|任意|integer|単元指定|

### 処理内容

1. `AnalyticsService` を `current_user`, `type`, `course_id`, `unit_id` で初期化する。
2. `call` を実行し、分析サービスの結果を取得する。
3. JSON を返却する。

### 業務ルール

- `type` によって呼び出される分析サービスが変わる。
- サポートされる分析タイプ:
  - `task_completion`
  - `understanding_score`
  - `grade_average`
  - `course_rank`
  - `unit_rank`
- `course_rank`・`unit_rank`は、対象コース・単元に対して生徒が解答した問題の正答率（在籍中の生徒に限り、正答数 ÷ 解答数で算出する値）を比較指標とし、同じコース・単元に解答履歴を持つ生徒集団の中で正答率の高い順に順位付けを行う。理解度スコア（`understanding_score`）や成績平均（`grade_average`）とは独立した算出処理であり、これらの分析結果を合成した指標ではない。

### Database変更

- なし

### Response

- 分析サービスが返す任意の JSON ペイロード

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|指定分析タイプが不正|例外|`InvalidAnalyticsTypeError` を発生させる可能性あり|

---

# 6. データモデル

- 直接のテーブル変更はなし
- 分析サービスは生徒の学習履歴・タスク・単元などの既存データを参照する

---

# 7. 状態管理

- なし

---

# 8. 権限制御

- `current_user` の学習データを前提に分析を行う。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::AnalyticsController`|
|Service|`Student::AnalyticsService`|
|Analysis|`Student::Analytics::TaskCompletion`, `Student::Analytics::UnderstandingScore`, `Student::Analytics::GradeAverage`, `Student::Analytics::Rank`|
