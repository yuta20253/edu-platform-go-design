# 管理者お知らせ管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が全ユーザー向けのお知らせを作成・編集・配信・削除できるようにすること。
- システム上の役割
  - 管理者が作成したお知らせの一覧・詳細表示を提供する。
  - お知らせの新規作成、内容の更新、削除を提供する。
  - 下書き・予約状態のお知らせを配信（公開）状態に切り替える。
  - 予約状態のお知らせは、配信予定日時になると、定期実行される処理によっても配信状態に切り替わる（この処理は管理者・教師が作成したお知らせに共通で、「9. 非同期処理」参照）。
  - 高校ごとに、その高校を対象とするお知らせの一覧を提供する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|管理者が作成したお知らせを検索・状態絞り込みして一覧取得する|
|詳細取得|指定お知らせの詳細を取得する|
|新規登録|お知らせを作成する（下書き・予約・即時配信のいずれかの状態で登録）|
|更新|自分が発信者である下書き・予約状態のお知らせの内容や状態を更新する|
|削除|自分が発信者である未配信のお知らせを削除する|
|配信|自分が発信者である下書き・予約状態のお知らせを配信（公開）する|
|高校別お知らせ参照|指定高校を対象とするお知らせの一覧を取得する|

---

# 3. 業務フロー

1. 管理者がお知らせ一覧画面を開く。
2. キーワードや状態で絞り込んで、管理者が作成したお知らせの一覧を取得する。
3. 特定のお知らせを選択すると、その詳細を取得する。
4. 新規にお知らせを作成する場合、タイトル・内容・状態（下書き／予約／即時配信）を入力して登録する。予約配信の場合は配信予定日時を指定する。
5. 下書きまたは予約状態のお知らせは、発信者本人が内容や状態を更新できる。
6. 下書きまたは予約状態のお知らせは、発信者本人が配信操作によって公開状態に切り替えられる。公開されると、その時点の日時が配信日時として記録される。予約状態のお知らせは、配信予定日時になると自動的にも公開状態に切り替わる。
7. 未配信のお知らせは、発信者本人が削除できる。配信済みのお知らせは削除・編集できない。
8. 他の管理者が発信者であるお知らせは、一覧・詳細での参照のみ可能で、更新・削除・配信はできない。ただし、発信者の管理者アカウントが無効化されている場合は、他の管理者が更新・削除・配信できる。
9. 高校ごとのお知らせ一覧画面では、指定した高校を対象に含むお知らせのみが表示される。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::AnnouncementsController`|`index`|GET|`/api/v1/admin/announcements`|お知らせ一覧を取得|
|`Api::V1::Admin::AnnouncementsController`|`show`|GET|`/api/v1/admin/announcements/:id`|お知らせ詳細を取得|
|`Api::V1::Admin::AnnouncementsController`|`create`|POST|`/api/v1/admin/announcements`|お知らせを作成|
|`Api::V1::Admin::AnnouncementsController`|`update`|PATCH|`/api/v1/admin/announcements/:id`|お知らせを更新|
|`Api::V1::Admin::AnnouncementsController`|`destroy`|DELETE|`/api/v1/admin/announcements/:id`|お知らせを削除|
|`Api::V1::Admin::AnnouncementsController`|`publish`|POST|`/api/v1/admin/announcements/:id/publish`|お知らせを配信|
|`Api::V1::Admin::HighSchools::AnnouncementsController`|`index`|GET|`/api/v1/admin/high_schools/:high_school_id/announcements`|高校別のお知らせ一覧を取得|

---

# 5. API / 処理詳細

## `Api::V1::Admin::AnnouncementsController#index`

### 概要

管理者が作成したお知らせを、キーワード・状態で絞り込んで一覧取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`q`|任意|string|タイトルによるキーワード検索|
|`status`|任意|string|状態による絞り込み（`draft`、`scheduled`、`published`）|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたり件数|

### 処理内容

1. 管理者が発信者であるお知らせを対象とする。
2. `q` が指定されていればタイトルの部分一致で絞り込む。
3. `status` が有効な値であれば状態で絞り込む。
4. 作成日時の新しい順に並び替え、ページングして返却する。

### 業務ルール

- 対象は管理者自身ではなく「管理者ロールのユーザーが発信者であるお知らせ」全体であり、他の管理者が作成したお知らせも一覧に含まれる。
- `status` に許可されていない値が指定された場合は絞り込みを行わない。

### Database変更

- なし

### Response

- `announcements`: 各お知らせの `id`, `title`, `status`, `target_type`, `published_at`, `scheduled_at`, `created_at`, `publisher`（`id`, `name`, `name_kana`）
- `meta`: `current_page`, `total_pages`, `total_count`, `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Admin::AnnouncementsController#show`

### 概要

指定お知らせの詳細（本文を含む）を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの ID|

### 処理内容

1. 管理者が発信者であるお知らせの中から、指定 ID のお知らせを取得する。
2. お知らせ詳細（本文を含む）を返却する。

### 業務ルール

- なし

### Database変更

- なし

### Response

- `id`, `title`, `content`, `status`, `target_type`, `published_at`, `scheduled_at`, `created_at`, `publisher`（`id`, `name`, `name_kana`）

### Errorケース

- 対象お知らせが存在しない|404|対象お知らせなし

## `Api::V1::Admin::AnnouncementsController#create`

### 概要

新しいお知らせを作成する。作成されたお知らせは全ユーザーを対象とする。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`announcement[title]`|必須|string|タイトル（最大255文字）|
|`announcement[content]`|必須|string|内容（最大10,000文字）|
|`announcement[status]`|必須|string|状態（`draft`、`scheduled`、`published`）|
|`announcement[scheduled_at]`|任意|datetime|配信予定日時（`scheduled` 指定時は必須）|

### 処理内容

1. 入力値を検証する。
2. お知らせを新規作成し、対象を「全ユーザー」として登録する。
3. 指定された状態（下書き・予約・即時配信）で保存する。即時配信の場合はその時点の日時が配信日時として記録される。

### 業務ルール

- タイトルは必須かつ255文字以内、内容は必須かつ10,000文字以内。
- 状態は `draft`、`scheduled`、`published` のいずれかを指定する。
- 状態を `scheduled` にする場合、配信予定日時は必須であり、かつ未来日時である必要がある。
- 対象は常に「全ユーザー」であり、特定の高校・学年・ユーザーを対象として指定することはできない。

### Database変更

作成:

```
announcements
announcement_targets
```

### Response

- `message`: `お知らせを作成しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|バリデーション失敗|422|`errors` を返却|

## `Api::V1::Admin::AnnouncementsController#update`

### 概要

既存のお知らせの内容や状態を更新する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの ID|
|`announcement[title]`|任意|string|タイトル|
|`announcement[content]`|任意|string|内容|
|`announcement[status]`|任意|string|状態|
|`announcement[scheduled_at]`|任意|datetime|配信予定日時|

### 処理内容

1. 管理者が発信者であるお知らせの中から対象を取得する。
2. 操作者が対象お知らせを更新できる管理者であるか確認する。
3. 入力された項目のみを更新する。
4. 状態を変更する場合は、現在の状態から遷移可能な状態であるかを検証する。

### 業務ルール

- 更新できるのは、お知らせの発信者本人である管理者のみである。他の管理者が発信者であるお知らせは更新できない。ただし、発信者の管理者アカウントが無効化されている場合は、他の管理者でも更新できる。
- 更新権限の確認は、配信済みかどうかの判定より先に行われる。そのため、他の管理者が発信者である配信済みのお知らせを更新しようとした場合も、配信済みのエラーではなく権限エラー（403）となる。
- 状態は `draft → scheduled`、`draft → published`、`scheduled → published` の順にのみ変更でき、それ以外への変更はできない。
- すでに配信済み（`published`）のお知らせは、内容・状態を含め一切更新できない。
- 状態を `scheduled` にする場合、配信予定日時は必須であり、かつ未来日時である必要がある。

### Database変更

更新:

```
announcements
```

### Response

- `message`: `お知らせを更新しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|発信者本人でない管理者が更新しようとした（発信者が無効化されている場合を除く）|403|`errors`: `この操作を行う権限がありません`|
|バリデーション失敗（不正な状態遷移・配信済みの更新など）|422|`errors` を返却（配信済みの場合は `は配信済みのため編集できません`）|
|対象お知らせが存在しない|404|対象お知らせなし|

## `Api::V1::Admin::AnnouncementsController#destroy`

### 概要

未配信のお知らせを削除する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの ID|

### 処理内容

1. 管理者が発信者であるお知らせの中から対象を取得する。
2. 操作者が対象お知らせを削除できる管理者であるか確認する。
3. お知らせを削除する。

### 業務ルール

- 削除できるのは、お知らせの発信者本人である管理者のみである。他の管理者が発信者であるお知らせは削除できない。ただし、発信者の管理者アカウントが無効化されている場合は、他の管理者でも削除できる。
- 削除権限の確認は、配信済みかどうかの判定より先に行われる。そのため、他の管理者が発信者である配信済みのお知らせを削除しようとした場合も、配信済みのエラーではなく権限エラー（403）となる。
- 配信済み（`published`）のお知らせは削除できない。

### Database変更

削除:

```
announcements
announcement_targets
```

### Response

- `204 No Content`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|発信者本人でない管理者が削除しようとした（発信者が無効化されている場合を除く）|403|`errors`: `この操作を行う権限がありません`|
|配信済みのお知らせを削除しようとした場合|422|`errors`: `は配信済みのため削除できません`|
|対象お知らせが存在しない|404|対象お知らせなし|

## `Api::V1::Admin::AnnouncementsController#publish`

### 概要

下書きまたは予約状態のお知らせを配信（公開）状態にする。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの ID|

### 処理内容

1. 管理者が発信者であるお知らせの中から対象を取得する。
2. 操作者が対象お知らせを配信できる管理者であるか確認する。
3. すでに配信済みであればエラーとする。
4. 状態を `published` に更新し、配信予定日時をクリアする。このとき配信日時としてその時点の日時が記録される。

### 業務ルール

- 配信できるのは、お知らせの発信者本人である管理者のみである。他の管理者が発信者であるお知らせは配信できない。ただし、発信者の管理者アカウントが無効化されている場合は、他の管理者でも配信できる。
- 配信権限の確認は、配信済みかどうかの判定より先に行われる。
- すでに配信済みのお知らせを再度配信することはできない。
- 配信後は、状態・配信予定日時を含め編集・削除ができなくなる。

### Database変更

更新:

```
announcements
```

### Response

- `message`: `お知らせを配信しました。`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|発信者本人でない管理者が配信しようとした（発信者が無効化されている場合を除く）|403|`errors`: `この操作を行う権限がありません`|
|すでに配信済み|422|`errors`: `はすでに配信済みです`|
|対象お知らせが存在しない|404|対象お知らせなし|

## `Api::V1::Admin::HighSchools::AnnouncementsController#index`

### 概要

指定された高校を対象に含むお知らせの一覧を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`high_school_id`|必須|integer|高校の ID|
|`page`|任意|integer|ページ番号|

### 処理内容

1. 指定された高校を取得する。
2. その高校を対象に含むお知らせを、作成日時の新しい順に取得する。
3. 各お知らせについて、指定高校に対応する対象指定情報とともに返却する。

### 業務ルール

- 対象範囲が「全ユーザー」ではなく、特定の高校・学年・ユーザーを対象とするお知らせのうち、指定高校を対象に含むもののみが一覧に含まれる。

### Database変更

- なし

### Response

- `announcements`: 各お知らせの `id`, `title`, `content`, `status`, `published_at`, `scheduled_at`, `created_at`, `publisher`（`id`, `name`, `name_kana`）, `targets`（指定高校に対応する対象指定情報）
- `meta`: `current_page`, `total_pages`, `total_count`, `per_page`

### Errorケース

- 対象高校が存在しない|404|対象高校なし

---

# 6. データモデル

## `announcements`

- 役割: お知らせを管理するテーブル
- 主なカラム: `id`, `title`, `content`, `status`, `published_at`, `scheduled_at`, `publisher_id`
- リレーション: `belongs_to :publisher`（ユーザー）, `has_many :announcement_targets`

## `announcement_targets`

- 役割: お知らせの配信対象を管理するテーブル
- 主なカラム: `id`, `announcement_id`, `target_type`, `user_role_id`, `grade_id`, `high_school_id`, `user_id`
- リレーション: `belongs_to :announcement`
- 管理者が作成するお知らせの対象は常に「全ユーザー」（`target_type: all_users`）であり、それ以外の対象種別（役割別・学年別・高校別・個人別）は教師によるお知らせ作成機能で使用される。

---

# 7. 状態管理

お知らせは以下の状態を持つ。

```
draft（下書き）
↓
scheduled（配信予約）
↓
published（配信済み）
```

状態変更条件:

- 下書き（`draft`）は、予約（`scheduled`）または配信済み（`published`）へ直接変更できる。
- 予約（`scheduled`）は配信済み（`published`）へのみ変更できる。
- 配信済み（`published`）は最終状態であり、それ以降の状態変更・内容編集・削除は一切できない。
- 予約（`scheduled`）状態にするには配信予定日時（未来日時）の指定が必須である。
- 配信操作（`publish`）が実行されると状態が `published` になり、その時点の日時が配信日時として記録される。この操作では、配信予定日時（`scheduled_at`）は空になる。
- 予約（`scheduled`）状態のお知らせは、上記の配信操作のほか、配信予定日時になった時点で、定期実行される処理（「9. 非同期処理」参照）によっても配信済み（`published`）に変更される。この場合も配信日時が記録されるが、配信予定日時は空にならず値が残る。

---

# 8. 権限制御

- 管理者は、管理者ロールのユーザーが発信者であるお知らせを参照できる。他の管理者が発信者であるお知らせも参照できる。
- 管理者は、お知らせを新規に作成できる。作成したお知らせの発信者は作成した管理者本人となる。
- お知らせの更新・削除・配信は、そのお知らせの発信者本人である管理者のみが実行できる。他の管理者が発信者であるお知らせを更新・削除・配信しようとした場合は権限エラー（403）となる。
- 発信者の管理者アカウントが無効化されている場合は、発信者本人が操作できなくなるため、他の管理者が更新・削除・配信できる（操作不能のまま残ることを防ぐ）。
- 更新・削除・配信では、権限の確認を配信済みかどうかの判定より先に行う。発信者本人が配信済みのお知らせを編集・削除・再配信しようとした場合は、権限エラーではなく業務エラー（422）となる。
- 配信済みのお知らせは、発信者本人であっても編集・削除の対象外となる。
- 高校別のお知らせ参照は、対象範囲にその高校が含まれるお知らせのみが表示される。

---

# 9. 非同期処理

- 予約配信の自動切替: 予約（`scheduled`）状態で配信予定日時に達したお知らせを、自動的に配信（`published`）状態へ切り替える定期ジョブが存在する（`PublishScheduledAnnouncementsJob`）。管理者が作成したお知らせも、教師が作成したお知らせと区別されずに対象となる。
  - 実行タイミング: 1分ごと（Sidekiq-cron。`config/schedule.yml`）。この定期実行の登録は、Sidekiq サーバーが環境変数 `ENABLE_SIDEKIQ_CRON` を `true` に設定して起動している場合にのみ行われる（設定値は環境ごとに異なり、コードからは本番環境で有効かどうかを確認できない）。
  - 処理内容: 予約状態で配信予定日時が現在時刻以前のお知らせを1件ずつ配信状態に更新し、配信日時を記録する（配信予定日時は空にしない）。1件の更新に失敗してもエラーとして記録し、他のお知らせの処理を継続する。
  - 影響範囲: `announcements` の `status` と `published_at`。
- 管理者による配信操作（`publish`）は、上記の自動切替とは別に、発信者本人が明示的に呼び出せる。この操作は配信予定日時を空にする。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::AnnouncementsController`, `Api::V1::Admin::HighSchools::AnnouncementsController`|
|Policy|`AnnouncementPolicy`|
|Query|`AnnouncementsQuery`|
|Form|`Admin::AnnouncementForm`|
|Service|`Admin::CreateAnnouncementService`, `Admin::PublishAnnouncementService`, `Common::AnnouncementCreateService`|
|Serializer|`Admin::AnnouncementListSerializer`, `Admin::AnnouncementDetailSerializer`, `Admin::AnnouncementSerializer`, `Admin::AnnouncementTargetSerializer`, `AnnouncementPublisherSerializer`|
|Model|`Announcement`, `AnnouncementTarget`|

注意:

この情報は現在実装との対応確認用です。

新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
