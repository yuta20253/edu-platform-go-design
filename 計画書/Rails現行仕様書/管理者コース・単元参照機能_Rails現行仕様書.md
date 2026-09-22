# 管理者コース・単元参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が学習コースおよび単元の内容を参照し、問題データの登録状況を確認できるようにすること。
- システム上の役割
  - コースの一覧・詳細を提供し、コースごとの単元数・問題数を集計して表示する。
  - 単元の詳細として、その単元に登録されている問題（選択肢・ヒント・解説を含む）と直近のインポート履歴を提供する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|コース一覧取得|コース一覧を検索・並び替えして取得し、単元数・問題数を表示する|
|コース詳細取得|指定コースの詳細と、コースに含まれる単元ごとの問題数を取得する|
|単元詳細取得|指定単元に登録されている問題（選択肢・ヒント・解説を含む）と直近のインポート履歴を取得する|

---

# 3. 業務フロー

1. 管理者がコース一覧画面を開く。
2. システムは検索条件・科目・並び順を適用してコース一覧を取得し、コースごとの単元数・問題数を集計して返却する。
3. 管理者が特定のコースを選択すると、コース詳細と、そのコースに属する単元ごとの問題数を取得する。
4. 管理者が特定の単元を選択すると、その単元に登録されている問題を選択肢・ヒント・解説とともに取得する。あわせて、その単元に対する直近のインポート履歴を取得する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::CoursesController`|`index`|GET|`/api/v1/admin/courses`|コース一覧を取得|
|`Api::V1::Admin::CoursesController`|`show`|GET|`/api/v1/admin/courses/:id`|コース詳細を取得|
|`Api::V1::Admin::UnitsController`|`show`|GET|`/api/v1/admin/courses/:course_id/units/:id`|単元詳細を取得|

※ ルーティング上は `courses` / `units` に対する作成・更新・削除の操作も定義されているが、現時点ではコントローラーに実装されておらず、参照系（一覧・詳細）のみが提供されている。

---

# 5. API / 処理詳細

## `Api::V1::Admin::CoursesController#index`

### 概要

コース一覧を検索・絞り込み・並び替えして取得し、各コースの単元数・問題数を合わせて返却する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`subject_id`|任意|integer|科目 ID による絞り込み|
|`q`|任意|string|コース名（`level_name`）・説明文（`description`）に対するキーワード検索|
|`sort`|任意|string|並び替え項目（`level_name`、`created_at`、`id` のいずれか。指定外は `created_at` を使用）|
|`order`|任意|string|並び順（`asc` または `desc`。指定外は `desc` を使用）|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたり件数|

### 処理内容

1. 論理削除されていないコースを対象に、`subject_id` による絞り込み、`q` によるキーワード検索、`sort`/`order` による並び替えを適用する。
2. 対象コースに属する有効な単元数を単元単位で集計する。
3. 対象コースに属する有効な単元に紐づく有効な問題数をコース単位で集計する。
4. ページングした結果を、単元数・問題数を含めて返却する。

### 業務ルール

- 論理削除済みのコースは一覧に含まれない。
- `q` は `level_name`（コース名に相当）または `description`（説明文）のいずれかに部分一致する場合にヒットする。
- `sort` に許可されていない値が指定された場合は登録日時（新しい順）で並び替える。
- `per_page` は最大 100 件に制限される。

### Database変更

- なし

### Response

- `courses`: 各コースの `id`, `subject`（`id`, `name`）, `level_number`, `level_name`, `units_count`, `questions_count`, `created_at`
- `meta`: `current_page`, `total_pages`, `total_count`, `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Admin::CoursesController#show`

### 概要

指定コースの詳細と、コースに含まれる単元ごとの問題数を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|コース ID|

### 処理内容

1. 指定されたコースを、所属科目・単元とともに取得する。
2. コースに含まれる単元ごとに、紐づく問題数を集計する。
3. コース情報と、単元ごとの `id`, `unit_name`, `questions_count` の一覧を返却する。

### 業務ルール

- 単元の並び順は単元 ID の昇順である。

### Database変更

- なし

### Response

- `id`, `subject`（`id`, `name`）, `level_number`, `level_name`, `description`
- `units`: 各単元の `id`, `unit_name`, `questions_count`

### Errorケース

- 対象コースが存在しない|404|対象コースなし

## `Api::V1::Admin::UnitsController#show`

### 概要

指定単元に登録されている問題を選択肢・ヒント・解説とともに取得し、あわせてその単元に対する直近のインポート履歴を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`course_id`|必須|integer|コース ID|
|`id`|必須|integer|単元 ID|

### 処理内容

1. 指定されたコースに属する単元を、コース・科目・問題・選択肢・ヒント・解説とともに取得する。
2. 単元に属する問題を ID 順に並べ、各問題について選択肢（選択肢番号順）、ヒント（ステップ番号順）、解説（登録順）を整理する。
3. その単元に対する直近のインポート履歴（新しい順に最大5件）をあわせて取得する。
4. 単元情報、所属コース情報、問題一覧、直近のインポート履歴を返却する。

### 業務ルール

- 単元は URL 上の `course_id` に属するものに限定される。異なるコースの単元 ID を指定した場合は対象として扱われない。
- 問題・選択肢・ヒント・解説はいずれも登録順（選択肢は選択肢番号順、ヒントはステップ番号順）に整理して返却される。

### Database変更

- なし

### Response

- `id`, `course_id`, `unit_name`
- `course`: `id`, `subject`（`id`, `name`）, `level_name`, `level_number`
- `questions`: 各問題の `id`, `question_text`, `correct_answer`, `choices`（`id`, `choice_number`, `choice_text`）, `hints`（`id`, `step_number`, `hint_text`）, `explanations`（`id`, `explanation_type`, `explanation_text`）
- `recent_import_histories`: 直近のインポート履歴（`id`, `file_name`, `status`, `success_count`, `error_count`, `total_count`, `created_at`）

### Errorケース

- 対象コース・単元が存在しない|404|対象単元なし

---

# 6. データモデル

## `courses`

- 役割: 学習コースを管理するテーブル
- 主なカラム: `id`, `subject_id`, `level_number`, `level_name`, `description`, `deleted_at`
- リレーション: `belongs_to :subject`, `has_many :units`

## `units`

- 役割: コースに含まれる学習単元を管理するテーブル
- 主なカラム: `id`, `course_id`, `unit_name`, `deleted_at`
- リレーション: `belongs_to :course`, `has_many :questions`, `has_many :import_histories`

## `questions`

- 役割: 単元に紐づく問題を管理するテーブル
- 主なカラム: `id`, `unit_id`, `question_text`, `correct_answer`, `deleted_at`
- リレーション: `belongs_to :unit`, `has_many :question_choices`, `has_many :question_hints`, `has_many :question_explanations`

## `question_choices`

- 役割: 問題の選択肢を管理するテーブル
- 主なカラム: `id`, `question_id`, `choice_number`, `choice_text`, `deleted_at`
- リレーション: `belongs_to :question`

## `question_hints`

- 役割: 問題のヒントを管理するテーブル
- 主なカラム: `id`, `question_id`, `step_number`, `hint_text`, `deleted_at`
- リレーション: `belongs_to :question`

## `question_explanations`

- 役割: 問題の解説を管理するテーブル
- 主なカラム: `id`, `question_id`, `explanation_type`, `explanation_text`, `deleted_at`
- リレーション: `belongs_to :question`

---

# 7. 状態管理

- なし（コース・単元・問題は論理削除の有無のみを持ち、業務上の状態遷移は行わない）

---

# 8. 権限制御

- 管理者はすべてのコース・単元・問題を参照可能。
- 論理削除済みのコース・単元は一覧・集計の対象から除外される。

---

# 9. 非同期処理

- なし

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::CoursesController`, `Api::V1::Admin::UnitsController`|
|Query|`Admin::CoursesQuery`|
|Serializer|`Admin::CourseListSerializer`, `Admin::CourseDetailSerializer`, `Admin::UnitDetailSerializer`, `Admin::QuestionDetailSerializer`, `ImportHistorySerializer`|
|Model|`Course`, `Unit`, `Question`, `QuestionChoice`, `QuestionHint`, `QuestionExplanation`, `ImportHistory`|

注意:

この情報は現在実装との対応確認用です。

新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
