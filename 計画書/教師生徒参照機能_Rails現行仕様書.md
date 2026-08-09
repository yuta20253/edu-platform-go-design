# 教師生徒参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師が同校の生徒一覧および生徒詳細を確認できるようにすること。
- システム上の役割
  - 同校生徒の絞り込みとページングを提供する。
  - 学年担当権限がある教師は担当学年のみ閲覧できる。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|同校の生徒一覧を取得する|
|詳細取得|同校の指定生徒の詳細を取得する|

---

# 3. 業務フロー

1. 教師が生徒一覧画面を表示する。
2. システムは同校の生徒を取得し、必要に応じて担当学年で絞り込む。
3. ページ番号に応じて 10 件ずつ返却する。
4. 教師が生徒を選択すると、対象生徒の詳細情報を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::StudentsController`|`index`|GET|`/api/v1/teacher/students`|同校の生徒一覧を取得|
|`Api::V1::Teacher::StudentsController`|`show`|GET|`/api/v1/teacher/students/:id`|生徒詳細を取得|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::StudentsController#index`

### 概要

同校の生徒一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号|

### 処理内容

1. `::Teacher::StudentsQuery.new(current_user.high_school.users)` を初期化する。
2. `current_user.teacher_permission.own_grade?` の場合は `grade_id: current_user.grade_id` で絞り込む。
3. `order(:name_kana).page(params[:page]).per(10)` でページングする。
4. `StudentSerializer` で返却する。

### 業務ルール

- `own_grade` 権限の教師は自身の学年のみ参照できる。
- 同校の生徒のみ対象とする。

### Database変更

- なし

### Response

- `students` : `StudentSerializer`
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::StudentsController#show`

### 概要

同校の指定生徒の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|生徒の ID|

### 処理内容

1. `students_query.find(params[:id])` で対象生徒を検索する。
2. `StudentSerializer` で返却する。

### 業務ルール

- `own_grade` 権限時は担当学年の生徒のみ取得可能。
- 同校以外の生徒は対象外。

### Database変更

- なし

### Response

- 生徒の基本情報と関連情報（`StudentSerializer` の内容）

### Errorケース

- 対象生徒が存在しない|404|対象生徒なし

---

# 6. データモデル

## `users`

- 役割: ユーザー情報を管理するテーブル
- 主なカラム: `id`, `name`, `name_kana`, `email`, `grade_id`, `high_school_id`
- リレーション: `belongs_to :grade`, `belongs_to :high_school`

---

# 7. 権限制御

- `current_user.teacher_permission.own_grade?` の場合、`grade_id` で絞り込みを行う。
- 同校の生徒のみ参照可能。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::StudentsController`|
|Query|`Teacher::StudentsQuery`|
|Serializer|`StudentSerializer`|
|Model|`User`, `Grade`, `HighSchool`|
