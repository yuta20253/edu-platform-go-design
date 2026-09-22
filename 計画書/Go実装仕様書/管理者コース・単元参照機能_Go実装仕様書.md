# 管理者コース・単元参照機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が学習コースおよび単元の内容を参照し、問題データの登録状況を確認できるようにする機能である。コース一覧（検索・科目絞り込み・並び替え・ページング付き、単元数・問題数の集計を含む）、コース詳細（単元ごとの問題数集計を含む）、単元詳細（登録済み問題を選択肢・ヒント・解説とともに取得し、あわせて直近の問題インポート履歴を取得）の3操作を提供する参照専用機能であり、作成・更新・削除は現時点で実装されていない（②「1. 機能概要」の要約）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」で **Transaction Script** を採用している。理由は以下のとおり（②の要約）。

- 業務ルールは「論理削除済みを除外する」「検索・並び替え条件の許可値以外は既定値にフォールバックする」「単元がroute上のコースに属するか確認する」という単純な条件判定にとどまる
- コース・単元・問題のいずれも、本機能の文脈では状態を変更しない（参照専用）
- 単元数・問題数の集計は「対象コース・単元に属する有効なレコードをカウントする」という手続きであり、Entityに振る舞いを持たせる意義が薄い

Aggregate・Value Object・Domain Serviceはいずれも②で「不要」と判断されており（②5〜8章）、本書もこれを変更しない。作成・更新・削除は現行Rails実装でも未実装であるため、②同様、本書でも設計対象に含めない。

## 本書が対象とする実装範囲

本書は、②で確定した設計方針（Bounded Context: `course-catalog`、Transaction Script、Repository設計・UseCase設計・Transaction設計・Validation方針・Authorization方針・Error設計・API仕様・DB方針・テスト戦略）を前提に、Goでのpackage構成・関数シグネチャ・struct定義・クエリ内容・Endpoint仕様・テストケースを具体化する。設計方針そのものの再検討は行わない。

①Rails実装の詳細（Controller/Query Object内部コードの具体的な記述内容）は本書作成時点で未提供のため、①の実装詳細を根拠とする記載はすべて「①未提供のため参照不可」として扱い、②に記載された範囲（23章「Railsとの責務対応」等の概念レベルの言及）のみを参照する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名: `course-catalog`（②3章）
- `internal/`配下のディレクトリ名: `course_catalog`

> **②からの補足**: ②にはディレクトリ名（アンダースコア表記）の指定がない。アーキテクチャ規約「8. 命名規約」に従い、Context名のkebab-caseをsnake_caseへ変換した`course_catalog`を採用する。これは管理者高校学年参照機能③文書が採用した変換規則と同一である。

## ②で採用した設計パターン

Transaction Script（②4章）。アーキテクチャ規約「3. 設計パターンごとの構造適用方針」の「Transaction Script」構造をそのまま適用する。domain層・usecase層（struct/interface）・Repository Interfaceは設けない。

## 作成するディレクトリ一覧

```
internal/course_catalog/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    ├── response/
    └── （routes.go はpresentation直下）
```

> **②からの補足**: 指示書のTransaction Script構造例には`presentation/request/`が含まれていない。本機能はクエリパラメータ・パスパラメータの検証のみで独自の入力DTO構造体をリクエストごとに複数持つほどの複雑さがないため、`presentation/request/`ディレクトリは設けず、クエリパラメータ用structはHandlerと同一package（`presentation/handler`）内に定義する（管理者高校学年参照機能③文書と同じ判断）。

## 作成するファイル一覧

```
internal/course_catalog/application/dto.go
internal/course_catalog/application/errors.go
internal/course_catalog/application/import_history_reader.go
internal/course_catalog/application/list_courses.go
internal/course_catalog/application/show_course.go
internal/course_catalog/application/show_unit.go
internal/course_catalog/infrastructure/model.go
internal/course_catalog/infrastructure/course_query.go
internal/course_catalog/infrastructure/unit_query.go
internal/course_catalog/infrastructure/question_summary_query.go
internal/course_catalog/infrastructure/import_history_reader.go
internal/course_catalog/presentation/handler/course_handler.go
internal/course_catalog/presentation/handler/unit_handler.go
internal/course_catalog/presentation/response/course_response.go
internal/course_catalog/presentation/response/unit_response.go
internal/course_catalog/presentation/routes.go
```

> **②からの補足**: ②・指示書のいずれにも、ファイル分割の単位までは記載がない。以下の方針で分割した（推測）。
> - `application/dto.go`: 3つの関数すべてが共有する入出力構造体を1箇所に集約するため
> - `application/errors.go`: Application Errorをapplication関数間で共有するため（11章参照）
> - `application/import_history_reader.go`: question-import Context参照用のInterface定義（コーディング規約「7. インターフェース」の「インターフェースは利用するパッケージ側で定義する」）を、本Context所有データのDTO（`dto.go`）とは分けて配置するため
> - `infrastructure/model.go`: GORMモデル定義（`courses`, `units`, `questions`, `question_choices`, `question_hints`, `question_explanations`テーブルに対応するstruct）を集約するため。`import_histories`はquestion-import Contextが所有するため対象外とする
> - `infrastructure/import_history_reader.go`: ②「11. Repository設計」が「ImportHistoryRepository（question-import Context提供、外部依存として利用）」と明示的に区別しているとおり、本Context所有データに対するクエリ関数（course_query.go/unit_query.go/question_summary_query.go）とは性質が異なる（SQLを発行せずquestion-import Context側の実装を呼び出すアダプタである）ため、ファイルを分けた
>
> ファイル分割自体は実装者の裁量で変更してよい実装細部であり、②の設計判断（採用パターン・責務分離）には影響しない。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層を設けない。アーキテクチャ規約「3. 設計パターンごとの構造適用方針」）。

Entity・Value Object・Repository Interface・Domain Service・Domain Event・Domain Errorはいずれも②5〜8章・18章の判断どおり設計しない。業務ルール検証・存在確認等の処理内容は「4. Application層設計」に記載する。

---

# 4. Application層設計

Transaction Script採用のため、UseCase struct/interfaceは設けない。②「12. UseCase設計」で定義された3つの業務操作（ListCourses / ShowCourse / ShowUnit）を、`application/`直下の関数としてそれぞれ実装する（アーキテクチャ規約「Transaction Scriptの関数は`動詞+対象`の形式にする」に準拠。②のUseCase名がすでにこの形式のため、そのまま関数名として用いる）。

## DTO（`application/dto.go`）

②「12. UseCase設計」の入出力を、Goのstructとして定義する。本機能はすべて読み取り専用のため、②の区分に倣い全DTOを「Query」に属するものとして扱う。

### 入力DTO（Query）

|struct名|フィールドと型|説明|
|-|-|-|
|`ListCoursesParams`|`SubjectID *int`, `Q string`, `Sort string`, `Order string`, `Page int`, `PerPage int`|検索・科目絞り込み・並び替え・ページング。②9章の入力（subject_id, q, sort, order, page, per_page）に対応|
|`ShowCourseParams`|`CourseID int`|対象コースID|
|`ShowUnitParams`|`CourseID int`, `UnitID int`|対象コースID・単元ID（route由来。IDORスコープ検証に用いる）|

`current admin`（②のUseCase入力に記載）は、認可判定に用いる情報のみでUseCase内での業務分岐に使われないため（②「16. Authorization設計」の「UseCase: 単元詳細取得時、route由来のcourse_id配下にunit_idが実在することを検証する」のとおり、判定材料はroute値のみ）、application関数の引数には含めない。

### 出力DTO（Query）

|struct名|フィールドと型|説明|
|-|-|-|
|`SubjectSummary`|`ID int`, `Name string`|②19章「Response: 各コースのsubject（id, name）」に対応|
|`CourseSummary`|`ID int`, `Subject SubjectSummary`, `LevelNumber int`, `LevelName string`, `UnitsCount int`, `QuestionsCount int`, `CreatedAt time.Time`|②6章「CourseSummary」に対応する参照用データ|
|`PaginationMeta`|`CurrentPage int`, `TotalPages int`, `TotalCount int`, `PerPage int`|②19章のmeta構造に対応|
|`ListCoursesResult`|`Courses []CourseSummary`, `Meta PaginationMeta`|`ListCourses`関数の戻り値|
|`UnitSummary`|`ID int`, `UnitName string`, `QuestionsCount int`|②19章「コース詳細のunits（id, unit_name, questions_count）」に対応|
|`CourseDetail`|`ID int`, `Subject SubjectSummary`, `LevelNumber int`, `LevelName string`, `Description string`, `Units []UnitSummary`|`ShowCourse`関数の戻り値|
|`ChoiceDetail`|`ID int`, `ChoiceNumber int`, `Text string`|②19章「choices」に対応|
|`HintDetail`|`ID int`, `StepNumber int`, `Text string`|②19章「hints」に対応|
|`ExplanationDetail`|`ID int`, `Text string`|②19章「explanations」に対応|
|`QuestionDetail`|`ID int`, `QuestionText string`, `CorrectAnswer int`, `Choices []ChoiceDetail`, `Hints []HintDetail`, `Explanations []ExplanationDetail`|②19章「questions」に対応|
|`CourseRef`|`ID int`, `Subject SubjectSummary`, `LevelName string`, `LevelNumber int`|②19章「単元詳細のcourse（id, subject, level_name, level_number）」に対応|
|`ImportHistorySummary`|`ID string`, `FileName string`, `Status string`, `SuccessCount int`, `ErrorCount int`, `TotalCount int`, `CreatedAt time.Time`|②19章「recent_import_histories」に対応。`ID`はquestion-import Context側のImportHistory Entity識別子型（`string`）に合わせる|
|`UnitDetail`|`ID int`, `CourseID int`, `UnitName string`, `Course CourseRef`, `Questions []QuestionDetail`, `RecentImportHistories []ImportHistorySummary`|`ShowUnit`関数の戻り値。②6章「UnitDetail」に対応|

**②からの補足**: 各出力DTOのフィールド構成は、②「19. API仕様」のResponse定義から導出した。②本文にDTOのフィールド型までの明記はないため、実装のために補った。

## Application関数

### ListCourses

- 関数名: `ListCourses`
- シグネチャ: `func ListCourses(ctx context.Context, db *gorm.DB, params ListCoursesParams) (ListCoursesResult, error)`
- 処理ステップ（②13章シーケンス図の要約、13章「単元詳細取得」に相当する一覧版の処理を明文化）:
  1. `params.Sort`/`params.Order`を許可値判定する（`sort`は`level_name`/`created_at`/`id`のいずれか、それ以外は`created_at`。`order`は`asc`/`desc`のいずれか、それ以外は`desc`。②15章のDomain節に対応するガード節）
  2. `infrastructure.SearchCourses`を呼び出し、論理削除除外・`subject_id`絞り込み・`q`によるキーワード検索・並び替え・ページング（1ページ`per_page`件、上限100件）を適用したコース一覧と総件数を取得する
  3. 取得したコースID群をまとめて`infrastructure.CountUnitsAndQuestionsByCourseIDs`に渡し、コースごとの単元数・問題数を一括集計する（②11章「一覧取得後にまとめて集計を行うことで、コースごとに逐次問い合わせるより効率的に処理できる」に対応）
  4. コース情報と集計結果を突き合わせ、`CourseSummary`の一覧を組み立てる
  5. 総件数からページ情報（`PaginationMeta`）を算出する
  6. `ListCoursesResult`として返す
- 呼び出すinfrastructure関数: `SearchCourses`, `CountUnitsAndQuestionsByCourseIDs`
- トランザクション境界: なし（読み取りのみ、②14章）
- 発生しうるApplication Error: なし（一覧取得の失敗はInfrastructure Errorとして扱う）

### ShowCourse

- 関数名: `ShowCourse`
- シグネチャ: `func ShowCourse(ctx context.Context, db *gorm.DB, params ShowCourseParams) (CourseDetail, error)`
- 処理ステップ（②12章「コースの存在確認を前提に、所属単元と単元ごとの問題数をまとめて取得する」）:
  1. `infrastructure.FindCourseByID`を呼び出し、対象コースを取得する
  2. 取得できなかった場合、`ErrCourseNotFound`を返す（②17章「対象コース未存在」）
  3. `infrastructure.ListUnitsByCourseID`を呼び出し、`id`昇順の所属単元一覧を取得する
  4. 取得した単元ID群をまとめて`infrastructure.CountQuestionsByUnitIDs`に渡し、単元ごとの問題数を一括集計する
  5. `CourseDetail`を組み立てて返す
- 呼び出すinfrastructure関数: `FindCourseByID`, `ListUnitsByCourseID`, `CountQuestionsByUnitIDs`
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrCourseNotFound`

### ShowUnit

- 関数名: `ShowUnit`
- シグネチャ: `func ShowUnit(ctx context.Context, db *gorm.DB, importReader ImportHistoryReader, params ShowUnitParams) (UnitDetail, error)`
- 処理ステップ（②13章シーケンス図の記載どおり）:
  1. `infrastructure.FindUnitInCourse`を呼び出し、`params.CourseID`配下に`params.UnitID`が存在するかを確認する（IDORスコープ検証。②16章「route由来のcourse_id配下にunit_idが実在することを検証する」）
  2. 存在しない場合、`ErrUnitNotFound`を返す（②17章「対象単元が存在しない、またはroute上のコースに属さない」。異なるコースの単元IDを指定した場合もこのエラーとして扱い、404で応答する。存在の有無とスコープ不一致を区別しない）
  3. `infrastructure.FindCourseByID`を呼び出し、単元が属するコース情報（`CourseRef`用）を取得する
  4. `infrastructure.ListQuestionsByUnitID`を呼び出し、単元に属する問題一覧を選択肢（choice_number順）・ヒント（step_number順）・解説（登録順）とともに取得する
  5. `importReader.FindRecentByUnit(ctx, params.UnitID, MaxRecentImportHistories)`を呼び出し、question-import Contextから対象単元の直近インポート履歴を取得する（`infrastructure`層の関数ではなく、本Context自身が定義する`ImportHistoryReader` Interface経由の呼び出しである点が他の3つと異なる）
  6. `UnitDetail`を組み立てて返す
- 呼び出す関数: `FindUnitInCourse`, `FindCourseByID`, `ListQuestionsByUnitID`（本Context内のinfrastructure関数）, `ImportHistoryReader.FindRecentByUnit`（question-import Context呼び出しのInterface、DI配線で注入される）
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrUnitNotFound`

**②からの補足**: application関数の第2引数として`*gorm.DB`を直接渡す形にした。②はUseCase設計で依存先を「呼び出す関数」としてのみ記載しており、DB接続の受け渡し方式までは規定していない（推測。管理者高校学年参照機能③文書と同じ判断）。直近インポート履歴の上限件数「5」は②「21. DB操作仕様」の「作成日時の降順で並び替えたうえで上位5件のみを取得する」に明記された業務ルールであり、実装時は`application`パッケージ内の定数（例：`MaxRecentImportHistories = 5`）として定義することを推測として補った。

## question-import Context参照用Interface

- interface名: `ImportHistoryReader`（`application/import_history_reader.go`。コーディング規約「7. インターフェース」の「インターフェースは利用するパッケージ側で定義する」に従い、本Context（course_catalog）側で定義する）
- メソッドシグネチャ: `FindRecentByUnit(ctx context.Context, unitID int, limit int) ([]ImportHistorySummary, error)`
- 責務: 指定単元（`unitID`）に対する直近`limit`件のインポート履歴を取得する
- 実装: `infrastructure/import_history_reader.go`（詳細は「5. Infrastructure層設計」参照）。`ShowUnit`はこのInterface経由でquestion-import Contextのデータを取得し、`import_histories`テーブルへ直接クエリしない（アーキテクチャ規約「5. Context間連携ルール」）

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

②「11. Repository設計」で定義された4つの責務（CourseRepository / UnitRepository / QuestionSummaryRepository / ImportHistoryRepository）のうち、本Context所有データを扱う3つ（CourseRepository / UnitRepository / QuestionSummaryRepository）は、アーキテクチャ規約「Repository Interfaceは定義しない。DBアクセスは`infrastructure/`内の関数から直接行う」のとおり`infrastructure/`内の関数として実装する。他Context（question-import）所有データを扱うImportHistoryRepository相当分は、本Context側で定義する`ImportHistoryReader` Interfaceの実装として`infrastructure/import_history_reader.go`に置く（後述）。

### `infrastructure/course_query.go`（②11章 CourseRepository相当）

|関数名|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`SearchCourses`|`ctx context.Context`, `db *gorm.DB`, `subjectID *int`, `keyword string`, `sort string`, `order string`, `page int`, `perPage int`|`([]CourseModel, totalCount int64, error)`|`deleted_at IS NULL`（論理削除除外）を前提に、`subject_id`が指定された場合のみ絞り込み条件を付加し、`keyword`が指定された場合は`level_name`/`description`への部分一致条件を付加する。正規化済みの`sort`/`order`でソートし、1ページ`perPage`件（上限100件）のOFFSET/LIMITページングを行う。総件数は同条件でのCOUNTを別途取得する。`subjects`とJoinして科目名を取得する（②21章）|
|`FindCourseByID`|`ctx context.Context`, `db *gorm.DB`, `id int`|`(*CourseModel, error)`|`deleted_at IS NULL`かつ`id`一致条件で1件取得する。`subjects`とJoinして科目名を取得する|

### `infrastructure/unit_query.go`（②11章 UnitRepository相当）

|関数名|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`ListUnitsByCourseID`|`ctx context.Context`, `db *gorm.DB`, `courseID int`|`([]UnitModel, error)`|`deleted_at IS NULL`かつ`course_id`一致条件で絞り込み、`id`昇順でソートする（②21章）|
|`FindUnitInCourse`|`ctx context.Context`, `db *gorm.DB`, `courseID int`, `unitID int`|`(*UnitModel, error)`|`deleted_at IS NULL`かつ`course_id`と`id`の両方が一致する場合のみ1件取得する（②21章「単元詳細取得時は`course_id` + `id`の組み合わせで一致するものだけを対象とする」。IDORスコープ検証を兼ねる）|

### `infrastructure/question_summary_query.go`（②11章 QuestionSummaryRepository相当）

|関数名|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`CountUnitsAndQuestionsByCourseIDs`|`ctx context.Context`, `db *gorm.DB`, `courseIDs []int`|`(map[int]CourseAggregateCount, error)`|`course_id IN (...)`で複数コースIDをまとめて絞り込み、単元数（`units`の`deleted_at IS NULL`件数）と問題数（`units`経由で`questions`の`deleted_at IS NULL`件数）を`course_id`単位でグルーピング集計する（N+1回避）。戻り値はコースIDをキーとしたマップとする|
|`CountQuestionsByUnitIDs`|`ctx context.Context`, `db *gorm.DB`, `unitIDs []int`|`(map[int]int, error)`|`unit_id IN (...)`で複数単元IDをまとめて絞り込み、`questions`の`deleted_at IS NULL`件数を`unit_id`単位でグルーピング集計する|
|`ListQuestionsByUnitID`|`ctx context.Context`, `db *gorm.DB`, `unitID int`|`([]QuestionModel, error)`|`deleted_at IS NULL`かつ`unit_id`一致条件で`id`順に取得し、`question_choices`（choice_number順）・`question_hints`（step_number順）・`question_explanations`（登録順）をそれぞれ結合して取得する（②21章）|

`CourseAggregateCount`は`infrastructure/question_summary_query.go`内で定義する補助struct（`UnitsCount int`, `QuestionsCount int`）とする。

### `infrastructure/import_history_reader.go`（②11章 ImportHistoryRepository相当、question-import Context連携）

`import_histories`テーブルはquestion-import Context（管理者問題インポート機能・管理者インポート履歴管理機能）が所有するため、本Contextでは対応するGORMモデル・クエリ関数を定義しない。「4. Application層設計」で定義した`ImportHistoryReader` Interfaceの実装は、question-import Contextが公開する`SearchImportHistoriesUseCase`（管理者インポート履歴管理機能_Go実装仕様書「4. Application層設計」。管理者問題インポート機能_Go実装仕様書が定義した`ImportHistoryRepository`を内部で利用するUseCase）を呼び出すアダプタとする。

- 実装struct名: 非公開struct（例: `importHistoryReader`）+ コンストラクタ`NewImportHistoryReader(searchUseCase *questionimport.SearchImportHistoriesUseCase) application.ImportHistoryReader`
- `FindRecentByUnit(ctx, unitID, limit)`: `searchUseCase.Execute(ctx, dto.RawSearchImportHistoriesParams{UnitID: strconv.Itoa(unitID), Sort: "created_at", Order: "desc", Page: 1, PerPage: limit})`を呼び出し、戻り値`dto.SearchImportHistoriesResult.Items`（`[]ImportHistoryListItem`、`ID`/`FileName`/`Status`/`TotalCount`/`SuccessCount`/`ErrorCount`/`CreatedAt`を保持）を本Contextの`ImportHistorySummary`へ変換する

> **②からの補足**: 本関数は管理者問題インポート機能・管理者インポート履歴管理機能が所有するquestion-import Contextのデータを参照する。②「3. Bounded Context」が「question-import Context: ...に依存する」と明記しており、本タスクで並行して確定した両文書が公開する`SearchImportHistoriesUseCase`（`UnitID`による絞り込み・`Sort`/`Order`・ページングに対応）のシグネチャに合わせて実装方式を確定した。これにより`import_histories`テーブルへの直接クエリは行わない（②が既に想定している依存を具体的な関数呼び出しへ落とし込んだものであり、新しい業務ルールの追加ではない）。

## Entity ⇔ GORMモデルの変換方針

Transaction Script採用のためEntityは存在しない。GORMモデル（`infrastructure/model.go`）から、application層のDTO（`CourseSummary`, `UnitDetail`等）への変換をinfrastructure関数の戻り値受け取り後にapplication関数内で行う。

## 外部連携実装

対象外。②にMail・Cache・Queue等の外部連携は記載されていない。

---

# 6. Presentation層設計

## Handler

②「23. Railsとの責務対応」で言及される`Admin::CoursesController`・`Admin::UnitsController`という2つのController単位に対応させ、Handlerも2つに分ける（管理者高校学年参照機能③文書と同じ判断。①Controller実装詳細そのものは未提供のため参照不可）。

### CourseHandler（`presentation/handler/course_handler.go`）

- struct名: `CourseHandler`
- 保持する依存: `DB *gorm.DB`
- 対応する呼び出し先: `application.ListCourses`, `application.ShowCourse`
- メソッド一覧:
  |メソッド|HTTPメソッド|パス|
  |-|-|-|
  |`List(c *gin.Context)`|GET|`/api/v1/admin/courses`|
  |`Show(c *gin.Context)`|GET|`/api/v1/admin/courses/:id`|
- 処理順序:
  - `List`: クエリパラメータ（`subject_id`, `q`, `sort`, `order`, `page`, `per_page`）をリクエスト用structへバインド → 型・フォーマットの検証（9章） → `application.ListCourses`を呼び出す → 結果を`CourseListResponse`へ変換 → 200で返す
  - `Show`: パスパラメータ`id`をバインド・検証 → `application.ShowCourse`を呼び出す → `ErrCourseNotFound`の場合は404に変換 → 成功時は`CourseDetailResponse`へ変換し200で返す

### UnitHandler（`presentation/handler/unit_handler.go`）

- struct名: `UnitHandler`
- 保持する依存: `DB *gorm.DB`, `ImportReader application.ImportHistoryReader`（question-import Context側の実装を、アーキテクチャ規約「14. 依存関係の組み立て（DI配線）」に従い`course_catalog.NewContext`がDI配線で注入する）
- 対応する呼び出し先: `application.ShowUnit`
- メソッド一覧:
  |メソッド|HTTPメソッド|パス|
  |-|-|-|
  |`Show(c *gin.Context)`|GET|`/api/v1/admin/courses/:course_id/units/:id`|
- 処理順序: パスパラメータ`course_id`/`id`をバインド・必須チェック → `application.ShowUnit(ctx, db, h.ImportReader, params)`を呼び出す → `ErrUnitNotFound`の場合は404に変換 → 成功時は`UnitDetailResponse`へ変換し200で返す

## Request / Response DTO

### Requestパラメータ（Handler内structとして定義。`presentation/request`は設けない。2章参照）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`listCoursesQuery`|`SubjectID *int \`form:"subject_id"\``, `Q string \`form:"q"\``, `Sort string \`form:"sort"\``, `Order string \`form:"order"\``, `Page int \`form:"page"\``, `PerPage int \`form:"per_page"\``|`subject_id`は型（整数）チェックのみで必須ではない。`page`/`per_page`は正の整数であることを検証する（②15章）。`sort`/`order`は文字列であることのみを検証し、許可値判定はapplication関数のガード節で行う。バインド失敗時は400|
|パスパラメータ`id` / `course_id`|`int`（`uri`タグでバインド）|整数であることを検証する。単元詳細では両方を必須パラメータとして扱う（②15章）|

### Response DTO（`presentation/response/`）

|struct名|フィールドと型|備考|
|-|-|-|
|`SubjectResponse`|`ID int`, `Name string`|②19章の`subject`に対応|
|`CourseResponse`|`ID int`, `Subject SubjectResponse`, `LevelNumber int`, `LevelName string`, `UnitsCount int`, `QuestionsCount int`, `CreatedAt time.Time`|②19章のコース一覧項目に対応|
|`MetaResponse`|`CurrentPage int`, `TotalPages int`, `TotalCount int`, `PerPage int`|②19章「meta」に対応|
|`CourseListResponse`|`Courses []CourseResponse`, `Meta MetaResponse`|一覧APIのレスポンス全体|
|`UnitSummaryResponse`|`ID int`, `UnitName string`, `QuestionsCount int`|②19章「コース詳細のunits」に対応|
|`CourseDetailResponse`|`ID int`, `Subject SubjectResponse`, `LevelNumber int`, `LevelName string`, `Description string`, `Units []UnitSummaryResponse`|コース詳細APIのレスポンス全体|
|`ChoiceResponse`|`ID int`, `ChoiceNumber int`, `Text string`|
|`HintResponse`|`ID int`, `StepNumber int`, `Text string`|
|`ExplanationResponse`|`ID int`, `Text string`|
|`QuestionResponse`|`ID int`, `QuestionText string`, `CorrectAnswer int`, `Choices []ChoiceResponse`, `Hints []HintResponse`, `Explanations []ExplanationResponse`|②19章「questions」に対応|
|`CourseRefResponse`|`ID int`, `Subject SubjectResponse`, `LevelName string`, `LevelNumber int`|②19章「単元詳細のcourse」に対応|
|`ImportHistorySummaryResponse`|`ID string`, `FileName string`, `Status string`, `SuccessCount int`, `ErrorCount int`, `TotalCount int`, `CreatedAt time.Time`|②19章「recent_import_histories」に対応|
|`UnitDetailResponse`|`ID int`, `CourseID int`, `UnitName string`, `Course CourseRefResponse`, `Questions []QuestionResponse`, `RecentImportHistories []ImportHistorySummaryResponse`|単元詳細APIのレスポンス全体|

## Routing（`presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/admin/courses`|`CourseHandler.List`|
|GET|`/api/v1/admin/courses/:id`|`CourseHandler.Show`|
|GET|`/api/v1/admin/courses/:course_id/units/:id`|`UnitHandler.Show`|

いずれも、認証Middleware・admin役割確認Middleware配下のルートグループに登録する（②16章）。認証・admin確認Middleware自体の実装は本機能のスコープ外であり、既存の共通Middleware（他機能で使用しているものと同一）を利用する想定とする（①未提供のため参照不可）。

---

# 7. API仕様

②19章のAPI仕様に基づくEndpoint一覧。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|`/api/v1/admin/courses`|`CourseHandler.List`|`subject_id`（任意, int）, `q`（任意, string）, `sort`（任意, string）, `order`（任意, string）, `page`（任意, int）, `per_page`（任意, int）|`CourseListResponse`|200|
|GET|`/api/v1/admin/courses/:id`|`CourseHandler.Show`|パスパラメータ`id`（int）|`CourseDetailResponse`|200|
|GET|`/api/v1/admin/courses/:course_id/units/:id`|`UnitHandler.Show`|パスパラメータ`course_id`（int）, `id`（int）|`UnitDetailResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（既存の共通エラー形式。②19章「既存のエラー形式を踏襲する」）|
|adminロールでない|403|認可エラー（既存の共通エラー形式）|
|`page`, `per_page`, `subject_id`, `id`, `course_id`が数値でない|400|入力形式エラー|
|指定`id`の対象コースが存在しない|404|`ErrCourseNotFound`（②17章「対象コース未存在」）|
|指定`course_id`/`id`の対象単元が存在しない、またはroute上のコースに属さない|404|`ErrUnitNotFound`（②17章「対象単元が存在しない、またはroute上のコースに属さない」）|
|DB接続失敗・クエリ失敗|500|Infrastructure Error（②17章）|

---

# 8. Transaction実装方針

②「14. Transaction設計」のとおり、本機能はすべて読み取り専用処理であり、明示的なトランザクションを使用しない。

- Transaction開始箇所: なし
- Transaction終了箇所（Commit/Rollback条件）: 該当なし
- 複数infrastructure関数にまたがる場合の扱い: `ListCourses`・`ShowCourse`・`ShowUnit`はそれぞれ複数のinfrastructure関数（検索/詳細取得 + 集計、または存在確認 + 参照）を呼び出すが、読み取りのみのため、`db.Transaction(...)`等によるトランザクション制御は行わず、各infrastructure関数は独立したクエリとして順次実行する（②14章「保護すべき整合性が存在せず、不要な複雑さを増やすだけ」）

---

# 9. Validation実装方針

## Presentation

②「15. Validation設計」のPresentation節を実装レベルに落とし込む。

- 型チェック: `subject_id`, `course_id`, `id`, `page`, `per_page`をGinのクエリ/パスバインド時に整数として検証する（バインド失敗時は400）
- 必須チェック: コース詳細の`id`・単元詳細の`course_id`/`id`はパスパラメータとして必須。ルーティング定義上、値が欠落するとGinのルーティング自体が一致しないため、バインド後の型検証で担保する
- フォーマットチェック: `page`/`per_page`が正の整数であることをHandler内のガード節で検証する。`per_page`の上限（100件）はapplication関数側で丸め込む。`sort`/`order`は文字列であることのみを検証し、許可値かどうかの判定はDomain側（実質的にはapplication関数のガード節）で行う（②15章）

## 業務ルール検証

Transaction Script採用時のため、application関数内のガード節で検証する内容を記載する。

- `ListCourses`は`sort`/`order`の許可値判定・既定値フォールバック（②15章「Domain: 業務ルール」）を行う
- `ShowCourse`は、`infrastructure.FindCourseByID`の結果が存在しない場合、`ErrCourseNotFound`を返すガード節を持つ
- `ShowUnit`は、`infrastructure.FindUnitInCourse`の結果が存在しない場合（route上のコースに属さない場合を含む）、`ErrUnitNotFound`を返すガード節を持つ（②15章「整合性チェック: 単元がroute上のcourse_idに属しているか」）

---

# 10. Authorization実装方針

②「16. Authorization設計」を実装レベルに落とし込む。

- Middleware: 認証済みユーザーを特定しコンテキストに保持する。役割が`admin`であることを確認し、`admin`でない場合は403で処理を打ち切る（②16章）
- Handler: パスパラメータ（`id`, `course_id`）をapplication関数に受け渡すのみで、権限判定は行わない（②16章）
- application関数: `ShowUnit`は、route由来の`course_id`配下に`unit_id`が実在することを検証する（IDOR対策。②16章「リクエストボディ・クエリの値は信用せず、routeの値のみを正とする」）。`ListCourses`・`ShowCourse`は追加のスコープ制限を行わない（管理者は全コース・全単元を参照可能）
- Domain: 該当なし（Domain層を設けないため）

---

# 11. Error実装方針

②「17. Error設計」を実装レベルに落とし込む。

## Domain Error → Application Errorへの変換方針

Domain Errorは定義しない（②17章、③3章）。Application Errorは`application/errors.go`にsentinel error（またはそれに相当するerror値）として定義する。

- `ErrCourseNotFound`: 対象コースが存在しない場合に、`ShowCourse`から返される（②17章「対象コース未存在」）
- `ErrUnitNotFound`: 対象単元が存在しない、またはroute上のコースに属さない場合に、`ShowUnit`から返される（②17章）

コーディング規約に従い、エラーは関数の最後の戻り値として返し、`errors.New`または`fmt.Errorf`のいずれかをプロジェクト内の既存の使い分けに合わせて用いる。

## Application Error → HTTPレスポンスへの変換方針

Handler側で`errors.Is(err, application.ErrCourseNotFound)` / `errors.Is(err, application.ErrUnitNotFound)`により判定し、該当する場合は404、それ以外のエラー（infrastructure由来）は500に変換する。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrCourseNotFound`|Application（application関数のガード節）|404|
|`ErrUnitNotFound`|Application（application関数のガード節）|404|
|クエリバインド・型検証エラー|Presentation（Handler）|400|
|認証エラー|Middleware|401|
|認可エラー（adminでない）|Middleware|403|
|DB接続失敗・クエリ実行エラー|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

infrastructure関数が返すDBエラー（接続失敗・クエリ失敗）は、application関数でラップせずにそのまま呼び出し元へ伝播させ（コーディング規約の`%w`によるラップを推奨）、Handlerで`ErrCourseNotFound`/`ErrUnitNotFound`以外のエラーとして一律500に変換する（②17章「技術的な障害を業務エラーと切り分ける」）。

---

# 12. GORM / DBクエリ設計

②「20. DB設計方針」のとおり、既存Rails DBを継続利用し、Schema変更は行わない。

## 利用するGORMモデルとテーブルの対応

`infrastructure/model.go`に、以下のGORMモデルを定義する。

|struct名|対応テーブル|備考|
|-|-|-|
|`CourseModel`|`courses`|`id`, `subject_id`, `level_number`, `level_name`, `description`, `deleted_at`, `created_at`を保持する。`subjects`との結合結果として科目名を含める想定|
|`UnitModel`|`units`|`id`, `course_id`, `unit_name`, `deleted_at`を保持する|
|`QuestionModel`|`questions`|`id`, `unit_id`, `question_text`, `correct_answer`, `deleted_at`を保持する|
|`QuestionChoiceModel`|`question_choices`|`id`, `question_id`, `choice_number`, `text`を保持する|
|`QuestionHintModel`|`question_hints`|`id`, `question_id`, `step_number`, `text`を保持する|
|`QuestionExplanationModel`|`question_explanations`|`id`, `question_id`, `text`を保持する|

`import_histories`に対応するGORMモデルは本Contextでは定義しない（question-import Contextが正規の所有者であるため。「5. Infrastructure層設計」の`import_history_reader.go`参照）。

> **②からの補足**: 各モデルの正確なフィールド一覧・カラム名は①未提供のため参照不可。②19章のレスポンス項目から逆算して必要な情報を列挙したが、実装時は既存DBスキーマを別途確認し、モデル定義を確定する必要がある（推測）。

## 主要クエリの条件・ソート・ページネーション方針

- コース一覧: `deleted_at IS NULL`を前提に`subject_id`が指定された場合はWHERE条件として付加、`q`指定時は`level_name`/`description`への部分一致検索、`sort`/`order`（許可値判定済み）でソート、`per_page`件（上限100件）のページング。総件数取得は同一条件でのCOUNTクエリを別途発行する
- コース詳細: `deleted_at IS NULL`かつ`id`一致条件での単一レコード取得
- 単元一覧（コース詳細内）: `deleted_at IS NULL`かつ`course_id`一致条件、`id`昇順ソート
- 単元詳細のスコープ確認: `deleted_at IS NULL`かつ`course_id`と`id`の両方が一致する場合のみ取得
- 単元数・問題数集計: 複数`course_id`/`unit_id`をIN条件でまとめて絞り込み、`course_id`/`unit_id`単位でグルーピング集計する一括クエリ（N+1回避、②21章）
- 問題一覧（単元詳細）: `deleted_at IS NULL`かつ`unit_id`一致条件、`id`順。選択肢（choice_number順）・ヒント（step_number順）・解説（登録順）を結合して取得
- 直近インポート履歴: `ImportHistoryReader.FindRecentByUnit`経由でquestion-import Context側の`SearchImportHistoriesUseCase`を呼び出す（`unit_id`一致条件、`created_at`降順、上位5件）。SQL・ソート・LIMIT自体の発行はquestion-import Context側の実装に従う

SQL文そのものはここに記載しない。GORMのクエリビルダ（`Where`, `Order`, `Limit`/`Offset`, `Group`, `Preload`/`Joins`等に相当する操作）を用いて実装する。

## 既存Schemaに対する変更

②20章のとおり変更なし。`courses`, `units`, `questions`, `question_choices`, `question_hints`, `question_explanations`の既存カラムのみで一覧・詳細・集計・単元参照のすべてを満たす。`import_histories`はquestion-import Contextが所有するため、本Contextからの変更提案は対象外。

---

# 13. テストケース設計

②「22. テスト戦略」を、アーキテクチャ規約・指示書のTransaction Script読み替えルール（「Domain Test」「Repository Test」は対象外、「UseCase Test」→「Application関数 Test」）に従って具体化する。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test（②の「UseCase Test」に相当）

|対象|テストケース|
|-|-|
|`ListCourses`|`subject_id`未指定時に全件が対象となること／`subject_id`指定時に絞り込まれること／`q`によるキーワード検索が`level_name`/`description`に適用されること／`sort`/`order`が許可値でない場合に既定値へフォールバックすること／ページングにより指定ページの件数のみ返ること／各コースの`UnitsCount`・`QuestionsCount`が集計結果と一致すること／論理削除済みコースが結果に含まれないこと|
|`ShowCourse`|存在する`course_id`で詳細と単元ごとの`QuestionsCount`が返ること／存在しない`course_id`で`ErrCourseNotFound`が返ること／論理削除済みコースを指定した場合に`ErrCourseNotFound`が返ること|
|`ShowUnit`|存在する`course_id`/`unit_id`の組み合わせで問題一覧（選択肢・ヒント・解説含む）と直近インポート履歴が返ること／存在しない`unit_id`で`ErrUnitNotFound`が返ること／`unit_id`は存在するが異なる`course_id`を指定した場合に`ErrUnitNotFound`が返ること（IDOR対策の重点検証項目）／直近インポート履歴が6件以上存在する場合でも返却件数が5件に絞り込まれること|

## Repository Test

対象外（Transaction Script採用のため、Repository Interfaceを設けない。指示書の読み替えルールに従う）。

Infrastructure関数のクエリ正確性検証（`infrastructure/*_test.go`として実装）:

|対象|テストケース|
|-|-|
|`SearchCourses`|`subject_id`絞り込み条件が正しく適用されること／`q`によるキーワード検索が正しいこと／`sort`/`order`が正しく反映されること／ページング（`per_page`区切り）が正しいこと／総件数取得が正しいこと／論理削除済みコースが除外されること|
|`FindCourseByID`|存在するIDで正しいレコードが返ること／存在しないID・論理削除済みIDでレコードなしとなること|
|`ListUnitsByCourseID`|指定`course_id`の単元のみ`id`昇順で返ること／論理削除済み単元が除外されること|
|`FindUnitInCourse`|`course_id`と`id`が一致する場合のみ取得できること／一致しない組み合わせでレコードなしとなること|
|`CountUnitsAndQuestionsByCourseIDs`|複数コースIDを渡した際にコースごとの単元数・問題数が正しく集計されること／該当データがないコースでカウント0が扱えること|
|`CountQuestionsByUnitIDs`|複数単元IDを渡した際に単元ごとの問題数が正しく集計されること|
|`ListQuestionsByUnitID`|指定`unit_id`の問題のみ`id`順で返ること／選択肢がchoice_number順、ヒントがstep_number順で取得できること|

`ImportHistoryReader`実装（`infrastructure/import_history_reader.go`、テストダブルとした`SearchImportHistoriesUseCase`を用いる）:

|対象|テストケース|
|-|-|
|`ImportHistoryReader.FindRecentByUnit`|`SearchImportHistoriesUseCase.Execute`への呼び出しパラメータ（`UnitID`, `Sort=created_at`, `Order=desc`, `Page=1`, `PerPage=limit`）が正しいこと／`ImportHistoryListItem`から`ImportHistorySummary`への変換が正しいこと／インポート履歴が0件の場合、空スライスが返ること|

直近5件が`created_at`降順で取得されること・対象単元以外の履歴が含まれないことのクエリ自体の正確性は、question-import Context側（管理者インポート履歴管理機能）の`ImportHistoryRepository.Search`のテストで検証済みの前提とし、本Contextでは上記のとおりアダプタの呼び出しパラメータ・変換ロジックの検証に限定する。

## Handler Test

|対象|テストケース|
|-|-|
|`CourseHandler.List`|クエリパラメータの解釈（`subject_id`, `q`, `sort`, `order`, `page`, `per_page`）が正しいこと／不正な`page`/`per_page`で400が返ること／正常系で200と期待するレスポンス構造が返ること|
|`CourseHandler.Show`|存在しない`id`で404が返ること／不正な`id`（数値以外）で400が返ること／正常系で200が返ること|
|`UnitHandler.Show`|存在しない`course_id`/`id`の組み合わせで404が返ること／正常系で200が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/admin/courses`|エンドポイント経由で一覧が正しいレスポンス構造（`courses`, `meta`）で返却されること|
|`GET /api/v1/admin/courses/:id`|エンドポイント経由で詳細が正しく返却されること／存在しないIDで404が返ること|
|`GET /api/v1/admin/courses/:course_id/units/:id`|エンドポイント経由で単元詳細が正しく返却されること／IDOR対策（route上のコースに属さない単元へのアクセスが404になること）が機能すること|
|認可|admin以外のロールでアクセスした場合に403が返ること／未認証の場合に401が返ること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に整理する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|`internal/`配下のディレクトリ名を`course_catalog`とする|②のContext名`course-catalog`はkebab-case表記のみで、ディレクトリ名の指定がないため、アーキテクチャ規約「8. 命名規約」に従い変換した（管理者高校学年参照機能③文書と同一規則）|推測|
|`presentation/request/`ディレクトリを設けず、クエリ/パスパラメータ用structをHandlerと同一package内に定義する|指示書のTransaction Script構造例に`request/`が含まれておらず、本機能の入力構造が単純なため|推測|
|`application/dto.go`・`application/errors.go`・`application/import_history_reader.go`・`infrastructure/model.go`という補助ファイルへの分割、および`import_history_reader.go`を他のクエリ関数と別ファイルに分けたこと|②・指示書ともにファイル粒度までは規定していないため、責務ごとに集約する分割を採用した。`import_history_reader.go`の分離は、②「11. Repository設計」がImportHistoryRepositoryを「question-import Context提供、外部依存として利用」と明示的に区別していることに対応する|推測|
|application関数に`current admin`を引数として渡さない|②16章「Handler: パスパラメータを受け渡すのみ」「UseCase: 追加のスコープ制限は行わない（単元詳細のみroute値によるスコープ検証）」という判断に基づき、認可情報はMiddleware/Handlerで完結させ、application関数の入力から除外した|②の判断に基づく具体化（推測要素は低い）|
|Handlerを`CourseHandler`・`UnitHandler`の2つに分割する|②23章のRailsとの対応表で`Admin::CoursesController`・`Admin::UnitsController`という2つのController単位が言及されていることを踏まえた（①Controller実装詳細そのものは未提供のため参照不可）|推測|
|`ShowUnit`において、単元が存在しない場合とroute上のコースに属さない場合を区別せず、いずれも`ErrUnitNotFound`（404）として扱う|②17章は両者を1つの業務シナリオ（「対象単元が存在しない、またはroute上のコースに属さない」）としてまとめて記載しており、区別する記載がないため、②の記述をそのまま実装に反映した|②の記載どおり（推測ではない）|
|GORMモデル（`CourseModel`, `UnitModel`, `QuestionModel`等）の正確なフィールド一覧・カラム名は確定できない|①（Railsの既存DBスキーマ）が未提供のため参照不可。②19章のレスポンス項目から必要な情報を逆算したのみ|推測（要・既存DBスキーマ確認）|
|`ImportHistoryModel`のようなGORMモデルを本機能のinfrastructure層に独自定義せず、`ImportHistoryReader` Interface（本Contextで定義）経由でquestion-import Context側の`SearchImportHistoriesUseCase`を呼び出す構成とした|本タスクで並行して確定した管理者問題インポート機能_Go実装仕様書・管理者インポート履歴管理機能_Go実装仕様書が`ImportHistoryRepository`（`Search`メソッドを含む）・`SearchImportHistoriesUseCase`を公開したため、コーディング規約「7. インターフェース」の「インターフェースは利用するパッケージ側で定義する」、アーキテクチャ規約「5. Context間連携ルール」に従い、`import_histories`テーブルへの直接クエリを解消した|②からの補足（③間の整合を取るための具体化であり、新しい業務ルールの追加ではない）|
|認証・admin確認Middlewareの具体的なpackage名・実装|①・②いずれにも記載がなく、既存の共通Middlewareを利用する方針のみを明記した|①未提供のため参照不可|
|application関数の第2引数として`*gorm.DB`を直接渡す|②はDB接続の受け渡し方式を規定していないため、Transaction Script構造に沿ったシンプルな受け渡し方法を採用した|推測|

上記以外の設計判断（採用パターン・Bounded Context・Repository/UseCase設計・Transaction/Validation/Authorization/Error設計・API仕様・DB方針・テスト戦略）は、すべて②の記載どおりであり、本書での変更は行っていない。
