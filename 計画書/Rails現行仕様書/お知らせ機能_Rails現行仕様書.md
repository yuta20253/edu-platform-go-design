# お知らせ機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒に対して配信対象のお知らせ一覧と詳細を提供すること。
- システム上の役割
  - お知らせの対象範囲を判定し、公開済みのお知らせを表示する。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|配信対象の公開お知らせをページング取得する|
|詳細取得|指定お知らせの詳細を取得する|

---

# 3. 業務フロー

1. 生徒がお知らせ一覧を表示する。
2. システムは生徒に適用される公開済みのお知らせを取得する。
3. ページングして表示する。
4. 生徒が個別お知らせを選択すると、詳細を取得する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::AnnouncementsController`|`index`|GET|`/api/v1/student/announcements`|お知らせ一覧を取得|
|`Api::V1::Student::AnnouncementsController`|`show`|GET|`/api/v1/student/announcements/:id`|お知らせ詳細を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::AnnouncementsController#index`

### 概要

生徒が閲覧可能なお知らせを取得する。

### Request

- `page` (任意)

### 処理内容

1. `Announcement.for_user(current_user)` で対象ユーザー向けのお知らせを絞り込む。
2. `published` 状態のみを対象とする。
3. `published_at` と `id` の降順で並べる。
4. ページングして JSON を返却する。

### 業務ルール

- お知らせは `published` 状態のもののみ表示される。
- 対象範囲は `announcement_targets` に基づき、生徒の所属高校、役割、学年、ユーザー単位で判定される。

### Database変更

- なし

### Response

- `announcements`:
  - `id`
  - `title`
  - `content`
  - `published_at`
  - `publisher` (`id`, `name`, `name_kana`)
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Student::AnnouncementsController#show`

### 概要

指定お知らせの詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの ID|

### 処理内容

1. 対象ユーザー向けのお知らせを検索する。
2. `AnnouncementSerializer` で JSON を返却する。

### 業務ルール

- コンテンツは生徒対象のお知らせのみ取得する。

### Database変更

- なし

### Response

- `id`
- `title`
- `content`
- `published_at`
- `publisher`:
  - `id`
  - `name`
  - `name_kana`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象のお知らせが存在しない|404|対象データが見つかりません|

---

# 6. データモデル

## `announcements`

- 役割: 学校や運営から生徒へ配信される情報を管理するテーブル
- 主なカラム: `id`, `title`, `content`, `status`, `published_at`, `publisher_id`, `scheduled_at`
- リレーション:
  - `belongs_to :publisher, class_name: 'User'`
  - `has_many :announcement_targets`

---

# 7. 状態管理

- `status` を enum で管理する。
- 定義:
  - `draft`
  - `scheduled`
  - `published`
- ステータス遷移ルールがあり、`draft`→`scheduled`→`published` の順序で遷移する。

---

# 8. 権限制御

- `Announcement.for_user(current_user)` により、生徒の属性に合致するお知らせのみ取得される。

---

# 9. 非同期処理

- `PublishScheduledAnnouncementsJob` がスケジュールされたお知らせを公開状態に変更する。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::AnnouncementsController`|
|Model|`Announcement`|
|Serializer|`AnnouncementSerializer`, `AnnouncementPublisherSerializer`|
