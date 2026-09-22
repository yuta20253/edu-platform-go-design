# 面談機能（生徒向け） Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒が担当教員に対して面談を申請し、教員とのやり取りを通じて面談の実施を進められるようにすること。
- システム上の役割
  - 生徒は自分が所属するクラスの担当教員を指定して面談を申請できる。
  - 申請した面談の一覧・詳細を確認し、状況に応じてメッセージのやり取りや申請の取り消しができる。
  - 面談の確定・完了操作は教員側の操作であり、本仕様書では扱わない（「教師面談機能」を参照）。ただし、生徒・教員の双方に共通する状態遷移ルールは本書でも正確に記載する。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|自分に関わる面談申請の一覧を取得する|
|詳細取得|指定した面談申請の詳細を取得する|
|新規申請|担当教員に対して面談を申請する|
|取り消し|進行中の面談申請を取り消す|
|メッセージ一覧取得|面談申請に紐づくメッセージ一覧を取得する|
|メッセージ投稿|面談申請に対してメッセージを投稿する|

---

# 3. 業務フロー

## 面談の申請

1. 生徒が、自分の所属クラスを担当する教員を選び、相談理由の種別・詳細を入力して面談を申請する。
2. システムは、指定した教員が生徒の所属クラスの担当教員であることを確認する。
3. 同じ生徒・教員の組み合わせで、既に進行中（申請中・調整中・確定済み）の面談が存在しないことを確認する。
4. 上記を満たせば、面談申請を「申請中」状態で登録する。
5. 教員へ面談申請があった旨の通知が行われる。

## 面談の取り消し

1. 生徒が、自分が申請した面談の取り消しを行う。取り消し時には理由（任意）と、表示していた面談情報の版数（`lock_version`）を指定する。
2. システムは、対象の面談が申請中・調整中・確定済みのいずれか（進行中）であることを確認する。既に完了・取り消し済みの面談は取り消せない。
3. 版数が最新のものと一致していることを確認する。一致しない場合は、他の操作（教員による確定操作など）で状態が既に変わっていることを意味し、取り消しは行われない。
4. 面談を「取り消し」状態に変更し、取り消し日時・取り消し実行者・理由を記録する。
5. 相手方（教員）へ、面談が取り消された旨の通知が行われる。

## メッセージのやり取り

1. 生徒が、自分が当事者となっている面談に対してメッセージを投稿する。
2. システムは、投稿者が生徒本人であり、対象の面談が終了（完了・取り消し）していないことを確認する。
3. メッセージを保存する。
4. 面談が「申請中」状態であった場合、メッセージのやり取りが始まったことを受けて「調整中」状態に進む。
5. 相手方（教員）へ、新着メッセージがある旨の通知が行われる。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::InterviewRequestsController`|`index`|GET|`/api/v1/student/interview_requests`|自分に関わる面談申請の一覧を取得|
|`Api::V1::Student::InterviewRequestsController`|`show`|GET|`/api/v1/student/interview_requests/:id`|面談申請の詳細を取得|
|`Api::V1::Student::InterviewRequestsController`|`create`|POST|`/api/v1/student/interview_requests`|面談を新規申請|
|`Api::V1::Student::InterviewRequestsController`|`destroy`|DELETE|`/api/v1/student/interview_requests/:id`|面談申請を取り消す|
|`Api::V1::Student::InterviewRequestMessagesController`|`index`|GET|`/api/v1/student/interview_requests/:interview_request_id/messages`|面談に紐づくメッセージ一覧を取得|
|`Api::V1::Student::InterviewRequestMessagesController`|`create`|POST|`/api/v1/student/interview_requests/:interview_request_id/messages`|面談にメッセージを投稿|

---

# 5. API / 処理詳細

## `Api::V1::Student::InterviewRequestsController#index`

### 概要

ログイン中の生徒が当事者（申請した生徒本人）となっている面談申請の一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`status`|任意|string|状態で絞り込む（`requested` / `scheduling` / `confirmed` / `completed` / `cancelled`）|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたりの件数|

### 処理内容

1. ログイン中の生徒が当事者となっている面談申請を、作成日時の新しい順に取得する。
2. `status` が指定されていれば、その状態のものだけに絞り込む。
3. ページングした結果を返却する。

### 業務ルール

- 自分が申請した面談のみが対象となる（教員側から申請された面談は「教師面談機能」側の一覧に現れる）。

### Database変更

- なし

### Response

- `interview_requests`（一覧。各要素は下記「6. データモデル」の主なカラムに加え、生徒名・教員名を含む）
- `meta`（`current_page`, `total_pages`, `total_count`, `per_page`）

### Errorケース

- 特定なし（`current_user` の認証前提）

## `Api::V1::Student::InterviewRequestsController#show`

### 概要

指定した面談申請1件の詳細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|面談申請の識別子|

### 処理内容

1. ログイン中の生徒が当事者となっている面談申請の中から、指定IDのものを取得する。

### 業務ルール

- 自分が当事者ではない面談は取得できない。

### Database変更

- なし

### Response

- 面談申請の詳細（「6. データモデル」参照）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の面談が存在しない、または自分が当事者ではない|404|対象データなし|

## `Api::V1::Student::InterviewRequestsController#create`

### 概要

生徒が担当教員に対して新しく面談を申請する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request[teacher_id]`|必須|integer|申請先教員のID|
|`interview_request[reason_category]`|必須|string|相談理由の種別（`study_method` 学習方法 / `study_plan` 学習計画 / `academic_performance` 成績 / `career` 進路 / `school_life` 学校生活 / `mental` メンタル / `other` その他）|
|`interview_request[reason_detail]`|必須|string|相談理由の詳細（2000文字以内）|

### 処理内容

1. 入力値を検証する。
2. 指定した教員が、生徒の所属クラスの担当教員であることを検証する。
3. 検証を満たせば、面談申請を「申請中」状態、申請者区分「生徒」として登録する。
4. 教員へ、面談の申請があった旨の通知を行う。

### 業務ルール

- 申請先教員は、申請する生徒の所属クラスを担当している教員に限られる。
- 相談理由の種別は生徒による申請では必須（教員による申請では逆に指定不可）。
- 同じ生徒・教員の組み合わせで、進行中（申請中・調整中・確定済み）の面談が既に存在する場合は申請できない。

### Database変更

作成:

```
interview_requests
```

### Response

- `message`（申請完了メッセージ）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|必須項目の不足、申請先教員が担当教員でない、進行中の面談が既に存在するなど|422|`errors` を返却|

## `Api::V1::Student::InterviewRequestsController#destroy`

### 概要

生徒が、自分が申請した進行中の面談を取り消す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|面談申請の識別子|
|`lock_version`|必須|integer|表示時点の面談情報の版数|
|`reason`|任意|string|取り消し理由|

### 処理内容

1. `lock_version` が指定されていない場合はエラーとする。
2. ログイン中の生徒が当事者となっている面談申請の中から、指定IDのものを取得する。
3. 面談が進行中（申請中・調整中・確定済み）であることを確認する。進行中でなければ取り消しは行われない。
4. 状態を「取り消し」に変更し、取り消し日時・取り消し実行者・理由を記録する。このとき、指定された `lock_version` が最新の値と一致しない場合は更新が行われずエラーとなる。
5. 相手方の教員へ、面談が取り消された旨の通知を行う。

### 業務ルール

- 進行中（申請中・調整中・確定済み）の面談のみ取り消せる。既に完了・取り消し済みの面談は取り消せない。
- 取り消し操作には、表示時点の版数（`lock_version`）の指定が必須であり、他の操作によって状態が先に変わっていた場合は競合として扱われる。

### Database変更

更新:

```
interview_requests
```

### Response

- `message`（取り消し完了メッセージ）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|`lock_version` 未指定|422|エラーメッセージを返却|
|対象の面談が存在しない、または自分が当事者ではない|404|対象データなし|
|既に完了・取り消し済みで進行中でない|422|進行中の面談のみキャンセル可能な旨のエラー|
|`lock_version` が最新のものと一致しない（競合）|409|データが他の操作で更新されている旨のエラー|

## `Api::V1::Student::InterviewRequestMessagesController#index`

### 概要

指定した面談申請に紐づくメッセージ一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request_id`|必須|integer|面談申請の識別子|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたりの件数|

### 処理内容

1. ログイン中の生徒が当事者となっている面談申請を取得する。
2. その面談に紐づくメッセージを投稿日時の古い順にページングして取得する。

### 業務ルール

- 自分が当事者ではない面談のメッセージは取得できない。

### Database変更

- なし

### Response

- `interview_request_messages`（一覧。本文・送信者ID・送信者名・投稿日時を含む）
- `meta`（`current_page`, `total_pages`, `total_count`, `per_page`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の面談が存在しない、または自分が当事者ではない|404|対象データなし|

## `Api::V1::Student::InterviewRequestMessagesController#create`

### 概要

生徒が、自分が当事者となっている面談に対してメッセージを投稿する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`interview_request_id`|必須|integer|面談申請の識別子|
|`interview_request_message[body]`|必須|string|メッセージ本文（2000文字以内）|

### 処理内容

1. ログイン中の生徒が当事者となっている面談申請を取得する。
2. メッセージを作成する。
3. 面談が「申請中」状態であれば「調整中」状態に進める。
4. 相手方の教員へ、新着メッセージがある旨の通知を行う。

### 業務ルール

- メッセージを投稿できるのは、その面談の当事者（申請した生徒本人、または相手教員）のみ。
- 面談が完了・取り消し済みの場合はメッセージを投稿できない。
- 本文は必須かつ2000文字以内。
- メッセージの投稿をきっかけに、面談が「申請中」から「調整中」へ進む。

### Database変更

作成:

```
interview_request_messages
```

更新:

```
interview_requests（状態が「申請中」の場合のみ「調整中」へ変更）
```

### Response

- `id`, `body`, `sender_id`, `sender_name`, `created_at`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|対象の面談が存在しない、または自分が当事者ではない|404|対象データなし|
|本文が空、文字数超過、面談が終了済みなど|422|`errors` を返却|

---

# 6. データモデル

## `interview_requests`

- 役割: 生徒と教員の面談申請・実施状況を記録するテーブル。
- 主なカラム:
  - `id`
  - `student_id`（申請対象の生徒）
  - `teacher_id`（申請対象の教員）
  - `initiator_id` / `initiator_role`（申請した側。生徒・教員のいずれか）
  - `status`（面談の状態）
  - `reason_category`（相談理由の種別。生徒からの申請時のみ設定される）
  - `reason_detail`（相談理由の詳細）
  - `scheduled_at`（確定した面談日時。教員による確定操作時に設定）
  - `completed_at`（完了日時）
  - `cancelled_at` / `cancelled_by_id` / `cancel_reason`（取り消し関連情報）
  - `lock_version`（楽観ロック用の版数）
- リレーション:
  - `belongs_to :student`（`User`）
  - `belongs_to :teacher`（`User`）
  - `belongs_to :initiator`（`User`）
  - `belongs_to :cancelled_by`（`User`、任意）
  - `has_many :interview_request_messages`

## `interview_request_messages`

- 役割: 面談申請に対する生徒・教員間のメッセージのやり取りを記録するテーブル。
- 主なカラム:
  - `id`
  - `interview_request_id`
  - `sender_id`
  - `body`
  - `created_at`
- リレーション:
  - `belongs_to :interview_request`
  - `belongs_to :sender`（`User`）

---

# 7. 状態管理

```
requested（申請中）
  ├→ scheduling（調整中）
  ├→ confirmed（確定済み）
  └→ cancelled（取り消し）

scheduling（調整中）
  ├→ confirmed（確定済み）
  └→ cancelled（取り消し）

confirmed（確定済み）
  ├→ completed（完了）
  └→ cancelled（取り消し）

completed（完了） … 以降の状態変化なし
cancelled（取り消し） … 以降の状態変化なし
```

状態変更条件:

- `requested` → `scheduling`：メッセージのやり取りが開始されたとき。
- `requested` / `scheduling` → `confirmed`：教員が面談日時を確定したとき（教員側の操作。「教師面談機能」を参照）。
- `confirmed` → `completed`：教員が面談完了を記録したとき（教員側の操作）。
- `requested` / `scheduling` / `confirmed` → `cancelled`：生徒または教員が取り消しを行ったとき。
- 上記以外の遷移は許可されない。

---

# 8. 権限制御

- `student` ロールのユーザーのみ、本仕様書に記載のエンドポイントを利用できる。
- 生徒は、自分が申請した（自分が生徒として当事者になっている）面談のみ閲覧・操作できる。他の生徒が関わる面談は一切参照できない。
- 面談の確定・完了の操作は教員側にのみ許可されており、生徒はこれらを実行できない。
- 取り消し操作には、表示中データの版数（`lock_version`）が要求され、他の操作との競合が防止される。

---

# 9. 非同期処理

- 面談が新規申請されると、相手方の教員へお知らせが送られる。
- 面談が取り消されると、相手方へ取り消しの旨（理由が入力されていればその内容も含む）のお知らせが送られる。
- メッセージが投稿されると、相手方へ新着メッセージのお知らせが送られる。
- いずれも、面談申請やメッセージの作成処理そのものとは切り離された非同期処理として実行される。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::InterviewRequestsController`, `Api::V1::Student::InterviewRequestMessagesController`|
|Form|`Student::CreateInterviewRequestForm`|
|Service|`Student::CreateInterviewRequestService`, `Student::CancelInterviewRequestService`, `Student::CreateInterviewRequestMessageService`, `Common::CancelInterviewRequestService`, `Common::PostInterviewRequestMessageService`|
|Model|`InterviewRequest`, `InterviewRequestMessage`|
|Serializer|`InterviewRequestSerializer`, `InterviewRequestMessageSerializer`|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
