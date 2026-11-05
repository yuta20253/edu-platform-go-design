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
- **論理削除済み（`deleted_at` が設定済み）の下書きタスクも取得できる。** `DraftTask` は `default_scope` を持たないため、`Goal`・`Task` と異なり論理削除済みのレコードが検索の対象に含まれる（「6. データモデル」）。ただし現行実装に下書きタスクを論理削除する操作は存在しないため、通常は `deleted_at` が設定されることはない。

### Database変更

- なし

### Response

- `id`, `user_id`, `goal_id`, `title`, `content`, `due_date`（`YYYY/MM/DD` 形式）, `priority`, `status`, `completed_at`
- `units`（紐づく単元一覧。各単元は `id`, `course_id`, `unit_name`, `course`（コースの `id`, `level_number`, `level_name`）を含む）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の下書きタスクが存在しない、または自分のものでない|404|`{"message": "Draft taskが見つかりません"}`（`ApplicationController#not_found`）。**モデル名の日本語訳が定義されていないため、名称が英語のまま出る**|

## `Api::V1::Student::DraftTasksController#create`

### 概要

生徒が下書きタスクを新規に保存する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`draft_task[goal_id]`|必須|integer|紐づける目標のID|
|`draft_task[title]`|必須|string|下書きタスクのタイトル|
|`draft_task[content]`|必須|string|下書きタスクの内容|
|`draft_task[priority]`|必須|integer|優先度。**整数のみ**（`1`〜`5`）。この機能の Form は優先度名から整数への変換を行わない（タスク管理機能とは異なる。「5. API / 処理詳細」の Errorケースを参照）|
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
- 期限日は日付として解釈できる値である必要がある（`Date.parse`。特定のフォーマットには限定していない）。
- 関連付ける単元IDは、すべて実在する単元のIDである必要がある（未指定の場合は単元を関連付けずに作成する）。
- `unit_ids` は空文字を除いて整数に変換したうえで扱う。
- 優先度は整数として受け取る。Form の属性が integer 型であるため、優先度名（`normal` 等）を渡すと `0` に変換される。`0` は未入力としては扱われないため必須チェックを通過し、保存時に優先度として解釈できず失敗する（Errorケースを参照）。
- `status` は入力を受け取らず、既定値の `not_started`（`0`）で作成する。
- `completed_at` は常に未設定で作成する。
- `estimated_time` は入力を受け取らず、常に `0` で作成する。
- `course_id` は入力を受け取らず、常に未設定で作成する。コースの関連付け（`draft_task_courses`）も行わない。

### Database変更

作成:

```
draft_tasks
draft_task_units（関連付けた単元の数だけ作成）
```

### Response

- **201 Created**
- 本文は作成した下書きタスクの ID のみ（`render json: form.draft_task_id, status: :created`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|`goal_id` 未入力|422|`Goal を入力してください`|
|`goal_id` が操作者自身の目標でない（存在しない場合を含む）|422|`Goal が不正です`|
|`title` 未入力|422|`Title を入力してください`|
|`content` 未入力|422|`Content を入力してください`|
|`priority` 未入力|422|`Priority を入力してください`|
|`due_date` 未入力|422|`Due date を入力してください`|
|`due_date` が日付として解釈できない|422|`Due date は正しい日付を入力してください`|
|`unit_ids` に存在しない単元が含まれる|422|`Unit ids に不正な値が含まれています`。この機能では検証の登録が1回のみであり、**メッセージは1件**である（タスク管理機能では2件返る）|
|`priority` が `1`〜`5` 以外（`0`・`9` や、`normal` のような優先度名を渡した場合を含む）|500|必須チェックを通過したうえで `draft_tasks.priority` への変換に失敗し、例外のハンドラがないため|

いずれの 422 も本文は `{"errors": ["メッセージ", ...]}` である。メッセージは「項目名 + メッセージ本体」の形で組み立てられるが、この機能の Form（`Student::CreateDraftTaskForm`）の項目名は日本語訳が定義されていないため、**項目名が英語のまま出る**。

---

# 6. データモデル

## `draft_tasks`

- 役割: 正式なタスク登録前の下書き内容を保持するテーブル。
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|生徒（`users`）への外部キー|
|`goal_id`|bigint|**可**|目標（`goals`）への外部キー。カラムは NULL 可だが、`belongs_to :goal` により登録時は必須である|
|`course_id`|bigint|可|コースへの外部キー。現行実装では値を設定する処理が存在しない|
|`title`|string(100)|不可|タイトル|
|`content`|text|不可|内容|
|`priority`|integer|不可|優先度。Rails の `enum` で `very_low: 1` / `low: 2` / `normal: 3` / `high: 4` / `very_high: 5` に対応する整数。既定値は `3`（`normal`）。**`0` は使っていない**|
|`due_date`|date|可|期限日。カラムは NULL 可だが、作成の Form は必須としている|
|`estimated_time`|integer|可|見積時間。作成時に常に `0` が入る|
|`status`|integer|不可|状態。Rails の `enum` で `not_started: 0` / `in_progress: 1` / `completed: 2` に対応する整数。既定値は `0`|
|`memo`|text|可|メモ。レスポンスには含まれない|
|`completed_at`|datetime|可|完了日時。現行実装に値を設定する処理は存在しない|
|`deleted_at`|datetime|可|論理削除日時。現行実装に値を設定する処理は存在しない|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- `user_id`・`goal_id`・`course_id` にそれぞれインデックスを持つ
- リレーション:
  - `belongs_to :user`
  - `belongs_to :goal`
  - `has_many :draft_task_courses`（`dependent: :destroy`）
  - `has_many :courses`（`draft_task_courses` 経由）
  - `has_many :draft_task_units`（`dependent: :destroy`）
  - `has_many :units`（`draft_task_units` 経由）
- **`default_scope` を持たない。** `Goal`・`Task` は `default_scope { where(deleted_at: nil) }` を持つが、`DraftTask` は持たないため、論理削除済みのレコードも通常の検索の対象に含まれる

## `draft_task_courses`

- 役割: 下書きタスクとコースの関連付けを保持する中間テーブル。現行実装の下書き作成処理ではコースの関連付けは行われておらず、テーブル自体はモデル上の関連として定義されている。
- 主なカラム: `id`, `draft_task_id`（不可）, `course_id`（不可）, `created_at`, `updated_at`

## `draft_task_units`

- 役割: 下書きタスクと単元の関連付けを保持する中間テーブル。下書きタスク作成時に指定された単元がここに登録される。
- 主なカラム: `id`, `draft_task_id`（不可）, `unit_id`（不可）, `created_at`, `updated_at`
- `draft_task_id` と `unit_id` の組に一意インデックスを持つため、同じ単元を同じ下書きタスクに重複して関連付けられない

---

# 7. 状態管理

- `status` を「未着手」「進行中」「完了」の3状態で管理する項目として保持しているが、下書きタスクの作成処理自体はこの値を明示的に更新しておらず、初期値のまま保存される。下書き段階での状態変更操作は、現行実装のこの機能内には存在しない。
- `completed_at`（完了日時）も同様に保持している項目だが、下書きタスクの作成処理では値を設定せず、常に未設定のまま保存される。現行実装には、下書きタスクを完了状態に変更する操作自体（更新・削除を含む）が存在しないため、この機能のスコープ内外を問わず、`completed_at` を更新する操作は提供されていない。

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
|Controller（共通）|`ApplicationController`（`not_found` 等の共通ハンドラ）|
|Form|`Student::CreateDraftTaskForm`（単元の存在確認・期限日の検証をいずれも自身に持つ。タスク管理機能の `UnitIdsValidatable`・`DueDateValidatable`・`PriorityCastable` は include していない）|
|Service|`Student::CreateDraftTaskService`|
|Model|`DraftTask`, `DraftTaskCourse`, `DraftTaskUnit`|
|Serializer|`DraftTaskSerializer`（`UnitSerializer` で `units` を含める。`memo` は含めない）|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
