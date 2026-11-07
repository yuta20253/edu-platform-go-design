# 問題解答機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒がタスク内の単元に対して問題を解答し、解答履歴を記録・更新すること。
- システム上の役割
  - 問題一覧を提供し、解答結果を判定して履歴を保存する。
  - 生徒の確認用表示に対して解答済み状態を伝える。
  - **解答の登録を契機に、当日の解答件数（Redis のカウンタ）を加算する。** この件数は「ダッシュボード機能」が表示に用いる（「9. 非同期処理」）。
  - **タスクの提出を契機に、タスクの状態と、タスクが属する目標の状態を更新する**（「5. API / 処理詳細」`#update`（提出））。
- 利用者
  - `student` ロールのユーザー（生徒）
- 関連機能
  - 同じタスク・単元の配下では、生徒が単元学習に取り組んだ開始・終了を記録する学習ログ機能も並行して提供されている。学習ログ自体の詳細は「学習ログ機能」を参照。

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
|`Api::V1::Student::AnswersController`|`create`|POST|`/api/v1/student/tasks/:task_id/units/:unit_id/answers`|解答を登録|
|`Api::V1::Student::AnswersController`|`update`|PATCH|`/api/v1/student/tasks/:task_id/units/:unit_id/answers/update`|解答を更新|
|`Api::V1::Student::ConfirmationsController`|`show`|GET|`/api/v1/student/tasks/:task_id/units/:unit_id/confirmation`|確認表示用データを取得|
|`Api::V1::Student::SubmissionsController`|`update`|PATCH|`/api/v1/student/tasks/:task_id/submission`|タスクを提出し状態を更新|

いずれもタスク配下に入れ子で定義されている。解答の更新は `resources :answers, only: [:create]` の `collection` に `patch :update` を足した形であり、**パスの末尾に `/update` が付く**。RESTの慣例では `PATCH /answers/:id` となるところだが、解答履歴の ID をパスに持たず、タスク・単元・問題の組で対象を特定する形になっている。

`task_id`・`unit_id` はパスパラメータであり、解答の登録・更新では Controller がこれをリクエストパラメータとして受け取って Form に渡している。

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

1. 操作者自身のタスクから指定 `task_id` のタスクを取得する。
2. そのタスクに紐づく単元から指定 `unit_id` の単元を取得する。
3. 単元に紐づく問題を取得する。単元・ヒント・選択肢を同時に読み込む（N+1 を避けるため）。
4. 操作者の解答履歴から、そのタスク・単元・当該単元の問題に該当するものの問題 ID を集めて集合にする。
5. 各問題に、その集合に含まれるかどうかを `answered` として付与して返却する。

### 業務ルール

- 生徒自身のタスクに紐づく単元のみ対象とする。
- 既解答の問題は `answered` が真になる。
- `answered` の判定に用いる解答履歴は、論理削除（`deleted_at`）を除外しない。
- 問題の取得では `Question` の論理削除の扱いに従う。

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

最上位は**配列**である（キーでラップしない）。`correct_answer`（正解）はこのレスポンスに含まれない。解答前に正解を明かさないためである。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象のタスクが存在しない、または他の生徒のタスク|404|`{"message": "タスクが見つかりません"}`（`ApplicationController#not_found`）|
|対象の単元がそのタスクに紐づかない|404|`{"message": "Unitが見つかりません"}`。**モデル名の日本語訳が定義されていないため、名称が英語のまま出る**|

このControllerは独自の例外ハンドラを持たないため、共通ハンドラの本文の形（`message` キー）になる。同じ404でも `ConfirmationsController#show` は独自ハンドラを持ち `errors` キーで返すため、**機能内で404の本文の形が2通りある**（「6. データモデル」の「エラーレスポンスの本文」）。

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

1. `Student::CreateQuestionHistoryForm` を `current_user` と入力で初期化し、必須・形式のバリデーションを実行する。
2. タスク・単元・問題の紐づきを順に検証する。`current_user` のタスク → そのタスクに紐づく単元 → その単元に紐づく問題の順で辿り、いずれかが見つからなければエラーとする。
3. 選択肢が対象問題のものであるかを検証する。
4. `Student::QuestionAnswerJudgeService` で正誤判定を行う。
5. `question_histories` に新規レコードを作成する。解答日時は現在時刻、コースは単元の所属コースを設定する。
6. **続けて `Student::IncrementTodayAnswerCountService` を呼び、当日の解答件数（Redis のカウンタ）を 1 加算する。**

### 業務ルール

- `task_id`・`unit_id`・`question_id`・`question_choice_id` は必須。
- `explanation_viewed` は真偽値である必要がある。未指定の場合は `false` として扱う。
- `time_spent_sec` は任意。指定する場合は 0 以上 3600 以下の整数である必要がある（`TimeSpentSecValidatable` の `MAX_TIME_SPENT_SEC` は 1 時間）。
- 選択肢は該当問題のものである必要がある。
- `task_id` に紐づく単元と、その単元に紐づく問題のみ解答を許可する。タスクは操作者自身のものである必要がある。
- **同一のタスク・単元・問題に対して、重複する解答履歴の作成を防ぐ仕組みはない。** 同じ問題に対して `#create` を複数回呼ぶと、複数の履歴が作られる。
- 解答の登録は、当日の解答件数に波及する（「9. 非同期処理」）。

### Database変更

- `question_histories` に新規レコードを作成
- Redis の当日の解答件数のカウンタを 1 加算（RDB の変更ではない。「9. 非同期処理」）

### Response

- **201 Created**
- `selected_answer`: 選択した選択肢の番号（`question_choices.choice_number`）
- `correct_answer`: 正解の番号（`questions.correct_answer` を整数に変換した値）
- `is_correct`: 上記2つが一致するかの真偽値

本文は `Student::QuestionAnswerJudgeService` の戻り値をそのまま返す。作成した解答履歴の ID は返さない。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|必須項目の未入力|422|`{"errors": ["メッセージ", ...]}`|
|`time_spent_sec` が 0 未満または 3600 超、整数でない|422|同上|
|`explanation_viewed` が真偽値でない|422|同上|
|`task_id` が操作者自身のタスクでない（存在しない場合を含む）|422|`Task が不正です`|
|`unit_id` がそのタスクに紐づかない|422|`Unit が不正です`|
|`question_id` がその単元に紐づかない|422|`Question が不正です`|
|`question_choice_id` が対象問題の選択肢でない|422|`Question choice はこの問題の選択肢ではありません`|
|解答履歴の保存がモデルの検証に失敗した|422|`ActiveRecord::RecordInvalid` を Form が捕まえ、例外のメッセージを `base` に積む|

いずれの 422 もメッセージは「項目名 + メッセージ本体」の形で組み立てられるが、この機能の Form の項目名は日本語訳が定義されていないため、**項目名が英語のまま出る**（「6. データモデル」の「エラーレスポンスの本文」）。

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

1. `Student::UpdateQuestionHistoryForm` を `current_user` と入力で初期化し、必須・形式のバリデーションを実行する。
2. 操作者・タスク・単元・問題の4条件に一致する既存の解答履歴を検索する。見つからなければエラーとする。
3. `Student::QuestionAnswerJudgeService` で正誤判定を再実行する。判定には**検索した解答履歴が参照している問題**を用いる。
4. 既存の解答履歴を更新する。解答日時は現在時刻で上書きする。

### 業務ルール

- `task_id`・`unit_id`・`question_id`・`question_choice_id` は必須。検証内容は `#create` と同じ。
- 既存の解答履歴が存在しない場合、更新は失敗する。
- **`#create` と異なり、タスク・単元・問題の紐づきを検証しない。** 解答履歴の検索条件（操作者・タスク・単元・問題の一致）がその役割を兼ねている。
- 同一条件の解答履歴が複数ある場合、`find_by` が最初に見つけた1件のみを更新する。
- 更新は当日の解答件数に波及しない。加算は `#create` のみである。
- 解答日時は更新のたびに現在時刻で上書きされる。初回の解答日時は保持されない。

### Database変更

- 既存 `question_histories` レコードを更新（`question_choice_id`・`answer_text`・`time_spent_sec`・`is_correct`・`explanation_viewed`・`answered_at`）

### Response

- **200 OK**
- `selected_answer`・`correct_answer`・`is_correct`（`#create` と同じ3項目）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|必須項目の未入力、`time_spent_sec`・`explanation_viewed` の形式不正|422|`{"errors": ["メッセージ", ...]}`|
|`question_choice_id` が対象問題の選択肢でない|422|`Question choice はこの問題の選択肢ではありません`|
|対象の解答履歴が見つからない|**422**|`解答履歴が見つかりません`（`base` に積まれるため項目名が付かない）。404 ではない|
|解答履歴の保存がモデルの検証に失敗した|422|`ActiveRecord::RecordInvalid` を Form が捕まえ、例外のメッセージを `base` に積む|

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

1. 操作者自身のタスクから指定 `task_id` のタスクを取得し、そのタスクに紐づく単元から指定 `unit_id` の単元を取得する。いずれかが見つからなければエラーとする。
2. その単元に紐づく**すべての**問題を取得する。
3. `answered_question_ids` をカンマ区切りで分解して整数の配列にし、操作者・タスク・単元・その問題 ID 群の4条件に一致する解答履歴を取得して問題 ID でインデックス化する。
4. `QuestionConfirmationSerializer` で JSON を返却する。

### 業務ルール

- タスクに紐づく単元の問題のみ対象。対象は**単元のすべての問題**であり、`answered_question_ids` で絞り込まれるのは解答履歴の検索条件だけである。
- `answered_question_ids` が未指定または空の場合、解答履歴を1件も取得しない。この場合すべての問題が `unanswered` として返る。
- 解答履歴が存在すれば `status` が `answered`、なければ `unanswered` となる。
- 解答履歴がない問題では `correct_answer` を返さない（`null`）。正解を未解答の問題について明かさないためである。
- 解答履歴の検索では論理削除（`deleted_at`）を除外しない。

### Database変更

- なし

### Response

- 最上位は**配列**である（キーでラップしない）
- 各要素は `QuestionConfirmationSerializer`
  - `question_id`
  - `question_text`
  - `correct_answer`（解答履歴がある場合のみ。ない場合は `null`）
  - `is_correct`（解答履歴がない場合は `null`）
  - `selected_choice_number`（解答履歴がない場合は `null`）
  - `status`（`answered` / `unanswered`）
  - `time_spent_sec`（解答履歴がない場合は `null`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象のタスクまたは単元が見つからない（他の生徒のタスクを含む）|404|`{"errors": ["対象データが見つかりません"]}`。**このControllerは `set_questions` で独自に `ActiveRecord::RecordNotFound` を捕まえており、`ApplicationController#not_found` の `{"message": ...}` とは本文の形が異なる**|

## `Api::V1::Student::SubmissionsController#update`

### 概要

タスクを提出し、進捗状態を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|

### 処理内容

1. `Student::SubmissionService` を `current_user` と `task_id` で実行する。以下の2〜4は**1つのトランザクション**で行う。
2. `Student::TaskCompletionService` でタスクのあるべき状態を判定する。
3. `Student::TaskStatusUpdaterService` でタスクの `status` と `completed_at` を更新する。
4. **続けて、そのタスクが属する目標の状態も再評価して更新する**（`Student::GoalCompletionService` で判定し、`Student::GoalStatusUpdaterService` で更新する）。
5. 判定した状態を返す。

### 業務ルール

- すでに完了状態のタスクなら、判定せず `completed` を返す。
- そうでない場合、解答済みの問題数と対象の問題数を比べて判定する。すべて解答済みなら `completed`、1件以上解答済みなら `in_progress`、1件も解答していなければ `not_started`。
- 判定に用いる集計は `TaskAnswerStatsQuery` である。教員画面の進捗率（`Common::TaskProgressService`）と判定がずれないよう、同じ集計を使っている。
- 集計の対象は、**論理削除されていない解答履歴のうち、論理削除されていない問題に対するもの**に限られる（`QuestionHistory` の `on_active_questions`）。
- 対象の問題数は、タスクに紐づく単元に属する問題の件数である。
- **対象の問題が0件の場合、「すべて解答済み」と判定され `completed` になる。** 解答済み件数（0）と対象の問題数（0）が一致するためである。
- **タスクの提出は、タスクの状態と、タスクが属する目標の状態の双方に波及する。** 目標の状態の判定内容は「目標管理機能」の「7. 状態管理」を参照。
- 提出を何度呼んでも結果は同じである（判定が現在の解答状況から決まるため）。

### Database変更

- `tasks` レコードの `status` と `completed_at` を更新
- 紐づく目標の `goals.status` を更新

### Response

- **200 OK**
- `{"status": "<判定した状態>"}`。状態を `status` キーでラップして返す。値は `not_started` / `in_progress` / `completed` のいずれか

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象タスクが存在しない、または他の生徒のタスク|404|`{"message": "タスクが見つかりません"}`（`ApplicationController#not_found`）。このControllerは独自のハンドラを持たないため、共通ハンドラの形になる|
|タスクに `goal_id` が設定されていない|404|`{"message": "目標が見つかりません"}`。目標の状態の再評価で `@user.goals.find(nil)` となり `ActiveRecord::RecordNotFound` が発生し、共通ハンドラが対象を `Goal` として本文を組み立てる。`tasks.goal_id` は NULL 可であり（「タスク管理機能」の「6. データモデル」）、API 経由で作成したタスクは必須だが既存データに NULL がありうる。タスクの状態の更新はトランザクションにより巻き戻る|

---

# 6. データモデル

## `question_histories`

- 役割: 生徒の問題解答履歴を記録するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|生徒（`users`）への外部キー|
|`task_id`|bigint|不可|タスク（`tasks`）への外部キー|
|`course_id`|bigint|不可|コース（`courses`）への外部キー。作成時に単元の所属コースを設定する|
|`unit_id`|bigint|不可|単元（`units`）への外部キー|
|`question_id`|bigint|不可|問題（`questions`）への外部キー|
|`question_choice_id`|bigint|**不可**|選択した選択肢（`question_choices`）への外部キー。必須であるため、選択肢を選ばない解答は記録できない|
|`answer_text`|text|可|テキスト解答|
|`time_spent_sec`|integer|可|解答時間（秒）。0 以上 3600 以下（Form の検証）|
|`is_correct`|boolean|不可|正誤。既定値は `false`|
|`explanation_viewed`|boolean|不可|解説を閲覧したか。既定値は `false`|
|`answered_at`|datetime|不可|解答日時。作成・更新のたびに現在時刻で設定される|
|`deleted_at`|datetime|可|論理削除日時。現行実装に値を設定する処理は存在しない|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション:
  - `belongs_to :user`
  - `belongs_to :course`
  - `belongs_to :unit`
  - `belongs_to :task`
  - `belongs_to :question`（**論理削除済みの問題も辿れるよう `deleted_at` の絞り込みを外している**）
  - `belongs_to :question_choice`（同様に絞り込みを外している）
- 絞り込み:
  - `active`: `deleted_at` が未設定のもの
  - `on_active_questions`: `active` かつ、紐づく問題が論理削除されていないもの。タスクの提出の判定に用いる
- **`default_scope` を持たない。** そのため、絞り込みを明示しない検索では論理削除済みの履歴も対象になる
- 同一のタスク・単元・問題の組に対する一意制約はない。重複する履歴が作られうる（「5. API / 処理詳細」`#create`）
- 論理削除済みの問題・選択肢でも履歴から辿れるようにしているのは、過去の解答内容を後から参照できるようにするためである

## 当日の解答件数（Redis。参照のみの所有）

- 保持先: **Redis**。RDB のテーブルではない
- この機能は解答の登録時に**加算する**。参照するのは「ダッシュボード機能」である
- キー・値・有効期限の詳細は「ダッシュボード機能」の「6. データモデル」を参照

## `questions` / `question_choices` / `question_hints`（参照のみ）

- 役割: 問題と、その選択肢・ヒント
- この機能が参照する項目はレスポンスに含まれるとおり
- `questions.correct_answer` は text 型であり、正誤判定では整数に変換して選択肢の番号と比較する（「5. API / 処理詳細」`#create`）
- この機能はこれらのテーブルを更新しない。問題の登録は「管理者問題インポート機能」の責務である

## エラーレスポンスの本文

**この機能は404の本文の形が2通りある。**

|条件|HTTP Status|本文|
|-|-|-|
|バリデーション失敗（Form）|422|`{"errors": ["メッセージ", ...]}`|
|`ConfirmationsController#show` で対象のタスク・単元が見つからない|404|`{"errors": ["対象データが見つかりません"]}`（Controller 独自の `rescue`）|
|上記以外で `ActiveRecord::RecordNotFound`|404|`{"message": "〇〇が見つかりません"}`（`ApplicationController#not_found`）。`〇〇` はモデル名の日本語訳であり、`Task` は `タスク`、`Goal` は `目標`、`Unit` は訳が定義されていないため `Unit` と英語になる|
|`Pundit::NotAuthorizedError`|403|`{"errors": ["この操作を行う権限がありません"]}`|

422 のメッセージは「項目名 + メッセージ本体」の形で組み立てられるが、この機能の Form（`Student::CreateQuestionHistoryForm` / `UpdateQuestionHistoryForm`）の項目名は日本語訳が定義されていないため、**項目名が英語のまま出る**。

|項目|メッセージに出る項目名|
|-|-|
|`task_id`|`Task`|
|`unit_id`|`Unit`|
|`question_id`|`Question`|
|`question_choice_id`|`Question choice`|
|`time_spent_sec`|`Time spent sec`|
|`explanation_viewed`|`Explanation viewed`|

`base` に積まれたメッセージ（解答履歴が見つからない場合、モデルの検証に失敗した場合）には項目名が付かない。

---

# 7. 状態管理

この機能自身に状態を持つデータはない。解答履歴は状態を持たず、正誤と解答日時を記録するのみである。

一方、この機能の操作は他機能の状態を変える。

|契機|波及先|内容|
|-|-|-|
|解答の登録（`#create`）|当日の解答件数（Redis）|1 加算する|
|タスクの提出（`#update`）|`tasks.status`・`tasks.completed_at`|解答状況から判定した状態へ更新する|
|同|`goals.status`|タスクが属する目標の状態を、目標に紐づくタスクの状況から再評価して更新する|
|解答の更新（`#update`）|—|波及しない|

タスクの状態の判定:

```
not_started（1件も解答していない）

↓ 1件以上解答した

in_progress

↓ すべての問題を解答した

completed
```

- すでに `completed` のタスクは、判定せず `completed` のままとする
- 対象の問題が0件の場合は `completed` と判定される（「5. API / 処理詳細」`#update`（提出））
- 判定は提出のたびに現在の解答状況から行う。解答を取り消す操作が存在しないため、`completed` から戻ることは通常起きない

---

# 8. 権限制御

- `student` ロールのユーザーのみ利用できる（`Api::V1::Student::BaseController` の認可）。
- すべての操作で `current_user.tasks` を起点にタスクを検索する。そのため生徒は自身のタスク・単元・問題のみ操作できる。
- 解答の更新では、操作者・タスク・単元・問題の4条件で解答履歴を検索するため、他の生徒の解答履歴は更新できない。
- 当日の解答件数のキーには操作者のユーザーIDを含めるため、他の生徒の件数を加算することはない。

---

# 9. 非同期処理

- Job / Mailer は見当たらない。
- ただし、解答の登録時に **Redis のカウンタを同期的に加算する**（`Student::IncrementTodayAnswerCountService`）。`INCR` と `EXPIREAT` をパイプラインでまとめて実行する。
- **Redis の障害時はログに記録するだけで、解答の記録自体は続行する。** 加算の失敗で解答が失敗することはない。
- 加算されたカウンタは「ダッシュボード機能」が参照する。詳細は同機能の「6. データモデル」を参照。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::QuestionsController`, `Api::V1::Student::AnswersController`, `Api::V1::Student::ConfirmationsController`, `Api::V1::Student::SubmissionsController`|
|Controller（共通）|`ApplicationController`（`not_found` 等の共通ハンドラ）, `Api::V1::Student::BaseController`（生徒向け API の認可）|
|Form|`Student::CreateQuestionHistoryForm`, `Student::UpdateQuestionHistoryForm`|
|Form の Concern|`TimeSpentSecValidatable`（解答時間を 0 以上 3600 以下に制限する）|
|Service|`Student::QuestionAnswerJudgeService`（正誤判定）, `Student::SubmissionService`（提出。タスクと目標の状態更新を1トランザクションで行う）, `Student::TaskCompletionService`（タスクのあるべき状態の判定）|
|Service（他機能）|`Student::TaskStatusUpdaterService`（タスク管理機能。タスクの状態と完了日時の更新）, `Student::GoalCompletionService` / `Student::GoalStatusUpdaterService`（目標管理機能。目標の状態の判定と更新）, `Student::IncrementTodayAnswerCountService`（当日の解答件数の加算。キーの構成は `Student::TodayAnswerCountKey`）|
|Query|`TaskAnswerStatsQuery`（解答済み件数と対象の問題数の集計。教員画面の進捗率と同じ集計を使う）|
|Model|`QuestionHistory`（`active`・`on_active_questions` を持つが `default_scope` は持たない）, `Question`, `QuestionChoice`, `QuestionHint`|
|Serializer|`QuestionSerializer`（`answered` を付与）, `QuestionConfirmationSerializer`（解答履歴がなければ正解を返さない）, `QuestionChoiceSerializer`, `QuestionHintSerializer`|
