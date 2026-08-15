# 管理者ダッシュボード機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者がシステム全体の概要を把握できるように、ユーザー数と最近のインポート状況を表示すること。
- システム上の役割
  - ユーザー種別ごとの人数を集計する。
  - 直近のインポート履歴を取得する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|表示|ユーザー種別ごとの件数と最近のインポート履歴を取得する|

---

# 3. 業務フロー

1. 管理者がダッシュボード画面を表示する。
2. システムはユーザー種別ごとの件数を集計する。
3. 直近のインポート履歴を取得する。
4. ダッシュボード情報を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::DashboardsController`|`show`|GET|`/api/v1/admin/dashboard`|管理者ダッシュボード情報を取得|

---

# 5. API / 処理詳細

## `Api::V1::Admin::DashboardsController#show`

### 概要

システム全体のユーザー数と最近のインポート状況を管理者用ダッシュボードとして取得する。

### Request

- なし

### 処理内容

1. `User.joins(:user_role).group('user_roles.name').count` でユーザー数を種別別に集計する。
2. `ImportHistory.order(created_at: :desc).limit(5)` で最近のインポート履歴を取得する。
3. `stats` と `recent_imports` を JSON で返却する。

### 業務ルール

- 管理者はすべての高校・ユーザーを集計対象とする。

### Database変更

- なし

### Response

- `stats`
  - `student_count`
  - `teacher_count`
  - `admin_count`
  - `total_questions`
- `recent_imports`
  - `id`, `file_name`, `status`, `success_count`, `error_count`, `total_count`, `created_at`

### Errorケース

- なし（認証前提）

---

# 6. データモデル

## `import_histories`

- 役割: CSV インポートの実行履歴を管理するテーブル
- 主なカラム: `id`, `user_id`, `unit_id`, `file_name`, `status`, `total_count`, `success_count`, `error_count`, `created_at`
- リレーション: `belongs_to :user`, `belongs_to :unit`

---

# 7. 権限制御

- `current_user` が `admin` ロールであることが前提。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::DashboardsController`|
|Serializer|`ImportHistorySerializer`|
|Model|`User`, `UserRole`, `ImportHistory`|
