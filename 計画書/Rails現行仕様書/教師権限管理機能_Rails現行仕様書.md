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
|`page`|任意|integer|ページ番号。1以上の整数以外を指定した場合は未指定として扱う|
|`per_page`|任意|integer|1ページあたりの件数。既定値 20、上限 100。1以上の整数以外を指定した場合は既定値を使う|

### 処理内容

1. `::Teacher::TeachersQuery.new(current_user.high_school.users).colleagues_for_permissions.result` を実行する。
2. `order(:name_kana).page(sanitized_page).per(sanitized_per_page)` でページングする。`sanitized_page` / `sanitized_per_page` は `ApplicationController` が提供する。
3. `::Teacher::TeacherPermissionManagementSerializer` で返却する。

### 業務ルール

- 同校の教員のみ対象とする。

### Database変更

- なし

### Response

- `current_user` : `TeacherSerializer`（`id`, `name`, `name_kana`, `grade`, `invitation_status`, `teacher_permission`）
- `teachers` : `::Teacher::TeacherPermissionManagementSerializer` 一覧（各要素は `id`, `name`, `teacher_permission`）
- `teacher_permission` : `TeacherPermissionSerializer`（`id`, `grade_scope`, `manage_other_teachers`）
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

`TeacherSerializer#invitation_status` は `instance_options[:latest_notifications]` から対象教員の最新の招待通知を引いて状態を返すが、本アクションは `latest_notifications` を渡していないため、常に既定値の `pending` を返す。

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

1. `before_action :set_teacher` が `teachers_query.active.find(params[:id])` で対象教員を検索する。
2. `::Teacher::TeacherPermissionManagementSerializer` で返却する。

### 業務ルール

- 同校の教員のみ対象とする。
- 論理削除済み（`deleted_at` が設定されている）の教員は対象外とする。

### Database変更

- なし

### Response

- `id`, `name`, `teacher_permission`（`TeacherPermissionSerializer`。`id`, `grade_scope`, `manage_other_teachers`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象教員が存在しない、または論理削除済み|404|`ApplicationController` の `ActiveRecord::RecordNotFound` ハンドラが `{ message: "<モデル名>が見つかりません" }` を返す（モデル名は `User.model_name.human`）|

## `Api::V1::Teacher::PermissionsController#update`

### 概要

指定教員の権限を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`teacher_permission[grade_scope]`|必須|string|学年スコープ|
|`teacher_permission[manage_other_teachers]`|必須|boolean|他教員管理権限|

### 処理内容

1. `before_action :set_teacher` が `teachers_query.active.find(params[:id])` で対象教員を検索する。
2. `before_action` が `require_manage_other_teachers!('他教員を編集する権限がありません')` を実行し、操作者に他教員管理権限がなければ 403 を返す。
3. 対象教員が自身のアカウントであれば更新を拒否する。
4. 対象教員が校内で最後の有効教員であれば更新を拒否する。
5. `Teacher::UpdatePermissionForm` により `teacher_permission` を更新する。

処理順は `before_action` の登録順に従い、対象教員の特定（2.の権限判定より先）→ 権限判定 → 自己更新・最後の教員の判定となる。存在しない ID を指定した場合は、権限の有無によらず 404 が返る。

### 業務ルール

- 自分自身の権限は更新できない。
- 校内の最後の有効教員の権限は変更できない。判定は、同校の有効な教員のうち対象教員以外が存在するかどうかで行う（`teachers_query.active.where.not(id: target.id).exists?`）。
- `grade_scope` は `TeacherPermission.grade_scopes.keys`（`own_grade` / `all_grades`）のいずれか。
- `manage_other_teachers` は boolean である必要がある。

### Database変更

- `teacher_permissions` の更新

### Response

- `message`: `権限更新に成功しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|権限なし|403|`他教員を編集する権限がありません`|
|対象教員が存在しない、または論理削除済み|404|`ActiveRecord::RecordNotFound` ハンドラのメッセージ|
|自身の更新|422|`自分自身は更新できません`|
|最後の教員の更新|422|`最後の教員は更新できません`|
|フォームのバリデーション失敗|422|`errors` に `form.errors.full_messages` を返却|
|`teacher_permissions` のモデル検証失敗|422|`ApplicationController` の `ActiveRecord::RecordInvalid` ハンドラが `errors` にモデルの `full_messages` を返却|

---

# 6. データモデル

## `teacher_permissions`

- 役割: 教師の権限を管理するテーブル
- 主なカラム: `user_id`, `grade_scope`, `manage_other_teachers`

---

# 7. 権限制御

- 更新には他教員管理権限が必要である。判定は `RequireManageOtherTeachers` concern の `require_manage_other_teachers!` が行い、`current_user.teacher_permission&.manage_other_teachers?` が真でなければ 403 を返す。`teacher_permission` が未設定の教師は権限なしとして扱う。
- 一覧取得・詳細取得に他教員管理権限は不要である。
- 対象教員は同校に所属している必要がある。
- 教師ロール自体の判定は `Api::V1::Teacher::BaseController` が Pundit の `authorize :teacher_service, :access?` で行う。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::PermissionsController`|
|Concern|`RequireManageOtherTeachers`|
|Query|`Teacher::TeachersQuery`（`colleagues_for_permissions`）|
|Form|`Teacher::UpdatePermissionForm`|
|Serializer|`Teacher::TeacherPermissionManagementSerializer`, `TeacherPermissionSerializer`, `TeacherSerializer`|
|Model|`TeacherPermission`|
