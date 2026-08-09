# 管理者教員管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が同校の教員一覧を参照し、招待中教員の登録・更新を行えるようにすること。
- システム上の役割
  - 同校教員一覧を取得する。
  - 新規教員アカウント招待を作成する。
  - 既存教員のプロフィール・権限・担当学年を更新する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|教員一覧取得|学校ごとの教員一覧を取得する|
|教員登録|新規教員を招待する|
|教員更新|既存教員の情報を更新する|

---

# 3. 業務フロー

1. 管理者が高校選択後に教員一覧画面を表示する。
2. システムは対象高校の教員一覧を取得する。
3. 新規教員を招待する場合はメールアドレスを指定して作成する。
4. 既存教員の情報や権限を更新する場合は対象教員を選択して変更を保存する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::TeachersController`|`index`|GET|`/api/v1/admin/high_schools/:high_school_id/teachers`|高校の教員一覧を取得|
|`Api::V1::Admin::TeachersController`|`create`|POST|`/api/v1/admin/high_schools/:high_school_id/teachers`|新規教員を招待|
|`Api::V1::Admin::TeachersController`|`update`|PATCH|`/api/v1/admin/high_schools/:high_school_id/teachers/:id`|教員情報を更新|

---

# 5. API / 処理詳細

## `Api::V1::Admin::TeachersController#index`

### 概要

対象高校の教員一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`high_school_id`|必須|integer|高校の ID|

### 処理内容

1. `HighSchool.find(params[:high_school_id])` を取得する。
2. `school.users.teachers.includes(:teacher_permission, :grades)` で教員一覧を取得する。
3. `Admin::TeacherSerializer` で返却する。

### 業務ルール

- 同校の教員のみ対象とする。

### Database変更

- なし

### Response

- `teachers` : 各教員の `id`, `name`, `email`, `grade_scope`, `manage_other_teachers`, `grades`

### Errorケース

- 対象高校が存在しない|404|対象高校なし

## `Api::V1::Admin::TeachersController#create`

### 概要

新規教員を招待する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`high_school_id`|必須|integer|高校の ID|
|`email`|必須|string|メールアドレス|

### 処理内容

1. `HighSchool.find(params[:high_school_id])` を取得する。
2. `Admin::CreateTeacherService.new(school: school, email: create_params[:email]).call` を実行する。
3. `Admin::TeacherSerializer` で招待対象ユーザーを返却する。

### 業務ルール

- `email` は必須。
- 対象高校に紐づく教員として作成される。

### Database変更

- `users` に教員レコード作成
- `teacher_permissions` に権限レコード作成

### Response

- `teacher` : `Admin::TeacherSerializer`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Admin::TeachersController#update`

### 概要

既存教員のプロフィール・権限・担当学年を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`high_school_id`|必須|integer|高校の ID|
|`id`|必須|integer|教員の ID|
|`name`|任意|string|名前|
|`email`|任意|string|メールアドレス|
|`grade_scope`|任意|integer|閲覧権限スコープ|
|`manage_other_teachers`|任意|boolean|他教員管理権限|
|`grade_ids`|任意|array|担当学年 ID 配列|

### 処理内容

1. `HighSchool.find(params[:high_school_id])` を取得する。
2. `school.users.teachers.find(params[:id])` で対象教員を取得する。
3. `Admin::UpdateTeacherService.new(user: user, params: update_params).call` を実行する。
4. `Admin::TeacherSerializer` で返却する。

### 業務ルール

- `grade_ids` が指定された場合、既存の担当学年を全て置き換える。

### Database変更

- `users`, `teacher_permissions`, `teacher_grades` を更新

### Response

- `teacher` : `Admin::TeacherSerializer`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

---

# 6. データモデル

## `teacher_permissions`

- 役割: 教師の閲覧権限を管理するテーブル
- 主なカラム: `user_id`, `grade_scope`, `manage_other_teachers`

## `teacher_grades`

- 役割: 教師の担当学年を管理するテーブル
- 主なカラム: `user_id`, `grade_id`

---

# 7. 権限制御

- 管理者は同校の教員のみ対象とする。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::TeachersController`|
|Service|`Admin::CreateTeacherService`, `Admin::UpdateTeacherService`|
|Serializer|`Admin::TeacherSerializer`|
|Model|`User`, `TeacherPermission`, `TeacherGrade`|
