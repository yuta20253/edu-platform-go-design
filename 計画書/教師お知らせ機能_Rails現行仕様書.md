# 教師お知らせ機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師が対象ユーザー向けにお知らせを作成・公開・閲覧できるようにすること。
- システム上の役割
  - お知らせ一覧と詳細表示を提供する。
  - 教師が下書きとしてお知らせを作成できるようにする。
  - 公開状態や公開日時を更新できるようにする。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|閲覧可能なお知らせ一覧または自身が作成した下書きを取得する|
|詳細取得|指定お知らせの詳細を取得する|
|新規登録|お知らせを下書きで作成する|
|更新|お知らせの公開状態や公開日時を更新する|

---

# 3. 業務フロー

1. 教師が閲覧画面を開く。
2. `authored` タブ時は自身が作成したお知らせ一覧を取得する。
3. それ以外では教師が閲覧可能な公開中のお知らせを取得する。
4. 教師が新規作成時に対象を指定し、下書きを登録する。
5. 作成済みお知らせのステータスや公開日時を更新する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::AnnouncementsController`|`index`|GET|`/api/v1/teacher/announcements`|お知らせ一覧を取得|
|`Api::V1::Teacher::AnnouncementsController`|`show`|GET|`/api/v1/teacher/announcements/:id`|お知らせ詳細を取得|
|`Api::V1::Teacher::AnnouncementsController`|`create`|POST|`/api/v1/teacher/announcements`|お知らせを作成|
|`Api::V1::Teacher::AnnouncementsController`|`update`|PATCH|`/api/v1/teacher/announcements/:id`|お知らせを更新|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::AnnouncementsController#index`

### 概要

教師が閲覧可能なお知らせ一覧、または自身が作成したお知らせ一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`tab`|任意|string|`authored` の場合は自身作成一覧|
|`page`|任意|integer|ページ番号|

### 処理内容

1. `tab` が `authored` の場合は `current_user.announcements` を参照する。
2. それ以外の場合は `Announcement.for_user(current_user).published` を参照する。
3. `order(published_at: :desc, id: :desc).page(params[:page]).per(20)` でページングする。
4. 適切なシリアライザを適用する。

### 業務ルール

- `authored` タブは自身の作成したお知らせのみ取得する。
- それ以外は教師が閲覧可能な公開中のお知らせのみ取得する。

### Database変更

- なし

### Response

- `announcements` : `AnnouncementSerializer` または `AuthoredAnnouncementSerializer`
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::AnnouncementsController#show`

### 概要

公開中のお知らせの詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの ID|

### 処理内容

1. `Announcement.for_user(current_user).published.find(params[:id])` を実行する。
2. `AnnouncementSerializer` で返却する。

### 業務ルール

- 教師が閲覧可能な公開中のお知らせのみ取得する。

### Database変更

- なし

### Response

- `id`, `title`, `content`, `status`, `published_at`, `scheduled_at`, `targets`, `publisher`

### Errorケース

- 対象お知らせが存在しない|404|対象お知らせなし

## `Api::V1::Teacher::AnnouncementsController#create`

### 概要

お知らせを下書きで作成する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`announcement[title]`|必須|string|タイトル|
|`announcement[content]`|必須|string|内容|
|`announcement[announcement_targets]`|必須|array|対象指定の配列|

### 処理内容

1. `create_announcement_params` を受け取る。
2. `Teacher::CreateAnnouncementForm.new(current_user: current_user, **params)` を初期化する。
3. バリデーションを実行する。
4. `Teacher::CreateAnnouncementService` を呼び出し、`Announcement` と `AnnouncementTarget` を作成する。

### 業務ルール

- `announcement_targets` は配列で指定する。
- `target_type` は `all_users`, `by_role`, `by_grade`, `by_school`, `by_user` のいずれか。
- `by_role` / `by_grade` / `by_user` では `user_role_id` または `user_id` を適切に指定する。
- `own_grade` 権限の教師は `by_grade` で自身の学年のみ指定可能。
- 指定先の学年・ユーザーは同校である必要がある。

### Database変更

- `announcements` に新規レコード作成
- `announcement_targets` に対象レコード作成

### Response

- `message`: `お知らせを下書きで作成しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Teacher::AnnouncementsController#update`

### 概要

自身が作成したお知らせの公開状態と公開日時を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`announcement[status]`|任意|string|公開ステータス|
|`announcement[scheduled_at]`|任意|string|公開日時|

### 処理内容

1. `current_user.announcements.find(params[:id])` で対象お知らせを取得する。
2. `update_announcement_params` で `status` / `scheduled_at` を更新する。

### 業務ルール

- 自身が作成したお知らせのみ更新できる。

### Database変更

- `announcements` の更新

### Response

- `message`: `お知らせのステータスを更新しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|更新失敗|422|`errors` を返却|

---

# 6. データモデル

## `announcements`

- 役割: お知らせを管理するテーブル
- 主なカラム: `id`, `title`, `content`, `status`, `published_at`, `scheduled_at`, `publisher_id`
- リレーション: `has_many :announcement_targets`

## `announcement_targets`

- 役割: お知らせ対象を管理するテーブル
- 主なカラム: `id`, `announcement_id`, `target_type`, `user_role_id`, `grade_id`, `high_school_id`, `user_id`
- リレーション: `belongs_to :announcement`

---

# 7. 権限制御

- `Announcement.for_user(current_user)` による閲覧制御が適用される。
- `create` 時のターゲット指定は同校の学年・ユーザーのみ許可される。
- `update` は自身が作成したお知らせのみ可能。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::AnnouncementsController`|
|Form|`Teacher::CreateAnnouncementForm`|
|Service|`Teacher::CreateAnnouncementService`|
|Serializer|`AnnouncementSerializer`, `AuthoredAnnouncementSerializer`|
|Model|`Announcement`, `AnnouncementTarget`|
