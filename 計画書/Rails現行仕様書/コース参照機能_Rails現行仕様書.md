# コース参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 学習コースとその単元一覧を取得すること。
- システム上の役割
  - コース一覧を表示し、科目別検索をサポートする。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|コースと単元を取得する|
|科目絞り込み|指定科目のコースを取得する|

---

# 3. 業務フロー

1. 生徒がコース一覧を表示する。
2. システムはコースと関連単元を読み込む。
3. 必要に応じて `subject` で絞り込む。
4. JSON 形式で返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::CoursesController`|`index`|GET|`/api/v1/student/courses`|コース一覧を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::CoursesController#index`

### 概要

コースと関連単元を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`subject`|任意|string|科目名による絞り込み|

### 処理内容

1. `CoursesQuery` でコースを取得し、`includes_units` で単元を同時に読み込む（N+1 を避けるため）。
2. `subject` が指定されている場合は、`join_subject` で科目と結合し `by_subject` で科目名により絞り込む。このとき 1 で組み立てたクエリは使わず、**`CoursesQuery` を新しく生成してやり直す**。
3. `as_json(include: :units)` で返却する。**シリアライザを使わない。**

### 業務ルール

- `subject` が指定されない場合はすべてのコースを返す。
- `subject` が指定される場合はその科目に属するコースのみ返す。絞り込みは科目名（`subjects.name`）の完全一致である。
- **論理削除済み（`deleted_at` が設定済み）のコース・単元も返る。** `Course`・`Unit` はいずれも `active`（`deleted_at` が未設定のもの）という絞り込みを持つが、`default_scope` を持たず、この機能では `active` を使っていない。
- 一覧の並び順は指定しておらず、利用者が任意の順序を指定する機能も提供しない。返却順序はデータベースから取得した順序に依存する。
- ページングを行わない。コースの全件を返す。

### Database変更

- なし

### Response

最上位は**配列**である（キーでラップしない）。

`as_json` を用いているため、**シリアライザで項目を選ばず、テーブルの全カラムがそのまま出る**。

- コース属性（`courses` の全カラム）:
  - `id`
  - `subject_id`（NULL 可）
  - `level_number`
  - `level_name`
  - `description`（NULL 可）
  - **`deleted_at`**（NULL 可）
  - **`created_at`**
  - **`updated_at`**
- `units`（`units` の全カラム）:
  - `id`
  - `course_id`
  - `unit_name`
  - **`deleted_at`**（NULL 可）
  - **`created_at`**
  - **`updated_at`**

`deleted_at`・`created_at`・`updated_at` が含まれるのは、他機能のシリアライザを使ったレスポンスとは異なる点である。日付・日時の整形も行わない。

科目の情報（`subjects` の属性）は含まれない。絞り込みに使うのみである。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|未認証|401|認証エラー|
|`student` ロールでない|403|`{"errors": ["この操作を行う権限がありません"]}`|

コースが 0 件の場合も空配列を返す。`subject` に存在しない科目名を指定した場合も空配列になり、エラーにしない。

---

# 6. データモデル

## `courses`

- 役割: 学習コースを管理するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`subject_id`|bigint|**可**|科目（`subjects`）への外部キー。カラムは NULL 可だが、`belongs_to :subject` により登録・更新時は必須である|
|`level_number`|integer|不可|レベル番号。既定値は `1`|
|`level_name`|string(255)|不可|レベル名|
|`description`|text|可|説明|
|`deleted_at`|datetime|可|論理削除日時|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション:
  - `belongs_to :subject`
  - `has_many :units`（`dependent: :destroy`）
  - `has_one :review_test`、`has_many :reflections`・`question_histories`・`task_courses`・`tasks`（`task_courses` 経由）・`user_unit_question_stats`
- `active`（`deleted_at` が未設定のもの）という絞り込みを持つが、**`default_scope` は持たない**

## `units`

- 役割: コースに含まれる学習単元を管理するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`course_id`|bigint|不可|コース（`courses`）への外部キー|
|`unit_name`|string(255)|不可|単元名|
|`deleted_at`|datetime|可|論理削除日時|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション:
  - `belongs_to :course`
  - `has_many :questions`（`dependent: :destroy`）
  - `has_many :task_units`・`tasks`（`task_units` 経由）・`question_histories`・`study_logs`・`user_unit_question_stats`・`import_histories`
- `active` という絞り込みを持つが、**`default_scope` は持たない**

## `subjects`（参照のみ）

- 役割: 科目。コースの絞り込みに用いる
- この機能が参照する項目: `name`（絞り込み条件）
- レスポンスには含まれない

---

# 7. 状態管理

- なし

---

# 8. 権限制御

- `student` ロールのユーザーのみ利用できる（`Api::V1::Student::BaseController` の認可）。
- 操作者による絞り込みは行わない。コースは全生徒に共通のマスタデータであり、どの生徒も同じ一覧を取得する。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::CoursesController`|
|Controller（共通）|`ApplicationController`, `Api::V1::Student::BaseController`（生徒向け API の認可）|
|Query|`CoursesQuery`（`join_subject`・`includes_units`・`by_subject`。`active` は持つが使っていない）|
|Model|`Course`（`active` を持つが `default_scope` は持たない）, `Unit`（同）, `Subject`（絞り込みのみ）|
|Serializer|**なし**（`as_json(include: :units)` を使用。テーブルの全カラムがそのまま出る）|
