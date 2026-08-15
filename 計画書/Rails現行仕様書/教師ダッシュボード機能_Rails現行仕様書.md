# 教師ダッシュボード機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師のホーム画面に同校の生徒数と教師向けのお知らせを表示すること。
- システム上の役割
  - 同校の学年別生徒数を集計する。
  - 教師に表示可能な公開中のお知らせを取得する。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|表示|同校の学年別生徒数と教師向けお知らせを取得する|

---

# 3. 業務フロー

1. 教師がダッシュボードを表示する。
2. システムは同校の生徒数を学年別に集計する。
3. 公開中の教師向けお知らせを最新順で取得する。
4. ダッシュボード情報を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::DashboardsController`|`show`|GET|`/api/v1/teacher/dashboard`|教師向けダッシュボード情報を取得|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::DashboardsController#show`

### 概要

同校の生徒数を学年別に集計し、教師向けのお知らせを最新 5 件取得する。

### Request

- なし

### 処理内容

1. 同校の生徒を `students.high_school_current` で抽出し、`grades.year` でグループ化して件数を算出する。
2. `Announcement.for_user(current_user).includes(:publisher).published.order(published_at: :desc).limit(5)` で公開中のお知らせを取得する。
3. `stats` と `announcements` を JSON で返却する。

### 業務ルール

- 生徒数は同校の生徒のみを対象とする。
- お知らせは教師が閲覧可能な公開中のもののみ返却する。

### Database変更

- なし

### Response

- `stats`
  - `grade_one_students_count`
  - `grade_two_students_count`
  - `grade_three_students_count`
- `announcements`
  - `id`, `title`, `content`, `status`, `published_at`, `scheduled_at`, `created_at`, `targets`, `publisher`

### Errorケース

- なし（認証前提）

---

# 6. データモデル

## `announcements`

- 役割: お知らせを管理するテーブル
- 主なカラム: `id`, `title`, `content`, `status`, `published_at`, `scheduled_at`, `publisher_id`
- リレーション: `has_many :announcement_targets`

---

# 7. 権限制御

- `current_user` が `teacher` ロールであることが前提。
- 公開中かつ教師が閲覧可能な `Announcement.for_user` の絞り込みが適用される。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::DashboardsController`|
|Serializer|`AnnouncementSerializer`|
|Model|`Announcement`|
