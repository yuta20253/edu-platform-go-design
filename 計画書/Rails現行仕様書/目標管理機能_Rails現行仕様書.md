# 目標管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒が自分の学習目標を登録・管理すること。
- システム上の役割
  - 生徒の目標を一覧表示し、詳細と紐づくタスクを確認できる。
  - 目標の新規作成と更新を受け付ける。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|生徒の目標を期日昇順で取得する|
|詳細取得|指定目標の詳細とその目標に紐づくタスク一覧を取得する|
|新規登録|目標を新規作成する|
|更新|既存の目標を更新する|

---

# 3. 業務フロー

1. 生徒が目標一覧画面にアクセスする。
2. システムは生徒の目標を `due_date` 昇順でページング取得する。
3. 生徒が目標を選択すると、目標の詳細と関連タスクを取得する。
4. 生徒が目標を入力し、作成または更新を実行する。
5. 入力値を検証する。
6. 妥当であれば `goals` テーブルへ登録または更新する。
7. 結果を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::GoalsController`|`index`|GET|`/api/v1/student/goals`|生徒の目標一覧を取得|
|`Api::V1::Student::GoalsController`|`show`|GET|`/api/v1/student/goals/:id`|目標詳細と紐づくタスクを取得|
|`Api::V1::Student::GoalsController`|`create`|POST|`/api/v1/student/goals`|目標を作成|
|`Api::V1::Student::GoalsController`|`update`|PATCH|`/api/v1/student/goals/:id`|目標を更新|
|`Api::V1::Student::GoalsController`|`destroy`|DELETE|`/api/v1/student/goals/:id`|目標を論理削除|

---

# 5. API / 処理詳細

## `Api::V1::Student::GoalsController#index`

### 概要

ログイン中の生徒が登録した目標一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号。0以下・数値以外は未指定として扱う（`ApplicationController#sanitized_page`が`nil`を返し、Kaminariが1ページ目とする）|
|`per_page`|任意|integer|1ページあたりの件数。既定値は 5（`GoalsController::DEFAULT_PER_PAGE`）、上限は 100（`ApplicationController::MAX_PER_PAGE`）。0以下・数値以外は既定値、上限超過は上限値として扱う（`ApplicationController#sanitized_per_page`）|

### 処理内容

1. `current_user.goals` を対象に `GoalsQuery.due_soon` を適用する。
2. `GoalsQuery.includes_tasks` で紐づくタスクを同時に読み込む（N+1 を避けるため）。
3. `paginate` でページングする。
4. `Student::GoalSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身の目標のみ取得する。
- 期日の昇順で並べる。期日が同じ場合は `id` の昇順で並べる（順序を決定的にするため）。
- 論理削除済み（`deleted_at` が設定済み）の目標は対象に含まれない（`Goal` の `default_scope`）。

### Database変更

- なし

### Response

- 最上位は `goals` と `meta` の2キーである
- `goals` の各要素は `Student::GoalSerializer`
  - `id`
  - `title`
  - `description`
  - `status`
  - `due_date`（`YYYY/MM/DD` 形式。未設定なら `null`）
  - `tasks`: 紐づくタスクの配列（`TaskSerializer`）。各要素は `id`・`user_id`・`goal_id`・`title`・`content`・`due_date`（`YYYY/MM/DD`）・`priority`・`status`・`completed_at`（`YYYY/MM/DD`）と、`units`（`UnitSerializer`）
- `meta`: `current_page` / `total_pages` / `total_count` / `per_page` の4項目
- 1ページあたりの件数の既定値は 5（`GoalsController::DEFAULT_PER_PAGE`）、上限は 100（`ApplicationController::MAX_PER_PAGE`）
- `meta.per_page` には実際に適用された件数（`limit_value`）を返す

### Errorケース

- 特定なし（`current_user` の認証前提）

## `Api::V1::Student::GoalsController#destroy`

### 概要

指定目標を論理削除する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|目標の ID|

### 処理内容

1. `GoalsQuery.new(current_user.goals).find(params[:id])` で対象目標を取得する。
2. `Goal#soft_delete!` を呼ぶ。進行中または完了のタスク（`status` が `not_started` 以外のタスク）が存在する場合は `Goal::HasActiveTasksError` を発生させ、削除しない。
3. 存在しない場合は `deleted_at` と `updated_at` を更新する（`update_columns`。バリデーション・コールバックを走らせない）。

### 業務ルール

- 生徒自身の目標のみ削除できる。
- **進行中または完了のタスクが紐づく目標は削除できない。** 判定は `status` が `not_started` 以外のタスクが存在するかで行う（`Goal#active_tasks`）。
- 物理削除ではなく論理削除である。タスク側の行は削除しない。

### Database変更

- `goals` の `deleted_at`・`updated_at` を更新

### Response

- 204 No Content（本文なし）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象目標が存在しない、または他の生徒の目標|404|`{"message": "目標が見つかりません"}`（`ApplicationController#not_found`）|
|進行中または完了のタスクが紐づく|（`Goal::HasActiveTasksError` が発生する。Controller・`ApplicationController` のいずれにもこの例外のハンドラがなく、500 となる）|メッセージは `進行中または完了のタスクがあるため削除できません`|

## `Api::V1::Student::GoalsController#show`

### 概要

指定目標の詳細と、その目標に紐づくタスクを取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|目標の識別子|

### 処理内容

1. `current_user.goals` から `GoalsQuery.includes_tasks` を適用する。
2. 指定 ID の目標を検索する。
3. `Student::GoalDetailSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身の目標のみ取得できる。
- 目標に紐づくタスクを含めて返却する。

### Database変更

- なし

### Response

- `id`
- `title`
- `description`
- `status`
- `due_date`（`YYYY/MM/DD` 形式）
- `tasks`
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

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の目標が存在しない|404|`{"message": "目標が見つかりません"}`（`ApplicationController#not_found`）|
|他ユーザーの目標にアクセス|404|同上（`current_user.goals` を起点に検索するため、他の生徒の目標は見つからない）|

## `Api::V1::Student::GoalsController#create`

### 概要

生徒が新しい目標を登録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`goal[title]`|必須|string|目標名|
|`goal[description]`|任意|string|目標の詳細|
|`goal[due_date]`|必須|string|期限日|

### 処理内容

1. `Student::CreateGoalForm` を `current_user` と入力で初期化する。
2. バリデーションを実行する。
3. 成功した場合、`goals` テーブルに新規登録する。

### 業務ルール

- `title` は必須。
- `due_date` は必須。

### Database変更

- `goals` に新規レコード作成

### Response

- **201 Created**
- 本文は作成した目標の ID のみ（`render json: form.goal.id, status: :created`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`{"errors": ["メッセージ", ...]}`（`form.errors.full_messages`）|

## `Api::V1::Student::GoalsController#update`

### 概要

生徒が既存の目標を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`goal[title]`|必須|string|目標名|
|`goal[description]`|任意|string|目標の詳細|
|`goal[due_date]`|必須|string|期限日|

### 処理内容

1. `current_user.goals` から指定の目標を検索する。
2. `Student::UpdateGoalForm` で更新処理を実行する。
3. バリデーションが通れば `goals` を更新する。

### 業務ルール

- `title` は必須。
- `due_date` は必須。
- 他ユーザーの目標は更新できない。

### Database変更

- 既存 `goals` レコード更新

### Response

- **200 OK**
- 本文は更新した目標の ID のみ（`render json: form.goal.id, status: :ok`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`{"errors": ["メッセージ", ...]}`（`form.errors.full_messages`）|
|対象の目標が存在しない、または他の生徒の目標|404|`{"message": "目標が見つかりません"}`（`ApplicationController#not_found`）|

---

# 6. データモデル

## `goals`

- 役割: 生徒の学習目標を記録するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|生徒（`users`）への外部キー|
|`title`|string(255)|不可|目標名|
|`description`|text|可|説明|
|`due_date`|date|**可**|期日。カラムは NULL 可であり、`Student::GoalSerializer` も未設定なら `null` を返す。ただし作成・更新の Form（`Student::CreateGoalForm` / `UpdateGoalForm`）は `presence: true` で必須としているため、API 経由では未設定にできない|
|`status`|integer|不可|進捗。Rails の `enum` で `not_started: 0` / `in_progress: 1` / `completed: 2` に対応する整数。既定値は `0`（`not_started`）|
|`deleted_at`|datetime|可|論理削除日時|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- `user_id` にインデックスを持つ
- リレーション:
  - `belongs_to :user`
  - `has_many :tasks`（`dependent: :destroy`）
  - `has_many :active_tasks`（`status` が `not_started` 以外のタスク。論理削除の可否判定に用いる）
  - `has_many :draft_tasks`（`dependent: :destroy`）
- `default_scope { where(deleted_at: nil) }` を持つため、論理削除済みの目標は通常の検索の対象外である
- 補足:
  - 1つの目標には、正式なタスクのほかに、生徒が作成途中で保存した下書きタスクが紐づくことがある。下書きタスクの登録・管理の詳細は「タスク下書き機能」を参照。

## エラーレスポンスの本文

この機能のエラー本文は `ApplicationController` の共通ハンドラが決める。本文の形は条件によって異なる。

|条件|HTTP Status|本文|
|-|-|-|
|バリデーション失敗（Form）|422|`{"errors": ["メッセージ", ...]}`|
|`ActiveRecord::RecordNotFound`|404|`{"message": "目標が見つかりません"}`（`errors` ではなく `message` の単一文字列）|
|`Pundit::NotAuthorizedError`|403|`{"errors": ["この操作を行う権限がありません"]}`|
|`Goal::HasActiveTasksError`|500|ハンドラがなく、例外がそのまま 500 になる|

---

# 7. 状態管理

- `status` を enum で管理する。DB 上は整数（`not_started: 0` / `in_progress: 1` / `completed: 2`）。
- **目標の `status` は、この機能の API では直接変更しない。** 紐づくタスクの状況から自動的に決まる。
- 判定と更新は次の2つの Service が担う。
  - `Student::GoalCompletionService`: 目標に紐づくタスクから、あるべき `status` を判定する。タスクが1件もなければ `not_started`、すべてのタスクが `completed` なら `completed`、1件でも `not_started` 以外があれば `in_progress`、それ以外は `not_started`
  - `Student::GoalStatusUpdaterService`: 上の判定結果で `status` を更新する
- 呼び出し元は目標管理機能の外にある。
  - `Student::TaskStartService`（タスクの着手。タスク管理機能）
  - `Student::SubmissionService`（問題の提出。問題解答機能）
- したがって、目標の `status` は「タスクが着手された」「問題が提出された」という他機能の操作を契機に変化する。

---

# 8. 権限制御

- すべての検索は `current_user.goals` を起点に行う。
- そのため、生徒は自分自身の目標のみ閲覧・更新・削除できる。
- `Api::V1::Student::BaseController` の認証認可に依存する設計。

---

# 9. 非同期処理

- `Goal` の作成・更新に紐づく Job / Mailer は見当たらない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::GoalsController`（`DEFAULT_PER_PAGE = 5`）|
|Controller（共通）|`ApplicationController`（`MAX_PER_PAGE = 100`・`sanitized_page`・`sanitized_per_page`・`not_found` 等の共通ハンドラ）|
|Form|`Student::CreateGoalForm`, `Student::UpdateGoalForm`（いずれも `DueDateValidatable` を include）|
|Form の Concern|`DueDateValidatable`（`due_date` を文字列で受け取り `Date.parse` で検証する。空なら検証をスキップする）|
|Query|`GoalsQuery`|
|Model|`Goal`（`soft_delete!`・`active_tasks`・`HasActiveTasksError`）|
|Serializer|`Student::GoalSerializer`, `Student::GoalDetailSerializer`（いずれも `tasks` を `TaskSerializer` で含める）|
|Service（他機能から呼ばれる）|`Student::GoalCompletionService`, `Student::GoalStatusUpdaterService`|
