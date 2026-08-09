# 教師教員管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師が同校の教員一覧を確認し、新規教員を招待できるようにすること。
- システム上の役割
  - 同校教員の一覧表示と詳細表示を提供する。
  - 新規教員アカウントの作成を受付ける。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|同校の教員一覧を取得する|
|詳細取得|指定教員の詳細を取得する|
|新規登録|教員アカウントを作成する|

---

# 3. 業務フロー

1. 教師が教員一覧画面を表示する。
2. システムは同校の教員を取得して表示する。
3. 教師が教員を選択すると、詳細情報を取得する。
4. 新規教員を追加する場合、メールアドレス・名前などを入力して作成する。
5. 招待メールを送信して教員アカウントを準備する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::TeachersController`|`index`|GET|`/api/v1/teacher/colleagues`|同校の教員一覧を取得|
|`Api::V1::Teacher::TeachersController`|`show`|GET|`/api/v1/teacher/colleagues/:id`|教員詳細を取得|
|`Api::V1::Teacher::TeachersController`|`create`|POST|`/api/v1/teacher/colleagues`|新規教員を作成|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::TeachersController#index`

### 概要

同校の教員一覧をページングして取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号|

### 処理内容

1. `::Teacher::TeachersQuery.new(current_user.high_school.users).colleagues.result` を実行する。
2. `order(:name_kana).page(params[:page]).per(20)` でページングする。
3. `TeacherSerializer` で JSON を返却する。

### 業務ルール

- 同校の教員のみ対象とする。

### Database変更

- なし

### Response

- `current_user` : `TeacherSerializer`
- `teachers` : `TeacherSerializer` 一覧
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::TeachersController#show`

### 概要

同校の指定教員の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|教員の ID|

### 処理内容

1. `teachers_query.find(params[:id])` で対象教員を検索する。
2. `TeacherSerializer` で返却する。

### 業務ルール

- 同校の教員のみ取得する。

### Database変更

- なし

### Response

- `id`, `name`, `email`, `grade_scope`, `manage_other_teachers`, `grades`

### Errorケース

- 対象教員が存在しない|404|対象教員なし

## `Api::V1::Teacher::TeachersController#create`

### 概要

新しい教員アカウントを作成する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`name`|必須|string|氏名|
|`name_kana`|必須|string|氏名（カタカナ）|
|`email`|必須|string|メールアドレス|
|`grade_id`|必須|integer|担当学年の ID|
|`grade_scope`|必須|integer|閲覧可能学年スコープ|
|`manage_other_teachers`|必須|boolean|他教員管理権限|

### 処理内容

1. `create_teacher_params` を受け取る。
2. `Teacher::CreateTeacherForm.new(current_user: current_user, **params)` を初期化する。
3. `form.save` で `User`, `TeacherPermission`, `TeacherGrade` を作成する。
4. 成功時は `201 Created` を返却する。

### 業務ルール

- `grade_id` は同校に所属する学年である必要がある。
- `grade_scope` は `TeacherPermission.grade_scopes.values` に含まれる値である必要がある。
- `manage_other_teachers` は `true` または `false`。

### Database変更

- `users` に教員レコード作成
- `teacher_permissions` に権限レコード作成
- `teacher_grades` に担当学年レコード作成

### Response

- `message`: `教員の新規作成に成功しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|
|作成中のレコードが無効|422|`errors` を返却|

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

- 同校の教員のみ対象とする。
- `grade_id` は現在の教師の高校に所属している必要がある。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::TeachersController`|
|Form|`Teacher::CreateTeacherForm`|
|Query|`Teacher::TeachersQuery`|
|Serializer|`TeacherSerializer`|
|Model|`User`, `TeacherPermission`, `TeacherGrade`|
