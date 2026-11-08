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
- **論理削除済み（`deleted_at` が設定されている）の教員も一覧に含まれる。** 有効性による絞り込みを行わない。教師生徒参照機能の生徒一覧が論理削除済みを除外するのとは扱いが異なる（教師生徒参照機能①「5. API / 処理詳細」`#index`）。
- 権限情報（`teacher_permissions` のレコード）が存在しない教員は、`grade_scope` と `manage_other_teachers` が `null` として返却される（エラーにはならない）。

### Database変更

- なし

### Response

- 最上位は `teachers` キーでラップした配列（`{ "teachers": [ ... ] }`）。ページ情報（`meta`）は返さない
- 配列の各要素は `Admin::TeacherSerializer` の項目である

`Admin::TeacherSerializer` が返す項目は次の6つである。

|項目|型|説明|
|-|-|-|
|`id`|integer|教員のID|
|`name`|string|氏名|
|`email`|string|メールアドレス|
|`grade_scope`|string|閲覧権限スコープ（`own_grade` / `all_grades`）。権限レコードがない教員は `null`|
|`manage_other_teachers`|boolean|他教員管理権限。権限レコードがない教員は `null`|
|`grades`|array|担当学年。各要素は `{ "id": 学年のID, "name": 学年の表示名 }`。表示名は学年の `year` に対応する `中３生` / `高１生` / `高２生` / `高３生` / `高卒生` / `新高１生` / `新高２生` / `新高３生`（該当しない場合は `不明`）。年次（`year`）自体はこのレスポンスには含めない|

所属校・作成日時・更新日時は返さない。

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
- 最上位は `teacher` キーでラップしたオブジェクト（`{ "teacher": { ... } }`）
- 中身は `Admin::TeacherSerializer` の項目であり、一覧の各要素と同じ構成である

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

- `name` / `email` は、送信された項目のうち `nil` でないものだけを更新する。いずれも送信されない場合、`users` の更新は行わない。
- `grade_scope` / `manage_other_teachers` も同様に、送信された項目のうち `nil` でないものだけを更新する。
- `grade_ids` が指定された場合、既存の担当学年を全て削除してから作り直す。
- `grade_scope` が `all_grades` の場合は、指定の `grade_ids` に関わらず対象教員の所属校の全学年を担当学年とする。それ以外の場合は、指定の `grade_ids` のうち所属校に属する学年のみを担当学年とし、**他校の学年 ID は無視する**（拒否はしない）。
- `grade_scope` の判定には、更新後の権限情報を用いる（担当学年の更新は権限情報の更新より後に行われる）。
- 1〜3の更新は1つのトランザクションで実行する。
- 対象の教員に権限情報（`teacher_permissions` のレコード）が存在しない場合について、特別な扱いは定められていない。`grade_scope` / `manage_other_teachers` を更新しようとした場合、または `grade_ids` を指定して担当学年を置き換えようとした場合は、権限情報を参照する処理が失敗し、想定外のエラー（`500 Internal Server Error`）となる。氏名・メールアドレスのみの更新では、権限情報を参照しないため影響を受けない。

### Database変更

- `users`, `teacher_permissions`, `teacher_grades` を更新

### Response

- 200 OK
- 最上位は `teacher` キーでラップしたオブジェクト（`{ "teacher": { ... } }`）
- 中身は `Admin::TeacherSerializer` の項目であり、一覧の各要素と同じ構成である

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|高校または対象教員が存在しない|404|`ActiveRecord::RecordNotFound` ハンドラのメッセージ|
|バリデーション失敗（`ActiveRecord::RecordInvalid`）|422|`errors` に対象レコードの `full_messages` を返却|
|`grade_scope` に許容外の値を指定（`ArgumentError`）|422|`errors` に例外のメッセージを返却|

---

# 6. データモデル

## `users`

- 役割: 全ロール（管理者・生徒・教員・保護者）のアカウントを格納するテーブル。教員は `user_roles.name` が `teacher` のレコードとして表される
- 主なカラム（本機能が扱う範囲）:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`email`|string|不可|メールアドレス。一意インデックスを持つ|
|`name`|string(100)|可|氏名|
|`name_kana`|string(100)|可|氏名カナ。本機能の招待では `name` と同じ値が設定される|
|`user_role_id`|bigint|可|`user_roles` への外部キー。ロール名そのものを保持するカラムはない|
|`high_school_id`|bigint|可|所属高校。本機能の一覧・取得はこのカラムで絞り込む|
|`grade_id`|bigint|可|学年。本機能の招待では設定しない|
|`password_reset_required`|boolean|不可|招待待ち状態（既定値 `false`）。本機能の招待では `false` のまま|
|`deleted_at`|datetime|可|論理削除日時|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- 本機能の対象外のカラム: `encrypted_password`, `jti`, `reset_password_token`, `reset_password_sent_at`, `remember_created_at`, `address_id`, `activated_at`, `school_class_id`, `student_number`
- リレーション: `belongs_to :user_role` / `belongs_to :high_school` / `belongs_to :grade` / `has_one :teacher_permission` / `has_many :teacher_grades`
- 教員の抽出: `User.teachers` スコープが `joins(:user_role).where(user_roles: { name: 'teacher' })` で判定する

## `user_roles`

- 役割: ロールのマスタテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`name`|integer|不可|ロール名。Rails の `enum` で `admin: 0` / `student: 1` / `guardian: 3` / `teacher: 2` に対応する整数。一意インデックスを持つ（既定値 `1`＝`student`）|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション: `has_many :users`

## `teacher_permissions`

- 役割: 教師の閲覧権限・他教員管理権限を管理するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|教員（`users`）への外部キー。一意インデックスを持ち、教員1人につき1件|
|`grade_scope`|integer|可|閲覧権限スコープ。Rails の `enum` で `own_grade: 0` / `all_grades: 1` に対応する整数。カラムは NULL 可だがモデルで必須検証がある|
|`manage_other_teachers`|boolean|可|他教員管理権限。カラムは NULL 可だがモデルで `true` / `false` のいずれかであることを検証する|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション: `belongs_to :user`
- モデルの検証: `user_id` の一意性、`grade_scope` の必須、`manage_other_teachers` が `true` / `false` のいずれかであること、対象ユーザーが教員であること

## `teacher_grades`

- 役割: 教師の担当学年を管理するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|教員（`users`）への外部キー|
|`grade_id`|bigint|不可|学年（`grades`）への外部キー|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- `(user_id, grade_id)` に一意インデックスを持ち、同じ教員に同じ学年を二重に登録できない
- リレーション: `belongs_to :user` / `belongs_to :grade`
- モデルの検証: `user_id` と `grade_id` の組の一意性、対象ユーザーが教員であること

---

# 7. 状態管理

## 教員アカウントの招待待ち状態

**本機能で作成した教員アカウントは、招待待ち状態（`password_reset_required`）にならない。** 偽のまま作成される。

```
作成（password_reset_required = false のまま）

↓

パスワード設定（招待メールのリンクから。状態は変わらない）
```

- 作成と同時に招待メールを送信するため、招待待ちの印を付ける必要がないという扱いである
- このため、本機能で作成した教員は**教員招待通知機能の「招待未完了の教員一覧」に現れない**
- 教師による教員の新規作成（教師教員管理機能）は招待待ち状態を真にする。同じ教員アカウントでも作成経路によって初期状態が異なる（教師教員管理機能①「7. 状態管理」）

## 権限区分と担当学年の整合

```
grade_scope = all_grades  → 担当学年は対象高校の全学年
grade_scope = own_grade   → 担当学年は指定された学年のうち対象高校に属するもの
```

状態変更条件:

- 作成時、および担当学年の指定を含む更新時に、上記の規則で担当学年を組み立て直す
- **権限区分が全学年閲覧可能のときは、指定された学年の一覧を使わない。** 対象高校の全学年に置き換える
- 更新では、権限情報を更新したあとの権限区分で判定する

## パスワード設定用トークン

作成時にパスワード設定用のトークンを発行し、招待メールのリンクに含める。**有効期間は6時間である。**

## 本機能に状態遷移はない

教員アカウント・権限・担当学年のいずれも、順序のある状態を持たない。更新はいつでも行える。

---
---

# 8. 権限制御

- 管理者は同校の教員のみ対象とする。

---

# 9. 非同期処理

## 招待メールの送信

|項目|内容|
|-|-|
|実行タイミング|教員の新規招待（`#create`）が成功したとき。**更新（`#update`）では送信しない**|
|処理内容|パスワード設定用のリンクを含む招待メール（件名「edu platform へのご招待」）を、指定されたメールアドレスへ送信する|
|影響範囲|招待された教員1名のみ|

- 送信はアカウント作成と同じ処理の中で登録されるが、**実際の送信はDBトランザクションのコミット後に行われる。** トランザクションがロールバックされた場合は送信されない
- 送信の失敗は作成の成否に影響しない。作成は成功し、メールだけが届かない状態になりうる
- この送信処理は生徒の新規登録（教師生徒参照機能・生徒CSVインポート機能）と共通である
- メールアドレスの変更を伴う更新でも、新しいメールアドレスへの通知は行われない

---
---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::TeachersController`|
|Service|`Admin::CreateTeacherService`（共通の作成処理 `Common::CreateUserService` を継承）, `Admin::UpdateTeacherService`|
|Serializer|`Admin::TeacherSerializer`|
|Mailer|`AuthMailer`（招待メール `invite_user`）|
|Model|`User`, `TeacherPermission`, `TeacherGrade`|
