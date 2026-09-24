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
3. 新規教員を招待する場合は、氏名・メールアドレス・閲覧権限スコープ・他教員管理権限・担当学年を指定して作成する。作成と同時に、パスワード設定用の招待メールが対象のメールアドレスへ送信される。
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
- ページングは行わない。対象高校の教員を全件返却する（`page` / `per_page` は受け付けず、ページ情報（`meta`）も返却しない）。
- 並び順は指定されておらず、定められていない。
- 権限情報（`teacher_permissions` のレコード）が存在しない教員は、`grade_scope` と `manage_other_teachers` が `null` として返却される（エラーにはならない）。

### Database変更

- なし

### Response

- `teachers` : 各教員の `id`, `name`, `email`, `grade_scope`, `manage_other_teachers`, `grades`

### Errorケース

- 対象高校が存在しない|404|対象高校なし

## `Api::V1::Admin::TeachersController#create`

### 概要

新規教員を招待する。教員アカウント・権限・担当学年を作成し、作成時に招待メールを送信する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`high_school_id`|必須|integer|高校の ID|
|`name`|必須|string|氏名|
|`email`|必須|string|メールアドレス|
|`grade_scope`|必須|string|閲覧権限スコープ（`own_grade`=自学年のみ、`all_grades`=全学年）。未指定・許容外の値は 422|
|`manage_other_teachers`|必須|boolean|他教員管理権限。未指定は 422|
|`grade_ids`|任意|array|担当学年 ID 配列。`grade_scope` が `all_grades` の場合は指定値を使わない|

### 処理内容

1. `HighSchool.find(params[:high_school_id])` を取得する。
2. `Admin::CreateTeacherService.new(school: school, attributes: create_params).call` を実行する。
   1. 仮パスワード（利用者へは開示しない乱数）を発行し、教員ロールを解決する。
   2. 以降を1つのDBトランザクションで実行する。
      - `name` が未入力の場合は、エラーとして扱う（作成しない）。
      - 教員アカウントを、指定の高校に所属する教員として作成する。氏名カナには `name` と同じ値を設定する（画面に氏名カナの入力欄がないため）。学年（`users.grade_id`）は設定しない。招待待ち状態（`password_reset_required`）は設定せず、偽のままとする。
      - 権限（`grade_scope` / `manage_other_teachers`）を作成する。
      - 担当学年を作成する。`grade_scope` が `all_grades` の場合は、指定の `grade_ids` に関わらず対象高校の全学年を担当学年とする。それ以外の場合は、指定の `grade_ids` のうち対象高校に属する学年のみを担当学年とし、他校の学年 ID は無視する。
      - パスワード設定用のトークン（有効期間6時間）を発行し、招待メール（件名「edu platform へのご招待」。本文にパスワード設定用のリンクを含む）の送信を、トランザクションのコミット後に非同期で登録する。トランザクションがロールバックされた場合は、送信されない。
3. `Admin::TeacherSerializer` で作成した教員を返却する。

### 業務ルール

- `name` / `email` / `grade_scope` / `manage_other_teachers` は必須。
- 対象高校に紐づく教員として作成される。
- 教員アカウントは招待待ち状態（`password_reset_required`）にならない。このため、教員招待通知機能の「招待未完了の教員一覧」には現れない（招待メールは作成時に送信済みである）。
- メールアドレスが既存のアカウント（論理削除済みを含む）で使用済みの場合は、作成できない（422）。

### Database変更

- `users` に教員レコード作成
- `teacher_permissions` に権限レコード作成
- `teacher_grades` に担当学年レコード作成

### Response

- 201 Created
- `teacher` : `Admin::TeacherSerializer`（`id`, `name`, `email`, `grade_scope`, `manage_other_teachers`, `grades`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗（`name` 未入力、メールアドレスの不備・重複、`grade_scope` の許容外の値、`grade_scope` / `manage_other_teachers` の未指定）|422|`errors` を返却|
|対象高校が存在しない|404|対象高校なし|

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
|`grade_scope`|任意|string|閲覧権限スコープ（`own_grade` / `all_grades`）|
|`manage_other_teachers`|任意|boolean|他教員管理権限|
|`grade_ids`|任意|array|担当学年 ID 配列|

### 処理内容

1. `HighSchool.find(params[:high_school_id])` を取得する。
2. `school.users.teachers.find(params[:id])` で対象教員を取得する。
3. `Admin::UpdateTeacherService.new(user: user, params: update_params).call` を実行する。
4. `Admin::TeacherSerializer` で返却する。

### 業務ルール

- `grade_ids` が指定された場合、既存の担当学年を全て置き換える。
- 対象の教員に権限情報（`teacher_permissions` のレコード）が存在しない場合について、特別な扱いは定められていない。`grade_scope` / `manage_other_teachers` を更新しようとした場合、または `grade_ids` を指定して担当学年を置き換えようとした場合は、権限情報を参照する処理が失敗し、想定外のエラー（`500 Internal Server Error`）となる。氏名・メールアドレスのみの更新では、権限情報を参照しないため影響を受けない。

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
|Service|`Admin::CreateTeacherService`（共通の作成処理 `Common::CreateUserService` を継承）, `Admin::UpdateTeacherService`|
|Serializer|`Admin::TeacherSerializer`|
|Mailer|`AuthMailer`（招待メール `invite_user`）|
|Model|`User`, `TeacherPermission`, `TeacherGrade`|
