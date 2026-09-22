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
|クラス詳細取得|指定クラスの詳細を取得する|
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

1. 対象クラスが、教師の所属高校の学年に属することを確認して検索する。
2. `SchoolClassSerializer` で返却する。

### 業務ルール

- 同校のクラスのみ取得できる。

### Database変更

- なし

### Response

- `id`, `name`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象クラスが存在しない、または同校でない|404|対象クラスなし|

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

1. 操作する教師が「他職員操作権限」を持っているか確認する。持たない場合は `403 Forbidden` を返却する。
2. 指定された状態が `approved` または `rejected` のいずれかであることを確認する。
3. 対象申請が、教師の所属高校に属し、かつ承認待ち（`pending`）状態であることを確認する。
4. 申請者自身が承認・却下しようとした場合はエラーとする。
5. 申請の状態・承認者・（承認時は）承認日時・却下理由を更新する。
6. 承認（`approved`）の場合、申請区分に応じてクラスデータを反映する（新設：クラスを作成、改名：クラスの学年・名称を更新、削除：クラスを削除）。
7. 申請者へ、承認または却下された旨のお知らせを送信する。

### 業務ルール

- この操作は「他職員操作権限」を持つ教師のみ実行できる。
- 承認待ち状態でない申請（既に承認・却下・取消済みの申請）は処理できない。
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

- `message`: `申請が承認されました` または `申請が却下されました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|他職員操作権限がない|403|`承認権限がないユーザーです`|
|指定できない状態を指定|422|`指定できないステータスです`|
|申請者自身が処理しようとした|403|`自身の申請は承認・却下できません`|
|`lock_version` がデータベースの最新値と不一致（楽観ロック競合）|409|`他のユーザーによってデータが更新されています。再読み込みしてください`|
|対象の申請が存在しない、承認待ちでない、または同校でない|404 相当（申請未検出時）／`false`扱い（承認待ちでない場合）|-|

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
|Form|`Teacher::CreateSchoolClassRequestForm`|
|Service|`Teacher::CreateSchoolClassRequestService`, `Teacher::ProcessSchoolClassRequestService`, `Teacher::CancelSchoolClassRequestService`, `Teacher::CreateSchoolClassRequestNotificationService`, `Teacher::CreateSchoolClassRequestResultNotificationService`|
|Job|`Teacher::CreateSchoolClassRequestNotificationJob`|
|Model|`Grade`, `SchoolClass`, `SchoolClassRequest`|
|Serializer|`GradeSerializer`, `SchoolClassSerializer`, `Teacher::TeacherSchoolClassGradeSerializer`|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
