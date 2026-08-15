# 管理者教員管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が自校（選択中の高校）に所属する教員アカウントを一覧参照し、新規教員の招待（作成）、既存教員のプロフィール・権限・担当学年の更新を行う機能である。教員の権限情報（`teacher_permissions`）と担当学年情報（`teacher_grades`）を教員本体と合わせて一貫管理する（②「1. 機能概要」）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により **Active Record** を採用する。

- 主要操作が一覧・作成・更新というCRUDに限定される
- 権限情報（`teacher_permissions`）は教員に対して1:1で従属する属性群であり、独立した振る舞いを持たない
- 担当学年（`teacher_grades`）は「更新時に全置換する」という単純な同期ルールであり、複雑な状態遷移を伴わない
- Handlerが直接Storeを呼び出し、認可・入力検証をHandler／Store側で行うことで、教員データの保存・関連付けの振る舞いを一元化しやすい

Transaction Scriptは権限・担当学年同期ロジックがUseCaseに偏り再利用性が下がることを理由に、Domain Modelは複雑な状態遷移が存在しないことを理由に、Event Sourcingは非同期通知・監査要件が現行仕様に明記されていないことを理由に、②でいずれも不採用と判断されている。本書はこれらの判断を変更しない。

本書は、`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」の **Active Record** 節の構造（Entity相当のstruct定義＋Store、usecase層なし、Repository Interfaceの分離なし）に従って実装レベルへ落とし込む。

## 本書が対象とする実装範囲

- `GET /api/v1/admin/high_schools/:high_school_id/teachers`（教員一覧取得）
- `POST /api/v1/admin/high_schools/:high_school_id/teachers`（教員招待＝作成）
- `PATCH /api/v1/admin/high_schools/:high_school_id/teachers/:id`（教員更新）

の3エンドポイントの実装に必要な、struct・Store・Handler・Routing・Request/Response構造体の実装単位を規定する。

- 対象外: HighSchool・Grade・教員アカウント（Userレコード）そのものの生成能力は他Contextの責務であり（②「3. Bounded Context」他Contextとの依存関係）、本書はteacher-management Contextが公開・利用する参照手段のインターフェースまでを扱う
- ①Rails実装（`Admin::TeachersController`・`CreateTeacherService`・`UpdateTeacherService`・`Admin::TeacherSerializer`等の実装詳細）は本書作成時点で未提供のため参照不可であり、該当箇所は「①未提供のため参照不可」として扱い、②の記載のみを実装仕様の根拠とする

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `teacher-management`（②「3. Bounded Context」）

## ②で採用した設計パターン

- Active Record（②「4. 設計パターン」）

## 採用パターンに対応する構造

`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」Active Record節に従い、domain/infrastructureのレイヤー分離およびusecase層を設けない。Entity相当のstructと永続化操作（Store）を同一package（`internal/teacher_management`）に置く。

## 作成するディレクトリ一覧

```
internal/teacher_management/
internal/teacher_management/presentation/
internal/teacher_management/presentation/handler/
internal/teacher_management/presentation/request/
internal/teacher_management/presentation/response/
```

**②からの補足**: アーキテクチャ規約「9. 命名規約」により、`internal/`配下のディレクトリ名は英単語1語または短いスネークケースとする。②のContext名`teacher-management`と`internal/`配下のディレクトリ名の対応関係は②に明記がないため、本書では`internal/teacher_management`と判断した（推測。詳細は「14. ②からの補足事項」参照）。

## 作成するファイル一覧

```
internal/teacher_management/teacher.go                        # Teacher struct・Validate()等
internal/teacher_management/teacher_permission.go              # TeacherPermission struct（VO相当）
internal/teacher_management/teacher_grade_assignment.go        # TeacherGradeAssignment struct・GradeAssignmentSet
internal/teacher_management/teacher_grade_assignment_policy.go # 学年所属妥当性判定（②「8. Domain Service」相当）
internal/teacher_management/dependency.go                      # 他Context参照用インターフェース定義（HighSchoolExistenceChecker / GradeReferenceChecker）
internal/teacher_management/errors.go                          # struct/Storeが返すエラー変数定義
internal/teacher_management/teacher_store.go                   # TeacherStore
internal/teacher_management/teacher_permission_store.go        # TeacherPermissionStore
internal/teacher_management/teacher_grade_store.go              # TeacherGradeStore
internal/teacher_management/presentation/handler/teacher_handler.go
internal/teacher_management/presentation/request/teacher_request.go
internal/teacher_management/presentation/response/teacher_response.go
internal/teacher_management/presentation/routes.go
```

`domain/` `application/` `infrastructure/` の各ディレクトリはActive Record採用のため作成しない（規約4章）。

---

# 3. Domain層設計

**実装上の位置づけ**: 本機能はActive Record採用のため、domain層のディレクトリ分離は行わない。以下は②「6〜9章」の設計意図を、Active Record構造（Model＝struct＋メソッド、Store＝永続化）に落とし込んだものである（規約4章 Active Record節「『Value Object』『Repository Interface』『Domain Service』は原則『対象外』とし、検証ルールはModelのメソッドとして記載する」に従う）。

## Model（Entity相当）

### Teacher（`internal/teacher_management/teacher.go`）

②「6. Entity設計」Teacherの責務を反映する。教員としての`User`を表す。

フィールド:

|フィールド|型|意味|
|-|-|-|
|ID|uint|教員ID（`users`テーブルの主キー）|
|HighSchoolID|uint|所属高校ID（②「1. 機能概要」対象は自校所属教員に限定される、の根拠フィールド）|
|Name|string|教員氏名|
|Email|string|メールアドレス（②「12. Validation設計」フォーマットチェック対象）|
|CreatedAt|time.Time|作成日時（GORM自動設定。Gorm規約「タイムスタンプのトラッキング」）|
|UpdatedAt|time.Time|更新日時（GORM自動設定）|

`users`テーブルにはteacher-management以外のContext（認証等）が管理するカラム（パスワードハッシュ・ロール等）が存在すると推測されるが、①未提供のため詳細は不明であり、本structは教員管理に必要な最小限のフィールドのみを保持する（**②からの補足・推測**。詳細は「14. ②からの補足事項」参照）。

公開method一覧（シグネチャのみ。実装ロジックは記載しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTeacher`|`(highSchoolID uint, name, email string) (*Teacher, error)`|`(*Teacher, error)`|新規Teacher生成時の不変条件（必須項目）を保証するファクトリ|
|`(t *Teacher) Validate() error`|なし|`error`|name・email等必須項目の妥当性を検証する（②「12. Validation設計」Domainの一部）|
|`(t *Teacher) ApplyProfile(name, email string) error`|`name, email string`|`error`|更新時のプロフィール項目差し替えと検証をあわせて行う|
|`(t Teacher) BelongsToHighSchool(highSchoolID uint) bool`|`highSchoolID uint`|`bool`|対象教員が指定高校に所属するかの判定（②「12. Validation設計」整合性チェック：更新対象の教員が対象高校に所属しているかどうか）|

不変条件（`NewTeacher`で保証する内容）:

- HighSchoolID・Name・Emailは空値（ゼロ値）を許容しない

### TeacherPermission（`internal/teacher_management/teacher_permission.go`）

②「7. Value Object設計」TeacherPermissionの採用理由・独自ルールを反映する。`teacher_permissions`テーブルに対応する、教員に1:1で従属する属性群として実装する。

フィールド:

|フィールド|型|意味|
|-|-|-|
|ID|uint|権限レコードID（主キー）|
|UserID|uint|教員（Teacher.ID）への参照|
|GradeScope|string|閲覧権限スコープ（②「7. Value Object設計」許容される範囲値のみを受け付ける。具体的な値集合は②に明記がないため「14. ②からの補足事項」参照）|
|ManageOtherTeachers|bool|他教員管理権限の有無|
|CreatedAt|time.Time|作成日時（GORM自動設定）|
|UpdatedAt|time.Time|更新日時（GORM自動設定）|

公開method一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTeacherPermission`|`(userID uint, gradeScope string, manageOtherTeachers bool) (*TeacherPermission, error)`|`(*TeacherPermission, error)`|`grade_scope`が許容範囲内であることを検証して生成するファクトリ（②「7. Value Object設計」独自ルール）|
|`(p *TeacherPermission) Validate() error`|なし|`error`|`grade_scope`の妥当性を検証する|
|`(p *TeacherPermission) Apply(gradeScope string, manageOtherTeachers bool) error`|`gradeScope string, manageOtherTeachers bool`|`error`|更新時の値差し替えと検証をあわせて行う|

②「7. Value Object設計」ではTeacherPermissionはEntity属性ではなくValue Objectとして採用されている（「将来的に権限の組み合わせに対するバリデーションや組み合わせ制約が追加された場合に、Teacherエンティティを肥大化させずに拡張できるため」）。Active Record構造では、この設計意図を独立structとして維持しつつ、永続化は独立したStore（後述TeacherPermissionStore）で扱う。

### TeacherGradeAssignment / GradeAssignmentSet（`internal/teacher_management/teacher_grade_assignment.go`）

②「6. Entity設計」TeacherGradeAssignmentの責務、②「7. Value Object設計」GradeAssignmentSetの採用理由・独自ルールを反映する。

`TeacherGradeAssignment`（`teacher_grades`テーブルの1行に対応）フィールド:

|フィールド|型|意味|
|-|-|-|
|ID|uint|紐付けID（主キー）|
|UserID|uint|教員（Teacher.ID）への参照|
|GradeID|uint|学年ID|
|CreatedAt|time.Time|作成日時（GORM自動設定）|

公開method一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTeacherGradeAssignment`|`(userID, gradeID uint) (*TeacherGradeAssignment, error)`|`(*TeacherGradeAssignment, error)`|userID・gradeIDが0でないことを保証するファクトリ|

`GradeAssignmentSet`（担当学年の全置換操作を表す型。②「7. Value Object設計」の「単なる配列以上の意味を持つ」という採用理由を反映）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewGradeAssignmentSet`|`(gradeIDs []uint) GradeAssignmentSet`|`GradeAssignmentSet`|重複IDを排除して生成する（②「7. Value Object設計」独自ルール：重複IDは排除する）。空配列は担当学年なしとして許容する（同：空配列の場合は担当学年なしとして扱う）|
|`(s GradeAssignmentSet) IDs() []uint`|なし|`[]uint`|重複排除後の学年ID一覧を返す|
|`(s GradeAssignmentSet) IsEmpty() bool`|なし|`bool`|担当学年なしかどうかを判定する|

## Value Object

規約4章「Active Record」節により、Value Objectを独立した層としては設けない（原則「対象外」）。②「7. Value Object設計」で定義されたTeacherPermission・GradeAssignmentSetの独自ルールは、上記「Model」節のとおりstruct・メソッドとしてすべて実現している。②「Value Objectを採用しないもの」（名前・メールアドレス）は、上記Teacherの単純な`string`フィールドとして扱う。

## Repository Interface

規約4章「Active Record」節により、domain層にRepository Interfaceを定義しない（原則「対象外」）。②「9. Repository設計」のTeacherStore・TeacherPermissionStore・TeacherGradeStoreの責務は「5. Infrastructure層設計」節のStoreとして実装する。

なお、②「9. Repository設計」のHighSchoolStore・GradeStoreは、高校・学年の実在確認や所属確認という、HighSchool Context・Grade Context（②「3. Bounded Context」他Contextとの依存関係）が公開する参照手段であり、teacher-managementがこれらのデータを所有・実装するものではない（`規約/アーキテクチャ規約.md`「6. Context間連携ルール」）。teacher-management側では、コーディング規約「5. インターフェース」（利用側でインターフェースを定義する）に従い、`internal/teacher_management/dependency.go`に以下の参照用interfaceを定義する。

```go
// HighSchoolExistenceChecker は HighSchool Context が公開する
// 「指定 high_school_id が存在するか」の参照手段を表す。
// （②「9. Repository設計」HighSchoolStoreの責務に対応）
type HighSchoolExistenceChecker interface {
    Exists(ctx context.Context, highSchoolID uint) (bool, error)
}

// GradeReferenceChecker は Grade Context が公開する
// 「指定 grade_ids のうち、対象高校に属するものはどれか」の参照手段を表す。
// （②「9. Repository設計」GradeStoreの責務に対応）
type GradeReferenceChecker interface {
    ExistingGradeIDsForHighSchool(ctx context.Context, highSchoolID uint, gradeIDs []uint) ([]uint, error)
}
```

これらのinterfaceの実装（HighSchool Context・Grade Context側のStore）は本書の対象外である（**②からの補足**。詳細は「14. ②からの補足事項」参照）。

## Domain Service

②「8. Domain Service」TeacherGradeAssignmentPolicyを、独立したstruct/interfaceではなく、`internal/teacher_management/teacher_grade_assignment_policy.go`内のpackageレベル関数として実装する（規約4章 Active Record節に従いDomain Serviceは原則「対象外」とするための構造上の読み替え）。

|関数|引数|戻り値|責務|
|-|-|-|-|
|`ValidateGradeAssignment`|`(requestedGradeIDs []uint, allowedGradeIDsForHighSchool []uint) (invalidGradeIDs []uint, ok bool)`|`([]uint, bool)`|要求された学年ID集合のうち、対象高校に属さないものを判定する（②「8. Domain Service」の判定内容そのもの。`allowedGradeIDsForHighSchool`は`GradeReferenceChecker.ExistingGradeIDsForHighSchool`の結果をHandler側で取得して渡す）|

②「8. Domain Service」の「追加で必要としないService」（権限属性の更新自体は単純な値の置き換え）の判断は変更しない。

## Domain Event

②「15. Domain Event」により、本機能ではDomain Eventを採用しない。「対象外」とする。招待メール送信・監査ログ記録は②の時点で要件化されておらず、本書でも追加しない。

## Domain Error

規約4章「Active Record」節に従い、「struct/Storeが返すエラー」として`internal/teacher_management/errors.go`に`sentinel error`を定義する。

|変数名|発生条件|対応する②の記載|
|-|-|-|
|`ErrHighSchoolNotFound`|指定`high_school_id`が存在しない|②「14. Error設計」Application Error：高校未存在|
|`ErrTeacherNotFound`|指定教員が存在しない、または対象高校に所属しない|②「14. Error設計」Application Error：教員未存在・対象教員が指定高校に所属しない|
|`ErrGradeNotInHighSchool`|指定`grade_ids`に対象高校に属さない学年が含まれる|②「14. Error設計」Domain Error：他校の学年を担当学年として指定した場合|
|`ErrInvalidGradeScope`|`grade_scope`が許容範囲外の値である|②「7. Value Object設計」TeacherPermission独自ルール（grade_scopeは許容される範囲値のみを受け付ける）|

---

# 4. Application層設計

**実装上の位置づけ**: 本機能はActive Record採用のためusecase層を設けない。「対象外（Active Record採用のため、usecase層を設けない）」とする。②「10. UseCase設計」に記載された3つの業務操作（ListTeachers／InviteTeacher／UpdateTeacher）は、Handlerが`internal/teacher_management`のStore・Modelを直接呼び出す処理として実装する。処理順序は「6. Presentation層設計」のHandler処理順序に統合して記載する。

## DTO（Command / Query）

DTOはPresentation層のRequest/Response DTOとして「6. Presentation層設計」にまとめて記載する（Active Record採用のため、Application層独自のCommand/Query DTOは設けない）。

---

# 5. Infrastructure層設計

**実装上の位置づけ**: 本機能はActive Record採用のためinfrastructure層のディレクトリ分離は行わない。「3. Domain層設計」で定義したModelと同一package（`internal/teacher_management`）にStoreを置く。

## Store実装

### TeacherStore（`internal/teacher_management/teacher_store.go`）

- struct名: `TeacherStore`
- 対応するGORMモデル: `Teacher`（`users`テーブル。同一structをGORMタグ付きで扱う。「12. GORM/DBクエリ設計」参照）

コンストラクタ:

```go
func NewTeacherStore(db *gorm.DB) *TeacherStore
```

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindAllByHighSchool`|`(ctx context.Context, highSchoolID uint)`|`([]*Teacher, error)`|`high_school_id`一致で対象高校に所属する教員一覧を取得する（②「9. Repository設計」`high_school_id`による絞り込み）|
|`FindByIDForHighSchool`|`(ctx context.Context, id, highSchoolID uint)`|`(*Teacher, error)`|`id`と`high_school_id`の一致で1件取得する所属高校スコープ検索（②「9. Repository設計」`id`による単一取得（所属高校スコープ付き））。該当なしは`ErrTeacherNotFound`を返す|
|`CreateWithPermission`|`(ctx context.Context, t *Teacher, permission *TeacherPermission)`|`(*Teacher, error)`|Teacher作成と初期TeacherPermission作成を1トランザクションで実行する（②「10. UseCase設計」InviteTeacherのトランザクション範囲。「8. Transaction実装方針」参照）|
|`UpdateWithPermissionAndGrades`|`(ctx context.Context, t *Teacher, permission *TeacherPermission, gradeAssignmentSet *GradeAssignmentSet)`|`error`|Teacherのプロフィール更新・TeacherPermission更新・（`gradeAssignmentSet`が非nilの場合のみ）担当学年の全置換を1トランザクションで実行する（②「10. UseCase設計」UpdateTeacherのトランザクション範囲。「8. Transaction実装方針」参照）|

保持しない責務（②「9. Repository設計」保持しない責務を踏襲）: 権限の妥当性判定・学年の所属確認はStoreに持たせない。

### TeacherPermissionStore（`internal/teacher_management/teacher_permission_store.go`）

- struct名: `TeacherPermissionStore`
- 対応するGORMモデル: `TeacherPermission`（`teacher_permissions`テーブル）

コンストラクタ:

```go
func NewTeacherPermissionStore(db *gorm.DB) *TeacherPermissionStore
```

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindByUserID`|`(ctx context.Context, userID uint)`|`(*TeacherPermission, error)`|`user_id`一致で1件取得する（②「9. Repository設計」`user_id`による取得）|
|`Create`|`(ctx context.Context, p *TeacherPermission)`|`error`|新規レコードを作成する（`TeacherStore.CreateWithPermission`内から同一トランザクションで呼び出される）|
|`Update`|`(ctx context.Context, p *TeacherPermission)`|`error`|既存レコードを更新する（`TeacherStore.UpdateWithPermissionAndGrades`内から同一トランザクションで呼び出される）|

保持しない責務（②「9. Repository設計」保持しない責務を踏襲）: 権限値の妥当性判定はStoreに持たせない。

### TeacherGradeStore（`internal/teacher_management/teacher_grade_store.go`）

- struct名: `TeacherGradeStore`
- 対応するGORMモデル: `TeacherGradeAssignment`（`teacher_grades`テーブル。「12. GORM/DBクエリ設計」参照）

コンストラクタ:

```go
func NewTeacherGradeStore(db *gorm.DB) *TeacherGradeStore
```

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindGradeIDsByUserID`|`(ctx context.Context, userID uint)`|`([]uint, error)`|`user_id`一致で紐づく`grade_id`一覧を取得する（②「9. Repository設計」`user_id`による担当学年取得）|
|`ReplaceAll`|`(ctx context.Context, userID uint, gradeIDs []uint)`|`error`|`user_id`一致の既存紐付けを削除し、`gradeIDs`分のレコードを再作成する（②「9. Repository設計」担当学年の全置換（既存削除＋新規作成）、②「5. Aggregate設計」整合性を保証する単位）|

保持しない責務（②「9. Repository設計」保持しない責務を踏襲）: 学年が対象高校に属するかの判定はStoreに持たせない。

`TeacherStore.CreateWithPermission`／`UpdateWithPermissionAndGrades`は、トランザクション用の`*gorm.DB`を`NewTeacherPermissionStore`／`NewTeacherGradeStore`に渡して同一トランザクション内で各Storeのメソッドを呼び出す（Entity ⇔ GORMモデルの変換は不要。同一structをそのまま永続化する）。

Entity ⇔ GORMモデルの変換方針: Active Record採用のため、`Teacher`・`TeacherPermission`・`TeacherGradeAssignment`のstructそのものをGORMの操作対象として扱う。変換処理は設けない。

## 外部連携実装

②「15. Domain Event」「9. Repository設計」に、Mail・Cache・Queue等の外部連携に関する記載はない。招待メール送信の詳細も②に明記がなく、①も未提供のため、本書では実装対象としない。「対象外」とする。

---

# 6. Presentation層設計

## Handler

### TeacherHandler（`internal/teacher_management/presentation/handler/teacher_handler.go`）

- struct名: `TeacherHandler`
- 依存: `*teacher_management.TeacherStore`、`*teacher_management.TeacherPermissionStore`、`*teacher_management.TeacherGradeStore`、`teacher_management.HighSchoolExistenceChecker`、`teacher_management.GradeReferenceChecker`（コンストラクタで注入する）

```go
func NewTeacherHandler(
    teacherStore *teacher_management.TeacherStore,
    permissionStore *teacher_management.TeacherPermissionStore,
    gradeStore *teacher_management.TeacherGradeStore,
    highSchoolChecker teacher_management.HighSchoolExistenceChecker,
    gradeChecker teacher_management.GradeReferenceChecker,
) *TeacherHandler
```

メソッド一覧（HTTPメソッド・パスとの対応は「7. API仕様」参照）:

|メソッド|対応API|
|-|-|
|`(h *TeacherHandler) ListTeachers(c *gin.Context)`|GET /api/v1/admin/high_schools/:high_school_id/teachers|
|`(h *TeacherHandler) InviteTeacher(c *gin.Context)`|POST /api/v1/admin/high_schools/:high_school_id/teachers|
|`(h *TeacherHandler) UpdateTeacher(c *gin.Context)`|PATCH /api/v1/admin/high_schools/:high_school_id/teachers/:id|

Active Record採用のためUseCase層を経由しない。以下、②「10. UseCase設計」で「Handler処理」として記載された業務操作の呼び出し順序を、権限チェック・呼び出し順序を含めてHandlerの処理順序として記載する（規約8章 横断的関心事の置き場所に基づき、認可（所有権・業務権限）は該当Storeまたは本Handlerで行う）。

#### ListTeachers 処理順序

1. Middlewareで設定済みのcurrent user（admin）をcontextから取得する
2. パスパラメータ`high_school_id`をバインドする
3. current adminが`high_school_id`を管理対象としているかを確認する（②「1. 機能概要」対象は自身が管理する高校に所属する教員に限定される。具体的な確認方法は②に明記がなく①未提供のため参照不可。current adminのHighSchoolIDとpath paramの一致確認と推測。「14. ②からの補足事項」参照）
4. `HighSchoolExistenceChecker.Exists`で対象高校の存在を確認する。存在しない場合は`ErrHighSchoolNotFound`（②「10. UseCase設計」ListTeachers 呼び出すStore：HighSchoolStore）
5. `TeacherStore.FindAllByHighSchool`を呼び出す（②「10. UseCase設計」ListTeachers 呼び出すStore：TeacherStore）
6. 取得した各教員について`TeacherPermissionStore.FindByUserID`・`TeacherGradeStore.FindGradeIDsByUserID`で権限・担当学年を取得する
7. 取得結果をResponse DTOへ変換して返す

#### InviteTeacher 処理順序

1. current user（admin）を取得する
2. Request DTOへバインドし、型・必須・フォーマットを検証する（②「12. Validation設計」Presentation）
3. current adminが`high_school_id`を管理対象としているかを確認する
4. `HighSchoolExistenceChecker.Exists`で対象高校の存在を確認する。存在しない場合は`ErrHighSchoolNotFound`
5. `teacher_management.NewTeacher`でTeacherを生成し、`Teacher.Validate()`で必須項目を検証する（②「12. Validation設計」Domain）
6. `teacher_management.NewTeacherPermission`で初期TeacherPermissionを生成する。`grade_scope`が許容範囲外の場合は`ErrInvalidGradeScope`
7. `TeacherStore.CreateWithPermission`をTeacherと初期TeacherPermissionで呼び出す（②「10. UseCase設計」InviteTeacher 呼び出すStore：HighSchoolStore→TeacherStore→TeacherPermissionStoreの順に対応）
8. 作成結果をResponse DTOへ変換して返す

教員アカウントのパスワード発行・招待メール送信等、アカウント生成に付随する詳細処理は、②「3. Bounded Context」により認証基盤（Account/Authentication Context）の責務とされており、本書の対象外とする（①未提供のため参照不可）。

#### UpdateTeacher 処理順序

1. current user（admin）を取得する
2. パスパラメータ（`high_school_id`・`id`）とRequest DTOをバインドし、型・必須・フォーマットを検証する
3. current adminが`high_school_id`を管理対象としているかを確認する
4. `HighSchoolExistenceChecker.Exists`で対象高校の存在を確認する。存在しない場合は`ErrHighSchoolNotFound`
5. `TeacherStore.FindByIDForHighSchool`で対象教員を取得する。存在しない、または対象高校に所属しない場合は`ErrTeacherNotFound`（②「12. Validation設計」整合性チェック：更新対象の教員が対象高校に所属しているかどうか）
6. プロフィール項目（name・email）が指定されている場合、`Teacher.ApplyProfile`で差し替え・検証する
7. 権限項目（grade_scope・manage_other_teachers）が指定されている場合、既存`TeacherPermission`を`TeacherPermissionStore.FindByUserID`で取得し、`TeacherPermission.Apply`で差し替え・検証する。`grade_scope`が許容範囲外の場合は`ErrInvalidGradeScope`
8. `grade_ids`が指定されている場合、`teacher_management.NewGradeAssignmentSet`で重複排除した集合を作成し、`GradeReferenceChecker.ExistingGradeIDsForHighSchool`で対象高校に属する学年IDを取得する
9. `teacher_management.ValidateGradeAssignment`で指定学年がすべて対象高校に属するか判定する。属さない学年が含まれる場合は`ErrGradeNotInHighSchool`（②「8. Domain Service」TeacherGradeAssignmentPolicy、②「12. Validation設計」業務ルール：指定学年が対象高校に属するかどうか）
10. `TeacherStore.UpdateWithPermissionAndGrades`をTeacher・TeacherPermission・（`grade_ids`指定時のみ）GradeAssignmentSetで呼び出す（②「10. UseCase設計」UpdateTeacher 呼び出すStore：HighSchoolStore→TeacherStore→TeacherPermissionStore→TeacherGradeStore→GradeStoreの順に対応）
11. 更新結果をResponse DTOへ変換して返す

## Request / Response DTO

### Request（`internal/teacher_management/presentation/request/teacher_request.go`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`InviteTeacherRequest`|`Name string`、`Email string`、`GradeScope *string`、`ManageOtherTeachers *bool`|`Name`・`Email`は`binding:"required"`、`Email`は`binding:"email"`（②「12. Validation設計」必須チェック：作成時のemail、フォーマットチェック：メールアドレス形式）。`GradeScope`・`ManageOtherTeachers`は省略時、Model側でデフォルト値を判断する（デフォルト値の具体的な内容は②に明記がなく「14. ②からの補足事項」参照）|
|`UpdateTeacherRequest`|`Name *string`、`Email *string`、`GradeScope *string`、`ManageOtherTeachers *bool`、`GradeIDs *[]uint`|部分更新のためポインタ型で「未指定」を表現する。`Email`指定時は`binding:"omitempty,email"`。`GradeIDs`は配列形式チェック（②「12. Validation設計」フォーマットチェック：grade_idsの配列形式）|

### Response（`internal/teacher_management/presentation/response/teacher_response.go`）

|struct名|フィールドと型|
|-|-|
|`TeacherResponse`|`ID uint`、`HighSchoolID uint`、`Name string`、`Email string`、`GradeScope string`、`ManageOtherTeachers bool`、`GradeIDs []uint`、`CreatedAt string`、`UpdatedAt string`|
|`TeacherListResponse`|`Teachers []TeacherResponse`|

EntityであるTeacher／TeacherPermission／TeacherGradeAssignmentをそのまま返さず、必ずResponse DTOへ変換する（規約「7. データフロー」）。

## Routing

`internal/teacher_management/presentation/routes.go`

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/admin/high_schools/:high_school_id/teachers|TeacherHandler.ListTeachers|
|POST|/api/v1/admin/high_schools/:high_school_id/teachers|TeacherHandler.InviteTeacher|
|PATCH|/api/v1/admin/high_schools/:high_school_id/teachers/:id|TeacherHandler.UpdateTeacher|

いずれのルートも認証Middleware（本人確認）・認可Middleware（admin roleチェック）を経由する（②「13. Authorization設計」Middleware、「10. Authorization実装方針」参照）。

---

# 7. API仕様

②「16. API互換方針」に基づき、Rails現行仕様と同一のエンドポイントを維持する。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/admin/high_schools/:high_school_id/teachers|ListTeachers|パスパラメータ`high_school_id`|`TeacherListResponse`|200|
|POST|/api/v1/admin/high_schools/:high_school_id/teachers|InviteTeacher|パスパラメータ`high_school_id` + `InviteTeacherRequest`|`TeacherResponse`|201（②「16. API互換方針」Status Code：作成成功の扱いは既存仕様に合わせて統一する、とあり具体的な使い分けは①未提供のため参照不可。REST慣例に基づき作成=201と推測。「14. ②からの補足事項」参照）|
|PATCH|/api/v1/admin/high_schools/:high_school_id/teachers/:id|UpdateTeacher|パスパラメータ`high_school_id`・`id` + `UpdateTeacherRequest`|`TeacherResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（Middleware）|
|admin以外のロール|403|認可エラー（Middleware）|
|current adminが管理しない`high_school_id`を指定|403|業務認可エラー（Handler。②「1. 機能概要」利用者の制約に基づく。判定方法は推測。「14. ②からの補足事項」参照）|
|Request DTOの型・必須・フォーマット不正|422|Presentation Validationエラー（②「16. API互換方針」422: 入力・業務ルール違反）|
|`grade_scope`が許容範囲外（`ErrInvalidGradeScope`）|422|業務ルール違反|
|指定`grade_ids`に対象高校に属さない学年が含まれる（`ErrGradeNotInHighSchool`）|422|業務ルール違反（②「14. Error設計」Domain Error：他校の学年を担当学年として指定した場合）|
|指定`high_school_id`が存在しない（`ErrHighSchoolNotFound`）|404|②「16. API互換方針」404: 対象高校不存在|
|対象教員不存在／所属高校不一致（`ErrTeacherNotFound`）|404|②「16. API互換方針」404: 対象教員不存在|
|DB接続失敗等のInfrastructure Error|500|内部エラー|

---

# 8. Transaction実装方針

②「11. Transaction設計」を実装単位に落とし込む。

## Transaction開始箇所

- Active Record採用のため、`TeacherStore.CreateWithPermission`・`TeacherStore.UpdateWithPermissionAndGrades`メソッド内で`db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })`を用いてトランザクションを開始する（②「11. Transaction設計」Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する）
- `ListTeachers`はトランザクションを使用しない（②「11. Transaction設計」ListTeachersの処理ではトランザクションを使用しない）

## Transaction終了箇所（Commit / Rollback条件）

- `CreateWithPermission`: Teacher作成と初期TeacherPermission作成が完了した時点でコミットする（②「11. Transaction設計」InviteTeacherの処理では、教員アカウントと権限レコードの作成が完了した時点でコミットする）。いずれかが失敗した場合はロールバックする
- `UpdateWithPermissionAndGrades`: プロフィール更新・権限更新・（指定時のみ）担当学年の全置換がすべて成功した時点でコミットする（②「11. Transaction設計」UpdateTeacherの処理では、プロフィール・権限・担当学年の同期が完了した時点でコミットする）。いずれかが失敗した場合はロールバックする

## 複数Storeにまたがる場合の扱い

`TeacherStore.CreateWithPermission`／`UpdateWithPermissionAndGrades`は、トランザクション用の`tx *gorm.DB`を用いて`TeacherPermissionStore`・`TeacherGradeStore`を`NewTeacherPermissionStore(tx)`・`NewTeacherGradeStore(tx)`で生成し、同一トランザクション内で各Storeのメソッドを呼び出す。`HighSchoolExistenceChecker`・`GradeReferenceChecker`による確認は、トランザクション開始前（Handler側）で完了させ、トランザクション内では行わない（②「9. Repository設計」HighSchoolStore/GradeStoreは「担当学年として妥当かどうかの最終判断」を保持しないとされている点と整合）。

---

# 9. Validation実装方針

②「12. Validation設計」を実装レベルに落とし込む。

## Presentation

- 型チェック: HTTP入力の型を検証する
- 必須チェック: `InviteTeacherRequest`の`Name`・`Email`に`binding:"required"`（②「12. Validation設計」必須チェック：作成時のemail、高校IDの必須性）
- フォーマットチェック: `Email`は`binding:"email"`、`GradeIDs`は配列形式チェック（②「12. Validation設計」フォーマットチェック：メールアドレス形式、grade_idsの配列形式）

## 業務ルール検証（Active Record採用時: Modelのメソッドで検証する内容）

- `Teacher.Validate()` / `Teacher.ApplyProfile()`: name・email等必須項目の妥当性
- `TeacherPermission.Validate()` / `TeacherPermission.Apply()`: grade_scopeの許容範囲チェック
- `GradeAssignmentSet`: 重複ID排除

## 業務ルール検証（Model単体では完結しないため、Handlerで実行する内容）

指定学年が対象高校に属するかどうかの確認（`GradeReferenceChecker` + `ValidateGradeAssignment`）、更新対象の教員が対象高校に所属しているかの確認（`TeacherStore.FindByIDForHighSchool`）は、他Contextのデータ参照または横断的なスコープ判定であり、Teacher Model単体では検証できないため、Handlerが呼び出し元となって検証する（②「12. Validation設計」業務ルール・整合性チェックの実装先を、Active Record構造に合わせて具体化したもの。**②からの補足**。詳細は「14. ②からの補足事項」参照）。

②「12. Validation設計」の「状態チェック: 現行仕様には教員の状態遷移がないため、更新可能かどうかの状態チェックは行わない」の判断は変更しない。本機能では状態遷移に関するValidationを実装しない。

## 責務分離

②の方針どおり、Presentationは「入力が正しいか」を、Domain（Model）／Handlerは「業務的に妥当か（自校の学年か、自校の教員か）」を担当する。

---

# 10. Authorization実装方針

②「13. Authorization設計」を実装レベルに落とし込む。

## Middleware

- JWT等の検証を行い、current userをcontextに格納する（規約「8. 横断的関心事の置き場所」認証）
- ロールがadminであることを確認する（②「13. Authorization設計」Middleware）

## Handler

- ルーティング層でAPIの入口を担当し、パスパラメータ（`high_school_id`等）を受け渡す。個別の業務権限判定はMiddlewareが担うシステムレベルの認可には持たせない（②「13. Authorization設計」Handler）
- ただし、Active Record採用によりUseCase層がないため、②「13. Authorization設計」UseCaseの記載（対象教員が指定高校に所属するかどうかの判断、学年が対象高校に属するかの確認の呼び出し）はHandlerが担う
- current adminが対象`high_school_id`を管理しているかどうかの確認もHandlerで行う（具体的な判定方法は②に明記がなく、推測を含む。「14. ②からの補足事項」参照）

## Store／Model

- `TeacherStore`の検索・更新メソッドは、渡された`high_school_id`によるスコープを常に条件へ含める（②「13. Authorization設計」Domain：教員・権限・担当学年の整合性ルールを保持する、を実装レベルで反映）
- Model（`Teacher`）は`BelongsToHighSchool`メソッドを提供するのみで、認可の主体としては扱わない

## 判断理由

②の判断（システムレベルの認可＝ロール確認と、業務レベルの認可＝自校スコープ確認を分離する）を変更しない。

---

# 11. Error実装方針

②「14. Error設計」を実装レベルに落とし込む。

## Domain Error → Application Errorへの変換方針

Active Record採用のためDomain Error層は独立させず、「3. Domain層設計」の`errors.go`で定義したsentinel error（`ErrTeacherNotFound`等）をそのままApplication Error相当として扱う（規約8章 横断的関心事の置き場所「Transaction Script/Active Record採用時はDomain Errorに相当する層がないため、関数・struct側で発生したエラーをApplication Error相当として扱う」）。

## Application Error → HTTPレスポンスへの変換方針

Handlerが`errors.Is`でsentinel errorを判定し、対応するHTTP Status Codeへ変換する。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrHighSchoolNotFound`|Handler（HighSchoolExistenceChecker経由）|404|
|`ErrTeacherNotFound`|Store（TeacherStore）|404|
|`ErrGradeNotInHighSchool`|Handler（ValidateGradeAssignment経由）|422|
|`ErrInvalidGradeScope`|Model（TeacherPermission.Validate / Apply）|422|
|Request DTOバインド／バリデーションエラー|Presentation|422|
|current adminが対象高校を管理していない|Handler|403|
|未認証|Middleware|401|
|ロール不一致|Middleware|403|
|上記以外（DB接続失敗等）|Store／Infrastructure|500|

## Infrastructure Errorのハンドリング方針

GORMが返すDB接続エラー等は、sentinel errorとして特別扱いせず、Storeからそのまま呼び出し元へ返し、Handlerでハンドリングされなかった場合は共通のエラーハンドリングMiddleware（本機能固有の設計ではないため対象外。`shared/`側の既存実装に従う）で500として応答する（②「14. Error設計」Infrastructure Error：DB接続失敗・永続化失敗を表現する）。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」により、既存Rails DBをそのまま継続利用し、Schema変更は行わない。

## 利用するGORMモデルとテーブルの対応

|Goモデル|テーブル名|備考|
|-|-|-|
|`Teacher`|`users`|struct名のデフォルト複数形は`teachers`だが、②「17. DB設計方針」の記載どおり実際は`users`テーブルの教員としてのレコードであるため、`Tabler`インターフェース（`func (Teacher) TableName() string { return "users" }`）による明示的な上書きが必要（Gorm規約「テーブル名」。**②からの補足**：②はテーブル名を明記しているのみで、GoモデルとGORM命名規則の不一致には触れていないため、③側で気づいて反映した）|
|`TeacherPermission`|`teacher_permissions`|GORMのデフォルト命名規則（struct名の複数形snake_case）で一致するため、`TableName()`のオーバーライドは不要（Gorm規約「複数形のテーブル名」）|
|`TeacherGradeAssignment`|`teacher_grades`|struct名のデフォルト複数形は`teacher_grade_assignments`だが、②「1. 機能概要」「17. DB設計方針」の記載どおり実際のテーブル名は`teacher_grades`であるため、`Tabler`インターフェース（`func (TeacherGradeAssignment) TableName() string { return "teacher_grades" }`）による明示的な上書きが必要（Gorm規約「テーブル名」。**②からの補足**）|

## 主要クエリの条件・ソート・ページネーション方針

- `TeacherStore.FindAllByHighSchool`: `high_school_id`一致必須（②「9. Repository設計」保持する検索機能）。ソート・ページネーションの具体的な方針は②に明記がなく、①未提供のため参照不可（「14. ②からの補足事項」参照）
- `TeacherStore.FindByIDForHighSchool`: `id`かつ`high_school_id`一致
- `TeacherPermissionStore.FindByUserID`: `user_id`一致
- `TeacherGradeStore.FindGradeIDsByUserID`: `user_id`一致
- `TeacherGradeStore.ReplaceAll`: `user_id`一致条件での削除、続けて指定`grade_ids`分の一括作成

SQL文そのものは本書に記載しない。

## 既存Schemaに対する変更

②「17. DB設計方針」により変更なし。`users` / `teacher_permissions` / `teacher_grades`の既存カラム構成をそのまま利用する。追加スキーマは不要。

---

# 13. テストケース設計

②「18. テスト戦略」を、Active Record採用時の区分（規約4章／指示書「Active Record採用時: 『Domain Test』→『Model Test』、『UseCase Test』は『対象外』、『Repository Test』→『Store Test』」）に読み替えて具体化する。

## Model Test（②「Domain Test」からの読み替え）

|対象|テストケース|
|-|-|
|`Teacher.Validate` / `NewTeacher`|name・email等必須項目の欠落を検出すること|
|`TeacherPermission.Validate` / `NewTeacherPermission`|`grade_scope`が許容範囲外の場合にエラーとなること|
|`GradeAssignmentSet`（`NewGradeAssignmentSet`）|重複する`grade_id`が排除されること、空配列で担当学年なしとして扱われること|
|`ValidateGradeAssignment`|要求学年がすべて対象高校に属する場合に`ok=true`となること、属さない学年が含まれる場合に該当IDが`invalidGradeIDs`として返ること（②「18. テスト戦略」Domain Test：TeacherGradeAssignmentPolicyによる学年所属判定、GradeAssignmentSetの重複排除ルールを検証する）|

## UseCase Test

対象外（Active Record採用のため、usecase層を設けない）。

## Store Test（②「Repository Test」からの読み替え）

|対象|テストケース|
|-|-|
|`TeacherStore.FindAllByHighSchool`|`high_school_id`一致の教員のみが取得されること|
|`TeacherStore.FindByIDForHighSchool`|所属高校が異なる教員は取得できず`ErrTeacherNotFound`となること|
|`TeacherStore.CreateWithPermission`|Teacher作成とTeacherPermission作成が同一トランザクションで成功すること|
|`TeacherStore.CreateWithPermission`|TeacherPermission作成が失敗した場合にTeacher作成もロールバックされること|
|`TeacherStore.UpdateWithPermissionAndGrades`|`grade_ids`指定時に既存担当学年が全置換されること|
|`TeacherStore.UpdateWithPermissionAndGrades`|`grade_ids`未指定時に既存担当学年が変更されないこと|
|`TeacherGradeStore.ReplaceAll`|既存紐付けの削除と新規紐付けの作成が一貫して行われること|

## Handler Test

|対象|テストケース|
|-|-|
|`ListTeachers`|存在しない`high_school_id`で404が返ること|
|`ListTeachers`|current adminが管理しない`high_school_id`で403が返ること|
|`InviteTeacher`|`email`欠落・不正フォーマットで422が返ること|
|`InviteTeacher`|`grade_scope`が許容範囲外の場合に422が返ること|
|`UpdateTeacher`|存在しない教員idで404が返ること|
|`UpdateTeacher`|`grade_ids`に他校の学年が含まれる場合に422が返ること|
|全Handler|未認証・非admin roleでのアクセスが401/403となること|

## Integration Test

|対象|テストケース|
|-|-|
|教員招待〜一覧取得〜更新|一連のエンドポイント呼び出しが正常に完了し、権限・担当学年の反映結果が一覧・更新レスポンスに一貫して反映されること（②「18. テスト戦略」Integration Test：エンドポイント経由で一覧・招待・更新が正常に動作し、他校の学年が拒否されることを確認する）|
|他校の学年指定|`grade_ids`に他校の学年を指定した更新が拒否されること（②「18. テスト戦略」Integration Test）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測か否か|
|-|-|-|
|Bounded Context `teacher-management` の内部ディレクトリ名を`internal/teacher_management`とした|②はContext名（teacher-management）のみを記載し、内部ディレクトリ名を明記していない。`規約/アーキテクチャ規約.md`「9. 命名規約」の「Context名とディレクトリ名が一致しない場合は②文書内で対応関係を明記する」に該当する明記がないため、既存の他機能（student-directory→internal/student_directory）の対応方針にならい、Context名のハイフンをアンダースコアに置き換えたディレクトリ名を採用した|推測（既存の他②③文書での対応付けからの類推であり、②文書内に明示的な対応付けの記載はない）|
|②「9. Repository設計」HighSchoolStore・GradeStoreをteacher-management内で実装せず、teacher_management package側に参照用interface（`HighSchoolExistenceChecker`／`GradeReferenceChecker`）を定義し、実装はHighSchool Context・Grade Context側に委ねる構成とした|②「9. Repository設計」はHighSchoolStore・GradeStoreをteacher-managementの節内で記載しているが、`規約/アーキテクチャ規約.md`「6. Context間連携ルール」により他Contextの内部実装に直接依存できないため、コーディング規約「5. インターフェース」（利用側での定義）に従って整理した。②の「責務」「保持しない責務」の記載内容自体は変更していない|補足（規約の適用による構造上の具体化であり、推測ではなく規約遵守のための判断）|
|②「8. Domain Service」TeacherGradeAssignmentPolicyを、独立したstruct/interfaceではなく`internal/teacher_management`パッケージ内のpackageレベル関数`ValidateGradeAssignment`として実装した|`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」Active Record節により、Domain Serviceに相当する独立層は原則「対象外」とされているため。判定内容そのものは②の記載を変更していない|補足（規約適用）|
|②「7. Value Object設計」TeacherPermission・GradeAssignmentSetを、独立層としてではなくstruct・メソッドとして`internal/teacher_management`パッケージ内に実装した|同上、規約4章Active Record節により、Value Objectは原則「対象外」とされているため|補足（規約適用）|
|`Teacher`のGORMテーブル名を`users`として`TableName()`で明示的に上書きする必要がある点、および`TeacherGradeAssignment`のGORMテーブル名を`teacher_grades`として同様に上書きする必要がある点|②「17. DB設計方針」はテーブル名`users`・`teacher_grades`を利用する旨のみ記載しており、Go構造体名のデフォルト複数形との不一致には触れていない。Gorm規約「テーブル名」に基づき③側で明示した|補足（Gorm規約の適用によりテーブル不一致を検出したもの）|
|`Teacher`structが`users`テーブルの一部カラム（教員管理に必要な最小限のフィールド）のみを保持し、パスワードハッシュ・ロール等他Contextが管理すると想定されるカラムを含めていない点|②はUserテーブルの全カラム構成を記載しておらず、①も未提供のため、`users`テーブルの完全な定義を参照できない。教員管理の業務範囲（②「6. Entity設計」Teacher）に必要な範囲のみをフィールド化した|推測（実装着手前に`users`テーブルの完全なカラム定義・NOT NULL制約等の確認が必要）|
|`grade_scope`の具体的な許容値集合（例："own_grade"/"all_grades"等）|②「7. Value Object設計」は「許容される範囲値のみを受け付ける」とのみ記載し、具体的な値集合を明記していない。①未提供のため参照不可|推測（実装着手前に要件確認が必要）|
|current adminが対象`high_school_id`を管理しているかどうかの具体的な確認方法（current adminのHighSchoolIDとpathパラメータの一致確認と仮定した）|②「1. 機能概要」利用者は「対象は自身が管理する高校に所属する教員に限定される」と記載するのみで、管理者と高校の対応関係の具体的なデータモデル・確認方法を明記していない。①も未提供のため参照不可。教員同様adminユーザーもHighSchoolIDを保持するという前提で設計した|推測（実装着手前にadminユーザーと高校の対応関係の確認が必要）|
|`InviteTeacherRequest`の`GradeScope`・`ManageOtherTeachers`省略時のデフォルト値の具体的な内容|②「10. UseCase設計」InviteTeacherは初期権限レコード作成を行うとのみ記載し、招待時に権限項目が必須か省略可かを明記していない。①未提供のため参照不可|推測（実装着手前に要件確認が必要）|
|作成時（InviteTeacher）のHTTP Status Codeを201とした|②「16. API互換方針」Status Codeの記載が「201/200: 作成成功の扱いは既存仕様に合わせて統一する」と曖昧であり、①未提供のため実際のRails挙動を参照できない。REST慣例に基づき作成=201と仮置きした|推測（実装着手前にRails現行仕様の確認を推奨）|
|`TeacherStore.CreateWithPermission`／`UpdateWithPermissionAndGrades`というメソッド名・トランザクションの起点をStoreの単一メソッドに集約する具体的な設計|②「11. Transaction設計」は「Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する」という方針のみを記載しており、具体的なメソッド名・分割単位は指定していない|補足（②の方針を実装可能な粒度に具体化したもの）|
|`TeacherStore.FindAllByHighSchool`のソート順・ページネーション方針を明記しなかった|②「9. Repository設計」TeacherStoreの「保持する検索機能」は`high_school_id`による絞り込みのみを記載し、ソート順・ページネーションの要否に言及していない。①未提供のため参照不可|推測（実装着手前に要件確認が必要。教員一覧は少数想定でページネーション不要の可能性もある）|

---
