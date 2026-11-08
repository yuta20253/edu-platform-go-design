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
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

**`teacher_permission` の項目は、同じレスポンスの中で2通りある。**

|出現位置|項目|
|-|-|
|`teachers[].teacher_permission`|`id`, `grade_scope`, `manage_other_teachers`, **`lock_version`**|
|`current_user.teacher_permission`|`id`, `grade_scope`, `manage_other_teachers`（`lock_version` を含まない）|

更新時に必要な版（`lock_version`）は、更新対象となる `teachers` の各要素にのみ付与される。操作者自身は更新できないため、`current_user` 側には不要という扱いである。

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

- `id`, `name`, `teacher_permission`（`id`, `grade_scope`, `manage_other_teachers`, **`lock_version`**）

一覧の各要素と同じ構成である。最上位は単体のオブジェクトであり、キーで包まない。

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
|`teacher_permission[grade_scope]`|必須|string|学年スコープ（`own_grade` / `all_grades`）|
|`teacher_permission[manage_other_teachers]`|必須|boolean|他教員管理権限|
|`teacher_permission[lock_version]`|必須|integer|取得時の版。更新の競合を検出するために用いる|

### 処理内容

1. `before_action :set_teacher` が `teachers_query.active.find(params[:id])` で対象教員を検索する。
2. `before_action` が `require_manage_other_teachers!('他教員を編集する権限がありません')` を実行し、操作者に他教員管理権限がなければ 403 を返す。
3. `before_action :require_lock_version!` が版の指定を確認する。指定がなければ 422 を返し、権限情報には触れない。
4. 対象教員が自身のアカウントであれば更新を拒否する。
5. 対象教員が校内で最後の有効教員であれば更新を拒否する。
6. `Teacher::UpdatePermissionForm` により `teacher_permission` を更新する。**更新時に、送信された版が現在の版と一致することを確認する。** 一致しない場合は更新せず 409 を返す。

処理順は `before_action` の登録順に従い、対象教員の特定（2.の権限判定より先）→ 権限判定 → 版の指定の確認 → 自己更新・最後の教員の判定となる。存在しない ID を指定した場合は、権限の有無によらず 404 が返る。

### 業務ルール

- 自分自身の権限は更新できない。
- 校内の最後の有効教員の権限は変更できない。判定は、同校の有効な教員のうち対象教員以外が存在するかどうかで行う（`teachers_query.active.where.not(id: target.id).exists?`）。
- `grade_scope` は `TeacherPermission.grade_scopes.keys`（`own_grade` / `all_grades`）のいずれか。
- `manage_other_teachers` は boolean である必要がある。
- **更新は楽観ロックで保護されている。** 送信された版が現在の版と一致しない場合は更新されない。版の指定そのものが欠けている場合も更新されない。
- **部分更新はできない。** `grade_scope` と `manage_other_teachers` の両方が必須であり、片方だけを送ると残りの検証に失敗する。
- 更新が成功すると版が1つ進む。同じ版で2回目の更新を行うと 409 となる。

### Database変更

- `teacher_permissions` の更新

### Response

- `message`: `権限更新に成功しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|権限なし|403|`他教員を編集する権限がありません`|
|対象教員が存在しない、または論理削除済み|404|`ActiveRecord::RecordNotFound` ハンドラのメッセージ|
|`lock_version` の指定がない|422|`lock_versionは必須です`|
|自身の更新|422|`自分自身は更新できません`|
|最後の教員の更新|422|`最後の教員は更新できません`|
|**版が一致しない（他の利用者が先に更新した）**|**409**|`他のユーザーによってデータが更新されています。再読み込みしてください`|
|フォームのバリデーション失敗|422|`errors` に `form.errors.full_messages` を返却|
|`teacher_permissions` のモデル検証失敗|422|`ApplicationController` の `ActiveRecord::RecordInvalid` ハンドラが `errors` にモデルの `full_messages` を返却|

---

# 6. データモデル

## `teacher_permissions`

- 役割: 教師の権限を管理するテーブル
- 主なカラム:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー|
|`user_id`|bigint|不可|教員（`users`）への外部キー。一意インデックスを持ち、教員1人につき1件|
|`grade_scope`|integer|可|閲覧権限スコープ。Rails の `enum` で `own_grade: 0` / `all_grades: 1` に対応する整数。カラムは NULL 可だがモデルで必須検証がある|
|`manage_other_teachers`|boolean|可|他教員管理権限。カラムは NULL 可だがモデルで `true` / `false` のいずれかであることを検証する|
|`lock_version`|integer|不可|更新の競合を検出するための版。既定値 `0`。更新が成功するたびに1つ進む|
|`created_at` / `updated_at`|datetime|不可|作成・更新日時|

- リレーション: `belongs_to :user`
- モデルの検証: `user_id` の一意性、`grade_scope` の必須、`manage_other_teachers` が `true` / `false` のいずれかであること、対象ユーザーが教員であること
- **`lock_version` という名前のカラムを持つため、楽観ロックが有効である。** 更新時に版が一致しない場合、更新は行われず競合として扱われる（「5. API / 処理詳細」`#update`）
- 権限レコードを持たない教員が存在しうる。その場合、権限なしとして扱う（「7. 権限制御」）

## `users`

- 役割: 全ロールのアカウントを格納するテーブル。本機能は教員のレコードを対象とする
- 主なカラム（本機能が参照する範囲）:

|カラム|型|NULL|説明|
|-|-|-|-|
|`id`|bigint|不可|主キー。`teacher_permissions.user_id` が参照する|
|`name`|string(100)|可|氏名。一覧・詳細の表示に用いる|
|`name_kana`|string(100)|可|氏名カナ。一覧の並び順に用いる|
|`user_role_id`|bigint|可|`user_roles` への外部キー。ロール名そのものを保持するカラムはない|
|`high_school_id`|bigint|可|所属高校。一覧・詳細・更新の対象をこのカラムで絞り込む|
|`deleted_at`|datetime|可|論理削除日時。「最後の有効教員」の判定に用いる|

- リレーション: `belongs_to :user_role` / `belongs_to :high_school` / `has_one :teacher_permission`
- 教員の抽出: `User.teachers` スコープが `user_roles` との結合でロールを判定する（`user_roles.name` は `admin`=0 / `student`=1 / `teacher`=2 / `guardian`=3 に対応する整数）

---

# 7. 状態管理

本機能が扱うのは権限情報の2つの属性と、更新の競合を検出するための版である。**いずれも業務上の順序を持つ状態遷移ではない。**

## 権限区分（`grade_scope`）

```
全学年閲覧可能（all_grades） ⇄ 担当学年のみ閲覧可能（own_grade）
```

状態変更条件:

- 他教員管理権限を持つ教師が更新操作を行ったとき
- **どちらからどちらへも変更できる。** 遷移の制約はない

## 他職員操作権限（`manage_other_teachers`）

```
権限あり（true） ⇄ 権限なし（false）
```

状態変更条件:

- 同上。遷移の制約はない

## 版（`lock_version`）による更新の競合検出

```
取得時の版 == 保存時の版 → 更新する（版が1つ進む）

取得時の版 != 保存時の版 → 更新しない（409 Conflict）
```

状態変更条件:

- 更新が成功するたびに版が1つ進む
- 更新リクエストには版が必須である。欠けている場合は `422` となり、権限情報には触れない
- 版が一致しない場合は、他の利用者が先に更新したものとして `409 Conflict` を返す

**更新できない相手に対するガードは状態ではなく対象の条件である。** 操作者自身と、同校で最後に残った在籍中の教員は、版が一致していても更新できない（「5. API / 処理詳細」`#update`）。

---
---

# 8. 権限制御

- 更新には他教員管理権限が必要である。判定は `RequireManageOtherTeachers` concern の `require_manage_other_teachers!` が行い、`current_user.teacher_permission&.manage_other_teachers?` が真でなければ 403 を返す。`teacher_permission` が未設定の教師は権限なしとして扱う。
- 一覧取得・詳細取得に他教員管理権限は不要である。
- 対象教員は同校に所属している必要がある。
- 教師ロール自体の判定は `Api::V1::Teacher::BaseController` が Pundit の `authorize :teacher_service, :access?` で行う。

---

# 9. 非同期処理

**本機能では非同期処理を行わない。**

権限の更新はメール送信・通知の送信を伴わない。更新された教員に変更が通知されることはなく、次回の参照時に新しい権限が反映されるだけである。

---
---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::PermissionsController`|
|Concern|`RequireManageOtherTeachers`|
|Query|`Teacher::TeachersQuery`（`colleagues_for_permissions`）|
|Form|`Teacher::UpdatePermissionForm`|
|Serializer|`Teacher::TeacherPermissionManagementSerializer`, `TeacherPermissionSerializer`, `TeacherSerializer`|
|Model|`TeacherPermission`|
