# クラス編成機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 学校の学年・クラス（組）の構成を管理すること。教師がクラスの新設・改名・削除を申請し、申請に承認権限を持つ教師（管理系権限を持つ教師）が承認・却下することで、クラス構成を変更できるようにする。
- システム上の役割
  - 同校の学年・クラス一覧の参照を提供する。
  - クラスの新設・改名・削除の申請受付を提供する。
  - 申請の承認・却下・取消（申請者自身による取り下げ）を提供する。
  - 承認された申請の内容に応じて、実際のクラスデータ（作成・更新・削除）に反映する。
  - 申請時・承認/却下結果通知時に、関係者へお知らせを送信する。
- 利用者
  - `teacher` ロールのユーザー（教師）
  - 承認・却下の操作は、他職員操作権限（`manage_other_teachers`）を持つ教師のみが行える。

---

# 2. 機能一覧

|操作|概要|
|-|-|
|学年一覧取得|同校の学年一覧を取得する|
|クラス一覧取得|同校の学年ごとのクラス一覧を取得する|
|クラス詳細取得|指定クラスの詳細（担当教員・在籍生徒を含む）を取得する|
|申請一覧取得|クラス編成申請の一覧を取得する（ステータスで絞り込み可能）|
|申請詳細取得|指定した申請の詳細を取得する|
|クラス編成申請|クラスの新設・改名・削除を申請する|
|申請の承認・却下|クラス編成申請を承認または却下する（要：他職員操作権限）|
|申請の取消|申請者自身が、承認待ちの申請を取り下げる|

---

# 3. 業務フロー

## 参照

1. 教師がクラス編成画面を表示する。
2. システムは同校の学年一覧を、それぞれに属するクラス一覧を含めて返却する。
3. 教師がクラスを選択すると、クラスの詳細を返却する。

## 申請・承認フロー

1. 教師が「クラスを新設する」「既存クラスを改名する」「既存クラスを削除する」のいずれかを選び、必要な情報（学年、クラス名、対象クラス）を入力して申請する。
2. システムは申請内容を検証し、申請（承認待ち状態）として記録する。
3. 同校で他職員操作権限を持つ教師全員に、クラス作成申請があった旨のお知らせが送信される。
4. 他職員操作権限を持つ教師が申請一覧を確認し、承認または却下する。
5. 承認された場合、申請内容（新設・改名・削除）が実際のクラスデータへ反映される。
6. 承認・却下いずれの場合も、申請した教師へ結果のお知らせが送信される。
7. 申請者は、承認処理が行われる前であれば、申請そのものを取り消すことができる。取り消しはクラスデータには影響しない。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::GradesController`|`index`|GET|`/api/v1/teacher/grades`|同校の学年一覧を取得|
|`Api::V1::Teacher::SchoolClassesController`|`index`|GET|`/api/v1/teacher/school_classes`|同校の学年別クラス一覧を取得|
|`Api::V1::Teacher::SchoolClassesController`|`show`|GET|`/api/v1/teacher/school_classes/:id`|クラス詳細を取得|
|`Api::V1::Teacher::SchoolClassRequestsController`|`index`|GET|`/api/v1/teacher/school_class_requests`|クラス編成申請の一覧を取得|
|`Api::V1::Teacher::SchoolClassRequestsController`|`show`|GET|`/api/v1/teacher/school_class_requests/:id`|クラス編成申請の詳細を取得|
|`Api::V1::Teacher::SchoolClassRequestsController`|`create`|POST|`/api/v1/teacher/school_class_requests`|クラス編成（新設・改名・削除）を申請|
|`Api::V1::Teacher::SchoolClassRequestsController`|`update`|PATCH|`/api/v1/teacher/school_class_requests/:id`|申請を承認・却下する|
|`Api::V1::Teacher::SchoolClassRequestsController`|`destroy`|DELETE|`/api/v1/teacher/school_class_requests/:id`|申請を取り消す|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::GradesController#index`

### 概要

同校の学年一覧を取得する。

### Request

なし

### 処理内容

1. 教師の所属高校に属する学年を取得する。
2. `GradeSerializer` で返却する。

### 業務ルール

- 同校の学年のみ対象とする。

### Database変更

- なし

### Response

- 学年一覧（`id`, `year`, 表示名）

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::SchoolClassesController#index`

### 概要

同校の学年一覧を、各学年に属するクラス一覧を含めて取得する。

### Request

なし

### 処理内容

1. 教師の所属高校に属する学年を、クラス情報を含めて取得する。
2. 学年ごとにクラス一覧を含めたシリアライズ結果を返却する。

### 業務ルール

- 同校の学年・クラスのみ対象とする。

### Database変更

- なし

### Response

- 学年一覧。各学年は `id`, `year`, 表示名, 所属クラス一覧（`id`, `name`）を含む

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::SchoolClassesController#show`

### 概要

指定クラスの詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|クラスの ID|

### 処理内容

1. 対象クラスが、教師の所属高校の学年に属することを確認して検索する。学年・在籍生徒・担当教員をあらかじめ読み込む。
2. `Teacher::TeacherSchoolClassDetailSerializer` で返却する。

### 業務ルール

- 同校のクラスのみ取得できる。
- 在籍生徒は氏名カナの昇順で並べる。
- 担当教員は `teacher_school_classes` の順で返し、各担当教員の役割（`role`）を含める。

### Database変更

- なし

### Response

- `id`, `name`
- `grade`: `GradeSerializer`
- `teachers`: 担当教員の配列（各要素は `id`, `name`, `role`）
- `students`: 在籍生徒の配列（各要素は `id`, `name`, `name_kana`。氏名カナ昇順）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象クラスが存在しない、または同校でない|404|対象クラスなし|

## `Api::V1::Teacher::SchoolClassRequestsController#index`

### 概要

クラス編成申請の一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`status`|任意|string|申請のステータスで絞り込む（指定がない、または空のときは絞り込まない）|
|`page`|任意|integer|ページ番号。1以上の整数以外を指定した場合は未指定として扱う|
|`per_page`|任意|integer|1ページあたりの件数。既定値 20、上限 100。1以上の整数以外を指定した場合は既定値を使う|

### 処理内容

1. `SchoolClassRequest` を、クラス・申請者・承認者・学年とともに取得する。
2. 学年の所属高校（`grades.high_school_id`）が操作者の所属高校であるものに絞り込む。
3. 操作者が他職員操作権限を持たない場合は、さらに申請者が操作者自身であるもの（`applicant_id`）に絞り込む。
4. `status` が指定されている場合は、そのステータスのものに絞り込む。
5. 申請日時の降順（`created_at` 降順、同一の場合は `id` 降順）で並べ、ページングする。
6. `Teacher::SchoolClassRequestSerializer` で返却する。`current_user` を渡し、各申請の操作可否（`can_process` / `can_cancel`）を算出する。

### 業務ルール

- 対象は、操作者の所属高校の学年に属する申請のみである。
- 他職員操作権限を持つ教師は同校のすべての申請を参照でき、持たない教師は自分が申請したものだけを参照できる。
- 一覧の取得自体に他職員操作権限は不要である（絞り込みの範囲が変わるだけ）。
- `status` に存在しない値を指定した場合は、該当0件として扱う（エラーにしない）。

### Database変更

- なし

### Response

- `school_class_requests`: `Teacher::SchoolClassRequestSerializer` 一覧
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

`Teacher::SchoolClassRequestSerializer` が返す項目:

|項目|説明|
|-|-|
|`id`|申請の ID|
|`action`|申請の種別（新設・改名・削除）|
|`status`|申請のステータス|
|`name`|申請で指定されたクラス名|
|`reason`|申請の理由（承認・却下・取消の際に入力された理由）|
|`lock_version`|楽観ロック用のバージョン|
|`approved_at`|承認日時|
|`cancelled_at`|取消日時|
|`created_at`|申請日時|
|`grade`|`GradeSerializer`|
|`school_class`|対象クラスの `{ id, name }`。新設の申請、および承認後に削除されたクラスでは `null`|
|`applicant`|申請者の `{ id, name }`|
|`approver`|承認者の `{ id, name }`。未処理の場合は `null`|
|`can_process`|操作者がこの申請を承認・却下できるか。承認待ちであり、かつ操作者が申請者本人でなく、かつ操作者が他職員操作権限を持つ場合に真|
|`can_cancel`|操作者がこの申請を取り消せるか。承認待ちであり、かつ操作者が申請者本人である場合に真|

`can_process` / `can_cancel` は、`update` / `destroy` が受け付ける条件と一致させている。

### Errorケース

- なし（同校の教師であることが前提）

## `Api::V1::Teacher::SchoolClassRequestsController#show`

### 概要

指定したクラス編成申請の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|申請の ID|

### 処理内容

1. `index` と同じ絞り込み（同校の学年に属する申請。他職員操作権限がない場合は自分の申請のみ）を適用した対象から、`id` で1件取得する。
2. `Teacher::SchoolClassRequestSerializer` で返却する（`current_user` を渡す）。

### 業務ルール

- 他職員操作権限を持たない教師は、自分が申請したもの以外を取得できない（404 となる）。

### Database変更

- なし

### Response

- `Teacher::SchoolClassRequestSerializer`（項目は `index` と同じ）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の申請が存在しない、同校でない、または他職員操作権限を持たない教師が他人の申請を指定した|404|`ActiveRecord::RecordNotFound` ハンドラのメッセージ|

## `Api::V1::Teacher::SchoolClassRequestsController#create`

### 概要

教師がクラスの新設・改名・削除を申請する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`school_class_request[action]`|必須|string|申請区分（`creation`：新設, `modification`：改名, `deletion`：削除）|
|`school_class_request[grade_id]`|必須|integer|対象学年の ID|
|`school_class_request[name]`|新設・改名時は必須|string|クラス名（255文字以内）|
|`school_class_request[school_class_id]`|改名・削除時は必須|integer|対象クラスの ID|

### 処理内容

1. 入力値を検証する。
2. `grade_id` が教師の所属高校の学年であることを確認する。
3. `school_class_id` が指定されている場合、それが `grade_id` の学年に属するクラスであることを確認する。
4. 削除申請の場合、対象クラスに在籍する生徒・所属する教員がいないことを確認する。
5. 検証に成功すると、申請（承認待ち状態）を作成する。
6. 同校で他職員操作権限を持つ教師全員に、クラス作成申請があった旨のお知らせを送信する。

### 業務ルール

- 申請区分は「新設」「改名」「削除」のいずれかでなければならない。
- 「新設」「改名」の場合はクラス名（255文字以内）が必須。
- 「改名」「削除」の場合は対象クラス（`school_class_id`）が必須で、指定した学年に属している必要がある。
- 「削除」の申請は、対象クラスに在籍する生徒または所属する教員が1人もいない場合のみ行える。
- 対象学年は、申請する教師の所属高校の学年でなければならない。
- 同一クラスに対して承認待ちの申請が既に存在する場合、重ねて申請することはできない。
- 通知の送信対象は、同校で「他職員操作権限」を持つ教師全員。

### Database変更

作成:

```
school_class_requests
```

### Response

- `message`: `学級作成申請を受け付けました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗（名称未入力、対象クラス未指定、学年不一致、在籍者ありの削除申請、重複申請等）|422|`errors` を返却|

## `Api::V1::Teacher::SchoolClassRequestsController#update`

### 概要

クラス編成申請を承認または却下する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`school_class_request[status]`|必須|string|`approved`（承認） または `rejected`（却下）|
|`school_class_request[lock_version]`|必須|integer|楽観ロック用のバージョン番号|
|`school_class_request[reason]`|任意|string|却下理由など（10,000文字以内）|

### 処理内容

1. `before_action` が `RequireManageOtherTeachers` concern の `require_manage_other_teachers!('承認権限がないユーザーです')` を実行し、操作する教師が「他職員操作権限」を持っているか確認する。持たない場合は `403 Forbidden` を返却する。権限情報（`teacher_permissions` のレコード）自体が存在しない教師も、安全参照（`&.`）により権限なしとして `403` となる。
2. `before_action` が、指定された状態が `approved` または `rejected` のいずれかであることを確認する。いずれでもない場合は `422` を返却する。
3. `before_action` が、`lock_version` が指定されていることを確認する。未指定の場合は `422` を返却する。
4. 対象申請が、教師の所属高校に属し、かつ承認待ち（`pending`）状態であることを確認する。承認待ちでない場合は `409 Conflict` を返却する。
5. 申請者自身が承認・却下しようとした場合はエラーとする（`403`）。
6. 申請の状態・承認者・（承認時は）承認日時・却下理由を更新する。このとき `lock_version` にリクエストで受け取った値を明示的に代入するため、`UPDATE` の `WHERE` 句にその値が使われる。その後に他の教員が更新していた場合は0件更新となり、楽観ロックの競合（`409`）となる。
7. 承認（`approved`）の場合、申請区分に応じてクラスデータを反映する（新設：クラスを作成、改名：クラスの学年・名称を更新、削除：クラスを削除）。
8. 申請者へ、承認または却下された旨のお知らせを送信する。お知らせの送信はトランザクションの外で行う。

### 業務ルール

- この操作は「他職員操作権限」を持つ教師のみ実行できる。権限の判定は `RequireManageOtherTeachers` concern に実装されており、教師教員管理機能・教師権限管理機能・教員招待通知機能と共通である。
- 承認待ち状態でない申請（既に承認・却下・取消済みの申請）は処理できない。画面を開いた後に他の教員が承認・却下した場合、または申請者が取り消した場合がこれにあたり、楽観ロックの競合と同じく再読み込みを促す `409 Conflict` を返す。
- `lock_version` は必須である。未指定の場合は、申請の状態を確認する前に `422` となる。
- 申請した本人が、自身の申請を承認・却下することはできない。
- 承認時は、対象クラスデータへの反映（作成・更新・削除）と申請レコードの更新を一つの処理としてまとめて行い、途中で失敗した場合は両方とも反映されない。
- 更新時は必ず `lock_version` を指定し、リクエスト時点のバージョンがデータベースの最新バージョンと一致しない場合は、他のユーザーによる更新と競合したものとして更新が拒否される。

### Database変更

更新:

```
school_class_requests
```

承認時、申請区分に応じて追加で以下が発生する:

```
school_classes（新設時は作成、改名時は更新、削除時は削除）
```

### Response

- `message`: 指定された状態に対応するメッセージ。`approved` なら `申請が承認されました`、`rejected` なら `申請が却下されました`（指定値から決まり、処理結果によって変わらない）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|他職員操作権限がない（権限情報のレコード自体が存在しない場合を含む）|403|`承認権限がないユーザーです`|
|申請者自身が処理しようとした|403|`自身の申請は承認・却下できません`|
|対象の申請が存在しない、または同校でない|404|`ActiveRecord::RecordNotFound` ハンドラのメッセージ|
|指定できない状態を指定|422|`指定できないステータスです`|
|`lock_version` が未指定|422|`lock_versionは必須です`|
|対象の申請が承認待ちでない（既に処理済み・取消済み）|409|`この申請はすでに処理されています。再読み込みしてください`|
|`lock_version` がデータベースの最新値と不一致（楽観ロック競合）|409|`他のユーザーによってデータが更新されています。再読み込みしてください`（`ApplicationController` の `ActiveRecord::StaleObjectError` ハンドラ）|

## `Api::V1::Teacher::SchoolClassRequestsController#destroy`

### 概要

申請者自身が、承認待ちの申請を取り消す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`reason`|任意|string|取消理由|

### 処理内容

1. 自身が申請者である申請のうち、承認待ち（`pending`）状態のものを対象とする。
2. 状態を「取消（`cancelled`）」にし、取消日時・理由を記録する。

### 業務ルール

- 取り消せるのは、自身が申請した、承認待ち状態の申請のみ。既に承認・却下・取消済みの申請は取り消せない。
- 申請の取り消しは、クラスデータには一切影響しない（承認による反映が行われていないため）。

### Database変更

更新:

```
school_class_requests
```

### Response

- `message`: `申請を取り消しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|承認待ちでない申請を取り消そうとした|422|`承認待ちの申請のみ取り消せます`|
|対象の申請が存在しない、または自身の申請でない|404|対象申請なし|

---

# 6. データモデル

## `grades`

- 役割: 学校の学年を管理するテーブル
- 主なカラム: `id`, `high_school_id`, `year`
- リレーション: `belongs_to :high_school`, `has_many :school_classes`, `has_many :school_class_requests`, `has_many :students`（生徒ユーザー）, `has_many :teachers`（担当教員、`teacher_grades`経由）

## `school_classes`

- 役割: 学年内のクラス（組）を管理するテーブル
- 主なカラム: `id`, `grade_id`, `name`
- リレーション: `belongs_to :grade`, `has_many :users`（在籍生徒）, `has_many :teachers`（所属教員、`teacher_school_classes`経由）, `has_many :school_class_requests`

## `teacher_school_classes`

- 役割: 教員とクラスの所属関係を管理するテーブル
- 主なカラム: `id`, `user_id`, `school_class_id`, `role`（`homeroom`：担任, `assistant`：副担任）
- リレーション: `belongs_to :user`（教員であること）, `belongs_to :school_class`
- 削除申請の可否確認（`school_class.teachers.exists?`）で参照する。担任・副担任の別、および教員の無効化済み（`deleted_at`あり）の別は問わず、行があれば所属する教員として扱う。在籍する生徒の確認（`school_class.users.exists?`）も、無効化済みのユーザーを含む。
- 現行のRailsには、このテーブルへ行を作成・更新する処理（Controller・Service・Form・seeds）は存在しない（モデル・マイグレーションのみ）。クラスの削除が承認されたときは、`SchoolClass`の`dependent: :destroy`により、当該クラスの行が連動して削除される。

## `school_class_requests`

- 役割: クラスの新設・改名・削除の申請と、その承認状況を記録するテーブル
- 主なカラム: `id`, `school_class_id`（新設時はなし）, `applicant_id`, `approver_id`, `grade_id`, `action`, `status`, `name`, `approved_at`, `cancelled_at`, `reason`, `lock_version`
- リレーション: `belongs_to :school_class`（任意）, `belongs_to :applicant`（申請者）, `belongs_to :approver`（承認者、任意）, `belongs_to :grade`

---

# 7. 状態管理

## クラス編成申請（`school_class_requests`）の状態

```
pending（承認待ち）
  ↓（承認権限を持つ教師が承認）        ↓（却下）              ↓（申請者が取消）
approved（承認済み）              rejected（却下）        cancelled（取消）
```

状態変更条件:

- `pending` → `approved`：他職員操作権限を持つ教師（申請者本人を除く）が承認操作を行ったとき。同時にクラスデータへの反映が行われる。
- `pending` → `rejected`：他職員操作権限を持つ教師（申請者本人を除く）が却下操作を行ったとき。
- `pending` → `cancelled`：申請者本人が取消操作を行ったとき。

## 申請区分（`action`）

- `creation`（新設）：承認時に新しいクラスを作成する。
- `modification`（改名）：承認時に対象クラスの学年・名称を更新する。
- `deletion`（削除）：承認時に対象クラスを削除する。削除申請自体は、在籍者・所属教員がいないクラスに対してのみ行える。

---

# 8. 権限制御

- 学年・クラスの参照は、同校の教師であれば誰でも行える。
- クラス編成の申請（新設・改名・削除）は、同校の教師であれば誰でも行える。
- 申請の一覧・詳細の参照は、同校の教師であれば誰でも行える。ただし参照できる範囲が権限によって変わり、「他職員操作権限」を持つ教師は同校のすべての申請を、持たない教師は自分が申請したものだけを参照できる。
- 申請の承認・却下は、同校かつ「他職員操作権限」を持つ教師のみ行える。加えて、申請者自身は自分の申請を承認・却下できない。
- 申請の取消は、申請者本人のみが行える。

---

# 9. 非同期処理

- クラス編成申請時：同校で「他職員操作権限」を持つ教師全員に「クラス作成申請」のお知らせを非同期ジョブで送信する。
- 申請の承認・却下時：申請者へ、承認または却下された旨（却下時は理由を含む）のお知らせを送信する。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::SchoolClassesController`, `Api::V1::Teacher::SchoolClassRequestsController`, `Api::V1::Teacher::GradesController`|
|Concern|`RequireManageOtherTeachers`|
|Form|`Teacher::CreateSchoolClassRequestForm`|
|Service|`Teacher::CreateSchoolClassRequestService`, `Teacher::ProcessSchoolClassRequestService`, `Teacher::CancelSchoolClassRequestService`, `Teacher::CreateSchoolClassRequestNotificationService`, `Teacher::CreateSchoolClassRequestResultNotificationService`|
|Job|`Teacher::CreateSchoolClassRequestNotificationJob`|
|Model|`Grade`, `SchoolClass`, `SchoolClassRequest`|
|Serializer|`GradeSerializer`, `Teacher::TeacherSchoolClassGradeSerializer`, `Teacher::TeacherSchoolClassDetailSerializer`, `Teacher::SchoolClassRequestSerializer`|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
