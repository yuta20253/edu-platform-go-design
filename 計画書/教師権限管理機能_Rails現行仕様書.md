# 教師権限管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師が同校教員の権限を確認・更新できるようにすること。
- システム上の役割
  - 教員権限一覧と権限詳細を提供する。
  - 他教員の `grade_scope` と `manage_other_teachers` を更新する。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|権限管理対象の教員一覧を取得する|
|詳細取得|指定教員の権限情報を取得する|
|更新|指定教員の権限を更新する|

---

# 3. 業務フロー

1. 教師が権限管理画面を表示する。
2. システムは同校教員の権限情報を取得する。
3. 教師が対象教員を選択すると、権限詳細を取得する。
4. 他教員の権限を更新する場合、変更内容を送信する。
5. `manage_other_teachers` 権限の教師のみ更新処理を許可する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::PermissionsController`|`index`|GET|`/api/v1/teacher/permissions`|教員権限一覧を取得|
|`Api::V1::Teacher::PermissionsController`|`show`|GET|`/api/v1/teacher/permissions/:id`|教員権限詳細を取得|
|`Api::V1::Teacher::PermissionsController`|`update`|PATCH|`/api/v1/teacher/permissions/:id`|教員権限を更新|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::PermissionsController#index`

### 概要

教員の権限管理対象リストを取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号|

### 処理内容

1. `::Teacher::TeachersQuery.new(current_user.high_school.users).colleagues_for_permissions.result` を実行する。
2. `order(:name_kana).page(params[:page]).per(20)` でページングする。
3. `TeacherPermissionManagementSerializer` で返却する。

### 業務ルール

- 同校の教員のみ対象とする。

### Database変更

- なし

### Response

- `current_user` : `TeacherSerializer`
- `teachers` : `TeacherPermissionManagementSerializer` 一覧
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::PermissionsController#show`

### 概要

指定教員の権限詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|教員の ID|

### 処理内容

1. `teachers_query.active.find(params[:id])` で対象教員を検索する。
2. `TeacherPermissionManagementSerializer` で返却する。

### 業務ルール

- 同校の教員のみ対象とする。

### Database変更

- なし

### Response

- `id`, `name`, `teacher_permission`

### Errorケース

- 対象教員が存在しない|404|対象教員なし

## `Api::V1::Teacher::PermissionsController#update`

### 概要

指定教員の権限を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`teacher_permission[grade_scope]`|必須|string|学年スコープ|
|`teacher_permission[manage_other_teachers]`|必須|boolean|他教員管理権限|

### 処理内容

1. `current_user.teacher_permission.manage_other_teachers?` が false の場合、エラーを返却する。
2. `teachers_query.active.find(params[:id])` で対象教員を検索する。
3. 自身のアカウントであれば更新を拒否する。
4. 校内で最後の有効教員であれば更新を拒否する。
5. `Teacher::UpdatePermissionForm` により `teacher_permission` を更新する。

### 業務ルール

- 自分自身の権限は更新できない。
- 校内の最後の有効教員の権限は変更できない。
- `grade_scope` は `TeacherPermission.grade_scopes.keys` のいずれか。
- `manage_other_teachers` は boolean である必要がある。

### Database変更

- `teacher_permissions` の更新

### Response

- `message`: `権限更新に成功しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|自身の更新|422|`自分自身は更新できません`|
|最後の教員の更新|422|`最後の教員は更新できません`|
|権限なし|422|`他教員を編集する権限がありません`|
|バリデーション失敗|422|`errors` を返却|

---

# 6. データモデル

## `teacher_permissions`

- 役割: 教師の権限を管理するテーブル
- 主なカラム: `user_id`, `grade_scope`, `manage_other_teachers`

---

# 7. 権限制御

- `current_user.teacher_permission.manage_other_teachers?` が必要。
- 対象教員は同校に所属している必要がある。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::PermissionsController`|
|Form|`Teacher::UpdatePermissionForm`|
|Serializer|`TeacherPermissionManagementSerializer`, `TeacherSerializer`|
|Model|`TeacherPermission`|
