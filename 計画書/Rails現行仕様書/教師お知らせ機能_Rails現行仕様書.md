# 教師お知らせ機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 教員が、自校の生徒・教員に向けたお知らせを作成・更新し、自分に配信されたお知らせも読めるようにすること。
- システム上の役割
  - 作成と参照の両方を担う。生徒向けのお知らせ機能が参照だけであるのに対し、本機能は**自分が発行したお知らせの管理**を含む。
  - 作成時に**配信先を指定する。** 全員・権限別・学年別・学校別・ユーザー個別の5種類から、1件以上を指定する。
  - **教員が作成するお知らせは自校内にのみ配信される。** 配信先として「全員」「権限別」を指定した場合も、発行者の所属校が自動的に付与される。
  - 担当学年が制限されている教員は、**自分の学年に絞り込める配信先しか指定できない。**
  - 下書き・予約・即時配信のいずれかの状態で作成できる。配信済みにしたお知らせは編集できない。
- 利用者
  - `teacher` ロールのユーザー（教員）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|お知らせ一覧取得|自分に配信されたお知らせ、または自分が作成したお知らせを、タブの指定で切り替えて取得する|
|お知らせ詳細取得|自分に配信された配信済みのお知らせ1件を取得する|
|新規作成フォームの選択肢取得|配信先として指定できる学年・権限・生徒の一覧と、自分の学年制限を取得する|
|編集フォームの内容取得|自分が作成した編集可能なお知らせと、その配信先を取得する|
|お知らせ作成|件名・本文・配信先・配信方法を指定してお知らせを作成する|
|お知らせ更新|自分が作成した編集可能なお知らせの内容・配信先・配信方法を更新する|

---

# 3. 業務フロー

## お知らせの作成

1. 教員が新規作成フォームを開く。
2. システムは配信先として指定できる学年・権限・生徒の一覧と、教員自身の学年制限を返す。
3. 教員が件名・本文・配信先・配信方法（下書き・予約・即時配信）を入力する。
4. システムは入力内容と配信先を検証する。学年制限のある教員が指定できない配信先を選んでいる場合、他校の学年・ユーザーを指定している場合は拒否する。
5. システムはお知らせと配信先を保存する。配信先には発行者の所属校を付与して自校内に絞り込む。
6. 配信方法に応じたメッセージを返す。

## お知らせの更新

1. 教員が自分の作成したお知らせの編集フォームを開く。
2. システムはお知らせが編集可能であることを確認する。配信済みの場合はエラーとして返す。
3. システムはお知らせの内容と、現在の配信先を返す。
4. 教員が内容・配信先・配信方法を変更する。
5. システムは作成時と同じ検証を行う。
6. システムはお知らせを更新し、**配信先を差分ではなく全件入れ替える。**

## 自分に配信されたお知らせの閲覧

1. 教員がお知らせ一覧を要求する。タブの指定を省略する。
2. システムは、配信先の指定と教員の所属・識別子を突き合わせ、配信済みのものだけを返す。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Teacher::AnnouncementsController`|`index`|GET|`/api/v1/teacher/announcements`|お知らせ一覧を取得|
|`Api::V1::Teacher::AnnouncementsController`|`show`|GET|`/api/v1/teacher/announcements/:id`|お知らせ詳細を取得|
|`Api::V1::Teacher::AnnouncementsController`|`new`|GET|`/api/v1/teacher/announcements/new`|新規作成フォームの選択肢を取得|
|`Api::V1::Teacher::AnnouncementsController`|`edit`|GET|`/api/v1/teacher/announcements/:id/edit`|編集フォームの内容を取得|
|`Api::V1::Teacher::AnnouncementsController`|`create`|POST|`/api/v1/teacher/announcements`|お知らせを作成|
|`Api::V1::Teacher::AnnouncementsController`|`update`|PATCH / PUT|`/api/v1/teacher/announcements/:id`|お知らせを更新|

`new`・`edit` が参照用のAPIとして提供されている点に注意する。画面を描くための選択肢・初期値を返すものであり、HTMLを返すものではない。

---

# 5. API / 処理詳細

## `Api::V1::Teacher::AnnouncementsController#index`

### 概要

タブの指定によって、自分に配信されたお知らせと自分が作成したお知らせを切り替えて返す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`tab`|任意|string|`authored` を指定すると自分が作成したお知らせを返す。それ以外・未指定のときは自分に配信されたお知らせを返す|
|`page`|任意|integer|ページ番号。0以下・数値以外は未指定として扱う|
|`per_page`|任意|integer|1ページあたりの件数。既定20、上限100|

### 処理内容

1. `tab` の指定を判定し、対象となるお知らせの範囲を決める。
2. 配信日時の降順、同値なら識別子の降順に並べる。
3. 指定されたページを取得する。
4. タブに応じた項目を返す。

### 業務ルール

#### タブによる切り替え

|`tab`|対象|返す項目|
|-|-|-|
|`authored`|**自分が発行したお知らせのうち、システムが自動生成したものを除く。** 状態による絞り込みを行わないため、下書き・予約も含む|識別子・件名・本文・状態・配信日時・予約日時。発行者は返さない|
|それ以外・未指定|自分に配信されたお知らせのうち、状態が配信済みのもの|識別子・件名・本文・配信日時・発行者|

- `authored` では自校・他校の区別を行わない。発行者が自分であることだけが条件である。
- `authored` では**システムが自動生成したお知らせを除外する。** 承認・却下等の通知は、発行者が教員になることがあるが、本人が作成したものではないためである。
- 配信されたお知らせ側の絞り込み（配信先の突き合わせ）は、お知らせ機能①「5. API / 処理詳細」`#index` と同じである。
- 並び順はいずれのタブでも配信日時の降順。**`authored` では下書き・予約が含まれ、それらの配信日時は未設定であるため、並び順が安定しない。**
- ページネーションの既定値は20件。生徒向けのお知らせ一覧（既定5件）と異なる。

### Database変更

- なし

### Response

HTTP Status は `200`。

`tab=authored` のとき:

|項目|型|説明|
|-|-|-|
|`announcements[].id`|integer|お知らせの識別子|
|`announcements[].title`|string|件名|
|`announcements[].content`|string|本文|
|`announcements[].status`|string|状態（`draft` / `scheduled` / `published`）|
|`announcements[].published_at`|datetime / null|配信日時|
|`announcements[].scheduled_at`|datetime / null|予約日時|
|`meta`|object|`current_page` / `total_pages` / `total_count` / `per_page`|

それ以外のとき:

|項目|型|説明|
|-|-|-|
|`announcements[].id`|integer|お知らせの識別子|
|`announcements[].title`|string|件名|
|`announcements[].content`|string|本文|
|`announcements[].published_at`|datetime|配信日時|
|`announcements[].publisher`|object|発行者（`id` / `name` / `name_kana`）|
|`meta`|object|同上|

**タブによってレスポンスの項目が変わる。** 配信先（`announcement_targets`）はいずれのタブでも返さない。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|該当なし|200|空の配列とページ情報を返す|
|未認証|401|認証されていない|
|`teacher` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

## `Api::V1::Teacher::AnnouncementsController#show`

### 概要

自分に配信された配信済みのお知らせ1件を返す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの識別子。パスに含まれる|

### 処理内容

1. 操作者に配信された配信済みのお知らせを絞り込む。
2. 指定された識別子のお知らせを取得する。該当しない場合はエラーを返す。

### 業務ルール

- 絞り込みは配信先の突き合わせと配信済みの条件のみ（お知らせ機能①「5. API / 処理詳細」`#index`）。
- **自分が作成したお知らせであっても、自分に配信されていなければ `404` となる。** `tab=authored` の一覧に現れるお知らせを詳細で開けるとは限らない。下書き・予約のお知らせも `404` である。
- 一覧と異なり `includes(:publisher)` を行わないが、レスポンスに発行者を含めるため結果は同じである。

### Database変更

- なし

### Response

HTTP Status は `200`。最上位はJSONオブジェクト。項目は `id`・`title`・`content`・`published_at`・`publisher`。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|自分に配信されていない、配信済みでない、存在しない|404|`{"message": "Announcementが見つかりません"}`|
|未認証|401|認証されていない|
|`teacher` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

## `Api::V1::Teacher::AnnouncementsController#new`

### 概要

配信先として指定できる学年・権限・生徒の一覧と、操作者の学年制限を返す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`keyword`|任意|string|生徒の絞り込みに使うキーワード|
|`page`|任意|integer|生徒一覧のページ番号|
|`per_page`|任意|integer|生徒一覧の1ページあたりの件数。既定20、上限100|

### 処理内容

1. 操作者の学年制限を取得する。
2. 学年制限がある場合はその学年のみ、ない場合は所属校の全学年を選択肢として取得する。
3. 権限の選択肢を取得する。
4. 所属校の生徒を、学年制限とキーワードで絞り込んでページ単位で取得する。
5. 学年・権限・生徒・学年制限をまとめて返す。

### 業務ルール

- 学年制限は、操作者の権限設定が「自分の学年のみ」の場合に操作者の学年となる。それ以外は制限なし（`null`）。
- 学年の選択肢は、学年制限があればその学年1件のみ、なければ所属校の全学年。
- **権限の選択肢から管理者と保護者を除く。** 教員が指定できるのは教員・生徒に相当する権限のみである。
- 生徒の一覧は所属校の生徒に限る。学年制限があればその学年のみ。並び順は氏名カナの昇順。
- 生徒の一覧のみページネーションの対象であり、学年・権限は全件を返す。

### Database変更

- なし

### Response

HTTP Status は `200`。

|項目|型|説明|
|-|-|-|
|`grades`|array|学年の選択肢（`id` / `year` / `display_name`）|
|`user_roles`|array|権限の選択肢|
|`students.items`|array|生徒の一覧|
|`students.meta`|object|生徒一覧のページ情報|
|`own_grade_restriction`|integer / null|操作者の学年制限。制限がない場合は `null`|

生徒一覧だけが `items` と `meta` の入れ子になっている点に注意する。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|未認証|401|認証されていない|
|`teacher` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

## `Api::V1::Teacher::AnnouncementsController#edit`

### 概要

自分が作成した編集可能なお知らせと、その配信先を返す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|お知らせの識別子。パスに含まれる|

### 処理内容

1. 操作者が発行したお知らせを絞り込み、指定された識別子のものを取得する。該当しない場合はエラーを返す。
2. お知らせが編集可能であることを確認する。配信済みの場合はエラーを返す。
3. お知らせの内容と、現在の配信先を返す。

### 業務ルール

- **自分が発行したお知らせのみ対象。** 他の教員が作成したものは `404` となる。
- **配信済みのお知らせは `404` となる。** 編集できないためであり、「存在しない」と同じ扱いで返す。`403` ではない。
- システムが自動生成したお知らせも、発行者が操作者であれば対象になりうる。`authored` タブの一覧とは絞り込みが異なる。
- 配信先は現在保存されている全件を返す。**ユーザー個別の配信先には、選択済みの生徒名も含める。**

### Database変更

- なし

### Response

HTTP Status は `200`。

|項目|型|説明|
|-|-|-|
|`announcement.id`|integer|お知らせの識別子|
|`announcement.title`|string|件名|
|`announcement.content`|string|本文|
|`announcement.status`|string|状態|
|`announcement.scheduled_at`|datetime / null|予約日時|
|`announcement_targets[].id`|integer|配信先の識別子|
|`announcement_targets[].target_type`|string|配信先の種類|
|`announcement_targets[].high_school_id`|integer / null|高校の識別子|
|`announcement_targets[].grade_id`|integer / null|学年の識別子|
|`announcement_targets[].user_role_id`|integer / null|権限の識別子|
|`announcement_targets[].user_id`|integer / null|ユーザーの識別子|
|`announcement_targets[].user_name`|string / null|ユーザーの氏名。ユーザー個別以外では `null`|

`announcement` に `published_at` は含めない。編集対象が配信済みでないため常に未設定である。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|他の教員が作成した、存在しない|404|`{"message": "Announcementが見つかりません"}`|
|**配信済みで編集できない**|404|同上|
|未認証|401|認証されていない|
|`teacher` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

## `Api::V1::Teacher::AnnouncementsController#create`

### 概要

件名・本文・配信先・配信方法を指定してお知らせを作成する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`announcement[title]`|必須|string|件名|
|`announcement[content]`|必須|string|本文。10,000文字以内|
|`announcement[status]`|任意|string|`draft` / `scheduled` / `published`。未指定・空文字のときは `draft`|
|`announcement[scheduled_at]`|条件付き必須|datetime|予約日時。`status` が `scheduled` のときに必須|
|`announcement[announcement_targets]`|必須|array|配信先の配列。1件以上|
|`announcement[announcement_targets][][target_type]`|必須|string|配信先の種類|
|`announcement[announcement_targets][][grade_id]`|条件付き|integer|学年別のときに必須|
|`announcement[announcement_targets][][user_role_id]`|条件付き|integer|権限別・学年別のときに必須|
|`announcement[announcement_targets][][user_id]`|条件付き|integer|ユーザー個別のときに必須|

`announcement` キー自体が存在しない場合は必須パラメータ不足として扱われる。

### 処理内容

1. 入力内容を検証する。
2. 配信先の形式・存在・自校内であることを検証する。
3. 学年制限のある教員の場合、指定できる配信先かを検証する。
4. 検証に失敗した場合はエラーを返す。
5. お知らせを作成する。
6. 配信先を1件ずつ作成する。全員・権限別には発行者の所属校を付与する。
7. 配信方法に応じたメッセージと、作成したお知らせの識別子を返す。

お知らせの作成と配信先の作成は1つのトランザクションで行う。

### 業務ルール

#### 入力の検証

|項目|ルール|エラーメッセージ|
|-|-|-|
|件名|必須|`Title を入力してください`|
|本文|必須、10,000文字以内|`Content を入力してください` / `Content は10000文字以内で入力してください`|
|状態|`draft` / `scheduled` / `published` のいずれか。空文字は許す|`Status は一覧にありません`|
|予約日時|状態が `scheduled` のとき必須、かつ未来日時|`配信日時 を指定してください` / `配信日時 は未来日時を指定してください`|
|配信先|必須（1件以上）、配列であること|`Announcement targets を入力してください` / `Announcement targets 配列で指定してください`|

件名・本文・状態・配信先の項目名が英語表記になるのは、日本語訳が定義されていないためである。予約日時のみ「配信日時」という訳がある。

#### 配信先の検証

|検証|条件|エラーメッセージ|
|-|-|-|
|種類の妥当性|既知の5種類のいずれかであること|`不正なtarget_typeが含まれています`|
|学年の指定|学年別のときに学年が指定されていること|`Announcement targets 学年を指定してください`|
|学年の存在|指定された学年が存在すること|`Announcement targets 存在しない学年です`|
|権限の指定|権限別・学年別のときに権限が指定されていること|`Announcement targets 権限を指定してください`|
|権限の存在|指定された権限が存在すること|`Announcement targets 存在しない権限です`|
|ユーザーの指定|ユーザー個別のときにユーザーが指定されていること|`Announcement targets ユーザーを指定してください`|
|ユーザーの存在|指定されたユーザーが存在すること|`Announcement targets 存在しないユーザーです`|
|ユーザーの所属校|指定されたユーザーが操作者と同じ高校に所属すること|`Announcement targets 他校のユーザーは指定できません`|
|学年の所属校|指定された学年が操作者と同じ高校に属すること|`Announcement targets 他校の学年は指定できません`|

**学年別では権限も必須である。** 学年だけでなく「その学年のどの権限の利用者か」まで指定する必要がある。

#### 学年制限のある教員の制約

操作者の権限設定が「自分の学年のみ」の場合、次の制約が加わる。

|制約|エラーメッセージ|
|-|-|
|全員・権限別・学校別を指定できない（学年で絞り込めず学年外にも届くため）|`Announcement targets 学年が制限されているため指定できない配信先です`|
|学年別は自分の学年のみ|`Announcement targets 指定できない学年です`|
|ユーザー個別は、自分の学年の生徒のみ|`Announcement targets 指定できないユーザーです`|

ユーザー個別の制約は「生徒であり、かつ学年が一致する」ことを求める。同じ学年の教員は指定できない。

#### 自校内への絞り込み

- 配信先が全員・権限別の場合、**発行者の所属校を自動的に付与する。** これにより、他校の利用者にお知らせが届かないようにしている。
- 配信先が学校別の場合、指定された値ではなく**発行者の所属校を使う。**
- 配信先が学年別・ユーザー個別の場合は、指定された学年・ユーザーをそのまま使う。検証で自校内であることが保証されている。

#### 状態と配信日時

- 状態が未指定・空文字のときは下書きとして作成する。
- 状態が配信済みのときは、配信日時に現在時刻が設定される。
- 状態が予約のときは、予約日時が必須かつ未来日時であることが求められる。
- システムが自動生成したお知らせではないため、`system_generated` は `false` となる。

### Database変更

作成:

```
announcements
announcement_targets
```

### Response

HTTP Status は `201`。

|項目|型|説明|
|-|-|-|
|`message`|string|状態に応じたメッセージ|
|`announcement_id`|integer|作成したお知らせの識別子|

状態に応じたメッセージ:

|状態|メッセージ|
|-|-|
|`draft`|お知らせを下書きで作成しました。|
|`scheduled`|お知らせの配信を予約しました。|
|`published`|お知らせを配信しました。|

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|入力・配信先の検証に失敗|422|`{"errors": [メッセージ, ...]}`。複数同時に含まれうる|
|`announcement` パラメータ自体が存在しない|400|必須パラメータ不足として扱われ、個別のエラー本文は定義されていない|
|配信先に既知でない種類が含まれ、検証をすり抜けた場合|500|配信先の組み立てで例外になる。ただし検証で弾かれるため到達しない|
|未認証|401|認証されていない|
|`teacher` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

## `Api::V1::Teacher::AnnouncementsController#update`

### 概要

自分が作成した編集可能なお知らせの内容・配信先・配信方法を更新する。

### Request

パスパラメータ `id` に加えて、作成と同じ項目を受け取る。

|項目|必須|型|説明|
|-|-|-|-|
|`announcement[title]`|-|string|件名。**Form では必須検証を行わない**|
|`announcement[content]`|-|string|本文。**Form では必須検証・長さ検証を行わない**|
|`announcement[status]`|任意|string|状態|
|`announcement[scheduled_at]`|条件付き|datetime|予約日時|
|`announcement[announcement_targets]`|必須|array|配信先の配列|

### 処理内容

1. 操作者が発行したお知らせを絞り込み、指定された識別子のものを取得する。該当しない場合はエラーを返す。
2. お知らせが編集可能であることを検証する。配信済みの場合はエラーを返す。
3. 配信先の形式・存在・自校内であること、学年制限の制約を検証する。
4. お知らせを更新する。
5. **既存の配信先をすべて削除し、指定された配信先を作り直す。**
6. 成功のメッセージを返す。

更新と配信先の入れ替えは1つのトランザクションで行う。処理の前にお知らせを排他ロックする。

### 業務ルール

- **自分が発行したお知らせのみ対象。** 他の教員が作成したものは `404` となる。
- **配信済みのお知らせは編集できない。** `422` で `配信済みのお知らせは編集できません` を返す。`edit` が `404` を返すのとは扱いが異なる。
- 配信先の検証・学年制限の制約は作成と同じ。
- **配信先は差分更新ではなく全件入れ替えである。** 指定しなかった配信先は削除される。
- 件名・本文の必須検証・長さ検証を Form では行わない。ただしモデル側の検証が働くため、空の件名・本文や10,000文字超の本文は拒否される。
- 配信先の組み立てには**お知らせの発行者**を使う。操作者と発行者は一致するため結果は同じだが、実装上は発行者を参照している。
- 状態の遷移は許される組み合わせに限る。下書き → 予約・配信済み、予約 → 配信済みのみ。それ以外は `status は {遷移元} から {遷移先} へは変更できません` を返す。
- 状態を変えない更新も可能である。

### Database変更

更新:

```
announcements
```

削除・作成:

```
announcement_targets
```

### Response

HTTP Status は `200`。

|項目|型|説明|
|-|-|-|
|`message`|string|お知らせを更新しました。|

更新後のお知らせの内容は返さない。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|配信済みのため編集できない|422|`{"errors": ["配信済みのお知らせは編集できません"]}`|
|配信先の検証に失敗|422|`{"errors": [メッセージ, ...]}`|
|件名・本文がモデルの検証に失敗|422|`{"errors": [メッセージ, ...]}`|
|状態の遷移が許されない組み合わせ|422|`{"errors": ["status は {遷移元} から {遷移先} へは変更できません"]}`|
|他の教員が作成した、存在しない|404|`{"message": "Announcementが見つかりません"}`|
|`announcement` パラメータ自体が存在しない|400|必須パラメータ不足として扱われる|
|未認証|401|認証されていない|
|`teacher` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

# 6. データモデル

本機能が参照・更新するテーブルは、お知らせ機能①「6. データモデル」が整理している `announcements`・`announcement_targets` と同じである。加えて配信先の検証・選択肢の取得で `users`・`user_roles`・`grades`・`high_schools`・`teacher_permissions` を参照する。

|テーブル|役割|本機能での使い方|
|-|-|-|
|`announcements`|お知らせ本体|作成・更新・参照|
|`announcement_targets`|お知らせの配信先|作成・全件削除・参照|
|`users`|ユーザー|発行者、配信先のユーザー、生徒の選択肢、所属校の確認|
|`user_roles`|ロール|配信先の権限、選択肢（管理者・保護者を除く）|
|`grades`|学年|配信先の学年、選択肢、所属校の確認|
|`high_schools`|高校|自校内への絞り込み、学年の選択肢|
|`teacher_permissions`|教員の権限設定|学年制限の判定|

リレーション・列の意味・状態と配信先の種類の対応は、お知らせ機能①「6. データモデル」のとおりである。

本機能が所有するのは `announcements`・`announcement_targets` への作成・更新の経路である。`users`・`user_roles`・`grades`・`high_schools`・`teacher_permissions` はいずれも他機能が管理する。

---

# 7. 状態管理

状態と遷移の規則は、お知らせ機能①「7. 状態管理」のとおりである。本機能が関与するのは次の経路である。

|操作|状態の変化|
|-|-|
|作成（状態未指定・`draft`）|なし（下書きで作成）|
|作成（`scheduled`）|なし（予約で作成）。予約日時が必須かつ未来日時|
|作成（`published`）|なし（配信済みで作成）。配信日時に現在時刻が設定される|
|更新|下書き → 予約・配信済み、予約 → 配信済みのみ許される。状態を変えない更新も可能|

- **配信済みのお知らせは編集できない。** 更新は `422`、編集フォームの取得は `404` となる。
- 本機能は配信済みのお知らせを削除する経路も持たない。削除の操作自体を提供していない。
- 予約状態のお知らせが予約日時に配信済みへ切り替わる処理は、本機能の外にある（管理者お知らせ管理機能①「9. 非同期処理」）。**教員が作成した予約も自動切替の対象である。**

---

# 8. 権限制御

- 利用可能ユーザー: `teacher` ロールのユーザーのみ。
- 未認証のアクセスは `401`。`teacher` 以外のロールは `403`。

|操作|制限|
|-|-|
|一覧（`authored`）|発行者が操作者のものに限る|
|一覧（それ以外）・詳細|配信先の突き合わせで操作者に配信されたものに限る|
|新規作成フォーム|所属校の生徒・学年に限る。学年制限があればその学年に限る|
|編集フォーム・更新|発行者が操作者のものに限る。かつ編集可能（配信済みでない）であること|
|作成|配信先が自校内であること。学年制限があれば指定できる配信先に限る|

- 所有権の確認は、操作者が発行したお知らせの範囲で識別子を探す形で行う。他の教員のお知らせは `404` となり、`403` にはならない。
- **学年制限は教員の権限設定に基づく。** 権限設定が「自分の学年のみ」の教員は、学年外に届く配信先を指定できない。
- 配信先の検証で他校のユーザー・学年を拒否することが、実質的な配信範囲の制限になっている。

---

# 9. 非同期処理

- 本機能に非同期処理はない。作成・更新はすべてリクエスト時に同期で実行される。
- 予約状態のお知らせを配信済みへ切り替える自動処理は別にある（管理者お知らせ管理機能①「9. 非同期処理」）。**本機能が作成した予約も対象である。**

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Teacher::AnnouncementsController`|
|Controller Concern|`TeacherStudentsScope`（生徒の選択肢の取得）|
|Form|`Teacher::CreateAnnouncementForm`, `Teacher::UpdateAnnouncementForm`|
|Form Concern|`AnnouncementTargetsValidatable`（配信先の形式・存在・自校内）, `AnnouncementTargetGradeScopeValidatable`（学年制限の制約）|
|Service|`Teacher::CreateAnnouncementService`（`Common::AnnouncementCreateService` を継承し自校内に絞り込む）, `Teacher::UpdateAnnouncementService`|
|Service（共通）|`Common::AnnouncementCreateService`, `Common::BuildAnnouncementTargetAttributesService`|
|Model|`Announcement`, `AnnouncementTarget`, `User`, `UserRole`, `Grade`, `HighSchool`|
|Query|`Teacher::StudentsQuery`（生徒の選択肢の絞り込み。教師生徒参照機能が所有）|
|Serializer|`AnnouncementSerializer`, `AnnouncementPublisherSerializer`, `AuthoredAnnouncementSerializer`, `Teacher::AnnouncementEditSerializer`, `Teacher::AnnouncementTargetSerializer`, `GradeSerializer`, `UserRoleSerializer`, `StudentSerializer`|
|Job|なし|

自校内への絞り込みは `Common::AnnouncementCreateService` の設定を教員向けに切り替えることで実現している。管理者が作成するお知らせでは同じ切り替えを行わないため、全校の利用者に届く（管理者お知らせ管理機能①「5. API / 処理詳細」`#create`）。
