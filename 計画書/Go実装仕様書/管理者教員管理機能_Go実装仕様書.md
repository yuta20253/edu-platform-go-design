# 管理者教員管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が自校（選択中の高校）に所属する教員アカウントを一覧参照し、新規教員の招待（作成）、既存教員のプロフィール・権限・担当学年の更新を行う機能である。新規教員の招待は、氏名・メールアドレス・権限・担当学年を指定して教員を作成し、作成時に招待メールが送信される（教員アカウント本体の作成と招待メールの送信依頼は`user` Contextの`CreateTeacherAccount`に任せ、本Contextは権限・担当学年の作成を担う）。教員の権限情報（`teacher_permissions`）と担当学年情報（`teacher_grades`）を教員本体と合わせて一貫管理する（②「1. 機能概要」）。

**本書の特別な位置づけ**: 本機能（管理者教員管理機能）と教師教員管理機能は、同一のBounded Context（`teacher-management`）・同一のAggregate（Teacher）を扱う一体の業務領域である。同じ3テーブル（`users`の教員レコード / `teacher_permissions` / `teacher_grades`）に対するEntity・Repository（Store）は、実際のGoコードでは同じpackage（`internal/teacher_management`）に1セットのみ存在し、管理者向けHandler（本書）と教師向けHandler（教師教員管理機能_Go実装仕様書）がこれを共有する。そのため本書は、Model（`Teacher` / `TeacherPermission` / `TeacherGradeAssignment`）とStore（`TeacherStore` / `TeacherPermissionStore` / `TeacherGradeStore`）を、管理者視点・教師視点の両方の操作を包含する形で**正の文書として完全に定義する**。教員の新規作成（管理者視点の招待・教師視点の新規作成）は、いずれも`users`の1行の作成を`user` Contextの`CreateTeacherAccount`に任せ、`TeacherStore`は権限・担当学年の作成とトランザクションの管理を担う（経路ごとの指定値は「3. Domain層設計」`TeacherAccountOptions`）。教師教員管理機能_Go実装仕様書は、本書で定義したModel・Storeをそのまま再利用し、教師視点固有の処理（同校スコープの参照、他職員操作権限の確認、招待通知状況の付与）とpresentation層のみを追加で定義する。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により **Active Record** を採用する。

- 主要操作が一覧・作成・更新というCRUDに限定される
- 権限情報（`teacher_permissions`）は教員に対して1:1で従属する属性群であり、独立した振る舞いを持たない
- 担当学年（`teacher_grades`）は「更新時に全置換する」という単純な同期ルールであり、複雑な状態遷移を伴わない
- Handlerが直接Storeを呼び出し、認可・入力検証をHandler／Store側で行うことで、教員データの保存・関連付けの振る舞いを一元化しやすい

Transaction Scriptは権限・担当学年同期ロジックがUseCaseに偏り再利用性が下がることを理由に、Domain Modelは複雑な状態遷移が存在しないことを理由に、Event Sourcingは非同期通知・監査要件が現行仕様に明記されていないことを理由に、②でいずれも不採用と判断されている。本書はこれらの判断を変更しない。

本書は、`規約/アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」の **Active Record** 節の構造（Entity相当のstruct定義＋Store、usecase層なし、Repository Interfaceの分離なし）に従って実装レベルへ落とし込む。

## 本書が対象とする実装範囲

- `GET /api/v1/admin/high_schools/:high_school_id/teachers`（教員一覧取得）
- `POST /api/v1/admin/high_schools/:high_school_id/teachers`（教員招待＝作成）
- `PATCH /api/v1/admin/high_schools/:high_school_id/teachers/:id`（教員更新）

の3エンドポイントの実装に必要な、struct・Store・Handler・Routing・Request/Response構造体の実装単位を規定する。

- 対象外: 教師視点のエンドポイント（`GET /api/v1/teacher/colleagues`・`GET /api/v1/teacher/colleagues/:id`・`POST /api/v1/teacher/colleagues`）のpresentation層は教師教員管理機能_Go実装仕様書の責務とする
- 対象外: HighSchool・Grade・教員アカウント（`users`の1行）そのものの生成能力は他Contextの責務であり（②「3. Bounded Context」他Contextとの依存関係。教員アカウントの作成は`user` Contextの`CreateTeacherAccount`）、本書はteacher-management Contextが利用する参照手段・作成操作のインターフェースまでを扱う
- ①Rails現行仕様書（管理者教員管理機能_Rails現行仕様書）の記載は、教員招待（POST）の入力項目（`name` / `email` / `grade_scope` / `manage_other_teachers` / `grade_ids`）・作成時の招待メール・担当学年の作成・201の応答について本書の根拠とする。それ以外の箇所で「①未提供のため参照不可」とある記載は、②の記載のみを実装仕様の根拠としている

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `teacher-management`（②「3. Bounded Context」）

## ②で採用した設計パターン

- Active Record（②「4. 設計パターン」）

## 採用パターンに対応する構造

`規約/アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」Active Record節に従い、domain/infrastructureのレイヤー分離およびusecase層を設けない。Entity相当のstructと永続化操作（Store）を同一package（`internal/teacher_management`）に置く。Model・Storeは管理者視点・教師視点で共有するため分割しない。

**②からの補足（本書固有の構造判断）**: 本機能は、同一Aggregate（Teacher）を管理者視点（本書）・教師視点（教師教員管理機能）という2つのHandler群が操作する構成を持つ。presentation層をどちらか一方のpackageにまとめると、Go言語のstruct名・ファイル名の重複（例：管理者視点・教師視点いずれも`TeacherHandler` / `TeacherResponse` / `TeacherListResponse`という名前が自然）が生じる。これを避けるため、presentation層を`admin`/`teacher`のサブパッケージへ分割する（詳細は「14. ②からの補足事項」参照）。

## 作成するディレクトリ一覧

```
internal/teacher_management/
└── presentation/
    └── admin/
        ├── handler/
        ├── request/
        ├── response/
        └── routes.go
```

`presentation/teacher/`は教師教員管理機能_Go実装仕様書が作成する（本書では作成しない）。教師視点固有のファイルも、教師教員管理機能_Go実装仕様書が同一package（`internal/teacher_management`）へ追加する。

**②からの補足**: アーキテクチャ規約「9. 命名規約」により、`internal/`配下のディレクトリ名は英単語1語または短いスネークケースとする。②のContext名`teacher-management`と`internal/`配下のディレクトリ名の対応関係は②に明記がないため、本書では`internal/teacher_management`と判断した（推測。詳細は「14. ②からの補足事項」参照）。

## 作成するファイル一覧

```
internal/teacher_management/teacher.go                        # Teacher struct・Validate()・TeacherAccountOptions等
internal/teacher_management/teacher_permission.go              # TeacherPermission struct（VO相当）
internal/teacher_management/teacher_grade_assignment.go        # TeacherGradeAssignment struct・GradeAssignmentSet
internal/teacher_management/teacher_grade_assignment_policy.go # 学年所属妥当性判定（②「8. Domain Service」相当）
internal/teacher_management/dependency.go                      # 他Context参照用インターフェース・入力型の定義（HighSchoolExistenceChecker / GradeReferenceChecker / TeacherAccountCreator・TeacherAccountInput）
internal/teacher_management/errors.go                          # struct/Storeが返すエラー変数定義
internal/teacher_management/teacher_store.go                   # TeacherStore
internal/teacher_management/teacher_permission_store.go        # TeacherPermissionStore
internal/teacher_management/teacher_grade_store.go              # TeacherGradeStore
internal/teacher_management/presentation/admin/handler/teacher_handler.go
internal/teacher_management/presentation/admin/request/teacher_request.go
internal/teacher_management/presentation/admin/response/teacher_response.go
internal/teacher_management/presentation/admin/routes.go
```

`domain/` `application/` `infrastructure/` の各ディレクトリはActive Record採用のため作成しない（規約4章）。

---

# 3. Domain層設計

**実装上の位置づけ**: 本機能はActive Record採用のため、domain層のディレクトリ分離は行わない。以下は②「6〜9章」の設計意図を、Active Record構造（Model＝struct＋メソッド、Store＝永続化）に落とし込んだものである（規約4章 Active Record節「『Value Object』『Repository Interface』『Domain Service』は原則『対象外』とし、検証ルールはModelのメソッドとして記載する」に従う）。

## Model（Entity相当）

### Teacher（`internal/teacher_management/teacher.go`）

②「6. Entity設計」Teacherの責務を反映する。教員としての`User`を表す。教師教員管理機能②「6. Entity設計」のTeacherと同一のEntityであり、教師視点の機能もこのstructを共有する。

フィールド:

|フィールド|型|意味|
|-|-|-|
|ID|uint|教員ID（`users`テーブルの主キー）|
|HighSchoolID|uint|所属高校ID（`users.high_school_id`。②「1. 機能概要」対象は自校所属教員に限定される、の根拠フィールド）|
|Name|string|教員氏名|
|NameKana|string|氏名カナ（`users.name_kana`。100文字以内。教師視点の一覧表示・新規作成で使用する。管理者視点の招待では、`NewTeacher`が氏名と同じ値を設定する（Rails現行の`Admin::CreateTeacherService`と同じ。`user`②の`CreateTeacherAccount`が氏名カナを必須とするため）。教師教員管理機能②「6. Entity設計」Teacher）|
|Email|string|メールアドレス（②「12. Validation設計」フォーマットチェック対象）|
|UserRoleID|uint|ロールを示す識別子（`users.user_role_id`。`user_roles`テーブルへの外部キー。Gorm規約「外部キー列は`{関連モデル名}ID`のまま明示的に保持する」に従い、認証機能_Go実装仕様書のAccountと同じ命名・型とする）。`users`テーブルにロール名そのものを保持する列はなく、教員であることの判定は`user_roles`との結合（`user_roles.name`が教員を表す値であること）で行う。`NewTeacher`は`UserRoleID`を設定しない。教員の作成時のロールの解決は`user` Contextの`CreateTeacherAccount`が行うため、本Contextは`UserRoleID`を書き込まない（取得結果の値としてのみ保持する）|
|CreatedAt|time.Time|作成日時（GORM自動設定。Gorm規約「タイムスタンプのトラッキング」）|
|UpdatedAt|time.Time|更新日時（GORM自動設定）|
|DeletedAt|gorm.DeletedAt|論理削除日時（`users.deleted_at`。GORMのソフトデリート対象とし、論理削除された教員は取得対象から除外する。教師教員管理機能②「9. Repository設計」`teacher`ロール・論理削除を除外する検索条件の根拠フィールド）|

`users`テーブルにはteacher-management以外のContext（認証等）が管理するカラム（`encrypted_password` / `jti` / `reset_password_token`等）が存在するため、本structは教員管理（管理者視点・教師視点の双方）に必要な最小限のフィールドのみを保持する（詳細は「14. ②からの補足事項」参照）。

公開method一覧（シグネチャのみ。実装ロジックは記載しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTeacher`|`(highSchoolID uint, name, email string) (*Teacher, error)`|`(*Teacher, error)`|新規Teacher生成時の不変条件（必須項目）を保証するファクトリ。管理者視点の招待で使用し、`NameKana`に`name`と同じ値を設定する|
|`(t *Teacher) Validate() error`|なし|`error`|name・email等必須項目の妥当性を検証する（②「12. Validation設計」Domainの一部）|
|`(t *Teacher) ApplyProfile(name, email string) error`|`name, email string`|`error`|更新時のプロフィール項目差し替えと検証をあわせて行う|
|`(t Teacher) BelongsToHighSchool(highSchoolID uint) bool`|`highSchoolID uint`|`bool`|対象教員が指定高校に所属するかの判定（②「12. Validation設計」整合性チェック：更新対象の教員が対象高校に所属しているかどうか）|

不変条件（`NewTeacher`で保証する内容）:

- HighSchoolID・Name・Emailは空値（ゼロ値）を許容しない（`UserRoleID`は`NewTeacher`の引数に含めない。`NameKana`は`Name`と同じ値とする。教師視点の新規作成における氏名カナの必須・形式検証は教師教員管理機能_Go実装仕様書「3. Domain層設計」が定める）

### TeacherAccountOptions（`internal/teacher_management/teacher.go`）

教員の作成時に、`user` Contextの`CreateTeacherAccount`へ渡す、経路ごとに異なる指定値を表す型である。`Teacher`が持つ氏名・氏名カナ・メールアドレス・所属校ID以外の入力を保持する。

|フィールド|型|意味|
|-|-|-|
|GradeID|uint|`users.grade_id`に設定する学年ID（0は未設定）。教師視点の新規作成は指定した学年ID、管理者視点の招待は未設定（0）|
|InvitationPending|bool|招待待ち状態（`password_reset_required`）で作成するか。教師視点は`true`、管理者視点は`false`|
|SendInvitation|bool|作成時に招待メールを送信するか。教師視点は`false`（招待メールは教員招待通知機能が後から送る）、管理者視点は`true`|

経路ごとの指定値は、`user`②「12. UseCase設計」の経路ごとの指定値と同じである（管理者による教員作成＝招待待ち`false`・送信`true`、教師による教員作成＝招待待ち`true`・送信`false`）。

### TeacherPermission（`internal/teacher_management/teacher_permission.go`）

②「7. Value Object設計」TeacherPermissionの採用理由・独自ルールを反映する。`teacher_permissions`テーブルに対応する、教員に1:1で従属する属性群として実装する。教師教員管理機能②「6. Entity設計」の（着任時の初期権限としての）TeacherPermissionと同一の概念であり、教師視点の機能もこのstructを共有する。

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

②「6. Entity設計」TeacherGradeAssignmentの責務、②「7. Value Object設計」GradeAssignmentSetの採用理由・独自ルールを反映する。教師教員管理機能②「6. Entity設計」のTeacherGradeAssignmentと同一のEntityであり、教師視点の機能もこのstructを共有する（教師視点の新規作成では、単一の`grade_id`を要素とする`GradeAssignmentSet`として扱う）。

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
    // AllGradeIDsForHighSchool は、指定高校の全学年のIDを返す
    // （管理者視点の招待で、grade_scopeが全学年の場合に用いる）。
    AllGradeIDsForHighSchool(ctx context.Context, highSchoolID uint) ([]uint, error)
}
```

これらのinterfaceの実装（HighSchool Context・Grade Context側のStore）は本書の対象外である（**②からの補足**。詳細は「14. ②からの補足事項」参照）。

教員アカウント本体の作成は、`user` Contextが公開する`CreateTeacherAccount`（`user`②「12. UseCase設計」）を、同じく利用側で定義するinterface`TeacherAccountCreator`を介して呼び出す。`TeacherAccountCreator`の実装は`user` Context側の公開関数であり、本書の対象外である。

|型|内容|
|-|-|
|`TeacherAccountCreator`|メソッド`CreateTeacherAccount(ctx context.Context, input TeacherAccountInput) (uint, error)`を持つ。作成された教員のID（`user`②の出力である基本属性の`id`）を返す。`ctx`は、呼び出し側（`TeacherStore`）が開始したトランザクションを`user` Contextの作成操作へ引き継ぐために用いる（`user`②「14. Transaction設計」。引き継ぎの具体的な方式は`user`②の未解決の論点9で、規約11の`context.Context`を介した引き継ぎを前提とする）。招待メール送信依頼の登録（`user` Context側の`jobs`への登録）も、このトランザクションに含まれる|
|`TeacherAccountInput`|`user`②の`CreateTeacherAccount`の入力に対応する。フィールドは、氏名（`Name`）・氏名カナ（`NameKana`）・メールアドレス（`Email`）・所属校ID（`HighSchoolID`）・学年ID（`GradeID`。0は未設定）・招待待ち状態で作成するか（`InvitationPending`）・作成時に招待メールを送信するか（`SendInvitation`）|

`user` Contextが返すValidationエラー（メールアドレスの形式不正・重複、氏名・氏名カナの必須・文字数）は、本Contextで別のエラーへ変換せず、`user` Contextのエラー種別のまま呼び出し元（Handler）へ返し、422として扱う（「11. Error実装方針」）。

## Domain Service

②「8. Domain Service」TeacherGradeAssignmentPolicyを（教師教員管理機能②「8. Domain Service」の同名Policyと共通のものとして）、独立したstruct/interfaceではなく、`internal/teacher_management/teacher_grade_assignment_policy.go`内のpackageレベル関数として実装する（規約4章 Active Record節に従いDomain Serviceは原則「対象外」とするための構造上の読み替え）。

|関数|引数|戻り値|責務|
|-|-|-|-|
|`ValidateGradeAssignment`|`(requestedGradeIDs []uint, allowedGradeIDsForHighSchool []uint) (invalidGradeIDs []uint, ok bool)`|`([]uint, bool)`|要求された学年ID集合のうち、対象高校に属さないものを判定する（②「8. Domain Service」の判定内容そのもの。`allowedGradeIDsForHighSchool`は`GradeReferenceChecker.ExistingGradeIDsForHighSchool`の結果をHandler側で取得して渡す）|

②「8. Domain Service」の「追加で必要としないService」（権限属性の更新自体は単純な値の置き換え）の判断は変更しない。

## Domain Event

②「15. Domain Event」により、本機能ではDomain Eventを採用しない。「対象外」とする。招待メールは`user` Contextの`CreateTeacherAccount`が送信依頼として登録するため、本Contextにメール送信のためのイベント・購読者は持たない。監査ログ記録は②の時点で要件化されておらず、本書でも追加しない。

## Domain Error

規約4章「Active Record」節に従い、「struct/Storeが返すエラー」として`internal/teacher_management/errors.go`に`sentinel error`を定義する。

|変数名|発生条件|対応する②の記載|
|-|-|-|
|`ErrHighSchoolNotFound`|指定`high_school_id`が存在しない|②「14. Error設計」Application Error：高校未存在|
|`ErrTeacherNotFound`|指定教員が存在しない、または対象高校に所属しない|②「14. Error設計」Application Error：教員未存在・対象教員が指定高校に所属しない|
|`ErrGradeNotInHighSchool`|指定`grade_ids`に対象高校に属さない学年が含まれる（教師視点の新規作成で指定された`grade_id`が操作者の所属校に属さない、または存在しない場合も同一のエラーで表す）|②「14. Error設計」Domain Error：他校の学年を担当学年として指定した場合。教師教員管理機能②「14. Error設計」Domain Error：指定学年が同校でない|
|`ErrInvalidGradeScope`|`grade_scope`が許容範囲外の値である|②「7. Value Object設計」TeacherPermission独自ルール（grade_scopeは許容される範囲値のみを受け付ける）。教師教員管理機能②「14. Error設計」のgrade_scope許容値違反も同一のエラーで表す|
|`ErrInvalidNameKana`|氏名カナが空、またはカタカナ・長音・中黒・空白以外を含む（教師視点の新規作成で発生。発生元は教師教員管理機能_Go実装仕様書「3. Domain層設計」）|教師教員管理機能②「7. Value Object設計」NameKana|
|`ErrInvalidEmail`|メールアドレスが一般的な形式でない（教師視点の新規作成で発生。発生元は教師教員管理機能_Go実装仕様書「3. Domain層設計」）|教師教員管理機能②「7. Value Object設計」Email|
|`ErrTeacherPermissionNotFound`|`TeacherPermissionStore.FindByUserID`が、指定`user_id`の権限レコードを取得できない（権限レコードが存在しない教員）。Rails現行は、権限レコードが存在しない場合に特別な扱いをしないため、呼び出し元ごとに次のとおり扱う。一覧・詳細では権限項目を値なし（`null`）として返す（エラーにしない）。教師視点の新規作成での操作者の権限確認、管理者視点の更新での権限項目・担当学年の更新では、想定外の状態として500に変換する（403にはしない）|Rails現行の`Teacher::TeachersController#create`（`current_user.teacher_permission`が存在しない場合に例外となり500）・`Admin::UpdateTeacherService`（対象教員の権限レコードが存在しない場合に例外となり500）。②「14. Error設計」Infrastructure Errorに対応|
|`ErrManageOtherTeachersRequired`|操作者（current user）が他職員操作権限（`manage_other_teachers`）を保持しない状態で教員の新規作成を試みた（教師視点。発生箇所は教師教員管理機能_Go実装仕様書「9. Presentation層設計」）|教師教員管理機能②「14. Error設計」Forbidden（Application Error）|

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
func NewTeacherStore(db *gorm.DB, accountCreator TeacherAccountCreator) *TeacherStore
```

`accountCreator`は、教員アカウント本体の作成（`user` Contextの`CreateTeacherAccount`）を呼び出すための依存である（「3. Domain層設計」`TeacherAccountCreator`）。`TeacherStore`は`users`の行を直接作成しない。

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindAllByHighSchool`|`(ctx context.Context, highSchoolID uint)`|`([]*Teacher, error)`|`high_school_id`一致かつ`users`と`user_roles`を結合し`user_roles.name`が教員を表す値であること（`users.user_role_id = user_roles.id`による結合）を条件に、論理削除されていない教員一覧を、ページングなしで全件、`id`昇順で取得する（②「9. Repository設計」`high_school_id`による絞り込み。Rails現行の管理者視点の一覧はページングせず、並び順を指定しないため、順序が不定にならないよう`id`昇順とする。②からの補足）。管理者視点の一覧取得で利用する|
|`FindPageByHighSchool`|`(ctx context.Context, highSchoolID uint, page, perPage int)`|`([]*Teacher, PageInfo, error)`|`high_school_id`一致かつ`users`と`user_roles`を結合し`user_roles.name`が教員を表す値であること（`users.user_role_id = user_roles.id`による結合）を条件に、論理削除されていない教員を対象として、`name_kana`昇順でソートし、ページ番号（`page`）と1ページあたり件数（`perPage`）に応じたオフセット・件数制限を適用して取得する（`page` / `perPage`は、呼び出し側のHandlerが、既定値・上限を適用して正規化した値を渡す。教師教員管理機能_Go実装仕様書「9. Presentation層設計」）。教師視点の一覧取得（教師教員管理機能②「9. Repository設計」TeacherStore：氏名カナ順・ページネーション）で利用する。`PageInfo`は`Page` / `PerPage` / `TotalPages` / `TotalCount`（いずれも`int`）を持つ本packageの型である|
|`FindByIDForHighSchool`|`(ctx context.Context, id, highSchoolID uint)`|`(*Teacher, error)`|`id`と`high_school_id`の一致かつ`users`と`user_roles`を結合し`user_roles.name`が教員を表す値であること（`users.user_role_id = user_roles.id`による結合）の条件で1件取得する所属高校スコープ検索（②「9. Repository設計」`id`による単一取得（所属高校スコープ付き））。該当なしは`ErrTeacherNotFound`を返す。管理者視点の更新対象取得・教師視点の詳細取得の双方で利用する|
|`CreateWithPermissionAndGrades`|`(ctx context.Context, t *Teacher, options TeacherAccountOptions, permission *TeacherPermission, gradeAssignmentSet *GradeAssignmentSet)`|`(*Teacher, error)`|1トランザクションで、次を順に実行する。(1) `TeacherAccountCreator.CreateTeacherAccount`を、`t`（氏名・氏名カナ・メールアドレス・所属校ID）と`options`（学年ID・招待待ち状態で作成するか・作成時に招待メールを送信するか）から組み立てた`TeacherAccountInput`で呼び出し、`users`の1行の作成（教員ロールの解決を含む）と、指定に応じた招待メール送信依頼の登録を`user` Contextに任せる。トランザクション用の`*gorm.DB`を保持する`ctx`を渡し、`user` Contextの作成操作を同一トランザクションに参加させる。(2) 返された教員のIDを`permission.UserID`に設定し、初期TeacherPermissionを作成する。(3) `gradeAssignmentSet`の担当学年を作成する。(4) 作成した教員を、同一トランザクション内で`users`から取得して返す（`user`②の作成結果には作成日時・更新日時が含まれないため）。管理者視点の招待（②「10. UseCase設計」InviteTeacher）・教師視点の新規作成（教師教員管理機能②「10. UseCase設計」CreateTeacher）の双方で利用する。「8. Transaction実装方針」参照|
|`UpdateWithPermissionAndGrades`|`(ctx context.Context, t *Teacher, permission *TeacherPermission, gradeAssignmentSet *GradeAssignmentSet)`|`error`|Teacherのプロフィール更新・TeacherPermission更新・（`gradeAssignmentSet`が非nilの場合のみ）担当学年の全置換を1トランザクションで実行する（②「10. UseCase設計」UpdateTeacherのトランザクション範囲。「8. Transaction実装方針」参照）|

保持しない責務（②「9. Repository設計」保持しない責務を踏襲）: 権限の妥当性判定・学年の所属確認はStoreに持たせない。

教員ロールの解決・`users`の行の作成・仮パスワードの発行・招待メール送信依頼の登録は、`user` Contextの`CreateTeacherAccount`が行う。`TeacherStore`は`user_roles`を読み取らない（教員判定の検索条件で`user_roles`と結合する場合を除く）。`user` Contextの内部エラー（ロールマスタ不在・`jobs`登録の失敗等）は、Infrastructure Error（500）として扱い、トランザクションはロールバックされる。

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
|`FindByUserID`|`(ctx context.Context, userID uint)`|`(*TeacherPermission, error)`|`user_id`一致で1件取得する（②「9. Repository設計」`user_id`による取得）。該当なしは`ErrTeacherPermissionNotFound`を返す（権限レコードが存在しない場合の扱いは、上記Domain Errorの`ErrTeacherPermissionNotFound`を参照）|
|`Create`|`(ctx context.Context, p *TeacherPermission)`|`error`|新規レコードを作成する（`TeacherStore.CreateWithPermissionAndGrades`内から、`user` Contextが作成した教員の`user_id`に対して同一トランザクションで呼び出される）|
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

`TeacherStore.CreateWithPermissionAndGrades`／`UpdateWithPermissionAndGrades`は、トランザクション用の`*gorm.DB`を`NewTeacherPermissionStore`／`NewTeacherGradeStore`に渡して同一トランザクション内で各Storeのメソッドを呼び出す（Entity ⇔ GORMモデルの変換は不要。同一structをそのまま永続化する）。

Entity ⇔ GORMモデルの変換方針: Active Record採用のため、`Teacher`・`TeacherPermission`・`TeacherGradeAssignment`のstructそのものをGORMの操作対象として扱う。変換処理は設けない。

## 外部連携実装

本Contextは、Mail・Cache・Queue等の外部連携を実装しない。招待メールの送信は、`user` Contextの`CreateTeacherAccount`が、アカウントの作成と同一トランザクションで招待メール送信依頼（`jobs`）として登録する（管理者視点の招待は「作成時に招待メールを送る=する」を指定する）。メール送信基盤・`jobs`は`user` Contextが使うものであり、本Contextは直接呼ばない。「対象外」とする。

---

# 6. Presentation層設計

本書のpresentation層は管理者視点として`internal/teacher_management/presentation/admin/`配下に置く。教師視点のpresentation層は`internal/teacher_management/presentation/teacher/`（教師教員管理機能_Go実装仕様書）に分割されており、同名のstruct（`TeacherHandler`・`TeacherResponse`等）はそれぞれ別packageに属するため衝突しない。

## Handler

### TeacherHandler（`internal/teacher_management/presentation/admin/handler/teacher_handler.go`）

- struct名: `TeacherHandler`（package: `internal/teacher_management/presentation/admin/handler`）
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
6. 取得した各教員について`TeacherPermissionStore.FindByUserID`・`TeacherGradeStore.FindGradeIDsByUserID`で権限・担当学年を取得する。`ErrTeacherPermissionNotFound`の場合は、その教員の`grade_scope` / `manage_other_teachers`を値なし（`null`）として扱い、エラーにしない（Rails現行の`Admin::TeacherSerializer`が権限レコードなしを`null`で返すことに合わせる）
7. 取得結果をResponse DTOへ変換して返す

#### InviteTeacher 処理順序

1. current user（admin）を取得する
2. Request DTOへバインドし、型・必須（`name` / `email` / `grade_scope` / `manage_other_teachers`）・フォーマットを検証する（②「12. Validation設計」Presentation）
3. current adminが`high_school_id`を管理対象としているかを確認する
4. `HighSchoolExistenceChecker.Exists`で対象高校の存在を確認する。存在しない場合は`ErrHighSchoolNotFound`
5. `teacher_management.NewTeacher`でTeacherを生成し（氏名カナには氏名と同じ値が設定される）、`Teacher.Validate()`で必須項目を検証する（②「12. Validation設計」Domain）
6. `teacher_management.NewTeacherPermission`で初期TeacherPermissionを生成する。`grade_scope`が許容範囲外の場合は`ErrInvalidGradeScope`（`UserID`は作成後に`TeacherStore.CreateWithPermissionAndGrades`が設定するため、生成時は未設定とする）
7. 担当学年の集合を決める。`grade_scope`が全学年（`all_grades`）の場合は、`GradeReferenceChecker.AllGradeIDsForHighSchool`で取得した対象高校の全学年（指定された`grade_ids`は使わない）を、それ以外の場合は、指定された`grade_ids`を`teacher_management.NewGradeAssignmentSet`で重複排除した集合とし、`GradeReferenceChecker.ExistingGradeIDsForHighSchool`と`teacher_management.ValidateGradeAssignment`で対象高校に属するかを判定する。属さない学年が含まれる場合は`ErrGradeNotInHighSchool`（UpdateTeacherの手順8・9と同じ判定）
8. `TeacherStore.CreateWithPermissionAndGrades`を、Teacher・`TeacherAccountOptions`（学年ID=未設定（0）・招待待ち=`false`・作成時に招待メールを送信=`true`）・初期TeacherPermission・GradeAssignmentSetで呼び出す（②「10. UseCase設計」InviteTeacher。`users`の行の作成と招待メール送信依頼の登録は、Store内で`user` Contextの`CreateTeacherAccount`に任せる）。`user` Contextが返すValidationエラー（メールアドレスの形式不正・重複、氏名の必須・文字数）はそのまま422とする
9. 作成結果をResponse DTOへ変換して返す

仮パスワードの発行・招待メールの内容とトークンの発行・送信は、`user` Contextの責務であり（`user`②「12. UseCase設計」RequestInvitationEmail）、本書では実装しない。招待メール送信依頼は、Store内のトランザクションのコミット後にのみ実行される。

#### UpdateTeacher 処理順序

1. current user（admin）を取得する
2. パスパラメータ（`high_school_id`・`id`）とRequest DTOをバインドし、型・必須・フォーマットを検証する
3. current adminが`high_school_id`を管理対象としているかを確認する
4. `HighSchoolExistenceChecker.Exists`で対象高校の存在を確認する。存在しない場合は`ErrHighSchoolNotFound`
5. `TeacherStore.FindByIDForHighSchool`で対象教員を取得する。存在しない、または対象高校に所属しない場合は`ErrTeacherNotFound`（②「12. Validation設計」整合性チェック：更新対象の教員が対象高校に所属しているかどうか）
6. プロフィール項目（name・email）が指定されている場合、`Teacher.ApplyProfile`で差し替え・検証する
7. 権限項目（grade_scope・manage_other_teachers）が指定されている場合、既存`TeacherPermission`を`TeacherPermissionStore.FindByUserID`で取得し、`TeacherPermission.Apply`で差し替え・検証する。`grade_scope`が許容範囲外の場合は`ErrInvalidGradeScope`。`FindByUserID`が`ErrTeacherPermissionNotFound`を返した場合は、権限レコードを新規作成せず、想定外の状態として500に変換する（Rails現行の`Admin::UpdateTeacherService`が、権限レコードが存在しない対象教員の権限項目・担当学年（`grade_ids`）の更新で例外となり500になることに合わせる）。権限項目も`grade_ids`も指定されない更新（氏名・メールアドレスのみ）では、権限レコードを参照しない
8. `grade_ids`が指定されている場合、`teacher_management.NewGradeAssignmentSet`で重複排除した集合を作成し、`GradeReferenceChecker.ExistingGradeIDsForHighSchool`で対象高校に属する学年IDを取得する
9. `teacher_management.ValidateGradeAssignment`で指定学年がすべて対象高校に属するか判定する。属さない学年が含まれる場合は`ErrGradeNotInHighSchool`（②「8. Domain Service」TeacherGradeAssignmentPolicy、②「12. Validation設計」業務ルール：指定学年が対象高校に属するかどうか）
10. `TeacherStore.UpdateWithPermissionAndGrades`をTeacher・TeacherPermission・（`grade_ids`指定時のみ）GradeAssignmentSetで呼び出す（②「10. UseCase設計」UpdateTeacher 呼び出すStore：HighSchoolStore→TeacherStore→TeacherPermissionStore→TeacherGradeStore→GradeStoreの順に対応）
11. 更新結果をResponse DTOへ変換して返す

## Request / Response DTO

### Request（`internal/teacher_management/presentation/admin/request/teacher_request.go`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`InviteTeacherRequest`|`Name string`、`Email string`、`GradeScope *string`、`ManageOtherTeachers *bool`、`GradeIDs []uint`|`Name`・`Email`・`GradeScope`・`ManageOtherTeachers`は`binding:"required"`（Rails現行は、`grade_scope` / `manage_other_teachers`の未指定を422とする。`ManageOtherTeachers`は`false`を有効な値として区別するためポインタ型）。`Email`は`binding:"email"`（②「12. Validation設計」必須チェック：作成時のname・email、フォーマットチェック：メールアドレス形式）。`GradeIDs`は任意（省略時は担当学年なし。`GradeScope`が全学年の場合は使わない）|
|`UpdateTeacherRequest`|`Name *string`、`Email *string`、`GradeScope *string`、`ManageOtherTeachers *bool`、`GradeIDs *[]uint`|部分更新のためポインタ型で「未指定」を表現する。`Email`指定時は`binding:"omitempty,email"`。`GradeIDs`は配列形式チェック（②「12. Validation設計」フォーマットチェック：grade_idsの配列形式）|

### Response（`internal/teacher_management/presentation/admin/response/teacher_response.go`）

|struct名|フィールドと型|
|-|-|
|`TeacherResponse`|`ID uint`、`HighSchoolID uint`、`Name string`、`Email string`、`GradeScope *string`、`ManageOtherTeachers *bool`（権限レコードが存在しない教員では`nil`＝JSONの`null`）、`GradeIDs []uint`、`CreatedAt string`、`UpdatedAt string`|
|`TeacherListResponse`|`Teachers []TeacherResponse`|

EntityであるTeacher／TeacherPermission／TeacherGradeAssignmentをそのまま返さず、必ずResponse DTOへ変換する（規約「7. データフロー」）。

## Routing

`internal/teacher_management/presentation/admin/routes.go`

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
|POST|/api/v1/admin/high_schools/:high_school_id/teachers|InviteTeacher|パスパラメータ`high_school_id` + `InviteTeacherRequest`|`TeacherResponse`|201（Rails現行は201を返す。①Rails現行仕様書「5. API / 処理詳細」create）|
|PATCH|/api/v1/admin/high_schools/:high_school_id/teachers/:id|UpdateTeacher|パスパラメータ`high_school_id`・`id` + `UpdateTeacherRequest`|`TeacherResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（Middleware）|
|admin以外のロール|403|認可エラー（Middleware）|
|current adminが管理しない`high_school_id`を指定|403|業務認可エラー（Handler。②「1. 機能概要」利用者の制約に基づく。判定方法は推測。「14. ②からの補足事項」参照）|
|Request DTOの型・必須・フォーマット不正|422|Presentation Validationエラー（②「16. API互換方針」422: 入力・業務ルール違反）|
|`user` Contextが返すValidationエラー（招待時のメールアドレス重複・形式不正、氏名の必須・文字数）|422|`user`②「17. Error設計」のValidationエラー（メールアドレスの使用状況を含めないメッセージ）を`errors`形式で返す|
|`grade_scope`が許容範囲外（`ErrInvalidGradeScope`）|422|業務ルール違反|
|指定`grade_ids`に対象高校に属さない学年が含まれる（`ErrGradeNotInHighSchool`）|422|業務ルール違反（②「14. Error設計」Domain Error：他校の学年を担当学年として指定した場合）|
|指定`high_school_id`が存在しない（`ErrHighSchoolNotFound`）|404|②「16. API互換方針」404: 対象高校不存在|
|対象教員不存在／所属高校不一致（`ErrTeacherNotFound`）|404|②「16. API互換方針」404: 対象教員不存在|
|更新時、対象教員の権限レコードが存在しない状態で、権限項目または`grade_ids`の更新を行った（`ErrTeacherPermissionNotFound`）|500|想定外の状態（Rails現行は例外となり500）|
|DB接続失敗等のInfrastructure Error|500|内部エラー|

---

# 8. Transaction実装方針

②「11. Transaction設計」を実装単位に落とし込む。

## Transaction開始箇所

- Active Record採用のため、`TeacherStore.CreateWithPermissionAndGrades`・`TeacherStore.UpdateWithPermissionAndGrades`メソッド内で`db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })`を用いてトランザクションを開始する（②「11. Transaction設計」Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する）
- `ListTeachers`はトランザクションを使用しない（②「11. Transaction設計」ListTeachersの処理ではトランザクションを使用しない）

## Transaction終了箇所（Commit / Rollback条件）

- `CreateWithPermissionAndGrades`（管理者視点の招待・教師視点の新規作成の双方で利用）: `user` Contextの`CreateTeacherAccount`（`users`の作成と、指定に応じた招待メール送信依頼の登録）・初期TeacherPermission作成・担当学年の作成が全て成功した時点でコミットする（②「11. Transaction設計」InviteTeacher、教師教員管理機能②「11. Transaction設計」）。いずれかが失敗した場合は、`users`の作成と招待メール送信依頼の登録も含めてロールバックする。招待メールは、コミットされた場合にのみ`user` Contextのワーカーが送信する
- `UpdateWithPermissionAndGrades`: プロフィール更新・権限更新・（指定時のみ）担当学年の全置換がすべて成功した時点でコミットする（②「11. Transaction設計」UpdateTeacherの処理では、プロフィール・権限・担当学年の同期が完了した時点でコミットする）。いずれかが失敗した場合はロールバックする

## 複数Storeにまたがる場合の扱い

`TeacherStore.CreateWithPermissionAndGrades`／`UpdateWithPermissionAndGrades`は、トランザクション用の`tx *gorm.DB`を用いて`TeacherPermissionStore`・`TeacherGradeStore`を`NewTeacherPermissionStore(tx)`・`NewTeacherGradeStore(tx)`で生成し、同一トランザクション内で各Storeのメソッドを呼び出す。`user` Contextの`CreateTeacherAccount`は、`tx`を保持する`ctx`を渡して呼び出し、同じトランザクションに参加させる（`user`②「14. Transaction設計」。引き継ぎ方式は`user`②の未解決の論点9のとおり、`user`③で確定するまで、規約11の`context.Context`を介した引き継ぎを前提とする）。`HighSchoolExistenceChecker`・`GradeReferenceChecker`による確認は、トランザクション開始前（Handler側）で完了させ、トランザクション内では行わない（②「9. Repository設計」HighSchoolStore/GradeStoreは「担当学年として妥当かどうかの最終判断」を保持しないとされている点と整合）。

---

# 9. Validation実装方針

②「12. Validation設計」を実装レベルに落とし込む。

## Presentation

- 型チェック: HTTP入力の型を検証する
- 必須チェック: `InviteTeacherRequest`の`Name`・`Email`・`GradeScope`・`ManageOtherTeachers`に`binding:"required"`（②「12. Validation設計」必須チェック：作成時のname・email・grade_scope・manage_other_teachers、高校IDの必須性）
- フォーマットチェック: `Email`は`binding:"email"`、`GradeIDs`は配列形式チェック（②「12. Validation設計」フォーマットチェック：メールアドレス形式、grade_idsの配列形式）

## 業務ルール検証（Active Record採用時: Modelのメソッドで検証する内容）

- `Teacher.Validate()` / `Teacher.ApplyProfile()`: name・email等必須項目の妥当性（招待時のメールアドレスの形式・重複と氏名の文字数は、`user` Contextの作成操作が検証する）
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
|`ErrTeacherPermissionNotFound`（管理者視点の更新で、権限項目または`grade_ids`を更新する対象教員の権限レコードが存在しない場合／教師視点の新規作成で、操作者の権限レコードが存在しない場合）|Store（TeacherPermissionStore）／Handler|500（一覧・詳細の権限項目の参照では、エラーにせず`null`として扱う）|
|`user` ContextのValidationエラー（招待時のメールアドレス重複・形式不正、氏名の必須・文字数）|`user` Context（`TeacherAccountCreator`経由）|422|
|`user` Contextの内部エラー（ロールマスタ不在・`jobs`登録の失敗等）|`user` Context（`TeacherAccountCreator`経由）|500（トランザクションはロールバックされ、権限・担当学年も作成されない）|
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
|`UserRole`|`user_roles`|`user_roles`テーブルを参照専用で読み取るための最小限のモデル（`ID` / `Name`のみ）。テーブルの所有はauthentication Contextであり、本Contextは教員判定の検索条件（`users`との結合）にのみ利用する。教員の作成時のロールの解決は`user` Contextが行う（管理者管理者ユーザー機能_Go実装仕様書「12. GORM / DBクエリ設計」の`UserRoleModel`と同じ扱い）。struct名のデフォルト複数形が実テーブル名と一致するため`TableName()`のオーバーライドは不要。`Name`はRailsの`enum name:`に対応し、DB上は整数（`admin: 0` / `student: 1` / `teacher: 2` / `guardian: 3`）である|
|`TeacherPermission`|`teacher_permissions`|GORMのデフォルト命名規則（struct名の複数形snake_case）で一致するため、`TableName()`のオーバーライドは不要（Gorm規約「複数形のテーブル名」）|
|`TeacherGradeAssignment`|`teacher_grades`|struct名のデフォルト複数形は`teacher_grade_assignments`だが、②「1. 機能概要」「17. DB設計方針」の記載どおり実際のテーブル名は`teacher_grades`であるため、`Tabler`インターフェース（`func (TeacherGradeAssignment) TableName() string { return "teacher_grades" }`）による明示的な上書きが必要（Gorm規約「テーブル名」。**②からの補足**）|

## 主要クエリの条件・ソート・ページネーション方針

- 教員判定の共通条件（`TeacherStore`の検索3メソッド共通）: `users`と`user_roles`を明示的な`Joins`（`users.user_role_id = user_roles.id`）で結合し、`user_roles.name`が教員を表す値（Rails `User.teachers`スコープの`joins(:user_role).where(user_roles: { name: 'teacher' })`に相当）であること。`Teacher`に`user_roles`のアソシエーションフィールドは持たせない（Gorm規約「関連データの取得は`Preload`または明示的な`Joins`を用いる」）。論理削除は`users.deleted_at`が未設定であること（`gorm.DeletedAt`によるソフトデリートの既定の除外条件）
- `TeacherStore.FindAllByHighSchool`: `high_school_id`一致必須、教員判定の共通条件、論理削除されていないこと（②「9. Repository設計」保持する検索機能）。ページネーションなし（全件）、`id`昇順（Rails現行の管理者視点の一覧はページングせず並び順を指定しない。順序が不定にならないよう`id`昇順とする。②からの補足）
- `TeacherStore.FindPageByHighSchool`（教師視点）: `high_school_id`一致必須、教員判定の共通条件、論理削除されていないこと。`name_kana`昇順、`page` / `perPage`に基づくオフセット・件数制限（1ページあたり件数の既定値は20件・上限は100件。教師教員管理機能②「10. UseCase設計」ListTeachers、Rails現行の`ApplicationController`の共通設定）
- `TeacherStore.FindByIDForHighSchool`: `id`かつ`high_school_id`一致、教員判定の共通条件
- `TeacherPermissionStore.FindByUserID`: `user_id`一致（該当なしは`ErrTeacherPermissionNotFound`）
- `TeacherGradeStore.FindGradeIDsByUserID`: `user_id`一致
- `TeacherGradeStore.ReplaceAll`: `user_id`一致条件での削除、続けて指定`grade_ids`分の一括作成（`TeacherStore.CreateWithPermissionAndGrades`内では、`user` Contextが作成した教員に対する担当学年の作成としても利用する）

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
|`TeacherStore.FindAllByHighSchool`|`high_school_id`一致の教員のみが、ページングなしで全件、`id`昇順で取得されること|
|`TeacherStore.FindByIDForHighSchool`|所属高校が異なる教員は取得できず`ErrTeacherNotFound`となること|
|`TeacherStore.CreateWithPermissionAndGrades`|`user` Contextの`CreateTeacherAccount`（テスト用のスタブ）を、管理者視点の指定（招待待ち=`false`・作成時に招待メールを送信=`true`・学年ID=未設定・氏名カナ=氏名）で呼び出すこと。返された教員IDでTeacherPermissionと担当学年が作成されること|
|`TeacherStore.CreateWithPermissionAndGrades`|TeacherPermission作成または担当学年の作成が失敗した場合に、`user` Contextが作成した`users`の行と招待メール送信依頼の登録もロールバックされること／`user` Contextがエラー（メールアドレス重複等）を返した場合に、権限・担当学年が作成されないこと|
|`TeacherStore.FindAllByHighSchool` / `FindPageByHighSchool` / `FindByIDForHighSchool`|`user_roles`結合の結果、教員以外（`student` / `admin` / `guardian`）のレコード・論理削除されたレコードが取得されないこと|
|`TeacherStore.FindPageByHighSchool`|他校の教員が混入しないこと／`name_kana`昇順でソートされること／`page` / `perPage`の指定に応じた件数・オフセットで取得され、`PageInfo`に`Page` / `PerPage` / `TotalPages` / `TotalCount`が返ること|
|`TeacherPermissionStore.FindByUserID`|権限レコードが存在する教員で取得できること／存在しない教員で`ErrTeacherPermissionNotFound`が返ること|
|`TeacherStore.CreateWithPermissionAndGrades`（教師視点の指定）|招待待ち=`true`・作成時に招待メールを送信=`false`・学年ID=指定した学年で`CreateTeacherAccount`を呼び出し、Teacher・TeacherPermission・担当学年の3レコードが同一トランザクションで全て作成されること|
|`TeacherStore.UpdateWithPermissionAndGrades`|`grade_ids`指定時に既存担当学年が全置換されること|
|`TeacherStore.UpdateWithPermissionAndGrades`|`grade_ids`未指定時に既存担当学年が変更されないこと|
|`TeacherGradeStore.ReplaceAll`|既存紐付けの削除と新規紐付けの作成が一貫して行われること|

## Handler Test

|対象|テストケース|
|-|-|
|`ListTeachers`|存在しない`high_school_id`で404が返ること|
|`ListTeachers`|current adminが管理しない`high_school_id`で403が返ること|
|`InviteTeacher`|`email`欠落・不正フォーマットで422が返ること|
|`InviteTeacher`|`grade_scope`が許容範囲外の場合に422が返ること／`name`・`grade_scope`・`manage_other_teachers`の欠落で422が返ること／`grade_ids`に他校の学年が含まれる場合に422が返ること／`user` ContextがメールアドレスのValidationエラーを返した場合に422が返ること／`grade_scope`が全学年の場合に対象高校の全学年が担当学年として作成されること|
|`UpdateTeacher`|存在しない教員idで404が返ること|
|`UpdateTeacher`|`grade_ids`に他校の学年が含まれる場合に422が返ること|
|全Handler|未認証・非admin roleでのアクセスが401/403となること|

## Integration Test

|対象|テストケース|
|-|-|
|教員招待〜一覧取得〜更新|一連のエンドポイント呼び出しが正常に完了し、権限・担当学年の反映結果が一覧・更新レスポンスに一貫して反映されること（②「18. テスト戦略」Integration Test：エンドポイント経由で一覧・招待・更新が正常に動作し、他校の学年が拒否されることを確認する）|
|教員招待と`user` Contextの連携|招待で作成された教員が招待待ちにならないこと（教員招待通知機能の招待未完了一覧に現れない）／招待メール送信依頼が`jobs`に登録され、権限・担当学年の作成に失敗した場合は`users`の行と依頼が残らないこと（`user`②「22. テスト戦略」Integration Test 教員管理）|
|他校の学年指定|`grade_ids`に他校の学年を指定した更新が拒否されること（②「18. テスト戦略」Integration Test）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測か否か|
|-|-|-|
|Bounded Context `teacher-management`（教師教員管理機能と共通）の内部ディレクトリ名を`internal/teacher_management`とした|②はContext名（teacher-management）のみを記載し、内部ディレクトリ名を明記していない。`規約/アーキテクチャ規約.md`「9. 命名規約」の「Context名とディレクトリ名が一致しない場合は②文書内で対応関係を明記する」に該当する明記がないため、既存の他機能（student-directory→internal/student_directory）の対応方針にならい、Context名のハイフンをアンダースコアに置き換えたディレクトリ名を採用した|推測（既存の他②③文書での対応付けからの類推であり、②文書内に明示的な対応付けの記載はない）|
|②「9. Repository設計」HighSchoolStore・GradeStoreをteacher-management内で実装せず、teacher_management package側に参照用interface（`HighSchoolExistenceChecker`／`GradeReferenceChecker`）を定義し、実装はHighSchool Context・Grade Context側に委ねる構成とした|②「9. Repository設計」はHighSchoolStore・GradeStoreをteacher-managementの節内で記載しているが、`規約/アーキテクチャ規約.md`「6. Context間連携ルール」により他Contextの内部実装に直接依存できないため、コーディング規約「5. インターフェース」（利用側での定義）に従って整理した。②の「責務」「保持しない責務」の記載内容自体は変更していない|補足（規約の適用による構造上の具体化であり、推測ではなく規約遵守のための判断）|
|②「8. Domain Service」TeacherGradeAssignmentPolicyを、独立したstruct/interfaceではなく`internal/teacher_management`パッケージ内のpackageレベル関数`ValidateGradeAssignment`として実装した|`規約/アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」Active Record節により、Domain Serviceに相当する独立層は原則「対象外」とされているため。判定内容そのものは②の記載を変更していない|補足（規約適用）|
|②「7. Value Object設計」TeacherPermission・GradeAssignmentSetを、独立層としてではなくstruct・メソッドとして`internal/teacher_management`パッケージ内に実装した|同上、規約4章Active Record節により、Value Objectは原則「対象外」とされているため|補足（規約適用）|
|`Teacher`のGORMテーブル名を`users`として`TableName()`で明示的に上書きする必要がある点、および`TeacherGradeAssignment`のGORMテーブル名を`teacher_grades`として同様に上書きする必要がある点|②「17. DB設計方針」はテーブル名`users`・`teacher_grades`を利用する旨のみ記載しており、Go構造体名のデフォルト複数形との不一致には触れていない。Gorm規約「テーブル名」に基づき③側で明示した|補足（Gorm規約の適用によりテーブル不一致を検出したもの）|
|`Teacher`structが`users`テーブルの一部カラム（管理者視点・教師視点の教員管理に必要な最小限のフィールド）のみを保持し、パスワードハッシュ等他Contextが管理するカラムを含めていない点|Rails `db/schema.rb`の`users`テーブルは`encrypted_password` / `jti` / `reset_password_token` / `address_id` / `grade_id` / `school_class_id` / `student_number`等、教員管理の対象外であるカラムを多数持つ。教員管理の業務範囲（②「6. Entity設計」Teacher、教師教員管理機能②「6. Entity設計」Teacher）に必要な範囲（`id` / `high_school_id` / `name` / `name_kana` / `email` / `user_role_id` / `deleted_at` / `created_at` / `updated_at`）のみをフィールド化した。`name` / `name_kana`は`limit: 100`でNULL許容、`high_school_id` / `user_role_id`もNULL許容のカラムである|補足（Rails `db/schema.rb`で確認済みの事実に基づく取捨選択）|
|`grade_scope`の具体的な許容値集合を`own_grade` / `all_grades`とした|②「7. Value Object設計」は「許容される範囲値のみを受け付ける」とのみ記載し、具体的な値集合を明記していない。①Rails現行仕様書・教師教員管理機能②「7. Value Object設計」GradeScopeの`own_grade` / `all_grades`に従う|補足（①・教師教員管理機能②に基づく）|
|current adminが対象`high_school_id`を管理しているかどうかの具体的な確認方法（current adminのHighSchoolIDとpathパラメータの一致確認と仮定した）|②「1. 機能概要」利用者は「対象は自身が管理する高校に所属する教員に限定される」と記載するのみで、管理者と高校の対応関係の具体的なデータモデル・確認方法を明記していない。①も未提供のため参照不可。教員同様adminユーザーもHighSchoolIDを保持するという前提で設計した|推測（実装着手前にadminユーザーと高校の対応関係の確認が必要）|
|`InviteTeacherRequest`の`GradeScope`・`ManageOtherTeachers`を必須とし、デフォルト値を持たせない|①Rails現行仕様書「5. API / 処理詳細」createのとおり、Rails現行は`grade_scope` / `manage_other_teachers`の未指定を422とする（権限レコードの作成で必須検証に失敗する）|補足（①に基づく）|
|作成時（InviteTeacher）のHTTP Status Codeを201とした|②「16. API互換方針」Status Codeの記載は「201」であり、Rails現行も201を返す（①Rails現行仕様書 createのResponse）|補足（①に基づく）|
|`TeacherStore.CreateWithPermissionAndGrades`／`UpdateWithPermissionAndGrades`というメソッド名・トランザクションの起点をStoreの単一メソッドに集約する具体的な設計|②「11. Transaction設計」は「Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する」という方針のみを記載しており、具体的なメソッド名・分割単位は指定していない|補足（②の方針を実装可能な粒度に具体化したもの）|
|`TeacherStore.FindAllByHighSchool`を、ページングなし・`id`昇順とした|Rails現行の管理者視点の一覧は、ページングを行わず並び順を指定しない（①「管理者教員管理機能」）。ページングなしはRails現行のとおり、`id`昇順は、順序が不定にならないための実装上の判断（②「9. Repository設計」）|ページングなしはRails現行の反映（推測ではない）。`id`昇順は②からの補足（業務ルールの追加ではない）|
|presentation層を`admin`/`teacher`サブパッケージへ分割した（`internal/teacher_management/presentation/admin/`が本書、`internal/teacher_management/presentation/teacher/`が教師教員管理機能_Go実装仕様書）|本機能は同一Aggregateを管理者視点（本書）・教師視点（教師教員管理機能）という2つのHandler群が操作する構成を持ち、同名のstruct・ファイル（`TeacherHandler`・`teacher_handler.go`・`TeacherResponse`・`TeacherListResponse`・`routes.go`）が衝突する。面談機能_Go実装仕様書・教師面談機能_Go実装仕様書の`student`/`teacher`サブパッケージ分割と同じ方針で解消した|補足（規約に反しない範囲での構造上の判断であり、②の設計方針自体は変更していない）|
|`Teacher`が`NameKana` / `UserRoleID` / `DeletedAt`を保持し、`TeacherStore`の検索メソッドが`user_roles`との結合による教員ロール条件・論理削除されていないこと、という条件を含む構成とした|教師教員管理機能②「9. Repository設計」が`teacher`ロールでの絞り込み・氏名カナ順ソートを、同②「6. Entity設計」が氏名カナを、それぞれTeacherの属性・検索条件として定めており、同一の`users`テーブルを扱うTeacherを両視点で共有するにはこれらを備える必要がある。Rails `db/schema.rb`の`users`テーブルにロール名を持つ列はなく、`user_role_id`（`user_roles`への外部キー）で役割を参照する。`user_roles.name`はRails `app/models/user_role.rb`の`enum name:`（`admin: 0` / `student: 1` / `teacher: 2` / `guardian: 3`）に対応する整数列であり、Railsの`User.teachers`スコープも`joins(:user_role).where(user_roles: { name: 'teacher' })`で教員を判定している。Go側も同様に`user_roles`との結合で判定する。教員ロール条件は管理者視点の検索にも適用する（②「9. Repository設計」はTeacherを「教員としてのUser」と定義しているため）。`name_kana`（100文字以内）・`deleted_at`（論理削除）も`users`テーブルに実在する|補足（教師教員管理機能②の設計意図をModel・Storeに反映したもの。`users.user_role_id` / `name_kana` / `deleted_at`および`user_roles.name`の定義はRails `db/schema.rb`・`app/models/user_role.rb`で確認済み）|
|教員作成時の`users`の行の作成（教員ロールのIDの解決・仮パスワードの発行・招待メール送信依頼の登録を含む）を`user` Contextの`CreateTeacherAccount`に任せ、`TeacherStore`は`TeacherAccountCreator`を介して呼び出したうえで、権限・担当学年を同一トランザクションで作成する構成とした。`TeacherAccountOptions`（学年ID・招待待ち・作成時メール）で管理者視点・教師視点の指定値の差を表す|`user`②「12. UseCase設計」「設計差分管理」の影響範囲が、教員管理②のActive Recordの`TeacherStore`による`users`の直接作成を`user`の作成操作の呼び出しへ置き換えると定めているため。トランザクションの引き継ぎ方式は`user`②の未解決の論点9であり、規約11の`context.Context`を介した引き継ぎを前提とした|推測（引き継ぎ方式・`TeacherAccountCreator`の型は`user`③で確定するまで暫定）|
|`TeacherStore`が教師視点用の`FindPageByHighSchool`を持ち、新規作成の`CreateWithPermissionAndGrades`を管理者視点・教師視点で共有する構成とした|教師視点の一覧取得（氏名カナ順・ページネーション）は、管理者視点の`FindAllByHighSchool`（ページネーションなし）では満たせない。新規作成は、管理者視点（招待）も、Rails現行で権限と担当学年を作成するため、どちらの視点でも`users`（`user` Context）・権限・担当学年の3つの作成となり、指定値（`TeacherAccountOptions`）のみが異なる。Storeを二重に定義しないため、共有するStoreに最小限のメソッドとして持たせた|補足（教師教員管理機能②の設計意図・①Rails現行仕様書をStoreメソッドとして具体化したもの）|
|`TeacherHandler`のコンストラクタ・依存を管理者視点のHandlerに限定し、`TeacherNotificationStatusProvider`（招待通知状況の参照）等の教師視点固有の依存を含めていない|招待通知状況の付与は教師視点のみの処理であり、共有するStoreに`teacher-notification` Contextへの依存を持ち込まないため、教師教員管理機能_Go実装仕様書のHandler側で扱う|補足|

---
