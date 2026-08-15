# 単元参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - タスクに紐づく単元を個別に取得すること。
- システム上の役割
  - タスクと単元の関連性を確認し、安全に単元データを返却する。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|詳細取得|タスクに紐づく単元情報を取得する|

---

# 3. 業務フロー

1. 生徒がタスク内の単元詳細を要求する。
2. システムは該当タスクと単元の紐づきを検証する。
3. 単元データを返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::UnitsController`|`show`|GET|`/api/v1/student/tasks/:task_id/units/:id`|タスクに紐づく単元詳細を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::UnitsController#show`

### 概要

タスクに紐づく単元の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|
|`id`|必須|integer|単元 ID|

### 処理内容

1. `current_user.tasks.find(task_id)` で対象タスクを取得する。
2. そのタスクに紐づく単元から `id` を検索する。
3. `UnitSerializer` で JSON を返却する。

### 業務ルール

- タスクに紐づかない単元は取得できない。

### Database変更

- なし

### Response

- `id`
- `course_id`
- `unit_name`
- `course`:
  - `id`
  - `level_number`
  - `level_name`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象タスクまたは対象単元が存在しない|404|対象データが見つかりません|

---

# 6. データモデル

## `units`

- 役割: 学習単元を管理するテーブル
- 主なカラム: `id`, `course_id`, `unit_name`
- リレーション: `belongs_to :course`, `has_many :tasks, through: :task_units`

---

# 7. 状態管理

- なし

---

# 8. 権限制御

- `current_user.tasks` を起点としており、生徒のタスクに紐づく単元のみアクセス可能。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::UnitsController`|
|Serializer|`UnitSerializer`|
|Model|`Unit`, `Course`|
