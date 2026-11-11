# 管理者インポート履歴管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者がCSVインポート（問題インポート）の実行履歴を検索・確認し、失敗内容を分析できるようにする機能である。状態・単元・コース・実行者・期間で絞り込み・並び替えを行う一覧取得、実行結果とエラー明細を含む詳細取得、エラー明細をCSVファイルとして出力するエクスポートの3操作を提供する。インポート履歴（`import_histories`）は問題インポート・生徒インポートの両方で共通して記録されるが、本機能が対象とするのは管理者が実行する問題インポートの履歴のみであり、教師が担当する生徒インポートの履歴は対象外である（②「1. 機能概要」の要約）。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により **Domain Model** を採用する。

- 本機能が検索・参照するImportHistoryは、既に管理者問題インポート機能_Go移行・設計仕様書においてDomain Modelとして設計され、Aggregate Root・Repository Interfaceによる依存性逆転構造を持つ。同一Aggregateに対して書き込み側と読み取り側で異なる実装構造を採用すると、Repository Interfaceの置き場所が機能ごとに矛盾し、同一Context（question-import）の内部構造が二重化する
- CSVエラー明細のエクスポートには、表計算ソフトでの数式注入を防ぐエスケープ処理というセキュリティ上重要な業務ルールが存在し、独立したDomain Serviceとして切り出す価値がある
- 検索条件の解釈（ImportHistorySearchCondition）とCSVエスケープ（CsvInjectionGuard）を独立したValue Object・Domain Serviceとして切り出すことで、DBアクセスなしに単体テストできる

Transaction Script・Active Record（同一AggregateをDomain Modelとして扱う管理者問題インポート機能との構造的一貫性が崩れるため不採用）、Event Sourcing（履歴の再構築・監査要件が現行仕様に存在せず過剰設計）はいずれも②で不採用と判断されている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- 対象Bounded Context: `question-import`（管理者問題インポート機能_Go移行・設計仕様書・管理者問題インポート機能_Go実装仕様書と同一Context。②「3. Bounded Context」の「同一データに対する参照専用の切り口という分割基準には該当せず、question-import Contextへ統合する」という判断による）
- ②「12. UseCase設計」のSearchImportHistoriesUseCase（一覧検索）、ShowImportHistoryUseCase（詳細取得）、ExportImportHistoryErrorsUseCase（CSVエクスポート）
- ②「19. API仕様」記載の3エンドポイント

**重要（Entity/Repositoryの再利用方針）**: 本機能が扱うImportHistory・ImportError Entity、およびImportHistoryRepository・ImportErrorRepository Interfaceの基本定義（struct名・フィールド・基本CRUD相当のメソッド）は、管理者問題インポート機能_Go実装仕様書「3. Domain層設計」で既に定義済みである。本書ではこれらを**再定義せず、そのまま再利用する**。本書が追加するのは、(a) 既存の`ImportHistoryRepository` interfaceへの検索系メソッドの追加、(b) 本機能固有のValue Object（`ImportHistorySearchCondition`, `CsvExportRow`）・Domain Service（`CsvInjectionGuard`）、(c) 本機能固有のUseCase・Handlerのみである。管理者問題インポート機能_Go実装仕様書「2. ディレクトリ構成」で作成済みの`internal/question_import/domain/entity/import_history.go`・`import_error.go`・`domain/repository/import_history_repository.go`・`domain/repository/import_error_repository.go`は本書でも同一ファイルとして扱い、重複するファイルを新規に作成しない。

- Railsの実装詳細（①）は本タスクで提供されておらず、参照が必要な箇所は「①未提供のため参照不可」として扱う

---

# 2. ディレクトリ構成

- 対象Bounded Context名: `question-import`（`internal/` 配下のディレクトリ名は、管理者問題インポート機能_Go実装仕様書と同一の `question_import` を用いる）
- ②で採用した設計パターン: Domain Model
- アーキテクチャ規約「3. 設計パターンごとの構造適用方針」の Domain Model 構造（標準フルレイヤー構成）を、既存の`internal/question_import/`配下に追加する形で適用する（新規Context・新規ディレクトリツリーを作らない）

## 既存ディレクトリ（管理者問題インポート機能_Go実装仕様書で作成済み、本書では変更しない）

```
internal/question_import/
├── domain/
│   ├── entity/            # import_history.go, import_error.go は変更しない
│   ├── valueobject/
│   ├── repository/        # import_history_repository.go は本書でメソッド追加（後述）
│   ├── service/
│   ├── event/
│   └── errors/
├── application/
│   ├── dto/
│   └── usecase/
├── infrastructure/
│   ├── persistence/gorm/
│   ├── repository/        # import_history_repository.go は本書でメソッド実装追加
│   └── queue/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

## 本書が追加するファイル一覧

```
internal/question_import/domain/valueobject/import_history_search_condition.go
internal/question_import/domain/valueobject/csv_export_row.go
internal/question_import/domain/service/csv_injection_guard.go
internal/question_import/application/dto/search_import_histories.go
internal/question_import/application/dto/show_import_history.go
internal/question_import/application/dto/export_import_history_errors.go
internal/question_import/application/usecase/search_import_histories_usecase.go
internal/question_import/application/usecase/show_import_history_usecase.go
internal/question_import/application/usecase/export_import_history_errors_usecase.go
internal/question_import/presentation/handler/import_history_handler.go
internal/question_import/presentation/request/search_import_histories_request.go
internal/question_import/presentation/response/import_history_response.go
internal/question_import/presentation/routes.go
```

`presentation/routes.go`は管理者問題インポート機能_Go実装仕様書で作成済みのファイルであり、本書ではそこへ`ImportHistoryHandler`のルート登録を追加する（新規ファイルとしては作成しない。10章参照）。

## 本書が変更（メソッド追加）する既存ファイル

|ファイル|追加内容|
|-|-|
|`domain/repository/import_history_repository.go`|`ImportHistoryRepository` interfaceへ`Search` / `FindDetailByID`メソッドを追加する（本書「3. Domain層設計」参照）|
|`infrastructure/repository/import_history_repository.go`|上記追加メソッドの実装を追加する（本書「5. Infrastructure層設計」参照）|

`domain/repository/import_error_repository.go`（`ImportErrorRepository`）は、管理者問題インポート機能_Go実装仕様書が既に定義した`FindByImportHistoryID`メソッドをそのまま利用し、変更を加えない（②「11. Repository設計」の「ImportErrorRepository（拡張部分）: ImportHistory IDによるエラー一覧取得（行番号順）。詳細取得・CSVエクスポートの両方で利用する」に対応。既存メソッドの用途を拡張するのみで、シグネチャの変更は不要と判断する）。

---

# 3. Domain層設計

## Entity（再利用、本書では再定義しない）

### ImportHistory / ImportError

管理者問題インポート機能_Go実装仕様書「3. Domain層設計」の`ImportHistory` / `ImportError` struct定義（フィールド・メソッド・不変条件）をそのまま再利用する。本書はこれらのEntityに新たなフィールド・メソッドを追加しない。

## 参照用の構成データ（Read Model、本書で新規追加）

### ImportHistoryListItem

- struct名: `ImportHistoryListItem`（配置: `application/dto`。②「6. Entity設計」は本データを「参照用の構成データ」と位置づけ、Entityではないと明記しているため、Domain層ではなくApplication層のDTOとして配置する。②の判断根拠「表示専用の結合結果を持たせるとEntityに他Contextへの参照責務が混入する」を、実装レイヤーの配置としても徹底する）
- 保持するフィールドと型: `ID string`, `CourseName string`, `UnitName string`, `UserName string`, `FileName string`, `Status string`, `Mode string`, `TotalCount int`, `SuccessCount int`, `ErrorCount int`, `CreatedAt time.Time`
- 各フィールドの意味: ②「6. Entity設計」の「一覧表示に必要な項目（course, unit, user, file_name, status, mode, 各種件数, created_at）」に対応

### ImportHistoryDetail

- struct名: `ImportHistoryDetail`（配置: `application/dto`。理由は`ImportHistoryListItem`と同様）
- 保持するフィールドと型: `ID string`, `CourseName string`, `UnitName string`, `UserName string`, `FileName string`, `Status string`, `Mode string`, `TotalCount int`, `SuccessCount int`, `ErrorCount int`, `StartedAt time.Time`, `CompletedAt *time.Time`, `Errors []ImportErrorDetail`
- `ImportErrorDetail`のフィールド: `RowNumber int`, `Message string`

**②からの補足**: ②「9. クラス図」はImportHistoryListItem/ImportHistoryDetailをDomain層のクラス図に含めているが、②「6. Entity設計」の判断根拠（「Entity本体に表示専用の結合結果を持たせるとEntityに他Contextへの参照責務が混入するため、application層の出力として分離する」）に従い、実装上はApplication層のDTOとして配置する。これは②の設計意図（表示専用データをEntityから分離する）をそのまま実装に反映したものであり、②の判断に反するものではない。

## Value Object（本書で新規追加）

### ImportHistorySearchCondition（`domain/valueobject/import_history_search_condition.go`）

- struct名: `ImportHistorySearchCondition`
- 保持するフィールドと型: `Status *valueobject.ImportStatus`, `UnitID *string`, `CourseID *string`, `UserID *string`, `From *time.Time`, `To *time.Time`, `Sort string`, `Order string`
- 生成時に検証するルール（②「7. Value Object設計」）:
  - `status`は許可値（`pending`/`processing`/`completed`/`failed`）以外が指定された場合、絞り込み条件として適用しない（`Status`フィールドを`nil`のまま保持する）
  - `sort`は許可値（`created_at`/`total_count`/`success_count`/`error_count`/`status`）以外の場合`created_at`にフォールバックし、同一条件時は`id`の降順を副次キーとする
  - `order`は`asc`/`desc`以外の場合`desc`にフォールバックする
  - `from`/`to`は日付単位の指定であり、開始日は当日0時、終了日は当日23時59分59秒までを含む期間へ展開する
  - 検索対象は常に`import_type`が問題インポートである履歴、かつ論理削除されていない履歴に限定する（この条件はVO自体のフィールドではなく、後述する`ImportHistoryRepository.Search`が常に付加する固定条件として扱う。10章参照）
- 公開するmethod一覧: `NewImportHistorySearchCondition(raw RawSearchParams) ImportHistorySearchCondition`（正規化を行うファクトリ。`RawSearchParams`は正規化前の生値を保持する入力structで、`application/dto`に定義する）、`(c ImportHistorySearchCondition) HasStatus() bool`

### CsvExportRow（`domain/valueobject/csv_export_row.go`）

- struct名: `CsvExportRow`
- 保持するフィールドと型: `RowNumber int`, `Status string`, `Message string`
- 生成時に検証するルール: なし（値の組み合わせ自体に不変条件はなく、生成時に`CsvInjectionGuard`によるエスケープを適用済みの`Message`を保持する）
- 公開するmethod一覧: `NewCsvExportRow(rowNumber int, status string, escapedMessage string) CsvExportRow`

## Repository Interface

### ImportHistoryRepository（既存interfaceへのメソッド追加）

管理者問題インポート機能_Go実装仕様書が定義した`ImportHistoryRepository` interface（`Create` / `Update` / `FindByID` / `FindRecentByUserID`）に、以下のメソッドを追加する（同一ファイル`domain/repository/import_history_repository.go`内、同一interfaceへの追記）。

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`Search`|`(ctx context.Context, condition valueobject.ImportHistorySearchCondition, page, perPage int)`|`([]*entity.ImportHistory, courseNames, unitNames, userNames map[string]string, totalCount int64, error)`|`ImportHistorySearchCondition`に基づく複合検索・並び替え・ページング付きの一覧取得（②「11. Repository設計」の「拡張部分」）|
|`FindDetailByID`|`(ctx context.Context, id string)`|`(*entity.ImportHistory, courseName, unitName, userName string, error)`|IDによる詳細取得（実行者・単元・コース情報を含む）。問題インポートでない履歴（`import_type`が異なる）の場合は取得しない（②「17. Error設計」の「対象インポート履歴が生徒インポートの履歴である...存在しないものとして扱う」に対応）|

**②からの補足**: `Search`/`FindDetailByID`の戻り値に、コース名・単元名・実行者名（`courseNames`/`unitNames`/`userNames`のマップ、または単一値）を含める構成とした。②「6. Entity設計」はこれらの結合結果をEntity本体に持たせない方針を示しているが、Repository実装（Infrastructure層）がJOINクエリでまとめて取得すること自体は妨げないため、Repositoryの戻り値としては結合結果を返し、Application層でDTO（`ImportHistoryListItem`等）へ組み立てる構成とした（推測。②に戻り値の型までの明記はない）。

### ImportErrorRepository（既存interfaceをそのまま再利用）

管理者問題インポート機能_Go実装仕様書が定義した`ImportErrorRepository.FindByImportHistoryID`をそのまま利用する。本書での変更はない。

## Domain Service（本書で新規追加）

### CsvInjectionGuard（`domain/service/csv_injection_guard.go`）

- struct/interface名: `CsvInjectionGuard`（struct、ステートレス）
- メソッドシグネチャ: `Escape(message string) string`
- 責務: CSV出力対象の文字列（エラーメッセージ）が表計算ソフトでの数式として解釈されうる先頭文字（`=`、`+`、`-`、`@`）を持つ場合に、`'`を付与してエスケープする（②「8. Domain Service」）

## Domain Error（本書で新規追加）

`domain/errors/errors.go`（管理者問題インポート機能_Go実装仕様書が作成済みのファイル）に、以下を追加する。

- `ErrImportHistoryNotFound`: 対象インポート履歴が存在しない、または問題インポートの履歴でない場合に発生する（②「17. Error設計」の「対象インポート履歴が存在しない」「対象インポート履歴が生徒インポートの履歴である」を1つのエラーとして扱う。②が両者を区別せず「存在しないものとして扱う」と明記しているため）

**②からの補足**: 管理者問題インポート機能_Go実装仕様書は`ErrImportHistoryNotFound`を`ExecuteQuestionImportUseCase`の発生しうるApplication Errorとして既に定義している（同名のエラーが両機能で使われる）。本書ではこれを再利用し、新たに別名のエラー変数を追加しない。生徒インポートの履歴を指定した場合も同一のエラーとして扱う（②の記載どおり）。

---

# 4. Application層設計

## DTO（Command / Query）

- struct名: `RawSearchImportHistoriesParams`（配置: `application/dto/search_import_histories.go`）
  - フィールド: `Status string`, `UnitID string`, `CourseID string`, `UserID string`, `From string`, `To string`, `Sort string`, `Order string`, `Page int`, `PerPage int`（すべて正規化前の生値。Presentation層から受け取った文字列をそのまま保持する）
  - 区分: Query（`valueobject.NewImportHistorySearchCondition`への入力）
- struct名: `SearchImportHistoriesResult`
  - フィールド: `Items []ImportHistoryListItem`, `Page int`, `PerPage int`, `TotalCount int64`
  - 区分: UseCase出力
- struct名: `ShowImportHistoryQuery`（配置: `application/dto/show_import_history.go`）
  - フィールド: `ImportHistoryID string`
  - 区分: Query
- struct名: `ExportImportHistoryErrorsQuery`（配置: `application/dto/export_import_history_errors.go`）
  - フィールド: `ImportHistoryID string`
  - 区分: Query
- struct名: `ExportImportHistoryErrorsResult`
  - フィールド: `FileName string`, `Content []byte`
  - 区分: UseCase出力（②「19. API仕様」の「Response: import_history_<id>.csv（text/csv、BOM付き）」に対応）

## UseCase

### SearchImportHistoriesUseCase（`application/usecase/search_import_histories_usecase.go`）

- struct名: `SearchImportHistoriesUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository`（管理者問題インポート機能_Go実装仕様書で定義済みのInterfaceを、本書が追加した`Search`メソッドとあわせて利用する）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, params dto.RawSearchImportHistoriesParams) (dto.SearchImportHistoriesResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `valueobject.NewImportHistorySearchCondition(params)`で検索条件を正規化する（許可値判定・フォールバック・期間展開。②「13. シーケンス図」の「ImportHistorySearchConditionへ正規化（許可値判定・期間展開）」）
  2. `ImportHistoryRepository.Search(ctx, condition, params.Page, params.PerPage)`を呼び出し、対象履歴一覧とコース名・単元名・実行者名、総件数を取得する
  3. 各履歴を`ImportHistoryListItem`へ変換する
  4. `dto.SearchImportHistoriesResult`として返す
- トランザクション境界: なし（②「14. Transaction設計」により読み取りのみ）
- 発生しうるApplication Error: なし（Infrastructure Errorのみ）

### ShowImportHistoryUseCase（`application/usecase/show_import_history_usecase.go`）

- struct名: `ShowImportHistoryUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository`, `ImportErrorRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ShowImportHistoryQuery) (dto.ImportHistoryDetail, error)`
- 処理ステップ:
  1. `ImportHistoryRepository.FindDetailByID(ctx, query.ImportHistoryID)`で対象履歴（問題インポートの履歴に限定済み）とコース名・単元名・実行者名を取得する
  2. 取得できない場合、`ErrImportHistoryNotFound`を返す（②「17. Error設計」）
  3. `ImportErrorRepository.FindByImportHistoryID(ctx, query.ImportHistoryID)`でエラー明細一覧（行番号順）を取得する
  4. `dto.ImportHistoryDetail`を組み立てて返す
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrImportHistoryNotFound`

### ExportImportHistoryErrorsUseCase（`application/usecase/export_import_history_errors_usecase.go`）

- struct名: `ExportImportHistoryErrorsUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository`, `ImportErrorRepository`, `service.CsvInjectionGuard`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ExportImportHistoryErrorsQuery) (dto.ExportImportHistoryErrorsResult, error)`
- 処理ステップ（②「13. シーケンス図（CSVエクスポート）」の記載どおり）:
  1. `ImportHistoryRepository.FindDetailByID`で対象履歴を取得する（問題インポートであることの確認を兼ねる）
  2. 取得できない場合、`ErrImportHistoryNotFound`を返す
  3. `ImportErrorRepository.FindByImportHistoryID`でエラー明細一覧（行番号順）を取得する
  4. 各エラー行について`CsvInjectionGuard.Escape(message)`でエスケープを適用し、`valueobject.NewCsvExportRow`で`CsvExportRow`を生成する
  5. サマリー行＋ヘッダー行＋エラー明細行から構成されるCSVコンテンツ（BOM付き）を組み立てる（②「19. API仕様」の「1行目サマリー、2行目ヘッダー、以降エラー明細」）
  6. `dto.ExportImportHistoryErrorsResult`として返す
- トランザクション境界: なし（②「14. Transaction設計」によりデータ変更を伴わないため）
- 発生しうるApplication Error: `ErrImportHistoryNotFound`

---

# 5. Infrastructure層設計

## Repository実装（既存実装へのメソッド追加）

### ImportHistoryRepository実装（`infrastructure/repository/import_history_repository.go`、既存実装への追記）

管理者問題インポート機能_Go実装仕様書が定義した実装struct（`gormrepo`パッケージ内）に、以下のメソッド実装を追加する。

|メソッド|発行するクエリ内容|
|-|-|
|`Search`|`import_type = 'question'`かつ`deleted_at IS NULL`を常に付加し、`ImportHistorySearchCondition`の`Status`/`UnitID`/`CourseID`（`units`との結合経由）/`UserID`/`From`〜`To`（`created_at`の範囲）で絞り込む。`units`・`courses`・`users`（admin_accountコンテキストの`users`テーブルを参照専用で利用）とJoinしてコース名・単元名・実行者名を取得する。`Sort`/`Order`（同条件時は`id`降順を副次キー）でソートし、`page`/`perPage`（上限100件）でOFFSET/LIMITページングする。総件数は同条件でのCOUNTを別途取得する（②「21. DB操作仕様」）|
|`FindDetailByID`|`id`一致かつ`import_type = 'question'`かつ`deleted_at IS NULL`の条件で1件取得し、`units`・`courses`・`users`とJoinしてコース名・単元名・実行者名を取得する|

### ImportErrorRepository実装（既存実装を再利用、変更なし）

管理者問題インポート機能_Go実装仕様書が定義した`FindByImportHistoryID`をそのまま利用する。行番号（`row_number`）昇順でのソートは既存実装の方針（管理者問題インポート機能_Go実装仕様書「5. Infrastructure層設計」で「推測」として補われたソート順）をそのまま踏襲する。

## 外部連携実装

対象外。②に本機能でのMail・Cache・Queue連携要件の記載はない（CSVエクスポートはHTTPレスポンスとして同期的に返却するのみで、非同期ジョブ化は行わない）。

---

# 6. Presentation層設計

## Handler

### ImportHistoryHandler（`presentation/handler/import_history_handler.go`）

- struct名: `ImportHistoryHandler`
- 対応する呼び出し先: `SearchImportHistoriesUseCase`, `ShowImportHistoryUseCase`, `ExportImportHistoryErrorsUseCase`
- メソッド一覧:
  - `Search(c *gin.Context)`: `GET /api/v1/admin/import_histories`に対応
  - `Show(c *gin.Context)`: `GET /api/v1/admin/import_histories/:id`に対応
  - `Export(c *gin.Context)`: `GET /api/v1/admin/import_histories/:id/export`に対応
- 処理順序:
  - `Search`: クエリパラメータ（`status`, `unit_id`, `course_id`, `user_id`, `from`, `to`, `sort`, `order`, `page`, `per_page`）を`request.SearchImportHistoriesRequest`にバインド → Request DTOのバリデーション（型・フォーマット） → `dto.RawSearchImportHistoriesParams`へ変換して`SearchImportHistoriesUseCase.Execute`を呼び出す → `response.ImportHistoryListResponse`へ変換し200で返す
  - `Show`: パスパラメータ`id`をバインド → `ShowImportHistoryUseCase.Execute`を呼び出す → `ErrImportHistoryNotFound`の場合は404に変換 → 成功時は`response.ImportHistoryDetailResponse`へ変換し200で返す
  - `Export`: パスパラメータ`id`をバインド → `ExportImportHistoryErrorsUseCase.Execute`を呼び出す → `ErrImportHistoryNotFound`の場合は404に変換 → 成功時は`Content-Type: text/csv`、`Content-Disposition: attachment; filename="import_history_<id>.csv"`のレスポンスとして200で返す（Response DTO structは設けず、`c.Data`相当の直接書き込みとする）

## Request / Response DTO

- struct名: `SearchImportHistoriesRequest`（`presentation/request/search_import_histories_request.go`）
  - フィールド: `Status string`（`form:"status"`）, `UnitID string`（`form:"unit_id"`）, `CourseID string`（`form:"course_id"`）, `UserID string`（`form:"user_id"`）, `From string`（`form:"from"`）, `To string`（`form:"to"`）, `Sort string`（`form:"sort"`）, `Order string`（`form:"order"`）, `Page int`（`form:"page"`）, `PerPage int`（`form:"per_page"`）
  - バリデーションタグ／チェック内容: `unit_id`/`course_id`/`user_id`/`page`/`per_page`は指定時に整数形式であることを検証する（②「15. Validation設計」）。`from`/`to`は指定時に日付形式（`binding:"omitempty,datetime=2006-01-02"`）であることを検証する。`status`/`sort`/`order`は文字列としての型チェックのみ行い、許可値かどうかの判定はDomain層（`ImportHistorySearchCondition`）に委ねる
- struct名: `ImportHistoryListItemResponse`（`presentation/response/import_history_response.go`）
  - フィールド: `ID string`（json: `id`）, `Course string`（json: `course`）, `Unit string`（json: `unit`）, `User string`（json: `user`）, `FileName string`（json: `file_name`）, `Status string`（json: `status`）, `Mode string`（json: `mode`）, `TotalCount int`（json: `total_count`）, `SuccessCount int`（json: `success_count`）, `ErrorCount int`（json: `error_count`）, `CreatedAt time.Time`（json: `created_at`）
- struct名: `ImportHistoryListResponse`
  - フィールド: `ImportHistories []ImportHistoryListItemResponse`（json: `import_histories`）, `Meta MetaResponse`
- struct名: `ImportErrorResponse`
  - フィールド: `RowNumber int`（json: `row_number`）, `Message string`（json: `message`）
- struct名: `ImportHistoryDetailResponse`
  - フィールド: `ID string`, `Course string`, `Unit string`, `User string`, `FileName string`, `Status string`, `Mode string`, `TotalCount int`, `SuccessCount int`, `ErrorCount int`, `Errors []ImportErrorResponse`（json: `errors`）, `Warnings []string`（json: `warnings`。②「19. API仕様」の「warnings（現状常に空）」に対応し、常に空配列を返す）

CSVエクスポートのResponse DTOは設けない（ファイルストリームを直接返すため）。

## Routing

`presentation/routes.go`（管理者問題インポート機能_Go実装仕様書で作成済みのファイル）へ、以下のルート登録を追加する。

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/admin/import_histories|ImportHistoryHandler.Search|
|GET|/api/v1/admin/import_histories/:id|ImportHistoryHandler.Show|
|GET|/api/v1/admin/import_histories/:id/export|ImportHistoryHandler.Export|

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/admin/import_histories|ImportHistoryHandler.Search|SearchImportHistoriesRequest|ImportHistoryListResponse|200|
|GET|/api/v1/admin/import_histories/:id|ImportHistoryHandler.Show|-（パスパラメータのみ）|ImportHistoryDetailResponse|200|
|GET|/api/v1/admin/import_histories/:id/export|ImportHistoryHandler.Export|-（パスパラメータのみ）|CSVファイル（`import_history_<id>.csv`）|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（②に明記なし。推測）|
|adminロールでない|403|権限エラー（②に明記なし。推測）|
|`unit_id`/`course_id`/`user_id`/`page`/`per_page`が数値でない、`from`/`to`が日付形式でない|400|Request DTOバリデーションエラー|
|対象インポート履歴が存在しない、または生徒インポートの履歴である（詳細・エクスポート）|404|`ErrImportHistoryNotFound`（②「17. Error設計」）|
|DB接続失敗・CSV生成失敗|500|Infrastructure Error|

---

# 8. Transaction実装方針

②「14. Transaction設計」のとおり、本機能はすべて読み取り専用処理であり、明示的なトランザクションを使用しない。

- Transaction開始箇所: なし
- Transaction終了箇所: 該当なし
- 複数Repositoryにまたがる場合の扱い: `ShowImportHistoryUseCase`・`ExportImportHistoryErrorsUseCase`は`ImportHistoryRepository`と`ImportErrorRepository`の両方を呼び出すが、いずれも読み取りのみであり、`TransactionManager`（アーキテクチャ規約「11. Transaction実装パターン」）を介したトランザクション制御は行わない（②「一覧検索・詳細取得・CSVエクスポートのいずれもImportHistory Aggregateへの書き込みを伴わないため、トランザクションによる整合性保証が不要」）

---

# 9. Validation実装方針

## Presentation

- `SearchImportHistoriesRequest`でのチェック内容: `unit_id`/`course_id`/`user_id`/`page`/`per_page`の型チェック、`from`/`to`の日付形式チェック（②「15. Validation設計」）
- `id`（詳細・エクスポート）はパスパラメータとして必須。ルーティング定義上、値が欠落するとGinのルーティング自体が一致しないため、バインド後の型検証で担保する

## 業務ルール検証

Domain Model採用のため、②の記載どおりEntity／Value Object生成時に検証する。

- `ImportHistorySearchCondition`: `status`が許可値でない場合は絞り込み条件として無視する。`sort`が許可値でない場合は`created_at`にフォールバックする。`order`が`asc`/`desc`でない場合は`desc`にフォールバックする。`from`/`to`は日境界（0時〜23:59:59）へ展開する。検索対象を常に問題インポートの履歴に限定する（②「15. Validation設計」の「Domain」節）
- UseCase内で判定する業務ルール: `ShowImportHistoryUseCase`・`ExportImportHistoryErrorsUseCase`は、対象履歴が存在し、かつ問題インポートの履歴であることを`ImportHistoryRepository.FindDetailByID`の結果（取得できるか否か）で確認する

## 責務分離

②「15. Validation設計」の記載どおり、Presentationは「入力が正しいか（型・形式）」を担当し、Domainは「検索条件として妥当か（許可値・対象種別）」を担当する。不正な値はエラーとせず無視・フォールバックすることで、Rails現行仕様の寛容な挙動を維持する（②の記載どおり、本機能ではDomain Errorを定義しない）。

---

# 10. Authorization実装方針

## Middlewareで行う処理

- 認証済みユーザーを特定し、adminロールであることを確認する（②「16. Authorization設計」）

## Handlerで行う処理

- 認証失敗時のレスポンス整形。業務権限の判定は持たせない

## UseCaseで行う処理

- 追加のスコープ制限は行わない（管理者はすべての問題インポート履歴を参照・エクスポート可能。②「16. Authorization設計」の「UseCase」節）
- 検索・参照対象を問題インポートの履歴に限定する（`ImportHistorySearchCondition`・`FindDetailByID`が常に`import_type = 'question'`を条件に含めることで実現する）

## Domainで行う処理

- `ImportHistorySearchCondition`が「問題インポートに限定する」という対象種別の条件を常に内包する（②「16. Authorization設計」の「Domain」節）

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

②「17. Error設計」のとおり、本機能では不正な検索条件値をエラーとせず無視・フォールバックで吸収するため、Domain Errorは定義しない。対象履歴の不存在・種別不一致のみをApplication Error（`ErrImportHistoryNotFound`、管理者問題インポート機能_Go実装仕様書で定義済みのものを再利用）として扱う。

## Application Error → HTTPレスポンスへの変換方針

Handler側で`errors.Is(err, domainerrors.ErrImportHistoryNotFound)`により判定し、該当する場合は404、それ以外のエラー（infrastructure由来）は500に変換する。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrImportHistoryNotFound`|Application（UseCase）|404|
|Request DTOバリデーションエラー|Presentation|400|
|DB接続失敗・CSV生成失敗|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

Repository実装が返すDBエラー（接続失敗・クエリ失敗）は`fmt.Errorf`でラップしてUseCase・Handlerへ伝播させ、Handlerで`ErrImportHistoryNotFound`以外のエラーとして一律500に変換する。

---

# 12. GORM / DBクエリ設計

②「20. DB設計方針」のとおり、`import_histories`/`import_errors`テーブルは管理者問題インポート機能と共有し、本機能単独でのスキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

- `gormmodel.ImportHistoryModel` ⇔ `import_histories`テーブル（管理者問題インポート機能_Go実装仕様書で定義済みのモデルをそのまま利用する。新規モデル定義は行わない）
- `gormmodel.ImportErrorModel` ⇔ `import_errors`テーブル（同上）
- `units`・`courses`・`users`は参照専用として既存モデル（curriculum Context・admin-account-management Context側で定義されるモデル、または本機能実装時点で未整備の場合はJOIN用の最小フィールド定義）をJoinして名称のみ取得する

## 主要クエリの条件・ソート・ページネーション方針

- `Search`: `import_type = 'question'`、`deleted_at IS NULL`、`status`（許可値のみ）、`unit_id`、`course_id`（unitsとの結合経由）、`user_id`、`created_at`の期間（日境界展開後）。`units`・`courses`・`users`とJoinしてコース名/単元名/実行者名を取得する。`sort`（`created_at`/`total_count`/`success_count`/`error_count`/`status`、既定`created_at`）・`order`（既定`desc`）による並び替え、同条件時は`id`降順を副次キーとする。`per_page`上限100件のページネーションを行う（②「21. DB操作仕様」）
- `FindDetailByID`: `id`一致、`import_type = 'question'`、`deleted_at IS NULL`
- `ImportErrorRepository.FindByImportHistoryID`: `import_history_id`条件、`row_number`昇順、全件取得

## 既存Schemaに対する変更

②20章のとおり変更なし。SQL文そのものは記載しない。

---

# 13. テストケース設計

②「22. テスト戦略」で採用パターンがDomain Modelであるため、区分はそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|`ImportHistorySearchCondition`|許可値の`status`が正しく設定されること／許可値でない`status`は絞り込み条件として無視されること|
|`ImportHistorySearchCondition`|許可値の`sort`がそのまま使用されること／許可値でない`sort`は`created_at`にフォールバックすること|
|`ImportHistorySearchCondition`|`order`が`asc`/`desc`以外の場合`desc`にフォールバックすること|
|`ImportHistorySearchCondition`|`from`/`to`が日境界（0時〜23:59:59）へ正しく展開されること|
|`CsvInjectionGuard.Escape`|`=`/`+`/`-`/`@`で始まるメッセージに`'`が付与されること／該当しないメッセージはそのまま返ること|

## UseCase Test

|対象|テストケース|
|-|-|
|`SearchImportHistoriesUseCase`|各検索条件（status/unit_id/course_id/user_id/期間）が正しく反映されること／並び替え・ページングが正しく適用されること|
|`ShowImportHistoryUseCase`|存在する履歴IDで詳細とエラー明細が返ること／存在しない履歴IDで`ErrImportHistoryNotFound`を返すこと／生徒インポートの履歴IDを指定した場合に`ErrImportHistoryNotFound`を返すこと（②の重点検証項目）|
|`ExportImportHistoryErrorsUseCase`|CSVコンテンツがサマリー行＋ヘッダー行＋エラー明細行の順で構成されること／エラーメッセージに`CsvInjectionGuard`のエスケープが適用されていること／BOMが付与されていること／存在しない履歴IDで`ErrImportHistoryNotFound`を返すこと|

## Repository Test

|対象|テストケース|
|-|-|
|`ImportHistoryRepository.Search`|複合検索条件（status/unit_id/course_id/user_id/期間）が正しく適用されること／並び替え・ページングが正しいこと／`import_type`が問題インポート以外の履歴が結果に含まれないこと|
|`ImportHistoryRepository.FindDetailByID`|存在する履歴IDで正しい詳細（コース名・単元名・実行者名を含む）が返ること／生徒インポートの履歴IDでは取得できないこと|
|`ImportErrorRepository.FindByImportHistoryID`|対象履歴に紐づくエラーのみ`row_number`昇順で取得できること|

## Handler Test

|対象|テストケース|
|-|-|
|`ImportHistoryHandler.Search`|クエリパラメータの解釈が正しいこと／不正な`from`/`to`で400が返ること／正常系で200が返ること|
|`ImportHistoryHandler.Show`|存在しない`id`で404が返ること／正常系で200が返ること|
|`ImportHistoryHandler.Export`|正常系で200とContent-Type: text/csvが返ること／存在しない`id`で404が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/admin/import_histories`|エンドポイント経由で一覧・詳細・CSVエクスポートが正しく動作すること|
|CSVエクスポート|CSV内の数式注入対策エスケープが実際に適用されていることを確認すること（②の記載どおり）|
|認可|admin以外のロールでアクセスした場合に403が返ること／未認証の場合に401が返ること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に整理する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ImportHistoryListItem/ImportHistoryDetailを、Domain層のEntityとしてではなくApplication層のDTOとして配置した|②「6. Entity設計」自身が「Entity本体に表示専用の結合結果を持たせるとEntityに他Contextへの参照責務が混入する」と明記しているため、これをレイヤー配置としても徹底した|②の判断根拠をレイヤー配置に反映したもの（推測ではない）|
|`ImportHistoryRepository`の`Search`/`FindDetailByID`の戻り値に、コース名・単元名・実行者名を含める構成とした|②に戻り値の型までの明記はないが、Repository実装がJOINクエリでまとめて取得する構成が自然であるため|推測|
|`ErrImportHistoryNotFound`を、管理者問題インポート機能_Go実装仕様書が既に定義したものと同一のエラー変数として再利用した|②「17. Error設計」は「対象インポート履歴が存在しない」「対象インポート履歴が生徒インポートの履歴である」を区別せず404として扱うと明記しており、別名のエラーを新設する根拠がないため|②の記載どおり（推測ではない）|
|`units`・`courses`・`users`の結合に用いるGORMモデルの具体的なフィールド構成を確定しなかった|curriculum Context・admin-account-management Context側の②③文書がそれぞれ独立して存在し、本タスクのスコープ外であるため、本書では参照専用の最小フィールドを利用する前提のみを記載した|推測|
|CSVエクスポートを同期的なHTTPレスポンスとして実装し、非同期ジョブ化しない|②に非同期化の記載がなく、既存Rails仕様（`Admin::ImportHistoryCsvExporterService`が同期的にCSVを生成する）をそのまま踏襲した|②の記載どおり（推測ではない）|

上記以外の設計判断（Bounded Context・Aggregate・Value Object・Domain Service・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API仕様・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
