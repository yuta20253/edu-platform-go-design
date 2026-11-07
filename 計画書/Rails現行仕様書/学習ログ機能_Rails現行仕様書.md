# 学習ログ機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒がタスク内の単元学習にどれだけ取り組んだかを記録すること。
- システム上の役割
  - 生徒が単元の学習を開始したタイミングで学習ログを作成し、学習を終えたタイミングで終了時刻と所要時間を記録する。
  - 同一のタスク・単元に対して、学習セッションごとに複数の学習ログを記録できる。
  - **学習ログの作成を契機に、対象タスクの状態を「未着手」から「進行中」へ変える。** さらにタスクが属する目標の状態も再評価される（`Student::TaskStartService` を呼び出す。「5. API / 処理詳細」`#create`、「タスク管理機能」の「5. API / 処理詳細」の「タスクの着手」）。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|学習開始記録|タスク内の単元学習を開始した記録を作成する|
|学習終了記録|開始済みの学習ログを完了させ、所要時間を記録する|

---

# 3. 業務フロー

1. 生徒がタスク内の単元学習を開始する。
2. システムは、そのタスクが生徒本人のものであり、対象の単元がタスクに紐づいていることを確認する。
3. 学習中状態の学習ログを、開始時刻とともに新規作成する。
4. 続けて、対象タスクが未着手であれば「進行中」へ変える。タスクが属する目標の状態も、目標に紐づくタスクの状況から再評価して更新する。3とこの処理は1つのトランザクションで行う。
5. 生徒が単元学習を終える。
6. システムは、対象の学習ログが生徒本人のものであり、未完了（学習中）であることを確認する。
7. 終了時刻を記録し、開始時刻からの経過時間（分単位）を算出したうえで、学習ログを完了状態に更新する。学習終了ではタスク・目標の状態を変えない。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::StudyLogsController`|`create`|POST|`/api/v1/student/tasks/:task_id/units/:unit_id/study_logs`|単元学習の開始を記録|
|`Api::V1::Student::StudyLogsController`|`update`|PATCH|`/api/v1/student/tasks/:task_id/units/:unit_id/study_logs/:id`|学習ログを完了させる|

---

# 5. API / 処理詳細

## `Api::V1::Student::StudyLogsController#create`

### 概要

生徒がタスク内の単元学習を開始したことを記録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスクの識別子|
|`unit_id`|必須|integer|単元の識別子|

### 処理内容

1. `current_user` 本人のタスクの中から、指定した `task_id` のタスクを取得する。見つからない場合はエラーとする。
2. そのタスクに紐づく単元の中から、指定した `unit_id` の単元を取得する。見つからない場合はエラーとする。
3. 学習中状態の学習ログを、現在時刻を開始時刻として新規作成する。
4. 続けて `Student::TaskStartService` を呼び出す。対象タスクが未着手であれば「進行中」へ変え、タスクが属する目標の状態も再評価して更新する。タスクが既に進行中・完了であれば何もしない。
5. 3と4は1つのトランザクションで行う。いずれかが失敗した場合は学習ログの作成も巻き戻る。
6. 作成した学習ログのIDを返却する。

### 業務ルール

- 対象のタスクは、ログイン中の生徒自身のタスクである必要がある。
- 対象の単元は、そのタスクに紐づいている単元である必要がある。
- 同一のタスク・単元の組み合わせに対して、学習ログを何度でも新規作成できる（過去に完了済みの学習ログが存在していても新規作成は可能）。学習中の学習ログが既にある場合も、重複を防ぐ仕組みはない。
- **学習ログの作成は、対象タスクと、そのタスクが属する目標の状態に波及する。** 波及の内容は「タスク管理機能」の「5. API / 処理詳細」の「タスクの着手」および「目標管理機能」の「7. 状態管理」を参照。

### Database変更

作成:

```
study_logs
```

更新（`Student::TaskStartService` による波及）:

```
tasks（status・completed_at。対象タスクが未着手だった場合のみ）
goals（status。上記でタスクの状態が変わった場合）
```

### Response

- **201 Created**
- `{"study_log_id": <作成した学習ログのID>}`（識別子を `study_log_id` キーでラップして返す）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象タスクが存在しない、または自分のタスクでない|404|`{"message": "タスクが見つかりません"}`（`ApplicationController#not_found`）|
|対象単元がタスクに紐づいていない|404|`{"message": "Unitが見つかりません"}`。**モデル名の日本語訳が定義されていないため、名称が英語のまま出る**|
|未認証|401|認証エラー|

## `Api::V1::Student::StudyLogsController#update`

### 概要

開始済みの学習ログを完了させ、終了時刻と所要時間を記録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスクの識別子|
|`unit_id`|必須|integer|単元の識別子|
|`id`|必須|integer|学習ログの識別子|

### 処理内容

1. `current_user` 本人の、指定したタスク・単元に紐づく学習ログを、指定IDで取得する。見つからない場合はエラーとする。
2. 対象の学習ログが既に完了状態の場合はエラーとし、処理を中止する。
3. 現在時刻を終了時刻として記録し、開始時刻からの経過時間を分単位（切り捨て）で算出する。
4. 学習ログを完了状態に更新する。
5. 更新後の学習ログを返却する。

なお、タスクの取得（`task_id`）と単元の取得（`unit_id`）も `#create` と同じ手順で行う。いずれかが見つからない場合は学習ログの検索に至らずエラーとなる。

### 業務ルール

- 対象の学習ログは、ログイン中の生徒自身の、指定したタスク・単元に紐づくものである必要がある。
- 既に完了している学習ログを重ねて完了させることはできない。
- 所要時間は「終了時刻 − 開始時刻」を60で割った分単位の値であり、**小数を切り捨てる**。1分未満の学習は `0` になる。
- 学習終了ではタスク・目標の状態を変えない。波及するのは学習開始のみである。

### Database変更

更新:

```
study_logs
```

### Response

- **200 OK**
- `id`, `status`（`studying` / `completed` の文字列）, `started_at`, `ended_at`, `duration_minutes`
- `started_at`・`ended_at` は**日付フォーマットの加工を行わず、日時のまま返す**（他機能のシリアライザが `YYYY/MM/DD` に整形しているのとは異なる）
- `user_id`・`task_id`・`unit_id` は含まれない

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象タスクが存在しない、または自分のタスクでない|404|`{"message": "タスクが見つかりません"}`|
|対象単元がタスクに紐づいていない|404|`{"message": "Unitが見つかりません"}`|
|対象の学習ログが存在しない、または自分のものでない|404|`{"message": "Study logが見つかりません"}`。**モデル名の日本語訳が定義されていないため、名称が英語のまま出る**|
|対象の学習ログが既に完了している|**400**|`{"errors": ["この学習ログはすでに完了しています"]}`。`Student::AlreadyCompletedStudyLogError` を `Api::V1::Student::BaseController` が `rescue_from` で捕まえ、400 で返す|
|未認証|401|認証エラー|

---

# 6. データモデル

## `study_logs`

- 役割: 生徒がタスク内の単元学習に取り組んだ開始・終了時刻と所要時間を記録するテーブル。
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|生徒（`users`）への外部キー|
|`task_id`|bigint|不可|タスク（`tasks`）への外部キー|
|`unit_id`|bigint|不可|単元（`units`）への外部キー|
|`status`|integer|不可|状態。Rails の `enum` で `studying: 0` / `completed: 1` に対応する整数。既定値は `0`。**2値のみで、他機能の `status`（3値）とは異なる**|
|`started_at`|datetime|**不可**|学習開始時刻。作成時に現在時刻が入る|
|`ended_at`|datetime|可|学習終了時刻。完了時に現在時刻が入る|
|`duration_minutes`|integer|可|所要時間（分）。完了時に算出して入る|
|`deleted_at`|datetime|可|論理削除日時。現行実装に値を設定する処理は存在しない|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション:
  - `belongs_to :user`
  - `belongs_to :task`
  - `belongs_to :unit`
- `active` という絞り込み（`deleted_at` が未設定のもの）を持つが、**`default_scope` は持たない**。そのため学習ログの検索では論理削除済みのレコードも対象に含まれる。この機能の `#update` も `active` を使っていない
- 同一のタスク・単元の組に対する一意制約はない。学習セッションごとに複数行が作られる

## エラーレスポンスの本文

この機能のエラー本文は共通の例外ハンドラが決める。本文の形は条件によって異なる。

|条件|HTTP Status|本文|
|-|-|-|
|`ActiveRecord::RecordNotFound`|404|`{"message": "〇〇が見つかりません"}`（`ApplicationController#not_found`）。`〇〇` はモデル名の日本語訳であり、`Task` は `タスク`、`Unit` と `StudyLog` は訳が定義されていないため `Unit`・`Study log` と英語になる|
|`Student::AlreadyCompletedStudyLogError`|400|`{"errors": ["この学習ログはすでに完了しています"]}`（`Api::V1::Student::BaseController#bad_request`）|
|`Pundit::NotAuthorizedError`|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

# 7. 状態管理

```
studying（学習中）

↓

completed（完了）
```

状態変更条件:

- 学習ログ作成時は「学習中」状態で開始される。DB 上は整数（`studying: 0` / `completed: 1`）。
- 生徒が学習終了操作を行うと「完了」状態に変わり、終了時刻・所要時間が確定する。
- 一度「完了」状態になった学習ログを再度完了させることはできない。重ねて完了させようとすると 400 になる。
- 「学習中」へ戻す操作は存在しない。

この機能の操作は、学習ログ自身の状態のほかに、他機能の状態も変える。

|契機|波及先|内容|
|-|-|-|
|学習開始（`#create`）|`tasks.status`|未着手のタスクを「進行中」にする（`Student::TaskStartService`）|
|同|`goals.status`|上記でタスクの状態が変わった場合、目標に紐づくタスクの状況から目標の状態を再評価して更新する|
|学習終了（`#update`）|—|波及しない|

---

# 8. 権限制御

- `student` ロールのユーザーのみ利用できる。
- 学習ログの作成・完了は、いずれもログイン中の生徒自身のタスク・単元・学習ログに対してのみ行える。他の生徒のタスクや学習ログは操作できない。

---

# 9. 非同期処理

- `StudyLog` の作成・完了に紐づく Job / Mailer は見当たらない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::StudyLogsController`|
|Controller（共通）|`ApplicationController`（`not_found` 等の共通ハンドラ）, `Api::V1::Student::BaseController`（生徒向け API の認可と、`Student::AlreadyCompletedStudyLogError` → 400 等の例外ハンドラ）|
|Service|`Student::CreateStudyLogService`（学習ログの作成と `Student::TaskStartService` の呼び出しを1トランザクションで行う）, `Student::CompleteStudyLogService`（完了と所要時間の算出）|
|Service（他機能）|`Student::TaskStartService`（タスク管理機能。タスクの着手と目標の状態の再評価）|
|エラークラス|`Student::AlreadyCompletedStudyLogError`（メッセージの配列を保持する）|
|Model|`StudyLog`（`active` の絞り込みを持つが `default_scope` は持たない）|
|Serializer|`StudyLogSerializer`（`id`・`status`・`started_at`・`ended_at`・`duration_minutes` のみ。日付の整形を行わない）|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
