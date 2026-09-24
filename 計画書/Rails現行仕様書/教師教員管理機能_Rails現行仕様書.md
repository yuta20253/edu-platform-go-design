# 教師教員管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師が同校の教員一覧を確認し、新規教員を招待できるようにすること。
- システム上の役割
  - 同校教員の一覧表示と詳細表示を提供する。
  - 新規教員アカウントの作成を受付ける（「他職員操作権限」を持つ教員のみ実行可能）。
  - 教員一覧には、各教員への招待通知の送信状況（教員招待通知機能と連携）を合わせて表示する。
- 利用者
  - `teacher` ロールのユーザー（教師）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|同校の教員一覧を取得する|
|詳細取得|指定教員の詳細を取得する|
|新規登録|教員アカウントを作成する|

---

# 3. 業務フロー

1. 教師が教員一覧画面を表示する。
2. システムは同校の教員を、各教員への直近の招待通知状況とあわせて取得して表示する。
3. 教師が教員を選択すると、詳細情報を取得する。
4. 新規教員を追加する場合、「他職員操作権限」を持つ教員が氏名・メールアドレス・担当学年・閲覧可能学年スコープ・他職員操作権限の付与有無を入力して作成する。権限がない教員は追加できない。
5. 作成された教員アカウントは、パスワード未設定の「招待待ち」の状態になる。この時点では招待メールは送信されない。招待メールは、教員招待通知機能で招待未完了の教員を選択して送信する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::TeachersController`|`index`|GET|`/api/v1/teacher/colleagues`|同校の教員一覧を取得|
|`Api::V1::Teacher::TeachersController`|`show`|GET|`/api/v1/teacher/colleagues/:id`|教員詳細を取得|
|`Api::V1::Teacher::TeachersController`|`create`|POST|`/api/v1/teacher/colleagues`|新規教員を作成|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::TeachersController#index`

### 概要

同校の教員一覧をページングして取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`page`|任意|integer|ページ番号（未指定・数値でない値・0以下の値のときは1ページ目として扱う）|
|`per_page`|任意|integer|1ページあたりの件数（未指定・数値でない値・0以下の値のときは既定値の20件。100を超える値は最大の100件として扱う）|

### 処理内容

1. 同校（`current_user.high_school`）に所属する教員ロールのユーザーを一覧取得する。
2. 氏名（カナ）順に並べ、指定件数（未指定時は20件、最大100件）でページングする。
3. 一覧に含まれる各教員について、送信済みの教員招待通知（教員招待通知機能で送信されるもの）を宛先ユーザーごとに送信日時の新しい順にまとめる。
4. `TeacherSerializer` で JSON を返却する。このとき各教員の招待状況（最新の招待通知の状態。通知が存在しない場合は「未送信」扱い）を合わせて返却する。

### 業務ルール

- 同校の教員のみ対象とする。
- 1ページあたりの件数は、既定値が20件、最大が100件である（`per_page` が1以上の場合はその値、ただし100件を超える場合は100件）。
- 並び順は氏名カナ順である。
- 教員一覧には、各教員に対して直近で送信された教員招待通知の状況が付与される。招待通知の送信・履歴管理自体は教員招待通知機能で行う。

### Database変更

- なし

### Response

- `current_user` : `TeacherSerializer`
- `teachers` : `TeacherSerializer` 一覧（各教員の招待状況 `invitation_status` を含む）
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::TeachersController#show`

### 概要

同校の指定教員の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|教員の ID|

### 処理内容

1. `teachers_query.find(params[:id])` で対象教員を検索する。
2. `TeacherSerializer` で返却する。

### 業務ルール

- 同校の教員のみ取得する。

### Database変更

- なし

### Response

- `id`, `name`, `email`, `grade_scope`, `manage_other_teachers`, `grades`

### Errorケース

- 対象教員が存在しない|404|対象教員なし

## `Api::V1::Teacher::TeachersController#create`

### 概要

新しい教員アカウントを作成する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`name`|必須|string|氏名|
|`name_kana`|必須|string|氏名（カタカナ）|
|`email`|必須|string|メールアドレス|
|`grade_id`|必須|integer|担当学年の ID|
|`grade_scope`|必須|integer|閲覧可能学年スコープ|
|`manage_other_teachers`|必須|boolean|他教員管理権限|

### 処理内容

1. 操作している教師が「他職員操作権限」を持っているか確認する。権限がない場合は `403 Forbidden` を返却し、以降の処理は行わない。
2. 入力値（氏名・氏名カナ・メールアドレス・担当学年・閲覧可能学年スコープ・他職員操作権限）を受け取り検証する。
3. 検証に成功すると、教員ロールのユーザーとして仮パスワードを発行してアカウントを作成し、権限情報（閲覧可能学年スコープ・他職員操作権限）と担当学年情報を登録する。
4. 成功時は `201 Created` を返却する。

### 業務ルール

- 教員の新規登録は「他職員操作権限」を持つ教員のみ実行できる。権限がない教員が実行した場合は「他職員操作権限がありません」というエラーメッセージとともに `403 Forbidden` が返却される。
- 操作している教師に権限情報（`teacher_permissions` のレコード）自体が存在しない場合について、特別な扱い（権限なしとして `403` を返す等）は定められていない。権限の有無を確認する処理が失敗し、想定外のエラーとして `500 Internal Server Error` になる。なお、権限情報は、管理者または教師が教員を作成したときには同時に作成されるが、教員が自己登録（新規登録）で作成された場合には作成されない。
- `grade_id` は同校に所属する学年である必要がある。
- `grade_scope` は定義済みの閲覧可能学年スコープの値である必要がある。
- `manage_other_teachers` は `true` または `false`。

### Database変更

- `users` に教員レコード作成
- `teacher_permissions` に権限レコード作成
- `teacher_grades` に担当学年レコード作成

### Response

- 201 Created
- `message`: `教員の新規作成に成功しました。`
- 作成された教員の情報は返却しない（`message` のみ）。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|他職員操作権限がない|403|`他職員操作権限がありません`|
|操作している教師の権限情報（`teacher_permissions`）が存在しない|500|特別な扱いはなく、想定外のエラーとなる|
|バリデーション失敗|422|`errors` を返却|
|作成中のレコードが無効|422|`errors` を返却|

---

# 6. データモデル

## `teacher_permissions`

- 役割: 教師の閲覧権限を管理するテーブル
- 主なカラム: `user_id`, `grade_scope`, `manage_other_teachers`

## `teacher_grades`

- 役割: 教師の担当学年を管理するテーブル
- 主なカラム: `user_id`, `grade_id`

---

# 7. 権限制御

- 同校の教員のみ対象とする。
- `grade_id` は現在の教師の高校に所属している必要がある。
- 教員の新規登録（`create`）は、操作する教師が「他職員操作権限」を持つ場合のみ実行できる。権限を持たない教員が実行しようとした場合は `403 Forbidden` となり、登録処理は行われない。権限情報のレコード自体が存在しない教師が実行した場合は、`403` ではなく `500 Internal Server Error` となる（特別な扱いは定められていない）。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::TeachersController`|
|Form|`Teacher::CreateTeacherForm`|
|Query|`Teacher::TeachersQuery`|
|Serializer|`TeacherSerializer`|
|Model|`User`, `TeacherPermission`, `TeacherGrade`, `TeacherNotification`（招待状況の参照のみ。詳細は教員招待通知機能を参照）|
