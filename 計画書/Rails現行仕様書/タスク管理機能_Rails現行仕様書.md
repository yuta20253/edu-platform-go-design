# タスク管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒が自分の学習タスクを登録・管理し、関連する目標や単元と合わせて確認すること。
- システム上の役割
  - タスクの一覧表示、詳細表示、作成、更新を提供する。
  - タスクに紐づく単元情報を含めて返却する。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|生徒のタスクをステータス絞り込みと期日昇順で取得する|
|詳細取得|指定タスクの詳細と単元情報を取得する|
|新規登録|タスクを作成し目標や単元を紐づける|
|更新|タスク情報と紐づく単元を変更する|

---

# 3. 業務フロー

1. 生徒がタスク一覧画面にアクセスする。
2. システムは生徒のタスクを status で絞り込み、`due_date` 昇順で 5 件ずつ取得する。
3. 生徒がタスクを選択すると、タスクの詳細と関連単元情報を取得する。
4. 生徒がタスク情報を入力し、作成または更新を実行する。
5. 入力値をバリデーションする。
6. 妥当であれば `tasks` テーブルへ登録または更新し、関連単元との関係を同期する。
7. 結果を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::TasksController`|`index`|GET|`/api/v1/student/tasks`|生徒のタスク一覧を取得|
|`Api::V1::Student::TasksController`|`show`|GET|`/api/v1/student/tasks/:id`|タスク詳細と単元情報を取得|
|`Api::V1::Student::TasksController`|`create`|POST|`/api/v1/student/tasks`|タスクを作成|
|`Api::V1::Student::TasksController`|`update`|PATCH|`/api/v1/student/tasks/:id`|タスクを更新|

---

# 5. API / 処理詳細

## `Api::V1::Student::TasksController#index`

### 概要

ログイン中の生徒のタスク一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`status`|任意|string|タスクの状態フィルタ（`not_started`, `in_progress`, `completed`）|
|`page`|任意|integer|ページ番号|

### 処理内容

1. `current_user.tasks` を対象に `by_status` を適用する。
2. `units: :course` を含めて関連データを読み込む。
3. `due_date` 昇順で並べる。
4. 1ページあたり 5 件でページングする。
5. `TaskSerializer` で JSON を返却する。

### 業務ルール

- `status` が未指定の場合は `not_started` または `in_progress` のタスクのみ取得する。
- 期限日は昇順で並べる。

### Database変更

- なし

### Response

- `tasks`:
  - `id`
  - `user_id`
  - `goal_id`
  - `title`
  - `content`
  - `due_date`
  - `priority`
  - `status`
  - `completed_at`
  - `units`
- `meta`:
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|特定なし|--|認証前提|

## `Api::V1::Student::TasksController#show`

### 概要

指定タスクの詳細と紐づく単元情報を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|タスクの識別子|

### 処理内容

1. `current_user.tasks` から指定 ID のタスクを検索する。
2. `units: :course` を含めて関連データを読み込む。
3. `TaskDetailSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身のタスクのみ取得できる。
- タスクに紐づく単元情報を含めて返却する。

### Database変更

- なし

### Response

- `id`
- `user_id`
- `goal_id`
- `title`
- `content`
- `due_date`（`YYYY/MM/DD` 形式）
- `priority`
- `status`
- `completed_at`（`YYYY/MM/DD` 形式）
- `units`（各単元は `UnitDetailSerializer` 形式）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象のタスクが存在しない|404|対象タスクなし|
|他ユーザーのタスクにアクセス|404|対象タスクなし|

## `Api::V1::Student::TasksController#create`

### 概要

生徒が新しいタスクを登録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task[goal_id]`|必須|integer|紐づく目標の ID|
|`task[title]`|必須|string|タスクのタイトル|
|`task[content]`|必須|string|タスクの詳細|
|`task[priority]`|必須|string|優先度|
|`task[due_date]`|必須|string|期限日|
|`task[memo]`|任意|string|メモ|
|`task[unit_ids]`|任意|array|紐づける単元の ID 一覧|

### 処理内容

1. `Student::CreateTaskForm` を `current_user` と入力で初期化する。
2. バリデーションを実行する。
3. `Student::CreateTaskService` でタスクを作成する。
4. `tasks` レコードを作成し、指定された単元を紐づける。

### 業務ルール

- `goal_id`, `title`, `content`, `due_date`, `priority` は必須。
- `goal_id` は現行生徒の目標である必要がある。
- `unit_ids` が指定された場合、すべての単元 ID が存在する必要がある。

### Database変更

- `tasks` に新規レコード作成
- `task_units` に紐づき単元レコードを追加

### Response

- `message`: `タスクが作成されました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|
|`goal_id` が存在しない|422|`goal_id` に無効な値|
|`unit_ids` に不正な値が含まれる|422|`unit_ids` にエラー|

## `Api::V1::Student::TasksController#update`

### 概要

生徒が既存タスクを更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task[title]`|必須|string|タスクのタイトル|
|`task[content]`|必須|string|タスクの詳細|
|`task[priority]`|必須|string|優先度|
|`task[due_date]`|必須|string|期限日|
|`task[memo]`|任意|string|メモ|
|`task[unit_ids]`|任意|array|紐づける単元の ID 一覧|

### 処理内容

1. `current_user.tasks` から指定 ID のタスクを検索する。
2. `Student::UpdateTaskForm` で入力を検証する。
3. `Student::UpdateTaskService` でタスクと単元紐付けを同期する。

### 業務ルール

- `title`, `content`, `due_date`, `priority` は必須。
- `unit_ids` が提供された場合、学習開始済みの単元を削除できない。
- `priority` は整数値または優先度文字列として受け取る。

### Database変更

- 既存の `tasks` レコード更新
- `task_units` の追加・削除で単元紐付けを同期

### Response

- `message`: `タスクが更新されました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|
|学習開始済み単元を削除しようとした|422|`unit_ids` にエラー|
|対象のタスクが存在しない|404|対象タスクなし|

---

# 6. データモデル

## `tasks`

- 役割: 生徒の学習タスクを記録するテーブル
- 主なカラム:
  - `id`
  - `user_id`
  - `goal_id`
  - `title`
  - `content`
  - `priority`
  - `due_date`
  - `estimated_time`
  - `status`
  - `memo`
  - `completed_at`
  - `deleted_at`
- リレーション:
  - `belongs_to :user`
  - `belongs_to :goal`
  - `has_many :task_units`
  - `has_many :units, through: :task_units`
  - `has_many :task_courses`
  - `has_many :courses, through: :task_courses`
  - `has_many :question_histories`

---

# 7. 状態管理

- `status` を enum で管理する。
- 定義:
  - `not_started`
  - `in_progress`
  - `completed`
- `completed` 状態になると `completed_at` を自動設定し、それ以外では `completed_at` をクリアする。

---

# 8. 権限制御

- すべて `current_user.tasks` を起点に検索する。
- 生徒は自分のタスクのみ閲覧・更新できる。

---

# 9. 非同期処理

- `Task` 作成・更新に直接紐づく Job / Mailer は見当たらない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::TasksController`|
|Form|`Student::CreateTaskForm`, `Student::UpdateTaskForm`|
|Service|`Student::CreateTaskService`, `Student::UpdateTaskService`|
|Model|`Task`|
|Serializer|`TaskSerializer`, `TaskDetailSerializer`|
|Form Concern|`UnitIdsValidatable`, `PriorityCastable`|
