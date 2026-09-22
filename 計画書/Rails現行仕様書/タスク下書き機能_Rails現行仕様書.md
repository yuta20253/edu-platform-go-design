# タスク下書き機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒が学習タスクを正式に登録する前段階として、内容を下書きの状態で保存できるようにすること。
- システム上の役割
  - 目標に紐づく下書きタスクを作成し、対象の単元を関連付けて保存する。
  - 保存した下書きタスクの詳細を、関連する単元・コース情報とあわせて取得できる。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|詳細取得|下書きタスクの詳細を、紐づく単元・コース情報とあわせて取得する|
|新規作成|下書きタスクを新規に保存する|

---

# 3. 業務フロー

1. 生徒が、下書きタスクとして保存したい目標・タイトル・内容・優先度・期限日・メモ・対象単元を入力する。
2. システムは入力値を検証する。
3. 検証を満たせば、下書きタスクを新規に保存し、指定された単元をこの下書きタスクに関連付ける。
4. 作成した下書きタスクのIDを返却する。
5. 生徒は、保存済みの下書きタスクの詳細を、関連する単元・コース情報とあわせていつでも取得できる。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::DraftTasksController`|`show`|GET|`/api/v1/student/draft_tasks/:id`|下書きタスクの詳細を取得|
|`Api::V1::Student::DraftTasksController`|`create`|POST|`/api/v1/student/draft_tasks`|下書きタスクを新規作成|

補足:

- ルーティング上は `resources :draft_tasks` として一覧取得・更新・削除を含むエンドポイントが定義されているが、現時点で実装されているアクションは `show` と `create` のみであり、一覧取得・更新・削除の機能は提供されていない。

---

# 5. API / 処理詳細

## `Api::V1::Student::DraftTasksController#show`

### 概要

生徒が保存した下書きタスク1件の詳細を、紐づく単元・コース情報とあわせて取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|下書きタスクの識別子|

### 処理内容

1. `current_user` 本人の下書きタスクの中から、指定IDのものを、紐づく単元とそのコース情報を含めて取得する。
2. 下書きタスクの詳細と、紐づく単元一覧（各単元のコース情報を含む）を返却する。

### 業務ルール

- 生徒自身が作成した下書きタスクのみ取得できる。

### Database変更

- なし

### Response

- `id`, `user_id`, `goal_id`, `title`, `content`, `due_date`（`YYYY/MM/DD` 形式）, `priority`, `status`, `completed_at`
- `units`（紐づく単元一覧。各単元は `id`, `course_id`, `unit_name`, `course`（コースの `id`, `level_number`, `level_name`）を含む）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の下書きタスクが存在しない、または自分のものでない|404|対象データなし|

## `Api::V1::Student::DraftTasksController#create`

### 概要

生徒が下書きタスクを新規に保存する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`draft_task[goal_id]`|必須|integer|紐づける目標のID|
|`draft_task[title]`|必須|string|下書きタスクのタイトル|
|`draft_task[content]`|必須|string|下書きタスクの内容|
|`draft_task[priority]`|必須|integer|優先度|
|`draft_task[due_date]`|必須|string|期限日|
|`draft_task[memo]`|任意|string|メモ|
|`draft_task[unit_ids]`|任意|integer[]|関連付ける単元IDの配列|

### 処理内容

1. `Student::CreateDraftTaskForm` を入力値で初期化し、バリデーションを実行する。
2. 指定された `goal_id` が、ログイン中の生徒自身の目標であることを確認する。
3. 期限日が正しい日付形式であることを確認する。
4. 指定された `unit_ids` がすべて実在する単元のIDであることを確認する。
5. 検証を満たせば、下書きタスクを新規に保存する。
6. 指定された単元を、この下書きタスクに関連付けて保存する。
7. 作成した下書きタスクのIDを返却する。

### 業務ルール

- `goal_id`、`title`、`content`、`priority`、`due_date` はいずれも必須。
- 紐づける目標は、ログイン中の生徒自身の目標である必要がある。
- 期限日は日付として解釈できる値である必要がある。
- 関連付ける単元IDは、すべて実在する単元のIDである必要がある（未指定の場合は単元を関連付けずに作成する）。

### Database変更

作成:

```
draft_tasks
draft_task_units（関連付けた単元の数だけ作成）
```

### Response

- `id`（作成した下書きタスクのID）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|必須項目の不足、目標が存在しない、期限日が不正、単元IDが不正など|422|`errors` を返却|

---

# 6. データモデル

## `draft_tasks`

- 役割: 正式なタスク登録前の下書き内容を保持するテーブル。
- 主なカラム:
  - `id`
  - `user_id`
  - `goal_id`
  - `course_id`
  - `title`
  - `content`
  - `priority`（`very_low` / `low` / `normal` / `high` / `very_high`）
  - `due_date`
  - `estimated_time`
  - `status`（`not_started` / `in_progress` / `completed`）
  - `memo`
  - `completed_at`
  - `deleted_at`
- リレーション:
  - `belongs_to :user`
  - `belongs_to :goal`
  - `has_many :draft_task_courses`
  - `has_many :courses`（`draft_task_courses` 経由）
  - `has_many :draft_task_units`
  - `has_many :units`（`draft_task_units` 経由）

## `draft_task_courses`

- 役割: 下書きタスクとコースの関連付けを保持する中間テーブル。現行実装の下書き作成処理ではコースの関連付けは行われておらず、テーブル自体はモデル上の関連として定義されている。
- 主なカラム: `id`, `draft_task_id`, `course_id`

## `draft_task_units`

- 役割: 下書きタスクと単元の関連付けを保持する中間テーブル。下書きタスク作成時に指定された単元がここに登録される。
- 主なカラム: `id`, `draft_task_id`, `unit_id`

---

# 7. 状態管理

- `status` を「未着手」「進行中」「完了」の3状態で管理する項目として保持しているが、下書きタスクの作成処理自体はこの値を明示的に更新しておらず、初期値のまま保存される。下書き段階での状態変更操作は、現行実装のこの機能内には存在しない。

---

# 8. 権限制御

- `student` ロールのユーザーのみ利用できる。
- 下書きタスクの詳細取得は、ログイン中の生徒自身が作成した下書きタスクに限られる。
- 下書きタスク作成時に指定する目標も、ログイン中の生徒自身の目標に限られる。

---

# 9. 非同期処理

- 下書きタスクの取得・作成に紐づく Job / Mailer は見当たらない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::DraftTasksController`|
|Form|`Student::CreateDraftTaskForm`|
|Service|`Student::CreateDraftTaskService`|
|Model|`DraftTask`, `DraftTaskCourse`, `DraftTaskUnit`|
|Serializer|`DraftTaskSerializer`|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
