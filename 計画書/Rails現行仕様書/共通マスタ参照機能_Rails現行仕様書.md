# 共通マスタ参照機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 会員登録やプロフィール編集の際に、ユーザーが都道府県・住所・在籍高校・学年を選択肢として選べるように、これらのマスタ情報を検索・参照できるようにする。
- システム上の役割
  - 都道府県の一覧を提供する。
  - 都道府県を指定して、該当する住所（市区町村・町域）を検索する。
  - 都道府県やキーワードを指定して、在籍高校を検索する。
  - 高校を指定して、その高校に存在する学年の一覧を提供する。
  - いずれも新規データの作成・更新・削除は行わない、参照専用の機能である。
- 利用者
  - 会員登録前の未ログインユーザー（都道府県・高校・学年の検索）
  - ログイン中のユーザー（住所の検索。プロフィール編集などで利用する）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|都道府県一覧取得|登録されている都道府県の一覧を取得する|
|住所検索|都道府県を指定し、市区町村・町域で絞り込んで住所を検索する|
|高校検索|都道府県やキーワードを指定して在籍高校を検索する|
|学年一覧取得|指定した高校に存在する学年の一覧を取得する|

---

# 3. 業務フロー

## 都道府県一覧取得

1. クライアントが都道府県一覧を要求する。
2. システムは登録されているすべての都道府県を返却する。

## 住所検索

1. クライアントが都道府県IDを指定して住所検索を要求する（市区町村・町域による絞り込みは任意）。
2. 都道府県IDが指定されていない場合、エラーを返却する。
3. 都道府県ID・市区町村・町域の条件に合致する住所を検索し、該当する住所（都道府県情報を含む）を返却する。

## 高校検索

1. クライアントが都道府県IDやキーワードを指定して高校検索を要求する。
2. 都道府県IDが指定されている場合はその都道府県に属する高校に絞り込む。
3. キーワードが指定されている場合は、高校名に部分一致する高校に絞り込む。
4. 該当する高校を名称順に並べ、最大20件まで返却する。

## 学年一覧取得

1. クライアントが高校IDを指定して学年一覧を要求する。
2. 指定した高校に属する学年の一覧を返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::PrefecturesController`|`index`|GET|`/api/v1/prefectures`|都道府県一覧取得|
|`Api::V1::AddressesController`|`index`|GET|`/api/v1/addresses`|住所検索|
|`Api::V1::HighSchoolsController`|`index`|GET|`/api/v1/high_schools`|高校検索|
|`Api::V1::GradesController`|`index`|GET|`/api/v1/high_schools/:high_school_id/grades`|学年一覧取得|

---

# 5. API / 処理詳細

## `Api::V1::PrefecturesController#index`

### 概要

登録されているすべての都道府県を取得する。会員登録フォームや住所検索の選択肢として利用される。

### Request

パラメータなし。

### 処理内容

1. 登録されているすべての都道府県を取得する。
2. 都道府県の一覧を返却する。

### 業務ルール

- 認証不要で、未ログインの状態でも利用できる（会員登録前の利用を想定）。

### Response

- 200 OK
- 都道府県の配列（`id`, `name`）

### Errorケース

確認できない事項（入力パラメータがなく、業務的なエラーは発生しない）。

---

## `Api::V1::AddressesController#index`

### 概要

指定した都道府県に属する住所を、市区町村・町域で絞り込んで検索する。プロフィール編集時の住所選択などに利用される。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|prefecture_id|必須|integer|都道府県ID|
|city|任意|string|市区町村名（部分一致）|
|town|任意|string|町域名（部分一致）|

### 処理内容

1. `prefecture_id` が指定されていない場合、エラーを返却する。
2. `prefecture_id` で住所を絞り込む。
3. `city` が指定されている場合、市区町村名に部分一致する住所に絞り込む。
4. `town` が指定されている場合、町域名に部分一致する住所に絞り込む。
5. 該当する住所（都道府県情報を含む）を返却する。

### 業務ルール

- ログイン中のユーザーのみ利用できる（未ログインの場合は401エラー）。
- `prefecture_id` は必須であり、未指定の場合は400エラーとなる。
- `city` / `town` は空文字の場合、絞り込み条件として扱われない。

### Response

- 200 OK
- 住所の配列（`id`, `postal_code`, `city`, `town`, `prefecture`）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|`prefecture_id` 未指定|400|`都道府県は必須です。`|
|未ログイン|401|認証エラー|

---

## `Api::V1::HighSchoolsController#index`

### 概要

都道府県やキーワードを指定して在籍高校を検索する。会員登録時の高校選択などに利用される。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|prefecture_id|任意|integer|都道府県ID|
|keyword|任意|string|高校名（部分一致）|

### 処理内容

1. `prefecture_id` が指定されている場合、その都道府県に属する高校に絞り込む。
2. `keyword` が指定されている場合、高校名に部分一致する高校に絞り込む。
3. 該当する高校を名称の昇順に並べ、最大20件を返却する。

### 業務ルール

- 認証不要で、未ログインの状態でも利用できる（会員登録前の利用を想定）。
- 検索結果は最大20件までに制限される。

### Response

- 200 OK
- 高校の配列（`id`, `name`, `csv_managed`）
  - `csv_managed` は、学校側が生徒名簿をCSVで管理しており、会員登録時に生徒コードの入力が必要になる高校かどうかを示す。

### Errorケース

確認できない事項（検索条件不一致時は空配列を返す）。

---

## `Api::V1::GradesController#index`

### 概要

指定した高校に存在する学年の一覧を取得する。会員登録時やプロフィール編集時の学年選択に利用される。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|high_school_id|必須|integer|高校ID|

### 処理内容

1. 指定した `high_school_id` に属する学年を取得する。
2. 学年の一覧を返却する。

### 業務ルール

- 認証不要で、未ログインの状態でも利用できる（会員登録前の利用を想定）。

### Response

- 200 OK
- 学年の配列（`id`, `year`, `display_name`）
  - `display_name` は学年を表す日本語表記（例: 高１生、高卒生、新高１生など）。

### Errorケース

確認できない事項（該当する学年がない場合は空配列を返す）。

---

# 6. データモデル

## `prefectures`

- 役割: 都道府県のマスタ情報を保持する。
- 主なカラム: `name`
- リレーション: `has_many :addresses`, `has_many :high_schools`
- 業務ルール: 都道府県名は一意である。

## `addresses`

- 役割: 住所（郵便番号・市区町村・町域・番地）のマスタ情報を保持する。
- 主なカラム: `postal_code`, `city`, `town`, `street_address`, `prefecture_id`
- リレーション: `belongs_to :prefecture`, `has_many :users`

## `high_schools`

- 役割: 在籍高校のマスタ情報を保持する。
- 主なカラム: `name`, `prefecture_id`, `school_code`, `csv_managed`
- リレーション: `belongs_to :prefecture`, `has_many :users`, `has_many :grades`
- 業務ルール: 高校ごとに一意な学校コード（`school_code`）が自動採番される。`csv_managed` が真の高校は、生徒名簿がCSVで管理されており、会員登録時に生徒コードの入力が必要になる。

## `grades`

- 役割: 高校ごとの学年のマスタ情報を保持する。
- 主なカラム: `high_school_id`, `year`
- リレーション: `belongs_to :high_school`
- 業務ルール: 同一高校内で学年（`year`）は重複登録できない。`year` の値に応じて「高１生」「高卒生」「新高１生」などの表示名が決まる。

---

# 7. 状態管理

該当なし（状態を持つマスタではなく、参照専用のデータである）。

---

# 8. 権限制御

- `PrefecturesController#index`、`HighSchoolsController#index`、`GradesController#index` は認証不要であり、未ログインの状態（会員登録前）でも利用できる。
- `AddressesController#index` のみ認証が必須であり、未ログインの場合は401エラーとなる。ログイン中であれば、ロールによらずいずれのユーザーも利用できる。
- いずれのAPIも新規データの作成・更新・削除は行わない参照専用のAPIである。

---

# 9. 非同期処理

確認できない事項（本機能に関連するJob・Mailerは存在しない）。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::PrefecturesController`, `Api::V1::AddressesController`, `Api::V1::HighSchoolsController`, `Api::V1::GradesController`|
|Query|`AddressesQuery`, `HighSchoolsQuery`|
|Model|`Prefecture`, `Address`, `HighSchool`, `Grade`|
|Serializer|`PrefectureSerializer`, `AddressSerializer`, `HighSchoolSerializer`, `GradeSerializer`|

注意:

この情報は現在実装との対応確認用です。

新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
