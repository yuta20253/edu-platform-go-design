# 教師生徒参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師が同校の生徒一覧および生徒詳細を確認できるようにすること。また、生徒アカウントを個別に新規登録できるようにすること。
- システム上の役割
  - 同校生徒の絞り込みとページングを提供する。
  - 学年担当権限がある教師は担当学年のみ閲覧できる。
  - 教師が氏名・メールアドレス・学年・クラスを指定して生徒アカウントを1件ずつ作成できる。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|同校の生徒一覧を取得する|
|詳細取得|同校の指定生徒の詳細を取得する。`tab`で表示内容（プロフィール・目標・タスク）を切り替える|
|新規登録|生徒アカウントを1件新規作成する|

---

# 3. 業務フロー

## 一覧・詳細取得

1. 教師が生徒一覧画面を表示する。
2. システムは同校の生徒を取得し、担当学年が限定されている教師の場合は担当学年で絞り込む。
3. 指定された件数（未指定時は10件）でページングして返却する。
4. 教師が生徒を選択すると、対象生徒の詳細情報を返却する。

## 新規登録

1. 教師が氏名・氏名（カナ）・メールアドレス・学年・クラスを入力する。
2. システムは入力値と、学年・クラスが自校に属しているか、メールアドレスが他の生徒に使用されていないかを検証する。
3. 検証に成功すると、仮パスワードを発行して生徒アカウントを作成し、招待メールを送信する。
4. 作成結果を教師に返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::StudentsController`|`index`|GET|`/api/v1/teacher/students`|同校の生徒一覧を取得|
|`Api::V1::Teacher::StudentsController`|`show`|GET|`/api/v1/teacher/students/:id`|生徒詳細を取得|
|`Api::V1::Teacher::StudentsController`|`create`|POST|`/api/v1/teacher/students`|生徒アカウントを新規登録|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::StudentsController#index`

### 概要

同校の生徒一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号（未指定時は1ページ目扱い）|
|`per_page`|任意|integer|1ページあたりの件数（未指定時は10件、最大100件）|

### 処理内容

1. 同校（`current_user.high_school`）に所属するユーザーのうち生徒ロールのユーザーを対象に一覧を取得する。
2. 教師の権限区分が「担当学年のみ閲覧可能」の場合は、教師自身の学年に一致する生徒のみに絞り込む。
3. 氏名（カナ）順に並べ、指定件数（未指定時は10件、最大100件）でページングする。
4. `StudentSerializer` で返却する。

### 業務ルール

- 担当学年のみ閲覧可能な権限区分の教師は、自身の学年に属する生徒のみ参照できる（担当学年の絞り込みは、教師の権限区分から算出される「絞り込み対象の学年ID」に基づく。権限区分が「全学年閲覧可能」の場合は絞り込みは行われない）。
- 同校の生徒のみ対象とする。
- 1ページあたりの件数はリクエストで変更でき、未指定時は10件、上限は100件である。

### Database変更

- なし

### Response

- `students` : `StudentSerializer`
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::StudentsController#show`

### 概要

同校の指定生徒の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|生徒の ID|
|`tab`|任意|string|表示内容の切り替え。`goals` / `tasks` のいずれか。未指定および上記以外の値はプロフィールを返す|
|`page`|任意|integer|`tab=tasks`のときのページ番号（未指定時は1ページ目扱い）|
|`per_page`|任意|integer|`tab=tasks`のときの1ページあたりの件数（未指定時は10件、最大100件）|

### 処理内容

1. `before_action :set_student` で `students_scope.find(params[:id])` により対象生徒を検索する。`students_scope`は同校かつ生徒ロールに限定し、操作者が担当学年のみの権限区分であれば担当学年にも限定する（`TeacherStudentsScope`）。
2. `params[:tab]` で分岐する。
    - `goals`: `GoalsQuery.new(@student.goals).due_soon.includes_tasks.result` で目標を取得し、`Teacher::StudentGoalSerializer` で返却する。`due_soon`は`due_date`昇順・`id`昇順。ページネーションは行わない。
    - `tasks`: `TasksQuery.new(@student.tasks).with_active_goal.due_soon.includes_units.paginate(...).result` でタスクを取得し、`Common::TaskProgressService`で解答状況を集計して`Teacher::StudentTaskSerializer`で返却する。`with_active_goal`は`goals`との`joins`であり、`Goal`の`default_scope`により論理削除済みの目標に紐づくタスクは除外される。
    - それ以外（未指定含む）: `StudentSerializer` で返却する。

### 業務ルール

- 担当学年のみ閲覧可能な権限区分の教師は、担当学年の生徒のみ取得可能。
- 同校以外の生徒は対象外。
- `tab=tasks`の解答状況は、タスク単位・タスク×単元単位の両方を返す。タスク単位の率は、単元ごとの率の平均ではなく、件数を合計してから算出する。
- 正答率（`correct_rate`）は、解答数が0件のときは`null`を返す（0ではない）。進捗率（`progress_rate`）は分母が0のとき0を返す。

### Database変更

- なし

### Response

`tab`未指定（プロフィール）:

- 生徒の基本情報と関連情報（`StudentSerializer` の内容）

`tab=goals`:

```json
{
  "student": { "id": 1, "name": "..." },
  "goals": [
    {
      "id": 1, "title": "...", "description": "...", "status": "...", "due_date": "...",
      "progress": { "task_completed_count": 0, "task_total_count": 0, "completion_rate": 0 },
      "tasks": [ { "id": 1, "title": "...", "status": "...", "due_date": "..." } ]
    }
  ]
}
```

`tab=tasks`:

```json
{
  "student": { "id": 1, "name": "..." },
  "tasks": [
    {
      "id": 1, "title": "...", "status": "...", "priority": "...", "due_date": "...", "completed_at": "...",
      "goal": { "id": 1, "title": "..." },
      "progress": { "total_questions": 0, "answered_count": 0, "correct_count": 0, "progress_rate": 0, "correct_rate": null },
      "units": [ { "...UnitSerializerの項目": "...", "total_questions": 0, "answered_count": 0, "correct_count": 0, "progress_rate": 0, "correct_rate": null } ]
    }
  ],
  "meta": { "current_page": 1, "total_pages": 1, "total_count": 0, "per_page": 10 }
}
```

### Errorケース

- 対象生徒が存在しない|404|対象生徒なし（`students_scope`の範囲外の生徒を指定した場合も404）

## `Api::V1::Teacher::StudentsController#create`

### 概要

教師が同校の生徒アカウントを1件新規登録する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`name`|必須|string|氏名|
|`name_kana`|必須|string|氏名（カナ）|
|`email`|必須|string|メールアドレス|
|`grade_id`|必須|integer|所属学年のID|
|`school_class_id`|必須|integer|所属クラスのID|

### 処理内容

1. 入力値を検証する。
2. `grade_id` が教師の所属高校の学年であることを確認する。存在しない場合はエラーとする。
3. `school_class_id` が、指定された学年に属するクラスであることを確認する。存在しない場合はエラーとする。
4. 指定されたメールアドレスが同校の既存生徒に使用されていないことを確認する。
5. 検証に成功すると、生徒ロールのユーザーとして仮パスワードを発行してアカウントを作成し、生徒番号を採番する。
6. 作成した生徒に招待メール（パスワード設定用リンクを含む）を送信する。
7. 成功時は `201 Created` を返却する。

### 業務ルール

- 学年・クラスは、操作する教師の所属高校に属するものでなければならない。指定した学年・クラスが所属高校に存在しない場合や、クラスが指定した学年に属さない場合はエラーとなる。
- 登録するメールアドレスが、同校の既存の生徒によってすでに使用されている場合は登録できない。
- 作成された生徒アカウントは仮パスワードが設定された状態となり、次回ログイン時にパスワード再設定が必要な状態（`password_reset_required`）で作成される。
- 生徒番号は自動採番される。
- この作成経路は、生徒CSVインポート機能と共通のアカウント作成処理（招待メール送信を含む）を利用する。

### Database変更

作成:

```
users
```

### Response

- `message`: `生徒の新規作成に成功しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗（学年・クラス不整合、メール重複等）|422|`errors` を返却|
|作成中のレコードが無効|422|`errors` を返却|

---

# 6. データモデル

## `users`

- 役割: ユーザー情報を管理するテーブル
- 主なカラム: `id`, `name`, `name_kana`, `email`, `grade_id`, `school_class_id`, `high_school_id`, `password_reset_required`
- リレーション: `belongs_to :grade`, `belongs_to :high_school`, `belongs_to :school_class`

---

# 7. 権限制御

- 担当学年のみ閲覧可能な権限区分の教師は、一覧・詳細取得時に自身の担当学年（`own_grade_restriction`により算出される学年ID）で絞り込みが行われる。全学年閲覧可能な権限区分の教師は絞り込みを受けない。
- 同校の生徒のみ参照・登録可能。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::StudentsController`|
|Form|`Teacher::CreateStudentForm`|
|Service|`Student::CreateStudentService`, `Common::CreateUserService`|
|Query|`Teacher::StudentsQuery`|
|Serializer|`StudentSerializer`|
|Model|`User`, `Grade`, `SchoolClass`, `HighSchool`|

注意: 絞り込み対象の学年IDは `User#own_grade_restriction` メソッドで算出される（教師の権限区分が担当学年限定の場合のみ自身の `grade_id` を返し、それ以外は `nil` を返す）。
