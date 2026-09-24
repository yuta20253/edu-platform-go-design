# 管理者ユーザー管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者自身が管理者アカウントの一覧を確認し、新規管理者を作成・更新・削除できるようにすること。
- システム上の役割
  - 管理者アカウントの検索とページングを提供する。
  - 管理者アカウントの詳細表示、作成、更新、論理削除を提供する。
  - 管理者情報の更新時に、都道府県・市区町村から住所候補を検索する補助機能を提供する。
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
|住所候補検索|都道府県・市区町村から住所候補を検索する|

---

# 3. 業務フロー

1. 管理者が管理者一覧画面を開く。
2. システムは検索条件とページングを適用して管理者一覧を返却する。
3. 詳細を表示したい管理者を選択すると、その管理者情報を取得する。
4. 新規管理者を作成する場合は名前とメールアドレスを入力する。
5. 既存管理者を更新する場合はプロフィール情報を更新する。住所を入力する際は、都道府県を指定して住所候補を検索し、候補の中から該当する住所を選択できる。
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
|`Api::V1::Admin::AddressesController`|`index`|GET|`/api/v1/admin/addresses`|都道府県・市区町村から住所候補を検索|

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

- 一覧の対象は、有効（論理削除されていない）な `admin` ロールのユーザーのみである。
- `q` が指定された場合は、`name` または `email` のいずれかに対する部分一致で絞り込む。
- 並び順は、作成日時（`created_at`）の降順（新しい順）である。作成日時が同一の管理者同士の順序を定める追加の並び替え条件はない。
- `per_page` の既定値は 25 件である（未指定・数値でない値・0以下の値のときに適用される）。1 以上の値は最大 100 件に制限する（100 を超える指定は 100 件として扱う）。

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

- 個人情報・住所を含めて返却する。個人情報（`user_personal_info`）が未登録の管理者では `user_personal_info` は `null`、住所が未設定の場合は `address` は `null` となる。
- 詳細レスポンス（作成・更新のレスポンスと共通）には、`id`、`name`、`name_kana`、`email`、`created_at`、`updated_at`、`activity_log`、`user_personal_info`、`address` を含む。
- `activity_log` は管理者の操作履歴を表す項目として返却されるが、現状は履歴を記録・生成する仕組みが存在せず、常に空の配列（`[]`）が返る（将来の履歴表示のためのプレースホルダー）。管理者の作成・更新・削除などの操作履歴が保存されることはない。

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
- 作成時には、個人情報（`user_personal_infos`）のレコードは作成しない。個人情報が未登録の状態で作成され、作成レスポンスの `user_personal_info` は `null` となる。

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
- 個人情報（`phone_number` / `birthday` / `gender`）が未登録の管理者に対する更新で、これら3項目がすべて未入力の場合は、個人情報のレコードを新規作成しない（名前・メールアドレス等のみの更新）。いずれかが入力された場合に限り、個人情報のレコードを新規作成する。
- 個人情報が既に登録されている管理者に対する更新では、`phone_number` / `birthday` / `gender` の3項目を、更新リクエストの値で置き換える（未入力の項目は空になる）。
- `name` / `name_kana` / `email` / `address_id` は、未指定（未入力）の項目は更新されず、既存の値が維持される。

### Database変更

- `users` を更新
- `user_personal_infos` を更新（既存レコードがある場合）、または新規作成（未登録かつ個人情報が1項目以上入力された場合のみ）

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

## `Api::V1::Admin::AddressesController#index`

### 概要

管理者情報の登録・更新画面で住所を入力する際に、都道府県・市区町村・町域から住所候補を検索する。管理者ユーザーのフォーム（`Admin::AdminForm`）が住所を紐づける際に、この検索結果から住所を選択する用途で利用される補助的な参照 API である。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`prefecture_id`|必須|integer|都道府県 ID|
|`city`|任意|string|市区町村名（部分一致）|
|`town`|任意|string|町域名（部分一致）|

### 処理内容

1. `prefecture_id` が指定されていない場合はエラーを返却する。
2. `prefecture_id` で住所を絞り込む。
3. `city` が指定されていれば部分一致で絞り込む。
4. `town` が指定されていれば部分一致で絞り込む。
5. 該当する住所候補の一覧を返却する。

### 業務ルール

- `prefecture_id` は必須であり、未指定の場合は候補を検索せずエラーとする。
- `city` / `town` は部分一致で検索される。

### Database変更

- なし

### Response

- 住所候補一覧: 各住所の `id`, `postal_code`, `city`, `town`, `prefecture`（`id`, `name`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|`prefecture_id` 未指定|400|`都道府県は必須です。`|

---

# 6. データモデル

## `users`

- 役割: 管理者アカウントを含むユーザー情報を管理するテーブル
- 主なカラム: `id`, `name`, `name_kana`, `email`, `user_role_id`, `deleted_at`
- リレーション: `belongs_to :user_role`, `has_one :user_personal_info`, `belongs_to :address`

## `user_personal_infos`

- 役割: ユーザーの個人情報を管理するテーブル
- 主なカラム: `phone_number`, `birthday`, `gender`

## `addresses`

- 役割: 郵便番号に紐づく住所候補を管理するテーブル
- 主なカラム: `id`, `postal_code`, `city`, `town`, `street_address`, `prefecture_id`
- リレーション: `belongs_to :prefecture`
- 管理者ユーザーの `address_id` はこのテーブルの住所を参照する。

---

# 7. 権限制御

- 管理者本人は削除できない。
- 最後の有効管理者は削除できない。
- 住所候補検索は管理者であれば誰でも実行できる。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::AdminsController`, `Api::V1::Admin::AddressesController`|
|Form|`Admin::AdminForm`|
|Query|`AdminsQuery`, `AddressesQuery`|
|Serializer|`Admin::AdminListSerializer`, `Admin::AdminDetailSerializer`, `AddressSerializer`|
|Service|`Admin::CreateAdminService`|
|Model|`User`, `UserPersonalInfo`, `Address`|
