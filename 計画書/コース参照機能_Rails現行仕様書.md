# コース参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 学習コースとその単元一覧を取得すること。
- システム上の役割
  - コース一覧を表示し、科目別検索をサポートする。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|コースと単元を取得する|
|科目絞り込み|指定科目のコースを取得する|

---

# 3. 業務フロー

1. 生徒がコース一覧を表示する。
2. システムはコースと関連単元を読み込む。
3. 必要に応じて `subject` で絞り込む。
4. JSON 形式で返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::CoursesController`|`index`|GET|`/api/v1/student/courses`|コース一覧を取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::CoursesController#index`

### 概要

コースと関連単元を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`subject`|任意|string|科目名による絞り込み|

### 処理内容

1. `CoursesQuery.includes_units` で単元を含める。
2. `subject` が指定されている場合は `join_subject.by_subject` で絞り込む。
3. `as_json(include: :units)` で返却する。

### 業務ルール

- `subject` が指定されない場合はすべてのコースを返す。
- `subject` が指定される場合はその科目に属するコースのみ返す。

### Database変更

- なし

### Response

- コース属性:
  - `id`
  - `subject_id`
  - `level_number`
  - `level_name`
  - `description`
- `units`:
  - `id`
  - `course_id`
  - `unit_name`

### Errorケース

- なし（認証前提）

---

# 6. データモデル

## `courses`

- 役割: 学習コースを管理するテーブル
- 主なカラム: `id`, `subject_id`, `level_number`, `level_name`, `description`
- リレーション:
  - `belongs_to :subject`
  - `has_many :units`

## `units`

- 役割: コースに含まれる学習単元を管理するテーブル
- 主なカラム: `id`, `course_id`, `unit_name`
- リレーション:
  - `belongs_to :course`

---

# 7. 状態管理

- なし

---

# 8. 権限制御

- なし（すべてのコースを取得可能）

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::CoursesController`|
|Query|`CoursesQuery`|
|Model|`Course`, `Unit`|
|Serializer|なし（`as_json` を使用）|
