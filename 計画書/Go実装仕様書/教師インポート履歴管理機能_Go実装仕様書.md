# 教師インポート履歴管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

教師が、自身が実行した生徒CSVインポートの結果を確認し、インポートされた生徒に配布する生徒コードの一覧をCSVファイルとして取得できるようにする機能である。状態・期間で絞り込み・並び替えを行う一覧取得、実行結果の件数・インポートされた生徒の一覧・エラー明細を含む詳細取得、インポートされた生徒の氏名・氏名カナ・学年・学級・生徒コードをCSVファイルとして出力するエクスポートの3操作を提供する。インポート履歴（`import_histories`）は問題インポート・生徒インポートの両方で共通して記録されるが、本機能が対象とするのは、ログイン中の教師自身が実行した生徒インポートの履歴のみであり、他の教師の履歴・問題インポートの履歴は対象外である（②「1. 機能概要」の要約）。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により **Domain Model** を採用する。

- 本機能が検索・参照するImportHistoryは、既に生徒CSVインポート機能_Go移行・設計仕様書においてDomain Modelとして設計され、Aggregate Root・Repository Interfaceによる依存性逆転構造を持つ。同一Aggregateに対して書き込み側と読み取り側で異なる実装構造を採用すると、Repository Interfaceの置き場所が機能ごとに矛盾し、同一Context（student-import）の内部構造が二重化する
- 参照範囲の限定（自身が実行した・生徒インポートの・未削除の履歴のみ）という授権に関わるルールと、数式注入対策のエスケープというセキュリティ上のルールを、検索条件のValue Object（StudentImportHistorySearchCondition）とDomain Service（CsvInjectionGuard）として切り出し、DBアクセスなしに単体テストできるようにする
- 本機能単体はTransaction Scriptでも表現可能であるが、同一AggregateをDomain Modelとして扱う生徒CSVインポート機能との構造的一貫性を優先する（管理者インポート履歴管理機能と同じ判断）

Transaction Script・Active Record（構造的一貫性が崩れるため不採用）、Event Sourcing（履歴の再構築・監査要件が現行仕様に存在せず過剰設計）はいずれも②で不採用と判断されている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- 対象Bounded Context: `student-import`（生徒CSVインポート機能_Go移行・設計仕様書・生徒CSVインポート機能_Go実装仕様書と同一Context。②「3. Bounded Context」の「同一データに対する参照専用の切り口という分割基準には該当せず、student-import Contextへ統合する」という判断による）
- ②「12. UseCase設計」のSearchStudentImportHistoriesUseCase（一覧検索）、ShowStudentImportHistoryUseCase（詳細取得）、ExportStudentCodeCsvUseCase（生徒コード配布用CSVエクスポート）
- ②「19. API仕様」記載の3エンドポイント

**重要（Entity/Repositoryの再利用方針）**: 本機能が扱うImportHistory・ImportError・ImportedStudent Entity、およびImportHistoryRepository・ImportErrorRepository・ImportedStudentRepository Interfaceの基本定義（struct名・フィールド・基本的なメソッド）は、生徒CSVインポート機能_Go実装仕様書「3. Domain層設計」で既に定義済みであり、同書が「正」である。本書ではこれらを**再定義せず、そのまま再利用する**。本書が追加するのは、(a) 既存の`ImportHistoryRepository` interfaceへの検索系メソッドの追加、(b) 既存の`ImportedStudentRepository` interfaceへの、生徒の表示情報付きの取得メソッドの追加、(c) 本機能固有のValue Object（`StudentImportHistorySearchCondition`, `StudentCodeCsvRow`）・Domain Service（`CsvInjectionGuard`）、(d) 本機能固有のUseCase・Handler・Request/Response DTOのみである。生徒CSVインポート機能_Go実装仕様書「2. ディレクトリ構成」で作成済みの`internal/student_import/domain/entity/import_history.go`・`import_error.go`・`imported_student.go`・`domain/repository/import_history_repository.go`・`import_error_repository.go`・`imported_student_repository.go`は本書でも同一ファイルとして扱い、重複するファイルを新規に作成しない。

- Railsの実装詳細（①）のうち、参照が必要な箇所は「Rails現行仕様書（教師インポート履歴管理機能）」として明示する

---

# 2. ディレクトリ構成

- 対象Bounded Context名: `student-import`（`internal/` 配下のディレクトリ名は、生徒CSVインポート機能_Go実装仕様書と同一の `student_import` を用いる）
- ②で採用した設計パターン: Domain Model
- アーキテクチャ規約「3. 設計パターンごとの構造適用方針」の Domain Model 構造（標準フルレイヤー構成）を、既存の`internal/student_import/`配下に追加する形で適用する（新規Context・新規ディレクトリツリーを作らない）

## 既存ディレクトリ（生徒CSVインポート機能_Go実装仕様書で作成済み、本書では変更しない）

```
internal/student_import/
├── domain/
│   ├── entity/            # import_history.go, import_error.go, imported_student.go は変更しない
│   ├── valueobject/
│   ├── repository/        # import_history_repository.go, imported_student_repository.go は本書でメソッド追加（後述）
│   ├── service/
│   ├── event/
│   └── errors/
├── application/
│   ├── dto/
│   └── usecase/           # errors.go の ErrImportHistoryNotFound を再利用する
├── infrastructure/
│   ├── persistence/gorm/
│   ├── repository/        # import_history_repository.go, imported_student_repository.go は本書でメソッド実装追加
│   ├── storage/
│   └── queue/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

## 本書が追加するファイル一覧

```
internal/student_import/domain/valueobject/student_import_history_search_condition.go
internal/student_import/domain/valueobject/student_code_csv_row.go
internal/student_import/domain/service/csv_injection_guard.go
internal/student_import/application/dto/search_student_import_histories.go
internal/student_import/application/dto/show_student_import_history.go
internal/student_import/application/dto/export_student_code_csv.go
internal/student_import/application/usecase/search_student_import_histories_usecase.go
internal/student_import/application/usecase/show_student_import_history_usecase.go
internal/student_import/application/usecase/export_student_code_csv_usecase.go
internal/student_import/presentation/handler/student_import_history_handler.go
internal/student_import/presentation/request/search_student_import_histories_request.go
internal/student_import/presentation/response/student_import_history_response.go
```

`presentation/routes.go`は生徒CSVインポート機能_Go実装仕様書で作成済みのファイルであり、本書ではそこへ`StudentImportHistoryHandler`のルート登録を追加する（新規ファイルとしては作成しない。6章「Routing」参照）。`internal/student_import`の組み立て関数（`NewContext`。アーキテクチャ規約「14. 依存関係の組み立て（DI配線）」）にも、本書の3 UseCaseと`StudentImportHistoryHandler`の生成を追加する。

## 本書が変更（メソッド追加）する既存ファイル

|ファイル|追加内容|
|-|-|
|`domain/repository/import_history_repository.go`|`ImportHistoryRepository` interfaceへ`Search` / `FindOwnedByID`メソッドを追加する（本書「3. Domain層設計」参照）|
|`domain/repository/imported_student_repository.go`|`ImportedStudentRepository` interfaceへ`FindViewsByImportHistoryID`メソッドと、その戻り値型`ImportedStudentView`を追加する（本書「3. Domain層設計」参照）|
|`infrastructure/repository/import_history_repository.go`|上記追加メソッドの実装を追加する（本書「5. Infrastructure層設計」参照）|
|`infrastructure/repository/imported_student_repository.go`|上記追加メソッドの実装を追加する（本書「5. Infrastructure層設計」参照）|

`domain/repository/import_error_repository.go`（`ImportErrorRepository`）は、生徒CSVインポート機能_Go実装仕様書が既に定義した`FindByImportHistoryID`メソッドをそのまま利用し、変更を加えない（②「11. Repository設計」の「ImportErrorRepository（拡張部分）: 生徒CSVインポート機能で定義済みの取得をそのまま利用し、本書での追加はない」に対応）。

---

# 3. Domain層設計

## Entity（再利用、本書では再定義しない）

### ImportHistory / ImportError / ImportedStudent

生徒CSVインポート機能_Go実装仕様書「3. Domain層設計」の`ImportHistory` / `ImportError` / `ImportedStudent` struct定義（フィールド・メソッド・不変条件）をそのまま再利用する。本書はこれらのEntityに新たなフィールド・メソッドを追加しない。本書が用いるフィールドは、`ImportHistory`の`ID` / `TeacherID` / `ImportType` / `Mode` / `Status` / `FileName` / `TotalCount` / `SuccessCount` / `ErrorCount` / `StartedAt` / `FinishedAt` / `CreatedAt`、`ImportError`の`RowNumber` / `Message`である。

## 参照用の構成データ（Read Model、本書が定義する）

### ImportedStudentView

- struct名: `ImportedStudentView`（配置: `domain/repository/imported_student_repository.go`。②「6. Entity設計」は本データを「参照用の構成データ」であり、Entityではないと位置づけ、application層の出力として分離するとしている。ただし、`ImportedStudentRepository`（Domain層のinterface）の戻り値型はDomain層で定義する必要がある（アーキテクチャ規約「2. レイヤー責務と依存方向」：DomainはApplicationに依存しない）ため、Entityの`entity`パッケージには置かず、Repository interfaceと同じファイルに参照専用の戻り値型として定義する。Entity本体に他Contextの表示情報を持たせないという②の判断は、この配置でも保たれる。**②からの補足**）
- 保持するフィールドと型:

|フィールド|型|意味|
|-|-|-|
|`StudentID`|`uint`|インポートされた生徒のユーザーID|
|`Name`|`string`|氏名（参照時点の内容）|
|`NameKana`|`string`|氏名カナ（参照時点の内容）|
|`Email`|`string`|メールアドレス（参照時点の内容。詳細レスポンスにのみ用い、CSVには出力しない）|
|`StudentNumber`|`string`|生徒コード（`users.student_number`）|
|`GradeDisplayName`|`*string`|学年の表示名（共通マスタ参照機能が定める表記。学年が設定されていない場合は`nil`）|
|`SchoolClassName`|`*string`|学級名（学級未所属の場合は`nil`）|
|`Action`|`valueobject.ImportedStudentAction`|新規作成（created）か既存生徒の更新（updated）か|

### StudentImportHistoryDetail

- struct名: `StudentImportHistoryDetail`（配置: `application/dto`。理由は②「6. Entity設計」の「詳細は、ImportHistory Entityに、ImportError一覧とImportedStudentView一覧を組み合わせて構成する（application層のDTOとして構成する）」による）
- 保持するフィールドと型: `History *entity.ImportHistory`, `Students []repository.ImportedStudentView`, `Errors []ImportErrorDetail`
- `ImportErrorDetail`のフィールド: `RowNumber int`, `Message string`

一覧は、ImportHistory Entityをそのまま結果に含めて返す（結合結果を持たないため、一覧用の参照用データは設けない。②「6. Entity設計」）。

## Value Object（本書が定義する）

### StudentImportHistorySearchCondition（`domain/valueobject/student_import_history_search_condition.go`）

- struct名: `StudentImportHistorySearchCondition`
- 保持するフィールドと型: `TeacherID uint`, `Status *string`, `From *time.Time`, `To *time.Time`, `Sort string`, `Order string`
  - `Status`は、許可値（`pending`/`processing`/`completed`/`failed`）のいずれかを`*string`で保持する（許可値以外は`nil`）。生徒CSVインポート機能のImportStatus（`processing`/`completed`/`failed`）は`pending`を持たないため、絞り込み条件専用に許可値を保持する（②「7. Value Object設計」。`pending`は生徒インポートの履歴では発生しないため、指定した場合は常に0件になる）
  - `From`/`To`は、展開後の期間（開始日は当日0時、終了日は当日23時59分59秒。Asia/Tokyoの日境界。下記）を保持する
- 入力用の型: `StudentImportHistorySearchParams`（同ファイルに定義。`Status string`, `From string`, `To string`, `Sort string`, `Order string`。正規化前の生値をそのまま保持する。Domain層がApplication層のDTOに依存しないよう、Domain側に定義する。**②からの補足**）
- 生成時に検証するルール（②「7. Value Object設計」）:
  - `TeacherID`は0を許容しない。0の場合は生成に失敗する（呼び出し側の不具合であり、業務上のエラーではないため、Domain Errorではなく通常の`error`として返す。UseCaseはこれを想定外のエラーとして500に変換する。②「17. Error設計」の「Domain Errorを定義しない」とは矛盾しない）。これにより、実行者を指定せずに検索条件を生成できない
  - 検索対象は常に`import_type`が生徒インポートである履歴、かつ論理削除されていない履歴に限定する（この条件はVO自体のフィールドではなく、後述する`ImportHistoryRepository.Search` / `FindOwnedByID`が常に付加する固定条件として扱う。実行者の限定は`TeacherID`が担う）
  - `status`は許可値以外が指定された場合、絞り込み条件として適用しない（`Status`を`nil`のまま保持する）
  - `sort`は許可値（`created_at`/`total_count`/`success_count`/`error_count`/`status`）以外の場合`created_at`にフォールバックし、同一条件時は`id`の降順を副次キーとする
  - `order`は`asc`/`desc`以外の場合`desc`にフォールバックする
  - `from`/`to`は`YYYY-MM-DD`形式の日付として解釈し、開始日は当日0時、終了日は当日23時59分59秒までを含む期間へ展開する。日境界はRails現行のアプリケーションタイムゾーン（Asia/Tokyo。Rails現行の`config/application.rb`）で展開し、検索時はUTCへ変換して用いる（コーディング規約「17. `time.Time`とタイムゾーン」）。日付として解釈できない値は指定されなかったものとして扱い、エラーにしない（Rails現行どおり。Railsの`Date.parse`は`YYYY-MM-DD`以外の書式も解釈するが、フロントエンドが送る形式は`YYYY-MM-DD`と推測し、本書では`YYYY-MM-DD`のみを解釈する。推測）
- 公開するmethod一覧: `NewStudentImportHistorySearchCondition(teacherID uint, params StudentImportHistorySearchParams) (StudentImportHistorySearchCondition, error)`（正規化を行うファクトリ）、`(c StudentImportHistorySearchCondition) HasStatus() bool`

### StudentCodeCsvRow（`domain/valueobject/student_code_csv_row.go`）

- struct名: `StudentCodeCsvRow`
- 保持するフィールドと型: `Name string`, `NameKana string`, `Grade string`, `SchoolClass string`, `StudentNumber string`
- 生成時に検証するルール: なし（値の組み合わせ自体に不変条件はなく、生成時に`CsvInjectionGuard`によるエスケープを適用済みの`Name`・`NameKana`・`Grade`・`SchoolClass`を保持する。`StudentNumber`はシステムが発行する値であるためエスケープしない。学年・学級が`nil`の場合は空文字とする。メールアドレスは保持しない）
- 公開するmethod一覧: `NewStudentCodeCsvRow(name, nameKana, grade, schoolClass, studentNumber string) StudentCodeCsvRow`（引数は`CsvInjectionGuard`適用済みの値。適用はUseCaseが行う）、`(r StudentCodeCsvRow) Fields() []string`（CSVの1行を、列の順（氏名・氏名カナ・学年・学級・生徒コード）で返す）

## Repository Interface

### ImportHistoryRepository（既存interfaceへのメソッド追加）

生徒CSVインポート機能_Go実装仕様書が定義した`ImportHistoryRepository` interface（`Create` / `Update` / `FindByID`）に、以下のメソッドを追加する（同一ファイル`domain/repository/import_history_repository.go`内、同一interfaceへメソッドを追加）。

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`Search`|`(ctx context.Context, condition valueobject.StudentImportHistorySearchCondition, page, perPage int)`|`([]*entity.ImportHistory, int64, error)`|`StudentImportHistorySearchCondition`に基づく複合検索・並び替え・ページング付きの一覧取得。第2戻り値は、ページングを適用する前の総件数（②「11. Repository設計」の「拡張部分」）|
|`FindOwnedByID`|`(ctx context.Context, id uint, teacherID uint)`|`(*entity.ImportHistory, error)`|IDによる詳細取得。実行者が`teacherID`であり、`import_type`が生徒インポートであり、論理削除されていない履歴のみ取得する。該当しない場合（存在しない・他の教師の履歴・問題インポートの履歴・論理削除済み）は`nil, nil`を返す（`FindByID`と同じ規約。②「16. Authorization設計」の「範囲外は存在しないものとして扱う」に対応）|

**②からの補足**: 既存の`FindByID`は、非同期ワーカー（`ExecuteStudentImportUseCase`）がID指定で履歴を取得するためのものであり、実行者・種別・論理削除の限定を持たない。教師向けの参照に`FindByID`を流用すると、他の教師の履歴が取得できてしまうため、実行者の限定を引数に含む`FindOwnedByID`を別メソッドとして追加した（`FindByID`は変更しない）。

### ImportedStudentRepository（既存interfaceへのメソッド追加）

生徒CSVインポート機能_Go実装仕様書が定義した`ImportedStudentRepository` interface（`CreateBatch` / `FindByImportHistoryID`）に、以下のメソッドを追加する。

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindViewsByImportHistoryID`|`(ctx context.Context, importHistoryID uint)`|`([]ImportedStudentView, error)`|ImportHistory IDによる成功行一覧を、参照時点の生徒の表示情報（氏名・氏名カナ・メールアドレス・生徒コード・学年の表示名・学級名）を付与した参照用データとして、記録された順（ID昇順）に取得する。詳細取得・CSVエクスポートの両方で利用する（②「11. Repository設計」）|

### ImportErrorRepository（既存interfaceをそのまま再利用）

生徒CSVインポート機能_Go実装仕様書が定義した`ImportErrorRepository.FindByImportHistoryID`をそのまま利用する。本書での変更はない。

## Domain Service（本書が定義する）

### CsvInjectionGuard（`domain/service/csv_injection_guard.go`）

- struct/interface名: `CsvInjectionGuard`（struct、ステートレス）
- メソッドシグネチャ: `Escape(value string) string`
- 責務: CSV出力対象の文字列（氏名・氏名カナ・学年・学級）が表計算ソフトでの数式として解釈されうる先頭文字（`=`、`+`、`-`、`@`）を持つ場合に、`'`を付与してエスケープする。空文字はそのまま返す（②「8. Domain Service」）
- 補足: 管理者インポート履歴管理機能_Go実装仕様書（question-import）にも同名のDomain Serviceがあるが、Context間で相手の内部の実装に依存しない（アーキテクチャ規約「5. Context間連携ルール」）ため、本Contextに別途定義する。規則が食い違わないよう、両Contextのテストで同一のケースを検証する（②「8. Domain Service」）

## Domain Error

本書では新たなDomain Errorを追加しない（②「17. Error設計」のとおり、不正な検索条件値は無視・フォールバックで吸収する）。

## Application Error（再利用）

`ErrImportHistoryNotFound`（`application/usecase/errors.go`。生徒CSVインポート機能_Go実装仕様書が`ExecuteStudentImportUseCase`の発生しうるApplication Errorとして定義済み）を、本機能でも再利用する（新たに別名のエラー変数を追加しない）。存在しない履歴、他の教師の履歴、問題インポートの履歴、論理削除済みの履歴は、②「17. Error設計」のとおり区別せず、同一のエラーとして扱う。生徒CSVインポート機能③ではワーカー内部エラー（HTTPステータスへの直接変換なし）として扱われているが、本機能ではHTTP 404に対応するAppError（`StatusCode()`が404）として利用する。**②からの補足**

---

# 4. Application層設計

## DTO（Command / Query）

- struct名: `SearchStudentImportHistoriesQuery`（配置: `application/dto/search_student_import_histories.go`）
  - フィールド: `CurrentTeacherID uint`, `Status string`, `From string`, `To string`, `Sort string`, `Order string`, `Page int`, `PerPage int`（`Page`/`PerPage`はHandlerが既定値・上限を適用済みの値。それ以外は正規化前の生値）
  - 区分: Query
- struct名: `SearchStudentImportHistoriesResult`
  - フィールド: `Items []*entity.ImportHistory`, `Page int`, `PerPage int`, `TotalCount int64`
  - 区分: UseCase出力
- struct名: `ShowStudentImportHistoryQuery`（配置: `application/dto/show_student_import_history.go`）
  - フィールド: `CurrentTeacherID uint`, `ImportHistoryID uint`
  - 区分: Query
- struct名: `ExportStudentCodeCsvQuery`（配置: `application/dto/export_student_code_csv.go`）
  - フィールド: `CurrentTeacherID uint`, `ImportHistoryID uint`
  - 区分: Query
- struct名: `ExportStudentCodeCsvResult`
  - フィールド: `FileName string`（`import_history_<id>.csv`）, `Content []byte`
  - 区分: UseCase出力（②「19. API仕様」の「Response: `import_history_<id>.csv`（`text/csv; charset=UTF-8`、BOM付き）」に対応）

## UseCase

### SearchStudentImportHistoriesUseCase（`application/usecase/search_student_import_histories_usecase.go`）

- struct名: `SearchStudentImportHistoriesUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository`（生徒CSVインポート機能_Go実装仕様書で定義済みのInterfaceを、本書が追加した`Search`メソッドとあわせて利用する）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.SearchStudentImportHistoriesQuery) (dto.SearchStudentImportHistoriesResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `valueobject.NewStudentImportHistorySearchCondition(query.CurrentTeacherID, valueobject.StudentImportHistorySearchParams{...})`で検索条件を正規化する（実行者の設定・許可値判定・フォールバック・期間展開。②「13. シーケンス図」の「StudentImportHistorySearchConditionへ正規化」）。生成に失敗した場合（`CurrentTeacherID`が0）は、想定外のエラーとして返す
  2. `ImportHistoryRepository.Search(ctx, condition, query.Page, query.PerPage)`を呼び出し、対象履歴一覧と総件数を取得する
  3. `dto.SearchStudentImportHistoriesResult`として返す
- トランザクション境界: なし（②「14. Transaction設計」により読み取りのみ）
- 発生しうるApplication Error: なし（Infrastructure Errorのみ）

### ShowStudentImportHistoryUseCase（`application/usecase/show_student_import_history_usecase.go`）

- struct名: `ShowStudentImportHistoryUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository`, `ImportedStudentRepository`, `ImportErrorRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ShowStudentImportHistoryQuery) (dto.StudentImportHistoryDetail, error)`
- 処理ステップ:
  1. `ImportHistoryRepository.FindOwnedByID(ctx, query.ImportHistoryID, query.CurrentTeacherID)`で対象履歴（実行者本人の生徒インポートの履歴に限定済み）を取得する
  2. 取得できない場合（`nil, nil`）、`ErrImportHistoryNotFound`を返す（②「17. Error設計」）
  3. `ImportedStudentRepository.FindViewsByImportHistoryID(ctx, query.ImportHistoryID)`で成功行一覧（生徒の表示情報付き、記録順）を取得する
  4. `ImportErrorRepository.FindByImportHistoryID(ctx, query.ImportHistoryID)`でエラー明細一覧（行番号順）を取得し、`ImportErrorDetail`へ変換する
  5. `dto.StudentImportHistoryDetail`を組み立てて返す（`Students`・`Errors`は該当がない場合も`nil`ではなく空のスライスとする）
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrImportHistoryNotFound`

### ExportStudentCodeCsvUseCase（`application/usecase/export_student_code_csv_usecase.go`）

- struct名: `ExportStudentCodeCsvUseCase`
- コンストラクタが受け取る依存: `ImportHistoryRepository`, `ImportedStudentRepository`, `service.CsvInjectionGuard`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ExportStudentCodeCsvQuery) (dto.ExportStudentCodeCsvResult, error)`
- 処理ステップ（②「13. シーケンス図（生徒コード配布用CSVエクスポート）」の記載どおり）:
  1. `ImportHistoryRepository.FindOwnedByID`で対象履歴を取得する（実行者本人の生徒インポートであることの確認を兼ねる）
  2. 取得できない場合、`ErrImportHistoryNotFound`を返す
  3. `ImportedStudentRepository.FindViewsByImportHistoryID`で成功行一覧（生徒の表示情報付き、記録順）を取得する
  4. 各生徒について、氏名・氏名カナ・学年の表示名・学級名（`nil`は空文字）に`CsvInjectionGuard.Escape`を適用し、生徒コードはそのままで`valueobject.NewStudentCodeCsvRow`を生成する
  5. ヘッダー行（氏名・氏名カナ・学年・学級・生徒コード）と各行を、標準の`encoding/csv`（改行はLF）で書き出し、先頭にBOMを付与したCSVコンテンツを組み立てる（サマリー行は出力しない。メールアドレスは含めない。②「19. API仕様」）
  6. `dto.ExportStudentCodeCsvResult`（`FileName`は`import_history_<履歴ID>.csv`）として返す
- トランザクション境界: なし（②「14. Transaction設計」によりデータ変更を伴わないため）
- 発生しうるApplication Error: `ErrImportHistoryNotFound`

---

# 5. Infrastructure層設計

## Repository実装（既存実装へのメソッド追加）

### ImportHistoryRepository実装（`infrastructure/repository/import_history_repository.go`、既存実装へのメソッド追加）

生徒CSVインポート機能_Go実装仕様書が定義した実装struct（`gormrepo`パッケージ内）に、以下のメソッド実装を追加する。

|メソッド|発行するクエリ内容|
|-|-|
|`Search`|`user_id`が`condition.TeacherID`に一致し、`import_type`が生徒インポートで、`deleted_at IS NULL`の条件を常に付加し、`StudentImportHistorySearchCondition`の`Status`（`nil`でなければ一致条件。DBの値表現への変換は`ImportHistoryModel`の変換方針に従う）/`From`〜`To`（`created_at`の範囲）で絞り込む。`Sort`/`Order`（許可値をカラム名へ対応づけたマップで解決し、文字列連結でカラム名を組み立てない。同条件時は`id`降順を副次キー）でソートし、`page`/`perPage`でOFFSET/LIMITページングする。総件数は同条件でのCOUNTを別途取得する（②「21. DB操作仕様」）。`users`等との結合は行わない|
|`FindOwnedByID`|`id`一致かつ`user_id`が`teacherID`に一致かつ`import_type`が生徒インポートかつ`deleted_at IS NULL`の条件で1件取得する。見つからない場合は`nil, nil`を返す|

### ImportedStudentRepository実装（`infrastructure/repository/imported_student_repository.go`、既存実装へのメソッド追加）

生徒CSVインポート機能_Go実装仕様書が定義した実装struct（`gormrepo`パッケージ内）に、以下のメソッド実装を追加する。

|メソッド|発行するクエリ内容|
|-|-|
|`FindViewsByImportHistoryID`|`imported_students`から`import_history_id`一致で取得し、`users`（`imported_students.user_id`）と内部結合して氏名・氏名カナ・メールアドレス・生徒コードを、`grades`（`users.grade_id`）と`school_classes`（`users.school_class_id`）と外部結合して学年・学級名を取得する（学級・学年が未設定の生徒を除外しない）。`imported_students.id`昇順で全件取得し、生徒ごとの個別問い合わせは行わない。結合結果は、実装内に非公開の取得用struct（`users`・`grades`・`school_classes`の必要な列のみ）へ受け、`ImportedStudentView`へ変換する。学年の表示名は、共通マスタ参照機能が定める`year`から表示名への導出規則で変換する（**②からの補足**: 導出の呼び出し方式は共通マスタ参照機能③の公開手段に従う。未確定の場合の扱いは「14. ②からの補足事項」参照）|

これらの結合対象（`users`・`grades`・`school_classes`）は、他Contextが所有するテーブルであり、本Contextでは新たな所有モデルとして定義しない参照専用の取得である（生徒CSVインポート機能_Go実装仕様書「15. GORM / DBクエリ設計」の`GradeClassResolutionRepository`・`StudentAccountRepository`と同じ扱い。②「3. Bounded Context」の「自前の参照モデル」）。

### ImportErrorRepository実装（既存実装を再利用、変更なし）

生徒CSVインポート機能_Go実装仕様書が定義した`FindByImportHistoryID`をそのまま利用する。

## 外部連携実装

対象外。②に本機能でのMail・Cache・Queue連携要件の記載はない（CSVエクスポートはHTTPレスポンスとして同期的に返却するのみで、非同期ジョブ化は行わない）。

---

# 6. Presentation層設計

## Handler

### StudentImportHistoryHandler（`presentation/handler/student_import_history_handler.go`）

- struct名: `StudentImportHistoryHandler`
- 対応する呼び出し先: `SearchStudentImportHistoriesUseCase`, `ShowStudentImportHistoryUseCase`, `ExportStudentCodeCsvUseCase`
- メソッド一覧:
  - `Search(c *gin.Context)`: `GET /api/v1/teacher/import_histories`に対応
  - `Show(c *gin.Context)`: `GET /api/v1/teacher/import_histories/:id`に対応
  - `Export(c *gin.Context)`: `GET /api/v1/teacher/import_histories/:id/export`に対応
- 処理順序:
  - `Search`: クエリパラメータ（`status`, `from`, `to`, `sort`, `order`, `page`, `per_page`）を`request.SearchStudentImportHistoriesRequest`にバインド → `page`/`per_page`に既定値・上限を適用（下記） → Middlewareが確定したcurrent teacherのIDをcontextから取得 → `dto.SearchStudentImportHistoriesQuery`へ変換して`SearchStudentImportHistoriesUseCase.Execute`を呼び出す → `response.StudentImportHistoryListResponse`へ変換し200で返す
  - `Show`: パスパラメータ`id`を整数として解釈（整数でない・0以下の場合は、Rails現行の挙動に合わせて存在しない履歴として扱い、`ErrImportHistoryNotFound`として404へ。**②からの補足**） → current teacherのIDをcontextから取得 → `ShowStudentImportHistoryUseCase.Execute`を呼び出す → `ErrImportHistoryNotFound`の場合は404に変換（集中エラーハンドリングミドルウェアへ`c.Error(err)`で登録） → 成功時は`response.StudentImportHistoryDetailResponse`へ変換し200で返す
  - `Export`: パスパラメータ`id`を`Show`と同様に解釈 → current teacherのIDをcontextから取得 → `ExportStudentCodeCsvUseCase.Execute`を呼び出す → `ErrImportHistoryNotFound`の場合は404に変換 → 成功時は`Content-Type: text/csv; charset=UTF-8`、`Content-Disposition: attachment; filename="import_history_<id>.csv"`のレスポンスとして200で返す（Response DTO structは設けず、`c.Data`相当の直接書き込みとする）
- `page`/`per_page`の既定値と上限: `page`は未指定・数値でない値・0以下の場合は1とする。`per_page`は未指定・数値でない値・0以下の場合は20、100を超える場合は100に丸める。いずれもエラーにしない（Rails現行の`sanitized_page` / `sanitized_per_page`と同一。②「15. Validation設計」。アーキテクチャ規約「15. API規約」の「ページネーション」）

## Request / Response DTO

- struct名: `SearchStudentImportHistoriesRequest`（`presentation/request/search_student_import_histories_request.go`）
  - フィールド: `Status string`（`form:"status"`）, `From string`（`form:"from"`）, `To string`（`form:"to"`）, `Sort string`（`form:"sort"`）, `Order string`（`form:"order"`）, `Page string`（`form:"page"`）, `PerPage string`（`form:"per_page"`）
  - バリデーションタグ／チェック内容: すべて文字列として受け取り、バインド時のエラーにしない（②「15. Validation設計」のとおり、不正な値はエラーにせず無視・既定値とするため）。`page`/`per_page`の解釈はHandlerが行う。`status`/`from`/`to`/`sort`/`order`の許可値・形式の判定はDomain層（`StudentImportHistorySearchCondition`）に委ねる。`unit_id`/`course_id`/`user_id`は受け付けず、指定されても無視する（②「19. API仕様」）
- struct名: `StudentImportHistoryListItemResponse`（`presentation/response/student_import_history_response.go`）
  - フィールド: `ID uint`（json: `id`）, `FileName string`（json: `file_name`）, `Status string`（json: `status`）, `Mode string`（json: `mode`）, `TotalCount int`（json: `total_count`）, `SuccessCount int`（json: `success_count`）, `ErrorCount int`（json: `error_count`）, `CreatedAt time.Time`（json: `created_at`）
- struct名: `MetaResponse`
  - フィールド: `CurrentPage int`（json: `current_page`）, `TotalPages int`（json: `total_pages`。総件数が0の場合は0）, `TotalCount int64`（json: `total_count`）, `PerPage int`（json: `per_page`）
- struct名: `StudentImportHistoryListResponse`
  - フィールド: `ImportHistories []StudentImportHistoryListItemResponse`（json: `import_histories`）, `Meta MetaResponse`（json: `meta`）
- struct名: `ImportedStudentResponse`
  - フィールド: `ID uint`（json: `id`）, `Name string`（json: `name`）, `NameKana string`（json: `name_kana`）, `Email string`（json: `email`）, `StudentNumber string`（json: `student_number`）, `Grade *string`（json: `grade`）, `SchoolClass *string`（json: `school_class`。学級未所属の場合は`null`）, `Action string`（json: `action`）
- struct名: `ImportErrorResponse`
  - フィールド: `RowNumber int`（json: `row_number`）, `Message string`（json: `message`）
- struct名: `StudentImportHistoryDetailResponse`
  - フィールド: `ID uint`, `FileName string`, `Status string`, `Mode string`, `TotalCount int`, `SuccessCount int`, `ErrorCount int`, `StartedAt *time.Time`（json: `started_at`）, `FinishedAt *time.Time`（json: `finished_at`）, `CreatedAt time.Time`, `Students []ImportedStudentResponse`（json: `students`。該当がない場合も空配列）, `Errors []ImportErrorResponse`（json: `errors`。該当がない場合も空配列）。管理者向けの詳細と異なり`warnings`は含まない（②「19. API仕様」）

CSVエクスポートのResponse DTOは設けない（ファイルストリームを直接返すため）。

## Routing

`presentation/routes.go`（生徒CSVインポート機能_Go実装仕様書で作成済みのファイル）の、`teacher`ロールの認証・認可Middlewareを適用した`/api/v1/teacher`のルートグループへ、以下のルート登録を追加する。

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/teacher/import_histories|StudentImportHistoryHandler.Search|
|GET|/api/v1/teacher/import_histories/:id|StudentImportHistoryHandler.Show|
|GET|/api/v1/teacher/import_histories/:id/export|StudentImportHistoryHandler.Export|

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/teacher/import_histories|StudentImportHistoryHandler.Search|SearchStudentImportHistoriesRequest|StudentImportHistoryListResponse|200|
|GET|/api/v1/teacher/import_histories/:id|StudentImportHistoryHandler.Show|-（パスパラメータのみ）|StudentImportHistoryDetailResponse|200|
|GET|/api/v1/teacher/import_histories/:id/export|StudentImportHistoryHandler.Export|-（パスパラメータのみ）|CSVファイル（`import_history_<id>.csv`）|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（Middleware。Rails現行仕様書に記載のとおり）|
|teacherロールでない|403|認可エラー（Middleware。Rails現行仕様書に記載のとおり）|
|対象インポート履歴が存在しない、他の教師が実行した履歴である、問題インポートの履歴である、または論理削除済みである（詳細・エクスポート）|404|`ErrImportHistoryNotFound`（②「17. Error設計」）|
|`id`が整数でない（詳細・エクスポート）|404|存在しない履歴として扱う（Rails現行の挙動。「6. Presentation層設計」参照）|
|`status`/`sort`/`order`が許可値でない、`from`/`to`が日付として解釈できない、`page`/`per_page`が不正|200|エラーにしない（無視または既定値で処理する）|
|DB接続失敗・CSV生成失敗|500|Infrastructure Error|

---

# 8. Transaction実装方針

②「14. Transaction設計」のとおり、本機能はすべて読み取り専用処理であり、明示的なトランザクションを使用しない。

- Transaction開始箇所: なし
- Transaction終了箇所: 該当なし
- 複数Repositoryにまたがる場合の扱い: `ShowStudentImportHistoryUseCase`は`ImportHistoryRepository`・`ImportedStudentRepository`・`ImportErrorRepository`を、`ExportStudentCodeCsvUseCase`は`ImportHistoryRepository`・`ImportedStudentRepository`を呼び出すが、いずれも読み取りのみであり、`TransactionManager`（アーキテクチャ規約「11. Transaction実装パターン」）を介したトランザクション制御は行わない（②「14. Transaction設計」）

---

# 9. Validation実装方針

## Presentation

- `SearchStudentImportHistoriesRequest`でのチェック内容: なし（すべて文字列として受け取り、エラーにしない）。`page`/`per_page`の既定値・上限の適用のみHandlerが行う（②「15. Validation設計」）
- `id`（詳細・エクスポート）はパスパラメータとして必須。ルーティング定義上、値が欠落するとGinのルーティング自体が一致しない。整数でない値は、Handlerで存在しない履歴として404に変換する

## 業務ルール検証

Domain Model採用のため、②の記載どおりEntity／Value Object生成時に検証する。

- `StudentImportHistorySearchCondition`: `status`が許可値でない場合は絞り込み条件として無視する。`sort`が許可値でない場合は`created_at`にフォールバックする。`order`が`asc`/`desc`でない場合は`desc`にフォールバックする。`from`/`to`は日付として解釈できない場合は無視し、解釈できる場合は日境界（0時〜23:59:59）へ展開する。実行者（`TeacherID`）を必須とし、検索対象を実行者本人の生徒インポートの履歴に限定する（②「15. Validation設計」の「Domain」節）
- UseCase内で判定する業務ルール: `ShowStudentImportHistoryUseCase`・`ExportStudentCodeCsvUseCase`は、対象履歴が存在し、かつ実行者本人の生徒インポートの履歴であることを`ImportHistoryRepository.FindOwnedByID`の結果（取得できるか否か）で確認する

## 責務分離

②「15. Validation設計」の記載どおり、Presentationは「入力が正しいか（型・形式）」を担当し、Domainは「検索条件として妥当か（許可値・対象範囲）」を担当する。不正な値はエラーとせず無視・フォールバックすることで、Rails現行仕様の寛容な挙動を維持する（②の記載どおり、本機能ではDomain Errorを定義しない）。

---

# 10. Authorization実装方針

## Middlewareで行う処理

- 認証済みユーザーを特定し、teacherロールであることを確認する（②「16. Authorization設計」。Casbin RBAC。アーキテクチャ規約「7. 横断的関心事の置き場所」）

## Handlerで行う処理

- 認証失敗時のレスポンス整形。業務権限の判定は持たせない。他の教師の操作を許可する権限（他職員操作権限）や担当学年による範囲制限は要求しない（②「16. Authorization設計」）
- Middlewareが確定したcurrent teacherのIDをcontextから取得し、UseCaseへ渡す（コーディング規約「20. context.Context」に従う）

## UseCaseで行う処理

- current teacherが実行した生徒インポートの履歴のみを参照範囲とする。`SearchStudentImportHistoriesUseCase`は`CurrentTeacherID`を検索条件（`StudentImportHistorySearchCondition.TeacherID`）へ、`ShowStudentImportHistoryUseCase`・`ExportStudentCodeCsvUseCase`は`ImportHistoryRepository.FindOwnedByID`の`teacherID`引数へ、必ず渡す
- 範囲外（存在しない・他の教師の履歴・問題インポートの履歴・論理削除済み）は`ErrImportHistoryNotFound`（404）とし、他の教師の履歴の存在を示さない

## Domainで行う処理

- `StudentImportHistorySearchCondition`が「実行者本人（`TeacherID`は必須）・生徒インポート・未削除」という範囲の条件を常に内包する（②「16. Authorization設計」の「Domain」節）

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

②「17. Error設計」のとおり、本機能では不正な検索条件値をエラーとせず無視・フォールバックで吸収するため、Domain Errorは定義しない。対象履歴の不存在・範囲外のみをApplication Error（`ErrImportHistoryNotFound`、生徒CSVインポート機能_Go実装仕様書で定義済みのものを再利用）として扱う。`ErrImportHistoryNotFound`は、アーキテクチャ規約「12. Error変換パターン（AppError）」の`AppError`を実装し、`StatusCode()`が404を返す。

## Application Error → HTTPレスポンスへの変換方針

Gin規約「8. エラーハンドリングミドルウェア」の集中エラーハンドリングミドルウェアで変換する。Handlerは`c.Error(err)`でエラーを登録するのみとする。

|業務シナリオ|Error変数名／型|発生層|HTTP Status|
|-|-|-|-|
|対象履歴が存在しない・他の教師の履歴・問題インポートの履歴・論理削除済み|`ErrImportHistoryNotFound`|Application（UseCase）|404|
|未認証|-|Middleware|401|
|teacherロールでない|-|Middleware|403|
|検索条件の生成失敗（`CurrentTeacherID`が0。呼び出し側の不具合）|通常の`error`|Application（UseCase）|500|
|DB接続失敗・CSV生成失敗|Infrastructure Error|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

Repository実装が返すDBエラー（接続失敗・クエリ失敗）は`fmt.Errorf`でラップしてUseCase・Handlerへ伝播させ、`AppError`に変換されていない想定外のエラーとして一律500に変換する（アーキテクチャ規約「12. Error変換パターン（AppError）」）。

---

# 12. GORM / DBクエリ設計

②「20. DB設計方針」のとおり、`import_histories` / `import_errors` / `imported_students`テーブルは生徒CSVインポート機能と共有し、本機能単独でのスキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

- `gormmodel.ImportHistoryModel` ⇔ `import_histories`テーブル（生徒CSVインポート機能_Go実装仕様書で定義済みのモデルをそのまま利用する。新規モデル定義は行わない）
- `gormmodel.ImportErrorModel` ⇔ `import_errors`テーブル（同上）
- `gormmodel.ImportedStudentModel` ⇔ `imported_students`テーブル（同上）
- `users`・`grades`・`school_classes`は参照専用として、`FindViewsByImportHistoryID`の実装内に非公開の取得用structを定義し、必要な列のみを受ける（他Contextのモデル定義には依存しない。「5. Infrastructure層設計」参照）

## 主要クエリの条件・ソート・ページネーション方針

|Repository|メソッド|対象テーブル|条件|結合|
|-|-|-|-|-|
|`ImportHistoryRepository`|`Search`|`import_histories`|`user_id`一致（実行者）、`import_type`が生徒インポート、`deleted_at IS NULL`、`status`（許可値のみ）、`created_at`の期間（日境界展開後、UTCに変換）。`sort`（`created_at`/`total_count`/`success_count`/`error_count`/`status`、既定`created_at`）・`order`（既定`desc`）、副次キー`id`降順。`page`/`perPage`（既定20件、上限100件）のOFFSET/LIMIT。総件数は別途COUNT|不要|
|`ImportHistoryRepository`|`FindOwnedByID`|`import_histories`|`id`一致、`user_id`一致（実行者）、`import_type`が生徒インポート、`deleted_at IS NULL`|不要|
|`ImportedStudentRepository`|`FindViewsByImportHistoryID`|`imported_students`（参照: `users`, `grades`, `school_classes`）|`import_history_id`一致。`imported_students.id`昇順。ページネーションなし|`users`と内部結合、`grades`・`school_classes`と外部結合|
|`ImportErrorRepository`|`FindByImportHistoryID`（既存）|`import_errors`|`import_history_id`一致、`row_number`昇順|不要|

SQL文そのものは記載しない。

## 既存Schemaに対する変更

②20章のとおり変更なし。

---

# 13. テストケース設計

②「22. テスト戦略」で採用パターンがDomain Modelであるため、区分はそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|`StudentImportHistorySearchCondition`|許可値の`status`が設定されること／許可値でない`status`は絞り込み条件として無視されること（`nil`）／`pending`が許可値として設定されること|
|`StudentImportHistorySearchCondition`|許可値の`sort`がそのまま使用されること／許可値でない`sort`は`created_at`にフォールバックすること|
|`StudentImportHistorySearchCondition`|`order`が`asc`/`desc`以外の場合`desc`にフォールバックすること|
|`StudentImportHistorySearchCondition`|`from`/`to`が日境界（0時〜23:59:59。Asia/Tokyo）へ正しく展開されること／日付として解釈できない値が指定されなかったものとして扱われること（エラーにならないこと）|
|`StudentImportHistorySearchCondition`|`TeacherID`が0の場合に生成が失敗すること／`TeacherID`が条件に保持されること|
|`CsvInjectionGuard.Escape`|`=`/`+`/`-`/`@`で始まる値に`'`が付与されること／該当しない値・空文字はそのまま返ること（管理者インポート履歴管理機能の`CsvInjectionGuard`と同一のケースを含める）|
|`StudentCodeCsvRow`|氏名・氏名カナ・学年・学級にエスケープ済みの値が保持され、生徒コードはエスケープされないこと／`Fields()`が氏名・氏名カナ・学年・学級・生徒コードの順であること／学級が空の場合に空欄となること|

## UseCase Test

|対象|テストケース|
|-|-|
|`SearchStudentImportHistoriesUseCase`|実行者・各検索条件（status/期間）が正しく`Search`へ渡されること／並び替え・ページングが適用されること／`CurrentTeacherID`が0の場合にエラーとなること|
|`ShowStudentImportHistoryUseCase`|存在する履歴IDで履歴・成功行・エラー明細が返ること／成功行・エラー明細がない場合に空のスライスが返ること／存在しない履歴IDで`ErrImportHistoryNotFound`を返すこと／他の教師が実行した履歴のIDを指定した場合に`ErrImportHistoryNotFound`を返すこと（②の重点検証項目）／問題インポートの履歴IDを指定した場合に`ErrImportHistoryNotFound`を返すこと（②の重点検証項目）／論理削除済みの履歴IDで`ErrImportHistoryNotFound`を返すこと|
|`ExportStudentCodeCsvUseCase`|CSVコンテンツがBOM付きで、ヘッダー行（氏名・氏名カナ・学年・学級・生徒コード）＋生徒の行（記録順）の順で構成されること／サマリー行が出力されないこと／メールアドレスが含まれないこと／氏名・氏名カナ・学年・学級に`CsvInjectionGuard`のエスケープが適用され、生徒コードには適用されないこと／学級未所属の生徒の学級が空欄になること／成功行がない履歴でヘッダー行のみのCSVとなること／存在しない・他の教師の・問題インポートの履歴IDで`ErrImportHistoryNotFound`を返すこと|

## Repository Test

|対象|テストケース|
|-|-|
|`ImportHistoryRepository.Search`|実行者本人の履歴のみが返ること（他の教師の履歴が含まれないこと）／`import_type`が生徒インポート以外の履歴が含まれないこと／論理削除済みの履歴が含まれないこと／`status`・期間の絞り込みが正しいこと／並び替え（同条件時は`id`降順）・ページング・総件数が正しいこと／許可外の`sort`が文字列連結でSQLに渡らないこと|
|`ImportHistoryRepository.FindOwnedByID`|実行者本人の生徒インポートの履歴が取得できること／他の教師の履歴・問題インポートの履歴・論理削除済みの履歴・存在しないIDでは`nil, nil`となること|
|`ImportedStudentRepository.FindViewsByImportHistoryID`|生徒の表示情報（氏名・氏名カナ・メールアドレス・生徒コード・学年の表示名・学級名）と`action`が正しく取得されること／記録された順に取得されること／学級未所属の生徒が除外されず`SchoolClassName`が`nil`になること／生徒数によらず問い合わせ数が増えないこと（N+1にならないこと）|
|`ImportErrorRepository.FindByImportHistoryID`|対象履歴に紐づくエラーのみ`row_number`昇順で取得できること|

## Handler Test

|対象|テストケース|
|-|-|
|`StudentImportHistoryHandler.Search`|クエリパラメータの解釈が正しいこと／`from`/`to`・`status`・`sort`・`order`が不正な場合にエラーにならず200が返ること／`page`/`per_page`が不正な場合に1ページ目・20件として扱われ、100件を超える`per_page`が100件に丸められること／`unit_id`等の管理者向けパラメータが無視されること／`meta`に`current_page`・`total_pages`・`total_count`・`per_page`が含まれること|
|`StudentImportHistoryHandler.Show`|存在しない`id`・他の教師の履歴の`id`・整数でない`id`で404が返ること／正常系で200が返り、`students`・`errors`が返ること|
|`StudentImportHistoryHandler.Export`|正常系で200と`Content-Type: text/csv`・`Content-Disposition: attachment; filename="import_history_<id>.csv"`が返ること／存在しない`id`・他の教師の履歴の`id`で404が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/teacher/import_histories`|エンドポイント経由で一覧・詳細・CSVエクスポートが正しく動作すること|
|生徒CSVインポートとの連携|生徒CSVインポートが成功した後に、詳細の`students`とCSVに、新規作成・更新の区別つきで成功行が反映されること／失敗したインポートでは`students`が空で、`errors`にエラー明細が返ること|
|CSVエクスポート|CSV内の数式注入対策エスケープが実際に適用されていること／メールアドレスが含まれないこと（②の記載どおり）|
|認可|teacher以外のロールでアクセスした場合に403が返ること／未認証の場合に401が返ること／他の教師の履歴の一覧・詳細・エクスポートに到達できないこと|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に整理する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|`ImportedStudentView`を、Entityの`entity`パッケージではなく`domain/repository/imported_student_repository.go`（Repository interfaceと同じファイル）に、参照専用の戻り値型として定義した|②「6. Entity設計」は参照用データをapplication層の出力として分離すると定めているが、Domain層のRepository interfaceの戻り値型はDomain層で定義する必要がある（アーキテクチャ規約2章：DomainはApplicationに依存しない）。Entityに他Contextの表示情報を持たせない②の判断は保たれる|②の判断根拠を、依存方向に適合する配置へ具体化したもの（推測ではない）|
|`StudentImportHistorySearchParams`（正規化前の生値の入力型）をDomain層（`domain/valueobject`）に定義した|管理者インポート履歴管理機能_Go実装仕様書は同種の入力型を`application/dto`に定義しているが、VOのファクトリが受け取る型をApplication層に置くと、DomainがApplicationに依存するため|推測|
|検索条件の実行者限定を、`ImportHistoryRepository.Search`の引数（`StudentImportHistorySearchCondition.TeacherID`）と、`FindOwnedByID`という別メソッドで表現した（既存の`FindByID`は流用しない）|②「16. Authorization設計」は「実行者本人への限定」をUseCase・Domainで行うと定めるが、具体的なメソッド設計は②に明記がない。既存`FindByID`は非同期ワーカー用でスコープ限定を持たないため、教師向け参照への流用は他人の履歴を取得できてしまう|推測|
|`from`/`to`を`YYYY-MM-DD`形式のみ解釈し、日境界をAsia/Tokyoで展開した|②は「日付単位、開始日は0時、終了日は23:59:59」とするのみで、書式とタイムゾーンを明記していない。Rails現行のアプリケーションタイムゾーンはAsia/Tokyo（`config/application.rb`）。書式は、Rails現行の`Date.parse`がより多くの書式を解釈するのに対し、フロントエンドが送る形式を`YYYY-MM-DD`と推測した|推測（Asia/Tokyoの日境界は、Rails現行の設定に基づく）|
|`page`/`per_page`のRequestフィールドを文字列で受け取り、Handlerで既定値・上限を適用する|②「15. Validation設計」・規約15章のとおり、不正な値をエラーにしない（Rails現行の`sanitized_page`/`sanitized_per_page`）。整数型でバインドすると数値でない値でバインドエラーになるため|推測（②の方針の実装方法）|
|`id`が整数でない場合を404として扱う|②「15. Validation設計」は`import_history_id`の整数形式を検証し、エラーメッセージ方針を示すが、HTTPステータスを明記していない。Rails現行は、存在しない履歴として404になる（`find`）|推測|
|`ErrImportHistoryNotFound`を、生徒CSVインポート機能③がワーカー内部エラーとして定義したものと同一のエラー変数として再利用し、`StatusCode()`を404とした|②「17. Error設計」が存在しない履歴・範囲外を区別せず404とすると明記しているため、別名のエラーを新設する根拠がない|②の記載どおり|
|`Grade`の表示名を、`FindViewsByImportHistoryID`の実装内で`grades.year`から共通マスタ参照機能が定める導出規則で変換する構成とした|②「3. Bounded Context」は「学年の表示名は共通マスタ参照機能が定める表記と同じものを用いる」とするのみで、導出の呼び出し方式（master-dataの公開関数の呼び出し、または本Contextでの再実装）を明記していない。共通マスタ参照機能③の公開手段が未確定の場合は、同機能③で確定するまで本Contextで再実装せず、確定後にその手段を用いる|推測|
|CSVの組み立てを標準の`encoding/csv`（改行はLF）で行い、先頭にBOMを付与した|②は「BOM付き、ヘッダー行＋生徒の行」とするのみで、CSVの書き出し方式を明記していない。Rails現行のCSV出力（`CSV.generate`）の改行はLFである|推測|
|参照用データの`GradeDisplayName`・`SchoolClassName`を`*string`とし、未設定は`nil`（レスポンスでは`null`、CSVでは空欄）とした|Rails現行仕様書（教師インポート履歴管理機能）の「学級に所属していない生徒の学級は空（`null`）」に基づく。学年が未設定の生徒についてもRails現行は`null`（安全なナビゲーション）になる|Rails現行の挙動の反映（推測ではない）|

上記以外の設計判断（Bounded Context・Aggregate・Value Object・Domain Service・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API仕様・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
