# 管理者問題インポート機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が CSV 形式の問題データをインポートし、既存問題の上書きまたは追加を行えるようにすること。
- システム上の役割
  - 指定単元に対して CSV ファイルをアップロードし、インポート履歴を記録する。
  - バッチ処理で問題・選択肢・解説・ヒントを作成・更新する。
  - インポートを実行する前に、CSV の内容が正しく取り込めるかどうかを事前検証（ドライラン）する。
  - CSV 入力用のテンプレートファイルをダウンロード提供する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|テンプレートダウンロード|CSV 入力用のテンプレートファイルをダウンロードする|
|事前検証（ドライラン）|アップロード予定の CSV を実際には取り込まずに検証する|
|インポート開始|単元に対して CSV をアップロードし、インポートを開始する|

---

# 3. 業務フロー

## テンプレートダウンロード

1. 管理者が CSV テンプレートのダウンロードを要求する。
2. システムはヘッダー行とサンプル行を含む CSV ファイルを生成して返却する。

## 事前検証（ドライラン）

1. 管理者が単元詳細画面から CSV ファイルを選択し、事前検証を要求する。
2. システムは対象のコースおよび単元を route で厳密にスコープする。
3. CSV ファイルの形式（拡張子・ヘッダー等）を検証する。
4. CSV を 1 行ずつ検証し、データベースへの登録は行わずに、全体件数・有効件数・エラー行の内容を返却する。

## インポート開始

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
|`Api::V1::Admin::CsvTemplatesController`|`questions`|GET|`/api/v1/admin/csv_template/questions`|問題インポート用 CSV テンプレートをダウンロード|
|`Api::V1::Admin::ImportQuestionsController`|`dry_run`|POST|`/api/v1/admin/courses/:course_id/units/:unit_id/import_questions/dry_run`|CSV 問題インポートの事前検証|
|`Api::V1::Admin::ImportQuestionsController`|`create`|POST|`/api/v1/admin/courses/:course_id/units/:unit_id/import_questions`|CSV 問題インポートを開始|

---

# 5. API / 処理詳細

## `Api::V1::Admin::CsvTemplatesController#questions`

### 概要

問題インポートに使用する CSV テンプレートファイルをダウンロードする。

### Request

- なし

### 処理内容

1. ヘッダー行（問題文・正解番号・解説・選択肢A〜D・ヒント1・ヒント2 に相当する項目）とサンプルデータ行から成る CSV ファイルを生成する。
2. 生成した CSV ファイルを添付ファイルとして返却する。

### 業務ルール

- テンプレートには文字化け防止のための BOM を付与する。
- テンプレートの列構成は問題インポートで要求される入力項目と一致する。

### Database変更

- なし

### Response

- `questions_template.csv` という名称の CSV ファイル（`text/csv`）

### Errorケース

- なし（認証前提）

## `Api::V1::Admin::ImportQuestionsController#dry_run`

### 概要

CSV ファイルをデータベースに反映せず、内容が正しく取り込めるかどうかを事前に検証する。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`course_id`|必須|integer|コース ID（URL パラメータ）|
|`unit_id`|必須|integer|単元 ID（URL パラメータ）|
|`file`|必須|file|検証対象の CSV ファイル|

### 処理内容

1. `find_unit!` で `Course.find(params[:course_id]).units.active.find(params[:unit_id])` を実行し、対象単元を route で厳密にスコープする。
2. `Csv::File::FileValidator.new(file).call` でファイル形式の妥当性を検証する。
3. CSV のヘッダーが問題インポートで要求される項目と一致しているかを検証する。ヘッダーが不正な場合はこの時点でエラーとする。
4. CSV を先頭から 1 行ずつ読み込み、行ごとに入力値を検証する。
5. 全行数、検証に合格した行数、検証エラーとなった行の内容（行番号・エラーメッセージ・当該行のデータ）を集計する。
6. データベースへの作成・更新は一切行わず、検証結果のみを返却する。

### 業務ルール

- `course_id` および `unit_id` は route から取得し、リクエストボディの `unit_id` は信用しない。
- 検証エラーの詳細として保持する行数には上限があり、上限を超えた分の行はエラー件数の集計には含まれるが、詳細一覧には表示されない。
- この処理では `ImportHistory` や問題データなど、いかなるデータも作成・更新されない。

### Database変更

- なし

### Response

- `total_count`: CSV の総行数
- `valid_count`: 検証に合格した行数
- `rows`: 検証エラーとなった行の一覧（`row_number`、`severity`、`message`、`data` を含む）

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|ファイル形式不正・ヘッダー不正|422|`errors` を返却|
|対象コース・単元が存在しない|404|対象コース・単元なし|

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

- 管理者は対象コースの単元に対してインポート・事前検証を実行できる。
- route で `course_id` / `unit_id` を厳密にスコープすることで IDOR を防ぐ。
- CSV テンプレートのダウンロードは管理者であれば誰でも実行できる。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::ImportQuestionsController`, `Api::V1::Admin::CsvTemplatesController`|
|Service|`Admin::QuestionCsvBatchImportService`, `Admin::QuestionCsvImportService`, `Admin::QuestionCsvDryRunService`, `Admin::QuestionCsvTemplateService`|
|Form|`Admin::QuestionImportForm`|
|Model|`ImportHistory`, `ImportError`, `Question`, `QuestionChoice`, `QuestionHint`, `QuestionExplanation`|
