# 管理者問題インポート機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が CSV 形式の問題データをインポートし、既存問題の上書きまたは追加を行えるようにすること。
- システム上の役割
  - 指定単元に対して CSV ファイルをアップロードし、インポート履歴を記録する。
  - バッチ処理で問題・選択肢・解説・ヒントを作成・更新する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|インポート開始|単元に対して CSV をアップロードし、インポートを開始する|

---

# 3. 業務フロー

1. 管理者が単元詳細画面から CSV インポートを開始する。
2. システムは対象のコースおよび単元を route で厳密にスコープする。
3. CSV ファイルの妥当性を検証する。
4. `ImportHistory` を作成し、ファイルを添付する。
5. バックグラウンドジョブで `Admin::QuestionCsvBatchImportService` を実行する。
6. インポート結果を `ImportHistory` と `ImportError` に記録する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::ImportQuestionsController`|`create`|POST|`/api/v1/admin/courses/:course_id/units/:unit_id/import_questions`|CSV 問題インポートを開始|

---

# 5. API / 処理詳細

## `Api::V1::Admin::ImportQuestionsController#create`

### 概要

指定単元に対して CSV ファイルをアップロードし、問題インポートを開始する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`course_id`|必須|integer|コース ID|
|`unit_id`|必須|integer|単元 ID|
|`file`|必須|file|CSV ファイル|
|`mode`|任意|string|`append` または `overwrite`（デフォルトは `append`）|

### 処理内容

1. `find_unit!` で `Course.find(params[:course_id]).units.active.find(params[:unit_id])` を実行し、対象単元を route で厳密にスコープする。
2. `Csv::File::FileValidator.new(file).call` でファイルの妥当性を検証する。
3. `current_user.import_histories.create!` で `ImportHistory` を `processing` 状態で作成し、ファイルを添付する。
4. `Admin::QuestionCsvImportJob.perform_later(import_history.id)` を呼び出して非同期処理を開始する。
5. レスポンスとして `202 Accepted` を返却する。

### 業務ルール

- `course_id` および `unit_id` は route から取得し、リクエストボディの `unit_id` は信用しない。
- `mode` が `append` または `overwrite` 以外の場合、`append` として扱う。

### Database変更

- `import_histories` にレコード作成
- 添付ファイルを ActiveStorage で保存

### Response

- `message`: `インポートを開始しました`

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|ファイル不正|422|`errors` を返却|

---

# 6. データモデル

## `import_histories`

- 役割: CSV インポート処理の進捗・結果を記録するテーブル
- 主なカラム: `user_id`, `unit_id`, `status`, `mode`, `file_name`, `file_size`, `content_type`, `started_at`, `finished_at`, `success_count`, `error_count`, `total_count`

## `import_errors`

- 役割: インポート失敗行のエラー情報を記録するテーブル
- 主なカラム: `import_history_id`, `row_number`, `message`

---

# 7. 権限制御

- 管理者は対象コースの単元に対してインポートを実行できる。
- route で `course_id` / `unit_id` を厳密にスコープすることで IDOR を防ぐ。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::ImportQuestionsController`|
|Service|`Admin::QuestionCsvBatchImportService`, `Admin::QuestionCsvImportService`|
|Form|`Admin::QuestionImportForm`|
|Model|`ImportHistory`, `ImportError`, `Question`, `QuestionChoice`, `QuestionHint`, `QuestionExplanation`|
