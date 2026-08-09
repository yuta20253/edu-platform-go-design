# 問題解答機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒がタスク内の単元に対して問題を解答し、解答履歴を記録・更新すること。
- システム上の役割
  - 問題一覧を提供し、解答結果を判定して履歴を保存する。
  - 生徒の確認用表示に対して解答済み状態を伝える。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|問題一覧取得|タスクと単元に紐づく問題を取得する|
|解答登録|問題への解答を保存し、正誤判定を行う|
|解答更新|既存解答を更新し、再判定する|
|確認表示|既回答の問題確認用データを取得する|
|提出|タスクを提出し、タスク状態を更新する|

---

# 3. 業務フロー

1. 生徒が単元内の問題一覧を表示する。
2. システムはタスクと単元の紐づきを検証し、該当問題を返却する。
3. 生徒が解答を送信する。
4. 解答を妥当性検証し、選択肢の正誤を判定する。
5. 解答履歴を `question_histories` に保存する。
6. 再解答時は既存履歴を更新する。
7. 生徒が提出操作を行うと、タスクの進捗状態を更新する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::QuestionsController`|`index`|GET|`/api/v1/student/tasks/:task_id/units/:unit_id/questions`|問題一覧を取得|
|`Api::V1::Student::AnswersController`|`create`|POST|`/api/v1/student/answers`|解答を登録|
|`Api::V1::Student::AnswersController`|`update`|PATCH|`/api/v1/student/answers`|解答を更新|
|`Api::V1::Student::ConfirmationsController`|`show`|GET|`/api/v1/student/tasks/:task_id/units/:unit_id/confirmation`|確認表示用データを取得|
|`Api::V1::Student::SubmissionsController`|`update`|PATCH|`/api/v1/student/tasks/:task_id/submission`|タスクを提出し状態を更新|

---

# 5. API / 処理詳細

## `Api::V1::Student::QuestionsController#index`

### 概要

タスクに紐づく単元の問題一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|
|`unit_id`|必須|integer|単元 ID|

### 処理内容

1. `current_user.tasks.find(task_id)` でタスクを取得する。
2. そのタスクに紐づく単元を検索する。
3. 単元の `questions` を取得し、選択肢とヒントを含めて返却する。
4. 既回答問題IDセットを計算し、`answered` フラグを付与する。

### 業務ルール

- 生徒自身のタスクに紐づく単元のみ対象とする。
- 既回答問題は `answered` で示す。

### Database変更

- なし

### Response

- `id`
- `unit_id`
- `question_text`
- `course_id`
- `answered`
- `question_hints`
  - `id`
  - `question_id`
  - `step_number`
  - `hint_text`
- `question_choices`
  - `id`
  - `question_id`
  - `choice_number`
  - `choice_text`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象タスク/単元が存在しない|404|対象データが見つかりません|

## `Api::V1::Student::AnswersController#create`

### 概要

生徒が問題への解答を登録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|
|`unit_id`|必須|integer|単元 ID|
|`question_id`|必須|integer|問題 ID|
|`question_choice_id`|必須|integer|選択した選択肢 ID|
|`answer_text`|任意|string|テキスト解答|
|`time_spent_sec`|任意|integer|解答時間（秒）|
|`explanation_viewed`|任意|boolean|解説閲覧フラグ|

### 処理内容

1. `CreateQuestionHistoryForm` を `current_user` と入力で初期化する。
2. タスク・単元・問題の紐づきを検証する。
3. 選択肢が対象問題のものであるかを検証する。
4. `QuestionAnswerJudgeService` で正誤判定を行う。
5. `QuestionHistory` を作成する。

### 業務ルール

- `task_id`, `unit_id`, `question_id`, `question_choice_id` は必須。
- 選択肢は該当問題のものである必要がある。
- `task_id` に紐づく単元と問題のみ解答を許可する。

### Database変更

- `question_histories` に新規レコードを作成

### Response

- `selected_answer`
- `correct_answer`
- `is_correct`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|
|対象データが不正|422|`task_id`, `unit_id`, `question_id` などにエラー|

## `Api::V1::Student::AnswersController#update`

### 概要

既存の解答履歴を更新し、再判定する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|
|`unit_id`|必須|integer|単元 ID|
|`question_id`|必須|integer|問題 ID|
|`question_choice_id`|必須|integer|選択した選択肢 ID|
|`answer_text`|任意|string|テキスト解答|
|`time_spent_sec`|任意|integer|解答時間（秒）|
|`explanation_viewed`|任意|boolean|解説閲覧フラグ|

### 処理内容

1. `UpdateQuestionHistoryForm` を `current_user` と入力で初期化する。
2. 既存の `QuestionHistory` を検索する。
3. 選択肢正誤判定を再実行する。
4. 既存 `QuestionHistory` を更新する。

### 業務ルール

- 既存の解答履歴が存在しない場合、更新は失敗する。
- 選択肢は該当問題のものである必要がある。

### Database変更

- 既存 `question_histories` レコードを更新

### Response

- `selected_answer`
- `correct_answer`
- `is_correct`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|解答履歴が見つからない|422|`errors` を返却|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Student::ConfirmationsController#show`

### 概要

指定した問題に対する解答確認用データを取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|
|`unit_id`|必須|integer|単元 ID|
|`answered_question_ids`|任意|string|カンマ区切りの解答済み問題 ID|

### 処理内容

1. `current_user.tasks.find(task_id)` でタスクを取得する。
2. タスクに紐づく単元から質問を取得する。
3. `question_histories` を対象質問ごとに取得してインデックス化する。
4. `QuestionConfirmationSerializer` で JSON を返却する。

### 業務ルール

- タスクに紐づく単元の問題のみ対象。
- 回答履歴が存在すれば `status` が `answered` となる。

### Database変更

- なし

### Response

- `question_id`
- `question_text`
- `correct_answer`（解答履歴がある場合のみ）
- `is_correct`
- `selected_choice_number`
- `status` (`answered` / `unanswered`)

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象データが見つからない|404|対象データが見つかりません|

## `Api::V1::Student::SubmissionsController#update`

### 概要

タスクを提出し、進捗状態を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|

### 処理内容

1. `Student::SubmissionService` を `current_user` と `task_id` で実行する。
2. `TaskCompletionService` でタスクの完了/進行状況を判定する。
3. `TaskStatusUpdaterService` でタスク状態を更新する。

### 業務ルール

- すでに完了状態なら `completed` を返す。
- すべての問題が回答済みなら `completed`、一部回答済みなら `in_progress`、未回答なら `not_started` を返す。

### Database変更

- `tasks` レコードの `status` と `completed_at` を更新

### Response

- `status`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象タスクが存在しない|404|対象データが見つかりません|

---

# 6. データモデル

## `question_histories`

- 役割: 生徒の問題解答履歴を記録するテーブル
- 主なカラム: `user_id`, `course_id`, `unit_id`, `question_id`, `question_choice_id`, `answer_text`, `time_spent_sec`, `is_correct`, `explanation_viewed`, `answered_at`, `task_id`
- リレーション:
  - `belongs_to :user`
  - `belongs_to :course`
  - `belongs_to :unit`
  - `belongs_to :question`
  - `belongs_to :task`
  - `belongs_to :question_choice`

---

# 7. 状態管理

- タスクの提出時に `status` が更新される。
- 判定ロジック:
  - すべての問題が回答済み → `completed`
  - 一部回答済み → `in_progress`
  - 未回答 → `not_started`

---

# 8. 権限制御

- すべて `current_user.tasks` からタスクを検索する。
- 生徒は自身のタスク・単元・問題のみ操作できる。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::QuestionsController`, `Api::V1::Student::AnswersController`, `Api::V1::Student::ConfirmationsController`, `Api::V1::Student::SubmissionsController`|
|Form|`Student::CreateQuestionHistoryForm`, `Student::UpdateQuestionHistoryForm`|
|Service|`Student::QuestionAnswerJudgeService`, `Student::SubmissionService`, `Student::TaskCompletionService`, `Student::TaskStatusUpdaterService`|
|Model|`QuestionHistory`, `Question`, `QuestionChoice`, `QuestionHint`|
|Serializer|`QuestionSerializer`, `QuestionConfirmationSerializer`, `QuestionChoiceSerializer`, `QuestionHintSerializer`|
