# 管理者高校学年参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が高校と学年の一覧および詳細を参照できるようにすること。
- システム上の役割
  - 高校情報の一覧と詳細を提供する。
  - 各高校の生徒数・教員数を集計する。
  - 各高校の学年一覧を提供する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|高校一覧取得|高校一覧を取得し、生徒数・教員数を表示する|
|高校詳細取得|指定高校の詳細と生徒数・教員数を取得する|
|学年一覧取得|指定高校の学年一覧を取得する|

---

# 3. 業務フロー

1. 管理者が高校一覧画面を開く。
2. システムは高校ごとの生徒数・教員数を集計して返却する。
3. 指定高校の詳細を表示する場合は高校詳細を取得する。
4. 指定高校の学年一覧を取得する場合はその高校に紐づく学年を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::HighSchoolsController`|`index`|GET|`/api/v1/admin/high_schools`|高校一覧を取得|
|`Api::V1::Admin::HighSchoolsController`|`show`|GET|`/api/v1/admin/high_schools/:id`|高校詳細を取得|
|`Api::V1::Admin::GradesController`|`index`|GET|`/api/v1/admin/high_schools/:high_school_id/grades`|高校の学年一覧を取得|

---

# 5. API / 処理詳細

## `Api::V1::Admin::HighSchoolsController#index`

### 概要

高校一覧と各高校の生徒数・教員数を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`prefecture_id`|任意|integer|都道府県で絞り込む|
|`page`|任意|integer|ページ番号|

### 処理内容

1. `HighSchool.includes(:prefecture).order(:id).by_prefecture(params[:prefecture_id]).page(params[:page]).per(20)` で高校を取得する。
2. 対象高校の `id` を抽出し、生徒数・教員数を `User.students.by_high_school` / `User.teachers.by_high_school` で集計する。
3. `Admin::HighSchoolSerializer` に `student_counts` / `teacher_counts` を渡して返却する。

### 業務ルール

- `prefecture_id` が指定された場合は都道府県で絞り込む。

### Database変更

- なし

### Response

- `schools` : 各高校の `id`, `name`, `prefecture_name`, `student_count`, `teacher_count`
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Admin::HighSchoolsController#show`

### 概要

指定高校の詳細と生徒数・教員数を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|高校の ID|

### 処理内容

1. `HighSchool.includes(:prefecture).find(params[:id])` で高校を取得する。
2. `User.students.by_high_school(school.id).count` と `User.teachers.by_high_school(school.id).count` を集計する。
3. `Admin::HighSchoolSerializer` を返却する。

### 業務ルール

- なし

### Database変更

- なし

### Response

- `id`, `name`, `prefecture_name`, `student_count`, `teacher_count`

### Errorケース

- 対象高校が存在しない|404|対象高校なし

## `Api::V1::Admin::GradesController#index`

### 概要

指定高校に属する学年一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`high_school_id`|必須|integer|高校の ID|

### 処理内容

1. `HighSchool.find(params[:high_school_id])` で高校を取得する。
2. `school.grades.order(:year)` で学年一覧を取得する。
3. `Admin::GradeSerializer` で返却する。

### 業務ルール

- なし

### Database変更

- なし

### Response

- 各学年の `id`, `year`, `display_name`

### Errorケース

- 対象高校が存在しない|404|対象高校なし

---

# 6. データモデル

## `high_schools`

- 役割: 高校を管理するテーブル
- 主なカラム: `id`, `name`, `prefecture_id`

## `grades`

- 役割: 高校に所属する学年を管理するテーブル
- 主なカラム: `id`, `high_school_id`, `year`

---

# 7. 権限制御

- 管理者はすべての高校・学年を参照可能。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::HighSchoolsController`, `Admin::GradesController`|
|Serializer|`Admin::HighSchoolSerializer`, `Admin::GradeSerializer`|
|Model|`HighSchool`, `Grade`, `User`|
