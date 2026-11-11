# 管理者問題インポート機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が指定単元（course_id / unit_id で route スコープされる）に対して CSV 形式の問題データをアップロードし、既存問題への追加（append）または上書き（overwrite）を行う機能である。アップロード自体は同期的に受け付け（202 Accepted）、実際の問題・選択肢・解説・ヒントの作成/更新は非同期処理で実行される。処理結果はインポート履歴（ImportHistory）と行単位のエラー情報（ImportError）として記録される。

本機能はさらに、本番アップロードに先立って CSV の内容をデータベースへ一切反映せずに検証する事前検証（dry_run）と、CSV 入力用テンプレートファイルのダウンロードを提供する（②「1. 機能概要」）。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により **Domain Model** を採用する。

- ImportHistory は「processing → 成功 / 一部失敗 / 失敗」という業務上意味のある状態遷移を持つ
- append/overwrite の判定ロジックは単純なデータ保存以上の業務ルールである
- 行単位の処理結果を集計して最終状態を決定するロジックは複数の処理結果を集約する業務ルールであり、Entity・Domain Serviceとして意味を持たせる価値が高い
- CSV行の形式検証は事前検証（dry_run）と本番実行の両方から同一のルールで再利用される必要があり、Domain Serviceとして1箇所に集約する価値が高い

Transaction Script（ルールが散在し保守性が低下する）、Active Record（状態遷移・集計ロジックによりモデルが肥大化する）、Event Sourcing（行単位イベントの永続化・再生要件が現時点で存在せず過剰設計）はいずれも②で不採用と判断されている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- 対象 Bounded Context: `question-import`
- ②「12. UseCase設計」の StartQuestionImportUseCase（CSVアップロード受付）、ExecuteQuestionImportUseCase（CSV行処理実行）、ValidateQuestionImportUseCase（事前検証／dry_run）、DownloadQuestionCsvTemplateUseCase（CSVテンプレート配布）の4UseCase
- ②「19. API仕様」記載の3エンドポイント（テンプレートダウンロード、事前検証、インポート開始）
- 非同期実行の基盤は、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」が定める `jobs` テーブル + ポーリングワーカーの標準実装に従う。旧版の本書が「具体的なメッセージング基盤の選定は本書のスコープ外とする」としていた記述は、同規約の制定（2026-08）に伴い標準実装へ置き換える
- Question Context（Question / QuestionChoice / QuestionHint / QuestionExplanation の作成・更新）は②の設計判断どおり本 Context のスコープ外であり、QuestionRepository を外部依存として参照するに留める。Question Context 自体の詳細設計（Entity構造・メソッドシグネチャ等）は、対応する②文書が存在しないため本書では定義しない
- Railsの実装詳細（①）は本タスクで提供されておらず、参照が必要な箇所は「①未提供のため参照不可」として扱う

---

# 2. ディレクトリ構成

- 対象 Bounded Context名: `question-import`（`internal/` 配下のディレクトリ名は `question_import` とする。アーキテクチャ規約「8. 命名規約」に基づき、Context名の kebab-case とディレクトリ名のスネークケースの対応をここに明記する）
- ②で採用した設計パターン: Domain Model
- アーキテクチャ規約「3. 設計パターンごとの構造適用方針」の Domain Model 構造（標準フルレイヤー構成）をそのまま適用する
- 本機能で `specification/`・`mail/`・`cache/` は対象外とする（②に該当する業務ルールの記載がないため。理由は本書「5. Infrastructure層設計」に後述）

## 作成するディレクトリ一覧

```
internal/question_import/
├── domain/
│   ├── entity/
│   ├── valueobject/
│   ├── repository/
│   ├── service/
│   ├── event/
│   └── errors/
├── application/
│   ├── dto/
│   └── usecase/
├── infrastructure/
│   ├── persistence/
│   │   └── gorm/
│   ├── repository/
│   └── queue/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

## 作成するファイル一覧

```
internal/question_import/domain/entity/import_history.go
internal/question_import/domain/entity/import_error.go
internal/question_import/domain/valueobject/import_mode.go
internal/question_import/domain/valueobject/import_status.go
internal/question_import/domain/valueobject/import_row_result.go
internal/question_import/domain/repository/import_history_repository.go
internal/question_import/domain/repository/import_error_repository.go
internal/question_import/domain/repository/course_repository.go
internal/question_import/domain/repository/unit_repository.go
internal/question_import/domain/repository/question_repository.go
internal/question_import/domain/service/question_csv_row_validator.go
internal/question_import/domain/service/question_import_policy.go
internal/question_import/domain/service/import_result_aggregation_policy.go
internal/question_import/domain/event/question_import_requested.go
internal/question_import/domain/errors/errors.go
internal/question_import/application/dto/start_question_import.go
internal/question_import/application/dto/execute_question_import.go
internal/question_import/application/dto/validate_question_import.go
internal/question_import/application/dto/download_question_csv_template.go
internal/question_import/application/usecase/start_question_import_usecase.go
internal/question_import/application/usecase/execute_question_import_usecase.go
internal/question_import/application/usecase/validate_question_import_usecase.go
internal/question_import/application/usecase/download_question_csv_template_usecase.go
internal/question_import/infrastructure/persistence/gorm/import_history_model.go
internal/question_import/infrastructure/persistence/gorm/import_error_model.go
internal/question_import/infrastructure/repository/import_history_repository.go
internal/question_import/infrastructure/repository/import_error_repository.go
internal/question_import/infrastructure/repository/course_repository.go
internal/question_import/infrastructure/repository/unit_repository.go
internal/question_import/infrastructure/queue/question_import_job_handler.go
internal/question_import/presentation/handler/question_import_handler.go
internal/question_import/presentation/request/start_question_import_request.go
internal/question_import/presentation/request/validate_question_import_request.go
internal/question_import/presentation/response/start_question_import_response.go
internal/question_import/presentation/response/validate_question_import_response.go
internal/question_import/presentation/routes.go
```

**②からの補足**: QuestionRepository の実装（Question Context が提供するもの）は question-import Context の外側にあるため、本書では Interface 定義のみを `domain/repository/question_repository.go` に置き、実装ファイルは一覧に含めない。実装は Question Context 側の③文書のスコープとする。CSVテンプレートのダウンロード（`DownloadQuestionCsvTemplateUseCase`）はDBアクセスを伴わない静的処理のため、対応するInfrastructure層のファイルは作成しない（②「12. UseCase設計」の「呼び出すRepository: なし」に対応）。非同期実行の基盤ファイルは、旧版の `question_import_event_publisher.go` / `question_import_worker.go` という独自実装から、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の標準実装（`JobPublisher` Interfaceと`jobs`テーブルポーリングワーカー）へ置き換え、本Context固有で保持するのは`infrastructure/queue/question_import_job_handler.go`（`jobs`テーブルのポーリングワーカーへ登録するジョブハンドラ本体）のみとする。`JobPublisher` Interface・`jobs`テーブルGORMモデル・ポーリングワーカー本体は同規約が定める共通基盤であり、本Context固有のファイルとしては作成しない。

---

# 3. Domain層設計

## Entity

### ImportHistory

- struct名: `ImportHistory`
- 保持するフィールドと型:
  - `id`: `string`（またはUint、GORM規約に従い主キー名は `ID`。値の型は他Contextの既存ID型に合わせるため、実装時に既存の共通ID型を確認する — ②に型の明記はなくコード実装時の判断が必要）
  - `courseID`: `string`
  - `unitID`: `string`
  - `userID`: `string`（実行した管理者のID。②「16. Authorization設計」の「ImportHistoryの所有者（user_id）を保持」に対応）
  - `mode`: `valueobject.ImportMode`
  - `status`: `valueobject.ImportStatus`
  - `fileName`: `string`
  - `fileReference`: `string`（ファイル保存先を表す情報。②「20. DB設計方針」で追加が提案されているカラムに対応。詳細は本書「12. GORM/DBクエリ設計」参照）
  - `totalCount`: `int`
  - `successCount`: `int`
  - `errorCount`: `int`
  - `startedAt`: `time.Time`
  - `completedAt`: `*time.Time`
  - `errors`: `[]ImportError`（Aggregate内の子Entity。②「5. Aggregate設計」の「ImportErrorはImportHistoryに従属し、単独では存在しない」に対応）
- 各フィールドの意味: 上記のとおり。`mode`/`status` は業務ルールを型に閉じ込めた Value Object であり、生の文字列を保持しない
- 公開する method 一覧:
  - `NewImportHistory(courseID, unitID, userID string, mode valueobject.ImportMode, fileName, fileReference string) (*ImportHistory, error)`: 新規作成用ファクトリ。生成直後は `status` を processing に固定する
  - `Complete(status valueobject.ImportStatus, successCount, errorCount, totalCount int, completedAt time.Time) error`: 行単位処理結果の集計（Domain Service `ImportResultAggregationPolicy` の出力）を受け取り、終了状態へ遷移させる。processing以外からの再遷移はエラーとする（②「7. Value Object設計」の「終了後の再遷移は許容しない」に対応）
  - `ID() string` / `CourseID() string` / `UnitID() string` / `UserID() string` / `Mode() valueobject.ImportMode` / `Status() valueobject.ImportStatus` / `Counts() (total, success, error int)`: 参照用アクセサ
- 不変条件（コンストラクタで保証する内容）:
  - courseID / unitID / userID は空文字を許容しない
  - 生成直後の `status` は必ず processing である
  - `totalCount` / `successCount` / `errorCount` は生成時は 0 とする

### ImportError

- struct名: `ImportError`
- 保持するフィールドと型:
  - `id`: `string`
  - `importHistoryID`: `string`
  - `rowNumber`: `int`
  - `message`: `string`
- 各フィールドの意味: `rowNumber` は失敗したCSV行番号、`message` は失敗理由
- 公開する method 一覧:
  - `NewImportError(importHistoryID string, rowNumber int, message string) (*ImportError, error)`: 生成時に `rowNumber > 0` かつ `message` が空でないことを検証する
  - `RowNumber() int` / `Message() string`: 参照用アクセサ
- 不変条件: `rowNumber` は1以上、`message` は必須。作成後の更新・削除は行わない（②「6. Entity設計」の記載どおり）

### Question / QuestionChoice / QuestionHint / QuestionExplanation（外部参照）

- ②「6. Entity設計」のとおり、question-import Context の Aggregate には含めない。QuestionRepository Interface（本書「3. Repository Interface」参照）を通じて「作成・更新を依頼する対象」として扱う
- 具体的な struct定義・フィールドは Question Context の②文書がまだ存在しないため（②「11. Repository設計」の QuestionRepository の記載を参照）、本書では定義しない。**②からの補足**: これは①未提供のためではなく、Question Context 自体の②文書が存在しないことに起因する

## Value Object

### ImportMode

- struct名: `ImportMode`
- 保持するフィールドと型: `value string`（非公開。`append` / `overwrite` のいずれかに正規化済みの値のみを保持する）
- 生成時に検証するルール: `NewImportMode(raw string) ImportMode` は `raw` が `append` / `overwrite` 以外の場合、②「7. Value Object設計」の記載どおり `append` として正規化する（エラーを返さない）
- 公開する method 一覧: `IsAppend() bool` / `IsOverwrite() bool` / `String() string`

### ImportStatus

- struct名: `ImportStatus`
- 保持するフィールドと型: `value string`
- 生成時に検証するルール:
  - `NewProcessingStatus() ImportStatus`: 常に processing 状態を返す
  - 終了状態（成功／一部失敗／失敗）は `ImportResultAggregationPolicy`（本書「3. Domain Service」参照）の出力からのみ生成される
- 公開する method 一覧: `IsTerminal() bool` / `IsProcessing() bool` / `String() string`

**②からの補足**: ②「6. Entity設計」では終了状態を「成功／一部失敗／失敗」と業務要件から推測した区分として記載しており、正式な英語表現・内部値までは定義していない。本書では実装のため `StatusProcessing` / `StatusSuccess` / `StatusPartialFailure` / `StatusFailure` という値を仮に置く。これは②に根拠のない値の具体化であり「推測」である。

### ImportRowResult

- struct名: `ImportRowResult`
- 保持するフィールドと型:
  - `rowNumber`: `int`
  - `success`: `bool`
  - `questionID`: `*string`（成功時のQuestion識別情報。本番実行のみ設定される）
  - `errorMessage`: `string`（失敗時のエラーメッセージ）
- 生成時に検証するルール: 成功時は `questionID` が必須（事前検証時は検証合格の事実のみを表し `questionID` は設定しない）、失敗時は `errorMessage` が必須（②「7. Value Object設計」の記載どおり。「本番実行だけでなく、事前検証（dry_run）でも同じ型を用いることで、両者の結果表現を統一する」）
- 公開する method 一覧: `NewSuccessRowResult(rowNumber int, questionID *string) ImportRowResult` / `NewFailureRowResult(rowNumber int, message string) ImportRowResult` / `IsSuccess() bool` / `RowNumber() int`

**②からの補足**: `questionID` を `*string`（ポインタ）とした。事前検証時は Question が実際に作成されないため `questionID` を持たない成功結果が必要であり、②「7. Value Object設計」の「独自ルール: 成功時はQuestion識別情報（本番実行時）または検証合格の事実（事前検証時）を...保持する」という記載を型として具体化する必要があったための実装判断（推測）である。

## Repository Interface

### ImportHistoryRepository

- interface名: `ImportHistoryRepository`
- メソッドシグネチャ一覧:
  - `Create(ctx context.Context, history *entity.ImportHistory) error`
  - `Update(ctx context.Context, history *entity.ImportHistory) error`: status・カウント・completedAt の更新に用いる
  - `FindByID(ctx context.Context, id string) (*entity.ImportHistory, error)`
  - `FindRecentByUserID(ctx context.Context, userID string, limit int) ([]*entity.ImportHistory, error)`: user_id 絞り込み・作成日時降順（②「11. Repository設計」の記載どおり）
- 各メソッドの責務: 永続化・検索に限定し、モード判定・状態遷移ルールの判断は持たない（②の記載どおり）。②「11. Repository設計」が言及する検索条件（状態・単元・コース・実行者・期間）による一覧取得・並び替え・ページングは、管理者インポート履歴管理機能_Go実装仕様書が本 interface に追加するメソッドとして定義する（本書では ImportHistory/ImportError Aggregate と基本 CRUD の定義のみを扱う）

### ImportErrorRepository

- interface名: `ImportErrorRepository`
- メソッドシグネチャ一覧:
  - `CreateBatch(ctx context.Context, errs []*entity.ImportError) error`: 行単位エラーの一括作成
  - `FindByImportHistoryID(ctx context.Context, importHistoryID string) ([]*entity.ImportError, error)`
- 各メソッドの責務: 永続化に特化し、エラー内容の妥当性判断は持たない（②の記載どおり）

### CourseRepository

- interface名: `CourseRepository`
- メソッドシグネチャ一覧:
  - `Exists(ctx context.Context, courseID string) (bool, error)`
- 各メソッドの責務: course_id のスコープ検証（IDOR防止）に必要な存在確認のみ。Course自体の作成・更新は持たない（②の記載どおり）

### UnitRepository

- interface名: `UnitRepository`
- メソッドシグネチャ一覧:
  - `FindActiveUnitInCourse(ctx context.Context, courseID, unitID string) (*UnitRef, error)`: course_id配下にunit_idが存在し、かつactiveであることを1回で確認する（②「11. Repository設計」の「course_id配下にunit_idが存在し、かつactiveであることの確認」に対応）
- 各メソッドの責務: routeスコープ検証に必要な参照確認に限定し、Unit自体の作成・更新は持たない
- **②からの補足**: `UnitRef` は確認結果を表す最小限のデータ（unit_idの存在・active可否）を返すための型。②に戻り値の型までは明記がないため、実装上の必要から補ったものであり「推測」である

### QuestionRepository（Question Context提供、外部依存として利用）

- interface名: `QuestionRepository`
- メソッドシグネチャ一覧:
  - `Create(ctx context.Context, row QuestionCSVRow) (questionID string, err error)`: 新規作成
  - `Update(ctx context.Context, questionID string, row QuestionCSVRow) error`: 既存問題の更新（overwrite時）
- 各メソッドの責務: 問題データの永続化。CSV解析・モード判定は持たない（②の記載どおり）。事前検証（`ValidateQuestionImportUseCase`）はこの Repository を呼び出さない（②「12. UseCase設計」の「ImportHistoryRepository・QuestionRepositoryは呼び出さない」に対応）
- **②からの補足**: `QuestionCSVRow`（CSV1行分のデータを表す型）の具体的なフィールド構成（問題文・選択肢・ヒント・解説等）は、①（Rails実装）未提供のため参照不可であり、かつ Question Context自体の②文書も存在しないため、本書では詳細フィールドを定義しない。question-import Context 側では「CSV1行分のデータを保持する型」として抽象的に扱い、フィールド定義は Question Context 側の設計時に確定させる前提とする

## Domain Service

### QuestionCsvRowValidator

- struct/interface名: `QuestionCsvRowValidator`（struct、ステートレス）
- メソッドシグネチャ: `Validate(row QuestionCSVRow) valueobject.ImportRowResult`
- 責務: CSV1行分のデータについて、必須列の充足・データ形式（正解番号の範囲、選択肢の必須数等）の構造的妥当性を検証する（②「8. Domain Service」の記載どおり）
- 判断根拠: 事前検証（`ValidateQuestionImportUseCase`）と本番実行（`ExecuteQuestionImportUseCase`）の両方から呼び出し、判定結果の一貫性を保証する共通コンポーネントとする。具体的な検証項目（必須列の名称・正解番号の許容範囲等）は①未提供かつQuestion Context自体の②文書が存在しないため、本書では定義しない

### QuestionImportPolicy

- struct/interface名: `QuestionImportPolicy`（struct、ステートレス）
- メソッドシグネチャ: `Decide(row QuestionCSVRow, mode valueobject.ImportMode, existingQuestionID *string) ImportAction`
  - `ImportAction` は「新規作成すべきか」「既存Questionを更新すべきか」を表す値（②「8. Domain Service」の記載に対応する型。具体的な列挙値・判定条件は①未提供のため参照不可であり、③でも詳細ロジックは記述しない）
- 責務: CSV1行のデータとmode（append/overwrite）から、Questionを新規作成すべきか既存Questionを更新すべきかを判定する（②の記載どおり）。`QuestionCsvRowValidator` による形式検証に合格した行のみを対象とし、事前検証（dry_run）では呼び出されない（②「8. Domain Service」の記載どおり）

### ImportResultAggregationPolicy

- struct/interface名: `ImportResultAggregationPolicy`（struct、ステートレス）
- メソッドシグネチャ: `Aggregate(results []valueobject.ImportRowResult) (status valueobject.ImportStatus, successCount, errorCount, totalCount int)`
- 責務: 行単位の処理結果（ImportRowResultの集合）から、ImportHistoryの最終的な状態（成功 / 一部失敗 / 失敗）と各種カウントを決定する（②の記載どおり）。事前検証（dry_run）では、本メソッドのうち件数集計部分（total_count / valid_count相当）のみを再利用し、ImportHistoryの状態決定には用いない（②「8. Domain Service」の記載どおり。件数集計のみを取り出す具体的な関数分割方法は②に明記がなく「推測」であり、本書では`Aggregate`の戻り値のうちstatusを無視して呼び出す運用、または別途`CountValid(results []valueobject.ImportRowResult) (total, valid int)`のような補助メソッドを設けるかは実装時の判断に委ねる）

## Domain Event

### QuestionImportRequested

- イベントstruct名: `QuestionImportRequested`
- 保持するフィールド: `ImportHistoryID string` / `CourseID string` / `UnitID string` / `UserID string` / `OccurredAt time.Time`
- 発火元: `StartQuestionImportUseCase`（ImportHistoryがprocessing状態で作成された直後。②「18. Domain Event」の記載どおり）
- 実行基盤: 本イベントの実際の非同期ディスパッチは、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」が定める`JobPublisher`（Application層）+ `jobs`テーブル（Infrastructure層）+ ポーリングワーカーの標準実装を用いる。`QuestionImportRequested`のフィールドは、`JobPublisher.Publish`に渡すjobペイロード（JSON化される）の内容として使用する。詳細は本書「5. Infrastructure層設計」「8. Transaction実装方針」参照

## Domain Error

- エラー種別ごとの型／変数定義方針: `domain/errors` パッケージに、区別可能なエラー変数（`errors.New` ベース）またはエラー型（`fmt.Errorf` + `%w` でラップ可能な型）を定義する。コーディング規約「18. エラーハンドリング」に従い、`errors.New` と `fmt.Errorf` の使い分けについて本書で独自ルールは設けない
- 発生条件（②「17. Error設計」より）:
  - `ErrInvalidCSVRow`: 不正なCSV行データ（必須列欠如、選択肢不整合等）。`QuestionCsvRowValidator.Validate`が失敗した場合に、本番実行時は`ImportError`として記録され、事前検証時は`ImportRowResult`の失敗結果としてレスポンスに含める（②「17. Error設計」の「CSV行データが業務ルールに違反する（事前検証時）: Domain Error（検証結果として返却、エラー自体は発生させない）」に対応。事前検証時はGoの`error`値としては発生させず、`ImportRowResult.IsSuccess() == false`という戻り値で表現する）
  - `ErrInvalidStatusTransition`: 不正な状態遷移（processing以外からの再遷移等）
  - `ErrCountMismatch`: ImportHistoryの状態不整合（success+error != total 等）

**②からの補足**: CSV行データの具体的な「必須列」の定義は①未提供のため参照不可であり、本書では列名までは列挙しない。列定義はQuestion Context側の設計、またはCSVフォーマット仕様の確定を待って別途整理する必要がある。

---

# 4. Application層設計

## DTO（Command / Query）

- struct名: `StartQuestionImportCommand`
  - フィールド: `CourseID string` / `UnitID string` / `AdminUserID string` / `File io.Reader` / `FileName string` / `RawMode string`
  - 区分: Command
- struct名: `StartQuestionImportResult`
  - フィールド: `ImportHistoryID string` / `Status string`
  - 区分: UseCase出力（Query相当の戻り値）
- struct名: `ExecuteQuestionImportCommand`
  - フィールド: `ImportHistoryID string`
  - 区分: Command
- struct名: `ExecuteQuestionImportResult`
  - フィールド: `ImportHistoryID string` / `Status string` / `SuccessCount int` / `ErrorCount int` / `TotalCount int`
  - 区分: UseCase出力
- struct名: `ValidateQuestionImportCommand`
  - フィールド: `CourseID string` / `UnitID string` / `AdminUserID string` / `File io.Reader` / `FileName string`
  - 区分: Command（②「12. UseCase設計」の「入力: current admin, course_id, unit_id（route由来）, file」に対応。modeは事前検証の入力に含まれない）
- struct名: `ValidateQuestionImportResult`
  - フィールド: `TotalCount int` / `ValidCount int` / `Rows []InvalidRowResult`
  - 区分: UseCase出力（②「19. API仕様」の「total_count、valid_count、rows」に対応）
- struct名: `InvalidRowResult`
  - フィールド: `RowNumber int` / `Message string`
  - 区分: UseCase出力（検証エラー行の明細。②「12. UseCase設計」の「検証エラー行一覧（row_number・エラーメッセージ・当該行データ。上限件数を超えた分は集計のみに反映し詳細一覧には含めない）」に対応。上限件数の具体的な値は②に明記がなく「推測」であり、本書では`application`パッケージ内の定数として定義する方針とする）
- struct名: `DownloadQuestionCsvTemplateResult`
  - フィールド: `FileName string` / `Content []byte`
  - 区分: UseCase出力（②「12. UseCase設計」の「出力: CSVファイル（BOM付き）」に対応）

## UseCase

### StartQuestionImportUseCase

- struct名: `StartQuestionImportUseCase`
- コンストラクタが受け取る依存: `CourseRepository` / `UnitRepository` / `ImportHistoryRepository` / `TransactionManager`（アーキテクチャ規約「11. Transaction実装パターン」） / `JobPublisher`（アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.StartQuestionImportCommand) (dto.StartQuestionImportResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `CourseRepository.Exists` で course_id の存在を確認する
  2. `UnitRepository.FindActiveUnitInCourse` で course_id 配下に unit_id が存在し active であることを確認する（②「16. Authorization設計」の IDOR対策に対応）
  3. `valueobject.NewImportMode(cmd.RawMode)` でモードを正規化する
  4. ファイルの保存（ファイル参照情報の生成。保存先の実装詳細は本書「5. Infrastructure層設計」参照）
  5. `entity.NewImportHistory(...)` で processing 状態のEntityを生成する
  6. `TransactionManager.WithinTransaction`のクロージャ内で、`ImportHistoryRepository.Create`による永続化と、`JobPublisher.Publish(ctx, "question_import", payload, time.Now())`によるジョブ登録を同一トランザクションで実行する（アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」のTransactional Outboxパターン。詳細は本書「8. Transaction実装方針」）
  7. `dto.StartQuestionImportResult` に変換して返す
- トランザクション境界: ②「14. Transaction設計」の「ImportHistoryの作成（processing状態）とファイル情報の保存を1トランザクションで実施する」を実装単位に落とし込み、ステップ1〜3を事前確認としてトランザクション外で実行し、ステップ6（ImportHistory作成 + ジョブ登録）を1トランザクションとする
- 発生しうる Application Error: `ErrCourseNotFound` / `ErrUnitNotFound` / `ErrUnitNotActive` / `ErrInvalidFileFormat`

**②からの補足**: 旧版の本書では「イベント発行をトランザクションコミット後に行う」という構成を推測として記載していたが、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」制定（2026-08）により、確実に実行したい処理（CSVインポート処理）はTransactional Outboxパターン（業務データの書き込みと同一トランザクション内でjobを登録する）に従うことが標準化されたため、本書もこれに合わせてジョブ登録をトランザクション内で行う構成へ修正した。これは②の設計判断（ImportHistoryの整合性重視）と矛盾せず、むしろ「ジョブは登録されたが業務データが保存されない」不整合を防ぐ点でより整合的である。

### ExecuteQuestionImportUseCase

- struct名: `ExecuteQuestionImportUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository` / `ImportErrorRepository` / `QuestionRepository` / `service.QuestionCsvRowValidator` / `service.QuestionImportPolicy` / `service.ImportResultAggregationPolicy`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.ExecuteQuestionImportCommand) (dto.ExecuteQuestionImportResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `ImportHistoryRepository.FindByID` で対象の ImportHistory を取得する
  2. 対象ファイルを読み込み、CSV行に分解する（パース処理の詳細は①未提供のため参照不可）
  3. 各行について `QuestionCsvRowValidator.Validate` で形式検証を行う
  4. 検証に合格した行のみ、`QuestionImportPolicy.Decide` でアクション（新規作成/更新）を判定する
  5. 判定結果に応じて `QuestionRepository.Create` または `QuestionRepository.Update` を呼び出す（行またはバッチ単位でトランザクション化。詳細は本書「8. Transaction実装方針」）
  6. 各行の結果を `valueobject.ImportRowResult` として収集する（検証不合格の行は`ErrInvalidCSVRow`起因の失敗結果として収集する）
  7. 全行処理後、`ImportResultAggregationPolicy.Aggregate` で最終状態とカウントを決定する
  8. `ImportHistory.Complete(...)` で Entity を終了状態へ遷移させる
  9. `ImportHistoryRepository.Update` で最終状態を永続化する（ステップ5とは別トランザクション）
  10. 失敗行があれば `ImportErrorRepository.CreateBatch` で一括保存する
  11. `dto.ExecuteQuestionImportResult` に変換して返す
- トランザクション境界: ②「14. Transaction設計」の記載どおり、行（またはバッチ）単位でQuestion作成/更新をトランザクション化し、全行処理完了後にImportHistoryの最終状態を別途コミットする。UseCase全体を1トランザクションにはしない（部分的成功を許容するための意図的な逸脱であることは②に明記済み）
- 発生しうる Application Error: `ErrImportHistoryNotFound` / `ErrInvalidStatusTransition`

**②からの補足**: バッチ処理の粒度（1行ごとか、複数行をまとめたバッチ単位か）や具体的なバッチサイズは②に明記がなく、「行（またはバッチ）単位」という選択の余地がある表現に留まっている。本書でも具体的な数値・粒度は確定させず、実装時に決定可能なパラメータとして扱う（推測ではなく②の未確定事項をそのまま実装仕様に引き継ぐ扱いとする）。

### ValidateQuestionImportUseCase（事前検証／dry_run）

- struct名: `ValidateQuestionImportUseCase`
- コンストラクタが受け取る依存: `CourseRepository` / `UnitRepository` / `service.QuestionCsvRowValidator`（②「12. UseCase設計」の「呼び出すRepository: CourseRepository, UnitRepository...ImportHistoryRepository・QuestionRepositoryは呼び出さない」に対応し、ImportHistoryRepository・QuestionRepositoryは依存に含めない）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.ValidateQuestionImportCommand) (dto.ValidateQuestionImportResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `CourseRepository.Exists` で course_id の存在を確認する
  2. `UnitRepository.FindActiveUnitInCourse` で course_id 配下に unit_id が存在し active であることを確認する（本番実行と同一のスコープ検証。②「16. Authorization設計」の「事前検証（dry_run）も本番実行と同じスコープ検証を行う」に対応）
  3. 対象ファイルを読み込み、CSV行に分解する
  4. 各行について `QuestionCsvRowValidator.Validate` を呼び出し、`ImportRowResult` を収集する（`ExecuteQuestionImportUseCase`と同一のDomain Serviceを用いることで判定結果の一貫性を保証する。②「12. UseCase設計」の判断根拠に対応）
  5. 収集した`ImportRowResult`から総行数（total_count）と検証合格行数（valid_count）を集計する
  6. 検証不合格の行を`InvalidRowResult`（row_number・message）に変換する。上限件数を超えた分は`Rows`に含めず、`TotalCount`/`ValidCount`の集計のみへ反映する（②「12. UseCase設計」の記載どおり）
  7. `dto.ValidateQuestionImportResult`として返す。ImportHistory・Question等いかなるデータも作成・更新しない
- トランザクション境界: 使用しない（②「14. Transaction設計」の「事前検証（ValidateQuestionImportUseCase）はデータ変更を一切行わないため、そもそもトランザクションの対象にならない」に対応）
- 発生しうる Application Error: `ErrCourseNotFound` / `ErrUnitNotFound` / `ErrUnitNotActive` / `ErrInvalidFileFormat`（CSVヘッダー不正等、行単位ではなくファイル全体の構造不正はDomain Errorとして扱い、Application Errorへ変換する。詳細は本書「11. Error実装方針」）

### DownloadQuestionCsvTemplateUseCase（CSVテンプレート配布）

- struct名: `DownloadQuestionCsvTemplateUseCase`
- コンストラクタが受け取る依存: なし（②「12. UseCase設計」の「呼び出すRepository: なし」に対応）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context) (dto.DownloadQuestionCsvTemplateResult, error)`
- 処理ステップ（呼び出し順序）:
  1. 固定のヘッダー行＋サンプル行から構成されるCSVコンテンツ（BOM付き）を生成する（②「12. UseCase設計」の「テンプレートの列構成はQuestionCsvRowValidatorが要求する入力項目に対応する固定的な内容」に対応。具体的な列名は①未提供かつQuestion Context側の②文書が存在しないため、本書では列挙しない）
  2. `dto.DownloadQuestionCsvTemplateResult`として返す
- トランザクション境界: 使用しない（DBアクセスを伴わない静的なファイル生成処理。②の記載どおり）
- 発生しうる Application Error: なし（②「12. UseCase設計」の記載どおり、DB参照を必要としない単純な生成処理であり、業務エラーが発生する余地がない）

---

# 5. Infrastructure層設計

## Repository実装

### ImportHistoryRepository実装

- 実装struct名: `ImportHistoryRepository`（package `gormrepo`。アーキテクチャ規約「8. 命名規約」に従い `〇〇RepositoryImpl` のような接尾辞は付けない）
- 対応するGORMモデル: `gormmodel.ImportHistoryModel`（テーブル `import_histories`）
- 各メソッドで発行するクエリ内容:
  - `Create`: `import_histories` への1件INSERT。`TransactionManager`経由でトランザクション用の`*gorm.DB`を取得して実行する（アーキテクチャ規約「11. Transaction実装パターン」の`dbFromContext`）
  - `Update`: id指定でstatus・success_count・error_count・total_count・completed_atを更新するUPDATE
  - `FindByID`: idの一致条件によるSELECT（1件）
  - `FindRecentByUserID`: user_idの一致条件、created_at降順ソート、limit指定によるSELECT
- Entity ⇔ GORMモデルの変換方針: `gormmodel.ImportHistoryModel` から `entity.ImportHistory` へは非公開の変換関数（例: `toEntity`）で行い、Value Object（ImportMode/ImportStatus）への変換もこの関数内で行う。逆方向（Entity → GORMモデル）も同様に非公開の変換関数で行う

### ImportErrorRepository実装

- 実装struct名: `ImportErrorRepository`（package `gormrepo`）
- 対応するGORMモデル: `gormmodel.ImportErrorModel`（テーブル `import_errors`）
- 各メソッドで発行するクエリ内容:
  - `CreateBatch`: `import_errors` への複数件一括INSERT
  - `FindByImportHistoryID`: import_history_idの一致条件によるSELECT（複数件、row_number昇順を推測。②に明示的なソート順の記載はないため「推測」）
- Entity ⇔ GORMモデルの変換方針: ImportHistoryRepositoryと同様、非公開の変換関数で行う

### CourseRepository / UnitRepository実装

- 実装struct名: `CourseRepository` / `UnitRepository`（package `gormrepo`）
- 対応するGORMモデル: School Context側で定義済みの既存 `Course` / `Unit` モデルを参照専用で利用する（本Contextでは新規に定義しない）
- 各メソッドで発行するクエリ内容:
  - `CourseRepository.Exists`: course_idの一致条件による存在確認（COUNT または LIMIT 1 相当）
  - `UnitRepository.FindActiveUnitInCourse`: course_id と unit_id の一致条件、かつ active フラグ条件によるSELECT（1件）

**②からの補足**: Course/Unitモデルの具体的なフィールド構成・GORMタグは curriculum Context（School領域）側の②③文書のスコープであり、本書では既存モデルを参照する前提のみを記載する。

## 外部連携実装

- Mail: 対象外（②に記載なし）
- Cache: 対象外（②に記載なし）
- Queue: 対象（②「18. Domain Event」「24. 採用しなかった設計」を踏まえた実装対象）

### Queue実装（非同期実行トリガー、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」準拠）

本機能はアーキテクチャ規約が定める「確実に実行したい処理」（プロセス再起動後も再実行が必要、リトライが必要）に該当するため、同規約の標準実装（`jobs`テーブル + ポーリングワーカー）を採用する。

- 依存する共通基盤（本Context固有の実装ではなく、アーキテクチャ規約13章が定める共通コンポーネントを利用する）:
  - `JobPublisher` Interface（Application層、共通配置）: `Publish(ctx context.Context, jobType string, payload any, runAfter time.Time) error`。`StartQuestionImportUseCase`が`jobType = "question_import"`、`payload = {ImportHistoryID, CourseID, UnitID, UserID}`（`QuestionImportRequested`相当のデータ）、`runAfter = time.Now()`（即時実行）で呼び出す
  - `jobs`テーブルGORMモデル・ポーリングワーカー本体（Infrastructure層、共通配置）: アーキテクチャ規約13章の標準実装をそのまま利用する
- 本Context固有の実装: `infrastructure/queue/question_import_job_handler.go`
  - `QuestionImportJobHandler`: `jobType = "question_import"`のjobをポーリングワーカーから受け取り、payloadを`dto.ExecuteQuestionImportCommand`へデコードして`ExecuteQuestionImportUseCase.Execute`を呼び出す関数（`main.go`起動時に`jobqueue.HandlerMap{"question_import": questionImportJobHandler}`として登録する。アーキテクチャ規約13章の「main.go（起動イメージ）」参照）

**②からの補足**: ②「18. Domain Event」は「具体的な非同期実行の仕組みは、アーキテクチャ規約.md「13. 非同期ジョブ実行パターン（JobQueue）」に定める`jobs`テーブル + ポーリングワーカーの標準実装に従う」と明記しており、②自体がこの標準実装への準拠を前提としている。旧版の本書（アーキテクチャ規約13章制定前）が独自定義していた`event.Publisher` Interface・専用ワーカー（`question_import_worker.go`）は、同規約の標準実装へ置き換える。

---

# 6. Presentation層設計

## Handler

- struct名: `QuestionImportHandler`
- 対応する呼び出し先: `StartQuestionImportUseCase` / `ValidateQuestionImportUseCase` / `DownloadQuestionCsvTemplateUseCase`
- メソッド一覧:
  - `StartImport(c *gin.Context)`: `POST /api/v1/admin/courses/:course_id/units/:unit_id/import_questions` に対応
  - `ValidateImport(c *gin.Context)`: `POST /api/v1/admin/courses/:course_id/units/:unit_id/import_questions/dry_run` に対応
  - `DownloadTemplate(c *gin.Context)`: `GET /api/v1/admin/csv_template/questions` に対応
- 処理順序:
  - `StartImport`:
    1. Middlewareで認証・adminロール確認済みであることを前提とする（本書「10. Authorization実装方針」参照）
    2. route paramから `course_id` / `unit_id` を取得する
    3. multipart formから `file` / `mode` を取得し `request.StartQuestionImportRequest` にバインドする
    4. Request DTOのバリデーション（型・必須・フォーマット）を行う
    5. `StartQuestionImportUseCase.Execute` を呼び出す
    6. 成功時は `response.StartQuestionImportResponse` に変換し、202を返す
    7. 失敗時はApplication Errorの種別に応じてHTTPステータスに変換する（本書「11. Error実装方針」参照）
  - `ValidateImport`:
    1. Middlewareで認証・adminロール確認済みであることを前提とする
    2. route paramから `course_id` / `unit_id` を取得する
    3. multipart formから `file` を取得し `request.ValidateQuestionImportRequest` にバインドする（modeはバインドしない。②「12. UseCase設計」の入力定義どおり）
    4. Request DTOのバリデーション（fileの必須・拡張子チェック）を行う
    5. `ValidateQuestionImportUseCase.Execute` を呼び出す
    6. 成功時は `response.ValidateQuestionImportResponse` に変換し、200を返す（行単位のエラーは`rows`に含め、HTTPエラーとはしない）
    7. Course/Unit不存在・ファイル形式不正時はApplication Errorの種別に応じてHTTPステータスに変換する
  - `DownloadTemplate`:
    1. Middlewareで認証・adminロール確認済みであることを前提とする
    2. リクエストパラメータのバインド・Validationは行わない（②「12. UseCase設計」の「入力: なし」のため）
    3. `DownloadQuestionCsvTemplateUseCase.Execute` を呼び出す
    4. 取得した`dto.DownloadQuestionCsvTemplateResult`を`Content-Type: text/csv`、`Content-Disposition: attachment`のレスポンスとして200で返す（Response DTO structは設けず、`c.Data`相当の直接書き込みとする）

`ExecuteQuestionImportUseCase` はHTTP経由では呼び出されない（非同期ワーカーから呼び出される）ため、対応するHandlerは存在しない。

## Request / Response DTO

- struct名: `StartQuestionImportRequest`
  - フィールド: `File *multipart.FileHeader`（`form:"file" binding:"required"`）/ `Mode string`（`form:"mode"`）
  - バリデーションタグ／チェック内容: fileの必須チェック、modeの値がある場合は文字列としての形式チェックのみ行う（append/overwrite以外の値を拒否せず、正規化はDomain側に委ねる。②「15. Validation設計」の責務分離に対応）
- struct名: `ValidateQuestionImportRequest`
  - フィールド: `File *multipart.FileHeader`（`form:"file" binding:"required"`）
  - バリデーションタグ／チェック内容: fileの必須チェック、ファイルの拡張子・Content-Typeがcsvであることの検証（②「15. Validation設計」の「フォーマットチェック」に対応）
- struct名: `StartQuestionImportResponse`
  - フィールド: `ImportHistoryID string`（json: `import_history_id`）/ `Message string`（json: `message`）/ `Status string`（json: `status`）
  - バリデーションタグ: 対象外（レスポンス用のため）
- struct名: `ValidateQuestionImportResponse`
  - フィールド: `TotalCount int`（json: `total_count`）/ `ValidCount int`（json: `valid_count`）/ `Rows []InvalidRowResponse`（json: `rows`）
  - バリデーションタグ: 対象外
- struct名: `InvalidRowResponse`
  - フィールド: `RowNumber int`（json: `row_number`）/ `Severity string`（json: `severity`）/ `Message string`（json: `message`）/ `Data map[string]string`（json: `data`。②「19. API仕様」の「rows（検証エラー行の row_number・severity・message・data）」に対応）
  - バリデーションタグ: 対象外
  - **②からの補足**: `Severity`・`Data`フィールドは②「19. API仕様」のResponse定義に記載があるが、②「12. UseCase設計」のUseCase出力定義（本書「4. Application層設計」の`InvalidRowResult`）には含まれていない。UseCase出力（`InvalidRowResult`）とResponse DTO（`InvalidRowResponse`）の間で構成が異なる点は②内の記述の粒度差によるものであり、本書ではResponse DTO側にのみ`Severity`（固定値`"error"`を推測で補う）・`Data`（元CSV行データ。UseCase出力への追加が必要になる可能性がある）を持たせる。この差異の解消（UseCase出力側にフィールドを追加するか、Handler側で補うか）は実装時の判断に委ねる「推測」である

CSVテンプレートダウンロードのResponse DTOは設けない（ファイルストリームを直接返すため。②「12. UseCase設計」参照）。

## Routing

|Method|Path|Handler|
|-|-|-|
|POST|/api/v1/admin/courses/:course_id/units/:unit_id/import_questions|QuestionImportHandler.StartImport|
|POST|/api/v1/admin/courses/:course_id/units/:unit_id/import_questions/dry_run|QuestionImportHandler.ValidateImport|
|GET|/api/v1/admin/csv_template/questions|QuestionImportHandler.DownloadTemplate|

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/admin/csv_template/questions|QuestionImportHandler.DownloadTemplate|なし|CSVファイル（`questions_template.csv`）|200|
|POST|/api/v1/admin/courses/:course_id/units/:unit_id/import_questions/dry_run|QuestionImportHandler.ValidateImport|ValidateQuestionImportRequest|ValidateQuestionImportResponse|200|
|POST|/api/v1/admin/courses/:course_id/units/:unit_id/import_questions|QuestionImportHandler.StartImport|StartQuestionImportRequest|StartQuestionImportResponse|202|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（②に明記なし。Middlewareでの認証失敗時の一般的な扱いとして「推測」で補う）|
|adminロールでない|403|権限エラー（②に明記なし。「推測」で補う）|
|course_id配下にunit_idが存在しない、またはunit_idが非active（インポート開始・事前検証共通）|404|対象単元が存在しない、または利用不可（②「11. Repository設計」「16. Authorization設計」のスコープ検証結果に基づく。Status Codeの数値自体は②に明記がないため「推測」）|
|fileが未指定、またはファイル形式・ヘッダーが不正（インポート開始）|422|ファイル不正（②「19. API仕様」の記載どおり）|
|fileが未指定、またはファイル形式・ヘッダーが不正（事前検証）|422|ファイル不正（②「19. API仕様」の「422（ファイル形式・ヘッダー不正）」に対応）|
|CSV行データの業務ルール違反（事前検証時）|200|エラーではなく検証結果として`rows`に含めて返す（②「19. API仕様」の記載どおり）|
|CSV行データの業務ルール違反（本番実行、行単位）|—（202受付後、非同期処理内でImportErrorとして記録され、HTTPレスポンスには現れない）|該当なし|

---

# 8. Transaction実装方針

## Transaction開始箇所

- `StartQuestionImportUseCase`: `CourseRepository`/`UnitRepository`での存在確認完了後、`TransactionManager.WithinTransaction`呼び出し時点でトランザクションを開始する
- `ExecuteQuestionImportUseCase`: 各行（またはバッチ）のQuestion作成/更新に入る直前に、行（またはバッチ）単位でトランザクションを開始する
- `ValidateQuestionImportUseCase` / `DownloadQuestionCsvTemplateUseCase`: トランザクションを使用しない（データ変更を伴わないため。②「14. Transaction設計」参照）

## Transaction終了箇所（Commit / Rollback条件）

- `StartQuestionImportUseCase`: `WithinTransaction`のクロージャ内で`ImportHistoryRepository.Create`と`JobPublisher.Publish`の両方が成功した時点でコミットする（アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」のTransactional Outboxパターン）。いずれかが失敗した場合は両方ロールバックし、「業務データは保存されたがジョブが登録されない」「ジョブは登録されたが業務データが保存されない」という不整合を防ぐ
- `ExecuteQuestionImportUseCase`: 各行（またはバッチ）のQuestion作成/更新が成功した時点でコミットする（失敗した行はそのバッチのみロールバックし、ImportRowResultとして失敗を記録して後続の行の処理を継続する）。全行処理完了後、ImportHistoryの最終状態（Complete後の状態）を別途コミットする

## 複数Repositoryにまたがる場合の扱い

- `StartQuestionImportUseCase`: `ImportHistoryRepository.Create`と`JobPublisher.Publish`（jobsテーブルへのINSERT）を同一トランザクション内で実行する（CourseRepository/UnitRepositoryは参照確認のみで、トランザクション開始前に完了させる）
- `ExecuteQuestionImportUseCase`: QuestionRepositoryへの作成/更新と、当該行に対応する処理結果の記録は、行（またはバッチ）単位のトランザクション内で完結させる。ImportHistoryRepository.Update（最終状態の反映）およびImportErrorRepository.CreateBatch（失敗行の記録）は、全行処理完了後の別トランザクションで実施する

②「14. Transaction設計」に明記されているとおり、UseCase単位でのトランザクション管理という基本方針からの意図的な逸脱であり、部分的成功を許容する業務要件を満たすための対応である。ジョブ登録をトランザクション内に含める点は、アーキテクチャ規約13章制定に伴う②からの補足（②の設計意図である「ImportHistoryの整合性維持」をより確実にする、矛盾しない具体化）である。

---

# 9. Validation実装方針

## Presentation

- `StartQuestionImportRequest` でのチェック内容:
  - 型チェック: course_id / unit_id はroute paramとして文字列で受け取り、Handler側で必要な型（既存の共通ID型）へ変換する
  - 必須チェック: fileの必須チェック
  - フォーマットチェック: modeが指定されている場合の文字列としての形式チェック（append/overwrite以外の値も受理し、正規化はDomain側に委ねる。②「15. Validation設計」の記載どおり）
- `ValidateQuestionImportRequest` でのチェック内容:
  - 型チェック: course_id / unit_idはroute paramとして文字列で受け取る
  - 必須チェック: fileの必須チェック
  - フォーマットチェック: ファイルの拡張子・Content-Typeがcsvであることの検証（②「15. Validation設計」の「フォーマットチェック」に対応）。modeは入力に含まれないためチェック対象外

## 業務ルール検証

- Entity／Value Object生成時に検証する内容:
  - `ImportMode`: append/overwrite以外の値をappendへ正規化
  - `ImportHistory`: courseID/unitID/userIDの必須チェック、生成直後のstatus固定
  - `ImportError`: rowNumber・messageの必須チェック
  - `ImportRowResult`: 成功/失敗に応じたquestionID/errorMessageの必須チェック
- UseCase内で判定する業務ルール:
  - `StartQuestionImportUseCase` / `ValidateQuestionImportUseCase`: CSVファイルの構造的妥当性（必須列の存在等。詳細は①未提供のため参照不可）。不正な場合は422として扱う
  - `ExecuteQuestionImportUseCase`: append/overwriteモードに応じた処理可否の判定（`QuestionImportPolicy`経由）、行データ内容の整合性確認（`QuestionCsvRowValidator`経由）
  - `ValidateQuestionImportUseCase`: 行データ内容の整合性確認（`QuestionCsvRowValidator`経由。`ExecuteQuestionImportUseCase`と同一のDomain Serviceを用いる）

## 責務分離

②「15. Validation設計」の記載どおり、Presentationは「ファイルが受理可能な形式か」を担当し、Domainは「行の内容が業務的に妥当か」「モードに応じた処理が可能か」を担当する。`QuestionCsvRowValidator`による行検証ロジックは事前検証・本番実行の両方から共通して呼び出され、判定結果の乖離を防ぐ。

---

# 10. Authorization実装方針

## Middlewareで行う処理

- 認証済みユーザーを特定し、adminロールであることを確認する（②「16. Authorization設計」の記載どおり）

## Handlerで行う処理

- 認証失敗時（Middlewareでのブロック時）のレスポンス整形。業務権限の判定はHandlerに持たせない

## UseCaseで行う処理

- `StartQuestionImportUseCase` / `ValidateQuestionImportUseCase`: route由来のcourse_id/unit_idを起点に、CourseRepository/UnitRepositoryを用いてCourse配下にUnitが実在し、かつactiveであることを検証する。リクエストボディに含まれうるunit_id等は信用せず、route paramの値のみを正として扱う（IDOR対策。②「16. Authorization設計」の「事前検証（dry_run）も本番実行と同じスコープ検証を行うことで、認可の抜け穴が生じないようにする」に対応）
- `DownloadQuestionCsvTemplateUseCase`: route由来のスコープ検証対象となるcourse_id/unit_idを持たない（テンプレートは単元に依存しない固定コンテンツのため）ため、追加のスコープ検証は行わない

## Domainで行う処理

- `ImportHistory` は所有者（userID）を保持する。②の記載どおり、これは将来的な参照制限（例: 自分が実行したインポート履歴のみ参照可能にする等）の材料として保持するものであり、本書の実装範囲では追加の参照制限ロジックは定義しない

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

- `domain/errors` パッケージのエラー（`ErrInvalidCSVRow` 等）は、UseCase内で `fmt.Errorf("...: %w", err)` により文脈情報を付与しつつApplication層のエラー型にラップする。コーディング規約「18. エラーハンドリング」の `%w` ラップ方針に従う
- `ValidateQuestionImportUseCase`は、行単位の`ErrInvalidCSVRow`をGoの`error`値としては上位に伝播させず、`ImportRowResult`の失敗結果（戻り値）として扱う。UseCase全体としてのApplication Errorは、ファイル全体の構造不正（course_id/unit_id不正、CSVヘッダー不正等）の場合のみ発生させる

## Application Error → HTTPレスポンスへの変換方針

- Handlerで `errors.Is` / `errors.As` を用いてApplication Errorの種別を判定し、対応するHTTPステータスに変換する（アーキテクチャ規約「12. Error変換パターン（AppError）」に従い、Gin規約のエラーハンドリングミドルウェアへ集約する）

## Infrastructure Errorのハンドリング方針

- ファイルストレージ保存失敗、`JobPublisher.Publish`（jobsテーブルへのINSERT）失敗、DB接続失敗は、Infrastructure層で発生した時点でラップしてApplication層に伝播させ、UseCase・Handlerでは技術的な詳細をレスポンスに含めず、汎用的な失敗として扱う（②「17. Error設計」の記載どおり）

|Error種別|発生層|HTTP Status|
|-|-|-|
|ErrCourseNotFound / ErrUnitNotFound|Application|404（②に明記なし。「推測」）|
|ErrUnitNotActive|Application|404（②に明記なし。「推測」）|
|ErrInvalidFileFormat（インポート開始）|Application|422（②「19. API仕様」の記載どおり）|
|ErrInvalidFileFormat（事前検証、ファイル形式・ヘッダー不正）|Application|422（②「19. API仕様」の記載どおり）|
|ErrInvalidCSVRow（事前検証、行単位）|Domain（Goのerrorとしては発生させず、ImportRowResultとして返却）|200（`rows`に明細を含める）|
|ErrInvalidCSVRow等（本番実行、行単位）|行処理内でImportErrorとして記録され、HTTPレスポンスには現れない（非同期処理のため）|該当なし|
|ファイルストレージ保存失敗・jobs登録失敗・DB接続失敗等|Infrastructure|500（②に明記なし。「推測」）|

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

- `gormmodel.ImportHistoryModel` ⇔ `import_histories` テーブル（Gorm規約の複数形命名規則どおり）
- `gormmodel.ImportErrorModel` ⇔ `import_errors` テーブル
- `Job`（アーキテクチャ規約13章の共通GORMモデル）⇔ `jobs` テーブル（本Context固有の定義ではなく共通基盤の一部として利用する）
- Course / Unit は既存のSchool Context側モデルを参照専用で利用する（新規モデル定義は行わない）

## 主要クエリの条件・ソート・ページネーション方針

- `ImportHistoryRepository.FindRecentByUserID`: user_id条件、created_at降順、limit指定（②「11. Repository設計」の記載どおり。offsetによるページネーションの要否は②に明記がなく、必要であれば実装時にlimit/offsetパラメータを追加する）
- `ImportErrorRepository.FindByImportHistoryID`: import_history_id条件。ソート順は row_number昇順を推測する（②に明記なし。「推測」）
- `jobs`テーブルへのポーリングクエリ: `status = 'pending' AND run_after <= ?`条件（アーキテクチャ規約13章の標準実装どおり。本Context固有の変更は加えない）

## 既存Schemaに対する変更（②「20. DB設計方針」の反映方針）

②は「import_historiesテーブルに、ファイルの保存先を直接表す項目（ファイルの保存パス、またはオブジェクトストレージ上のキーに相当する情報）を追加することを提案する（推測: 具体的なカラム名・型はGo実装仕様書で検討する）」としている。本書ではこれを受け、以下のカラム追加を提案する。

|カラム名|型|意味|
|-|-|-|
|file_reference|VARCHAR|ファイルの保存パス、またはオブジェクトストレージ上のキーに相当する情報|
|file_name|VARCHAR|アップロード時の元ファイル名|

加えて、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の標準実装に伴い、共通基盤として`jobs`テーブル（`id` / `type` / `payload` / `status` / `attempts` / `max_attempts` / `run_after` / `created_at` / `updated_at`）が必要になる。これは本機能固有のスキーマ変更ではなく、非同期ジョブ実行パターンを利用する全機能で共有する基盤テーブルであるため、本書では既存の共通基盤として利用する前提とし、テーブル定義自体はアーキテクチャ規約13章の定義をそのまま参照する。

**②からの補足**: `file_reference` / `file_name` のカラム名・型は②に明記がなく、本書で初めて具体化するものであり「推測」である。オブジェクトストレージ選定（S3・GCS等）自体は②のスコープ外（Infrastructure層設計「外部連携実装」参照）であり、本書でも選定は行わない。マイグレーション（既存ActiveStorageデータからの移行スクリプト）の詳細は②「20. DB設計方針」の「影響範囲」に記載があるが、具体的な移行手順は本書のスコープ外とする。

SQL文そのものは記載しない。

---

# 13. テストケース設計

②「22. テスト戦略」で採用パターンがDomain Modelであるため、区分はそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|ImportMode|不正な値（append/overwrite以外）を渡した場合にappendへ正規化されること|
|ImportMode|append/overwriteそれぞれが正しく判定されること（IsAppend/IsOverwrite）|
|ImportStatus|processing以外からのCompleteに対するエラー|
|ImportRowResult|成功時にquestionIDが必須であること／失敗時にerrorMessageが必須であること|
|ImportRowResult|questionIDを持たない成功結果（事前検証時）が表現できること|
|ImportHistory|生成直後のstatusが必ずprocessingであること|
|ImportHistory|Completeによる状態遷移が正しく反映されること|
|ImportError|rowNumber <= 0、message空の場合に生成エラーとなること|
|QuestionCsvRowValidator|必須列が欠如した行、正解番号が範囲外の行等を不正と判定すること|
|QuestionCsvRowValidator|事前検証・本番実行のいずれから呼び出しても同一の行に対して同一の判定結果を返すこと（②「8. Domain Service」の「事前検証と本番実行で判定基準がずれるリスク」への対策確認）|
|QuestionImportPolicy|append/overwriteそれぞれのモードで新規作成/更新の判定が正しく行われること|
|ImportResultAggregationPolicy|全行成功時にstatusがsuccessとなること|
|ImportResultAggregationPolicy|一部失敗時にstatusがpartial failureとなること|
|ImportResultAggregationPolicy|全行失敗時にstatusがfailureとなること|
|ImportResultAggregationPolicy|success/error/totalカウントの整合性|
|ImportResultAggregationPolicy|事前検証で用いる件数集計（total_count/valid_count相当）が本番実行と同一の集計ロジックで算出されること|

## UseCase Test

|対象|テストケース|
|-|-|
|StartQuestionImportUseCase|Course/Unitが存在し有効な場合にImportHistoryがprocessingで作成されること|
|StartQuestionImportUseCase|Courseが存在しない場合にErrCourseNotFoundを返すこと|
|StartQuestionImportUseCase|Unitがcourse配下に存在しない、または非activeの場合にエラーを返すこと|
|StartQuestionImportUseCase|ImportHistory作成とjob登録（JobPublisher.Publish呼び出し）が同一トランザクション内で行われ、一方が失敗した場合に両方ロールバックされること|
|ExecuteQuestionImportUseCase|全行成功時に最終状態がsuccessとなり、ImportHistoryが更新されること|
|ExecuteQuestionImportUseCase|一部行が失敗した場合に、成功した行のQuestionが保持されたまま失敗行がImportErrorとして記録されること（部分的成功の担保）|
|ExecuteQuestionImportUseCase|存在しないImportHistoryIDを指定した場合にエラーを返すこと|
|ValidateQuestionImportUseCase|Course/Unitが存在し有効な場合に検証結果（total_count/valid_count/rows）が返ること|
|ValidateQuestionImportUseCase|実行後にImportHistoryが1件も作成されていないこと|
|ValidateQuestionImportUseCase|実行後にQuestionRepositoryが一切呼び出されていないこと|
|ValidateQuestionImportUseCase|不正な行がrows（row_number・message）として返ること|
|ValidateQuestionImportUseCase|上限件数を超えるエラー行がある場合、rowsには上限件数までしか含まれず、total_count/valid_countの集計には全行が反映されること|
|ValidateQuestionImportUseCase|Courseが存在しない場合にErrCourseNotFoundを返すこと|
|DownloadQuestionCsvTemplateUseCase|呼び出しの都度、固定のヘッダー行＋サンプル行を含むCSVコンテンツが生成されること|
|DownloadQuestionCsvTemplateUseCase|生成されるCSVがBOM付きであること|

## Repository Test

|対象|テストケース|
|-|-|
|ImportHistoryRepository|Create後にFindByIDで取得できること|
|ImportHistoryRepository|Updateによりstatus・カウントが反映されること|
|ImportHistoryRepository|FindRecentByUserIDがuser_id絞り込み・作成日時降順で取得できること|
|ImportErrorRepository|CreateBatchによる一括作成が正しく行われること|
|ImportErrorRepository|FindByImportHistoryIDが対象のImportHistoryに紐づくエラーのみ取得すること|
|CourseRepository|存在しないcourse_idに対してExistsがfalseを返すこと|
|UnitRepository|course_id配下に存在しないunit_id、または非activeなunit_idに対して検出できること|

## Handler Test

|対象|テストケース|
|-|-|
|QuestionImportHandler.StartImport|正常系リクエストで202とImportHistoryIDを含むレスポンスが返ること|
|QuestionImportHandler.StartImport|fileが未指定の場合に422が返ること|
|QuestionImportHandler.StartImport|Course/Unitが存在しない場合に404が返ること（②に明記なきStatus Codeのため「推測」に基づくテスト）|
|QuestionImportHandler.ValidateImport|正常系リクエストで200とtotal_count/valid_count/rowsを含むレスポンスが返ること|
|QuestionImportHandler.ValidateImport|fileが未指定の場合に422が返ること|
|QuestionImportHandler.ValidateImport|Course/Unitが存在しない場合に404が返ること|
|QuestionImportHandler.DownloadTemplate|200とCSVコンテンツ（Content-Type: text/csv）が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|question-import機能全体|エンドポイント経由でのアップロード受付から、非同期処理完了後のImportHistory状態・ImportError記録までを一貫して確認する（②の記載どおり）|
|question-import機能全体|route由来のunit_idとリクエストボディのunit_idが異なる場合でも、route側の値のみが使用されること（IDOR対策の確認）|
|question-import機能全体|事前検証（dry_run）実行後にImportHistoryが作成されていないこと（②「22. テスト戦略」の記載どおり）|
|question-import機能全体|CSVテンプレートダウンロードのエンドポイントが、DBアクセスなしにCSVファイルを返すこと|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|Context名 `question-import` に対応する `internal/` ディレクトリ名を `question_import` とした|アーキテクチャ規約「8. 命名規約」に従うための実装上の対応|推測ではない（規約に基づく機械的な変換）|
|ImportStatusの終了状態の内部値を `StatusSuccess` / `StatusPartialFailure` / `StatusFailure` と具体的に命名した|②は「成功／一部失敗／失敗」という区分を業務要件からの推測として記載するに留まり、コード上の識別子までは定義していない|推測|
|非同期実行の基盤として、アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」が定める`JobPublisher` + `jobs`テーブル + ポーリングワーカーの標準実装を採用した|同規約が2026-08に制定され、旧版の本書が独自定義していた`event.Publisher`・専用ワーカーに代わる標準実装として明示的に定められたため。旧版の「具体的なメッセージング基盤の選定は本書のスコープ外とする」という記述はこの標準実装で置き換える|規約制定に伴う反映（推測ではない）|
|`StartQuestionImportUseCase`のジョブ登録（`JobPublisher.Publish`）をImportHistory作成と同一トランザクション内で行う構成とした|アーキテクチャ規約13章のTransactional Outboxパターンに従うための実装上の対応。旧版が「推測」としていたコミット後発行の構成から変更した|規約準拠のための修正（推測ではない）|
|バッチ処理の粒度（行単位かバッチ単位か）・具体的なバッチサイズを確定させなかった|②「14. Transaction設計」が「行（またはバッチ）単位」という選択の余地を残した表現のままであり、③で数値を確定させる根拠がない|未確定事項の引き継ぎ（推測ではない）|
|`import_histories` への追加カラムを `file_reference` / `file_name` と具体的に命名した|②「20. DB設計方針」が「具体的なカラム名・型はGo実装仕様書で検討する」と明記しているため、③で初めて具体化した|推測|
|401/403/404/500のHTTP Statusを②に明記のない箇所で補った（202/422/200(dry_run)は②に明記あり）|②「19. API仕様」には202・422・200のみ明記されており、他のエラーケースは一般的なREST API設計から補う必要があった|推測|
|QuestionCSVRowの具体的なフィールド構成を定義しなかった|①（Rails実装詳細）未提供のため参照不可であり、かつQuestion Context自体の②文書が存在しないため、詳細フィールドを創作しない方針とした|判断（未提供情報を補わない方針）|
|UnitRepository.FindActiveUnitInCourseの戻り値型（UnitRef）の具体的な内容を確定させなかった|②に戻り値の型の詳細記載がなく、実装時に必要最小限の情報を返す設計とした|推測|
|Course/UnitのGORMモデルは新規定義せず、School Context側の既存モデルを参照する前提とした|②「11. Repository設計」で「Course, Unit（参照用）」と明記されており、Course/Unitのモデル自体はSchool領域（curriculum Context）の責務であるため|②の記載に基づく判断（推測ではない）|
|`InvalidRowResponse`に`Severity`・`Data`フィールドを追加し、UseCase出力（`InvalidRowResult`）との間に構成差を許容した|②「19. API仕様」のResponse定義には`severity`・`data`が含まれるが、②「12. UseCase設計」のUseCase出力定義には含まれておらず、②内の記述粒度差をそのまま実装に反映した|推測|
|ImportRowResultの`questionID`フィールドをポインタ型（`*string`）とした|事前検証時はQuestionが作成されないため`questionID`を持たない成功結果を表現する必要があり、②「7. Value Object設計」の記載を型として具体化する必要があったため|推測|

---

# 設計差分に関する補足

本書は②「24. 採用しなかった設計」「25. 設計判断サマリー」「設計差分管理」に記載された内容（Transaction Script/Active Record/Event Sourcingの不採用、Rails ActiveStorageからの脱却、受付処理と実行処理のUseCase分離、事前検証と本番実行の行検証ロジック共有等）をすべて前提とし、変更していない。非同期実行基盤の具体化（アーキテクチャ規約13章準拠）は、②が示した設計判断（Domain Eventとしての`QuestionImportRequested`採用）に矛盾しない実装レベルの具体化である。
