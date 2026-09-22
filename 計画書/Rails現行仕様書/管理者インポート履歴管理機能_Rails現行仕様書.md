# 管理者インポート履歴管理機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が CSV インポートの実行履歴を検索・確認し、失敗内容を分析できるようにすること。
- システム上の役割
  - インポート履歴を状態・単元・コース・実行者・期間で絞り込み、並び替えして一覧表示する。
  - インポート履歴の詳細として、実行結果とエラー明細を提供する。
  - インポート履歴のエラー明細を CSV ファイルとして出力する。
- 利用者
  - `admin` ロールのユーザー（管理者）

インポート履歴（`import_histories`）は問題インポート・生徒インポートのいずれの実行時にも共通して記録されるデータであるが、本機能（管理者インポート履歴管理機能）が一覧・詳細・CSV エクスポートの対象として扱うのは、管理者が実行する問題インポートの履歴のみである。生徒インポートは教師が担当する別機能であり、その履歴は本機能の対象には含まれない。

---

# 2. 機能一覧

|操作|概要|
|-|-|
|一覧取得|条件で絞り込んだインポート履歴を一覧取得する|
|詳細取得|指定インポート履歴の詳細とエラー明細を取得する|
|CSVエクスポート|指定インポート履歴のエラー明細を CSV ファイルとして出力する|

---

# 3. 業務フロー

1. 管理者がインポート履歴一覧画面を開く。
2. 状態・単元・コース・実行者・期間などの条件を指定すると、システムは条件に合致するインポート履歴を並び替えて一覧取得する。
3. 特定のインポート履歴を選択すると、その履歴の詳細（実行結果の件数、エラー明細）を取得する。
4. エラー内容を確認・共有したい場合は、その履歴のエラー明細を CSV ファイルとしてダウンロードする。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::ImportHistoriesController`|`index`|GET|`/api/v1/admin/import_histories`|インポート履歴一覧を取得|
|`Api::V1::Admin::ImportHistoriesController`|`show`|GET|`/api/v1/admin/import_histories/:id`|インポート履歴詳細を取得|
|`Api::V1::Admin::ImportHistoriesController`|`export`|GET|`/api/v1/admin/import_histories/:id/export`|インポート履歴のエラー明細を CSV 出力|

---

# 5. API / 処理詳細

## `Api::V1::Admin::ImportHistoriesController#index`

### 概要

問題インポートの実行履歴を、条件で絞り込み・並び替えして一覧取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`status`|任意|string|状態による絞り込み（`pending`、`processing`、`completed`、`failed`）|
|`unit_id`|任意|integer|単元 ID による絞り込み|
|`course_id`|任意|integer|コース ID による絞り込み|
|`user_id`|任意|integer|実行した管理者の ID による絞り込み|
|`from`|任意|date|作成日時の期間絞り込み（開始日）|
|`to`|任意|date|作成日時の期間絞り込み（終了日）|
|`sort`|任意|string|並び替え項目（`created_at`、`total_count`、`success_count`、`error_count`、`status` のいずれか。指定外は `created_at` を使用）|
|`order`|任意|string|並び順（`asc` または `desc`。指定外は `desc` を使用）|
|`page`|任意|integer|ページ番号|
|`per_page`|任意|integer|1ページあたり件数|

### 処理内容

1. 論理削除されていない問題インポートの履歴を対象とする。
2. `status` が有効な値であれば状態で絞り込む。
3. `unit_id`、`course_id`（単元の所属コースで判定）、`user_id`（実行者）が指定されていればそれぞれ絞り込む。
4. `from`/`to` が指定されていれば、作成日時がその期間に含まれる履歴のみに絞り込む。
5. `sort`/`order` を適用して並び替える（同一条件の場合は ID の降順で並べる）。
6. ページングした結果を、コース・単元・実行者の情報とともに返却する。

### 業務ルール

- 対象は問題インポートの履歴のみであり、生徒インポートの履歴は一覧に含まれない。
- 論理削除済みの履歴は一覧に含まれない。
- `status`、`sort`、`order` に許可されていない値が指定された場合は、絞り込み条件を無視するか既定値（`created_at` の降順）で並び替える。
- 期間指定は日付単位で行われ、開始日は当日の 0 時、終了日は当日の 23 時 59 分 59 秒までを含む。
- `per_page` は最大 100 件に制限される。

### Database変更

- なし

### Response

- `import_histories`: 各履歴の `id`, `course`（`id`, `level_name`）, `unit`（`id`, `unit_name`）, `user`（`id`, `name`）, `file_name`, `status`, `mode`, `total_count`, `success_count`, `error_count`, `created_at`
- `meta`: `current_page`, `total_pages`, `total_count`, `per_page`

### Errorケース

- なし（認証前提）

## `Api::V1::Admin::ImportHistoriesController#show`

### 概要

指定された問題インポート履歴の詳細と、行ごとのエラー明細を取得する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|インポート履歴の ID|

### 処理内容

1. 指定された問題インポートの履歴を、実行者・エラー明細・単元・コースとともに取得する。
2. 履歴に紐づくエラー明細（行番号順）を整理する。
3. 履歴情報とエラー明細を返却する。

### 業務ルール

- 対象は問題インポートの履歴のみであり、生徒インポートの履歴 ID を指定した場合は取得できない。

### Database変更

- なし

### Response

- `id`, `course`（`id`, `level_name`）, `unit`（`id`, `unit_name`）, `user`（`id`, `name`）, `file_name`, `status`, `mode`, `total_count`, `success_count`, `error_count`, `started_at`, `finished_at`, `created_at`
- `errors`: エラー明細一覧（`row_number`, `message`）
- `warnings`: 警告一覧（現状は常に空）

### Errorケース

- 対象履歴が存在しない|404|対象履歴なし

## `Api::V1::Admin::ImportHistoriesController#export`

### 概要

指定された問題インポート履歴のエラー明細を CSV ファイルとして出力する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`id`|必須|integer|インポート履歴の ID|

### 処理内容

1. 指定された問題インポートの履歴を、エラー明細とともに取得する。
2. 総件数・成功件数・エラー件数のサマリー行と、エラー明細（行番号・状態・メッセージ）を含む CSV データを生成する。
3. CSV ファイルとしてダウンロード返却する。

### 業務ルール

- 対象は問題インポートの履歴のみである。
- CSV はメッセージが `=`、`+`、`-`、`@` で始まる場合、表計算ソフトでの誤動作（数式注入）を防ぐため、先頭に `'` を付与してエスケープする。
- 文字化けを防ぐため CSV には BOM を付与する。

### Database変更

- なし

### Response

- `import_history_<id>.csv` という名称の CSV ファイル（`text/csv`）。1行目はサマリーコメント行、2行目はヘッダー行（`row_number`, `status`, `message`）、以降がエラー明細行。

### Errorケース

- 対象履歴が存在しない|404|対象履歴なし

---

# 6. データモデル

## `import_histories`

- 役割: CSV インポート（問題インポート・生徒インポート）の実行履歴を管理するテーブル
- 主なカラム: `id`, `user_id`, `unit_id`, `file_name`, `file_size`, `content_type`, `status`, `mode`, `import_type`, `total_count`, `success_count`, `error_count`, `started_at`, `finished_at`, `deleted_at`, `created_at`
- リレーション: `belongs_to :user`, `belongs_to :unit`（任意）, `has_many :import_errors`, `has_one_attached :file`
- `import_type` により問題インポート・生徒インポートの区別を持つが、本機能が扱うのは問題インポート（`import_type: question`）の履歴のみである。

## `import_errors`

- 役割: インポート処理で発生した行単位のエラー内容を記録するテーブル
- 主なカラム: `id`, `import_history_id`, `row_number`, `message`
- リレーション: `belongs_to :import_history`

---

# 7. 状態管理

インポート履歴は以下の状態を持つ。

```
pending（実行待ち）
↓
processing（実行中）
↓
completed（完了） または failed（失敗）
```

状態変更条件:

- インポート処理の開始・進行・完了・失敗はインポート実行機能（問題インポート機能）側の非同期処理によって更新される。
- 本機能（インポート履歴管理機能）は履歴の状態を参照するのみであり、状態を変更する操作は提供しない。

---

# 8. 権限制御

- 管理者はすべての問題インポート履歴を参照・出力可能。
- 生徒インポートの履歴は本機能の対象外であり、参照できない。

---

# 9. 非同期処理

- なし（インポート処理自体の非同期実行は問題インポート機能側で行われ、本機能はその結果を参照する）

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::ImportHistoriesController`|
|Query|`Admin::ImportHistoriesQuery`|
|Service|`Admin::ImportHistoryCsvExporterService`|
|Serializer|`Admin::ImportHistoryListSerializer`, `Admin::ImportHistoryDetailSerializer`|
|Model|`ImportHistory`, `ImportError`|

注意:

この情報は現在実装との対応確認用です。

新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
