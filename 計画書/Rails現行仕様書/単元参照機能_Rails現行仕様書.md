# 単元参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - タスクに紐づく単元を個別に取得すること。
- システム上の役割
  - タスクと単元の関連性を確認し、安全に単元データを返却する。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|詳細取得|タスクに紐づく単元情報を取得する|

---

# 3. 業務フロー

1. 生徒がタスク内の単元詳細を要求する。
2. システムは該当タスクと単元の紐づきを検証する。
3. 単元データを返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::UnitsController`|`show`|GET|`/api/v1/student/tasks/:task_id/units/:id`|タスクに紐づく単元詳細を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::UnitsController#show`

### 概要

タスクに紐づく単元の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`task_id`|必須|integer|タスク ID|
|`id`|必須|integer|単元 ID|

### 処理内容

1. 操作者自身のタスクから指定 `task_id` のタスクを取得する。見つからなければエラーとする。
2. そのタスクに紐づく単元から指定 `id` の単元を検索する。所属コースを同時に読み込む（N+1 を避けるため）。見つからなければエラーとする。
3. `UnitSerializer` で JSON を返却する。

### 業務ルール

- タスクに紐づかない単元は取得できない。
- タスクは操作者自身のものである必要がある。
- **論理削除済み（`deleted_at` が設定済み）の単元・コースも取得できる。** `Unit`・`Course` はいずれも `active` という絞り込みを持つが `default_scope` を持たず、この機能では `active` を使っていない。
- 単元の詳細はタスク配下でのみ取得できる。タスクに属さない単元を単体で取得するエンドポイントはこの機能に存在しない。

### Database変更

- なし

### Response

最上位をラップしないオブジェクトである。項目は `UnitSerializer` のとおり。

- `id`
- `course_id`
- `unit_name`
- `course`: 所属コースの情報
  - `id`
  - `level_number`
  - `level_name`

`course` はシリアライザの独自メソッドが組み立てる 3 項目であり、`courses` の全カラムではない。`description`・`subject_id`・`deleted_at` 等は含まれない。**コース参照機能が `as_json` でコースの全カラムを返すのとは異なる。**

`units` の `deleted_at`・`created_at`・`updated_at` も含まれない。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象のタスクが存在しない、または他の生徒のタスク|404|`{"message": "タスクが見つかりません"}`（`ApplicationController#not_found`）|
|対象の単元がそのタスクに紐づかない|404|`{"message": "Unitが見つかりません"}`。**モデル名の日本語訳が定義されていないため、名称が英語のまま出る**|
|未認証|401|認証エラー|
|`student` ロールでない|403|`{"errors": ["この操作を行う権限がありません"]}`|

このControllerは独自の例外ハンドラを持たないため、共通ハンドラの本文の形（`message` キー）になる。

---

# 6. データモデル

## `units`

- 役割: 学習単元を管理するテーブル
- 主なカラム: `id`（不可）, `course_id`（不可）, `unit_name`（string(255)・不可）, `deleted_at`（可）, `created_at` / `updated_at`（不可）
- リレーション: `belongs_to :course`, `has_many :tasks, through: :task_units`, `has_many :questions`
- `active` という絞り込みを持つが、**`default_scope` は持たない**
- 詳細なカラム定義は「コース参照機能」の「6. データモデル」を参照（同じテーブルを扱う）

## `courses`（参照のみ）

- 役割: 単元の所属コース
- この機能が参照する項目: `id`・`level_number`・`level_name`
- 詳細は「コース参照機能」の「6. データモデル」を参照

## `tasks` / `task_units`（参照のみ）

- 役割: 操作者のタスクと、タスクに紐づく単元の関係
- この機能はタスクを起点に単元を検索するために参照する
- 詳細は「タスク管理機能」の「6. データモデル」を参照

---

# 7. 状態管理

- なし

---

# 8. 権限制御

- `current_user.tasks` を起点としており、生徒のタスクに紐づく単元のみアクセス可能。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::UnitsController`|
|Controller（共通）|`ApplicationController`（`not_found` 等の共通ハンドラ）, `Api::V1::Student::BaseController`（生徒向け API の認可）|
|Serializer|`UnitSerializer`（`id`・`course_id`・`unit_name` と、`course` の 3 項目。コース参照機能・タスク管理機能・タスク下書き機能と共用）|
|Model|`Unit`（`active` を持つが `default_scope` は持たない）, `Course`（同）, `Task`（参照のみ）|
