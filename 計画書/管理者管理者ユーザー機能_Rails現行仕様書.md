# 管理者ユーザー管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者自身が管理者アカウントの一覧を確認し、新規管理者を作成・更新・削除できるようにすること。
- システム上の役割
  - 管理者アカウントの検索とページングを提供する。
  - 管理者アカウントの詳細表示、作成、更新、論理削除を提供する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|管理者ユーザー一覧を取得する|
|詳細取得|指定管理者の詳細を取得する|
|新規登録|管理者ユーザーを作成する|
|更新|管理者ユーザー情報を更新する|
|削除|管理者ユーザーを論理削除する|

---

# 3. 業務フロー

1. 管理者が管理者一覧画面を開く。
2. システムは検索条件とページングを適用して管理者一覧を返却する。
3. 詳細を表示したい管理者を選択すると、その管理者情報を取得する。
4. 新規管理者を作成する場合は名前とメールアドレスを入力する。
5. 既存管理者を更新する場合はプロフィール情報を更新する。
6. 管理者を削除する場合は自分自身以外かつ最後の管理者でないことを確認して論理削除する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::AdminsController`|`index`|GET|`/api/v1/admin/admins`|管理者一覧を取得|
|`Api::V1::Admin::AdminsController`|`show`|GET|`/api/v1/admin/admins/:id`|管理者詳細を取得|
|`Api::V1::Admin::AdminsController`|`create`|POST|`/api/v1/admin/admins`|管理者を作成|
|`Api::V1::Admin::AdminsController`|`update`|PATCH|`/api/v1/admin/admins/:id`|管理者情報を更新|
|`Api::V1::Admin::AdminsController`|`destroy`|DELETE|`/api/v1/admin/admins/:id`|管理者を削除|

---

# 5. API / 処理詳細

## `Api::V1::Admin::AdminsController#index`

### 概要

管理者アカウントの一覧を検索およびページングして取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`q`|任意|string|検索キーワード|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたり件数|

### 処理内容

1. `AdminsQuery.new.search(params[:q]).order_default.result` を実行する。
2. `page(params[:page]).per(sanitized_per_page)` でページングする。
3. `AdminListSerializer` で返却する。

### 業務ルール

- `per_page` は 1 以上で最大 100 件に制限する。

### Database変更

- なし

### Response

- `admins` : `AdminListSerializer` 一覧
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Admin::AdminsController#show`

### 概要

指定管理者の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|管理者の ID|

### 処理内容

1. `admin_scope.find(params[:id])` で対象管理者を検索する。
2. `AdminDetailSerializer` で返却する。

### 業務ルール

- 個人情報・住所を含めて返却する。

### Database変更

- なし

### Response

- `admin` : `AdminDetailSerializer`

### Errorケース

- 対象管理者が存在しない|404|対象管理者なし

## `Api::V1::Admin::AdminsController#create`

### 概要

新しい管理者ユーザーを作成する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`name`|任意|string|名前|
|`email`|必須|string|メールアドレス|

### 処理内容

1. `create_params` を受け取る。
2. `Admin::AdminForm.new(create_params)` を初期化する。
3. `form.save` が成功すれば `AdminDetailSerializer` を返却する。

### 業務ルール

- `email` は必須。
- 入力がなければ `name` はメールアドレスのローカル部をデフォルトにする。

### Database変更

- `users` に管理者レコード作成

### Response

- `admin` : `AdminDetailSerializer`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Admin::AdminsController#update`

### 概要

管理者ユーザーのプロフィール情報を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`name`|任意|string|名前|
|`name_kana`|任意|string|名前（カタカナ）|
|`email`|任意|string|メールアドレス|
|`address_id`|任意|integer|住所 ID|
|`phone_number`|任意|string|電話番号|
|`birthday`|任意|date|生年月日|
|`gender`|任意|string|性別|

### 処理内容

1. `update_params` を受け取る。
2. `Admin::AdminForm.new(update_params.merge(user: admin))` を初期化する。
3. `form.save` が成功すれば `AdminDetailSerializer` を返却する。

### 業務ルール

- `phone_number` は 10〜11 桁の数字である必要がある。
- `gender` は `UserPersonalInfo.genders.keys` のいずれかである必要がある。
- `birthday` は未来日付にできない。

### Database変更

- `users` および `user_personal_infos` を更新

### Response

- `admin` : `AdminDetailSerializer`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Admin::AdminsController#destroy`

### 概要

管理者アカウントを論理削除する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|管理者の ID|

### 処理内容

1. `User.admins.active.find(params[:id])` で対象管理者を取得する。
2. 自分自身であれば削除を拒否する。
3. 最後の有効管理者であれば削除を拒否する。
4. `deleted_at` を更新して論理削除する。

### 業務ルール

- 自分自身は削除できない。
- 最後の有効管理者は削除できない。

### Database変更

- `users.deleted_at` を更新する論理削除

### Response

- `204 No Content`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|最後の管理者の削除|422|`最後の管理者は削除できません`|
|自身の削除|422|`自分自身は削除できません`|

---

# 6. データモデル

## `users`

- 役割: 管理者アカウントを含むユーザー情報を管理するテーブル
- 主なカラム: `id`, `name`, `name_kana`, `email`, `user_role_id`, `deleted_at`
- リレーション: `belongs_to :user_role`, `has_one :user_personal_info`, `belongs_to :address`

## `user_personal_infos`

- 役割: ユーザーの個人情報を管理するテーブル
- 主なカラム: `phone_number`, `birthday`, `gender`

---

# 7. 権限制御

- 管理者本人は削除できない。
- 最後の有効管理者は削除できない。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::AdminsController`|
|Form|`Admin::AdminForm`|
|Serializer|`Admin::AdminListSerializer`, `Admin::AdminDetailSerializer`|
|Service|`Admin::CreateAdminService`|
|Model|`User`, `UserPersonalInfo`, `Address`|
