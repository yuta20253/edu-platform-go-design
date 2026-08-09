# ダッシュボード機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒のホーム画面に表示する、期限が近い目標を取得すること。
- システム上の役割
  - 生徒の目標を絞り込み、最優先で表示するデータを提供する。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|表示|期限が近い目標を最大 5 件取得する|

---

# 3. 業務フロー

1. 生徒がダッシュボード画面を表示する。
2. システムは生徒の目標を `due_date` 昇順で取得する。
3. 取得した最大 5 件の目標を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::DashboardsController`|`show`|GET|`/api/v1/student/dashboard`|期限が近い目標を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::DashboardsController#show`

### 概要

生徒の目標のうち期限が近いものを 5 件まで取得する。

### Request

- なし

### 処理内容

1. `current_user.goals` に `GoalsQuery.due_soon.limit_five` を適用する。
2. `Student::GoalSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身の目標のみ対象にする。
- 期限が近い順で最大 5 件まで取得する。

### Database変更

- なし

### Response

- 目標リスト（`id`, `title`, `description`, `status`, `due_date`）

### Errorケース

- なし（認証前提）

---

# 6. データモデル

## `goals`

- 役割: 生徒の学習目標を記録するテーブル
- 主なカラム: `id`, `user_id`, `title`, `description`, `due_date`, `status`
- リレーション: `belongs_to :user`

---

# 7. 状態管理

- なし（目標状態の取得のみ）

---

# 8. 権限制御

- `current_user.goals` から取得するため、生徒自身の目標のみ対象となる。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::DashboardsController`|
|Query|`GoalsQuery`|
|Serializer|`Student::GoalSerializer`|
|Model|`Goal`|
