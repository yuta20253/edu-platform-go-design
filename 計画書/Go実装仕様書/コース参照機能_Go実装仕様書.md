# コース参照機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

学習コースと、それに紐づく単元の一覧を取得する参照専用機能である。`subject`クエリパラメータによる絞り込みに対応し、コースに含まれる単元情報を合わせて返却する（②「1. 機能概要」より要約）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」よりTransaction Scriptを採用する。状態管理・権限制御・業務ルールのいずれも現行仕様に存在せず、CourseとUnitは更新や状態遷移を伴わない単純なマスタデータであるためである。②「5. Aggregate設計」「7. Value Object設計」「8. Domain Service」もそれぞれ不要と判断されており、本書もこれを前提とする。

## 本書が対象とする実装範囲

- `GET /api/v1/student/courses` エンドポイントの実装
- application層の関数（`ListCourses`）
- infrastructure層のDBアクセス関数（`FindCoursesWithUnits`）
- presentation層のHandler・Request/Response DTO・Routing
- 既存DB（`courses`・`units`テーブル）をそのまま利用し、Schema変更は行わない（②「17. DB設計方針」）

本書は②で確定した設計方針（Transaction Script採用、Aggregate/Value Object/Domain Service/Domain Event不採用、権限制御なし）を変更しない。①Rails実装の詳細は本タスクでは提供されていないため、実装の根拠として必要な箇所は「①未提供のため参照不可」として扱う。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `curriculum`（②「3. Bounded Context」、アーキテクチャ規約「5. Bounded Context構成」の機能Context一覧にも`curriculum`として登録済み）

## ②で採用した設計パターン

- Transaction Script（②「4. 設計パターン」）

## ディレクトリ一覧（採用パターンに対応する構造）

アーキテクチャ規約「4. 設計パターンごとの構造適用方針」のTransaction Script構造に従う。

```
internal/curriculum/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    └── response/
```

**②からの補足**: ②はBounded Context名を`curriculum`と定義しているのみで、`internal/`配下のディレクトリ名までは明記していない。アーキテクチャ規約「9. 命名規約」に従い、Context名をそのまま英単語1語のディレクトリ名`internal/curriculum`として採用した。

## 作成するファイル一覧

```
internal/curriculum/application/list_courses.go
internal/curriculum/infrastructure/course_query.go
internal/curriculum/presentation/handler/course_handler.go
internal/curriculum/presentation/response/course_response.go
internal/curriculum/presentation/routes.go
```

**②からの補足**: Transaction Script構造にはRequest DTO専用ディレクトリ（`presentation/request/`）が規約の例に含まれていない。クエリパラメータをバインドする小さなstructは`presentation/handler/course_handler.go`内に定義する方針とした（規約「4. 設計パターンごとの構造適用方針」のTransaction Script構成例に基づく判断。②に明記なし）。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層を設けない）。Entity・Value Object・Repository Interface・Domain Service・Domain Event・Domain Errorに相当する内容は設けず、関数の入出力・処理ステップは「4. Application層設計」に記載する。

②「6. Entity設計」で言及されているCourse・Unitの属性（`id, subject_id, level_number, level_name, description`／`id, course_id, unit_name`）は、Domain Entityとしてではなく、後述のinfrastructure層のクエリ結果を表すstructおよびapplication層のDTOのフィールドとして反映する。

---

# 4. Application層設計

**実装上の位置づけ**: Transaction Script採用のため、UseCase struct・Repository Interfaceは設けない。`application/`直下に置く関数として実装する（②「10. UseCase設計」の実装上の位置づけと同じ）。

## DTO（Command / Query）

|struct名|フィールドと型|Command/Query|
|-|-|-|
|`ListCoursesInput`|`Subject *string`|Query（入力）|
|`CourseResult`|`ID uint`、`SubjectID uint`、`LevelNumber int`、`LevelName string`、`Description string`、`Units []UnitResult`|Query（出力）|
|`UnitResult`|`ID uint`、`CourseID uint`、`UnitName string`|Query（出力）|

**②からの補足**: 各フィールドの型（`uint`/`int`/`string`等）は②「6. Entity設計」の属性名（`subject_id`・`level_number`・`level_name`・`description`・`unit_name`）から一般的なGo/GORMの型対応として推測した。①未提供のため実際のカラム型は参照不可であり、実装時にDB定義と突き合わせて確認が必要（推測）。

`Subject`を`*string`とした理由: ②「12. Validation設計」Presentationで「必須チェック: なし（subjectは任意項目）」とされているため、未指定を`nil`として区別できるポインタ型を採用した（②からの補足、推測ではなく②の必須チェック方針からの直接の実装反映）。

## Application関数: ListCourses

- 関数名: `ListCourses`
- 引数: `ctx context.Context, input ListCoursesInput`
- 戻り値: `([]CourseResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `infrastructure.FindCoursesWithUnits(ctx, input.Subject)` を呼び出し、コースと単元をまとめて取得する
  2. 取得した`infrastructure.CourseRecord`のスライスを`CourseResult`（および内包する`UnitResult`）へ変換する
  3. 変換結果を返す
- 呼び出すinfrastructure関数: `FindCoursesWithUnits`
- トランザクション境界: なし（②「11. Transaction設計」より、読み取りのみでトランザクション不要）
- 発生しうるApplication Error: `infrastructure.FindCoursesWithUnits`が返すInfrastructure Error（DB接続失敗等）をラップしたApplication Error相当のエラー（②「14. Error設計」、アーキテクチャ規約「8. 横断的関心事の置き場所」の“Transaction Script/Active Record採用時はDomain Errorに相当する層がないため、関数・struct側で発生したエラーをApplication Error相当として扱う”に従う）

②「10. UseCase設計」で言及されている「業務ルール判定」は存在しないため、`ListCourses`内にガード節以上の業務ロジックは持たせない。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

- 関数名: `FindCoursesWithUnits`
- 引数: `ctx context.Context, subjectID *uint`
- 戻り値: `([]CourseRecord, error)`
- 発行するクエリ内容:
  - `subjectID`が`nil`でない場合、`subject_id`カラムに対する等価条件で絞り込む（②「9. Repository設計」“subject_idによる絞り込み”）
  - `subjectID`が`nil`の場合は絞り込みなしで全件取得する
  - 関連するUnitはGORMのEager Loading（Preload）を用いて一括取得し、N+1問題を避ける（②「9. Repository設計」“関連するUnitの一括取得（N+1を避けるための取得方法を含む）”の具体化）
  - ソート順: ②・①いずれにも明記がないため、本書では規定しない。①未提供のため参照不可であり、実装時に主キー（`id`）昇順をデフォルトとするか、業務要件を別途確認する必要がある（推測、要確認。詳細は「14. ②からの補足事項」参照）

**②からの補足**: 引数名を`subjectID *uint`としたが、②「16. API仕様」ではRequestパラメータを`subject`（文字列表記）としている。`subject`クエリパラメータの値がそのまま`subject_id`カラムの値（数値）を表すのか、文字列コードから変換が必要なのかは②に明記がなく、①未提供のため参照不可。本書ではAPI境界（クエリパラメータ）は文字列、`courses.subject_id`カラムとの比較時に数値へ変換する前提で設計したが、この変換要否自体は推測であり、実装時の確認事項として「14. ②からの補足事項」に記載する。

## GORMモデル（Course/Unitクエリ結果を表すstruct）

`infrastructure/course_query.go`に、`FindCoursesWithUnits`の戻り値として使用するGORMモデルを定義する。

|struct名|対応テーブル|フィールド|
|-|-|-|
|`CourseRecord`|`courses`|`ID uint`（主キー）、`SubjectID uint`、`LevelNumber int`、`LevelName string`、`Description string`、`Units []UnitRecord`（`course_id`で関連付くhas many）|
|`UnitRecord`|`units`|`ID uint`（主キー）、`CourseID uint`、`UnitName string`|

Gorm規約に従い、構造体名`CourseRecord`／`UnitRecord`はテーブル名`courses`／`units`と一致しないため、`TableName()`メソッドで明示的にテーブル名を指定する（Gorm規約「テーブル名」節）。カラム名は構造体フィールド名のsnake_caseとの規約準拠を基本とし、`SubjectID`→`subject_id`等、命名規則上ズレが生じないため`column`タグの追加は不要と判断する。

**②からの補足**: ②「6. Entity設計」はEntityとしてCourse/Unitを定義しているが、Transaction Script採用のためDomain Entityは設けず、代わりにinfrastructure層のクエリ結果structとして同等の属性を保持させた。この対応関係自体はアーキテクチャ規約「10. Railsとの対応」には明記がないため、②からの補足として明記する。

## 外部連携実装

対象外。Mail・Cache・Queue等の外部連携は②のいずれの章にも記載がない。

---

# 6. Presentation層設計

## Handler

- struct名: `CourseHandler`
- 対応する呼び出し先: `application.ListCourses`関数（Transaction Script採用のため、UseCaseではなくapplication関数を直接呼び出す）
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`ListCourses`|GET|`/api/v1/student/courses`|

- 処理順序:
  1. クエリパラメータ`subject`を`ListCoursesRequestQuery`にバインドする（`c.ShouldBindQuery`相当）
  2. Validation: `subject`が文字列としてバインド可能であることを確認する（型チェックのみ。②「12. Validation設計」より必須チェック・フォーマットチェックは不要）
  3. `application.ListCoursesInput`へ変換する（空文字列は`nil`として扱う）
  4. `application.ListCourses(ctx, input)`を呼び出す
  5. 戻り値の`[]application.CourseResult`を`CourseListResponse`へ変換する
  6. HTTP 200でJSONレスポンスを返す。エラー発生時は「11. Error実装方針」に従いHTTPステータスへ変換する

②「13. Authorization設計」より、HandlerでもUseCase相当の関数でも業務権限判定は行わない。認証確認はMiddlewareで完結する。

## Request / Response DTO

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`ListCoursesRequestQuery`|`Subject string` `form:"subject"`|必須タグなし（任意項目）。型は文字列そのものであるため追加のフォーマットチェックは行わない（②「12. Validation設計」）|
|`CourseListResponse`|`Courses []CourseResponse`|-|
|`CourseResponse`|`ID uint`、`SubjectID uint`、`LevelNumber int`、`LevelName string`、`Description string`、`Units []UnitResponse`|-|
|`UnitResponse`|`ID uint`、`CourseID uint`、`UnitName string`|-|

②「16. API仕様」のResponse定義（コース属性: id, subject_id, level_number, level_name, description ＋ units: id, course_id, unit_name）をそのままフィールド構成に反映した。

## Routing

`internal/curriculum/presentation/routes.go`に以下を登録する。

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/student/courses`|`CourseHandler.ListCourses`|

認証Middleware（本人確認）を経由させる（②「13. Authorization設計」Middleware節）。ロール（student）レベルの粗い制御が既存の認証/認可Middlewareで行われている前提とするが、①未提供のため既存Middlewareの実装詳細は参照不可。本機能固有の追加Middlewareは不要。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/student/courses|CourseHandler.ListCourses|`ListCoursesRequestQuery`（`subject`任意）|`CourseListResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証（認証Middlewareで検証失敗）|401|認証エラー（既存の認証Middleware共通レスポンス形式に準拠。本機能固有のエラー内容ではない）|
|DB接続失敗等、想定外のInfrastructure Error|500|既存形式に準じたエラーレスポンス（②「16. API仕様」Error Responseより）|

②「16. API仕様」Error Responseに記載のとおり、本機能では業務エラーがほぼ発生しないため、上記2件以外のケースは想定しない。

---

# 8. Transaction実装方針

- Transaction開始箇所: なし。`ListCourses`関数（application層）はGORMのトランザクションを開始しない
- Transaction終了箇所（Commit/Rollback条件）: 該当なし
- 複数関数にまたがる場合の扱い: 本機能はinfrastructure関数を1つ（`FindCoursesWithUnits`）しか呼び出さないため、複数の永続化操作をまたぐ整合性管理は発生しない

②「11. Transaction設計」の「状態を変更する処理が存在しないため、トランザクションによる整合性保証は不要である」という判断をそのまま実装レベルに反映した。

---

# 9. Validation実装方針

## Presentation

- `ListCoursesRequestQuery.Subject`について、クエリパラメータの型が文字列であることを確認する（②「12. Validation設計」Presentation“型チェック: subjectが文字列であることを検証する”）
- 必須チェック: なし（`subject`は任意項目）
- フォーマットチェック: なし

## 業務ルール検証（Transaction Script採用時: application関数内のガード節）

- ②「12. Validation設計」Domainより「業務ルール: 特になし」「状態チェック: 特になし」「整合性チェック: 特になし」とされているため、`ListCourses`関数内に業務ルール検証のガード節は設けない
- `input.Subject`が空文字列の場合は「絞り込みなし」として扱い、`nil`に正規化したうえでinfrastructure関数に渡す（②に明記のない入力正規化のため「14. ②からの補足事項」に記載する）

---

# 10. Authorization実装方針

- Middlewareで行う処理: 認証済みユーザーであることの確認のみ（②「13. Authorization設計」Middleware）
- Handlerで行う処理: 業務権限判定は行わない（②「13. Authorization設計」Handler）
- application関数で行う処理: 権限判定は行わない。全コースが参照対象であるため（②「13. Authorization設計」UseCase）

②「13. Authorization設計」の判断理由（Rails現行仕様書で「権限制御: なし（すべてのコースを取得可能）」と明記されている）をそのまま踏襲する。①Rails実装コード自体は未提供のため参照不可だが、②内で引用された仕様書上の記載として扱う。

---

# 11. Error実装方針

- Domain Error → Application Errorへの変換方針: 該当なし。Transaction Script採用のためDomain Errorに相当する層を持たない（②「14. Error設計」Domain Error“原則として使用しない”）
- Application Error → HTTPレスポンスへの変換方針: `application.ListCourses`が返すエラー（infrastructure由来のエラーをラップしたもの）は、Handlerで一律500 Internal Server Errorとして既存共通形式のエラーレスポンスに変換する
- Infrastructure Errorのハンドリング方針: DB接続失敗等はinfrastructure関数からエラーとして返却し、application層でラップしてHandlerへ伝播させる。业務ロジック側で個別のリカバリは行わない

|Error種別|発生層|HTTP Status|
|-|-|-|
|未認証エラー|Middleware（Presentation手前）|401|
|Infrastructure Error（DB接続失敗等）|infrastructure|500|
|その他の想定外エラー|application|500|

②「14. Error設計」の「Application Error・Infrastructure Errorを中心としたシンプルなエラー設計とする」という方針をそのまま反映した。

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

|GORMモデル|テーブル|
|-|-|
|`CourseRecord`|`courses`|
|`UnitRecord`|`units`|

## 主要クエリの条件・ソート・ページネーション方針

- 条件: `subjectID`が指定された場合のみ`subject_id`カラムへの等価条件を付与する（②「9. Repository設計」）
- 関連取得: `Units`をPreloadにより同時取得し、N+1を回避する（②「9. Repository設計」の具体化）
- ソート: ②・①いずれにも明記なし。①未提供のため参照不可。実装時に主キー昇順をデフォルトとするか、業務側で仕様を確認する必要がある（推測、要確認）
- ページネーション: ②に記載なし。現行仕様（①未提供のため参照不可）でページネーションの有無が確認できないため、本書では「なし（全件取得）」として扱う（推測）

## 既存Schemaへの変更

②「17. DB設計方針」より、`courses`・`units`テーブルへの変更は提案されていない。本機能の実装においてもSchema変更は行わない。

SQL文そのものはここには記載しない。

---

# 13. テストケース設計

**実装上の位置づけ**: Transaction Script採用のため、②「18. テスト戦略」の区分をアーキテクチャ規約「13. テストケース設計」の読み替え方針に従い変換する。「Domain Test」「Repository Test」は対象外、「UseCase Test」は「Application関数Test」に読み替える。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数Test

|対象|テストケース|
|-|-|
|`ListCourses`|`Subject`が`nil`の場合、全コースと各コースに紐づく単元が取得できること|
|`ListCourses`|`Subject`が指定された場合、該当科目のコースのみ取得できること（②「18. テスト戦略」UseCase Test“subjectの有無によるコース一覧取得結果の違いを検証する”）|
|`ListCourses`|`infrastructure.FindCoursesWithUnits`がエラーを返した場合、エラーがそのまま（またはラップされて）返却されること|

## Repository Test

対象外（Transaction Script採用のため、Repository Interfaceを設けない）。代わりにInfrastructure関数のテストを以下に記載する。

|対象|テストケース|
|-|-|
|`FindCoursesWithUnits`|`subjectID`が`nil`の場合、全件のコースが取得できること|
|`FindCoursesWithUnits`|`subjectID`が指定された場合、該当`subject_id`のコースのみ取得できること（②「18. テスト戦略」Repository Test“CourseRepositoryの検索条件（subject絞り込み・単元の同時取得）の正確性を検証する”）|
|`FindCoursesWithUnits`|各コースに紐づく`Units`が正しくPreloadされていること（N+1が発生しないことの確認を含む）|
|`FindCoursesWithUnits`|該当コースが0件の場合、空スライスが返ること|

## Handler Test

|対象|テストケース|
|-|-|
|`CourseHandler.ListCourses`|`subject`クエリパラメータなしでリクエストした場合、200と全コース一覧が返ること|
|`CourseHandler.ListCourses`|`subject`クエリパラメータありでリクエストした場合、200と絞り込まれたコース一覧が返ること|
|`CourseHandler.ListCourses`|`application.ListCourses`がエラーを返した場合、500と既存形式のエラーレスポンスが返ること（②「18. テスト戦略」Handler Test“リクエストパラメータの検証結果とレスポンス形式を検証する”）|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/student/courses`|認証済みユーザーがエンドポイントを呼び出した場合、コース一覧と各コースの単元情報が正しいレスポンス構造で取得できること（②「18. テスト戦略」Integration Test）|
|`GET /api/v1/student/courses`|`subject`パラメータ付きで呼び出した場合、絞り込まれた結果が返ること|
|`GET /api/v1/student/courses`|未認証で呼び出した場合、401が返ること|

---

# 14. ②からの補足事項

②に明記がなく、本書作成のために実装レベルで追加判断した内容を以下に整理する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|`internal/curriculum`というディレクトリ名を採用した|②はContext名`curriculum`のみを定義しており、`internal/`配下のパスは明記していないため、アーキテクチャ規約「9. 命名規約」に従いContext名をそのままディレクトリ名とした|推測ではなく規約適用による直接的な判断|
|Request用struct（`ListCoursesRequestQuery`）を`presentation/handler`内に配置した|アーキテクチャ規約のTransaction Script構造例に`presentation/request/`ディレクトリが含まれていないため|推測ではなく規約の構造例に基づく判断|
|`CourseResult`/`UnitResult`/`CourseRecord`/`UnitRecord`等のDTO・GORMモデルの具体的なフィールド型（`uint`/`int`/`string`）|②はEntityの属性名のみを定義しており、Go/DB上の型までは記載していないため、一般的な対応関係から補った|推測（①未提供のため実際のカラム型は参照不可。実装時にDB定義との突き合わせが必要）|
|`subject`クエリパラメータ（文字列）と`courses.subject_id`カラム（数値属性）との対応関係・変換要否|②「3. Bounded Context」で科目情報の詳細テーブルが存在しないことが述べられているのみで、クエリパラメータとカラムの対応関係は明記がない|推測（①未提供のため参照不可。実装着手前に業務側・DB定義側の確認が必要な要検討事項）|
|コース一覧取得時のソート順（未定義のため主キー昇順を暫定案として提示）|②・①いずれにも記載がないため|推測（要確認）|
|ページネーションの有無（「なし・全件取得」として扱う）|②に記載がなく、①も未提供のため確認できないため|推測|
|`input.Subject`が空文字列の場合は「絞り込みなし」として`nil`に正規化するガード節|②「12. Validation設計」に業務ルールの記載がないため、Transaction Script構造上必要な最小限の入力正規化として補った|推測ではなく実装上必要な最小限の処理として補足|
|Application Error・Infrastructure Errorの発生層とHTTP Statusの対応表（401/500）|②「14. Error設計」「16. API仕様」は方針レベルの記載にとどまるため、アーキテクチャ規約「8. 横断的関心事の置き場所」のError変換方針に従い具体化した|推測ではなく規約適用による具体化|

上記以外の設計判断（採用パターン・Bounded Context・Aggregate/Value Object/Domain Service/Domain Event不採用・権限制御方針・DB方針等）はすべて②の記載をそのまま踏襲しており、変更は行っていない。
