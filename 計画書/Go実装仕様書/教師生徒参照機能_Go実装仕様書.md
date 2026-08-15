# 教師生徒参照機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

教師が同校の生徒一覧・生徒詳細を参照できる、読み取り専用の機能である。担当学年権限を持つ教師は、担当学年の生徒のみに閲覧範囲が制限される。データの作成・更新は一切行わない（②「1. 機能概要」）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により、本機能は **Transaction Script** を採用する。

- 状態遷移・状態管理が存在しない参照専用機能である
- 業務ルールは「同校であること」「担当学年権限がある場合は担当学年のみ」という2条件の絞り込みに限定される
- 検索条件構築とページングという手続き的処理が中心であり、Entityに振る舞いを持たせる必要性が薄い

Active Record・Domain Model・Event Sourcingは、更新責務がない／状態やライフサイクルを持たない／通知すべき状態変化がないことを理由に②で不採用と判断されている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- `GET /api/v1/teacher/students`（生徒一覧取得）
- `GET /api/v1/teacher/students/:id`（生徒詳細取得）

の2エンドポインの実装に必要な、`application`関数・`infrastructure`関数・Handler・Routing・Request/Response構造体の実装単位を規定する。①Rails実装（Teacher::StudentsQuery・StudentSerializer等の実装詳細）は本書作成時点で未提供のため参照不可であり、該当箇所は②の記載のみを根拠とする。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `student-directory`（②「3. Bounded Context」）

## ②で採用した設計パターン

- Transaction Script（②「4. 設計パターン」）

規約「4. 設計パターンごとの構造適用方針」の Transaction Script構造（domain層・usecase層・Repository Interfaceを設けない）を適用する。

## ディレクトリ一覧

```
internal/student_directory/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    └── response/
```

**②からの補足**: アーキテクチャ規約「9. 命名規約」により、`internal/`配下のディレクトリ名は英単語1語または短いスネークケースとする。②のContext名`student-directory`と`internal/`配下のディレクトリ名の対応関係は②に明記がないため、本書では`internal/student_directory`と判断した（推測）。

## 作成するファイル一覧

```
internal/student_directory/application/list_students.go
internal/student_directory/application/show_student.go
internal/student_directory/infrastructure/student_query.go
internal/student_directory/presentation/handler/student_handler.go
internal/student_directory/presentation/response/student_response.go
internal/student_directory/presentation/routes.go
```

---

# 3. Domain層設計

**対象外（Transaction Script採用のため、Domain層を設けない）。**

②「6. Entity設計」「7. Value Object設計」「9. Repository設計」「14. Error設計」で示された設計意図は、Domain層のstruct/interfaceとしては実装せず、以下のとおり読み替えて反映する。

- Student Entity（②6章）: 独立したEntity structは作らない。生徒情報は「4. Application層設計」のDTOと「5. Infrastructure層設計」の検索結果として表現する
- GradeScope Value Object（②7章）: 独立したValue Object型は作らない。「担当学年権限の有無」「対象学年ID」という絞り込み条件は、`application/`直下の関数の引数・ローカル変数として表現する（詳細は4章参照）
- StudentRepository（②9章）: interfaceとしては定義しない。責務（同校生徒一覧取得・指定ID生徒取得・high_school_id/grade_id絞り込み・name_kanaソート・ページネーション）は「5. Infrastructure層設計」のinfrastructure関数として実装する
- Domain Error（②14章）: 独立したDomain Error型は設けない。「11. Error実装方針」に示すとおり、application関数が返すエラーをApplication Error相当として扱う（規約「8. 横断的関心事の置き場所」）

---

# 4. Application層設計

**実装上の位置づけ**: Transaction Script採用のため、usecase層（struct）は設けない。`application/`直下に、1業務操作＝1関数として実装する。

## DTO（Command / Query）

いずれもQueryに属する（本機能はデータ変更を行わないため）。

| struct名 | フィールド | 型 | 備考 |
|-|-|-|-|
| `RequestingTeacher` | `TeacherID` | `uint` | current teacherの識別子 |
| | `HighSchoolID` | `uint` | 所属校ID（②13章：同校生徒のみ参照可能） |
| | `HasGradeAuthority` | `bool` | 担当学年権限の有無（②7章 GradeScope相当） |
| | `AuthorizedGradeID` | `*uint` | 担当学年ID。`HasGradeAuthority`が`false`の場合は未使用 |
| `ListStudentsInput` | `Teacher` | `RequestingTeacher` | 呼び出し元教師情報 |
| | `Page` | `int` | ページ番号 |
| `ListStudentsOutput` | `Students` | `[]StudentSummary` | 一覧結果 |
| | `CurrentPage` | `int` | 現在ページ |
| | `TotalPages` | `int` | 総ページ数 |
| | `TotalCount` | `int` | 総件数 |
| | `PerPage` | `int` | 1ページあたり件数 |
| `StudentSummary` | `ID` | `uint` | 生徒ID |
| | `Name` | `string` | 氏名 |
| | `NameKana` | `string` | かな氏名 |
| | `GradeID` | `uint` | 学年ID |
| | `HighSchoolID` | `uint` | 所属校ID |
| `ShowStudentInput` | `Teacher` | `RequestingTeacher` | 呼び出し元教師情報 |
| | `StudentID` | `uint` | 対象生徒ID |
| `ShowStudentOutput` | `Student` | `StudentSummary`相当＋関連情報 | 生徒詳細情報。詳細フィールドは①未提供のため参照不可（14章参照） |

**②からの補足**: `RequestingTeacher`のフィールド構成は②「13. Authorization設計」の記述（「current_userの所属校IDによる絞り込みを適用する」「担当学年権限がある場合はgrade_idによる絞り込みを適用する」）から導出したものであり、具体的な型・フィールド名は②に明記がないため実装仕様書側の判断である（推測）。current teacher情報自体の取得元（認証Middleware・shared/authパッケージ等）は本機能のスコープ外とする。

## application関数

### ListStudents

- 関数シグネチャ: `func ListStudents(ctx context.Context, input ListStudentsInput) (ListStudentsOutput, error)`
- 処理ステップ（呼び出し順序）:
  1. `input.Teacher.HighSchoolID`を絞り込み条件（必須）とする
  2. `input.Teacher.HasGradeAuthority`が`true`の場合、`input.Teacher.AuthorizedGradeID`を絞り込み条件に追加する（②7章 GradeScopeの独自ルール：権限がない場合は学年による絞り込みを行わない）
  3. 組み立てた絞り込み条件・`input.Page`・ソート条件（name_kana順）を渡して、infrastructure層の生徒一覧検索関数を呼び出す
  4. 検索結果の総件数からページ情報（`TotalPages`・`TotalCount`・`PerPage`）を組み立てる
  5. `ListStudentsOutput`を構築して返す
- トランザクション境界: なし（読み取りのみ、②11章のとおり）
- 発生しうるApplication Error: なし（該当条件に一致する生徒が0件の場合も正常系として空一覧を返す）。infrastructure層からのエラーはそのまま上位へ伝播させる（②に個別のApplication Error定義がないため、11章の方針に従う）

### ShowStudent

- 関数シグネチャ: `func ShowStudent(ctx context.Context, input ShowStudentInput) (ShowStudentOutput, error)`
- 処理ステップ（呼び出し順序）:
  1. `input.Teacher.HighSchoolID`を絞り込み条件（必須）とする
  2. `input.Teacher.HasGradeAuthority`が`true`の場合、`input.Teacher.AuthorizedGradeID`を絞り込み条件に追加する
  3. `input.StudentID`と絞り込み条件を渡して、infrastructure層の生徒詳細取得関数を呼び出す
  4. 対象が見つからない場合（権限範囲外である場合を含む）、`ErrStudentNotFound`を返す（②14章：権限範囲外アクセスと未存在を区別しない）
  5. 見つかった場合、`ShowStudentOutput`へ変換して返す
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrStudentNotFound`（対象生徒が存在しない、または権限範囲外である場合）

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

`internal/student_directory/infrastructure/student_query.go`に配置する。

### FindStudents

- 関数シグネチャ: `func FindStudents(ctx context.Context, condition StudentSearchCondition, page int) (records []StudentRecord, totalCount int, err error)`
- 発行するクエリ内容:
  - `high_school_id`が`condition.HighSchoolID`と一致する条件
  - `condition.GradeID`が指定されている場合、`grade_id`が一致する条件を追加
  - `name_kana`昇順でソート
  - `page`に基づくページネーション（1ページあたり件数は②に明記がないため14章参照）
  - 絞り込み条件に一致する総件数を別途取得する（ページ情報組み立てのため）

### FindStudentByID

- 関数シグネチャ: `func FindStudentByID(ctx context.Context, studentID uint, condition StudentSearchCondition) (record StudentRecord, found bool, err error)`
- 発行するクエリ内容:
  - `id`が`studentID`と一致する条件
  - `high_school_id`が`condition.HighSchoolID`と一致する条件
  - `condition.GradeID`が指定されている場合、`grade_id`が一致する条件を追加
  - 対象が存在しない場合は`found=false`を返す（権限範囲外で条件に一致しない場合も同様に`found=false`となる）

### StudentSearchCondition（infrastructure内の検索条件型）

| フィールド | 型 | 備考 |
|-|-|-|
| `HighSchoolID` | `uint` | 必須 |
| `GradeID` | `*uint` | 担当学年権限がある場合のみ設定 |

### StudentRecord（クエリ結果を保持する型）

| フィールド | 型 | 備考 |
|-|-|-|
| `ID` | `uint` | |
| `Name` | `string` | |
| `NameKana` | `string` | |
| `GradeID` | `uint` | |
| `HighSchoolID` | `uint` | |

②6章の記載範囲（氏名・かな氏名・学年・所属校）に基づく。詳細レスポンス用の関連情報フィールドは①未提供のため本書では確定しない（14章参照）。

## 外部連携実装

対象外。本機能はMail・Cache・Queue等の外部連携を必要としない（②に記載なし）。

---

# 6. Presentation層設計

## Handler

- struct名: `StudentHandler`
- 対応する呼び出し先: `application`パッケージの`ListStudents`関数・`ShowStudent`関数（Transaction Script採用のためusecase層を経由しない。規約「4. 設計パターンごとの構造適用方針」の「Handlerは`application/`の関数を直接呼び出す」に従う）
- 依存: なし（`application`パッケージの関数を直接呼び出すため、DIするRepository・Store等を持たない）
- メソッド一覧:

| メソッド | HTTPメソッド | パス |
|-|-|-|
| `List(c *gin.Context)` | GET | `/api/v1/teacher/students` |
| `Show(c *gin.Context)` | GET | `/api/v1/teacher/students/:id` |

- 処理順序（`List`）:
  1. Ginコンテキストから認証Middlewareが格納したcurrent teacher情報を取得し、`RequestingTeacher`へ変換する（Handlerでは権限判定を行わず、値の受け渡しのみ行う。②13章「Handler」：具体的な権限判定は持たせない）
  2. クエリパラメータ`page`をバインドする（型が整数でない場合の扱いは14章参照）
  3. `application.ListStudentsInput`を組み立て、`application.ListStudents`を呼び出す
  4. 取得結果を`StudentListResponse`へ変換する
  5. `200`でレスポンスを返す
- 処理順序（`Show`）:
  1. current teacher情報を取得し、`RequestingTeacher`へ変換する
  2. パスパラメータ`id`をバインドする（整数変換に失敗した場合の扱いは14章参照）
  3. `application.ShowStudentInput`を組み立て、`application.ShowStudent`を呼び出す
  4. `ErrStudentNotFound`が返却された場合は`404`を返す
  5. 成功時は`StudentDetailResponse`へ変換し、`200`で返す

## Request / Response DTO

**②からの補足**: 規約「4. 設計パターンごとの構造適用方針」のTransaction Script構造には`request/`ディレクトリが定義されていない。本書では正式なDTO packageを設けず、Handler内でクエリ・パスパラメータを直接バインドする方針とする（推測、実装判断）。

- `page`（クエリパラメータ）: `int`。バインド方式・必須有無は14章参照
- `id`（パスパラメータ）: `uint`。整数への変換が必要（②12章「必須チェック: 詳細取得時のid（route由来）を検証する」）

Response DTO（`internal/student_directory/presentation/response/student_response.go`）:

| struct名 | フィールド | 型 |
|-|-|-|
| `StudentListResponse` | `Students` | `[]StudentSummaryResponse` |
| | `Meta` | `MetaResponse` |
| `StudentSummaryResponse` | `ID` | `uint` |
| | `Name` | `string` |
| | `NameKana` | `string` |
| | `GradeID` | `uint` |
| | `HighSchoolID` | `uint` |
| `StudentDetailResponse` | `ID` | `uint` |
| | `Name` | `string` |
| | `NameKana` | `string` |
| | `GradeID` | `uint` |
| | `HighSchoolID` | `uint` |
| | （関連情報） | — | ①未提供のため参照不可。②16章「生徒の基本情報と関連情報を維持する」に留める（14章参照） |
| `MetaResponse` | `CurrentPage` | `int` |
| | `TotalPages` | `int` |
| | `TotalCount` | `int` |
| | `PerPage` | `int` |

バリデーションタグ／チェック内容:

- `page`: Handler内で整数変換を行う。変換失敗時の挙動は②に明記がないため14章参照
- `id`: Handler内で整数変換を行う。変換失敗時は`400`として扱う（推測。②に明記なし、詳細は14章）

## Routing

`internal/student_directory/presentation/routes.go`

| Method | Path | Handler |
|-|-|-|
| GET | `/api/v1/teacher/students` | `StudentHandler.List` |
| GET | `/api/v1/teacher/students/:id` | `StudentHandler.Show` |

いずれのルートも、認証Middleware・teacherロール確認Middlewareを経由する（②13章「Middleware」）。当該Middlewareは本機能内で新規実装せず、既存の共通Middlewareを利用する想定とする（①未提供のため実装詳細不明）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/teacher/students|StudentHandler.List|page（query, optional）|StudentListResponse|200|
|GET|/api/v1/teacher/students/:id|StudentHandler.Show|id（path, required）|StudentDetailResponse|200|

Errorケース:

|条件|Status Code|Error内容|
|-|-|-|
|対象生徒が存在しない|404|既存のerrors形式を踏襲（②16章）|
|対象生徒が権限範囲外（他校／担当学年外）|404|未存在の場合と同一のレスポンスとして扱う（②14章：権限範囲外アクセスと未存在を区別しない設計判断） |
|pageパラメータが不正な形式|400（推測、14章参照）|バリデーションエラー|
|idパラメータが不正な形式|400（推測、14章参照）|バリデーションエラー|
|未認証|401（推測、14章参照）|認証エラー（②13章Middlewareの記載に基づくが、Status Codeは②16章に明記がない）|
|teacherロールでない|403（推測、14章参照）|権限エラー（同上）|

---

# 8. Transaction実装方針

- Transaction開始箇所: なし（②11章「Transaction開始位置: 使用しない」）
- Transaction終了箇所: 該当なし
- 複数関数（infrastructure関数）にまたがる場合の扱い: `ListStudents`は一覧取得クエリと総件数取得クエリの2回のDBアクセスを行うが、いずれも読み取りのみであり、整合性維持のためのTransaction境界は不要である（②11章「理由」：データ変更を一切伴わない読み取り専用機能のため）

---

# 9. Validation実装方針

## Presentation

- `page`: 整数であることを検証する（②12章「型チェック」）
- `id`: 詳細取得時、route由来の値が整数として解釈できることを検証する（②12章「必須チェック」）
- フォーマットチェック: 特になし（②12章のとおり）

## 業務ルール検証

Transaction Script採用のため、application関数内のガード節で以下を検証する。

- `ListStudents`: `RequestingTeacher.HasGradeAuthority`の値に応じて、`GradeID`絞り込み条件を追加するかどうかを判定する（②7章 GradeScope相当のルール。関数内条件分岐として実装し、独立したValue Object型は作らない）
- `ShowStudent`: 絞り込み条件（所属校・担当学年）に一致しない場合は`ErrStudentNotFound`として扱う（②14章）

②12章「Domain」の記載どおり、入力値そのものに対する業務ルール違反（バリデーションエラー以外の業務ルール）は本機能には存在しない。

---

# 10. Authorization実装方針

## Middleware

- 認証済みユーザーを特定し、teacherロールであることを確認する（②13章）

## Handler

- current teacher情報をcontextから取得し、`RequestingTeacher`へ変換してapplication関数へ渡す
- 具体的な権限判定は持たせない（②13章「Handler」）

## application関数（Transaction Script採用時）

- `current_user`の所属校IDによる絞り込みを適用する
- 担当学年権限がある場合はgrade_idによる絞り込みを適用する
- 対象生徒が権限範囲外である場合は、存在しない場合と同様に扱う（②13章「UseCase」の記載をapplication関数に読み替える）

②13章「判断理由」のとおり、権限ロジック（同校・担当学年）は単純な条件分岐であり、複雑なドメインルールではないためapplication関数に一元化し、Domain層への分散は行わない。

---

# 11. Error実装方針

- Domain Error → Application Error変換方針: 対象外（Domain層を設けないため）。規約「8. 横断的関心事の置き場所」に基づき、Transaction Script採用時はapplication関数が返すエラーをApplication Error相当として扱う
- Application Error → HTTPレスポンス変換方針: `ErrStudentNotFound`を`404`へ変換する（②14章・②16章）
- Infrastructure Errorのハンドリング方針: DB接続失敗等のInfrastructure Errorは、Handlerでラップし`500`として扱う（②に個別の記載はなく、一般的なエラーハンドリング方針として補った。14章参照）

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrStudentNotFound`（未存在・権限範囲外）|Application|404|
|pageパラメータ不正|Presentation|400（推測）|
|idパラメータ不正|Presentation|400（推測）|
|未認証|Middleware|401（推測）|
|teacherロール不一致|Middleware|403（推測）|
|DB接続障害等|Infrastructure|500（推測）|

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

②17章のとおり、既存Rails DBを継続利用し、Schema変更は行わない。

| 参照対象 | テーブル | 備考 |
|-|-|-|
| 生徒 | `users` | 生徒データの実体はUser Contextが管理する（②3章）。GORMモデル定義自体の所在は14章参照 |
| 学年 | `grades` | 絞り込み条件・レスポンスの学年情報として参照 |
| 所属校 | `high_schools` | 絞り込み条件・レスポンスの所属校情報として参照 |

**②からの補足**: 規約「5. Bounded Context構成」の機能Context一覧では、User Context自体の②文書はまだ存在しないとされている。そのため、生徒データに対応するGORMモデル（`users`テーブル）を本Contextが独自定義するか、User Context側の既存定義を参照するかは②に明記がなく、本書時点では確定できない（推測、14章参照）。

## 主要クエリの条件・ソート・ページネーション方針

- `FindStudents`: `high_school_id`一致 →（担当学年権限がある場合）`grade_id`一致 → `name_kana`昇順ソート → ページネーション
- `FindStudentByID`: `id`一致 AND `high_school_id`一致 →（担当学年権限がある場合）`grade_id`一致

SQL文そのものは記載しない。

## Schema変更の反映方針

②17章のとおりSchema変更は提案されていないため、本書でも反映しない。

---

# 13. テストケース設計

Transaction Script読み替え: 「Domain Test」は対象外、「UseCase Test」→「Application関数 Test」とする（規約・自動生成プロンプトの読み替え指示に従う）。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test（旧: UseCase Test）

|対象|テストケース|
|-|-|
|`ListStudents`|担当学年権限を持たない教師が呼び出した場合、同校の全生徒が取得できること|
|`ListStudents`|担当学年権限を持つ教師が呼び出した場合、担当学年の生徒のみ取得できること|
|`ListStudents`|`page`指定に応じてページングされた結果が返ること|
|`ListStudents`|該当生徒が0件の場合、空の一覧とページ情報が返ること|
|`ShowStudent`|同校かつ権限範囲内の生徒IDを指定した場合、詳細情報が取得できること|
|`ShowStudent`|他校の生徒IDを指定した場合、`ErrStudentNotFound`が返ること|
|`ShowStudent`|担当学年権限を持つ教師が担当学年外の生徒IDを指定した場合、`ErrStudentNotFound`が返ること|
|`ShowStudent`|存在しない生徒IDを指定した場合、`ErrStudentNotFound`が返ること|

## Repository Test

対象外（Transaction Script採用のため、Repository層を設けない）。②18章「Repository Test」が目的とした「検索条件・ページングの正確性の検証」は、`infrastructure`パッケージの`FindStudents`・`FindStudentByID`に対する個別テスト（例: `student_query_test.go`）で担う。

|対象|テストケース|
|-|-|
|`FindStudents`|`high_school_id`・`grade_id`の絞り込み条件が正しく適用されること|
|`FindStudents`|`name_kana`昇順でソートされること|
|`FindStudents`|ページネーションの境界値（最終ページ・空ページ）が正しく扱われること|
|`FindStudentByID`|絞り込み条件に一致する場合、`found=true`で結果が返ること|
|`FindStudentByID`|絞り込み条件に一致しない場合（他校・他学年・不存在）、`found=false`が返ること|

## Handler Test

|対象|テストケース|
|-|-|
|`StudentHandler.List`|`page`が正しい整数の場合、200と`StudentListResponse`が返ること|
|`StudentHandler.List`|`page`が不正な形式の場合の挙動（14章の判断に基づく想定挙動）が返ること|
|`StudentHandler.Show`|`id`が正しい整数かつ対象が存在する場合、200と`StudentDetailResponse`が返ること|
|`StudentHandler.Show`|`id`が不正な形式の場合の挙動（14章の判断に基づく想定挙動）が返ること|
|`StudentHandler.Show`|対象生徒が存在しない、または権限範囲外の場合、404が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/teacher/students`|担当学年権限の有無に応じて、エンドポイント経由で正しい生徒一覧が取得できること|
|`GET /api/v1/teacher/students/:id`|エンドポイント経由で、権限範囲内の生徒詳細が取得できること|
|`GET /api/v1/teacher/students/:id`|権限範囲外の生徒IDに対して404が返ること|
|全体|未認証・非teacherロールでのアクセスがMiddlewareで拒否されること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

| No. | 判断した内容 | 判断理由 | 推測かどうか |
|-|-|-|-|
| 1 | `internal/`配下のディレクトリ名を`internal/student_directory`とした | ②のContext名`student-directory`とディレクトリ名の対応関係が②に明記がないため、規約「9. 命名規約」の変換ルールに基づき判断した | 推測 |
| 2 | ②7章のGradeScope（Value Object）を独立した型として実装せず、application関数内の引数・ローカル変数として表現することとした | 規約「4. 設計パターンごとの構造適用方針」のTransaction Script構造では、Value Object等の型を作らず関数内のガード節で検証を行う方針が定められているため。②の設計判断（絞り込み条件という概念）自体は変更していない | 実装構造上の判断（規約に基づく） |
| 3 | current teacher情報を表す`RequestingTeacher`の具体的なフィールド構成（`TeacherID`・`HighSchoolID`・`HasGradeAuthority`・`AuthorizedGradeID`） | ②13章の記述から導出したが、具体的な型・フィールド名の明記は②にない | 推測 |
| 4 | `page`パラメータが不正な形式の場合の挙動（400を返す、またはデフォルト値を適用する等） | ②12章では「型チェック」を行うことのみ記載され、具体的な失敗時挙動の記載がない | 推測 |
| 5 | `id`パラメータが不正な形式の場合、400として扱う方針 | ②12章では「必須チェック」を行うことのみ記載され、具体的な失敗時挙動の記載がない | 推測 |
| 6 | 1ページあたりの件数（`PerPage`）の具体的な値 | ②16章ではレスポンスに`per_page`を含むことのみ記載され、具体的な値の記載がない | 推測 |
| 7 | 未認証時401・ロール不一致時403というHTTP Statusの割り当て | ②16章のStatus Code一覧は200・404のみで、認証・認可失敗時のStatus Codeの明記がない。②13章のMiddleware記載から一般的な対応として補った | 推測 |
| 8 | DB接続障害等のInfrastructure Errorを500として扱う方針 | ②にInfrastructure Errorに対応するHTTP Statusの記載がなく、一般的なエラーハンドリング方針として補った | 推測 |
| 9 | 生徒詳細レスポンス（`StudentDetailResponse`）の関連情報フィールドの具体的な内容を確定していない | ②16章は「生徒の基本情報と関連情報を維持する」とのみ記載し、具体的なフィールドはRails実装（①）に依存する。①は本書作成時点で未提供のため参照不可 | ①未提供のため参照不可（推測ではなく未確定として明記） |
| 10 | 生徒データ（`users`テーブル）に対応するGORMモデルを本Contextが独自定義するか、User Context側の既存定義を参照するかを確定していない | 規約「5. Bounded Context構成」により、User Context自体の②文書がまだ存在しないため、参照方法を本書時点で確定できない | 推測 |
| 11 | Request DTOを正式なpackageとして持たず、Handler内でクエリ・パスパラメータを直接バインドする方針とした | 規約「4. 設計パターンごとの構造適用方針」のTransaction Script構造には`request/`ディレクトリが定義されていないため | 実装構造上の判断（規約に基づく） |
