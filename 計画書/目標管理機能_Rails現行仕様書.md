# 目標管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒が自分の学習目標を登録・管理すること。
- システム上の役割
  - 生徒の目標を一覧表示し、詳細と紐づくタスクを確認できる。
  - 目標の新規作成と更新を受け付ける。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|生徒の目標を期日昇順で取得する|
|詳細取得|指定目標の詳細とその目標に紐づくタスク一覧を取得する|
|新規登録|目標を新規作成する|
|更新|既存の目標を更新する|

---

# 3. 業務フロー

1. 生徒が目標一覧画面にアクセスする。
2. システムは生徒の目標を `due_date` 昇順でページング取得する。
3. 生徒が目標を選択すると、目標の詳細と関連タスクを取得する。
4. 生徒が目標を入力し、作成または更新を実行する。
5. 入力値を検証する。
6. 妥当であれば `goals` テーブルへ登録または更新する。
7. 結果を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::GoalsController`|`index`|GET|`/api/v1/student/goals`|生徒の目標一覧を取得|
|`Api::V1::Student::GoalsController`|`show`|GET|`/api/v1/student/goals/:id`|目標詳細と紐づくタスクを取得|
|`Api::V1::Student::GoalsController`|`create`|POST|`/api/v1/student/goals`|目標を作成|
|`Api::V1::Student::GoalsController`|`update`|PATCH|`/api/v1/student/goals/:id`|目標を更新|

---

# 5. API / 処理詳細

## `Api::V1::Student::GoalsController#index`

### 概要

ログイン中の生徒が登録した目標一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたりの件数|

### 処理内容

1. `current_user.goals` を対象に `GoalsQuery.due_soon` を適用する。
2. `paginate` でページングする。
3. `Student::GoalSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身の目標のみ取得する。
- 期日順で並べる。

### Database変更

- なし

### Response

- `id`
- `title`
- `description`
- `status`
- `due_date`（`YYYY/MM/DD` 形式）

### Errorケース

- 特定なし（`current_user` の認証前提）

## `Api::V1::Student::GoalsController#show`

### 概要

指定目標の詳細と、その目標に紐づくタスクを取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|目標の識別子|

### 処理内容

1. `current_user.goals` から `GoalsQuery.includes_tasks` を適用する。
2. 指定 ID の目標を検索する。
3. `Student::GoalDetailSerializer` で JSON を返却する。

### 業務ルール

- 生徒自身の目標のみ取得できる。
- 目標に紐づくタスクを含めて返却する。

### Database変更

- なし

### Response

- `id`
- `title`
- `description`
- `status`
- `due_date`（`YYYY/MM/DD` 形式）
- `tasks`
  - `id`
  - `user_id`
  - `goal_id`
  - `title`
  - `content`
  - `due_date`
  - `priority`
  - `status`
  - `completed_at`
  - `units`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の目標が存在しない|404|対象目標なし|
|他ユーザーの目標にアクセス|404|対象目標なし|

## `Api::V1::Student::GoalsController#create`

### 概要

生徒が新しい目標を登録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`goal[title]`|必須|string|目標名|
|`goal[description]`|任意|string|目標の詳細|
|`goal[due_date]`|必須|string|期限日|

### 処理内容

1. `Student::CreateGoalForm` を `current_user` と入力で初期化する。
2. バリデーションを実行する。
3. 成功した場合、`goals` テーブルに新規登録する。

### 業務ルール

- `title` は必須。
- `due_date` は必須。

### Database変更

- `goals` に新規レコード作成

### Response

- `id`（作成した目標の ID）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Student::GoalsController#update`

### 概要

生徒が既存の目標を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`goal[title]`|必須|string|目標名|
|`goal[description]`|任意|string|目標の詳細|
|`goal[due_date]`|必須|string|期限日|

### 処理内容

1. `current_user.goals` から指定の目標を検索する。
2. `Student::UpdateGoalForm` で更新処理を実行する。
3. バリデーションが通れば `goals` を更新する。

### 業務ルール

- `title` は必須。
- `due_date` は必須。
- 他ユーザーの目標は更新できない。

### Database変更

- 既存 `goals` レコード更新

### Response

- `id`（更新した目標の ID）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|
|対象の目標が存在しない|404|対象目標なし|

---

# 6. データモデル

## `goals`

- 役割: 生徒の学習目標を記録するテーブル
- 主なカラム:
  - `id`
  - `user_id`（生徒に紐付く）
  - `title`（必須）
  - `description`
  - `due_date`（必須）
  - `status`（`not_started`, `in_progress`, `completed`）
  - `deleted_at`
- リレーション:
  - `belongs_to :user`
  - `has_many :tasks`
  - `has_many :draft_tasks`

---

# 7. 状態管理

- `status` を enum で管理する。
- 定義:
  - `not_started`
  - `in_progress`
  - `completed`
- 現状のコードでは、`status` の変更ロジックはこの機能の API 内では直接定義されていない。

---

# 8. 権限制御

- すべての検索は `current_user.goals` を起点に行う。
- そのため、生徒は自分自身の目標のみ閲覧・更新できる。
- `Api::V1::Student::BaseController` の認証認可に依存する設計。

---

# 9. 非同期処理

- `Goal` の作成・更新に紐づく Job / Mailer は見当たらない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::GoalsController`|
|Form|`Student::CreateGoalForm`, `Student::UpdateGoalForm`|
|Query|`GoalsQuery`|
|Model|`Goal`|
|Serializer|`Student::GoalSerializer`, `Student::GoalDetailSerializer`|
