# タスク管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒が自分の学習タスクを登録・管理し、紐づく目標や単元と合わせて確認すること。
- システム上の役割
  - タスクの一覧表示、詳細表示、作成、更新を提供する。
  - タスクに紐づく単元情報を含めて返却する。詳細では単元ごとに学習を開始済みかどうかを示す。
  - タスクの状態（未着手・進行中・完了）を保持する。状態の変更はこの機能の API では行わず、学習ログの記録を契機に自動的に行われる。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|生徒のタスクを状態で絞り込み、期日昇順でページング取得する|
|詳細取得|指定タスクの詳細と、紐づく単元情報（学習開始済みかを含む）を取得する|
|新規登録|タスクを作成し、目標と単元を紐づける|
|更新|タスク情報と紐づく単元を変更する|
|着手（他機能からの契機）|未着手のタスクを進行中に変更する。学習ログの記録から呼ばれる|

---

# 3. 業務フロー

1. 生徒がタスク一覧画面にアクセスする。
2. システムは生徒のタスクを状態で絞り込み、`due_date` 昇順でページング取得する。
3. 生徒がタスクを選択すると、タスクの詳細と紐づく単元情報を取得する。
4. 生徒がタスク情報を入力し、作成または更新を実行する。
5. 入力値を検証する。目標が自分のものか、単元が存在するかを確認する。
6. 妥当であれば `tasks` テーブルへ登録または更新し、紐づく単元との関係を同期する。
7. 結果を返却する。

タスクの状態は上記の流れでは変化しない。生徒が単元の学習ログを記録した時点で、未着手のタスクが進行中に変わる（「7. 状態管理」）。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::TasksController`|`index`|GET|`/api/v1/student/tasks`|生徒のタスク一覧を取得|
|`Api::V1::Student::TasksController`|`show`|GET|`/api/v1/student/tasks/:id`|タスク詳細と単元情報を取得|
|`Api::V1::Student::TasksController`|`create`|POST|`/api/v1/student/tasks`|タスクを作成|
|`Api::V1::Student::TasksController`|`update`|PATCH|`/api/v1/student/tasks/:id`|タスクを更新|

ルーティングは `resources :tasks` で定義されているため `DELETE /api/v1/student/tasks/:id` も経路として存在するが、`TasksController` に `destroy` が実装されていない。タスクの削除は提供されていない。

`/api/v1/student/tasks/:task_id` の下には、提出（`submission`）・単元詳細（`units`）・問題（`questions`）・学習ログ（`study_logs`）・解答（`answers`）・確認（`confirmation`）が入れ子で定義されている。これらはそれぞれ別機能であり、本仕様書の対象外である。

---

# 5. API / 処理詳細

## `Api::V1::Student::TasksController#index`

### 概要

ログイン中の生徒のタスク一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`status`|任意|string|タスクの状態での絞り込み（`not_started` / `in_progress` / `completed`）|
|`page`|任意|integer|ページ番号。0以下・数値以外は未指定として扱う（`ApplicationController#sanitized_page`）|
|`per_page`|任意|integer|1ページあたりの件数。既定値は 5（`TasksController::DEFAULT_PER_PAGE`）、上限は 100（`ApplicationController::MAX_PER_PAGE`）|

### 処理内容

1. `current_user.tasks` を対象に、紐づく単元とその所属コースを同時に読み込む（N+1 を避けるため）。
2. `status` による絞り込みを適用する（`Task` の `by_status`）。
3. `due_date` 昇順で並べる。
4. ページングする。
5. `TaskSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身のタスクのみ取得する。
- `status` が未指定の場合、および定義されていない値が指定された場合は、`not_started` または `in_progress` のタスクのみを取得する（`Task` の `active_status`）。完了済みのタスクは含まれない。
- 期限日は昇順で並べる。期限日が同じ場合の順序は決まっていない（第2の並び順が指定されていない）。
- 論理削除済み（`deleted_at` が設定済み）のタスクは対象に含まれない（`Task` の `default_scope`）。

### Database変更

- なし

### Response

- 最上位は `tasks` と `meta` の2キーである
- `tasks` の各要素は `TaskSerializer`
  - `id`
  - `user_id`
  - `goal_id`
  - `title`
  - `content`
  - `due_date`（`YYYY/MM/DD` 形式。未設定なら `null`）
  - `priority`（`very_low` / `low` / `normal` / `high` / `very_high` の文字列）
  - `status`（`not_started` / `in_progress` / `completed` の文字列）
  - `completed_at`（`YYYY/MM/DD` 形式。未設定なら `null`）
  - `units`: 紐づく単元の配列（`UnitSerializer`）。各要素は `id`・`course_id`・`unit_name` と、`course`（`id`・`level_number`・`level_name` の3項目）
- `meta`: `current_page` / `total_pages` / `total_count` / `per_page` の4項目。`per_page` には実際に適用された件数を返す
- 一覧の単元には、詳細で返す `started`（学習開始済みか）が含まれない

### Errorケース

- 特定なし（`current_user` の認証前提）

## `Api::V1::Student::TasksController#show`

### 概要

指定タスクの詳細と、紐づく単元情報を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|タスクの識別子|

### 処理内容

1. `current_user.tasks` から、紐づく単元とその所属コースを同時に読み込んだうえで指定 ID のタスクを検索する。
2. そのタスクで学習を開始済みの単元の ID 一覧を求める（`Task#started_unit_ids`。解答履歴 `question_histories` に記録のある単元）。
3. `TaskDetailSerializer` に開始済み単元の ID 一覧を渡して JSON を返却する。

### 業務ルール

- 生徒自身のタスクのみ取得できる。
- 単元ごとに、そのタスクで学習を開始済みかどうかを示す。判定は解答履歴の有無で行う。

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
- `units`: 紐づく単元の配列（`UnitDetailSerializer`）。各要素は `id`・`course_id`・`unit_name`・`course`（`id`・`level_number`・`level_name`）と、**`started`**（そのタスクで学習を開始済みかの真偽値）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象のタスクが存在しない|404|`{"message": "タスクが見つかりません"}`（`ApplicationController#not_found`）|
|他ユーザーのタスクにアクセス|404|同上（`current_user.tasks` を起点に検索するため、他の生徒のタスクは見つからない）|

## `Api::V1::Student::TasksController#create`

### 概要

生徒が新しいタスクを登録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task[goal_id]`|必須|integer|紐づく目標の ID|
|`task[title]`|必須|string|タスクのタイトル|
|`task[content]`|必須|string|タスクの詳細|
|`task[priority]`|必須|string|優先度。優先度名（`normal` 等）と数値（`1`〜`5`）のどちらでも受け取る|
|`task[due_date]`|必須|string|期限日|
|`task[memo]`|任意|string|メモ|
|`task[unit_ids]`|任意|array|紐づける単元の ID 一覧|

### 処理内容

1. `Student::CreateTaskForm` を `current_user` と入力で初期化する。
2. バリデーションを実行する。
3. `Student::CreateTaskService` がタスクの作成と単元の紐づけを1つのトランザクションで行う。

### 業務ルール

- `goal_id`・`title`・`content`・`due_date`・`priority` は必須。
- `goal_id` は**操作者自身の目標**である必要がある（`current_user.goals` から検索し、見つからなければエラー）。
- `due_date` は日付として解釈できる必要がある（`Date.parse`。特定のフォーマットには限定していない）。
- `unit_ids` が指定された場合、すべての単元 ID が存在する必要がある。
- `unit_ids` は空文字を除いて整数に変換したうえで扱う。
- `priority` は数値で渡された場合に優先度名へ変換する。定義されていない数値（例: `9`）は変換結果が空になり、必須チェックで弾かれる。
- `estimated_time` は入力を受け取らず、常に `0` で作成する。
- 状態は入力を受け取らず、既定値の `not_started` で作成する。

### Database変更

- `tasks` に新規レコード作成
- `task_units` に紐づく単元のレコードを追加

### Response

- **201 Created**
- `{"message": "タスクが作成されました。"}`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|必須項目の未入力、`due_date` が日付として解釈できない|422|`{"errors": ["メッセージ", ...]}`。未入力は `Title を入力してください` の形、日付として解釈できない場合は `Due date は正しい日付を入力してください`|
|`goal_id` が操作者自身の目標でない（存在しない場合を含む）|422|`Goal が不正です`|
|`unit_ids` に存在しない単元が含まれる|422|`Unit ids に不正な値が含まれています`。`Student::CreateTaskForm` は単元の存在確認を2回登録しているため（`UnitIdsValidatable` が `include` 時に登録するものと、クラス側で重ねて宣言しているもの）、同じメッセージが2件入る|
|定義されていない優先度名（例: `urgent`）を指定した|500|必須チェックを通過したうえで `tasks.priority` への変換に失敗し、例外のハンドラがないため|

## `Api::V1::Student::TasksController#update`

### 概要

生徒が既存タスクを更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|タスクの識別子|
|`task[title]`|必須|string|タスクのタイトル|
|`task[content]`|必須|string|タスクの詳細|
|`task[priority]`|必須|string|優先度|
|`task[due_date]`|必須|string|期限日|
|`task[memo]`|任意|string|メモ|
|`task[unit_ids]`|任意|array|紐づける単元の ID 一覧|

### 処理内容

1. `current_user.tasks` から指定 ID のタスクを検索する。
2. `Student::UpdateTaskForm` で入力を検証する。
3. `Student::UpdateTaskService` がタスクの更新と単元の紐づけの同期を1つのトランザクションで行う。

### 業務ルール

- `title`・`content`・`due_date`・`priority` は必須。
- `goal_id` は更新の対象外である。作成時に決めた目標は変更できない。
- 状態・完了日時も更新の対象外である。
- `unit_ids` が渡された場合のみ単元の紐づけを同期する。渡されなかった場合は現在の紐づけを変更しない。
- **学習開始済みの単元は紐づけから外せない。** 判定はそのタスクの解答履歴に記録のある単元かどうかで行う。
- 同期では、増えた単元を追加し、減った単元を削除する。

### Database変更

- 既存の `tasks` レコード更新
- `task_units` の追加・削除で単元の紐づけを同期

### Response

- **200 OK**
- `{"message": "タスクが更新されました。"}`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|必須項目の未入力、`due_date` が日付として解釈できない|422|`{"errors": ["メッセージ", ...]}`。未入力は `Title を入力してください` の形、日付として解釈できない場合は `Due date は正しい日付を入力してください`|
|学習開始済みの単元を紐づけから外そうとした|422|`Unit ids 学習開始済みの単元は削除できません`|
|`unit_ids` に存在しない単元が含まれる|422|`Unit ids に不正な値が含まれています`|
|対象のタスクが存在しない、または他の生徒のタスク|404|`{"message": "タスクが見つかりません"}`|

`Student::UpdateTaskForm` は、更新処理の結果を見ずに常に成功を返す。そのため更新処理の中でモデルの検証に失敗した場合でも、**200 と成功メッセージが返る**（更新自体はトランザクションにより巻き戻る）。

## タスクの着手（API を持たない操作）

### 概要

未着手のタスクを進行中に変更する。この機能の API としては公開されておらず、生徒が単元の学習ログを記録した時点で実行される（`Student::CreateStudyLogService` から `Student::TaskStartService` が呼ばれる）。

### 処理内容

1. 対象タスクが未着手でなければ何もしない。
2. 未着手であれば状態を進行中に変更する。完了日時は未設定にする。
3. そのタスクが属する目標の状態を、目標に紐づくタスクの状況から再評価して更新する（「目標管理機能」の「7. 状態管理」）。

### 業務ルール

- 対象が未着手のときだけ状態を変える。進行中・完了のタスクには影響しない。
- タスクの着手は、紐づく目標の状態にも波及する。

### Database変更

- `tasks` の `status`・`completed_at` を更新
- 紐づく目標の `goals.status` を更新（目標管理機能側の処理）

---

# 6. データモデル

## `tasks`

- 役割: 生徒の学習タスクを記録するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|生徒（`users`）への外部キー|
|`goal_id`|bigint|**可**|目標（`goals`）への外部キー。カラムは NULL 可だが、`belongs_to :goal` により登録・更新時は必須である|
|`title`|string(100)|不可|タスクのタイトル|
|`content`|text|不可|タスクの詳細|
|`priority`|integer|不可|優先度。Rails の `enum` で `very_low: 1` / `low: 2` / `normal: 3` / `high: 4` / `very_high: 5` に対応する整数。既定値は `3`（`normal`）。**`0` は使っていない**|
|`due_date`|date|可|期限日。カラムは NULL 可だが、作成・更新の Form は必須としている|
|`estimated_time`|integer|可|見積時間。API からは入力を受け取らず、作成時に常に `0` が入る|
|`status`|integer|不可|状態。Rails の `enum` で `not_started: 0` / `in_progress: 1` / `completed: 2` に対応する整数。既定値は `0`|
|`memo`|text|可|メモ|
|`completed_at`|datetime|可|完了日時|
|`deleted_at`|datetime|可|論理削除日時|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- `user_id`・`goal_id` にそれぞれインデックスを持つ
- リレーション:
  - `belongs_to :user`
  - `belongs_to :goal`
  - `has_many :task_units`（`dependent: :destroy`）
  - `has_many :units, through: :task_units`
  - `has_many :task_courses`（`dependent: :destroy`）
  - `has_many :courses, through: :task_courses`
  - `has_many :question_histories`（`dependent: :destroy`）
- `default_scope { where(deleted_at: nil) }` を持つため、論理削除済みのタスクは通常の検索の対象外である
- 状態による絞り込み:
  - `active_status`: 未着手または進行中のタスク
  - `by_status(status)`: 定義された状態名が渡された場合はその状態で絞り込み、それ以外（未指定・未定義の値）は `active_status` を適用する
- `started_unit_ids`: そのタスクの解答履歴に記録のある単元の ID 一覧。詳細の `started` と、更新時の「学習開始済みの単元は外せない」判定に用いる
- 論理削除を行う操作はこの機能に存在しない

## `task_units`

- 役割: タスクと単元の紐づきを記録するテーブル
- 主なカラム: `id`・`task_id`（不可）・`unit_id`（不可）・`created_at`・`updated_at`
- `task_id` と `unit_id` の組に一意インデックスを持つため、同じ単元を同じタスクに重複して紐づけられない

## `units`（参照のみ）

- 役割: 学習の単元。この機能では参照のみ行う
- API のレスポンスに含まれる項目: `id`・`course_id`・`unit_name` と、所属コースの `id`・`level_number`・`level_name`

## エラーレスポンスの本文

この機能のエラー本文は `ApplicationController` の共通ハンドラが決める。本文の形は条件によって異なる。

|条件|HTTP Status|本文|
|-|-|-|
|バリデーション失敗（Form・モデル）|422|`{"errors": ["メッセージ", ...]}`|
|`ActiveRecord::RecordNotFound`|404|`{"message": "タスクが見つかりません"}`（`errors` ではなく `message` の単一文字列）|
|`Pundit::NotAuthorizedError`|403|`{"errors": ["この操作を行う権限がありません"]}`|

422 のメッセージは「項目名 + メッセージ本体」の形で組み立てられる。この機能の Form（`Student::CreateTaskForm` / `Student::UpdateTaskForm`）の項目名は日本語訳が定義されていないため、**項目名が英語のまま出る**（`Title を入力してください`・`Unit ids に不正な値が含まれています` 等）。メッセージ本体のみ日本語である。

|項目|メッセージに出る項目名|
|-|-|
|`goal_id`|`Goal`|
|`title`|`Title`|
|`content`|`Content`|
|`due_date`|`Due date`|
|`priority`|`Priority`|
|`unit_ids`|`Unit ids`|

一方、404 のモデル名は日本語訳が定義されているため `タスクが見つかりません` となる。

---

# 7. 状態管理

- `status` を enum で管理する。DB 上は整数（`not_started: 0` / `in_progress: 1` / `completed: 2`）。
- **タスクの `status` は、この機能の API では直接変更しない。** 作成時は未着手で、以後は学習の進み方に応じて変わる。

```
not_started
  ↓ 単元の学習ログを記録した（Student::CreateStudyLogService → Student::TaskStartService）
in_progress
  ↓ 完了の条件を満たした（問題解答機能側の処理）
completed
```

- 状態を変える処理は `Student::TaskStatusUpdaterService` に集約されている。状態と合わせて完了日時も設定する（完了なら現在時刻、それ以外は未設定）。
- `Task` 自体も、検証の直前に状態と完了日時の整合を取る。完了であれば完了日時を設定し、完了でなければ完了日時を消す。
- タスクの状態が変わると、紐づく目標の状態も再評価される（「目標管理機能」の「7. 状態管理」）。着手の場合は `Student::TaskStartService` がこの再評価を行う。

---

# 8. 権限制御

- すべての検索は `current_user.tasks` を起点に行う。
- そのため、生徒は自分自身のタスクのみ閲覧・更新できる。
- 作成時の `goal_id` も `current_user.goals` から検索するため、他の生徒の目標にタスクを紐づけられない。
- `Api::V1::Student::BaseController` の認証認可に依存する設計。

---

# 9. 非同期処理

- `Task` の作成・更新に紐づく Job / Mailer は見当たらない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::TasksController`（`DEFAULT_PER_PAGE = 5`）|
|Controller（共通）|`ApplicationController`（`MAX_PER_PAGE = 100`・`sanitized_page`・`sanitized_per_page`・`not_found` 等の共通ハンドラ）|
|Form|`Student::CreateTaskForm`, `Student::UpdateTaskForm`|
|Form の Concern|`UnitIdsValidatable`（単元 ID の正規化と存在確認）, `PriorityCastable`（優先度の数値→名称変換と必須チェック）|
|Service|`Student::CreateTaskService`, `Student::UpdateTaskService`|
|Service（状態変更）|`Student::TaskStatusUpdaterService`（状態と完了日時の更新）, `Student::TaskStartService`（未着手→進行中。学習ログ機能から呼ばれる）|
|Model|`Task`（`active_status`・`by_status`・`started_unit_ids`）, `TaskUnit`|
|Serializer|`TaskSerializer`（一覧）, `TaskDetailSerializer`（詳細）, `UnitSerializer`, `UnitDetailSerializer`（`started` を持つ）|
