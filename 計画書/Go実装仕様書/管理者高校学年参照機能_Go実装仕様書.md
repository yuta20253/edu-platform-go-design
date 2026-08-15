# 管理者高校学年参照機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が高校と学年の一覧・詳細を参照するための機能である。以下の3操作を提供する参照専用機能であり、作成・更新・削除は存在しない。

- 高校一覧（都道府県絞り込み・ページング、生徒数・教員数の集計付き）
- 高校詳細（生徒数・教員数の集計付き）
- 指定高校配下の学年一覧

（②「1. 機能概要」の要約）

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」で **Transaction Script** を採用している。理由は以下のとおり（②の要約）。

- 業務ルールは「都道府県で絞り込む」程度で、複雑な判断ロジックが存在しない
- 高校・学年は本機能の文脈で状態を持たず、状態遷移という概念自体が存在しない
- 生徒数・教員数の集計は「対象高校に属するユーザーをロール別にカウントする」単純な手続きであり、Entityに振る舞いを持たせる意義が薄い
- 参照専用機能にDomain Model / Active Record相当の構造を適用すると、実体のない抽象化を生みやすく、かえって保守性を下げる

Aggregate・Value Object・Domain Service・Domain Eventはいずれも②で「採用しない」と判断されており（②5〜8章、15章）、本書もこれを変更しない。

## 本書が対象とする実装範囲

本書は、②で確定した設計方針（Bounded Context: `school-directory`、Transaction Script、Repository設計・UseCase設計・Transaction設計・Validation方針・Authorization方針・Error設計・API互換方針・DB方針・テスト戦略）を前提に、Goでのpackage構成・関数シグネチャ・struct定義・クエリ内容・Endpoint仕様・テストケースを具体化する。設計方針そのものの再検討は行わない。

①Rails実装の詳細（Controller/Model内部コードの具体的な記述内容）は本書作成時点で未提供のため、①の実装詳細を根拠とする記載はすべて「①未提供のため参照不可」として扱い、②に記載された範囲（19章「Railsとの責務対応」等の概念レベルの言及）のみを参照する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名: `school-directory`（②3章）
- `internal/`配下のディレクトリ名: `school_directory`

> **②からの補足**: ②にはディレクトリ名（アンダースコア表記）の指定がない。アーキテクチャ規約5章「Context名とディレクトリ名が一致しない場合は、②文書内で対応関係を明記する」に従い、Context名のkebab-caseをsnake_caseへ変換した`school_directory`を採用する。これは指示書のActive Record例（`teacher-directory` → `internal/teacher_directory`）と同じ変換規則に倣った判断であり、②に明記のない実装上の補足である。

## ②で採用した設計パターン

Transaction Script（②4章）。アーキテクチャ規約「4. 設計パターンごとの構造適用方針」の「Transaction Script」構造をそのまま適用する。domain層・usecase層（struct/interface）・Repository Interfaceは設けない。

## 作成するディレクトリ一覧

```
internal/school_directory/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    └── response/
```

> **②からの補足**: 指示書のTransaction Script構造例には`presentation/request/`が含まれていない。本機能はクエリパラメータ・パスパラメータの検証のみで独自の入力DTO構造体をリクエストごとに複数持つほどの複雑さがないため、`presentation/request/`ディレクトリは設けず、クエリパラメータ用structはHandlerと同一package（`presentation/handler`）内に定義する。これは規約のTransaction Script構造（requestディレクトリなし）に整合させた判断である。

## 作成するファイル一覧

```
internal/school_directory/application/dto.go
internal/school_directory/application/errors.go
internal/school_directory/application/list_high_schools.go
internal/school_directory/application/show_high_school.go
internal/school_directory/application/list_grades.go
internal/school_directory/infrastructure/model.go
internal/school_directory/infrastructure/high_school_query.go
internal/school_directory/infrastructure/grade_query.go
internal/school_directory/infrastructure/membership_count_query.go
internal/school_directory/presentation/handler/high_school_handler.go
internal/school_directory/presentation/handler/grade_handler.go
internal/school_directory/presentation/response/high_school_response.go
internal/school_directory/presentation/response/grade_response.go
internal/school_directory/presentation/routes.go
```

> **②からの補足**: ②・指示書のいずれにも、`dto.go`・`errors.go`・`model.go`のような補助ファイルの分割単位までは記載がない。以下の方針で分割した（推測）。
> - `application/dto.go`: 3つの関数すべてが共有する入出力構造体を1箇所に集約するため
> - `application/errors.go`: Application Errorをapplication関数間で共有するため（11章参照）
> - `infrastructure/model.go`: GORMモデル定義（`high_schools`, `grades`, `users`テーブルに対応するstruct）を集約するため
>
> ファイル分割自体は実装者の裁量で変更してよい実装細部であり、②の設計判断（採用パターン・責務分離）には影響しない。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層を設けない。アーキテクチャ規約「4. 設計パターンごとの構造適用方針」）。

Entity・Value Object・Repository Interface・Domain Service・Domain Event・Domain Errorはいずれも②5〜8章・15章の判断どおり設計しない。業務ルール検証・存在確認等の処理内容は「4. Application層設計」に記載する。

---

# 4. Application層設計

Transaction Script採用のため、UseCase struct/interfaceは設けない。②「10. UseCase設計」で定義された3つの業務操作（ListHighSchools / ShowHighSchool / ListGrades）を、`application/`直下の関数としてそれぞれ実装する（アーキテクチャ規約「Transaction Scriptの関数は`動詞+対象`の形式にする」に準拠。②のUseCase名がすでにこの形式のため、そのまま関数名として用いる）。

## DTO（`application/dto.go`）

②「10. UseCase設計」の入出力を、Goのstructとして定義する。Command/Query区分について、本機能はすべて読み取り専用のため、②の区分に倣い全DTOを「Query」に属するものとして扱う。

### 入力DTO（Query）

|struct名|フィールドと型|説明|
|-|-|-|
|`ListHighSchoolsParams`|`PrefectureID *int`, `Page int`|都道府県絞り込み（任意）・ページ番号。②9章の入力（prefecture_id, page）に対応|
|`ShowHighSchoolParams`|`HighSchoolID int`|対象高校ID|
|`ListGradesParams`|`HighSchoolID int`|対象高校ID|

`current admin`（②のUseCase入力に記載）は、認可判定に用いる情報のみでUseCase内での業務分岐に使われないため（②13章「UseCase: 追加のスコープ制限は行わない」）、application関数の引数には含めない。認証・認可はMiddleware/Handlerで完結させる（10章参照）。

> **②からの補足**: ②は「入力: current admin, prefecture_id, page」のようにcurrent adminを明示しているが、13章の判断根拠でUseCase側にスコープ制限がないと明言されているため、関数引数から除外した。これは②の設計判断（Authorization責務の配置）に基づく実装上の具体化であり、②の判断に反するものではない。

### 出力DTO（Query）

|struct名|フィールドと型|説明|
|-|-|-|
|`HighSchoolSummary`|`ID int`, `Name string`, `PrefectureName string`, `StudentCount int`, `TeacherCount int`|②6章「HighSchoolSummary」に対応する参照用データ|
|`PaginationMeta`|`CurrentPage int`, `TotalPages int`, `TotalCount int`, `PerPage int`|②16章のmeta構造に対応|
|`ListHighSchoolsResult`|`HighSchools []HighSchoolSummary`, `Meta PaginationMeta`|`ListHighSchools`関数の戻り値|
|`GradeSummary`|`ID int`, `Year int`, `DisplayName string`|②6章「GradeSummary」に対応する参照用データ|

## Application関数

### ListHighSchools

- 関数名: `ListHighSchools`
- シグネチャ: `func ListHighSchools(ctx context.Context, db *gorm.DB, params ListHighSchoolsParams) (ListHighSchoolsResult, error)`
- 処理ステップ（②10章の「呼び出す関数」を具体化）:
  1. `infrastructure.SearchHighSchools`を呼び出し、`prefecture_id`絞り込み・`id`昇順・ページング（1ページ20件、②9章）を適用した高校一覧と総件数を取得する
  2. 取得した高校ID群をまとめて`infrastructure.CountMembershipsByHighSchoolIDs`に渡し、高校ごとの生徒数・教員数を一括集計する（②9章「複数高校IDをまとめて渡した一括集計」）
  3. 高校情報と集計結果を突き合わせ、`HighSchoolSummary`の一覧を組み立てる
  4. 総件数からページ情報（`PaginationMeta`）を算出する
  5. `ListHighSchoolsResult`として返す
- 呼び出すinfrastructure関数: `SearchHighSchools`, `CountMembershipsByHighSchoolIDs`
- トランザクション境界: なし（読み取りのみ、②11章）
- 発生しうるApplication Error: なし（一覧取得の失敗はInfrastructure Errorとして扱う。11章参照）

### ShowHighSchool

- 関数名: `ShowHighSchool`
- シグネチャ: `func ShowHighSchool(ctx context.Context, db *gorm.DB, params ShowHighSchoolParams) (HighSchoolSummary, error)`
- 処理ステップ:
  1. `infrastructure.FindHighSchoolByID`を呼び出し、対象高校を取得する
  2. 取得できなかった場合、`ErrHighSchoolNotFound`を返す（②14章「対象高校未存在」）
  3. 対象高校IDを`infrastructure.CountMembershipsByHighSchoolIDs`に渡し、生徒数・教員数を集計する（②10章「詳細取得でも一覧と同じ集計処理を再利用する」）
  4. `HighSchoolSummary`を組み立てて返す
- 呼び出すinfrastructure関数: `FindHighSchoolByID`, `CountMembershipsByHighSchoolIDs`
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrHighSchoolNotFound`（②14章）

### ListGrades

- 関数名: `ListGrades`
- シグネチャ: `func ListGrades(ctx context.Context, db *gorm.DB, params ListGradesParams) ([]GradeSummary, error)`
- 処理ステップ（②10章「高校の存在確認を前提としたうえで、学年一覧を取得する」）:
  1. `infrastructure.FindHighSchoolByID`を呼び出し、対象高校の存在を確認する
  2. 存在しない場合、`ErrHighSchoolNotFound`を返す
  3. `infrastructure.ListGradesByHighSchoolID`を呼び出し、`year`昇順の学年一覧を取得する（②9章）
  4. `GradeSummary`の一覧を組み立てて返す
- 呼び出すinfrastructure関数: `FindHighSchoolByID`, `ListGradesByHighSchoolID`
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrHighSchoolNotFound`

> **②からの補足**: application関数の第2引数として`*gorm.DB`を直接渡す形にした。②はUseCase設計で依存先を「呼び出す関数」としてのみ記載しており、DB接続の受け渡し方式までは規定していない（推測）。Transaction Scriptではusecase struct・DIコンテナを設けないため、関数呼び出し時にDB接続を直接渡すシンプルな形が規約の方針（構造をむやみに増やさない）と整合すると判断した。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

②「9. Repository設計」で定義された3つの責務（HighSchoolRepository / GradeRepository / SchoolMembershipCountRepository）を、それぞれ`infrastructure/`内の関数として実装する（アーキテクチャ規約「Repository Interfaceは定義しない。DBアクセスは`infrastructure/`内の関数から直接行う」）。

### `infrastructure/high_school_query.go`（②9章 HighSchoolRepository相当）

|関数名|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`SearchHighSchools`|`ctx context.Context`, `db *gorm.DB`, `prefectureID *int`, `page int`|`([]HighSchoolModel, totalCount int64, error)`|`prefecture_id`が指定された場合のみ絞り込み条件を付加し、`id`昇順でソート、1ページ20件（②9章）のOFFSET/LIMITページングを行う。総件数は同条件でのCOUNTを別途取得する|
|`FindHighSchoolByID`|`ctx context.Context`, `db *gorm.DB`, `id int`|`(*HighSchoolModel, error)`|`id`一致条件で1件取得する。存在しない場合はレコードなし（`gorm.ErrRecordNotFound`相当）を呼び出し元に伝える|

### `infrastructure/grade_query.go`（②9章 GradeRepository相当）

|関数名|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`ListGradesByHighSchoolID`|`ctx context.Context`, `db *gorm.DB`, `highSchoolID int`|`([]GradeModel, error)`|`high_school_id`一致条件で絞り込み、`year`昇順でソートする（②9章）|

### `infrastructure/membership_count_query.go`（②9章 SchoolMembershipCountRepository相当）

|関数名|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`CountMembershipsByHighSchoolIDs`|`ctx context.Context`, `db *gorm.DB`, `highSchoolIDs []int`|`(map[int]MembershipCount, error)`|`high_school_id IN (...)`で複数高校IDをまとめて絞り込み、ロール（生徒/教員）別に`high_school_id`単位でグルーピング集計する（②9章「複数高校IDをまとめて渡した一括集計（N+1回避のため）」）。戻り値は高校IDをキーとしたマップとし、application層での突き合わせを1回のクエリ結果で完結させる|

`MembershipCount`は`infrastructure/membership_count_query.go`内で定義する補助struct（`StudentCount int`, `TeacherCount int`）とする。

> **②からの補足**: 集計クエリの具体的な条件（ロールをどのカラム・値で判定するか、例えば`users.role = 'student'`等）は①未提供のため参照不可。②9章に記載された「ロール別にカウントする」という方針のみを実装方針として反映し、実際のカラム名・値は既存DBスキーマ（①相当）を別途確認して実装時に確定する必要がある（推測）。

## Entity ⇔ GORMモデルの変換方針

Transaction Script採用のためEntityは存在しない。GORMモデル（`infrastructure/model.go`）から、application層のDTO（`HighSchoolSummary`, `GradeSummary`等）への変換をinfrastructure関数の戻り値受け取り後にapplication関数内で行う。

## 外部連携実装

対象外。②にMail・Cache・Queue等の外部連携は記載されていない（②9〜10章に該当する言及なし）。

---

# 6. Presentation層設計

## Handler

②19章「Railsとの責務対応」で言及される`HighSchoolsController`・`GradesController`という2つのController単位に対応させ、Handlerも2つに分ける。

> **②からの補足**: ②19章はRailsのController名を言及しているのみで、Go側のHandler分割そのものを指示してはいない。ただしControllerの分割単位（高校系・学年系）は業務操作の単位と一致しており、これをそのままHandlerの分割に踏襲することは自然な判断と考え採用した（推測。①Controller実装詳細そのものは未提供のため参照不可）。

### HighSchoolHandler（`presentation/handler/high_school_handler.go`）

- struct名: `HighSchoolHandler`
- 保持する依存: `DB *gorm.DB`
- 対応する呼び出し先: `application.ListHighSchools`, `application.ShowHighSchool`
- メソッド一覧:
  |メソッド|HTTPメソッド|パス|
  |-|-|-|
  |`List(c *gin.Context)`|GET|`/api/v1/admin/high_schools`|
  |`Show(c *gin.Context)`|GET|`/api/v1/admin/high_schools/:id`|
- 処理順序:
  - `List`: クエリパラメータ（`prefecture_id`, `page`）をリクエスト用structへバインド → 型・フォーマットの検証（9章） → `application.ListHighSchools`を呼び出す → 結果を`HighSchoolListResponse`へ変換 → 200で返す。Transaction Script採用のためusecase層を経由しない分、権限チェック済みの前提（Middlewareで完了）を除き、Handler内で追加の呼び出し順序制御は行わない
  - `Show`: パスパラメータ`id`をバインド・検証 → `application.ShowHighSchool`を呼び出す → `ErrHighSchoolNotFound`の場合は404に変換 → 成功時は`HighSchoolResponse`へ変換し200で返す

### GradeHandler（`presentation/handler/grade_handler.go`）

- struct名: `GradeHandler`
- 保持する依存: `DB *gorm.DB`
- 対応する呼び出し先: `application.ListGrades`
- メソッド一覧:
  |メソッド|HTTPメソッド|パス|
  |-|-|-|
  |`List(c *gin.Context)`|GET|`/api/v1/admin/high_schools/:high_school_id/grades`|
- 処理順序: パスパラメータ`high_school_id`をバインド・必須チェック → `application.ListGrades`を呼び出す → `ErrHighSchoolNotFound`の場合は404に変換 → 成功時は`GradeListResponse`へ変換し200で返す

## Request / Response DTO

### Requestパラメータ（Handler内structとして定義。`presentation/request`は設けない。2章参照）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`listHighSchoolsQuery`|`PrefectureID *int \`form:"prefecture_id"\``, `Page int \`form:"page"\``|`prefecture_id`は型（整数）チェックのみで必須ではない。`page`は正の整数であることを検証する（②12章）。バインド失敗時は400|
|パスパラメータ`id` / `high_school_id`|`int`（`uri`タグでバインド）|整数であることを検証する。学年一覧では必須パラメータとして扱う（②12章「`high_school_id`を要求するエンドポイントでのパス必須性」）|

### Response DTO（`presentation/response/`）

|struct名|フィールドと型|備考|
|-|-|-|
|`HighSchoolResponse`|`ID int`, `Name string`, `PrefectureName string`, `StudentCount int`, `TeacherCount int`|②16章のレスポンス項目に対応|
|`MetaResponse`|`CurrentPage int`, `TotalPages int`, `TotalCount int`, `PerPage int`|②16章「meta」に対応|
|`HighSchoolListResponse`|`HighSchools []HighSchoolResponse`, `Meta MetaResponse`|一覧APIのレスポンス全体|
|`GradeResponse`|`ID int`, `Year int`, `DisplayName string`|②16章の学年一覧レスポンス項目に対応|
|`GradeListResponse`|`Grades []GradeResponse`|学年一覧APIのレスポンス全体|

## Routing（`presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/admin/high_schools`|`HighSchoolHandler.List`|
|GET|`/api/v1/admin/high_schools/:id`|`HighSchoolHandler.Show`|
|GET|`/api/v1/admin/high_schools/:high_school_id/grades`|`GradeHandler.List`|

いずれも、認証Middleware・admin役割確認Middleware配下のルートグループに登録する（②13章）。認証・admin確認Middleware自体の実装は本機能のスコープ外であり、既存の共通Middleware（他機能で使用しているものと同一）を利用する想定とする。

> **②からの補足**: 共通Middlewareの具体的なpackage名・関数名は②・①いずれにも記載がなく、①未提供のため参照不可。ルーティング登録時に既存の認証基盤を利用する旨のみを方針として明記する。

---

# 7. API仕様

②16章のAPI互換方針に基づくEndpoint一覧。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|`/api/v1/admin/high_schools`|`HighSchoolHandler.List`|`prefecture_id`（任意, int）, `page`（任意, int）|`HighSchoolListResponse`|200|
|GET|`/api/v1/admin/high_schools/:id`|`HighSchoolHandler.Show`|パスパラメータ`id`（int）|`HighSchoolResponse`|200|
|GET|`/api/v1/admin/high_schools/:high_school_id/grades`|`GradeHandler.List`|パスパラメータ`high_school_id`（int）|`GradeListResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（既存の共通エラー形式。②16章「既存のエラー形式を踏襲する」）|
|adminロールでない|403|認可エラー（既存の共通エラー形式）|
|`page`, `prefecture_id`, `id`, `high_school_id`が数値でない|400|入力形式エラー|
|`high_school_id`が指定されない（学年一覧）|400|パス必須パラメータ不足エラー|
|指定`id`/`high_school_id`の高校が存在しない|404|`ErrHighSchoolNotFound`（②14章「対象高校未存在」）|
|DB接続失敗・クエリ失敗|500|Infrastructure Error（②14章）|

---

# 8. Transaction実装方針

②「11. Transaction設計」のとおり、本機能はすべて読み取り専用処理であり、明示的なトランザクションを使用しない。

- Transaction開始箇所: なし
- Transaction終了箇所（Commit/Rollback条件）: 該当なし
- 複数infrastructure関数にまたがる場合の扱い: `ListHighSchools`・`ShowHighSchool`はそれぞれ複数のinfrastructure関数（高校検索/詳細取得 + 集計）を呼び出すが、読み取りのみのため、`db.Transaction(...)`等によるトランザクション制御は行わず、各infrastructure関数は独立したクエリとして順次実行する（②11章「保護すべき整合性が存在せず、不要な複雑さを増やすだけ」）

---

# 9. Validation実装方針

## Presentation

②「12. Validation設計」のPresentation節を実装レベルに落とし込む。

- 型チェック: `prefecture_id`, `page`, `id`, `high_school_id`をGinのクエリ/パスバインド時に整数として検証する（バインド失敗時は400）
- 必須チェック: `id`（高校詳細）・`high_school_id`（学年一覧）はパスパラメータとして必須。ルーティング定義上、値が欠落するとGinのルーティング自体が一致しないため、バインド後の型検証で担保する
- フォーマットチェック: `page`が1以上の正の整数であることをHandler内のガード節で検証する。不正な場合は400を返す

## 業務ルール検証

Transaction Script採用時のため、application関数内のガード節で検証する（②12章「Domain: 状態チェック: 対象高校が存在するかどうかの確認（Application層のエラーとして扱う）」）。

- `ShowHighSchool`・`ListGrades`は、`infrastructure.FindHighSchoolByID`の結果が存在しない場合、`ErrHighSchoolNotFound`を返すガード節を持つ
- `ListHighSchools`は絞り込み条件（`prefecture_id`）が任意項目であり、必須の業務整合性チェックを持たない（②12章）

---

# 10. Authorization実装方針

②「13. Authorization設計」を実装レベルに落とし込む。

- Middleware: 認証済みユーザーを特定しコンテキストに保持する。役割が`admin`であることを確認し、`admin`でない場合は403で処理を打ち切る（②13章）
- Handler: パスパラメータ（`id`, `high_school_id`）をapplication関数に受け渡すのみで、権限判定は行わない（②13章）
- application関数: 追加のスコープ制限は行わない。管理者はすべての高校・学年を参照可能なため、`current admin`を用いたフィルタ処理は実装しない（②13章「UseCase: 追加のスコープ制限は行わない」）
- Domain: 該当なし（Domain層を設けないため）

---

# 11. Error実装方針

②「14. Error設計」を実装レベルに落とし込む。

## Domain Error → Application Errorへの変換方針

Domain Errorは定義しない（②14章、③3章）。Application Errorは`application/errors.go`にsentinel error（またはそれに相当するerror値）として定義する。

- `ErrHighSchoolNotFound`: 対象高校が存在しない場合に、`ShowHighSchool`・`ListGrades`から返される（②14章「対象高校未存在（一覧・詳細・学年一覧すべてに共通し得る）」）

> **②からの補足**: ②は「一覧・詳細・学年一覧すべてに共通し得る」と記載しているが、10章のUseCase設計を確認すると、一覧（`ListHighSchools`）自体は特定の高校IDを入力に取らないため、存在確認エラーが発生する余地がない。本書では②10章の入出力定義に合わせ、`ErrHighSchoolNotFound`は`ShowHighSchool`・`ListGrades`のみで発生しうるものとして扱う（②の記述間の整合を取るための判断であり、②の業務ルールを追加・変更するものではない）。

コーディング規約6章に従い、エラーは関数の最後の戻り値として返し、`errors.New`または`fmt.Errorf`のいずれかをプロジェクト内の既存の使い分けに合わせて用いる（規約21章「変更しない方針」）。

## Application Error → HTTPレスポンスへの変換方針

Handler側で`errors.Is(err, application.ErrHighSchoolNotFound)`により判定し、該当する場合は404、それ以外のエラー（infrastructure由来）は500に変換する。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrHighSchoolNotFound`|Application（application関数のガード節）|404|
|クエリバインド・型検証エラー|Presentation（Handler）|400|
|認証エラー|Middleware|401|
|認可エラー（adminでない）|Middleware|403|
|DB接続失敗・クエリ実行エラー|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

infrastructure関数が返すDBエラー（接続失敗・クエリ失敗）は、application関数でラップせずにそのまま呼び出し元へ伝播させ（コーディング規約6章の`%w`によるラップを推奨）、Handlerで`ErrHighSchoolNotFound`以外のエラーとして一律500に変換する（②14章「技術的な障害を業務エラーと切り分ける」）。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」のとおり、既存Rails DBを継続利用し、Schema変更は行わない。

## 利用するGORMモデルとテーブルの対応

`infrastructure/model.go`に、以下のGORMモデルを定義する。

|struct名|対応テーブル|備考|
|-|-|-|
|`HighSchoolModel`|`high_schools`|GORMの複数形命名規則（Gorm規約）によりテーブル名は自動解決される想定。`id`, `name`, `prefecture_id`を保持する|
|`GradeModel`|`grades`|`id`, `high_school_id`, `year`を保持する|
|`UserModel`|`users`|集計専用の参照に用いる。`id`, `high_school_id`, ロールを判別するためのカラムを保持する|

> **②からの補足**: 各モデルの正確なフィールド一覧・カラム名（特に`prefecture_name`の取得方法、ロール判別カラムの名称と値）は①未提供のため参照不可。②16章のレスポンス項目（`prefecture_name`, `student_count`, `teacher_count`等）から逆算して必要な情報を列挙したが、実装時は既存DBスキーマを別途確認し、モデル定義を確定する必要がある（推測）。

## 主要クエリの条件・ソート・ページネーション方針

- 高校一覧: `prefecture_id`が指定された場合はWHERE条件として付加、`id`昇順ソート、1ページ20件のページング（②9章）。総件数取得は同一条件でのCOUNTクエリを別途発行する
- 高校詳細: `id`一致条件での単一レコード取得
- 学年一覧: `high_school_id`一致条件、`year`昇順ソート
- 生徒数・教員数集計: 複数`high_school_id`をIN条件でまとめて絞り込み、ロール別に`high_school_id`単位でグルーピング集計する一括クエリ（N+1回避、②9章）

SQL文そのものはここに記載しない。GORMのクエリビルダ（`Where`, `Order`, `Limit`/`Offset`, `Group`等に相当する操作）を用いて実装する。

## 既存Schemaに対する変更

②17章のとおり変更なし。`high_schools`, `grades`, `users`の既存カラムのみで一覧・詳細・集計・学年参照のすべてを満たす。

---

# 13. テストケース設計

②「18. テスト戦略」を、アーキテクチャ規約・指示書のTransaction Script読み替えルール（「Domain Test」「Repository Test」は対象外、「UseCase Test」→「Application関数 Test」）に従って具体化する。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test（②の「UseCase Test」に相当）

|対象|テストケース|
|-|-|
|`ListHighSchools`|`prefecture_id`未指定時に全件が`id`昇順で返ること／`prefecture_id`指定時に絞り込まれること／ページングにより指定ページの件数のみ返ること／各高校の`StudentCount`・`TeacherCount`が集計結果と一致すること／`Meta`のページ情報が総件数と一致すること|
|`ShowHighSchool`|存在する`high_school_id`で詳細と集計値が返ること／存在しない`high_school_id`で`ErrHighSchoolNotFound`が返ること|
|`ListGrades`|存在する`high_school_id`で`year`昇順の学年一覧が返ること／存在しない`high_school_id`で`ErrHighSchoolNotFound`が返ること|

## Repository Test

対象外（Transaction Script採用のため、Repository Interfaceを設けない。指示書の読み替えルールに従う）。

> **②からの補足**: ただし②18章「Repository Test」は「HighSchoolRepository・GradeRepository・SchoolMembershipCountRepositoryによる検索・集計クエリの正確性を検証する」ことを目的として挙げており、この検証自体の必要性は②の設計判断に含まれる。Transaction Script採用によりRepository Interfaceという単位そのものが存在しないため見出しとしては対象外とするが、②が求める検証内容は下記「Infrastructure関数のクエリ正確性検証」としてIntegration Test（またはinfrastructure/配下の`_test.go`）で実施する。これは指示書の章立てを変更せず、②の要求内容を欠落させないための実装上の補足である。

Infrastructure関数のクエリ正確性検証（`infrastructure/*_test.go`として実装）:

|対象|テストケース|
|-|-|
|`SearchHighSchools`|`prefecture_id`絞り込み条件が正しく適用されること／`id`昇順ソートが正しいこと／ページング（20件区切り）が正しいこと／総件数取得が正しいこと|
|`FindHighSchoolByID`|存在するIDで正しいレコードが返ること／存在しないIDでレコードなしとなること|
|`ListGradesByHighSchoolID`|指定`high_school_id`の学年のみ返ること／`year`昇順であること|
|`CountMembershipsByHighSchoolIDs`|複数高校IDを渡した際に高校ごとの生徒数・教員数が正しく集計されること／該当ユーザーがいない高校でカウント0が扱えること|

## Handler Test

|対象|テストケース|
|-|-|
|`HighSchoolHandler.List`|クエリパラメータの解釈（`prefecture_id`, `page`）が正しいこと／不正な`page`で400が返ること／正常系で200と期待するレスポンス構造が返ること|
|`HighSchoolHandler.Show`|存在しない`id`で404が返ること／不正な`id`（数値以外）で400が返ること／正常系で200が返ること|
|`GradeHandler.List`|存在しない`high_school_id`で404が返ること／正常系で200が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/admin/high_schools`|エンドポイント経由で一覧が正しいレスポンス構造（`high_schools`, `meta`）で返却されること|
|`GET /api/v1/admin/high_schools/:id`|エンドポイント経由で詳細が正しく返却されること／存在しないIDで404が返ること|
|`GET /api/v1/admin/high_schools/:high_school_id/grades`|エンドポイント経由で学年一覧が正しく返却されること|
|認可|admin以外のロールでアクセスした場合に403が返ること／未認証の場合に401が返ること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に整理する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|`internal/`配下のディレクトリ名を`school_directory`とする|②のContext名`school-directory`はkebab-case表記のみで、ディレクトリ名の指定がないため、指示書のActive Record例（`teacher-directory`→`teacher_directory`）と同じ変換規則を適用した|推測|
|`presentation/request/`ディレクトリを設けず、クエリ/パスパラメータ用structをHandlerと同一package内に定義する|指示書のTransaction Script構造例に`request/`が含まれておらず、本機能の入力構造が単純なため|推測|
|`application/dto.go`・`application/errors.go`・`infrastructure/model.go`という補助ファイルへの分割|②・指示書ともにファイル粒度までは規定していないため、責務ごとに集約する分割を採用した|推測|
|application関数に`current admin`を引数として渡さない|②13章「UseCaseで追加のスコープ制限は行わない」という判断に基づき、認可情報はMiddleware/Handlerで完結させ、application関数の入力から除外した|②の判断に基づく具体化（推測要素は低い）|
|Handlerを`HighSchoolHandler`・`GradeHandler`の2つに分割する|②19章のRailsとの対応表で`HighSchoolsController`・`GradesController`という2つのController単位が言及されていることを踏まえた（①Controller実装詳細そのものは未提供のため参照不可）|推測|
|`ErrHighSchoolNotFound`を`ShowHighSchool`・`ListGrades`のみで発生しうるものとして扱い、`ListHighSchools`では対象外とする|②14章は「一覧・詳細・学年一覧すべてに共通し得る」と記載するが、②10章のUseCase入出力定義では一覧が特定の高校IDを入力に取らず、存在確認エラーが発生する余地がないため、②内の記述の整合を取った|②の記述間の整合を取るための判断|
|GORMモデル（`HighSchoolModel`, `GradeModel`, `UserModel`）の正確なフィールド一覧・ロール判別カラム名は確定できない|①（Railsの既存DBスキーマ）が未提供のため参照不可。②16章のレスポンス項目から必要な情報を逆算したのみ|推測（要・既存DBスキーマ確認）|
|認証・admin確認Middlewareの具体的なpackage名・実装|①・②いずれにも記載がなく、既存の共通Middlewareを利用する方針のみを明記した|①未提供のため参照不可|
|application関数の第2引数として`*gorm.DB`を直接渡す|②はDB接続の受け渡し方式を規定していないため、Transaction Script構造に沿ったシンプルな受け渡し方法を採用した|推測|

上記以外の設計判断（採用パターン・Bounded Context・Repository/UseCase設計・Transaction/Validation/Authorization/Error設計・API互換方針・DB方針・テスト戦略）は、すべて②の記載どおりであり、本書での変更は行っていない。
