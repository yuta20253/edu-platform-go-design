# 管理者問題インポート機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が指定単元（course_id / unit_id で route スコープされる）に対して CSV 形式の問題データをアップロードし、既存問題への追加（append）または上書き（overwrite）を行う機能である。アップロード自体は同期的に受け付け（202 Accepted）、実際の問題・選択肢・解説・ヒントの作成/更新は非同期処理で実行される。処理結果はインポート履歴（ImportHistory）と行単位のエラー情報（ImportError）として記録される。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により **Domain Model** を採用する。

- ImportHistory は「processing → 成功 / 一部失敗 / 失敗」という業務上意味のある状態遷移を持つ
- append/overwrite の判定ロジックは単純なデータ保存以上の業務ルールである
- 行単位の処理結果を集計して最終状態を決定するロジックは複数の処理結果を集約する業務ルールであり、Entity・Domain Serviceとして意味を持たせる価値が高い

Transaction Script（ルールが散在し保守性が低下する）、Active Record（状態遷移・集計ロジックによりモデルが肥大化する）、Event Sourcing（行単位イベントの永続化・再生要件が現時点で存在せず過剰設計）はいずれも②で不採用と判断されている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- 対象 Bounded Context: `question-import`
- ②「10. UseCase設計」の StartQuestionImportUseCase（CSVアップロード受付）、ExecuteQuestionImportUseCase（CSV行処理実行）
- ②「16. API互換方針」記載の同期受付エンドポイント1本
- Question Context（Question / QuestionChoice / QuestionHint / QuestionExplanation の作成・更新）は②の設計判断どおり本 Context のスコープ外であり、QuestionRepository を外部依存として参照するに留める。Question Context 自体の詳細設計（Entity構造・メソッドシグネチャ等）は、対応する②文書が存在しないため本書では定義しない
- Railsの実装詳細（①）は本タスクで提供されておらず、参照が必要な箇所は「①未提供のため参照不可」として扱う

---

# 2. ディレクトリ構成

- 対象 Bounded Context名: `question-import`（`internal/` 配下のディレクトリ名は `question_import` とする。アーキテクチャ規約「5. 命名規則」に基づき、Context名の kebab-case とディレクトリ名のスネークケースの対応をここに明記する）
- ②で採用した設計パターン: Domain Model
- アーキテクチャ規約「4. 設計パターンごとの構造適用方針」の Domain Model 構造（標準フルレイヤー構成）をそのまま適用する
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
internal/question_import/domain/service/question_import_policy.go
internal/question_import/domain/service/import_result_aggregation_policy.go
internal/question_import/domain/event/question_import_requested.go
internal/question_import/domain/event/publisher.go
internal/question_import/domain/errors/errors.go
internal/question_import/application/dto/start_question_import.go
internal/question_import/application/dto/execute_question_import.go
internal/question_import/application/usecase/start_question_import_usecase.go
internal/question_import/application/usecase/execute_question_import_usecase.go
internal/question_import/infrastructure/persistence/gorm/import_history_model.go
internal/question_import/infrastructure/persistence/gorm/import_error_model.go
internal/question_import/infrastructure/repository/import_history_repository.go
internal/question_import/infrastructure/repository/import_error_repository.go
internal/question_import/infrastructure/repository/course_repository.go
internal/question_import/infrastructure/repository/unit_repository.go
internal/question_import/infrastructure/queue/question_import_event_publisher.go
internal/question_import/infrastructure/queue/question_import_worker.go
internal/question_import/presentation/handler/question_import_handler.go
internal/question_import/presentation/request/start_question_import_request.go
internal/question_import/presentation/response/start_question_import_response.go
internal/question_import/presentation/routes.go
```

**②からの補足**: QuestionRepository の実装（Question Context が提供するもの）は question-import Context の外側にあるため、本書では Interface 定義のみを `domain/repository/question_repository.go` に置き、実装ファイルは一覧に含めない。実装は Question Context 側の③文書のスコープとする。

---

# 3. Domain層設計

## Entity

### ImportHistory

- struct名: `ImportHistory`
- 保持するフィールドと型:
  - `id`: `string`（またはUint、GORM規約に従い主キー名は `ID`。値の型は他Contextの既存ID型に合わせるため、実装時に既存の共通ID型を確認する — ②に型の明記はなくコード実装時の判断が必要）
  - `courseID`: `string`
  - `unitID`: `string`
  - `userID`: `string`（実行した管理者のID。②「13. Authorization設計」の「ImportHistoryの所有者（user_id）を保持」に対応）
  - `mode`: `valueobject.ImportMode`
  - `status`: `valueobject.ImportStatus`
  - `fileName`: `string`
  - `fileReference`: `string`（ファイル保存先を表す情報。②「17. DB設計方針」で追加が提案されているカラムに対応。詳細は本書「12. GORM/DBクエリ設計」参照）
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
- 具体的な struct定義・フィールドは Question Context の②文書がまだ存在しないため（②「9. Repository設計」の QuestionRepository の記載を参照）、本書では定義しない。**②からの補足**: これは①未提供のためではなく、Question Context 自体の②文書が存在しないことに起因する

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
  - `questionID`: `*string`（成功時のQuestion識別情報）
  - `errorMessage`: `string`（失敗時のエラーメッセージ）
- 生成時に検証するルール: 成功時は `questionID` が必須、失敗時は `errorMessage` が必須（②「7. Value Object設計」の記載どおり）
- 公開する method 一覧: `NewSuccessRowResult(rowNumber int, questionID string) ImportRowResult` / `NewFailureRowResult(rowNumber int, message string) ImportRowResult` / `IsSuccess() bool` / `RowNumber() int`

## Repository Interface

### ImportHistoryRepository

- interface名: `ImportHistoryRepository`
- メソッドシグネチャ一覧:
  - `Create(ctx context.Context, history *entity.ImportHistory) error`
  - `Update(ctx context.Context, history *entity.ImportHistory) error`: status・カウント・completedAt の更新に用いる
  - `FindByID(ctx context.Context, id string) (*entity.ImportHistory, error)`
  - `FindRecentByUserID(ctx context.Context, userID string, limit int) ([]*entity.ImportHistory, error)`: user_id 絞り込み・作成日時降順（②「9. Repository設計」の記載どおり）
- 各メソッドの責務: 永続化・検索に限定し、モード判定・状態遷移ルールの判断は持たない（②の記載どおり）

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
  - `FindActiveUnitInCourse(ctx context.Context, courseID, unitID string) (*UnitRef, error)`: course_id配下にunit_idが存在し、かつactiveであることを1回で確認する（②「9. Repository設計」の「course_id配下にunit_idが存在し、かつactiveであることの確認」に対応）
- 各メソッドの責務: routeスコープ検証に必要な参照確認に限定し、Unit自体の作成・更新は持たない
- **②からの補足**: `UnitRef` は確認結果を表す最小限のデータ（unit_idの存在・active可否）を返すための型。②に戻り値の型までは明記がないため、実装上の必要から補ったものであり「推測」である

### QuestionRepository（Question Context提供、外部依存として利用）

- interface名: `QuestionRepository`
- メソッドシグネチャ一覧:
  - `Create(ctx context.Context, row QuestionCSVRow) (questionID string, err error)`: 新規作成
  - `Update(ctx context.Context, questionID string, row QuestionCSVRow) error`: 既存問題の更新（overwrite時）
- 各メソッドの責務: 問題データの永続化。CSV解析・モード判定は持たない（②の記載どおり）
- **②からの補足**: `QuestionCSVRow`（CSV1行分のデータを表す型）の具体的なフィールド構成（問題文・選択肢・ヒント・解説等）は、①（Rails実装）未提供のため参照不可であり、かつ Question Context自体の②文書も存在しないため、本書では詳細フィールドを定義しない。question-import Context 側では「CSV1行分のデータを保持する型」として抽象的に扱い、フィールド定義は Question Context 側の設計時に確定させる前提とする

## Domain Service

### QuestionImportPolicy

- struct/interface名: `QuestionImportPolicy`（struct、ステートレス）
- メソッドシグネチャ: `Decide(row QuestionCSVRow, mode valueobject.ImportMode, existingQuestionID *string) ImportAction`
  - `ImportAction` は「新規作成すべきか」「既存Questionを更新すべきか」を表す値（②「8. Domain Service」の記載に対応する型。具体的な列挙値・判定条件は①未提供のため参照不可であり、③でも詳細ロジックは記述しない）
- 責務: CSV1行のデータとmode（append/overwrite）から、Questionを新規作成すべきか既存Questionを更新すべきかを判定する（②の記載どおり）

### ImportResultAggregationPolicy

- struct/interface名: `ImportResultAggregationPolicy`（struct、ステートレス）
- メソッドシグネチャ: `Aggregate(results []valueobject.ImportRowResult) (status valueobject.ImportStatus, successCount, errorCount, totalCount int)`
- 責務: 行単位の処理結果（ImportRowResultの集合）から、ImportHistoryの最終的な状態（成功 / 一部失敗 / 失敗）と各種カウントを決定する（②の記載どおり）

## Domain Event

### QuestionImportRequested

- イベントstruct名: `QuestionImportRequested`
- 保持するフィールド: `ImportHistoryID string` / `CourseID string` / `UnitID string` / `UserID string` / `OccurredAt time.Time`
- 発火元: `StartQuestionImportUseCase`（ImportHistoryがprocessing状態で作成された直後。②「15. Domain Event」の記載どおり）

## Domain Error

- エラー種別ごとの型／変数定義方針: `domain/errors` パッケージに、区別可能なエラー変数（`errors.New` ベース）またはエラー型（`fmt.Errorf` + `%w` でラップ可能な型）を定義する。コーディング規約「6. エラーハンドリング」「21. 変更しない方針」に従い、`errors.New` と `fmt.Errorf` の使い分けについて本書で独自ルールは設けない
- 発生条件（②「14. Error設計」より）:
  - `ErrInvalidCSVRow`: 不正なCSV行データ（必須列欠如、選択肢不整合等）
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

## UseCase

### StartQuestionImportUseCase

- struct名: `StartQuestionImportUseCase`
- コンストラクタが受け取る依存: `CourseRepository` / `UnitRepository` / `ImportHistoryRepository` / `event.Publisher`（Domain Event発行用。定義は本書「5. Infrastructure層設計」参照）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.StartQuestionImportCommand) (dto.StartQuestionImportResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `CourseRepository.Exists` で course_id の存在を確認する
  2. `UnitRepository.FindActiveUnitInCourse` で course_id 配下に unit_id が存在し active であることを確認する（②「13. Authorization設計」の IDOR対策に対応）
  3. `valueobject.NewImportMode(cmd.RawMode)` でモードを正規化する
  4. ファイルの保存（ファイル参照情報の生成。保存先の実装詳細は本書「5. Infrastructure層設計」参照）
  5. `entity.NewImportHistory(...)` で processing 状態のEntityを生成する
  6. `ImportHistoryRepository.Create` で永続化する（ここまでを1トランザクションとする。詳細は本書「8. Transaction実装方針」）
  7. トランザクションコミット後、`event.Publisher.Publish` で `QuestionImportRequested` を発行する
  8. `dto.StartQuestionImportResult` に変換して返す
- トランザクション境界: ②「11. Transaction設計」の「ImportHistoryの作成（processing状態）とファイル情報の保存を1トランザクションで実施する」を実装単位に落とし込み、ステップ1〜6を1トランザクションとする。イベント発行（ステップ7）はコミット確定後に行う
- 発生しうる Application Error: `ErrCourseNotFound` / `ErrUnitNotFound` / `ErrUnitNotActive` / `ErrInvalidFileFormat`

**②からの補足**: イベント発行をトランザクションコミット後に行う点は、②「11. Transaction設計」に明記がないため、一般的な「トランザクション未確定の状態変化を外部に通知しない」という実装上の判断として補ったものであり「推測」である。

### ExecuteQuestionImportUseCase

- struct名: `ExecuteQuestionImportUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository` / `ImportErrorRepository` / `QuestionRepository` / `service.QuestionImportPolicy` / `service.ImportResultAggregationPolicy`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.ExecuteQuestionImportCommand) (dto.ExecuteQuestionImportResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `ImportHistoryRepository.FindByID` で対象の ImportHistory を取得する
  2. 対象ファイルを読み込み、CSV行に分解する（パース処理の詳細は①未提供のため参照不可）
  3. 各行について `QuestionImportPolicy.Decide` でアクション（新規作成/更新）を判定する
  4. 判定結果に応じて `QuestionRepository.Create` または `QuestionRepository.Update` を呼び出す（行またはバッチ単位でトランザクション化。詳細は本書「8. Transaction実装方針」）
  5. 各行の結果を `valueobject.ImportRowResult` として収集する
  6. 全行処理後、`ImportResultAggregationPolicy.Aggregate` で最終状態とカウントを決定する
  7. `ImportHistory.Complete(...)` で Entity を終了状態へ遷移させる
  8. `ImportHistoryRepository.Update` で最終状態を永続化する（ステップ4とは別トランザクション）
  9. 失敗行があれば `ImportErrorRepository.CreateBatch` で一括保存する
  10. `dto.ExecuteQuestionImportResult` に変換して返す
- トランザクション境界: ②「11. Transaction設計」の記載どおり、行（またはバッチ）単位でQuestion作成/更新をトランザクション化し、全行処理完了後にImportHistoryの最終状態を別途コミットする。UseCase全体を1トランザクションにはしない（部分的成功を許容するための意図的な逸脱であることは②に明記済み）
- 発生しうる Application Error: `ErrImportHistoryNotFound` / `ErrInvalidStatusTransition`

**②からの補足**: バッチ処理の粒度（1行ごとか、複数行をまとめたバッチ単位か）や具体的なバッチサイズは②に明記がなく、「行（またはバッチ）単位」という選択の余地がある表現に留まっている。本書でも具体的な数値・粒度は確定させず、実装時に決定可能なパラメータとして扱う（推測ではなく②の未確定事項をそのまま実装仕様に引き継ぐ扱いとする）。

---

# 5. Infrastructure層設計

## Repository実装

### ImportHistoryRepository実装

- 実装struct名: `ImportHistoryRepository`（package `gormrepo`。アーキテクチャ規約「9. 命名規約」に従い `〇〇RepositoryImpl` のような接尾辞は付けない）
- 対応するGORMモデル: `gormmodel.ImportHistoryModel`（テーブル `import_histories`）
- 各メソッドで発行するクエリ内容:
  - `Create`: `import_histories` への1件INSERT
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
- Queue: 対象（②「15. Domain Event」「20. 採用しなかった設計」を踏まえた実装対象）

### Queue実装（Domain Event発行・非同期実行トリガー）

- 実装対象1: `event.Publisher` interface の実装（`infrastructure/queue/question_import_event_publisher.go`）
  - `Publish(ctx context.Context, event event.QuestionImportRequested) error`
  - 呼び出し元: `StartQuestionImportUseCase`
  - 実装方針: `QuestionImportRequested` を受け取り、非同期実行（`ExecuteQuestionImportUseCase` の起動）のトリガーとして送出する
- 実装対象2: 非同期ワーカー（`infrastructure/queue/question_import_worker.go`）
  - `QuestionImportRequested` を受信し、`ExecuteQuestionImportUseCase.Execute` を呼び出す
  - 呼び出し元: なし（Publisherからの送出を購読する側）

**②からの補足**: ②「15. Domain Event」は「非同期実行の具体的な仕組み（メッセージキュー、ワーカー構成等）はGo実装仕様書（③）で検討する対象とし、本書では『進行状態の変化をトリガーとして非同期処理を起動する』という設計判断のみを扱う」「具体的なメッセージング基盤の選定は本書のスコープ外とする（推測）」と明記している。これを受け、本書でも具体的なメッセージング基盤（例: SQS、Redis Streams、Kafka等）の選定は行わず、`event.Publisher` という抽象 Interface とワーカーの呼び出し関係のみを定義する。基盤選定は本書のスコープ外として明示的に据え置く。

---

# 6. Presentation層設計

## Handler

- struct名: `QuestionImportHandler`
- 対応する呼び出し先: `StartQuestionImportUseCase`
- メソッド一覧:
  - `StartImport(c *gin.Context)`: `POST /api/v1/admin/courses/:course_id/units/:unit_id/import_questions` に対応
- 処理順序:
  1. Middlewareで認証・adminロール確認済みであることを前提とする（本書「10. Authorization実装方針」参照）
  2. route paramから `course_id` / `unit_id` を取得する
  3. multipart formから `file` / `mode` を取得し `request.StartQuestionImportRequest` にバインドする
  4. Request DTOのバリデーション（型・必須・フォーマット）を行う
  5. `StartQuestionImportUseCase.Execute` を呼び出す
  6. 成功時は `response.StartQuestionImportResponse` に変換し、202を返す
  7. 失敗時はApplication Errorの種別に応じてHTTPステータスに変換する（本書「11. Error実装方針」参照）

`ExecuteQuestionImportUseCase` はHTTP経由では呼び出されない（非同期ワーカーから呼び出される）ため、対応するHandlerは存在しない。

## Request / Response DTO

- struct名: `StartQuestionImportRequest`
  - フィールド: `File *multipart.FileHeader`（`form:"file" binding:"required"`）/ `Mode string`（`form:"mode"`）
  - バリデーションタグ／チェック内容: fileの必須チェック、modeの値がある場合は文字列としての形式チェックのみ行う（append/overwrite以外の値を拒否せず、正規化はDomain側に委ねる。②「12. Validation設計」の責務分離に対応）
- struct名: `StartQuestionImportResponse`
  - フィールド: `ImportHistoryID string`（json: `import_history_id`）/ `Message string`（json: `message`）/ `Status string`（json: `status`）
  - バリデーションタグ: 対象外（レスポンス用のため）

## Routing

|Method|Path|Handler|
|-|-|-|
|POST|/api/v1/admin/courses/:course_id/units/:unit_id/import_questions|QuestionImportHandler.StartImport|

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|POST|/api/v1/admin/courses/:course_id/units/:unit_id/import_questions|QuestionImportHandler.StartImport|StartQuestionImportRequest|StartQuestionImportResponse|202|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（②に明記なし。Middlewareでの認証失敗時の一般的な扱いとして「推測」で補う）|
|adminロールでない|403|権限エラー（②に明記なし。「推測」で補う）|
|course_id配下にunit_idが存在しない、またはunit_idが非active|404|対象単元が存在しない、または利用不可（②「9. Repository設計」「13. Authorization設計」のスコープ検証結果に基づく。Status Codeの数値自体は②に明記がないため「推測」）|
|fileが未指定、またはファイル形式が不正|422|ファイル不正（②「16. API互換方針」の記載どおり）|
|CSV行データの業務ルール違反（行単位）|— （202受付後、非同期処理内でImportErrorとして記録され、HTTPレスポンスには現れない）|該当なし|

---

# 8. Transaction実装方針

## Transaction開始箇所

- `StartQuestionImportUseCase`: UseCase開始直後、CourseRepository/UnitRepositoryでの存在確認完了後（更新系操作に入る直前）にトランザクションを開始する
- `ExecuteQuestionImportUseCase`: 各行（またはバッチ）のQuestion作成/更新に入る直前に、行（またはバッチ）単位でトランザクションを開始する

## Transaction終了箇所（Commit / Rollback条件）

- `StartQuestionImportUseCase`: ImportHistoryの作成が完了した時点でコミットする。CourseRepository/UnitRepositoryでの存在確認失敗時、またはImportHistoryRepository.Createでのエラー時はロールバックする
- `ExecuteQuestionImportUseCase`: 各行（またはバッチ）のQuestion作成/更新が成功した時点でコミットする（失敗した行はそのバッチのみロールバックし、ImportRowResultとして失敗を記録して後続の行の処理を継続する）。全行処理完了後、ImportHistoryの最終状態（Complete後の状態）を別途コミットする

## 複数Repositoryにまたがる場合の扱い

- `StartQuestionImportUseCase`: ImportHistoryRepositoryの単一トランザクション内での操作に限定する（CourseRepository/UnitRepositoryは参照確認のみで、トランザクション開始前に完了させる）
- `ExecuteQuestionImportUseCase`: QuestionRepositoryへの作成/更新と、当該行に対応する処理結果の記録は、行（またはバッチ）単位のトランザクション内で完結させる。ImportHistoryRepository.Update（最終状態の反映）およびImportErrorRepository.CreateBatch（失敗行の記録）は、全行処理完了後の別トランザクションで実施する

②「11. Transaction設計」に明記されているとおり、UseCase単位でのトランザクション管理という基本方針からの意図的な逸脱であり、部分的成功を許容する業務要件を満たすための対応である。

---

# 9. Validation実装方針

## Presentation

- `StartQuestionImportRequest` でのチェック内容:
  - 型チェック: course_id / unit_id はroute paramとして文字列で受け取り、Handler側で必要な型（既存の共通ID型）へ変換する
  - 必須チェック: fileの必須チェック
  - フォーマットチェック: modeが指定されている場合の文字列としての形式チェック（append/overwrite以外の値も受理し、正規化はDomain側に委ねる。②「12. Validation設計」の記載どおり）

## 業務ルール検証

- Entity／Value Object生成時に検証する内容:
  - `ImportMode`: append/overwrite以外の値をappendへ正規化
  - `ImportHistory`: courseID/unitID/userIDの必須チェック、生成直後のstatus固定
  - `ImportError`: rowNumber・messageの必須チェック
  - `ImportRowResult`: 成功/失敗に応じたquestionID/errorMessageの必須チェック
- UseCase内で判定する業務ルール:
  - `StartQuestionImportUseCase`: CSVファイルの構造的妥当性（必須列の存在等。詳細は①未提供のため参照不可）
  - `ExecuteQuestionImportUseCase`: append/overwriteモードに応じた処理可否の判定（`QuestionImportPolicy`経由）、行データ内容の整合性確認（`QuestionImportPolicy`経由）

## 責務分離

②「12. Validation設計」の記載どおり、Presentationは「ファイルが受理可能な形式か」を担当し、Domainは「行の内容が業務的に妥当か」「モードに応じた処理が可能か」を担当する。

---

# 10. Authorization実装方針

## Middlewareで行う処理

- 認証済みユーザーを特定し、adminロールであることを確認する（②「13. Authorization設計」の記載どおり）

## Handlerで行う処理

- 認証失敗時（Middlewareでのブロック時）のレスポンス整形。業務権限の判定はHandlerに持たせない

## UseCaseで行う処理

- `StartQuestionImportUseCase`: route由来のcourse_id/unit_idを起点に、CourseRepository/UnitRepositoryを用いてCourse配下にUnitが実在し、かつactiveであることを検証する。リクエストボディに含まれうるunit_id等は信用せず、route paramの値のみを正として扱う（IDOR対策。②の記載どおり）

## Domainで行う処理

- `ImportHistory` は所有者（userID）を保持する。②の記載どおり、これは将来的な参照制限（例: 自分が実行したインポート履歴のみ参照可能にする等）の材料として保持するものであり、本書の実装範囲では追加の参照制限ロジックは定義しない

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

- `domain/errors` パッケージのエラー（`ErrInvalidCSVRow` 等）は、UseCase内で `fmt.Errorf("...: %w", err)` により文脈情報を付与しつつApplication層のエラー型にラップする。コーディング規約「6. エラーハンドリング」の `%w` ラップ方針に従う

## Application Error → HTTPレスポンスへの変換方針

- Handlerで `errors.Is` / `errors.As` を用いてApplication Errorの種別を判定し、対応するHTTPステータスに変換する

## Infrastructure Errorのハンドリング方針

- ファイルストレージ保存失敗、非同期ジョブディスパッチ失敗、DB接続失敗は、Infrastructure層で発生した時点でラップしてApplication層に伝播させ、UseCase・Handlerでは技術的な詳細をレスポンスに含めず、汎用的な失敗として扱う（②「14. Error設計」の記載どおり）

|Error種別|発生層|HTTP Status|
|-|-|-|
|ErrCourseNotFound / ErrUnitNotFound|Application|404（②に明記なし。「推測」）|
|ErrUnitNotActive|Application|404（②に明記なし。「推測」）|
|ErrInvalidFileFormat|Application|422（②「16. API互換方針」の記載どおり）|
|ErrInvalidCSVRow等（Domain Error由来）|行処理内でImportErrorとして記録され、HTTPレスポンスには現れない（非同期処理のため）|該当なし|
|ファイルストレージ保存失敗・DB接続失敗等|Infrastructure|500（②に明記なし。「推測」）|

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

- `gormmodel.ImportHistoryModel` ⇔ `import_histories` テーブル（Gorm規約の複数形命名規則どおり）
- `gormmodel.ImportErrorModel` ⇔ `import_errors` テーブル
- Course / Unit は既存のSchool Context側モデルを参照専用で利用する（新規モデル定義は行わない）

## 主要クエリの条件・ソート・ページネーション方針

- `ImportHistoryRepository.FindRecentByUserID`: user_id条件、created_at降順、limit指定（②「9. Repository設計」の記載どおり。offsetによるページネーションの要否は②に明記がなく、必要であれば実装時にlimit/offsetパラメータを追加する）
- `ImportErrorRepository.FindByImportHistoryID`: import_history_id条件。ソート順は row_number昇順を推測する（②に明記なし。「推測」）

## 既存Schemaに対する変更（②「17. DB設計方針」の反映方針）

②は「import_historiesテーブルに、ファイルの保存先を直接表す項目（ファイルの保存パス、またはオブジェクトストレージ上のキーに相当する情報）を追加することを提案する（推測: 具体的なカラム名・型はGo実装仕様書で検討する）」としている。本書ではこれを受け、以下のカラム追加を提案する。

|カラム名|型|意味|
|-|-|-|
|file_reference|VARCHAR|ファイルの保存パス、またはオブジェクトストレージ上のキーに相当する情報|
|file_name|VARCHAR|アップロード時の元ファイル名|

**②からの補足**: カラム名・型は②に明記がなく、本書で初めて具体化するものであり「推測」である。オブジェクトストレージ選定（S3・GCS等）自体は②のスコープ外（Infrastructure層設計「外部連携実装」参照）であり、本書でも選定は行わない。マイグレーション（既存ActiveStorageデータからの移行スクリプト）の詳細は②「17. DB設計方針」の「影響範囲」に記載があるが、具体的な移行手順は本書のスコープ外とする。

SQL文そのものは記載しない。

---

# 13. テストケース設計

②「18. テスト戦略」で採用パターンがDomain Modelであるため、区分はそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|ImportMode|不正な値（append/overwrite以外）を渡した場合にappendへ正規化されること|
|ImportMode|append/overwriteそれぞれが正しく判定されること（IsAppend/IsOverwrite）|
|ImportStatus|processing以外からのCompleteに対するエラー|
|ImportRowResult|成功時にquestionIDが必須であること／失敗時にerrorMessageが必須であること|
|ImportHistory|生成直後のstatusが必ずprocessingであること|
|ImportHistory|Completeによる状態遷移が正しく反映されること|
|ImportError|rowNumber <= 0、message空の場合に生成エラーとなること|
|QuestionImportPolicy|append/overwriteそれぞれのモードで新規作成/更新の判定が正しく行われること|
|ImportResultAggregationPolicy|全行成功時にstatusがsuccessとなること|
|ImportResultAggregationPolicy|一部失敗時にstatusがpartial failureとなること|
|ImportResultAggregationPolicy|全行失敗時にstatusがfailureとなること|
|ImportResultAggregationPolicy|success/error/totalカウントの整合性|

## UseCase Test

|対象|テストケース|
|-|-|
|StartQuestionImportUseCase|Course/Unitが存在し有効な場合にImportHistoryがprocessingで作成されること|
|StartQuestionImportUseCase|Courseが存在しない場合にErrCourseNotFoundを返すこと|
|StartQuestionImportUseCase|Unitがcourse配下に存在しない、または非activeの場合にエラーを返すこと|
|StartQuestionImportUseCase|ImportHistory作成後にQuestionImportRequestedイベントが発行されること|
|ExecuteQuestionImportUseCase|全行成功時に最終状態がsuccessとなり、ImportHistoryが更新されること|
|ExecuteQuestionImportUseCase|一部行が失敗した場合に、成功した行のQuestionが保持されたまま失敗行がImportErrorとして記録されること（部分的成功の担保）|
|ExecuteQuestionImportUseCase|存在しないImportHistoryIDを指定した場合にエラーを返すこと|

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

## Integration Test

|対象|テストケース|
|-|-|
|question-import機能全体|エンドポイント経由でのアップロード受付から、非同期処理完了後のImportHistory状態・ImportError記録までを一貫して確認する（②の記載どおり）|
|question-import機能全体|route由来のunit_idとリクエストボディのunit_idが異なる場合でも、route側の値のみが使用されること（IDOR対策の確認）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|Context名 `question-import` に対応する `internal/` ディレクトリ名を `question_import` とした|アーキテクチャ規約「5. 命名規則」の「英単語1語または短いスネークケースにする」に従うための実装上の対応|推測ではない（規約に基づく機械的な変換）|
|ImportStatusの終了状態の内部値を `StatusSuccess` / `StatusPartialFailure` / `StatusFailure` と具体的に命名した|②は「成功／一部失敗／失敗」という区分を業務要件からの推測として記載するに留まり、コード上の識別子までは定義していない|推測|
|`event.Publisher` Interfaceと非同期ワーカーの抽象構造を定義した一方、具体的なメッセージング基盤（キュー製品等）は選定していない|②「15. Domain Event」が「具体的なメッセージング基盤の選定は本書のスコープ外」と明記しているため、③でも選定を行わずInterfaceレベルに留めた|②の明示的な方針に基づく（推測ではない）|
|バッチ処理の粒度（行単位かバッチ単位か）・具体的なバッチサイズを確定させなかった|②「11. Transaction設計」が「行（またはバッチ）単位」という選択の余地を残した表現のままであり、③で数値を確定させる根拠がない|未確定事項の引き継ぎ（推測ではない）|
|`import_histories` への追加カラムを `file_reference` / `file_name` と具体的に命名した|②「17. DB設計方針」が「具体的なカラム名・型はGo実装仕様書で検討する」と明記しているため、③で初めて具体化した|推測|
|401/403/404/500のHTTP Statusを②に明記のない箇所で補った（202/422は②に明記あり）|②「16. API互換方針」には202と422のみ明記されており、他のエラーケースは一般的なREST API設計から補う必要があった|推測|
|QuestionCSVRowの具体的なフィールド構成を定義しなかった|①（Rails実装詳細）未提供のため参照不可であり、かつQuestion Context自体の②文書が存在しないため、詳細フィールドを創作しない方針とした|判断（未提供情報を補わない方針）|
|UnitRepository.FindActiveUnitInCourseの戻り値型（UnitRef）の具体的な内容を確定させなかった|②に戻り値の型の詳細記載がなく、実装時に必要最小限の情報を返す設計とした|推測|
|Course/UnitのGORMモデルは新規定義せず、School Context側の既存モデルを参照する前提とした|②「9. Repository設計」で「Course, Unit（参照用）」と明記されており、Course/Unitのモデル自体はSchool領域（curriculum Context）の責務であるため|②の記載に基づく判断（推測ではない）|

---

# 設計差分に関する補足

本書は②「20. 採用しなかった設計」「21. 設計判断サマリー」「設計差分管理」に記載された内容（Transaction Script/Active Record/Event Sourcingの不採用、Rails ActiveStorageからの脱却、受付処理と実行処理のUseCase分離等）をすべて前提とし、変更していない。
