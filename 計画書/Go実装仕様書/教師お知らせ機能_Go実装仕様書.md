# 教師お知らせ機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

教師が対象ユーザー向けにお知らせを作成・公開・閲覧する機能である。お知らせは `draft`（下書き）→ `scheduled`（公開予約）または `published`（公開）という状態遷移を持ち、`scheduled` の場合はさらに `published` へ遷移する。お知らせには対象条件（全ユーザー・権限別・学年別・学校別・個人別）を複数指定でき、閲覧側は対象条件に自身が合致するお知らせのみを閲覧できる。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」より、本機能は **Domain Model** を採用する。

- Announcementは `draft/scheduled/published` という明確な状態と、状態に付随する整合性ルール（`scheduled` では未来日時必須、`published` 遷移時に公開日時を自動確定）を持つ
- 対象指定（AnnouncementTarget）は `own_grade` 権限制約・同校制約という複数Entity・複数Contextにまたがる業務ルールを持つ
- これらの業務ルールをEntity/Domain Serviceに集約することで、UseCase・Handlerを薄く保つ

Transaction Script／Active Record／Event Sourcingは②「4. 設計パターン」内で不採用と判断されており、本書もその判断を変更しない。

## 本書が対象とする実装範囲

- Bounded Context: `announcement`
- 対象UseCase: お知らせ一覧取得・詳細取得・作成・状態更新（②「10. UseCase設計」に対応する4つ）
- 対象外: Domain Event実装（②「15. Domain Event」で未採用と明記）、非同期通知連携（本機能スコープ外）
- ①Rails実装の詳細は本書作成時点で未提供のため、参照が必要な箇所は「①未提供のため参照不可」と明記する

---

# 2. ディレクトリ構成

## 対象Bounded Context名

`announcement`（②「3. Bounded Context」のContext名をそのまま採用。アーキテクチャ規約「5. Bounded Context構成」の上位ドメイン `Announcement` に対応し、既存Context一覧表の `announcement` と一致する）

## ②で採用した設計パターン

Domain Model

## 採用パターンに対応する構造

アーキテクチャ規約「2. ディレクトリ構成」「4. 設計パターンごとの構造適用方針（Domain Model）」に従い、標準のフルレイヤー構成を適用する。ただし本機能で不要なディレクトリは作成しない。

- `domain/event/`: 対象外（②「15. Domain Event」でDomain Event不採用のため作成しない）
- `infrastructure/mail/`, `infrastructure/cache/`, `infrastructure/queue/`: 対象外（②のスコープに通知連携・キャッシュ・非同期処理の記載がないため作成しない）

## 作成するディレクトリ一覧

```
internal/announcement/
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

## 作成するファイル一覧

```
internal/announcement/domain/entity/announcement.go
internal/announcement/domain/entity/announcement_target.go

internal/announcement/domain/valueobject/announcement_status.go
internal/announcement/domain/valueobject/scheduled_at.go
internal/announcement/domain/valueobject/published_at.go
internal/announcement/domain/valueobject/target_criteria.go
internal/announcement/domain/valueobject/viewer_context.go

internal/announcement/domain/repository/announcement_repository.go
internal/announcement/domain/repository/grade_reference_repository.go
internal/announcement/domain/repository/user_reference_repository.go
internal/announcement/domain/repository/teacher_permission_reference_repository.go

internal/announcement/domain/service/announcement_targeting_policy.go
internal/announcement/domain/service/announcement_visibility_policy.go

internal/announcement/domain/errors/errors.go

internal/announcement/application/dto/list_announcements_dto.go
internal/announcement/application/dto/show_announcement_dto.go
internal/announcement/application/dto/create_announcement_dto.go
internal/announcement/application/dto/update_announcement_status_dto.go
internal/announcement/application/dto/pagination_dto.go

internal/announcement/application/usecase/list_announcements_usecase.go
internal/announcement/application/usecase/show_announcement_usecase.go
internal/announcement/application/usecase/create_announcement_usecase.go
internal/announcement/application/usecase/update_announcement_status_usecase.go
internal/announcement/application/usecase/errors.go

internal/announcement/infrastructure/persistence/gorm/announcement_model.go
internal/announcement/infrastructure/persistence/gorm/announcement_target_model.go

internal/announcement/infrastructure/repository/announcement_repository.go
internal/announcement/infrastructure/repository/grade_reference_repository.go
internal/announcement/infrastructure/repository/user_reference_repository.go
internal/announcement/infrastructure/repository/teacher_permission_reference_repository.go

internal/announcement/presentation/handler/announcement_handler.go
internal/announcement/presentation/request/list_announcements_request.go
internal/announcement/presentation/request/create_announcement_request.go
internal/announcement/presentation/request/update_announcement_status_request.go
internal/announcement/presentation/response/announcement_list_response.go
internal/announcement/presentation/response/announcement_detail_response.go
internal/announcement/presentation/response/announcement_create_response.go
internal/announcement/presentation/routes.go
```

**②からの補足**: `domain/repository/`配下の`grade_reference_repository.go` `user_reference_repository.go` `teacher_permission_reference_repository.go`は、②「9. Repository設計」の「外部参照Repository（Grade / User / UserRole / TeacherPermission）」を、コーディング規約「5. インターフェース」（インターフェースは利用側で定義する）に従い、announcementコンテキスト側の利用インターフェースとして具体化したものである。実際の実装（infrastructure側）は、School/User/Teacher Permission各Contextが公開する参照手段を呼び出すアダプタとする（アーキテクチャ規約「6. Context間連携ルール」）。具体的なメソッド名・戻り値は②に記載がないため推測で補った。

---

# 3. Domain層設計

## Entity

### Announcement（Aggregate Root）

- struct名: `Announcement`
- 保持するフィールドと型:

|フィールド|型|意味|
|-|-|-|
|id|uint|お知らせID|
|publisherID|uint|発信者（教師）のユーザーID|
|title|string|タイトル|
|content|string|本文|
|status|valueobject.AnnouncementStatus|現在の状態（draft/scheduled/published）|
|scheduledAt|*valueobject.ScheduledAt|公開予定日時（`scheduled`のときのみ設定）|
|publishedAt|*valueobject.PublishedAt|公開確定日時（`published`のときのみ設定）|
|targets|[]*AnnouncementTarget|対象指定群（Aggregate内の子Entity）|

- 公開method一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|ID|-|uint|IDを返す|
|PublisherID|-|uint|発信者IDを返す|
|Title|-|string|タイトルを返す|
|Content|-|string|本文を返す|
|Status|-|valueobject.AnnouncementStatus|現在状態を返す|
|ScheduledAt|-|*valueobject.ScheduledAt|公開予定日時を返す|
|PublishedAt|-|*valueobject.PublishedAt|公開確定日時を返す|
|Targets|-|[]*AnnouncementTarget|対象指定群を返す|
|IsOwnedBy|userID uint|bool|指定ユーザーが発信者本人かどうかを判定する|
|Schedule|scheduledAt time.Time|error|`draft`→`scheduled`へ遷移し、`ScheduledAt`の未来日時検証を行う。許可されない遷移の場合はDomain Errorを返す|
|Publish|-|error|`draft`または`scheduled`→`published`へ遷移し、`PublishedAt`未設定時は現在時刻を確定する。許可されない遷移の場合はDomain Errorを返す|

- 不変条件（コンストラクタ／ファクトリで保証する内容）:
  - ファクトリ関数 `NewAnnouncement(publisherID uint, title, content string, targets []*AnnouncementTarget) (*Announcement, error)` で生成する
  - 生成時の状態は必ず `draft` とする（②「6. Entity設計」ライフサイクル: 作成（draft）→…）
  - `targets` が空の場合は生成不可とする（②「5. Aggregate設計」: 対象未指定は作成不可）
  - `Schedule` / `Publish` は `valueobject.AnnouncementStatus.CanTransitionTo` による許可された遷移のみを受け付ける（②「7. Value Object設計」AnnouncementStatus）

### AnnouncementTarget（Aggregate内の子Entity）

- struct名: `AnnouncementTarget`
- 保持するフィールドと型:

|フィールド|型|意味|
|-|-|-|
|id|uint|対象指定ID|
|announcementID|uint|従属するAnnouncementのID|
|criteria|valueobject.TargetCriteria|対象種別と参照先の組|

- 公開method一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|ID|-|uint|IDを返す|
|Criteria|-|valueobject.TargetCriteria|対象条件を返す|

- 不変条件: ファクトリ関数 `NewAnnouncementTarget(criteria valueobject.TargetCriteria) (*AnnouncementTarget, error)` で生成する。`criteria`が`TargetCriteria`のバリデーション（後述）を通過していることを前提とする。②「6. Entity設計」のとおり、作成後の更新APIは存在しないため更新用methodは設けない。

## Value Object

### AnnouncementStatus

- struct名: `AnnouncementStatus`（内部は`string`をラップ）
- 保持するフィールド: 内部値（draft/scheduled/publishedのいずれか）
- 生成時に検証するルール: `NewAnnouncementStatus(value string) (AnnouncementStatus, error)` で `draft` / `scheduled` / `published` のいずれか以外はDomain Errorとする（②「7. Value Object設計」AnnouncementStatus）
- 公開method一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|String|-|string|文字列表現を返す|
|CanTransitionTo|target AnnouncementStatus|bool|`draft→scheduled`、`draft→published`、`scheduled→published`のみ許可し、それ以外はfalseを返す（②「6. Entity設計」の状態変化定義）|

### ScheduledAt

- struct名: `ScheduledAt`（内部は`time.Time`をラップ）
- 生成時に検証するルール: `NewScheduledAt(t time.Time) (ScheduledAt, error)` で、現在時刻より未来でない場合はDomain Errorとする（②「7. Value Object設計」ScheduledAt / PublishedAt）
- 公開method一覧: `Time() time.Time`

### PublishedAt

- struct名: `PublishedAt`（内部は`time.Time`をラップ）
- 生成時に検証するルール: `NewPublishedAt(t time.Time) (PublishedAt, error)` はゼロ値でないことを検証する。`Announcement.Publish()`から呼ばれる用途として、未設定時に現在時刻を設定するための `NewPublishedAtNow() PublishedAt` を併せて用意する（②「7. Value Object設計」: `published`への遷移時に未設定であれば現在時刻を設定する）
- 公開method一覧: `Time() time.Time`

### TargetCriteria

- struct名: `TargetCriteria`
- 保持するフィールドと型:

|フィールド|型|意味|
|-|-|-|
|targetType|string|対象種別（all_users/by_role/by_grade/by_school/by_user）|
|roleID|*uint|`by_role`指定時の権限ID|
|gradeID|*uint|`by_grade`指定時の学年ID|
|userID|*uint|`by_user`指定時のユーザーID|

- 生成時に検証するルール: `NewTargetCriteria(targetType string, roleID, gradeID, userID *uint) (TargetCriteria, error)` で、②「7. Value Object設計」TargetCriteriaに記載の組み合わせルールをそのまま適用する。
  - `by_role` / `by_grade` は権限（`user_role_id`）の指定を要する
  - `by_grade` は学年（`grade_id`）の指定を要する
  - `by_user` はユーザー（`user_id`）の指定を要する
  - `by_school` は発信者の所属校を参照する（参照先IDの明示指定は不要）
  - `all_users` は参照先の指定を要しない

  **②からの補足**: 「`by_role` / `by_grade` は権限（`user_role_id`）の指定を要する」という記述と「`by_grade` は学年（`grade_id`）の指定を要する」という記述は②本文の原文表現をそのまま引用したものであり、`by_grade`が`user_role_id`と`grade_id`の両方を要求するのか、それとも記述上の重複かは②の文面のみでは判別できない。実装時は②本文（第7章 TargetCriteria）に立ち返って確認すること。

- 公開method一覧: `TargetType() string` / `RoleID() *uint` / `GradeID() *uint` / `UserID() *uint`

### ViewerContext（②からの補足）

- struct名: `ViewerContext`
- 保持するフィールドと型: `UserID uint` / `RoleID uint` / `SchoolID uint` / `GradeID *uint`
- 用途: 閲覧者（現在の教師ユーザー）の属性を表し、`AnnouncementVisibilityPolicy`および`AnnouncementRepository`の検索条件組み立てに用いる
- **②からの補足**: ②には閲覧者属性を表す型の定義がないため、②「8. Domain Service」AnnouncementVisibilityPolicyが「閲覧者側の属性」と表現している内容を型として補った（推測）。取得元（JWTクレーム等）は①未提供のため参照不可であり、既存認証基盤からcurrent userとして渡される前提とする。

## Value Objectを採用しないもの

- タイトル・本文: ②「7. Value Object設計」のとおり、文字列としての意味が強く独自の業務ルールを持たないためValue Object化しない

## Repository Interface

### AnnouncementRepository

- interface名: `AnnouncementRepository`
- メソッドシグネチャ一覧:

```go
type AnnouncementRepository interface {
    FindVisibleByViewer(ctx context.Context, viewer valueobject.ViewerContext, page dto.PageRequest) ([]*entity.Announcement, dto.PageInfo, error)
    FindByAuthor(ctx context.Context, publisherID uint, page dto.PageRequest) ([]*entity.Announcement, dto.PageInfo, error)
    FindVisibleByID(ctx context.Context, id uint, viewer valueobject.ViewerContext) (*entity.Announcement, error)
    FindByIDAndOwner(ctx context.Context, id uint, ownerID uint) (*entity.Announcement, error)
    Create(ctx context.Context, announcement *entity.Announcement) error
    Update(ctx context.Context, announcement *entity.Announcement) error
    FindRecentPublished(ctx context.Context, limit int) ([]*entity.Announcement, error)
}
```

- 各メソッドの責務:

|メソッド|責務|
|-|-|
|FindVisibleByViewer|閲覧者が閲覧可能な公開中お知らせを、公開日時降順・ページネーションで取得する（一覧・非authoredタブ）|
|FindByAuthor|発信者本人が作成したお知らせを取得する（一覧・authoredタブ）|
|FindVisibleByID|閲覧者が閲覧可能な公開中お知らせを単一取得する（詳細表示）|
|FindByIDAndOwner|発信者本人が作成したお知らせを所有権チェック込みで単一取得する（状態更新用）|
|Create|Announcement本体とAnnouncementTarget群を1つの整合性単位として永続化する|
|Update|Announcementの状態・公開予定日時・公開確定日時を永続化する|
|FindRecentPublished|公開中お知らせを公開日時降順で上位N件取得する（Teacher Dashboard Contextからの参照用、②「3. Bounded Context」他Contextとの依存関係）|

### GradeReferenceRepository / UserReferenceRepository / TeacherPermissionReferenceRepository

- interface名: `GradeReferenceRepository` / `UserReferenceRepository` / `TeacherPermissionReferenceRepository`
- メソッドシグネチャ一覧:

```go
type GradeReferenceRepository interface {
    ExistsInSchool(ctx context.Context, gradeID uint, schoolID uint) (bool, error)
}

type UserReferenceRepository interface {
    ExistsInSchool(ctx context.Context, userID uint, schoolID uint) (bool, error)
}

type TeacherPermissionReferenceRepository interface {
    IsOwnGradeScope(ctx context.Context, teacherID uint) (bool, error)
    OwnGradeID(ctx context.Context, teacherID uint) (uint, error)
}
```

- 各メソッドの責務: 対象指定の妥当性判定（`AnnouncementTargetingPolicy`）に必要な、学年・ユーザーの同校確認と、発信者の`own_grade`権限スコープ確認を提供する。書き込みは行わない（②「9. Repository設計」外部参照Repositoryの「保持しない責務」）

**②からの補足**: 上記3インターフェースのメソッド名・シグネチャは②に明記がなく、②「9. Repository設計」に記載された責務（存在確認・同校確認・権限確認）から実装のために補った（推測）。

## Domain Service

### AnnouncementTargetingPolicy

- struct名: `AnnouncementTargetingPolicy`
- コンストラクタ: `NewAnnouncementTargetingPolicy(gradeRef repository.GradeReferenceRepository, userRef repository.UserReferenceRepository, permissionRef repository.TeacherPermissionReferenceRepository) *AnnouncementTargetingPolicy`
- メソッドシグネチャ:

```go
func (p *AnnouncementTargetingPolicy) Validate(
    ctx context.Context,
    publisherID uint,
    publisherSchoolID uint,
    criteriaList []valueobject.TargetCriteria,
) error
```

- 責務: ②「8. Domain Service」AnnouncementTargetingPolicyのとおり、以下を判定する
  - `own_grade`権限の教師が自身の学年以外を`by_grade`で指定していないか
  - `by_grade`で指定する学年、`by_user`で指定するユーザーが発信者と同校か
  - 違反時は`domain/errors`のDomain Errorを返す

### AnnouncementVisibilityPolicy

- struct名: `AnnouncementVisibilityPolicy`
- メソッドシグネチャ:

```go
func (p *AnnouncementVisibilityPolicy) BuildVisibilityCondition(
    viewer valueobject.ViewerContext,
) valueobject.VisibilityCondition
```

- 責務: ②「8. Domain Service」AnnouncementVisibilityPolicyのとおり、教師が閲覧可能な条件（対象条件との突き合わせ条件）を表現する。判定結果の実行（DB問い合わせ）はRepositoryに委ねる。
- **②からの補足**: 戻り値型`VisibilityCondition`（`valueobject`配下に置く条件記述用の型）の具体的なフィールド構成は②に記載がなく、「閲覧者側の属性」と「お知らせ側の対象条件」を突き合わせるための条件値として実装上補った（推測）。フィールドは`ViewerContext`と同じ属性（UserID/RoleID/SchoolID/GradeID）をそのまま保持する構造を想定する。

## Domain Event

②「15. Domain Event」のとおり、本機能ではDomain Eventを採用しない。`domain/event/`ディレクトリは作成しない（対象外）。

将来的に公開時の通知連携が必要になった場合の`AnnouncementPublished`イベント導入は②に「推測」と明記された将来検討事項であり、本書では実装しない。

## Domain Error

- `domain/errors/errors.go` にセンチネルエラーとして定義する（コーディング規約「6. エラーハンドリング」「21. 変更しない方針」の`errors.New`利用方針に従う）

|エラー変数|発生条件|
|-|-|
|ErrInvalidStatusTransition|`AnnouncementStatus.CanTransitionTo`が許可しない状態遷移が`Schedule`/`Publish`で試みられた場合|
|ErrScheduledAtNotFuture|`NewScheduledAt`で指定日時が現在時刻以前の場合|
|ErrEmptyTargets|`NewAnnouncement`で対象指定が0件の場合|
|ErrInvalidTargetCriteria|`NewTargetCriteria`で対象種別に対する参照先の組み合わせが不正な場合|
|ErrTargetPermissionViolation|`AnnouncementTargetingPolicy.Validate`で`own_grade`権限の教師が自学年以外を指定した場合|
|ErrTargetSchoolMismatch|`AnnouncementTargetingPolicy.Validate`で指定学年・ユーザーが発信者と同校でない場合|

---

# 4. Application層設計

## DTO（Command / Query）

|struct名|フィールドと型|Command/Query区分|
|-|-|-|
|ListAnnouncementsQuery|CurrentUserID uint / Tab string / Page dto.PageRequest|Query|
|ShowAnnouncementQuery|CurrentUserID uint / AnnouncementID uint|Query|
|CreateAnnouncementCommand|CurrentUserID uint / Title string / Content string / Targets []TargetCriteriaInput|Command|
|TargetCriteriaInput|TargetType string / RoleID *uint / GradeID *uint / UserID *uint|Command（CreateAnnouncementCommandの内包型）|
|UpdateAnnouncementStatusCommand|CurrentUserID uint / AnnouncementID uint / Status string / ScheduledAt *time.Time|Command|
|PageRequest|Page int / PerPage int|Query（一覧系Queryの内包型）|
|PageInfo|Page int / PerPage int / TotalCount int / TotalPages int|Query（一覧系Resultの内包型）|
|AnnouncementListItem|ID uint / Title string / Status string / PublishedAt *time.Time / ScheduledAt *time.Time|Query（Result内包型）|
|ListAnnouncementsResult|Items []AnnouncementListItem / PageInfo PageInfo|Query|
|AnnouncementDetailResult|ID uint / Title string / Content string / PublisherID uint / Status string / PublishedAt *time.Time / ScheduledAt *time.Time / Targets []TargetCriteriaInput|Query|
|CreateAnnouncementResult|ID uint|Command|
|UpdateAnnouncementStatusResult|ID uint / Status string|Command|

**②からの補足**: `PageRequest`/`PageInfo`のフィールド名・初期値（1ページあたり件数等）は②に明記がなく、①も未提供のため参照不可。実装のため一般的なページネーション項目（Page/PerPage/TotalCount/TotalPages）で補った（推測）。実装時は既存フロントエンド互換のため実際のRailsレスポンス仕様を別途確認する必要がある。

## UseCase

### ListAnnouncementsUseCase

- struct名: `ListAnnouncementsUseCase`
- コンストラクタが受け取る依存: `repo repository.AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ListAnnouncementsQuery) (dto.ListAnnouncementsResult, error)`
- 処理ステップ:
  1. `query.Tab`が`authored`かどうかで分岐する
  2. `authored`の場合: `repo.FindByAuthor(ctx, query.CurrentUserID, query.Page)`を呼び出す
  3. それ以外の場合: `ViewerContext`を組み立て、`repo.FindVisibleByViewer(ctx, viewer, query.Page)`を呼び出す
  4. 取得したEntity群を`dto.ListAnnouncementsResult`へ変換して返す
- トランザクション境界: 読み取りのみのためトランザクションは使用しない（②「11. Transaction設計」）
- 発生しうるApplication Error: なし（一覧取得自体は失敗しない前提。Infrastructure Errorはそのまま上位へ伝播する）

### ShowAnnouncementUseCase

- struct名: `ShowAnnouncementUseCase`
- コンストラクタが受け取る依存: `repo repository.AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ShowAnnouncementQuery) (dto.AnnouncementDetailResult, error)`
- 処理ステップ:
  1. `ViewerContext`を組み立てる
  2. `repo.FindVisibleByID(ctx, query.AnnouncementID, viewer)`を呼び出す
  3. 取得できなければ`ErrAnnouncementNotFound`（Application Error）を返す
  4. 取得したEntityを`dto.AnnouncementDetailResult`へ変換して返す
- トランザクション境界: 読み取りのみのためトランザクションは使用しない
- 発生しうるApplication Error: `ErrAnnouncementNotFound`（対象お知らせが存在しない、または閲覧不可）

### CreateAnnouncementUseCase

- struct名: `CreateAnnouncementUseCase`
- コンストラクタが受け取る依存: `repo repository.AnnouncementRepository` / `targetingPolicy *service.AnnouncementTargetingPolicy`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.CreateAnnouncementCommand) (dto.CreateAnnouncementResult, error)`
- 処理ステップ:
  1. `cmd.Targets`各要素から`valueobject.NewTargetCriteria`でVOを組み立てる（不正な組み合わせはDomain Errorとして返る）
  2. 発信者の所属校IDを取得する（②「3. Bounded Context」User Context参照。取得元インターフェースは`domain/repository`の外部参照Repositoryを想定）
  3. `targetingPolicy.Validate(ctx, cmd.CurrentUserID, publisherSchoolID, criteriaList)`を呼び出す
  4. `[]*entity.AnnouncementTarget`を`valueobject.NewTargetCriteria`の結果から生成する
  5. `entity.NewAnnouncement(cmd.CurrentUserID, cmd.Title, cmd.Content, targets)`でAggregateを生成する
  6. `repo.Create(ctx, announcement)`で永続化する（本メソッド内でAnnouncement本体とAnnouncementTarget群を1トランザクションとして扱う。詳細は「8. Transaction実装方針」）
  7. `dto.CreateAnnouncementResult`を返す
- トランザクション境界: Announcement本体と全対象指定の作成を1トランザクションで扱う（②「11. Transaction設計」）。トランザクション自体は`repo.Create`実装内で完結させる（「8. Transaction実装方針」参照）
- 発生しうるApplication Error: `ErrReferenceNotFound`（参照先の学年・ユーザー等が存在しない場合）
- 発生しうるDomain Error: `ErrEmptyTargets` / `ErrInvalidTargetCriteria` / `ErrTargetPermissionViolation` / `ErrTargetSchoolMismatch`

### UpdateAnnouncementStatusUseCase

- struct名: `UpdateAnnouncementStatusUseCase`
- コンストラクタが受け取る依存: `repo repository.AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.UpdateAnnouncementStatusCommand) (dto.UpdateAnnouncementStatusResult, error)`
- 処理ステップ:
  1. `repo.FindByIDAndOwner(ctx, cmd.AnnouncementID, cmd.CurrentUserID)`で対象を取得する
  2. 取得できなければ`ErrNotOwner`（Application Error。存在しない場合も含め自身のお知らせでないものとして扱う）を返す
  3. `cmd.Status`が`scheduled`の場合: `announcement.Schedule(*cmd.ScheduledAt)`を呼び出す
  4. `cmd.Status`が`published`の場合: `announcement.Publish()`を呼び出す
  5. Entity側で返るDomain Error（`ErrInvalidStatusTransition` / `ErrScheduledAtNotFuture`）はそのまま上位へ伝播する
  6. `repo.Update(ctx, announcement)`で永続化する
  7. `dto.UpdateAnnouncementStatusResult`を返す
- トランザクション境界: 状態更新の完了時点でコミットする（②「11. Transaction設計」）
- 発生しうるApplication Error: `ErrNotOwner`
- 発生しうるDomain Error: `ErrInvalidStatusTransition` / `ErrScheduledAtNotFuture`

---

# 5. Infrastructure層設計

## Repository実装（Domain Model / Event Sourcing採用時）

### infrastructure/repository.AnnouncementRepository（package名: `gormrepo`）

- 実装struct名: `AnnouncementRepositoryImpl`とせず、package名`gormrepo`内の`AnnouncementRepository`とする（アーキテクチャ規約「9. 命名規約」: 実装側は接尾辞を付けずpackage名で区別する）
- 対応するGORMモデル: `persistence/gorm.AnnouncementModel`（テーブル`announcements`） / `persistence/gorm.AnnouncementTargetModel`（テーブル`announcement_targets`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件・ソート・ページネーション|
|-|-|
|FindVisibleByViewer|`status = published`かつ`published_at <= now`、かつ対象条件（`target_type = all_users`、または`target_type = by_role AND role_id = viewer.RoleID`、または`target_type = by_grade AND grade_id = viewer.GradeID`、または`target_type = by_school AND (発信者のschool_idがviewer.SchoolIDと一致)`、または`target_type = by_user AND user_id = viewer.UserID`）のOR条件。`published_at`降順、`page.Page`/`page.PerPage`によるOFFSET/LIMIT|
|FindByAuthor|`publisher_id = ?`、`created_at`降順、OFFSET/LIMIT|
|FindVisibleByID|`id = ?`かつFindVisibleByViewerと同じ可視条件|
|FindByIDAndOwner|`id = ?`かつ`publisher_id = ?`|
|Create|`announcements`へ1件INSERT後、`announcement_targets`へ対象件数分INSERT（詳細は「8. Transaction実装方針」）|
|Update|`announcements`の`status` / `scheduled_at` / `published_at`をUPDATE|
|FindRecentPublished|`status = published`、`published_at`降順、`LIMIT limit`|

- Entity ⇔ GORMモデルの変換方針: `infrastructure/repository`内に非公開の変換関数（`toEntity(m gorm.AnnouncementModel, targets []gorm.AnnouncementTargetModel) (*entity.Announcement, error)` / `fromEntity(a *entity.Announcement) (gorm.AnnouncementModel, []gorm.AnnouncementTargetModel)`）を置き、Repository実装メソッド内から呼び出す。EntityのValue Object（`AnnouncementStatus`等）とGORMモデルのプリミティブ型（`string`等）の相互変換もこの関数内で行う。

### infrastructure/repository.GradeReferenceRepository / UserReferenceRepository / TeacherPermissionReferenceRepository

- 実装方針: 各Contextが公開するRepository（例: School Contextの学年Repository、User Contextのユーザーaccount Repository、Teacher Permission Contextの権限Repository）を呼び出すアダプタとして実装する。アーキテクチャ規約「6. Context間連携ルール」に従い、他Contextの内部Entity・Infrastructure実装には直接依存しない。
- **②からの補足**: 呼び出し先となる各Contextの具体的なRepository・メソッド名は、各Context自身の②Go移行・設計仕様書（announcement以外のContext）に依存するため、本書では確定できない。①も未提供のため参照不可。実装時に該当Contextの②/③文書を参照して確定する必要がある（推測を含む）。

## 外部連携実装

Mail・Cache・Queueは②の記載範囲に存在しないため対象外とする。

---

# 6. Presentation層設計

## Handler

### AnnouncementHandler

- struct名: `AnnouncementHandler`
- 対応する呼び出し先: `application/usecase`配下の4 UseCase（`ListAnnouncementsUseCase` / `ShowAnnouncementUseCase` / `CreateAnnouncementUseCase` / `UpdateAnnouncementStatusUseCase`）
- メソッド一覧:

|メソッド|HTTPメソッド・パス|
|-|-|
|List|GET /api/v1/teacher/announcements|
|Show|GET /api/v1/teacher/announcements/:id|
|Create|POST /api/v1/teacher/announcements|
|UpdateStatus|PATCH /api/v1/teacher/announcements/:id|

- 処理順序（共通パターン）:
  1. 入力バインド（Gin `ShouldBindQuery` / `ShouldBindJSON` / `ShouldBindUri`）
  2. Request DTOのバリデーションタグによる検証（失敗時は422を返す）
  3. Middlewareで確定済みのcurrent user情報をcontextから取得する
  4. 対応するUseCaseのCommand/Queryへ変換して`Execute`を呼び出す
  5. 戻り値のDomain Error/Application Errorを「11. Error実装方針」の変換表に従いHTTPステータスへ変換する
  6. 成功時はResult DTOをResponse DTOへ変換してJSONで返す

## Request / Response DTO

### Request

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|ListAnnouncementsRequest|Tab string `form:"tab"` / Page int `form:"page"`|Pageは`min=1`（未指定時デフォルト1）。Tabは`oneof=authored published`程度の形式チェック（②「16. API互換方針」`tab`パラメータ準拠）|
|CreateAnnouncementRequest|Announcement CreateAnnouncementBody `json:"announcement" binding:"required"`|ネストされたJSON構造（②「16. API互換方針」`announcement[title]`等の意味を維持）|
|CreateAnnouncementBody|Title string `json:"title" binding:"required"` / Content string `json:"content" binding:"required"` / AnnouncementTargets []AnnouncementTargetRequest `json:"announcement_targets" binding:"required,min=1,dive"`|title/content必須、announcement_targetsは配列かつ非空（②「12. Validation設計」Presentation）|
|AnnouncementTargetRequest|TargetType string `json:"target_type" binding:"required,oneof=all_users by_role by_grade by_school by_user"` / RoleID *uint `json:"role_id"` / GradeID *uint `json:"grade_id"` / UserID *uint `json:"user_id"`|target_typeは定義済み値のみ許容する形式チェック（②「12. Validation設計」Presentation フォーマットチェック）|
|UpdateAnnouncementStatusRequest|Status string `json:"status" binding:"required,oneof=scheduled published"` / ScheduledAt *time.Time `json:"scheduled_at" binding:"required_if=Status scheduled"`|statusは`scheduled`/`published`のみ許容し、`scheduled`指定時は`scheduled_at`必須（②「12. Validation設計」フォーマットチェック: scheduled_atの日付形式検証）|

### Response

|struct名|フィールドと型|
|-|-|
|AnnouncementListItemResponse|ID uint / Title string / Status string / PublishedAt *time.Time / ScheduledAt *time.Time|
|AnnouncementListResponse|Items []AnnouncementListItemResponse / Page int / PerPage int / TotalCount int / TotalPages int|
|AnnouncementDetailResponse|ID uint / Title string / Content string / PublisherID uint / Status string / PublishedAt *time.Time / ScheduledAt *time.Time / Targets []AnnouncementTargetResponse|
|AnnouncementTargetResponse|TargetType string / RoleID *uint / GradeID *uint / UserID *uint|
|AnnouncementCreateResponse|ID uint|
|AnnouncementUpdateResponse|ID uint / Status string|
|ErrorResponse|Errors []string（既存のerrors形式を踏襲。②「16. API互換方針」Error Response）|

**②からの補足**: `AnnouncementSerializer` / `AuthoredAnnouncementSerializer`（②「16. API互換方針」）が実際にどのフィールドを出力していたかは①未提供のため参照不可。上記フィールド構成は②本文の記載（お知らせ本体・対象条件・状態・日時）から推測して補ったものであり、実装時にフロントエンド互換性の観点で最終確認が必要である。

## Routing

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/teacher/announcements|AnnouncementHandler.List|
|GET|/api/v1/teacher/announcements/:id|AnnouncementHandler.Show|
|POST|/api/v1/teacher/announcements|AnnouncementHandler.Create|
|PATCH|/api/v1/teacher/announcements/:id|AnnouncementHandler.UpdateStatus|

`presentation/routes.go`に`RegisterRoutes(r *gin.RouterGroup, h *handler.AnnouncementHandler)`として登録する。`teacher`ロール確認Middlewareは、本Contextのルーティング登録時にルーターグループへ適用する前提とする（②「13. Authorization設計」Middleware）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/teacher/announcements|AnnouncementHandler.List|ListAnnouncementsRequest|AnnouncementListResponse|200|
|GET|/api/v1/teacher/announcements/:id|AnnouncementHandler.Show|-（URIパラメータのみ）|AnnouncementDetailResponse|200|
|POST|/api/v1/teacher/announcements|AnnouncementHandler.Create|CreateAnnouncementRequest|AnnouncementCreateResponse|200|
|PATCH|/api/v1/teacher/announcements/:id|AnnouncementHandler.UpdateStatus|UpdateAnnouncementStatusRequest|AnnouncementUpdateResponse|200|

各Endpointについて、Errorケースを整理する。

|条件|Status Code|Error内容|
|-|-|-|
|GET一覧: 認証なし/teacherロールでない|401/403|Middlewareで拒否（②「13. Authorization設計」Middleware）|
|GET詳細: 対象お知らせが存在しない、または閲覧不可|404|ErrAnnouncementNotFound|
|POST作成: title/content/announcement_targets不足・不正|422|Request DTOバリデーション失敗|
|POST作成: own_grade制約違反|422|ErrTargetPermissionViolation|
|POST作成: 同校制約違反|422|ErrTargetSchoolMismatch|
|POST作成: 参照先（学年・ユーザー等）が存在しない|422|ErrReferenceNotFound|
|PATCH更新: 自身が作成したお知らせでない|404|ErrNotOwner|
|PATCH更新: 不正な状態遷移|422|ErrInvalidStatusTransition|
|PATCH更新: 予約日時が未来日時でない|422|ErrScheduledAtNotFound|

上記Status Codeは②「16. API互換方針」の「200: 取得・更新成功 / 422: 入力・業務ルール違反 / 404: 対象お知らせ不存在または閲覧不可」に基づく。`ErrNotOwner`を404に割り当てる判断は「14. ②からの補足事項」に記載する。

---

# 8. Transaction実装方針

## Transaction開始箇所

- 書き込みを伴うUseCase（`CreateAnnouncementUseCase` / `UpdateAnnouncementStatusUseCase`）の実行中、Repository実装（`infrastructure/repository`の`AnnouncementRepository`）の`Create`/`Update`メソッド内でGORMのトランザクション（`db.Transaction(func(tx *gorm.DB) error { ... })`）を開始する
- UseCase自体はトランザクション管理の詳細（GORMの`*gorm.DB`等）を持たない。トランザクション境界の意思決定（②「11. Transaction設計」でUseCase単位と定義）は、UseCaseが1回のRepository呼び出しでAnnouncement本体と対象指定群の作成・更新を完結させることで表現する

## Transaction終了箇所

- `Create`: `announcements`への1件INSERTと、`announcement_targets`への全件INSERTが成功した時点でコミットする。いずれかが失敗した場合はロールバックする
- `Update`: `announcements`の状態・日時UPDATEが成功した時点でコミットする

## 複数Repository（またはStore・関数）にまたがる場合の扱い

- `CreateAnnouncementUseCase`は`AnnouncementTargetingPolicy`の判定で外部参照Repository（Grade/User/TeacherPermission）を呼び出すが、これらは読み取り専用でありトランザクションの対象外とする（書き込みは`AnnouncementRepository`のみ）
- 本機能内でAnnouncementRepository以外に書き込みを行うRepositoryは存在しない

**②からの補足**: ②「11. Transaction設計」は「UseCaseの開始時にトランザクションを開始する」という記載にとどまり、実装上どのレイヤー（UseCase／Repository実装）でGORMトランザクションのコードを書くかまでは指定していない。本書ではアーキテクチャ規約「8. 横断的関心事の置き場所」の「Transaction管理は業務処理の単位（UseCase）で開始・終了する」という原則を保ちつつ、UseCaseからGORMへの直接依存（依存方向違反）を避けるため、実装上はRepository実装内でトランザクションを完結させる方式とした（判断・②に根拠はあるが実装位置は補足）。

---

# 9. Validation実装方針

## Presentation

- 型チェック: Request DTOの型（`string`/`[]AnnouncementTargetRequest`/`*time.Time`等）で担保する
- 必須チェック: `binding:"required"`により、title/content/announcement_targets（配列かつ非空）を検証する
- フォーマットチェック: `binding:"oneof=..."`によりtarget_type・statusが定義済みの値であることを検証する。scheduled_atはGinの`time.Time`バインドによる日付形式検証を行う

## 業務ルール検証

Domain Model採用のため、以下はDomain（Entity／Value Object／Domain Service）で検証する。

- `valueobject.NewTargetCriteria`: 対象種別ごとの参照先の組み合わせ（②「7. Value Object設計」TargetCriteria独自ルール）
- `valueobject.NewScheduledAt`: `scheduled`状態での公開予定日時の未来日時チェック
- `entity.Announcement.Schedule` / `Publish`: 状態遷移が許可された組み合わせかどうか
- `service.AnnouncementTargetingPolicy.Validate`: `own_grade`権限の教師が自身の学年以外を指定していないか、対象の学年・ユーザーが同校かどうか

②「12. Validation設計」の責務分離方針（Presentationは形式、Domainは業務的妥当性）をそのまま踏襲する。

---

# 10. Authorization実装方針

## Middleware

- 認証済みユーザーを特定し、`teacher`ロールであることを確認する（②「13. Authorization設計」Middleware）。実装は既存の認証Middleware（`authentication`Context側で提供）を`announcement`のルーティングに適用する形とし、本書では新規実装しない

## Handler

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定は持たせない（②「13. Authorization設計」Handler）

## UseCase

- `ListAnnouncementsUseCase`: `tab=authored`の場合、`repo.FindByAuthor`により`CurrentUserID`一致のもののみを対象とする
- `UpdateAnnouncementStatusUseCase`: `repo.FindByIDAndOwner`により、発信者本人のお知らせのみを更新対象とする（所有者確認）

## Domain

- `AnnouncementTargetingPolicy`: `own_grade`権限の教師が指定できる対象範囲を制限する
- `Announcement.Schedule` / `Publish`: 許可されない状態遷移を拒否する

②「13. Authorization設計」の判断理由（認証・ロール確認はMiddleware、所有権確認はUseCase、業務上許される操作の判定はDomain）をそのまま踏襲する。

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

- UseCaseは、Repository/Domain Serviceが返すDomain Errorを`errors.Is`で判定し、そのまま呼び出し元（Handler）へ伝播させる（Domain ErrorをApplication Errorへラップし直す変換は行わない。Domain ErrorとApplication Errorは別種のセンチネルエラーとして併存し、HTTP変換時にHandlerが両方を判定する）
- Application Error（`ErrAnnouncementNotFound` / `ErrNotOwner` / `ErrReferenceNotFound`）はUseCase内で独自に発生させ、`application/usecase/errors.go`に定義する

## Application Error → HTTPレスポンスへの変換方針（Status Code対応表）

|Error種別|発生層|HTTP Status|
|-|-|-|
|ErrInvalidStatusTransition|Domain|422|
|ErrScheduledAtNotFuture|Domain|422|
|ErrEmptyTargets|Domain|422|
|ErrInvalidTargetCriteria|Domain|422|
|ErrTargetPermissionViolation|Domain|422|
|ErrTargetSchoolMismatch|Domain|422|
|ErrAnnouncementNotFound|Application|404|
|ErrNotOwner|Application|404|
|ErrReferenceNotFound|Application|422|
|その他（DB接続エラー等）|Infrastructure|500|

Handler側で`errors.Is`によるswitch的な判定を行い、上記対応表に従いHTTPステータスとErrorResponseへ変換する。

## Infrastructure Errorのハンドリング方針

- DB接続・クエリ実行時のエラーはRepository実装内で`fmt.Errorf("...: %w", err)`によりラップしてUseCaseへ返す（コーディング規約「6. エラーハンドリング」エラーのラップ）
- UseCase・Handlerでは、Domain Error/Application Errorのいずれにも該当しないエラーは全て500として扱う

**②からの補足**: `ErrNotOwner`のHTTP Statusを404とする判断、および`ErrReferenceNotFound`を422とする判断は、②「16. API互換方針」のStatus Code定義（200/422/404）と②「14. Error設計」のApplication Error分類から実装のために補ったものである（推測）。②本文には各Application Errorの個別Statusコード対応までは明記されていない。

---

# 12. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

②「17. DB設計方針」のとおり既存Rails DBを継続利用し、Schema変更は行わない。

|GORMモデル|対応テーブル|
|-|-|
|persistence/gorm.AnnouncementModel|announcements|
|persistence/gorm.AnnouncementTargetModel|announcement_targets|

Gorm規約に従い、構造体名の複数形がそのままテーブル名規則に合致するため、`TableName()`のオーバーライドは不要とする（`Announcement`→`announcements`、`AnnouncementTarget`→`announcement_targets`）。`CreatedAt`/`UpdatedAt`フィールドを両モデルに保持し、GORMの自動タイムスタンプ機能（Gorm規約「タイムスタンプのトラッキング」）に委ねる。

## 主要クエリの条件・ソート・ページネーション方針

「5. Infrastructure層設計」記載のクエリ内容表のとおり。SQL文そのものはここに記載しない。要点のみ再掲する。

- 閲覧可能条件: `status = published AND published_at <= now()` に加え、対象条件（all_users / by_role / by_grade / by_school / by_user）のOR条件を組み合わせる
- 作成者条件: `publisher_id = ?`
- ソート: 一覧・詳細とも`published_at`降順（`FindByAuthor`のみ`created_at`降順とし、下書き・予約中も含めて新しい順に表示する）
- ページネーション: `Page`/`PerPage`からOFFSET/LIMITを算出する
- ダッシュボード向け: `status = published`、`published_at`降順、`LIMIT limit`

## 既存Schemaに対する変更が②で提案されている場合の反映方針

②「17. DB設計方針」に変更提案はなく、本書でも既存Schemaに対する変更は行わない。

---

# 13. テストケース設計

②「18. テスト戦略」で定義された5区分（Domain Test / UseCase Test / Repository Test / Handler Test / Integration Test）をそのまま使用する（Domain Model採用のため読み替えは不要）。

## Domain Test

|対象|テストケース|
|-|-|
|AnnouncementStatus.CanTransitionTo|draft→scheduled/published、scheduled→publishedが許可されること／published→他の状態が拒否されること|
|Announcement.Schedule|draft状態から未来日時でscheduledへ遷移できること／過去日時ではErrScheduledAtNotFutureとなること／published状態からはErrInvalidStatusTransitionとなること|
|Announcement.Publish|draft/scheduledからpublishedへ遷移し、PublishedAtが確定すること／published状態から再度呼ぶとErrInvalidStatusTransitionとなること|
|NewAnnouncement|対象指定が0件の場合ErrEmptyTargetsとなること／初期状態がdraftであること|
|NewTargetCriteria|target_typeごとに必要な参照先が不足している場合にErrInvalidTargetCriteriaとなること|
|AnnouncementTargetingPolicy.Validate|own_grade権限の教師が自学年以外を指定した場合にErrTargetPermissionViolationとなること／同校でない学年・ユーザーを指定した場合にErrTargetSchoolMismatchとなること|

## UseCase Test

|対象|テストケース|
|-|-|
|ListAnnouncementsUseCase|tab=authoredで発信者本人の一覧のみ取得すること／それ以外で閲覧可能な公開中お知らせのみ取得すること|
|ShowAnnouncementUseCase|閲覧可能なお知らせを取得できること／存在しない・閲覧不可の場合にErrAnnouncementNotFoundを返すこと|
|CreateAnnouncementUseCase|正しい入力で作成が成功すること／targetingPolicy違反時にDomain Errorがそのまま伝播すること／参照先不存在時にErrReferenceNotFoundを返すこと|
|UpdateAnnouncementStatusUseCase|所有者本人による正しい状態遷移が成功すること／所有者でない場合にErrNotOwnerを返すこと／不正な状態遷移がDomain Errorとして伝播すること|

## Repository Test

|対象|テストケース|
|-|-|
|FindVisibleByViewer|対象条件（all_users/by_role/by_grade/by_school/by_user）ごとに正しく絞り込まれること／未公開・未来日時のscheduledが含まれないこと／ページネーションが正しく機能すること|
|FindByAuthor|発信者本人のお知らせのみ取得され、状態を問わず含まれること|
|Create|Announcement本体とAnnouncementTarget群が一括で永続化されること／一部失敗時にロールバックされること|
|Update|状態・公開予定日時・公開確定日時の更新が反映されること|
|FindRecentPublished|公開中のお知らせが公開日時降順で上位N件取得されること|

## Handler Test

|対象|テストケース|
|-|-|
|List|クエリパラメータのバリデーション、200レスポンスの構造|
|Show|存在しないIDで404が返ること|
|Create|title/content/announcement_targets不足時に422が返ること|
|UpdateStatus|不正なstatus値でバインドエラー（422）になること|

## Integration Test

|対象|テストケース|
|-|-|
|作成→一覧（authored）取得|作成したお知らせがauthoredタブの一覧に反映されること|
|作成→状態更新（scheduled）→状態更新（published）|一連の状態遷移がAPI経由で正しく行われ、published後は一般の一覧・詳細から閲覧可能になること|
|他ユーザーによる更新試行|所有者でないユーザーからのPATCHが404で拒否されること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|外部参照Repository（GradeReferenceRepository / UserReferenceRepository / TeacherPermissionReferenceRepository）の具体的なメソッドシグネチャ|②「9. Repository設計」は責務のみを記載しており、メソッド名・引数構成までは記載がないため、コーディング規約「5. インターフェース」（利用側で定義する）に従い実装のために補った|推測|
|`ViewerContext`（閲覧者属性を表す型）の新設|②「8. Domain Service」AnnouncementVisibilityPolicyが言及する「閲覧者側の属性」を型として表現する必要があるが、②に型定義はないため補った|推測|
|`AnnouncementVisibilityPolicy.BuildVisibilityCondition`の戻り値型`VisibilityCondition`の具体的なフィールド構成|②はPolicyの役割（条件の「意味」を定義する）のみを記載し、具体的な型構造は記載がないため、`ViewerContext`と同構造として補った|推測|
|Transactionの実装位置をUseCaseではなくRepository実装（`AnnouncementRepository.Create`/`Update`）内に置く方針|②「11. Transaction設計」は「UseCaseの開始時にトランザクションを開始する」と記載するのみで実装レイヤーまで指定していない。UseCaseからGORMへの直接依存を避けるため（アーキテクチャ規約「3. レイヤー責務と依存方向」の依存性逆転を維持するため）、Repository実装内で完結させる方式とした|判断（②の方針とは矛盾しないが、実装位置は②に明記がないため推測を含む）|
|ページネーション項目（Page/PerPage/TotalCount/TotalPages）のフィールド名・デフォルト値|②・①ともに具体的なページネーション仕様の記載がないため、一般的な項目名で補った。①未提供のため参照不可|推測|
|Response DTOの具体的なフィールド構成（`AnnouncementListItemResponse`等）|②「16. API互換方針」は`AnnouncementSerializer`等への言及にとどまり、実際の出力フィールドはRails実装（①）に依存する。①が未提供のため、②本文（お知らせ本体・対象条件・状態・日時）から推測して補った|推測（①未提供のため参照不可）|
|`ErrNotOwner`のHTTP Statusを404とする対応|②「16. API互換方針」のStatus Code定義（200/422/404）には`ErrNotOwner`個別の割り当てが明記されていないため、「対象お知らせ不存在または閲覧不可」の404区分に含める形で判断した|推測|
|`ErrReferenceNotFound`のHTTP Statusを422とする対応|②「14. Error設計」でApplication Errorに分類されているが個別Statusは明記されていないため、「入力・業務ルール違反」の422区分に含める形で判断した|推測|
|`TargetCriteria`における`by_grade`の必須項目（`user_role_id`と`grade_id`の両方を要するかの解釈）|②本文の記述をそのまま転記するにとどめ、解釈の確定は行っていない。実装者は②「7. Value Object設計」TargetCriteriaの原文を確認すること|②原文をそのまま引用（解釈は保留）|
|各外部参照Contextの具体的なRepository・メソッド名（実装時の接続先）|①未提供のため参照不可。各Context自身の②/③文書に依存するため本書では確定できない|参照不可につき保留|

---
