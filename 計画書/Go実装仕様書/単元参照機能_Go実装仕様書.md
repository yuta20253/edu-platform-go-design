# 単元参照機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒が自分のタスクに紐づく単元の詳細情報（単元名・所属コース情報）を取得する参照専用機能である。指定タスクの所有権確認と、そのタスクへの単元の紐付き確認という2段階のアクセス制御を経て、単元情報を返却する。状態変更・作成・更新は行わない。

## 採用設計パターンとその理由（②からの要約）

- **採用パターン**: Transaction Script
- **理由**: 「タスクの所有権確認→紐づく単元の検索→返却」という一連の手続きのみで構成される参照系機能であり、状態管理・状態遷移が存在しない。単元はマスタデータであり、業務ルールを持って振る舞うEntityとして扱う必要がない。業務ルールも「タスクに紐づかない単元は取得できない」という単純なアクセス制御ルールのみである（②「4. 設計パターン」）。
- Aggregate・Value Object・Domain Service・Domain Eventはいずれも②で「不要」と判断されている（②「5〜8, 15」）。

## 本書が対象とする実装範囲

- 対象Bounded Context: `curriculum`（②「3. Bounded Context」。コース参照機能と同一Contextとして扱う）
- 対象UseCase: `ShowTaskUnit`（タスクに紐づく単元詳細の取得。②「10. UseCase設計」）
- 本書は上記1機能分の実装単位（package構成・関数シグネチャ・クエリ内容・API仕様・Validation/Authorization/Error/テスト方針）を対象とする。コース参照機能側で既に定義済みの構成要素（Unit/CourseのGORMモデル等）は新規に再定義せず、利用する前提として扱う（詳細は本書「2. ディレクトリ構成」「12. GORM / DBクエリ設計」で補足する）。
- ①Railsの実装詳細は本タスクでは提供されていないため、①を根拠とする記載は行わず、必要箇所は「①未提供のため参照不可」として扱う。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `curriculum`（②「3. Bounded Context」）

## ②で採用した設計パターン

- Transaction Script（②「4. 設計パターン」）

規約「アーキテクチャ規約.md」4章「設計パターンごとの構造適用方針 > Transaction Script」に従い、domain層・usecase層（struct/interface）・Repository Interfaceは設けない。1つの業務操作を`application/`直下の関数として実装し、DBアクセスは`infrastructure/`内の関数から直接（または他Contextが公開する参照関数を経由して）行う。

## 作成するディレクトリ一覧

```
internal/curriculum/application/
internal/curriculum/infrastructure/
internal/curriculum/presentation/handler/
internal/curriculum/presentation/response/
```

`internal/curriculum/presentation/routes.go` は、コース参照機能側で既にルーティングファイルが存在する場合はそこに本機能のルートを追記する。存在しない場合は新規作成する（②からの補足：どちらが先行実装されるかは②に記載がなく、実装順序に依存するため両ケースを明記する）。

## 作成するファイル一覧

```
internal/curriculum/application/show_task_unit.go
internal/curriculum/infrastructure/task_unit_query.go
internal/curriculum/presentation/handler/task_unit_handler.go
internal/curriculum/presentation/response/task_unit_response.go
internal/curriculum/presentation/routes.go        # 追記または新規（上記のとおり）
```

**②からの補足**: ②「9. Repository設計」の`UnitRepository`が扱う`Unit`・`Course`のGORMモデル定義自体は、コース参照機能側の実装仕様書（`コース参照機能_Go実装仕様書.md`、本タスクでは未参照）で定義される想定であり、本書では新規のモデル定義ファイルを作成しない。本機能固有に必要となるのは、タスクとの紐付きを条件とした検索クエリ（`task_unit_query.go`）と、参照専用のHandler/Response DTOのみである。この判断は②に明記がなく、Context構成上の妥当な推測である。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層を設けない。規約「アーキテクチャ規約.md」4章）。

②「6. Entity設計」「7. Value Object設計」「8. Domain Service」「14. Error設計」で示された設計意図（Unit・Course・Taskの扱い、業務ルールの所在、エラー種別）は、本書「4. Application層設計」「11. Error実装方針」に読み替えて反映する。

---

# 4. Application層設計

②「10. UseCase設計」の`ShowTaskUnit`は、Transaction Script採用のためstruct化せず、`application/`直下に置く関数として実装する（規約「アーキテクチャ規約.md」4章）。

## DTO（Command / Query）

Transaction Script採用のため、入力は関数の引数として直接受け取り、専用のCommand/Query構造体は設けない（引数は3つ程度でありコーディング規約8章「関数の引数」の基準上、構造体化は不要と判断する。**②からの補足・推測**）。出力のみ、Handlerへ返す戻り値の型として以下を定義する。

|struct名|フィールドと型|Command/Query区分|
|-|-|-|
|`TaskUnitResult`|`ID uint`, `CourseID uint`, `UnitName string`, `Course CourseSummary`|Query（`ShowTaskUnit`関数の戻り値）|
|`CourseSummary`|`ID uint`, `LevelNumber int`, `LevelName string`|Query（`TaskUnitResult`のネスト値）|

`TaskUnitResult`のフィールド構成は②「16. API互換方針」のResponse項目（id, course_id, unit_name, course{id, level_number, level_name}）に対応する。

## UseCase（application関数）

### `ShowTaskUnit`

- 配置: `internal/curriculum/application/show_task_unit.go`
- 関数シグネチャ:
  ```go
  func ShowTaskUnit(ctx context.Context, db *gorm.DB, userID uint, taskID uint, unitID uint) (*TaskUnitResult, error)
  ```
- 処理ステップ（②「10. UseCase設計」の呼び出し順序をそのまま関数内の手順として表現する。ロジック本体は記述しない）:
  1. `infrastructure`層のタスク所有権確認関数を呼び出し、`taskID`が`userID`の所有するタスクであることを確認する
  2. 所有確認が取れない場合は、所有権エラー（本書「11. Error実装方針」参照）を返す
  3. `infrastructure`層の単元取得関数を呼び出し、`taskID`に紐づく`unitID`の単元をコース情報込みで取得する
  4. 紐付きが確認できない場合は、紐付きエラー（本書「11. Error実装方針」参照）を返す
  5. 取得結果を`TaskUnitResult`に変換して返す
- トランザクション境界: なし（読み取りのみ。②「11. Transaction設計」）
- 発生しうるApplication Error: タスク未所有／未存在エラー、単元未紐付きエラー、Infrastructure起因のエラー（本書「11. Error実装方針」参照）

**②からの補足**: ②「9. Repository設計」で`TaskRepository`は「task-management由来」と明記されているため、所有権確認は`internal/curriculum/infrastructure`内で直接`tasks`テーブルへGORMアクセスするのではなく、task-management Contextが公開する参照用関数を呼び出す形とする（規約「アーキテクチャ規約.md」6章「他Contextのデータを利用する場合は、相手Contextが公開する参照手段（Transaction Script採用時は参照用の関数）を呼び出す」）。task-management Context側の具体的な公開関数名・シグネチャは、task-management Contextの②③文書に依存するため、本書「5. Infrastructure層設計」では想定インターフェースのみを示す（**推測**）。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

配置: `internal/curriculum/infrastructure/task_unit_query.go`

### タスク所有権確認関数（task-managementへの参照呼び出し）

- 関数名・引数・戻り値:
  ```go
  func confirmTaskOwnership(ctx context.Context, db *gorm.DB, taskID uint, userID uint) (bool, error)
  ```
- 内容: task-management Contextが公開する参照用関数（例: `taskmanagement.FindOwnedTask` 相当。実際のpackage・関数名はtask-management Context側の実装仕様書に従う。**②からの補足・推測**）を呼び出し、指定`taskID`が指定`userID`の所有するタスクとして存在するかを確認する。curriculum Context側では`tasks`テーブルへの直接クエリは行わない。
- ②「9. Repository設計」`TaskRepository`の「task_id + user_idによる所有権確認検索」に対応する。

### 単元取得関数（紐付き確認込み）

- 関数名・引数・戻り値:
  ```go
  func findUnitLinkedToTask(ctx context.Context, db *gorm.DB, taskID uint, unitID uint) (*application.TaskUnitResult, error)
  ```
- 発行するクエリ内容: `units`テーブルを主対象に、`task_id`と`unit_id`の組み合わせでタスクへの紐付きを確認する条件を付与し、併せて`course_id`をキーに`courses`テーブルから`level_number`・`level_name`を取得する（結合の方針は本書「12. GORM / DBクエリ設計」参照）。該当レコードが存在しない場合は「対象なし」を表すエラー（`gorm.ErrRecordNotFound`相当）を返す。ソート・ページネーションは対象外（単一レコード取得のため）。
- ②「9. Repository設計」`UnitRepository`の「task_id + unit_idの組み合わせによる紐付き確認検索、コース情報の同時取得」に対応する。

**②からの補足**: ②「3. Bounded Context」の依存関係節では「指定タスクが現在の生徒に属するか、また指定単元がそのタスクに紐づいているかの確認」の両方をtask-managementへの依存として記載している一方、②「9. Repository設計」では紐付き確認を`UnitRepository`（curriculum Context側）の検索機能として記載しており、表現に差がある。本書では、②「9. Repository設計」の記載を優先し、紐付き確認（`task_id`と`unit_id`の一致確認）は`units`テーブル（および紐付けを保持するテーブル）への検索条件としてcurriculum Context内で扱い、タスクの所有権確認のみをtask-managementへの参照呼び出しとする。この整理は②に明示されておらず、実装のために補った判断である（**②からの補足**）。

## 外部連携実装

対象外。Mail・Cache・Queue等の外部連携は②に記載がなく、本機能では不要である。

---

# 6. Presentation層設計

## Handler

### `TaskUnitHandler`

- struct名: `TaskUnitHandler`
- 対応する呼び出し先: `application.ShowTaskUnit`関数（Transaction Script採用のためUseCase層を経由しない）
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`Show`|GET|`/api/v1/student/tasks/:task_id/units/:id`|

- 処理順序（規約「アーキテクチャ規約.md」4章「Transaction Script採用時は、Handler処理順序の中に本来UseCaseが担う手順を明記する」に従い、Handler内で以下の順序を明示する）:
  1. Middlewareで確定済みのcurrent user（`user_id`）をcontextから取得する
  2. パスパラメータ`task_id`・`id`を`ShowTaskUnitRequest`にバインドし、型・必須チェックを行う（本書「9. Validation実装方針」参照）
  3. バインド・検証に失敗した場合は400を返す
  4. `application.ShowTaskUnit(ctx, db, userID, taskID, unitID)`を呼び出す
  5. 戻り値のエラーが所有権エラー・紐付きエラーのいずれかであれば404を返す（本書「11. Error実装方針」参照）
  6. 戻り値のエラーがInfrastructure起因であれば500を返す
  7. 成功した場合、`TaskUnitResult`を`TaskUnitResponse`に変換し、200で返す

## Request / Response DTO

### Request

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`ShowTaskUnitRequest`|`TaskID uint`（`uri:"task_id" binding:"required"`）, `ID uint`（`uri:"id" binding:"required"`）|パスパラメータのため`ShouldBindUri`等で整数かつ必須であることを検証する|

**②からの補足**: リクエストDTOをパスパラメータ用のuriバインディング構造体として定義する点は、②「12. Validation設計」の「task_id・idが整数であることを検証する」「両方が必須項目である」という要件を満たすための実装手段であり、②には具体的な実装方法の指定がないため実装時の判断として補足する。

### Response

|struct名|フィールドと型|
|-|-|
|`TaskUnitResponse`|`ID uint json:"id"`, `CourseID uint json:"course_id"`, `UnitName string json:"unit_name"`, `Course CourseResponse json:"course"`|
|`CourseResponse`|`ID uint json:"id"`, `LevelNumber int json:"level_number"`, `LevelName string json:"level_name"`|

フィールド構成は②「16. API互換方針」のResponse項目（id, course_id, unit_name, course{id, level_number, level_name}）と一致させ、既存レスポンス構造を変更しない。

## Routing

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/student/tasks/:task_id/units/:id`|`TaskUnitHandler.Show`|

認証・student role確認のMiddlewareは、既存の共通Middleware（`shared/`または既存の認証基盤）を利用する想定とし、本機能で新規に作成しない（②「13. Authorization設計」Middleware節）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|`/api/v1/student/tasks/:task_id/units/:id`|`TaskUnitHandler.Show`|`ShowTaskUnitRequest`（task_id, id）|`TaskUnitResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|`task_id`または`id`が整数でない／未指定|400|リクエストパラメータ不正|
|`task_id`が現在の生徒の所有するタスクとして存在しない|404|対象データが見つかりません（既存エラーレスポンス形式を踏襲。②「16. API互換方針」）|
|`id`（単元）が指定タスクに紐づいていない|404|対象データが見つかりません（同上）|
|DB接続失敗等のInfrastructure障害|500|内部エラー|

---

# 8. Transaction実装方針

## Transaction開始箇所

なし。状態を変更する処理が存在しないため、`application.ShowTaskUnit`関数内でトランザクションを開始しない（②「11. Transaction設計」）。

## Transaction終了箇所（Commit / Rollback条件）

該当なし。

## 複数関数にまたがる場合の扱い

`ShowTaskUnit`関数はタスク所有権確認関数・単元取得関数の2つを順に呼び出すが、いずれも読み取りのみであり、整合性保証のためのトランザクションは不要である（②「11. Transaction設計」の理由をそのまま踏襲）。

---

# 9. Validation実装方針

## Presentation

- `ShowTaskUnitRequest`の`TaskID`・`ID`について、`uri`バインディングにより整数型であることをGinのパースエラーとして検知する
- `binding:"required"`により、両パラメータが未指定の場合はバインドエラーとして400を返す
- フォーマットチェック: 対象外（②「12. Validation設計」Presentation節のとおり）

## 業務ルール検証

Transaction Script採用時: `application.ShowTaskUnit`関数内のガード節で以下を検証する（②「12. Validation設計」Domain節を読み替え）。

- 指定`taskID`が`userID`の所有であるか（タスク所有権確認関数の結果で判定）
- 指定`unitID`が指定`taskID`に紐づいているか（単元取得関数の結果で判定）

いずれも条件不成立の場合は関数内で早期returnし、対応するエラー（本書「11. Error実装方針」参照）を返す（コーディング規約6章「エラー処理を先に行う」に従う）。

---

# 10. Authorization実装方針

②「13. Authorization設計」を実装レベルに落とし込む。

## Middleware

- 認証済みユーザーを特定し、`user_id`をcontextへ保持する
- 役割が`student`であることを確認する

## Handler

- 業務権限判定は持たせない（contextから`user_id`を取得し、application関数に渡すのみ）

## Application関数（Transaction Script採用時）

- `userID`を用いて、指定`taskID`が自分の所有であることをタスク所有権確認関数で確認する
- 所有確認後、そのタスクに紐づく単元のみを単元取得関数で取得する

## 判断理由

②「13. Authorization設計」の判断理由（Railsの関連スコープに埋め込まれた暗黙的アクセス制御を避け、所有権確認・紐付き確認をapplication関数内の明示的な業務手順として表現する）をそのまま踏襲する。

---

# 11. Error実装方針

Transaction Script採用のためDomain Errorに相当する層は存在しない（規約「アーキテクチャ規約.md」8章）。`infrastructure`関数・`application`関数で発生したエラーをApplication Error相当として扱う。

## Infrastructure Error → Application Error相当への変換方針

- タスク所有権確認関数が「所有タスクとして存在しない」ことを返した場合、`application`層で`ErrTaskNotOwned`（sentinel error、`internal/curriculum/application/show_task_unit.go`に定義）に変換する
- 単元取得関数が`gorm.ErrRecordNotFound`相当を返した場合、`application`層で`ErrUnitNotLinked`（同上）に変換する
- 上記以外のDB接続失敗等は、`fmt.Errorf`による`%w`ラップでそのまま上位に伝播させる（コーディング規約6章「エラーのラップ」）

## Application Error相当 → HTTPレスポンスへの変換方針

Handlerで`errors.Is`により判定し、Status Codeへ変換する。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrTaskNotOwned`（タスクが存在しない、または自分の所有でない）|application|404|
|`ErrUnitNotLinked`（単元が指定タスクに紐づかない）|application|404|
|上記以外（DB接続失敗等）|infrastructure|500|

②「14. Error設計」の判断理由（「タスクが存在しない・自分のものでない」「単元がタスクに紐づかない」を404としてまとめて扱う現行仕様の挙動を維持しつつ、原因ごとにエラー種類を分離し内部的な原因追跡をしやすくする）をそのまま踏襲する。

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

|モデル|テーブル|本機能での役割|
|-|-|-|
|`UnitModel`（コース参照機能側で定義済み想定）|`units`|単元情報の取得対象|
|`CourseModel`（コース参照機能側で定義済み想定）|`courses`|単元が属するコース情報の取得対象|
|タスクと単元の紐付けを表すモデル（名称未確定）|`task_units`|`task_id`・`unit_id`の紐付き確認に使用|
|task-management Context側のタスクモデル|`tasks`|所有権確認のためtask-management Contextの参照関数経由でのみアクセスし、curriculum Context側では直接モデル定義しない|

**②からの補足**: `task_units`テーブルの正確なカラム構成（外部キー・複合ユニーク制約の有無等）は①未提供のため参照不可であり、②本文にも詳細な記載がない。実装時は`task_units`テーブルの既存スキーマ（マイグレーション定義）を別途確認する必要がある。本書では②「17. DB設計方針」の「現行の`units`・`courses`・`tasks`・`task_units`テーブルで本機能の要件を満たしている」という記載に基づき、テーブル名のみを前提として設計している。

## 主要クエリの条件・ソート・ページネーション方針

- 単元取得クエリ: `units`テーブルを主対象に、`task_units`テーブルとの結合により`task_id = :taskID AND unit_id = :unitID`の条件で1件検索する。併せて`units.course_id`をキーに`courses`テーブルを結合し、`level_number`・`level_name`を同時取得する
- ソート: 不要（単一レコード取得のため）
- ページネーション: 不要（単一レコード取得のため）
- タスク所有権確認クエリ: curriculum Context内では発行しない（task-management Context側の参照関数に委譲。本書「5. Infrastructure層設計」参照）

## 既存Schemaに対する変更

②「17. DB設計方針」のとおり変更なし。本書でもSchema変更は提案しない。

SQL文そのものは本書に記載しない。

---

# 13. テストケース設計

Transaction Script採用のため、②「18. テスト戦略」の区分を以下のように読み替える（規約「Go実装仕様書_自動生成プロンプト.md」13章の読み替え規則に従う）。

- 「Domain Test」「Repository Test」は対象外
- 「UseCase Test」→「Application関数 Test」

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## UseCase Test（Application関数 Test）

|対象|テストケース|
|-|-|
|`ShowTaskUnit`|自分が所有するタスクに紐づく単元を指定した場合、`TaskUnitResult`が正しく返却される|
|`ShowTaskUnit`|所有しないタスクIDを指定した場合、`ErrTaskNotOwned`が返る|
|`ShowTaskUnit`|存在しないタスクIDを指定した場合、`ErrTaskNotOwned`が返る|
|`ShowTaskUnit`|指定タスクに紐づかない単元IDを指定した場合、`ErrUnitNotLinked`が返る|
|`ShowTaskUnit`|存在しない単元IDを指定した場合、`ErrUnitNotLinked`が返る|

## Repository Test

対象外（Transaction Script採用のため、Repository Interfaceを設けない。infrastructure関数のテストは下記「Handler Test」相当ではなくInfrastructure関数単体テストとして別途用意する）。

|対象|テストケース|
|-|-|
|`findUnitLinkedToTask`|`task_id`・`unit_id`が一致するレコードが存在する場合、コース情報を含めて正しく取得できる|
|`findUnitLinkedToTask`|`task_id`・`unit_id`の組み合わせが存在しない場合、レコード未検出のエラーが返る|
|`confirmTaskOwnership`|task-management Contextの参照関数呼び出し結果を正しく`bool`/`error`へ変換する（**②からの補足**: task-management側の参照関数実装が未確定のため、モック等での検証を想定）|

## Handler Test

|対象|テストケース|
|-|-|
|`TaskUnitHandler.Show`|`task_id`・`id`が整数として正しく指定された場合、200と`TaskUnitResponse`が返る|
|`TaskUnitHandler.Show`|`task_id`または`id`が整数でない場合、400が返る|
|`TaskUnitHandler.Show`|`task_id`または`id`が未指定の場合、400が返る|
|`TaskUnitHandler.Show`|`application.ShowTaskUnit`が`ErrTaskNotOwned`／`ErrUnitNotLinked`を返した場合、404が返る|
|`TaskUnitHandler.Show`|`application.ShowTaskUnit`がその他のエラーを返した場合、500が返る|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/student/tasks/:task_id/units/:id`|自分の所有するタスクに紐づく単元を指定した場合、単元詳細（コース情報含む）が200で取得できる|
|`GET /api/v1/student/tasks/:task_id/units/:id`|他ユーザーが所有するタスクIDを指定した場合、404が返る|
|`GET /api/v1/student/tasks/:task_id/units/:id`|指定タスクに紐づかない単元IDを指定した場合、404が返る|
|`GET /api/v1/student/tasks/:task_id/units/:id`|未認証状態でアクセスした場合、Middlewareにより認証エラーとなる|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測か否か|
|-|-|-|
|`UnitModel`・`CourseModel`は本書で新規定義せず、コース参照機能側の実装仕様書で定義済みのものを利用する|②「3. Bounded Context」でコース参照機能と同一Context（`curriculum`）として扱うと明記されているため、モデルの重複定義を避ける判断|推測（コース参照機能側の実装仕様書の内容は本タスクで未参照のため）|
|タスク所有権確認は、curriculum Context内で`tasks`テーブルへ直接GORMアクセスするのではなく、task-management Contextが公開する参照関数を呼び出す形とする|②「9. Repository設計」で`TaskRepository`が「task-management由来」と明記されており、規約「アーキテクチャ規約.md」6章の他Context連携ルール（相手Contextの参照手段を経由する）に従うため|推測（task-management側の具体的な公開関数名・シグネチャは未確定）|
|単元とタスクの紐付き確認（`task_units`テーブルの参照）は、task-managementへの参照呼び出しではなく、curriculum Context内の`findUnitLinkedToTask`関数内のクエリ条件として扱う|②「3. Bounded Context」の依存関係節と②「9. Repository設計」の記載に表現差があり、後者（`UnitRepository`の検索機能として明記）を優先した|②からの補足（②内の記載差異を実装のために解消する判断）|
|Request DTO（`ShowTaskUnitRequest`）をパスパラメータ用のuriバインディング構造体として定義する|②「12. Validation設計」のPresentation要件（型・必須チェック）を満たす具体的な実装手段として、Ginの一般的な実装パターンを採用した|推測|
|`task_units`テーブルの正確なカラム構成・制約は本書に記載していない|①未提供のため参照不可であり、②本文にもテーブル名以上の詳細記載がないため|—（不明事項として明記）|
|Error種別を`ErrTaskNotOwned`・`ErrUnitNotLinked`というsentinel errorとして定義する具体的な命名は、②「14. Error設計」の責務説明（Domain Error／Application Error）を、Transaction Script構造におけるGoのエラー変数命名規則（コーディング規約6章）に落とし込んだもの|②にはエラーの具体的な型名・変数名の指定がないため|推測|

上記以外の判断はなく、②の設計方針（Transaction Script採用・Aggregate不要・Value Object不要・Domain Service不要・Domain Event不要・Transaction不要・API互換方針・DB方針）はすべてそのまま踏襲している。
