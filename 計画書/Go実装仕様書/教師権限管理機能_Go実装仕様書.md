# 教師権限管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

教師が同校教員の権限（学年閲覧範囲 `grade_scope`、他教員管理権限 `manage_other_teachers`）を確認・更新できる機能である。更新操作は `manage_other_teachers` 権限を持つ教師のみが行え、さらに「自分自身の権限は更新できない」「校内で最後の有効教員の権限は更新できない」という組織運用上の保護ルールが課される（②「1. 機能概要」）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により、本機能は **Domain Model** を採用する。

- 更新操作に「更新者が `manage_other_teachers` 権限を持つか」「更新者が自分自身を更新していないか」「更新対象が校内で最後の有効教員でないか」という業務上の不変条件が課されており、単純なCRUD検証を超えている
- 特に「最後の有効教員保護」は、対象教員1件の属性だけでは判定できず、同校に所属する他の有効教員の集合を踏まえて初めて判定できる、複数エンティティにまたがる業務ルールである
- 保護ルールをUseCase内の条件分岐として散在させず、Entity + Domain Service（`TeacherPermissionUpdateGuard`）に集約することで、ルールの見落としを防ぎ、保守性・テスト容易性を高める

Transaction Script・Active Record・Event Sourcingは、②「4. 設計パターン」の「採用しなかったパターン」節の理由により不採用とされている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- `GET /api/v1/teacher/permissions`（同校教員の権限一覧取得）
- `GET /api/v1/teacher/permissions/:id`（指定教員の権限詳細取得）
- `PATCH /api/v1/teacher/permissions/:id`（指定教員の権限更新）

の3エンドポイントの実装に必要な、Domain層（Entity・Value Object・Repository Interface・Domain Service・Domain Error）、Application層（UseCase・DTO）、Infrastructure層（Repository実装）、Presentation層（Handler・Request/Response・Routing）の実装単位を規定する。①Rails実装（Controller・Form・Query・Serializerの実装詳細）は本書作成時点で未提供のため参照不可であり、該当箇所は②の記載のみを根拠とする。

規約`アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」の「Domain Model」節に従い、`{context}/domain`・`application`・`infrastructure`・`presentation`のフルレイヤー構成を適用する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名（②「3. Bounded Context」）: `teacher-permission`
- ディレクトリ名: `internal/teacher_permission`

> **②からの補足**: ②にはディレクトリ名の明記がない。アーキテクチャ規約「9. 命名規約」（Context名はkebab-case、`internal/`配下は英単語1語または短いスネークケース）に従い、`teacher-permission`を`internal/teacher_permission`に変換した（推測）。

## ②で採用した設計パターン

Domain Model

## 作成するディレクトリ一覧

```
internal/teacher_permission/
├── domain/
│   ├── entity/
│   ├── valueobject/
│   ├── repository/
│   ├── service/
│   └── errors/
├── application/
│   ├── dto/
│   └── usecase/
├── infrastructure/
│   ├── persistence/
│   │   └── gorm/
│   └── repository/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

`domain/specification/`・`domain/event/`・`infrastructure/mail/`・`infrastructure/cache/`・`infrastructure/queue/`は本機能では対象外とする。②「15. Domain Event」でDomain Eventは現時点で不採用と明記されており、Mail・Cache・Queue等の外部連携要件も②に記載がないため。

## 作成するファイル一覧

```
internal/teacher_permission/domain/entity/teacher_permission.go

internal/teacher_permission/domain/valueobject/grade_scope.go
internal/teacher_permission/domain/valueobject/manage_other_teachers_flag.go

internal/teacher_permission/domain/repository/teacher_permission_repository.go
internal/teacher_permission/domain/repository/active_teacher_count_repository.go

internal/teacher_permission/domain/service/teacher_permission_update_guard.go

internal/teacher_permission/domain/errors/errors.go

internal/teacher_permission/application/dto/list_teacher_permissions_dto.go
internal/teacher_permission/application/dto/show_teacher_permission_dto.go
internal/teacher_permission/application/dto/update_teacher_permission_dto.go

internal/teacher_permission/application/usecase/list_teacher_permissions_usecase.go
internal/teacher_permission/application/usecase/show_teacher_permission_usecase.go
internal/teacher_permission/application/usecase/update_teacher_permission_usecase.go

internal/teacher_permission/infrastructure/persistence/gorm/teacher_permission_model.go
internal/teacher_permission/infrastructure/repository/teacher_permission_repository.go
internal/teacher_permission/infrastructure/repository/active_teacher_count_repository.go

internal/teacher_permission/presentation/handler/teacher_permission_handler.go
internal/teacher_permission/presentation/request/teacher_permission_request.go
internal/teacher_permission/presentation/response/teacher_permission_response.go
internal/teacher_permission/presentation/routes.go
```

---

# 3. Domain層設計

## Entity

### TeacherPermission（`domain/entity/teacher_permission.go`）

- struct名: `TeacherPermission`
- フィールド:

|フィールド|型|意味|
|-|-|-|
|`ID`|`uint`|権限レコード自体の識別子（`teacher_permissions.id`相当）|
|`TeacherID`|`uint`|権限の帰属先である教員のID。Teacher Directory Contextが管理する教員アカウントへの参照（②7章「Value Objectを採用しないもの」）|
|`HighSchoolID`|`uint`|所属校ID。権限更新のスコープ判定（同校であるか）に用いる|
|`GradeScope`|`valueobject.GradeScope`|学年閲覧範囲|
|`ManageOtherTeachers`|`valueobject.ManageOtherTeachersFlag`|他教員管理権限|

- 公開メソッド一覧（引数・戻り値のみ。ロジックは記述しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTeacherPermission`|`(id uint, teacherID uint, highSchoolID uint, gradeScope valueobject.GradeScope, manageOtherTeachers valueobject.ManageOtherTeachersFlag)`|`(*TeacherPermission, error)`|不変条件を満たしたTeacherPermissionを生成するファクトリ|
|`UpdateGradeScope`|`(newScope valueobject.GradeScope)`|なし|`grade_scope`を更新する|
|`UpdateManageOtherTeachers`|`(newFlag valueobject.ManageOtherTeachersFlag)`|なし|`manage_other_teachers`を更新する|
|`CanManageOtherTeachers`|`()`|`bool`|保有する権限で他教員を管理できるかを判定する（②6章「保有する権限で『他教員を管理できるか』を自ら判定できるようにする」）|

- 不変条件（ファクトリで保証する内容）:
  - `GradeScope`は`valueobject.GradeScope`型としてのみ保持され、生成時に許容値検証済みであることが保証される
  - `ManageOtherTeachers`は`valueobject.ManageOtherTeachersFlag`型としてのみ保持される
  - `TeacherID`は0を許容しない

> **②からの補足**: `UpdateGradeScope` / `UpdateManageOtherTeachers`は、引数として渡すValue Objectの生成時点（`valueobject.NewGradeScope`等）で妥当性検証が完了している前提のため、戻り値を`error`とせず`void`とした。②「6. Entity設計」に更新メソッドのシグネチャそのものの明記はなく、実装上の判断である（推測）。

## Value Object

### GradeScope（`domain/valueobject/grade_scope.go`）

- struct名: `GradeScope`
- フィールド: `value string`（非公開）
- 生成時に検証するルール: `own_grade` / `all_grades` の定義済み値以外を許容しない（②7章）
- 公開メソッド: `NewGradeScope(raw string) (GradeScope, error)` / `(g GradeScope) String() string` / `(g GradeScope) IsAllGrades() bool`

### ManageOtherTeachersFlag（`domain/valueobject/manage_other_teachers_flag.go`）

- struct名: `ManageOtherTeachersFlag`
- フィールド: `value bool`（非公開）
- 生成時に検証するルール: true/false以外の値を許容しない（②7章）
- 公開メソッド: `NewManageOtherTeachersFlag(raw bool) ManageOtherTeachersFlag` / `(f ManageOtherTeachersFlag) CanManage() bool`

> **②からの補足**: `NewManageOtherTeachersFlag`はGoの`bool`型自体が true/false 以外の値を取り得ないため、実質的にエラーを返す必要がない。②7章の「独自ルール: true/false以外の値を許容しない」はRailsの動的型言語を前提とした記述と考えられ、Go実装では型システムによって自動的に満たされる。そのためコンストラクタはエラーを返さない形にした（②の設計判断自体は変更せず、Go言語の型特性に合わせた実装上の判断であることを明示する）。

## Value Objectを採用しないもの

②7章の方針どおり、対象教員のID・氏名はTeacherPermissionが直接保持せず、`TeacherID`（Teacher Directory Context管理の教員アカウントへの参照）としてのみ保持する。氏名・かな氏名はApplication層のDTOで、Repositoryが返す付帯情報として扱う（4章・5章参照）。

## Repository Interface

### TeacherPermissionRepository（`domain/repository/teacher_permission_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`ListBySchool`|`(ctx context.Context, highSchoolID uint, page int)`|`(records []TeacherPermissionRecord, totalCount int, err error)`|同校教員の権限一覧取得（氏名カナ順、ページネーション）|
|`FindByTeacherID`|`(ctx context.Context, highSchoolID uint, teacherID uint)`|`(*TeacherPermissionRecord, error)`|指定教員（同校スコープ）の権限詳細取得|
|`Update`|`(ctx context.Context, permission *entity.TeacherPermission)`|`error`|権限の更新|

`TeacherPermissionRecord`（同ファイル内で定義する、Entity + 付帯情報の複合型）:

|フィールド|型|備考|
|-|-|-|
|`Permission`|`*entity.TeacherPermission`|権限Entity本体|
|`TeacherName`|`string`|対象教員の氏名（一覧・詳細表示用）|
|`TeacherNameKana`|`string`|対象教員のかな氏名（一覧・詳細表示用、ソートキー）|

> **②からの補足**: ②9章のTeacherPermissionRepositoryは「氏名カナ順のソート」を検索機能として保持するとしているが、Entity自体（②6章）は氏名を保持しない設計である。そのため一覧・詳細表示に必要な氏名情報を、Entityとは別にRepositoryの戻り値（`TeacherPermissionRecord`）として付帯させる構成とした。これは②のEntity設計・Repository設計の記載を矛盾なく実装に落とし込むための判断であり、新しい業務ルールの追加ではない（推測ではなく、②の記載同士の整合を取るための実装判断）。

### ActiveTeacherCountRepository（`domain/repository/active_teacher_count_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`ExistsOtherActiveTeacher`|`(ctx context.Context, highSchoolID uint, excludeTeacherID uint)`|`(bool, error)`|指定教員を除いた、校内の有効教員が他に存在するかを確認する|

いずれのRepositoryも、保護ルールの判断そのもの（更新可否）は行わない。判断は`TeacherPermissionUpdateGuard`（Domain Service）の責務とする（②9章「保持しない責務」）。

## Domain Service

### TeacherPermissionUpdateGuard（`domain/service/teacher_permission_update_guard.go`）

- struct名: `TeacherPermissionUpdateGuard`
- 依存: なし（Repositoryに依存しない。判定に必要な事実は呼び出し元UseCaseが取得し、引数として渡す）
- メソッドシグネチャ: `(g TeacherPermissionUpdateGuard) Authorize(updater *entity.TeacherPermission, target *entity.TeacherPermission, hasOtherActiveTeacher bool) error`
- 責務: 以下の順に保護ルールを判定する（②8章）
  - `updater.CanManageOtherTeachers()`が`false`の場合、権限不足として扱う
  - `updater.TeacherID == target.TeacherID`の場合、自己更新として扱う
  - `hasOtherActiveTeacher`が`false`の場合、最後の有効教員保護として扱う
  - 上記いずれにも該当しない場合のみ`nil`を返す

> **②からの補足**: `Authorize`のシグネチャが`ActiveTeacherCountRepository`を直接受け取らず`hasOtherActiveTeacher bool`という判定済みの事実を引数で受け取る構成としたのは、②8章「テスト容易性: 保護ルールを独立したDomain Serviceとして切り出すことで、DBや他の全業務フローを経由せずにルール単体を検証できる」を満たすための実装判断である（推測）。事実確認（`ActiveTeacherCountRepository`の呼び出し）はUseCase側の責務とし（4章参照）、Domain ServiceはRepository Interfaceにも依存しない純粋な判定ロジックとする。これによりDomainがInfrastructureは元より、Repository Interfaceの呼び出しコストにも依存しない構成となる。

## 追加で必要としないService

②8章のとおり、`grade_scope` / `manage_other_teachers`の値そのものの妥当性検証はValue Objectの責務に閉じており、別途Domain Serviceを設けない。

## Domain Error

`domain/errors/errors.go`に、`errors.New`によるセンチネルエラー変数として定義する（②「14. Error設計」のDomain Errorに対応）。

|変数名|発生条件|
|-|-|
|`ErrPermissionDenied`|更新者が`manage_other_teachers`権限を持たない|
|`ErrSelfUpdateNotAllowed`|更新者が更新対象本人である（自己更新禁止）|
|`ErrLastActiveTeacherProtected`|更新対象が校内で最後の有効教員である|
|`ErrInvalidGradeScope`|`grade_scope`が許容値（`own_grade` / `all_grades`）でない|

---

# 4. Application層設計

## DTO（Command / Query）

|struct名|フィールド|型|区分|
|-|-|-|-|
|`RequestingTeacher`|`TeacherID`|`uint`|current teacherの識別子（Command/Query共通の入力要素）|
||`HighSchoolID`|`uint`|所属校ID|
|`ListTeacherPermissionsQuery`|`RequestingTeacher`|`RequestingTeacher`|Query|
||`Page`|`int`|Query|
|`ListTeacherPermissionsResult`|`RequestingTeacher`|`RequestingTeacher`|出力|
||`Permissions`|`[]TeacherPermissionSummary`|出力|
||`CurrentPage`|`int`|出力|
||`TotalPages`|`int`|出力|
||`TotalCount`|`int`|出力|
||`PerPage`|`int`|出力|
|`ShowTeacherPermissionQuery`|`RequestingTeacher`|`RequestingTeacher`|Query|
||`TeacherID`|`uint`|Query|
|`ShowTeacherPermissionResult`|`Permission`|`TeacherPermissionSummary`|出力|
|`UpdateTeacherPermissionCommand`|`RequestingTeacher`|`RequestingTeacher`|Command|
||`TeacherID`|`uint`|Command|
||`GradeScope`|`string`|Command|
||`ManageOtherTeachers`|`bool`|Command|
|`UpdateTeacherPermissionResult`|`Permission`|`TeacherPermissionSummary`|出力|
|`TeacherPermissionSummary`|`TeacherID`|`uint`|出力共通型|
||`TeacherName`|`string`|出力共通型|
||`TeacherNameKana`|`string`|出力共通型|
||`GradeScope`|`string`|出力共通型|
||`ManageOtherTeachers`|`bool`|出力共通型|

> **②からの補足**: `RequestingTeacher`の具体的なフィールド構成（`TeacherID`・`HighSchoolID`）は②13章「UseCase」の記載（「current userの所属校IDを用いて、権限一覧・詳細の取得範囲を同校にスコープする」）から導出したものであり、型・フィールド名自体は②に明記がない（推測）。current teacher情報自体の取得元（認証Middleware・shared/authパッケージ等）は本機能のスコープ外とする。

## UseCase

### ListTeacherPermissionsUseCase（`application/usecase/list_teacher_permissions_usecase.go`）

- struct名: `ListTeacherPermissionsUseCase`
- コンストラクタが受け取る依存: `repository.TeacherPermissionRepository`
- 公開メソッド: `(u *ListTeacherPermissionsUseCase) Execute(ctx context.Context, query dto.ListTeacherPermissionsQuery) (dto.ListTeacherPermissionsResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `TeacherPermissionRepository.ListBySchool(ctx, query.RequestingTeacher.HighSchoolID, query.Page)`を呼び出す
  2. 取得した`TeacherPermissionRecord`群を`TeacherPermissionSummary`に変換する
  3. 総件数からページ情報（`TotalPages`・`PerPage`）を組み立てる
  4. `ListTeacherPermissionsResult`を構築して返す
- トランザクション境界: なし（読み取りのみ、②11章）
- 発生しうるApplication Error: なし（該当0件の場合も正常系として空一覧を返す）

### ShowTeacherPermissionUseCase（`application/usecase/show_teacher_permission_usecase.go`）

- struct名: `ShowTeacherPermissionUseCase`
- コンストラクタが受け取る依存: `repository.TeacherPermissionRepository`
- 公開メソッド: `(u *ShowTeacherPermissionUseCase) Execute(ctx context.Context, query dto.ShowTeacherPermissionQuery) (dto.ShowTeacherPermissionResult, error)`
- 処理ステップ:
  1. `TeacherPermissionRepository.FindByTeacherID(ctx, query.RequestingTeacher.HighSchoolID, query.TeacherID)`を呼び出す
  2. 見つからない場合（同校でない場合を含む）、`ErrTargetTeacherNotFound`（Application Error）を返す
  3. 見つかった場合、`TeacherPermissionSummary`に変換し`ShowTeacherPermissionResult`を返す
- トランザクション境界: なし（読み取りのみ）
- 発生しうるApplication Error: `ErrTargetTeacherNotFound`

### UpdateTeacherPermissionUseCase（`application/usecase/update_teacher_permission_usecase.go`）

- struct名: `UpdateTeacherPermissionUseCase`
- コンストラクタが受け取る依存: `repository.TeacherPermissionRepository`, `repository.ActiveTeacherCountRepository`, `*service.TeacherPermissionUpdateGuard`, `TransactionManager`（後述8章）
- 公開メソッド: `(u *UpdateTeacherPermissionUseCase) Execute(ctx context.Context, cmd dto.UpdateTeacherPermissionCommand) (dto.UpdateTeacherPermissionResult, error)`
- 処理ステップ（呼び出し順序）:
  1. `valueobject.NewGradeScope(cmd.GradeScope)`で入力形式を検証する（不正時は`ErrInvalidGradeScope`）
  2. `valueobject.NewManageOtherTeachersFlag(cmd.ManageOtherTeachers)`を生成する
  3. `TransactionManager`内で以下（4〜8）を実行する
  4. `TeacherPermissionRepository.FindByTeacherID(ctx, cmd.RequestingTeacher.HighSchoolID, cmd.RequestingTeacher.TeacherID)`で更新者自身の権限を取得する（②13章「Domain」の「更新者が`manage_other_teachers`権限を持つか」の判定に必要な入力）
  5. `TeacherPermissionRepository.FindByTeacherID(ctx, cmd.RequestingTeacher.HighSchoolID, cmd.TeacherID)`で更新対象の権限を取得する。見つからない場合は`ErrTargetTeacherNotFound`を返す
  6. `ActiveTeacherCountRepository.ExistsOtherActiveTeacher(ctx, cmd.RequestingTeacher.HighSchoolID, cmd.TeacherID)`で校内の他有効教員の存在を確認する
  7. `TeacherPermissionUpdateGuard.Authorize(updater, target, hasOtherActiveTeacher)`で保護ルールを判定する。違反時はDomain Error（`ErrPermissionDenied` / `ErrSelfUpdateNotAllowed` / `ErrLastActiveTeacherProtected`）をそのまま返す
  8. `target.UpdateGradeScope(newScope)` / `target.UpdateManageOtherTeachers(newFlag)`で更新後の状態を反映し、`TeacherPermissionRepository.Update(ctx, target)`で永続化する
  9. `UpdateTeacherPermissionResult`を構築して返す
- トランザクション境界: ②11章「UpdateTeacherPermissionUseCaseの開始時にトランザクションを開始する」「保護ルール判定を通過し、TeacherPermissionの更新が完了した時点でコミットする」に従い、ステップ4〜8をトランザクション内で実行する（詳細は8章）
- 発生しうるApplication Error: `ErrTargetTeacherNotFound`。Domain Error（`ErrPermissionDenied` / `ErrSelfUpdateNotAllowed` / `ErrLastActiveTeacherProtected` / `ErrInvalidGradeScope`）はUseCaseで変換せずそのまま上位（Presentation）へ伝播させる

> **②からの補足**: ステップ4（更新者自身の`TeacherPermission`をRepository経由で取得する処理）は②に明記されたRepository呼び出し一覧（②10章「呼び出すRepository: TeacherPermissionRepository（対象取得・更新）、ActiveTeacherCountRepository」）には直接記載がない。しかし②13章「Domain」で「更新者が`manage_other_teachers`権限を持つか」の判定がGuardの責務とされており、Guardが`*entity.TeacherPermission`を受け取る設計（本書3章）である以上、更新者自身のEntityをどこかで取得する必要がある。UseCase内で`TeacherPermissionRepository.FindByTeacherID`を再利用してこれを取得する構成とした（推測、②の記載を矛盾なく実装するための補足）。

---

# 5. Infrastructure層設計

## Repository実装

### TeacherPermissionRepositoryImpl（`infrastructure/repository/teacher_permission_repository.go`）

- 実装struct名: 非公開struct（例: `teacherPermissionRepository`）+ コンストラクタ`NewTeacherPermissionRepository(db *gorm.DB) repository.TeacherPermissionRepository`（規約「9. 命名規約」により`Impl`接尾辞を付けず、コンストラクタで公開する）
- 対応するGORMモデル: `gormmodel.TeacherPermissionModel`（`infrastructure/persistence/gorm/teacher_permission_model.go`、`teacher_permissions`テーブルに対応）。氏名・所属校・有効性の絞り込みには、`users`テーブルの必要列（`id`・`high_school_id`・`name_kana`・`deleted_at`）のみを射影する読み取り専用の結合用struct（非公開）を同ファイル内に定義する
- 各メソッドで発行するクエリ内容:
  - `ListBySchool`: `users.high_school_id`が指定校IDと一致し、かつ`users.deleted_at`が未設定（有効教員）である教員に紐づく`teacher_permissions`を結合取得する。`users.name_kana`昇順でソートし、`page`に基づくページネーションを行う。絞り込み条件に一致する総件数を別途取得する
  - `FindByTeacherID`: `teacher_permissions`の対象教員IDと`users.high_school_id`が指定校IDに一致し、`users.deleted_at`が未設定であるレコードを取得する（同校スコープを兼ねる。②13章「UseCase」の「更新時、対象教員が同校であることを確認する」をクエリレベルで反映）
  - `Update`: 対象レコードの`grade_scope`・`manage_other_teachers`列を更新する
- Entity ⇔ GORMモデルの変換方針: 取得した行の`grade_scope`文字列から`valueobject.NewGradeScope`、`manage_other_teachers`真偽値から`valueobject.NewManageOtherTeachersFlag`を生成し、`entity.NewTeacherPermission`でEntityを組み立てる。結合先`users`テーブルの`name`・`name_kana`は`TeacherPermissionRecord`の付帯情報として保持する（3章参照）。変換関数（`toEntity`・`toRecord`）はrepository実装内の非公開関数とする

> **②からの補足**: `teacher_permissions`テーブルが`high_school_id`列を独自に持つか、常に`users`テーブルとの結合で所属校を判定するかは①未提供のため確定できない。②17章「『最後の有効教員』判定は`users.deleted_at`による既存の論理削除の仕組みを利用できる」という記載から、有効性判定は`users`テーブルに依存する設計であることが示唆されるため、本書では所属校判定・氏名取得も同様に`users`テーブルとの結合を前提とした（推測）。また、規約「6. Context間連携ルール」は他Contextの内部Entity・Infrastructure実装への直接依存を禁止しているが、本書では`users`テーブルの必要列のみを射影する読み取り専用の結合用structを本Context内に定義することで、Teacher Directory/User Context側のGORMモデル実装への直接依存を避ける（実装上の判断）。規約「5. Bounded Context構成 今後の課題」のとおりUser Context自体の②文書が未整備であるため、正式なモデル参照方法は将来的に見直す余地がある。

### ActiveTeacherCountRepositoryImpl（`infrastructure/repository/active_teacher_count_repository.go`）

- 実装struct名: 非公開struct（例: `activeTeacherCountRepository`）+ コンストラクタ`NewActiveTeacherCountRepository(db *gorm.DB) repository.ActiveTeacherCountRepository`
- 対応するGORMモデル: 上記と同じ`users`テーブルの読み取り専用射影
- クエリ内容: `ExistsOtherActiveTeacher`は、`users.high_school_id`が指定校IDと一致し、`users.deleted_at`が未設定であり、`users.id`が`excludeTeacherID`と異なり、かつ教員ロールである教員が1件以上存在するかどうかの存在確認クエリを発行する

> **②からの補足**: 「教員ロールである」という条件は②9章の「同校の有効教員集合」という記載から導出した（推測）。`users`テーブルにロールを表す列（`user_role_id`等）が存在する前提は、②「20. 採用しなかった設計」等の記載から推測されるが、具体的な列名は①未提供のため確定できない。

## 外部連携実装

対象外。本機能はMail・Cache・Queue等の外部連携を必要としない（②に記載なし）。

---

# 6. Presentation層設計

## Handler

- struct名: `TeacherPermissionHandler`
- 対応する呼び出し先: `usecase.ListTeacherPermissionsUseCase` / `usecase.ShowTeacherPermissionUseCase` / `usecase.UpdateTeacherPermissionUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`List(c *gin.Context)`|GET|`/api/v1/teacher/permissions`|
|`Show(c *gin.Context)`|GET|`/api/v1/teacher/permissions/:id`|
|`Update(c *gin.Context)`|PATCH|`/api/v1/teacher/permissions/:id`|

- 処理順序（`List`）:
  1. Ginコンテキストから認証Middlewareが格納したcurrent teacher情報を取得し、`dto.RequestingTeacher`へ変換する（Handlerでは業務権限判定を行わない。②13章「Handler」）
  2. クエリパラメータ`page`をバインドする
  3. `dto.ListTeacherPermissionsQuery`を組み立て、`ListTeacherPermissionsUseCase.Execute`を呼び出す
  4. 取得結果を`TeacherPermissionListResponse`へ変換する
  5. `200`でレスポンスを返す
- 処理順序（`Show`）:
  1. current teacher情報を取得し`dto.RequestingTeacher`へ変換する
  2. パスパラメータ`id`をバインドする
  3. `dto.ShowTeacherPermissionQuery`を組み立て、`ShowTeacherPermissionUseCase.Execute`を呼び出す
  4. `ErrTargetTeacherNotFound`が返却された場合は`404`を返す
  5. 成功時は`TeacherPermissionDetailResponse`へ変換し`200`で返す
- 処理順序（`Update`）:
  1. current teacher情報を取得し`dto.RequestingTeacher`へ変換する
  2. パスパラメータ`id`をバインドする
  3. `UpdateTeacherPermissionRequest`をJSONボディからバインドし、Presentation Validation（9章）を行う
  4. `dto.UpdateTeacherPermissionCommand`を組み立て、`UpdateTeacherPermissionUseCase.Execute`を呼び出す
  5. 戻り値のエラーを11章のError実装方針に従いHTTPステータスへ変換する（`ErrTargetTeacherNotFound`→404、Domain Error群→422）
  6. 成功時は`TeacherPermissionDetailResponse`へ変換し`200`で返す

## Request / Response DTO

Request DTO（`internal/teacher_permission/presentation/request/teacher_permission_request.go`）:

|struct名|フィールド|型|バリデーションタグ／チェック内容|
|-|-|-|-|
|`UpdateTeacherPermissionRequest`|`TeacherPermission`|`UpdateTeacherPermissionBody`|`binding:"required"`（②12章「Presentation」必須チェック）|
|`UpdateTeacherPermissionBody`|`GradeScope`|`string`|`json:"grade_scope" binding:"required"`（型チェック・必須チェック、②12章）|
||`ManageOtherTeachers`|`*bool`|`json:"manage_other_teachers" binding:"required"`（②12章）|

> **②からの補足**: リクエストボディの構造（`teacher_permission`キーでネストする形）は②16章「Request」の「`teacher_permission[grade_scope]` / `teacher_permission[manage_other_teachers]`はGo側で入力DTOとして吸収する」という記載から、JSONボディ上で`teacher_permission`オブジェクトとしてネストされる構造を採用した（推測。①未提供のため、Rails側のパラメータ形式そのものは参照不可）。`ManageOtherTeachers`を`*bool`（ポインタ）にしたのは、Goの`bool`ゼロ値（`false`）と「未送信」を区別し必須チェックを機能させるための実装判断である（推測）。

Response DTO（`internal/teacher_permission/presentation/response/teacher_permission_response.go`）:

|struct名|フィールド|型|
|-|-|-|
|`TeacherPermissionListResponse`|`Permissions`|`[]TeacherPermissionSummaryResponse`|
||`Meta`|`MetaResponse`|
|`TeacherPermissionSummaryResponse`|`TeacherID`|`uint`|
||`TeacherName`|`string`|
||`TeacherNameKana`|`string`|
||`GradeScope`|`string`|
||`ManageOtherTeachers`|`bool`|
|`TeacherPermissionDetailResponse`|`TeacherID`|`uint`|
||`TeacherName`|`string`|
||`TeacherNameKana`|`string`|
||`GradeScope`|`string`|
||`ManageOtherTeachers`|`bool`|
|`MetaResponse`|`CurrentPage`|`int`|
||`TotalPages`|`int`|
||`TotalCount`|`int`|
||`PerPage`|`int`|

②16章「一覧・詳細・更新のレスポンス構造はRails現行仕様に近い意味を維持する」との記載にとどまり、フィールドの過不足・命名の詳細（snake_case変換等はJSONタグで対応）は①未提供のため確定できない範囲がある（14章参照）。

## Routing

`internal/teacher_permission/presentation/routes.go`

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/teacher/permissions`|`TeacherPermissionHandler.List`|
|GET|`/api/v1/teacher/permissions/:id`|`TeacherPermissionHandler.Show`|
|PATCH|`/api/v1/teacher/permissions/:id`|`TeacherPermissionHandler.Update`|

いずれのルートも、認証Middleware・teacherロール確認Middlewareを経由する（②13章「Middleware」）。当該Middlewareは本機能内で新規実装せず、既存の共通Middlewareを利用する想定とする（①未提供のため実装詳細不明）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/teacher/permissions|TeacherPermissionHandler.List|page（query, optional）|TeacherPermissionListResponse|200|
|GET|/api/v1/teacher/permissions/:id|TeacherPermissionHandler.Show|id（path, required）|TeacherPermissionDetailResponse|200|
|PATCH|/api/v1/teacher/permissions/:id|TeacherPermissionHandler.Update|id（path, required）, UpdateTeacherPermissionRequest（body）|TeacherPermissionDetailResponse|200|

Errorケース:

|条件|Status Code|Error内容|
|-|-|-|
|更新者が`manage_other_teachers`権限を持たない|422|`ErrPermissionDenied`（既存のerrors形式の日本語メッセージを踏襲、②16章）|
|更新者が自分自身を更新しようとした|422|`ErrSelfUpdateNotAllowed`|
|更新対象が校内で最後の有効教員である|422|`ErrLastActiveTeacherProtected`|
|`grade_scope`が許容値でない|422|`ErrInvalidGradeScope`|
|`manage_other_teachers`が未送信（Presentation Validation）|422|入力不正（②16章のとおり422に集約）|
|対象教員が存在しない、または同校でない|404|`ErrTargetTeacherNotFound`|
|`page`パラメータが不正な形式|400（推測、14章参照）|バリデーションエラー|
|`id`パラメータが不正な形式|400（推測、14章参照）|バリデーションエラー|
|未認証|401（推測、14章参照）|認証エラー|
|teacherロールでない|403（推測、14章参照）|権限エラー|

---

# 8. Transaction実装方針

## Transaction開始箇所

- `UpdateTeacherPermissionUseCase.Execute`内、入力DTOの形式検証（VO生成）の直後、更新者・更新対象の権限取得（4章ステップ4）を行う前に`TransactionManager`を用いてトランザクションを開始する（②11章「UpdateTeacherPermissionUseCaseの開始時にトランザクションを開始する」を実装単位に落とし込んだもの）

## Transaction終了箇所

- `TeacherPermissionUpdateGuard.Authorize`による保護ルール判定を通過し、`TeacherPermissionRepository.Update`による永続化が完了した時点でコミットする（②11章）
- 保護ルール判定（Domain Error）・対象未検出（`ErrTargetTeacherNotFound`）のいずれかが発生した場合はロールバックする
- `ListTeacherPermissionsUseCase` / `ShowTeacherPermissionUseCase`ではトランザクションを使用しない（②11章）

## 複数Repositoryにまたがる場合の扱い

`UpdateTeacherPermissionUseCase`は、`TeacherPermissionRepository`（更新者取得・対象取得・更新）と`ActiveTeacherCountRepository`（有効教員存在確認）の2つのRepositoryを1つのトランザクション内で呼び出す。両者は同一の`TransactionManager`（同一DBコネクション/トランザクション）を経由して呼び出す構成とする。

> **②からの補足**: `TransactionManager`のインターフェース設計（例: `WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error`）は②に明記がなく、他機能（認証機能等）の実装仕様書と同様の構成を踏襲した実装判断である（推測）。

---

# 9. Validation実装方針

## Presentation

- 型チェック: `grade_scope`が文字列、`manage_other_teachers`が真偽値であることをJSONバインド時に検証する（②12章「型チェック」）
- 必須チェック: `grade_scope` / `manage_other_teachers`が送信されていることを`binding:"required"`で検証する（②12章「必須チェック」）
- フォーマットチェック: `grade_scope`が文字列として送信されていること（②12章「フォーマットチェック」）

## 業務ルール検証

Domain Model採用のため、Entity／Value Object生成時とUseCase内で以下を検証する。

- `valueobject.NewGradeScope`: `grade_scope`が`own_grade` / `all_grades`の許容値であるかを検証する（②12章「整合性チェック」）
- `valueobject.NewManageOtherTeachersFlag`: `manage_other_teachers`の型的な真偽値であることを保証する（Go型システムにより自動的に満たされる、3章参照）
- `TeacherPermissionUpdateGuard.Authorize`（UseCase内で呼び出す）: 更新者が`manage_other_teachers`権限を持つか（②12章「業務ルール」）、自己更新でないか・最後の有効教員でないか（②12章「状態チェック」）を検証する

## 責務分離

②12章の方針どおり、Presentationは「入力の形式が正しいか」を、Domainは「業務的に妥当か（更新してよい権限者か、保護対象でないか）」を担当する。これにより入力仕様の変更と組織運用ルールの変更を独立して扱える。

---

# 10. Authorization実装方針

## Middleware

- 認証済みユーザーを特定し、`teacher`ロールであることを確認する（②13章「Middleware」）

## Handler

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定は持たせない（②13章「Handler」）。current teacher情報を`dto.RequestingTeacher`へ変換してUseCaseへ渡すのみ

## UseCase

- `ListTeacherPermissionsUseCase` / `ShowTeacherPermissionUseCase`: `RequestingTeacher.HighSchoolID`を用いて取得範囲を同校にスコープする（②13章「UseCase」）
- `UpdateTeacherPermissionUseCase`: `TeacherPermissionRepository.FindByTeacherID`のクエリ条件（同校スコープ）により、対象教員が同校であることを確認する（②13章「UseCase」「更新時、対象教員が同校であることを確認する」）

## Domain

- `TeacherPermissionUpdateGuard.Authorize`において、以下を判定する（②13章「Domain」）
  - 更新者が`manage_other_teachers`権限を持つか
  - 更新者が更新対象本人でないか
  - 更新対象が校内で最後の有効教員でないか

## 判断理由

②13章「判断理由」のとおり、「同校教員のみを対象とする」という認証・スコープ確認はMiddleware/UseCaseに配置し、「更新してよい権限者か」「保護対象でないか」という組織運用上の判断はDomain（Guard）に集約することで、権限体系や保護ルールの変更が生じてもDomain層のみの修正で対応できるようにする。

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

`TeacherPermissionUpdateGuard`・Value Objectが返すDomain Error（`ErrPermissionDenied` / `ErrSelfUpdateNotAllowed` / `ErrLastActiveTeacherProtected` / `ErrInvalidGradeScope`）は、UseCase内で別のApplication Error型へ変換せず、そのままPresentation層まで伝播させる。Presentation層で`errors.Is`によるエラー種別判定を行い、HTTPステータス・メッセージへ変換する。

## Application Error → HTTPレスポンスへの変換方針

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrPermissionDenied`|Domain|422|
|`ErrSelfUpdateNotAllowed`|Domain|422|
|`ErrLastActiveTeacherProtected`|Domain|422|
|`ErrInvalidGradeScope`|Domain|422|
|`ErrTargetTeacherNotFound`|Application|404|
|Presentation Validation失敗（`binding`タグ違反）|Presentation|422（推測、14章参照）|
|`page`パラメータ不正|Presentation|400（推測、14章参照）|
|`id`パラメータ不正|Presentation|400（推測、14章参照）|
|未認証|Middleware|401（推測、14章参照）|
|teacherロール不一致|Middleware|403（推測、14章参照）|
|DB接続障害等|Infrastructure|500（推測、14章参照）|

## Infrastructure Errorのハンドリング方針

DB接続失敗等のInfrastructure Errorは、Repository実装から`fmt.Errorf`でラップして返し（コーディング規約「6. エラーのラップ」）、Handlerで判別不能な未分類エラーとして`500`に変換する（②に個別の記載はなく、一般的なエラーハンドリング方針として補った）。

## 判断理由（②からの引用）

②14章「判断理由」のとおり、自己更新禁止・最後の教員保護・権限不足といった組織運用上の拒否理由は、単なる入力ミスとは性質が異なる業務ルール違反であるため、Domain Errorとして明確に区別し、Presentation層でメッセージへ変換する構造とする（既存のerrors形式・日本語メッセージを踏襲、②16章）。

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

②17章のとおり、既存Rails DBを継続利用し、Schema変更は行わない。

|struct名|テーブル|備考|
|-|-|-|
|`gormmodel.TeacherPermissionModel`|`teacher_permissions`|Gorm規約「複数形のテーブル名」のデフォルト変換では`teacher_permission_models`となり実テーブル名と一致しないため、`Tabler`インターフェース（`func (TeacherPermissionModel) TableName() string { return "teacher_permissions" }`）でテーブル名を明示する（Gorm規約「テーブル名」節に従う）|
|（読み取り専用射影）|`users`|`id`・`high_school_id`・`name`・`name_kana`・`deleted_at`列のみを射影する非公開struct。所属校絞り込み・有効教員判定・氏名表示・名前順ソートに用いる|

`TeacherPermissionModel`は主キーとして`ID`フィールドを持ち（Gorm規約「主キーとしてのID」のデフォルト規約に従う）、`CreatedAt` / `UpdatedAt`フィールドを持たせ、GORMの自動タイムスタンプ管理（Gorm規約「タイムスタンプのトラッキング」）に従う。カラム名はGoフィールド名のsnake_case変換に従い、`GradeScope string`は`grade_scope`列、`ManageOtherTeachers bool`は`manage_other_teachers`列に対応する。

## 主要クエリの条件・ソート・ページネーション方針

- `ListBySchool`: `users.high_school_id`一致 AND `users.deleted_at IS NULL` → `teacher_permissions`を教員IDで結合 → `users.name_kana`昇順ソート → ページネーション → 総件数を別途取得
- `FindByTeacherID`: `teacher_permissions`の対象教員ID一致 AND `users.high_school_id`一致 AND `users.deleted_at IS NULL`
- `Update`: 対象レコードの`grade_scope`・`manage_other_teachers`列を更新
- `ExistsOtherActiveTeacher`: `users.high_school_id`一致 AND `users.deleted_at IS NULL` AND `users.id != excludeTeacherID` AND 教員ロール一致、の存在確認

SQL文そのものは記載しない。

## 既存Schemaに対する変更

②17章のとおり、Schema変更は提案されていないため、本書でも反映しない。「最後の有効教員」判定は`users.deleted_at`による既存の論理削除の仕組みを利用する（②17章）。

---

# 13. テストケース設計

Domain Model採用のため、②18章の区分をそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|`valueobject.NewGradeScope`|`own_grade` / `all_grades`を指定した場合、正しく生成されること|
|`valueobject.NewGradeScope`|許容値以外を指定した場合、`ErrInvalidGradeScope`相当のエラーが返ること|
|`entity.TeacherPermission.CanManageOtherTeachers`|`ManageOtherTeachers`が`true`のとき`true`を返すこと|
|`entity.TeacherPermission.CanManageOtherTeachers`|`ManageOtherTeachers`が`false`のとき`false`を返すこと|
|`TeacherPermissionUpdateGuard.Authorize`|更新者が`manage_other_teachers`権限を持たない場合、`ErrPermissionDenied`が返ること|
|`TeacherPermissionUpdateGuard.Authorize`|更新者が更新対象本人である場合、`ErrSelfUpdateNotAllowed`が返ること|
|`TeacherPermissionUpdateGuard.Authorize`|`hasOtherActiveTeacher`が`false`の場合、`ErrLastActiveTeacherProtected`が返ること|
|`TeacherPermissionUpdateGuard.Authorize`|いずれの保護ルールにも抵触しない場合、`nil`が返ること|

## UseCase Test

|対象|テストケース|
|-|-|
|`ListTeacherPermissionsUseCase`|同校教員の権限一覧が氏名カナ順・ページネーション付きで取得できること|
|`ListTeacherPermissionsUseCase`|該当0件の場合、空一覧とページ情報が返ること|
|`ShowTeacherPermissionUseCase`|同校の対象教員を指定した場合、詳細情報が取得できること|
|`ShowTeacherPermissionUseCase`|他校の教員IDを指定した場合、`ErrTargetTeacherNotFound`が返ること|
|`UpdateTeacherPermissionUseCase`|`manage_other_teachers`権限を持つ更新者が、自分以外かつ校内で最後でない教員の権限を更新できること|
|`UpdateTeacherPermissionUseCase`|`manage_other_teachers`権限を持たない更新者が更新しようとした場合、`ErrPermissionDenied`が返り更新が行われないこと|
|`UpdateTeacherPermissionUseCase`|更新者が自分自身を更新しようとした場合、`ErrSelfUpdateNotAllowed`が返り更新が行われないこと|
|`UpdateTeacherPermissionUseCase`|更新対象が校内で最後の有効教員である場合、`ErrLastActiveTeacherProtected`が返り更新が行われないこと|
|`UpdateTeacherPermissionUseCase`|対象教員が存在しない、または他校である場合、`ErrTargetTeacherNotFound`が返ること|
|`UpdateTeacherPermissionUseCase`|保護ルール違反時、トランザクションがロールバックされ`Update`が呼ばれていないこと|

## Repository Test

|対象|テストケース|
|-|-|
|`TeacherPermissionRepositoryImpl.ListBySchool`|`high_school_id`・有効教員（`deleted_at IS NULL`）の絞り込みが正しく適用されること|
|`TeacherPermissionRepositoryImpl.ListBySchool`|`name_kana`昇順でソートされること|
|`TeacherPermissionRepositoryImpl.ListBySchool`|ページネーションの境界値（最終ページ・空ページ）が正しく扱われること|
|`TeacherPermissionRepositoryImpl.FindByTeacherID`|同校かつ有効な対象教員の場合、レコードが取得できること|
|`TeacherPermissionRepositoryImpl.FindByTeacherID`|他校・無効教員・不存在の場合、レコードが取得できないこと|
|`TeacherPermissionRepositoryImpl.Update`|`grade_scope` / `manage_other_teachers`が正しく更新されること|
|`ActiveTeacherCountRepositoryImpl.ExistsOtherActiveTeacher`|対象教員以外に同校の有効教員が存在する場合、`true`が返ること|
|`ActiveTeacherCountRepositoryImpl.ExistsOtherActiveTeacher`|対象教員が校内で最後の有効教員である場合、`false`が返ること|

## Handler Test

|対象|テストケース|
|-|-|
|`TeacherPermissionHandler.List`|`page`が正しい整数の場合、200と`TeacherPermissionListResponse`が返ること|
|`TeacherPermissionHandler.Show`|`id`が正しい整数かつ対象が存在する場合、200と`TeacherPermissionDetailResponse`が返ること|
|`TeacherPermissionHandler.Show`|対象が存在しない場合、404が返ること|
|`TeacherPermissionHandler.Update`|正しい入力の場合、200と更新後の`TeacherPermissionDetailResponse`が返ること|
|`TeacherPermissionHandler.Update`|`grade_scope` / `manage_other_teachers`未送信の場合、422が返ること|
|`TeacherPermissionHandler.Update`|Domain Error（権限不足・自己更新・最後の教員保護）発生時、422と対応するエラーメッセージが返ること|
|`TeacherPermissionHandler.Update`|対象教員が存在しない場合、404が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/teacher/permissions`|エンドポイント経由で同校教員の権限一覧が取得できること|
|`GET /api/v1/teacher/permissions/:id`|エンドポイント経由で対象教員の権限詳細が取得できること|
|`PATCH /api/v1/teacher/permissions/:id`|正常な更新が反映されること|
|`PATCH /api/v1/teacher/permissions/:id`|自己更新が拒否されること（422）|
|`PATCH /api/v1/teacher/permissions/:id`|最後の有効教員保護が機能すること（422）|
|`PATCH /api/v1/teacher/permissions/:id`|`manage_other_teachers`権限を持たない教員による更新が拒否されること（422）|
|全体|未認証・非teacherロールでのアクセスがMiddlewareで拒否されること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

|No.|判断した内容|判断理由|推測かどうか|
|-|-|-|-|
|1|`internal/`配下のディレクトリ名を`internal/teacher_permission`とした|②のContext名`teacher-permission`とディレクトリ名の対応関係が②に明記がないため、規約「9. 命名規約」の変換ルールに基づき判断した|推測|
|2|`TeacherPermission` Entityの更新メソッド（`UpdateGradeScope` / `UpdateManageOtherTeachers`）を戻り値`error`なしとした|引数のValue Object生成時点で妥当性検証が完了している前提のため。②6章にメソッドシグネチャの明記はない|推測（実装判断）|
|3|`ManageOtherTeachersFlag`のコンストラクタをエラーを返さない形にした|Goの`bool`型自体が真偽値以外を取り得ないため、②7章の「true/false以外の値を許容しない」というルールはGoの型システムにより自動的に満たされる|実装上の判断（②の設計判断自体は変更していない）|
|4|`TeacherPermissionRepository`の戻り値として、Entityとは別に氏名・かな氏名を含む`TeacherPermissionRecord`型を定義した|②6章のEntityは氏名を保持しない設計だが、②9章のRepositoryは氏名カナ順ソートを責務として持つため、一覧・詳細表示に必要な氏名情報をEntityとは別の付帯情報として扱う構成とした|実装上の判断（②の記載同士の整合を取るため）|
|5|`TeacherPermissionUpdateGuard.Authorize`が`ActiveTeacherCountRepository`を直接呼び出さず、判定済みの事実（`hasOtherActiveTeacher bool`）を引数で受け取る構成とした|②8章「テスト容易性」の記載（DBを経由せずルール単体を検証できること）を満たすための実装判断|推測|
|6|`UpdateTeacherPermissionUseCase`内で、更新者自身の`TeacherPermission`をRepository経由で取得する処理ステップを追加した|②13章「Domain」の権限判定（Guardが更新者のEntityを必要とする）を満たすために必要な処理だが、②10章のUseCase設計には明記がない|推測|
|7|Request DTO（`UpdateTeacherPermissionRequest`）を`teacher_permission`キーでネストしたJSON構造とし、`ManageOtherTeachers`を`*bool`とした|②16章「Request」の記載（Rails側のネストしたパラメータ形式をGo側で入力DTOとして吸収する）から導出。`*bool`は必須チェックのためのGo実装上の判断|推測|
|8|Response DTOの詳細フィールド構成（一覧・詳細で氏名・かな氏名を含める等）|②16章は「Rails現行仕様に近い意味を維持する」とのみ記載し、具体的なフィールドはRails実装（①）に依存する。①は本書作成時点で未提供のため参照不可|①未提供のため参照不可（一部推測を含む）|
|9|`teacher_permissions`テーブルが`high_school_id`列を持たず、`users`テーブルとの結合で所属校・有効性・氏名を判定する前提とした|②17章の「『最後の有効教員』判定は`users.deleted_at`を利用できる」という記載から、有効性判定が`users`テーブル依存であることが示唆されるため、所属校・氏名判定も同様に扱った|推測|
|10|`users`テーブルの必要列のみを射影する読み取り専用structを本Context内に定義し、Teacher Directory/User Context側のGORMモデルを直接importしない方針とした|規約「6. Context間連携ルール」の「相手Contextの内部Entity・Infrastructure実装に直接依存しない」に配慮した実装判断。規約「5. Bounded Context構成 今後の課題」のとおりUser Context自体の②文書が未整備のため、正式な参照方法は将来的に見直す余地がある|推測|
|11|`ExistsOtherActiveTeacher`の判定条件に「教員ロールであること」を含めた|②9章の「同校の有効教員集合」という記載から、生徒・管理者を含まない教員のみを対象とすべきと判断した。具体的なロール列名は①未提供のため確定できない|推測|
|12|`page` / `id`パラメータ不正時に400、未認証時401・ロール不一致時403というHTTP Statusの割り当て|②16章のStatus Code一覧は200・404・422のみで、これら個別のケースへの明記がない。一般的なAPI設計慣習として補った|推測|
|13|DB接続障害等のInfrastructure Errorを500として扱う方針|②にInfrastructure Errorに対応するHTTP Statusの記載がなく、一般的なエラーハンドリング方針として補った|推測|
|14|`TransactionManager`のインターフェース設計（`WithinTransaction`等）|②11章はトランザクション境界（開始・終了位置）のみを定めており、具体的なインターフェース形状の明記はない|推測|
