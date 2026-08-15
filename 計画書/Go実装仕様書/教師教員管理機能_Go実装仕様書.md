# 教師教員管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

教師が同校の教員一覧・詳細を確認し、新規教員アカウントを作成（招待）できる機能である。新規教員作成時は、`User`（アカウント本体）・`TeacherPermission`（初期権限）・`TeacherGrade`（担当学年）の3レコードを1つの業務操作として整合性を保ちながら作成する。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」より、**Active Record**を採用する。

- 主要操作が一覧・詳細・作成というCRUD中心の操作であり、複雑な状態遷移が存在しない（教員アカウントの有効/無効は`deleted_at`による論理削除のみで、本機能の対象外）
- 作成時の業務ルール（氏名カナ形式・メール形式・同校学年制約・grade_scope値チェック）は、値の妥当性検証に近く、Entity（Active Record上はModel）の検証責務として表現できる
- アカウント・権限・担当学年の関連付けは、1回の作成操作内でデータ整合性を保てば足り、複雑なドメイン振る舞いを要しない

②「4. 設計パターン」で採用しなかったTransaction Script（検証ルールが複数あり手続きに埋没する懸念）・Domain Model（状態遷移や複数Entity横断の複雑な業務ルールが現時点でない）・Event Sourcing（履歴再構築・監査要件がない）についても、②の判断をそのまま踏襲し、本書では変更しない。

## 本書が対象とする実装範囲

- Bounded Context: `teacher-directory`（教員名簿・着任管理コンテキスト）
- 対象UseCase相当の操作: 教員一覧取得（ListTeachers）、教員詳細取得（ShowTeacher）、新規教員作成（CreateTeacher）
- 規約（`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」）に従い、Active Record採用機能としてdomain層・application層（usecase層）・Repository Interfaceを設けない構造で実装する
- ①Rails実装の詳細（`CreateTeacherForm`等のコード内容）は本タスクでは提供されておらず、参照が必要な箇所は「①未提供のため参照不可」と明記する

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `teacher-directory`（アーキテクチャ規約5章の命名規則に従い、実装ディレクトリ名は `internal/teacher_directory`）

## ②で採用した設計パターン

- Active Record（②「4. 設計パターン」）

## 採用パターンに対応する構造

アーキテクチャ規約「4. 設計パターンごとの構造適用方針」のActive Record構造に従う。domain層・infrastructure層のレイヤー分離、usecase層は設けない。Entity相当のstruct（Model）と永続化操作（Store）を同一packageに置く。

②「9. Repository設計」の記載どおり、②文書中の「Repository」「UseCase」という表記は、本書では以下の簡略構造へ読み替える（②文書自体は変更しない）。

```
internal/teacher_directory/
├── model.go              # Model（Entity相当）定義: TeacherAccount / InitialTeacherPermission / TeacherGradeAssignment
├── store.go               # Store定義: TeacherDirectoryStore / TeacherPermissionInitializationStore / TeacherGradeAssignmentStore
├── errors.go              # struct/Storeが返すエラー定義
└── presentation/
    ├── handler/
    │   └── teacher_handler.go
    ├── request/
    │   └── teacher_request.go
    ├── response/
    │   └── teacher_response.go
    └── routes.go
```

## 作成するファイル一覧

|パス|内容|
|-|-|
|`internal/teacher_directory/model.go`|`TeacherAccount` / `InitialTeacherPermission` / `TeacherGradeAssignment` のstruct定義と検証メソッド|
|`internal/teacher_directory/store.go`|`TeacherDirectoryStore` / `TeacherPermissionInitializationStore` / `TeacherGradeAssignmentStore` の定義とメソッド|
|`internal/teacher_directory/errors.go`|Model/Storeが返すエラー変数・エラー型の定義|
|`internal/teacher_directory/presentation/handler/teacher_handler.go`|`TeacherHandler`（List/Show/Create）|
|`internal/teacher_directory/presentation/request/teacher_request.go`|Request DTO|
|`internal/teacher_directory/presentation/response/teacher_response.go`|Response DTO|
|`internal/teacher_directory/presentation/routes.go`|ルーティング登録|

**対象外**: `domain/`, `application/usecase/`, `infrastructure/repository/`（Active Record採用のため設けない）

**②からの補足**: `GradeStore`（学年の存在確認・所属校取得）は②「9. Repository設計」に「School/Grade Context提供・参照専用」と明記されている。同一リポジトリ内の別Bounded Context（School/Grade系。アーキテクチャ規約5章の一覧では`school-directory`が該当候補）が公開する参照専用Storeを、アーキテクチャ規約「6. Context間連携ルール」に従い直接呼び出す想定とする。ただし当該Contextの②/③文書は本タスクでは提供されていないため、正確なpackage pathは「推測」であり、実装時に確認が必要。

---

# 3. Domain層設計

**対象外（Active Record採用のため、domain層を設けない）。** 以下、②「6. Entity設計」「7. Value Object設計」「8. Domain Service」「14. Error設計」を、アーキテクチャ規約「4. 設計パターンごとの構造適用方針」に従いActive Record向けに読み替えて記載する。

## Model（Entity相当）

### TeacherAccount

②「6. Entity設計」の`TeacherAccount`に対応する。

|項目|内容|
|-|-|
|struct名|`TeacherAccount`|
|フィールド|`ID uint`（主キー）／`SchoolID uint`（所属校ID。同校スコープ判定に使用）／`Name string`（氏名、漢字表記。②7節「Value Objectを採用しないもの」より単純文字列）／`NameKana string`（氏名カナ）／`Email string`（メールアドレス）／`Role string`（教員ロール識別、②からの補足：既存`users`テーブルのロール区分に合わせる想定）／`CreatedAt time.Time`／`UpdatedAt time.Time`／`DeletedAt gorm.DeletedAt`（論理削除。②4節「有効/無効の区別は`deleted_at`による論理削除のみ」に基づく）|
|公開メソッド|`Validate() error`：氏名の必須チェック、`NameKana`のカタカナ・長音・中黒・空白のみ許容の形式チェック（②7節`NameKana`のルール）、`Email`の一般的なメールアドレス形式チェック（②7節`Email`のルール）。実装ロジックは記述しない|
|不変条件|`Validate()`を通過したインスタンスのみが`Store.Create`で永続化される（Storeが呼び出し順序として保証する。コンストラクタでの強制は行わない）|

**②からの補足**: `Role`・`CreatedAt`・`UpdatedAt`・`DeletedAt`は②「6. Entity設計」に明記がなく、Gorm規約のタイムスタンプ／論理削除慣行と②17節「既存Rails DBを継続利用する」を踏まえて実装上必要なフィールドとして補った。

### InitialTeacherPermission

②「6. Entity設計」の`InitialTeacherPermission`に対応する。

|項目|内容|
|-|-|
|struct名|`InitialTeacherPermission`|
|フィールド|`ID uint`／`TeacherID uint`（`TeacherAccount.ID`への参照）／`GradeScope string`（`own_grade` / `all_grades`）／`ManageOtherTeachers bool`／`CreatedAt time.Time`／`UpdatedAt time.Time`|
|公開メソッド|`Validate() error`：`GradeScope`が`own_grade`/`all_grades`のいずれかであることの検証（②7節`GradeScope`の許容値ルール）。`ManageOtherTeachers`はGoの`bool`型であるため、②12節が求める「真偽値として妥当か」の検証はGoの型システムにより静的に保証され、追加のランタイム検証は不要（②からの補足）|
|不変条件|作成時点でのみ存在し、以後の更新は本Context・本Modelの責務範囲外（②6節・②9節「以後の変更はTeacher Permission Contextの責務」）|

### TeacherGradeAssignment

②「6. Entity設計」の`TeacherGradeAssignment`に対応する。

|項目|内容|
|-|-|
|struct名|`TeacherGradeAssignment`|
|フィールド|`ID uint`／`TeacherID uint`（`TeacherAccount.ID`への参照）／`GradeID uint`（担当学年ID）／`CreatedAt time.Time`／`UpdatedAt time.Time`|
|公開メソッド|`ValidateSameSchool(teacherSchoolID, gradeSchoolID uint) error`：指定学年の所属校が教員の所属校と一致するかを検証する。②「8. Domain Service」の`TeacherGradeAssignmentPolicy`に相当する判定を、Active Record採用のためDomain Serviceとして独立させず、Modelのメソッドとして持たせる（アーキテクチャ規約4章「Active Record採用時、検証ルールはModelのメソッドとして記載」に従う）|
|不変条件|`ValidateSameSchool`を満たした学年IDのみが`Store.Create`で永続化される|

**②からの補足**: ②「8. Domain Service」の`TeacherGradeAssignmentPolicy`は概念上Domain Serviceとして設計されているが、本書ではアーキテクチャ規約4章の指示（Active Record採用時はDomain Serviceを原則「対象外」とし、検証ルールはModelのメソッドとして記載）に従い、`TeacherGradeAssignment.ValidateSameSchool`として読み替えた。②の判定ロジック・判断根拠自体は変更していない。

## Value Object

**対象外**（アーキテクチャ規約4章の指示により、Active Record採用時は原則対象外とする）。②「7. Value Object設計」の`NameKana`／`Email`／`GradeScope`の形式・許容値検証ルールは、上記各Modelの`Validate()`メソッド内の検証内容として統合した。

## Repository Interface

**対象外**（Active Record採用のため、domain層にRepository Interfaceを定義しない。依存性逆転を行わない）。永続化操作は「5. Infrastructure層設計」のStoreとして直接実装する。

## Domain Service

**対象外**。②「8. Domain Service」の`TeacherGradeAssignmentPolicy`は、上記`TeacherGradeAssignment.ValidateSameSchool`に統合した。

## Domain Event

②「15. Domain Event」のとおり、本機能ではDomain Eventを採用しない。よって対象外。

## Domain Error（struct/Storeが返すエラー）

アーキテクチャ規約4章の指示に従い、②「14. Error設計」の「Domain Error」は、Model/Storeが返すエラーとして読み替える。

|エラー|発生元|発生条件|
|-|-|-|
|`ErrInvalidNameKana`|`TeacherAccount.Validate()`|氏名カナがカタカナ・長音・中黒・空白以外を含む|
|`ErrInvalidEmail`|`TeacherAccount.Validate()`|メールアドレスが一般的な形式でない|
|`ErrInvalidGradeScope`|`InitialTeacherPermission.Validate()`|`GradeScope`が`own_grade`/`all_grades`以外|
|`ErrGradeSchoolMismatch`|`TeacherGradeAssignment.ValidateSameSchool()`|指定学年の所属校が教員の所属校と一致しない（②14節「指定学年が同校でない」）|

エラー種別ごとの型／変数定義方針は「11. Error実装方針」で扱う。

---

# 4. Application層設計

**対象外（Active Record採用のため、usecase層を設けない）。** ②「10. UseCase設計」の`ListTeachers`／`ShowTeacher`／`CreateTeacher`は、Handlerが`Store`を直接呼び出す処理として「6. Presentation層設計」のHandler処理順序に統合して記載する。DTO（Command/Query）は独立したapplication層のDTOとして設けず、「6. Presentation層設計」のRequest/Response DTOがその役割を兼ねる。

---

# 5. Infrastructure層設計

**Repository実装**: 対象外（Active Record採用のため、Repository Interfaceおよびその実装を設けない）。

## Store実装

②「9. Repository設計」で定義された各Storeを、アーキテクチャ規約4章に従い「3. Domain層設計」のModelと同一package（`internal/teacher_directory`）に配置する。GORMモデルはModelのstructをそのままGORMタグ付きで扱う（Model⇔GORMモデルの別途変換は行わない）。

### TeacherDirectoryStore

②9節の`TeacherDirectoryStore`に対応する。

|項目|内容|
|-|-|
|struct名|`TeacherDirectoryStore`|
|対応GORMモデル|`TeacherAccount`（テーブル: `users`）|
|コンストラクタ|依存として`*gorm.DB`を受け取る|

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindBySchoolID`|`ctx context.Context, schoolID uint, page int`|`[]TeacherAccount, PageInfo, error`|`school_id = schoolID` かつ `role = teacher` かつ論理削除されていないレコードを対象に、`name_kana`昇順でソートし、ページ番号に応じたオフセット・件数制限を適用して取得する（②9節「氏名カナ順のソート」「ページネーション」）|
|`FindByIDAndSchoolID`|`ctx context.Context, id uint, schoolID uint`|`*TeacherAccount, error`|`id = id` かつ `school_id = schoolID` かつ `role = teacher` の条件で1件取得する。該当なしの場合はレコード不存在を表すエラーを返す|
|`CreateTeacherWithInitialSetup`|`ctx context.Context, account *TeacherAccount, permission *InitialTeacherPermission, assignment *TeacherGradeAssignment`|`*TeacherAccount, error`|トランザクション内で`account`を作成し、採番された`account.ID`を`permission.TeacherID`・`assignment.TeacherID`に設定した上で、`permission`・`assignment`を作成する（③独自の集約メソッド。詳細は「8. Transaction実装方針」参照）|

**②からの補足**: ②9節では`TeacherDirectoryStore`の責務は「一覧取得」「詳細取得」「新規教員アカウントの作成」のみで、権限・担当学年の作成は各専用Storeの責務とされている。しかし②11節「Transaction開始位置」は「CreateTeacherの処理に対応するStoreメソッド内」という単一の記述であり、3つのStoreにまたがる処理をどのStoreに集約するかは②に明記がない。本書では、`TeacherDirectoryStore`にトランザクション全体を集約する`CreateTeacherWithInitialSetup`を置く設計とした（推測。判断理由は「8. Transaction実装方針」参照）。

### TeacherPermissionInitializationStore

②9節の`TeacherPermissionInitializationStore`に対応する。

|項目|内容|
|-|-|
|struct名|`TeacherPermissionInitializationStore`|
|対応GORMモデル|`InitialTeacherPermission`（テーブル: `teacher_permissions`）|

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`Create`|`ctx context.Context, tx *gorm.DB, permission *InitialTeacherPermission`|`error`|渡されたトランザクション（`tx`）を用いて`teacher_permissions`へ1件挿入する（②9節「着任時の初期権限レコードの作成」）|

検索機能は持たない（②9節「保持する検索機能: なし（作成専用）」）。

### TeacherGradeAssignmentStore

②9節の`TeacherGradeAssignmentStore`に対応する。

|項目|内容|
|-|-|
|struct名|`TeacherGradeAssignmentStore`|
|対応GORMモデル|`TeacherGradeAssignment`（テーブル: `teacher_grades`）|

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`Create`|`ctx context.Context, tx *gorm.DB, assignment *TeacherGradeAssignment`|`error`|渡されたトランザクションを用いて`teacher_grades`へ1件挿入する|
|`FindByTeacherID`|`ctx context.Context, teacherID uint`|`[]TeacherGradeAssignment, error`|`teacher_id = teacherID`の条件で担当学年一覧を取得する（②9節「教員IDによる担当学年取得」）|

## 外部連携実装

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`GradeStore`（School/Gradeコンテキスト提供・参照専用、②9節）|`TeacherHandler.Create`（作成時、指定学年の存在・所属校確認）|本Context内では実装しない。School/Gradeコンテキストが公開する参照専用Storeを直接呼び出す（アーキテクチャ規約6章「Context間連携ルール」。Active Record採用のため依存性逆転は行わず、具象Storeを直接呼び出す）。②からの補足：正確な呼び出し先package pathは②に明記がなく「推測」。実装時に該当コンテキストの実装状況を確認する必要がある|

Mail・Cache・Queueは②15節「Domain Event」の記載のとおり本機能では扱わないため対象外。

---

# 6. Presentation層設計

## Handler

### TeacherHandler

|項目|内容|
|-|-|
|struct名|`TeacherHandler`|
|コンストラクタが受け取る依存|`*TeacherDirectoryStore`／`*TeacherPermissionInitializationStore`／`*TeacherGradeAssignmentStore`／School/Gradeコンテキストの参照専用Store（②からの補足の型。「5. Infrastructure層設計」参照）|
|対応する呼び出し先|Store（Active Record採用のためUseCase層を経由しない）|

メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`List`|GET|`/api/v1/teacher/colleagues`|
|`Show`|GET|`/api/v1/teacher/colleagues/:id`|
|`Create`|POST|`/api/v1/teacher/colleagues`|

### `List` 処理順序（②10節`ListTeachers`に対応）

1. Middlewareが設定したcurrent user（`teacher`ロール、所属校ID）をcontextから取得する
2. クエリパラメータ`page`をRequest DTOにバインドし、型・範囲を検証する（Presentation Validation）
3. `TeacherDirectoryStore.FindBySchoolID(ctx, currentUser.SchoolID, page)`を呼び出す（②13節：Handlerがcurrent userの所属校IDでスコープを付与する。usecase層を経由しない分、本来UseCaseが担っていたスコープ制御をここで明記する）
4. 取得結果をResponse DTOへ変換して返す

### `Show` 処理順序（②10節`ShowTeacher`に対応）

1. current userをcontextから取得する
2. パスパラメータ`id`をRequest DTOにバインドし、型を検証する
3. `TeacherDirectoryStore.FindByIDAndSchoolID(ctx, id, currentUser.SchoolID)`を呼び出す。該当なしの場合は404を返す（②16節「404: 対象教員不存在」）
4. `TeacherGradeAssignmentStore.FindByTeacherID(ctx, id)`を呼び出し、担当学年情報を取得する
5. 権限情報（`InitialTeacherPermission`相当）を取得する（②からの補足：下記参照）
6. 取得結果をResponse DTOへ変換して返す

**②からの補足（推測）**: ②10節`ShowTeacher`の出力は「教員詳細（権限・担当学年を含む）」とされているが、「呼び出すStore」には`TeacherDirectoryStore`と`TeacherGradeAssignmentStore`のみが列挙され、`TeacherPermissionInitializationStore`は含まれていない。かつ②9節では同Storeは「保持する検索機能: なし（作成専用）」と明記されている。したがって作成時点の初期権限を詳細画面でどう参照するかは②に情報がなく、本書では「Teacher Permission Context（②3節で言及されている以後の権限ライフサイクル管理コンテキスト）が公開する参照専用の手段を呼び出す」と仮定した。当該Contextの②/③文書は本タスクでは提供されておらず、具体的な呼び出し方は実装時に確認が必要（推測）。

### `Create` 処理順序（②10節`CreateTeacher`に対応）

1. current user（`teacher`ロール、所属校ID）をcontextから取得する
2. Request Bodyを`CreateTeacherRequest`にバインドし、型・必須・フォーマットを検証する（Presentation Validation。②12節「Presentation」）
3. `TeacherAccount`・`InitialTeacherPermission`・`TeacherGradeAssignment`のインスタンスをRequest DTOとcurrent userの所属校IDから組み立てる
4. 各Modelの`Validate()`を呼び出す（氏名カナ形式・メール形式・grade_scope許容値。②12節「Domain」の「状態チェック」に相当）
5. School/Gradeコンテキストの`GradeStore`を呼び出し、指定学年の存在・所属校を取得する。学年が存在しない場合の扱いは②に明記がないため、手順6の同校制約違反と同様にDomain Errorとして扱う（推測。「11. Error実装方針」参照）
6. `TeacherGradeAssignment.ValidateSameSchool(currentUser.SchoolID, grade.SchoolID)`を呼び出す（②13節「Domain」：同校でない学年の割当を拒否）
7. `TeacherDirectoryStore.CreateTeacherWithInitialSetup(ctx, account, permission, assignment)`を呼び出す（1トランザクションで3レコードを作成。「8. Transaction実装方針」参照）
8. 作成結果をResponse DTOへ変換し、201を返す

## Request / Response DTO

### Request DTO

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`ListTeachersRequest`|`Page int`（クエリパラメータ`page`）|型チェックのみ。未指定時のデフォルト値・妥当な範囲は②に明記がなく、①未提供のため参照不可。実装時に既存Rails仕様のデフォルト挙動を確認する必要がある（②からの補足）|
|`ShowTeacherRequest`|`ID uint`（パスパラメータ`id`）|必須、正の整数であること|
|`CreateTeacherRequest`|`Name string`／`NameKana string`／`Email string`／`GradeID uint`／`GradeScope string`／`ManageOtherTeachers bool`|全フィールド必須（②12節「必須チェック」）。`Email`はメールアドレス形式チェック、`NameKana`はカタカナ形式チェック（②12節「フォーマットチェック」）。`GradeScope`は`own_grade`/`all_grades`のいずれかであることをタグレベルでも検証してよい（Modelの`Validate()`と役割は重複するが、②12節「責務分離」の「Presentationは入力の形式が正しいかを担当する」に沿った形式レベルのチェックとして許容する）|

### Response DTO

|struct名|フィールドと型|
|-|-|
|`TeacherSummaryResponse`|`ID uint`／`Name string`／`NameKana string`／`Email string`（一覧表示用。①未提供のため詳細フィールド構成は②に記載の意味的範囲に留めた）|
|`TeacherListResponse`|`Teachers []TeacherSummaryResponse`／`Pagination PaginationResponse`|
|`PaginationResponse`|`Page int`／`TotalPages int`／`TotalCount int`（②からの補足：具体的なページング情報の項目は①未提供のため一般的な構成を仮定）|
|`TeacherGradeResponse`|`GradeID uint`（担当学年。学年名等の付随情報は①未提供のため含めるか未確定）|
|`TeacherDetailResponse`|`ID uint`／`Name string`／`NameKana string`／`Email string`／`GradeScope string`／`ManageOtherTeachers bool`／`Grades []TeacherGradeResponse`|
|`CreateTeacherResponse`|`ID uint`／`Name string`／`NameKana string`／`Email string`／`GradeScope string`／`ManageOtherTeachers bool`／`GradeID uint`|

**②からの補足**: Response DTOの詳細フィールド構成は、Rails側のSerializer実装（①）に依存するが①は未提供のため参照できない。②16節「一覧・詳細・作成のレスポンス構造はRails現行仕様に近い意味を維持する」を踏まえ、②の記載範囲（氏名・氏名カナ・メール・権限・担当学年）から推測できる最小限の構成とした。実装時にRailsのSerializer実装またはフロントエンドの利用箇所を確認し、フィールドを確定する必要がある。

## Routing

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/teacher/colleagues`|`TeacherHandler.List`|
|GET|`/api/v1/teacher/colleagues/:id`|`TeacherHandler.Show`|
|POST|`/api/v1/teacher/colleagues`|`TeacherHandler.Create`|

`routes.go`にて、`teacher`ロールを要求する認証Middlewareを経由した上でこれらのルートを登録する（②13節「Middleware」）。

---

# 7. API仕様

②「16. API互換方針」をもとに、実装対象のEndpointを一覧化する。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|`/api/v1/teacher/colleagues`|`TeacherHandler.List`|`ListTeachersRequest`|`TeacherListResponse`|200|
|GET|`/api/v1/teacher/colleagues/:id`|`TeacherHandler.Show`|`ShowTeacherRequest`|`TeacherDetailResponse`|200|
|POST|`/api/v1/teacher/colleagues`|`TeacherHandler.Create`|`CreateTeacherRequest`|`CreateTeacherResponse`|201|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|`page`が不正な型・範囲外|422|Presentation Validationエラー（②12節「Presentation」）|
|指定した教員IDが存在しない、または同校でない|404|②16節「404: 対象教員不存在」|
|`CreateTeacherRequest`の必須項目欠落・フォーマット不正|422|Presentation Validationエラー（②12節「Presentation」）|
|氏名カナがカタカナ形式でない|422|`TeacherAccount.Validate()`が返す`ErrInvalidNameKana`|
|メールアドレス形式が不正|422|`TeacherAccount.Validate()`が返す`ErrInvalidEmail`|
|`grade_scope`が許容値でない|422|`InitialTeacherPermission.Validate()`が返す`ErrInvalidGradeScope`|
|指定学年が同校でない（または存在しない、②からの補足・推測）|422|`TeacherGradeAssignment.ValidateSameSchool()`が返す`ErrGradeSchoolMismatch`|
|DB接続失敗等の技術的障害|500|Infrastructure Error（②14節「Infrastructure Error」）|

---

# 8. Transaction実装方針

②「11. Transaction設計」をもとに、実装単位へ落とし込む。

## Transaction開始箇所

- `TeacherDirectoryStore.CreateTeacherWithInitialSetup`メソッド内で、GORMのトランザクション（`db.Transaction(func(tx *gorm.DB) error { ... })`相当）を開始する

## Transaction終了箇所

- メソッド内で`TeacherAccount`の作成 → `InitialTeacherPermission`の作成（`TeacherPermissionInitializationStore.Create`にトランザクション用の`*gorm.DB`を渡して呼び出す） → `TeacherGradeAssignment`の作成（`TeacherGradeAssignmentStore.Create`にトランザクション用の`*gorm.DB`を渡して呼び出す）が全て成功した時点でコミットする
- いずれか1つでも失敗した場合はロールバックする（②11節「アカウント・権限・担当学年のいずれかのみが作成される不整合な状態を防ぐ」）
- `List`・`Show`の処理ではトランザクションを使用しない（②11節）

## 複数Store（関数）にまたがる場合の扱い

- `TeacherPermissionInitializationStore.Create`・`TeacherGradeAssignmentStore.Create`は、トランザクション制御を持たず、呼び出し元（`TeacherDirectoryStore.CreateTeacherWithInitialSetup`）から渡された`*gorm.DB`（トランザクションコンテキスト）を用いて実行する
- `GradeStore`（School/Gradeコンテキスト）への問い合わせは学年の存在・所属校確認のための読み取り専用アクセスであり、書き込みトランザクションの外（Handler側、手順5の時点）で実行する

**②からの補足（推測）**: ②11節は「CreateTeacherの処理に対応するStoreメソッド内でトランザクションを開始する」と記載するのみで、3つのStoreにまたがる処理をどのStoreメソッドに集約するかは明記されていない。本書では`TeacherDirectoryStore`（教員アカウント自体を管理する中心的Store）にトランザクション制御を集約する設計とした。Active Record採用機能ではusecase層・Repository Interfaceによる依存性逆転を追加しない方針（規約「4. 設計パターンごとの構造適用方針」）のため、複数Storeを束ねる別structを新設せず、既存Storeの1メソッドとして実装する。

---

# 9. Validation実装方針

②「12. Validation設計」を実装レベルに落とし込む。

## Presentation

- `ListTeachersRequest.Page`: 型（整数）チェック
- `ShowTeacherRequest.ID`: 型（正の整数）・必須チェック
- `CreateTeacherRequest`: `Name`／`NameKana`／`Email`／`GradeID`／`GradeScope`／`ManageOtherTeachers`の必須チェック、`Email`のメールアドレス形式チェック、`NameKana`のカタカナ形式チェック（②12節「Presentation」）

## 業務ルール検証（Active Record: Modelのメソッド）

- `TeacherAccount.Validate()`: 氏名カナがカタカナ・長音・中黒・空白のみで構成されるか（②7節`NameKana`）、メールアドレスが一般的な形式か（②7節`Email`）
- `InitialTeacherPermission.Validate()`: `GradeScope`が`own_grade`/`all_grades`のいずれかか（②7節`GradeScope`）
- `TeacherGradeAssignment.ValidateSameSchool()`: 指定学年の所属校が教員の所属校と一致するか（②8節`TeacherGradeAssignmentPolicy`相当、②13節「Domain」）

②12節「責務分離」のとおり、Presentationは入力形式の妥当性、Model側のメソッドは業務的な妥当性（同校の学年か、許容される権限値か）を担当する。

---

# 10. Authorization実装方針

②「13. Authorization設計」を実装レベルに落とし込む。

## Middleware

- JWT等により認証済みユーザーを特定し、`teacher`ロールであることを確認する（②13節「Middleware」）

## Handler

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定自体は持たせない（②13節「Handler」）
- ただしActive Record採用によりUseCase層を経由しないため、本来UseCaseが担っていた以下の手順をHandlerが行う（②13節「UseCase」相当）:
  - current userの所属校IDを用いて、`List`／`Show`の取得範囲を同校にスコープする
  - `Create`時、指定学年が同校かどうかの判定材料としてcurrent userの所属校を`TeacherGradeAssignment.ValidateSameSchool`に渡す

## Store／Model

- `TeacherGradeAssignment.ValidateSameSchool`において、同校でない学年の割当を拒否する（②13節「Domain」相当）
- `TeacherDirectoryStore.FindBySchoolID`／`FindByIDAndSchoolID`のクエリ条件に`school_id`を含めることで、Store層でも同校スコープを担保する

---

# 11. Error実装方針

②「14. Error設計」を実装レベルに落とし込む。アーキテクチャ規約8章の指示（Active Record採用時はDomain Errorに相当する層がないため、関数・struct側で発生したエラーをApplication Error相当として扱う）に従い、本機能ではModel/Storeが返すエラーをそのままHandlerでHTTPレスポンスへ変換する2段階構成とする。

## Model/Storeが返すエラー → HTTPレスポンスへの変換方針

- `TeacherAccount.Validate()`／`InitialTeacherPermission.Validate()`／`TeacherGradeAssignment.ValidateSameSchool()`が返すエラー（②14節「Domain Error」相当）は、Handlerで422に変換する
- `TeacherDirectoryStore.FindByIDAndSchoolID`が該当なしを表すエラーを返した場合（②14節「Application Error」の「対象教員が存在しない、または同校でない」に相当）、Handlerで404に変換する
- `TeacherDirectoryStore.CreateTeacherWithInitialSetup`のトランザクション内で一部レコードのみ作成に失敗した場合（②14節「Application Error」の「作成処理における関連レコードの不整合」に相当）は、ロールバックの上でエラーを返し、Handlerで422または500（原因に応じて）に変換する
- DB接続失敗等（②14節「Infrastructure Error」）は、Handlerで500に変換する

## Status Code対応表

|Error種別|発生層|HTTP Status|
|-|-|-|
|Presentation Validationエラー|Presentation（Request DTO）|422|
|`ErrInvalidNameKana` / `ErrInvalidEmail`|Model（`TeacherAccount.Validate`）|422|
|`ErrInvalidGradeScope`|Model（`InitialTeacherPermission.Validate`）|422|
|`ErrGradeSchoolMismatch`|Model（`TeacherGradeAssignment.ValidateSameSchool`）|422|
|対象教員不存在／同校でない|Store（`TeacherDirectoryStore`）|404|
|作成処理中の関連レコード不整合|Store（`TeacherDirectoryStore.CreateTeacherWithInitialSetup`、トランザクションロールバック後）|422 または 500|
|DB接続・永続化失敗|Store（Infrastructure的な失敗）|500|

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」（既存Rails DBを継続利用、スキーマ変更なし）をもとに整理する。SQL文そのものは記載しない。

## 利用するGORMモデルとテーブルの対応

|Model|テーブル|
|-|-|
|`TeacherAccount`|`users`（既存。`role`が教員を表す値、`school_id`相当のカラムで絞り込む想定。正確なカラム名は①未提供のため参照不可であり、実装時に既存マイグレーション定義を確認する必要がある）|
|`InitialTeacherPermission`|`teacher_permissions`（既存）|
|`TeacherGradeAssignment`|`teacher_grades`（既存）|

## 主要クエリの条件・ソート・ページネーション方針

|Store／メソッド|条件|ソート|ページネーション|
|-|-|-|-|
|`TeacherDirectoryStore.FindBySchoolID`|`school_id`一致、`role`が教員であること、論理削除されていないこと|`name_kana`昇順|`page`パラメータに基づくオフセット・件数制限（1ページあたり件数は②に明記なし。①未提供のため参照不可、実装時に既存Rails仕様を確認する）|
|`TeacherDirectoryStore.FindByIDAndSchoolID`|`id`一致、`school_id`一致、`role`が教員であること|-|-|
|`TeacherDirectoryStore.CreateTeacherWithInitialSetup`|-（挿入処理）|-|-|
|`TeacherPermissionInitializationStore.Create`|-（挿入処理、`teacher_id`は作成された`TeacherAccount.ID`）|-|-|
|`TeacherGradeAssignmentStore.Create`|-（挿入処理、`teacher_id`は作成された`TeacherAccount.ID`）|-|-|
|`TeacherGradeAssignmentStore.FindByTeacherID`|`teacher_id`一致|-|-|

## 既存Schemaへの変更

- ②17節「変更なし」のとおり、本機能によるスキーマ変更は行わない

---

# 13. テストケース設計

②「18. テスト戦略」を、アーキテクチャ規約「4. 設計パターンごとの構造適用方針」・出力フォーマット指示（Active Record: 「Domain Test」→「Model Test」、「UseCase Test」は対象外、「Repository Test」→「Store Test」）に従い読み替えて具体化する。

## Model Test（②「Domain Test」相当）

|対象|テストケース|
|-|-|
|`TeacherAccount.Validate`|氏名カナがカタカナのみで構成される場合に成功する／長音・中黒・空白を含む場合に成功する／漢字・ひらがな・英数字を含む場合に失敗する|
|`TeacherAccount.Validate`|一般的な形式のメールアドレスで成功する／`@`を含まない等の不正形式で失敗する|
|`InitialTeacherPermission.Validate`|`grade_scope`が`own_grade`または`all_grades`の場合に成功する／それ以外の値の場合に失敗する|
|`TeacherGradeAssignment.ValidateSameSchool`|教員の所属校IDと学年の所属校IDが一致する場合に成功する／一致しない場合に失敗する|

## UseCase Test

対象外（Active Record採用のため、usecase層を設けない）。

## Store Test（②「Repository Test」相当）

|対象|テストケース|
|-|-|
|`TeacherDirectoryStore.FindBySchoolID`|指定校の教員のみが取得されること（他校の教員が混入しないこと）／`name_kana`昇順でソートされること／ページ指定に応じた件数・オフセットで取得されること|
|`TeacherDirectoryStore.FindByIDAndSchoolID`|存在する教員IDかつ同校の場合に取得できること／存在しないIDの場合にエラーとなること／同校でない教員IDの場合にエラーとなること|
|`TeacherDirectoryStore.CreateTeacherWithInitialSetup`|3レコード（`TeacherAccount`／`InitialTeacherPermission`／`TeacherGradeAssignment`）が全て作成されること／途中で失敗した場合に全レコードがロールバックされ、一部のみ作成された状態が残らないこと|
|`TeacherPermissionInitializationStore.Create`|渡されたトランザクション内で正しく作成されること|
|`TeacherGradeAssignmentStore.Create`|渡されたトランザクション内で正しく作成されること|
|`TeacherGradeAssignmentStore.FindByTeacherID`|指定教員の担当学年一覧が取得できること|

## Handler Test

|対象|テストケース|
|-|-|
|`TeacherHandler.List`|正常系：200で教員一覧・ページ情報が返ること／`page`が不正な場合に422が返ること|
|`TeacherHandler.Show`|正常系：200で教員詳細が返ること／存在しない・同校でないIDの場合に404が返ること|
|`TeacherHandler.Create`|正常系：201で作成結果が返ること／必須項目欠落・フォーマット不正の場合に422が返ること／氏名カナ形式不正の場合に422が返ること／メール形式不正の場合に422が返ること／`grade_scope`不正の場合に422が返ること／指定学年が同校でない場合に422が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|一覧取得エンドポイント|`GET /api/v1/teacher/colleagues`を認証済みteacherユーザーで呼び出し、同校の教員一覧が正しく返ること|
|詳細取得エンドポイント|`GET /api/v1/teacher/colleagues/:id`を呼び出し、権限・担当学年を含む詳細が正しく返ること|
|作成エンドポイント|`POST /api/v1/teacher/colleagues`で新規教員を作成した後、一覧・詳細取得で作成結果が反映されていることを確認する|

---

# 14. ②からの補足事項

②に明記がなく、本書で実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|3レコード作成のトランザクション制御を`TeacherDirectoryStore.CreateTeacherWithInitialSetup`に集約する構成とした|②11節は「対応するStoreメソッド内でトランザクションを開始する」とのみ記載しており、3つのStoreにまたがる処理をどのStoreに置くかは明記がない。教員アカウントを管理する中心的Storeに集約するのが自然と判断した|推測|
|`GradeStore`（School/Gradeコンテキスト）の正確なpackage path|②9節は「School/Grade Context提供・参照専用」とのみ記載。当該コンテキストの②/③文書は本タスクで提供されていない|推測|
|`ShowTeacher`における権限情報の取得手段|②10節の出力仕様は「権限・担当学年を含む」だが、「呼び出すStore」に`TeacherPermissionInitializationStore`が含まれず、同Storeは検索機能を持たない（②9節）。Teacher Permission Context側の参照手段を呼び出すと仮定した|推測|
|`users`・`teacher_permissions`・`teacher_grades`テーブルの詳細カラム名（`school_id`相当のカラム名等）|①Rails実装（マイグレーション定義）が本タスクでは未提供のため参照不可|①未提供のため参照不可|
|Response DTOの詳細フィールド構成|Rails側のSerializer実装（①）が未提供。②16節「Rails現行仕様に近い意味を維持する」の記載から、②の言及範囲（氏名・氏名カナ・メール・権限・担当学年）に基づく最小構成とした|推測|
|Errorレスポンスの具体的なJSON構造|②16節「既存のerrors形式を踏襲」とあるが、具体的な形式は①未提供のため確認できない|①未提供のため参照不可|
|指定学年が存在しない場合のステータスコードを、同校制約違反と同様に422（Domain Error相当）とした|②14節「Domain Error」は「指定学年が同校でない」のみを挙げており、「学年が存在しない」場合の扱いは明記がない|推測|
|`page`のデフォルト値・1ページあたりの件数|②に具体的な数値の記載がなく、①（Rails実装）も未提供のため確認できない|①未提供のため参照不可|
|`InitialTeacherPermission.ManageOtherTeachers`について、Go言語の型システム（`bool`型）により②12節の「真偽値として妥当か」の検証を静的に保証されるとし、追加のランタイム検証を設けなかった|Goの型安全性に基づく技術的判断であり、②の業務ルールを変更するものではない|-|
|新規教員作成時のパスワード・招待フローはCreateTeacherの入力・処理から除外した|②15節で「具体的な送信タイミング・手段は本資料の調査範囲では特定できなかった」と明記されており、②10節`CreateTeacher`の入力にもパスワード関連項目が含まれていない|②に準拠（新規の推測ではない）|

---
