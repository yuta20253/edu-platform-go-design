# 教師面談機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教師と生徒の間で行われる面談（進路相談・学習相談など）の申請、日程確定、キャンセル、メッセージのやり取りを管理すること。
- システム上の役割
  - 教師が担当範囲の生徒に対して面談を申請できる。
  - 教師が、生徒から届いた面談申請を確認し、日程を確定（`confirmed`）したり完了扱いにしたりできる。
  - 教師が進行中の面談をキャンセルできる。
  - 教師と生徒が、1件の面談ごとにメッセージをやり取りできる。メッセージの投稿は、申請後の日程調整段階への移行を伴う。
  - 面談の申請・確定・キャンセル・メッセージ投稿のたびに、相手方へお知らせ（システム通知）を送信する。
- 利用者
  - `teacher` ロールのユーザー（教師）
  - 本仕様書は教師視点の操作を対象とする。生徒側の面談申請・キャンセル操作については「面談機能」仕様書を参照。ただし、面談の状態遷移や通知など教師・生徒間で共有される業務ルールはここに記載する。

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|自身が担当教師である面談の一覧を取得する（状態で絞り込み可能）|
|詳細取得|指定した面談の詳細を取得する|
|新規申請|教師から担当生徒に対して面談を申請する|
|更新（確定・完了）|面談日程を確定する、または面談を完了扱いにする|
|キャンセル|進行中の面談を取り消す|
|メッセージ一覧取得|面談に紐づくメッセージ一覧を取得する|
|メッセージ投稿|面談にメッセージを投稿する|

---

# 3. 業務フロー

## 面談の申請〜完了

1. 生徒または教師のいずれかが面談を申請する（教師が申請する場合は、担当範囲内の生徒に対してのみ申請できる）。
2. 申請された面談は「申請中（`requested`）」の状態になり、相手方にお知らせが送信される。
3. 教師または生徒がメッセージを投稿すると、面談は自動的に「日程調整中（`scheduling`）」の状態に移行する。
4. 教師が日程（`scheduled_at`）を指定して面談を確定すると、面談は「確定（`confirmed`）」の状態になり、生徒にお知らせが送信される。
5. 面談実施後、教師が面談を「完了（`completed`）」にする。
6. 「申請中」「日程調整中」「確定」のいずれかの状態にある面談は、教師・生徒どちらからでもキャンセルでき、キャンセルされると相手方にお知らせが送信される。

## メッセージのやり取り

1. 教師が対象の面談を指定してメッセージを投稿する。
2. 投稿者が面談の当事者（担当教師または対象生徒）であることを確認する。
3. 面談が進行中（申請中・日程調整中・確定のいずれか）であることを確認する。完了・キャンセル済みの面談にはメッセージを投稿できない。
4. メッセージを保存し、面談が「申請中」であった場合は「日程調整中」に移行する。この移行はメッセージの保存とは独立して行われ、他の操作（相手方の同時のメッセージ投稿・確定・キャンセル等）との競合により移行できなかった場合は、他の操作で既に状態が進んだものとみなして移行を行わず、メッセージは保存されたまま正常に完了する。
5. 相手方にメッセージ着信のお知らせを送信する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::InterviewRequestsController`|`index`|GET|`/api/v1/teacher/interview_requests`|担当面談の一覧を取得|
|`Api::V1::Teacher::InterviewRequestsController`|`show`|GET|`/api/v1/teacher/interview_requests/:id`|面談詳細を取得|
|`Api::V1::Teacher::InterviewRequestsController`|`create`|POST|`/api/v1/teacher/interview_requests`|面談を申請|
|`Api::V1::Teacher::InterviewRequestsController`|`update`|PATCH|`/api/v1/teacher/interview_requests/:id`|面談を確定・完了にする|
|`Api::V1::Teacher::InterviewRequestsController`|`destroy`|DELETE|`/api/v1/teacher/interview_requests/:id`|面談をキャンセルする|
|`Api::V1::Teacher::InterviewRequestMessagesController`|`index`|GET|`/api/v1/teacher/interview_requests/:interview_request_id/messages`|面談メッセージ一覧を取得|
|`Api::V1::Teacher::InterviewRequestMessagesController`|`create`|POST|`/api/v1/teacher/interview_requests/:interview_request_id/messages`|面談にメッセージを投稿|

---

# 5. API / 処理詳細

## `Api::V1::Teacher::InterviewRequestsController#index`

### 概要

ログイン中の教師が担当教師となっている面談の一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`status`|任意|string|状態での絞り込み（`requested`, `scheduling`, `confirmed`, `completed`, `cancelled`）|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたりの件数（未指定時は20件、最大100件）|

### 処理内容

1. `teacher_id` が自身と一致する面談を対象とする。
2. `status` が指定された場合はその状態のものに絞り込む。
3. 申請日時の新しい順に並べ、指定件数でページングする。
4. `InterviewRequestSerializer` で返却する。

### 業務ルール

- 自身が担当教師となっている面談のみ取得できる。

### Database変更

- なし

### Response

- `interview_requests`（各要素は `InterviewRequestSerializer` の内容: `id`, `status`, `initiator_role`（申請者区分）, `reason_category`, `reason_detail`, `scheduled_at`（確定した面談日時）, `completed_at`, `cancelled_at`, `cancel_reason`, `lock_version`, `created_at`, `student_id`, `student_name`（生徒の氏名）, `teacher_id`, `teacher_name`（教師の氏名））
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Teacher::InterviewRequestsController#show`

### 概要

指定した面談の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|面談の ID|

### 処理内容

1. 自身が担当教師である面談から対象を検索する。
2. `InterviewRequestSerializer` で返却する。

### 業務ルール

- 自身が担当教師でない面談は取得できない。

### Database変更

- なし

### Response

- 面談の詳細情報（一覧の各要素と同じ項目。生徒名・教師名を含む）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の面談が存在しない、または自身が担当教師でない|404|対象面談なし|

## `Api::V1::Teacher::InterviewRequestsController#create`

### 概要

教師が担当範囲内の生徒に対して面談を申請する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request[student_id]`|必須|integer|申請先の生徒の ID|
|`interview_request[reason_detail]`|必須|string|申請理由（2000文字以内）|

### 処理内容

1. 入力値を検証する。
2. 申請先の生徒が、教師の閲覧権限範囲内（担当学年のみ閲覧可能な権限の場合は自身の担当学年）の生徒であることを確認する。
3. 検証に成功すると、状態「申請中」・申請者区分「教師」として面談レコードを作成する。
4. 生徒にお知らせを送信する。

### 業務ルール

- 申請理由は必須で、2000文字以内。
- 申請先の生徒は、申請する教師の閲覧権限範囲内でなければならない。
- 教師が申請する場合、申請理由カテゴリ（`reason_category`）は指定できない（生徒からの申請時のみ使用する項目）。
- 同じ生徒・教師の組み合わせで、進行中（申請中・日程調整中・確定）の面談が既に存在する場合は申請できない。
- 申請先の生徒は生徒ロールのユーザー、申請元は教員ロールのユーザーでなければならない。

### Database変更

作成:

```
interview_requests
```

### Response

- `message`: `面談を申請しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗（理由未入力、担当外の生徒指定、重複申請等）|422|`errors` を返却|

## `Api::V1::Teacher::InterviewRequestsController#update`

### 概要

面談の日程を確定する、または完了状態にする。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request[status]`|必須|string|変更後の状態（`confirmed` または `completed` のみ指定可）|
|`interview_request[lock_version]`|必須|integer|楽観ロック用のバージョン番号|
|`interview_request[scheduled_at]`|`confirmed`指定時は必須|datetime|確定する面談日時|

### 処理内容

1. 指定された状態が `confirmed` または `completed` のいずれかであることを確認する。それ以外が指定された場合はエラーとする。
2. `lock_version` が指定されていることを確認する。
3. 自身が担当教師である面談を対象に、現在の状態から指定状態への変更が許可された遷移であるかを確認する。
4. `confirmed` を指定する場合は `scheduled_at` が指定されていることを確認する。
5. 面談の状態・（`confirmed`の場合は日程・`completed`の場合は完了日時）を更新する。更新時に`lock_version`が一致しない場合は更新に失敗する。
6. `confirmed` に更新した場合、生徒に日程確定のお知らせを送信する。

### 業務ルール

- 状態は「申請中→日程調整中・確定・キャンセル」「日程調整中→確定・キャンセル」「確定→完了・キャンセル」の順にのみ遷移でき、それ以外の遷移は許可されない。
- `confirmed` へ変更する際は面談日時（`scheduled_at`）の指定が必須。
- 更新時は必ず `lock_version` を指定する必要があり、リクエスト時点のバージョンとデータベース上の最新バージョンが一致しない場合は、他のユーザーによる更新と競合したものとして更新が拒否される。
- 自身が担当教師でない面談は更新できない。

### Database変更

更新:

```
interview_requests
```

### Response

- `message`: `ステータスを更新しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|指定できない状態を指定|422|`指定できないステータスです`|
|`lock_version` 未指定|422|`lock_versionは必須です`|
|許可されない状態遷移、`confirmed`指定時に`scheduled_at`未指定等|422|`指定できない操作です`|
|`lock_version` がデータベースの最新値と不一致（楽観ロック競合）|409|`他のユーザーによってデータが更新されています。再読み込みしてください`|
|対象の面談が存在しない、または自身が担当教師でない|404|対象面談なし|

## `Api::V1::Teacher::InterviewRequestsController#destroy`

### 概要

進行中の面談をキャンセルする。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`reason`|任意|string|キャンセル理由|
|`lock_version`|必須|integer|楽観ロック用のバージョン番号（表示時点の面談情報の版数）|

### 処理内容

1. `lock_version` が指定されていることを確認する。
2. 自身が担当教師である面談を対象とする。
3. 面談が進行中（申請中・日程調整中・確定のいずれか）であることを確認する。進行中でない場合はキャンセルできない。
4. 状態を「キャンセル」にし、キャンセル日時・キャンセル実行者・理由を記録する。指定された `lock_version` がデータベース上の最新のバージョンと一致しない場合は、更新は行われずエラーとなる。
5. 相手方（生徒）にキャンセルのお知らせを送信する。

### 業務ルール

- キャンセルできるのは進行中の面談のみで、完了済み・キャンセル済みの面談はキャンセルできない。
- キャンセル理由は任意入力。
- キャンセル時は必ず `lock_version` を指定する必要があり、リクエスト時点のバージョンとデータベース上の最新バージョンが一致しない場合は、他のユーザーによる更新と競合したものとして更新が拒否される。

### Database変更

更新:

```
interview_requests
```

### Response

- `message`: `面談をキャンセルしました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|`lock_version` 未指定|422|`lock_versionは必須です`|
|進行中でない面談をキャンセルしようとした|422|`進行中の面談のみキャンセルできます`|
|`lock_version` がデータベースの最新値と不一致（楽観ロック競合）|409|`他のユーザーによってデータが更新されています。再読み込みしてください`|
|対象の面談が存在しない、または自身が担当教師でない|404|対象面談なし|

## `Api::V1::Teacher::InterviewRequestMessagesController#index`

### 概要

指定した面談に投稿されたメッセージの一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request_id`|必須|integer|面談の ID|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたりの件数（未指定時は20件、最大100件）|

### 処理内容

1. 自身が担当教師である面談を対象とする。
2. 紐づくメッセージを投稿日時の古い順に並べ、指定件数でページングする。
3. `InterviewRequestMessageSerializer` で返却する。

### 業務ルール

- 自身が担当教師である面談のメッセージのみ取得できる。

### Database変更

- なし

### Response

- `interview_request_messages`（`id`, `body`, `sender_id`（投稿者ID）, `sender_name`（投稿者の氏名）, `created_at`）
- `meta`
  - `current_page`
  - `total_pages`
  - `total_count`
  - `per_page`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の面談が存在しない、または自身が担当教師でない|404|対象面談なし|

## `Api::V1::Teacher::InterviewRequestMessagesController#create`

### 概要

面談にメッセージを投稿する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request_id`|必須|integer|面談の ID|
|`interview_request_message[body]`|必須|string|メッセージ本文（2000文字以内）|

### 処理内容

1. 自身が担当教師である面談を対象とする。
2. 投稿者が面談の当事者（担当教師または対象生徒）であることを確認する。
3. 面談が進行中であることを確認する。
4. メッセージを作成する。
5. 面談が「申請中」であった場合は「日程調整中」に移行する。この移行は、メッセージの保存後に面談の最新の状態を確認して行う。他の操作との競合（同時更新）により移行できなかった場合は、他の操作で既に状態が進んだものとみなして移行を行わず、エラーにもならない（保存したメッセージは取り消されない）。
6. 相手方（生徒）にメッセージ着信のお知らせを送信する。

### 業務ルール

- 本文は必須で、2000文字以内。
- 完了・キャンセル済みの面談にはメッセージを投稿できない。
- メッセージ投稿は面談を「申請中」から「日程調整中」へ進める効果を持つ。メッセージの保存と「日程調整中」への移行は独立して扱われ、移行が他の操作との競合により行えなかった場合でも、メッセージの保存は取り消されず、競合エラー（409）にもならない。

### Database変更

作成:

```
interview_request_messages
```

更新（状態が「申請中」の場合）:

```
interview_requests
```

### Response

- 作成したメッセージ（`id`, `body`, `sender_id`, `sender_name`, `created_at`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|本文未入力、当事者でない、面談が進行中でない等|422|`errors` を返却|
|対象の面談が存在しない、または自身が担当教師でない|404|対象面談なし|

---

# 6. データモデル

## `interview_requests`

- 役割: 教師と生徒の面談1件分の申請・状態・日程を管理するテーブル
- 主なカラム: `id`, `student_id`, `teacher_id`, `initiator_id`, `initiator_role`, `status`, `reason_category`, `reason_detail`, `scheduled_at`, `completed_at`, `cancelled_at`, `cancelled_by_id`, `cancel_reason`, `lock_version`
- リレーション: `belongs_to :student`（生徒ユーザー）, `belongs_to :teacher`（教師ユーザー）, `belongs_to :initiator`（申請したユーザー）, `belongs_to :cancelled_by`（任意）, `has_many :interview_request_messages`

## `interview_request_messages`

- 役割: 面談ごとのやり取りメッセージを記録するテーブル
- 主なカラム: `id`, `interview_request_id`, `sender_id`, `body`
- リレーション: `belongs_to :interview_request`, `belongs_to :sender`（投稿したユーザー）

---

# 7. 状態管理

```
requested（申請中）
  ↓（メッセージ投稿）
scheduling（日程調整中）
  ↓（教師が日程を指定して確定）
confirmed（確定）
  ↓（教師が完了操作）
completed（完了）
```

上記のいずれの状態（`completed` を除く）からも `cancelled`（キャンセル）へ遷移できる。`completed` および `cancelled` は最終状態であり、それ以降の状態変更はできない。

状態変更条件:

- `requested` → `scheduling`：教師または生徒がメッセージを投稿したとき
- `requested`/`scheduling` → `confirmed`：教師が面談日時を指定して確定操作を行ったとき
- `confirmed` → `completed`：教師が完了操作を行ったとき
- `requested`/`scheduling`/`confirmed` → `cancelled`：教師または生徒がキャンセル操作を行ったとき

---

# 8. 権限制御

- 一覧・詳細・更新・キャンセル・メッセージの閲覧/投稿は、いずれも自身が担当教師（`teacher_id`）となっている面談のみが対象。他の教師が担当する面談は操作できない。
- 教師が新規に面談を申請できる相手の生徒は、閲覧権限範囲内（担当学年のみ閲覧可能な権限の場合は自身の担当学年）の生徒に限られる。
- 面談の確定（`confirmed`）・完了（`completed`）への変更は教師のみが行える操作として提供されている。

---

# 9. 非同期処理

- 面談申請時：相手方への「面談の申請があります」お知らせを非同期ジョブで送信する。
- 面談確定時：生徒への「面談日程が確定しました」お知らせを非同期ジョブで送信する。
- 面談キャンセル時：相手方への「面談がキャンセルされました」お知らせ（理由がある場合は理由を含む）を非同期ジョブで送信する。
- メッセージ投稿時：相手方への「面談に新しいメッセージが届いています」お知らせを非同期ジョブで送信する。
- いずれも、お知らせの実体作成はお知らせ機能と共通の仕組み（システムお知らせ作成処理）を利用する。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::InterviewRequestsController`, `Api::V1::Teacher::InterviewRequestMessagesController`|
|Form|`Teacher::CreateInterviewRequestForm`|
|Service|`Teacher::CreateInterviewRequestService`, `Teacher::ProcessInterviewRequestService`, `Teacher::CancelInterviewRequestService`, `Teacher::CreateInterviewRequestMessageService`, `Common::CancelInterviewRequestService`, `Common::PostInterviewRequestMessageService`, `Common::CreateInterviewRequestNotificationService`, `Common::CreateInterviewConfirmedNotificationService`, `Common::CreateInterviewCancelledNotificationService`, `Common::CreateInterviewRequestMessageNotificationService`|
|Job|`Common::CreateInterviewRequestNotificationJob`, `Common::CreateInterviewConfirmedNotificationJob`, `Common::CreateInterviewCancelledNotificationJob`, `Common::CreateInterviewRequestMessageNotificationJob`|
|Model|`InterviewRequest`, `InterviewRequestMessage`|
|Serializer|`InterviewRequestSerializer`, `InterviewRequestMessageSerializer`|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
