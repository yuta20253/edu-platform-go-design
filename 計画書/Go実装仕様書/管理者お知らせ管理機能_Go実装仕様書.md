# 管理者お知らせ管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者が全ユーザー向けのお知らせを作成・編集・配信（公開）・削除できる機能である。一覧・詳細・作成・更新・削除・配信の6操作に加え、指定高校を対象に含むお知らせ（発信者のロールを問わない）を横断的に確認する高校別お知らせ参照を提供する。管理者が作成するお知らせの対象は常に「全ユーザー」に固定され、教師のように特定の高校・学年・ユーザーへ対象を絞り込むことはできない（②「1. 機能概要」）。一覧・詳細の参照対象は、自分自身が作成したお知らせに限らず、管理者ロールのユーザーが発信者であるお知らせ全体である。更新・削除・配信を実行できるのは、お知らせの発信者本人である管理者のみであり、発信者の管理者アカウントが無効化されている場合に限り他の管理者も実行できる。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により **Domain Model** を採用する。

- 本機能が作成・更新・削除・配信の対象とするAnnouncementは、教師お知らせ機能_Go移行・設計仕様書において既にDomain Modelとして設計され、状態遷移ルール（draft→scheduled/published、scheduled→published、published→遷移不可）を持つ。同一Aggregateに対して教師側・管理者側で異なる実装構造を採用すると、状態遷移ロジックの複製またはRepository Interfaceの位置づけの矛盾が生じる
- 状態遷移ルール自体（published状態からは一切の変更を受け付けない等）は管理者・教師で共通であり、既存のAnnouncement Entityに集約された判定ロジックをそのまま再利用する必要がある

Transaction Script・Active Record（同一AggregateをDomain Modelとして扱う教師お知らせ機能との構造的一貫性が崩れるため不採用）、Event Sourcing（状態遷移の全履歴を再構築する要件が現行仕様に存在せず過剰設計）はいずれも②で不採用と判断されている。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- 対象Bounded Context: `announcement`（お知らせ機能・教師お知らせ機能と同一Context。②「3. Bounded Context」の「対象データ・状態遷移ルールが完全に同一であり、参照・集約専用Contextへの分割基準に該当しない」という判断による）
- ②「12. UseCase設計」のListAnnouncementsUseCase・ShowAnnouncementUseCase・CreateAnnouncementUseCase・UpdateAnnouncementUseCase・DeleteAnnouncementUseCase・PublishAnnouncementUseCase・ListHighSchoolAnnouncementsUseCase（管理者向けの7UseCase。教師お知らせ機能_Go実装仕様書のUseCase群とは別のstructとして実装する。同名の業務操作（一覧・詳細）が存在するが、発信者スコープが異なるため別UseCaseとする）
- ②「19. API仕様」記載の7エンドポイント

**重要（Entity/Repository/Value Objectの再利用方針）**: 本機能が扱うAnnouncement・AnnouncementTarget Entity、AnnouncementStatus・ScheduledAt・PublishedAt・TargetCriteria Value Object、および`AnnouncementRepository` Interfaceの基本定義（struct名・フィールド・状態遷移メソッド・基本CRUD相当のメソッド）は、教師お知らせ機能_Go実装仕様書「3. Domain層設計」で既に定義済みである。本書ではこれらを**再定義せず、そのまま再利用する**。本書が追加するのは、(a) 既存の`AnnouncementRepository` interfaceへの管理者向け検索・削除・高校別検索・発信者無効化状態確認メソッドの追加、(b) 高校の存在確認のための外部参照Repository（`HighSchoolRepository`）、(c) 本機能固有のUseCase・Handler・Request/Response DTOのみである。教師お知らせ機能_Go実装仕様書「2. ディレクトリ構成」で作成済みの`internal/announcement/domain/entity/announcement.go`・`announcement_target.go`・`domain/valueobject/announcement_status.go`・`scheduled_at.go`・`published_at.go`・`target_criteria.go`・`domain/repository/announcement_repository.go`は本書でも同一ファイルとして扱い、重複するファイルを新規に作成しない。

なお、教師お知らせ機能_Go実装仕様書が定義した`AnnouncementTargetingPolicy`（own_grade制約・同校制約の判定）・`AnnouncementVisibilityPolicy`（閲覧者視点の可視性判定）は、②「8. Domain Service」の判断どおり本機能では利用しない。管理者向けの対象指定は常に`all_users`固定であり、対象妥当性判定・閲覧可否判定という可変ルールが存在しないためである。

- Railsの実装詳細（①）は本タスクで提供されておらず、参照が必要な箇所は「①未提供のため参照不可」として扱う

---

# 2. ディレクトリ構成

- 対象Bounded Context名: `announcement`（`internal/`配下のディレクトリ名は、教師お知らせ機能_Go実装仕様書と同一の`internal/announcement`を用いる）
- ②で採用した設計パターン: Domain Model
- アーキテクチャ規約「3. 設計パターンごとの構造適用方針」のDomain Model構造（標準フルレイヤー構成）を、既存の`internal/announcement/`配下に追加する形で適用する（新規Context・新規ディレクトリツリーを作らない）

## 既存ディレクトリ（教師お知らせ機能_Go実装仕様書で作成済み、本書では変更しない）

```
internal/announcement/
├── domain/
│   ├── entity/         # announcement.go, announcement_target.go は変更しない
│   ├── valueobject/     # announcement_status.go 等は変更しない
│   ├── repository/      # announcement_repository.go は本書でメソッド追加（後述）
│   ├── service/         # AnnouncementTargetingPolicy等は本書では利用しない
│   └── errors/
├── application/
│   ├── dto/
│   └── usecase/
├── infrastructure/
│   ├── persistence/gorm/
│   └── repository/       # announcement_repository.go は本書でメソッド実装追加
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

## 本書が追加するファイル一覧

```
internal/announcement/domain/repository/high_school_repository.go
internal/announcement/application/dto/list_admin_announcements.go
internal/announcement/application/dto/show_admin_announcement.go
internal/announcement/application/dto/create_admin_announcement.go
internal/announcement/application/dto/update_admin_announcement.go
internal/announcement/application/dto/list_high_school_announcements.go
internal/announcement/application/usecase/list_admin_announcements_usecase.go
internal/announcement/application/usecase/show_admin_announcement_usecase.go
internal/announcement/application/usecase/create_admin_announcement_usecase.go
internal/announcement/application/usecase/update_admin_announcement_usecase.go
internal/announcement/application/usecase/delete_admin_announcement_usecase.go
internal/announcement/application/usecase/publish_admin_announcement_usecase.go
internal/announcement/application/usecase/list_high_school_announcements_usecase.go
internal/announcement/infrastructure/repository/high_school_repository.go
internal/announcement/presentation/handler/admin_announcement_handler.go
internal/announcement/presentation/handler/admin_high_school_announcement_handler.go
internal/announcement/presentation/request/admin_announcement_request.go
internal/announcement/presentation/response/admin_announcement_response.go
```

`presentation/routes.go`は教師お知らせ機能_Go実装仕様書で作成済みのファイルであり、本書ではそこへ`AdminAnnouncementHandler`・`AdminHighSchoolAnnouncementHandler`のルート登録を追加する（新規ファイルとしては作成しない）。

## 本書が変更（メソッド追加）する既存ファイル

|ファイル|追加内容|
|-|-|
|`domain/repository/announcement_repository.go`|`AnnouncementRepository` interfaceへ`SearchByPublisherRole` / `FindByIDAndPublisherRole` / `Delete` / `SearchByHighSchoolTarget` / `IsPublisherDeactivated`メソッドを追加する（本書「3. Domain層設計」参照）|
|`infrastructure/repository/announcement_repository.go`|上記追加メソッドの実装を追加する（本書「5. Infrastructure層設計」参照）|

---

# 3. Domain層設計

## Entity（再利用、本書では再定義しない）

### Announcement / AnnouncementTarget

教師お知らせ機能_Go実装仕様書「3. Domain層設計」の`Announcement`（Aggregate Root、`NewAnnouncement` / `Schedule` / `Publish`等）・`AnnouncementTarget`（`NewAnnouncementTarget`）のstruct定義・メソッド・不変条件をそのまま再利用する。本書はこれらのEntityに新たなフィールド・メソッドを追加しない。

**②からの補足**: ②「6. Entity設計」は「発信者（publisher）が管理者ロールである場合、対象指定は常に1件・target_type: all_usersのみであるという制約を、AnnouncementTarget作成時のルールとして扱う」としている。この制約はEntity自体には実装せず（Entity側の不変条件は「対象指定が0件でないこと」のみで教師・管理者共通のまま）、後述する`CreateAnnouncementUseCase`（管理者向け）が`TargetCriteria`を`all_users`のみで生成することによって実現する（②「12. UseCase設計」の判断根拠に対応）。

## Value Object（再利用、本書では再定義しない）

### AnnouncementStatus / ScheduledAt / PublishedAt / TargetCriteria

教師お知らせ機能_Go実装仕様書「3. Domain層設計」の定義をそのまま再利用する。②「7. Value Object設計」の「既存Value Objectの再利用」に対応する。管理者作成時は`TargetCriteria`のうち`target_type: all_users`のみを生成し、他の種別（`by_role`/`by_grade`/`by_school`/`by_user`）は使用しない。

## 外部参照Entity（master-data Context提供、本書で新規追加）

### HighSchool

- struct名: `HighSchool`（`domain/repository/high_school_repository.go`に、`HighSchoolRepository` Interfaceの戻り値型として定義する。本Contextが所有するAggregateには含めない）
- フィールド: `ID uint`, `Name string`
- ライフサイクル・状態変化: 本Contextでは扱わない。`high_schools`テーブルの正規の所有者はmaster-data Context（共通マスタ参照機能。共通マスタ参照機能_Go実装仕様書「15. GORM / DBクエリ設計」参照）であり、作成・更新・削除の責務も同Contextが持つ

## Repository Interface

### AnnouncementRepository（既存interfaceへのメソッド追加）

教師お知らせ機能_Go実装仕様書が定義した`AnnouncementRepository` interface（`FindVisibleByViewer` / `FindByAuthor` / `FindVisibleByID` / `FindByIDAndOwner` / `Create` / `Update` / `FindRecentPublished`）に、以下のメソッドを追加する（同一ファイル`domain/repository/announcement_repository.go`内、同一interfaceへの追記）。

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`SearchByPublisherRole`|`(ctx context.Context, keyword string, status *valueobject.AnnouncementStatus, page dto.PageRequest)`|`([]*entity.Announcement, dto.PageInfo, error)`|発信者が管理者ロールであるお知らせ（自分自身に限らない）を、タイトル部分一致・状態で絞り込んで一覧取得する（②「11. Repository設計」の「拡張部分」）|
|`FindByIDAndPublisherRole`|`(ctx context.Context, id uint)`|`(*entity.Announcement, error)`|発信者が管理者ロールであるお知らせを単一取得する。所有者確認（`FindByIDAndOwner`が行う「本人が作成したものか」）ではなく、「管理者ロールの誰かが作成したものか」のみを確認する（発信者本人かどうかの操作権限確認はUseCaseが行う）点が教師向けメソッドとの違いである|
|`Delete`|`(ctx context.Context, id uint)`|`error`|Announcement本体とAnnouncementTargetを削除する（②「11. Repository設計」の「発信者が管理者ロールであるお知らせの削除」）|
|`SearchByHighSchoolTarget`|`(ctx context.Context, highSchoolID uint, page dto.PageRequest)`|`([]*entity.Announcement, dto.PageInfo, error)`|指定高校を対象に含むお知らせを、発信者のロールを問わず横断検索する（②「11. Repository設計」の「指定高校を対象に含むお知らせの横断検索」）|
|`IsPublisherDeactivated`|`(ctx context.Context, publisherID uint)`|`(bool, error)`|発信者アカウントが無効化（論理削除）されているかを返す。更新・削除・配信の操作権限確認に用いる（②「11. Repository設計」の「発信者アカウントが無効化されているかどうかの参照」）。無効化されたユーザーも取得対象に含める必要があるため、論理削除による自動除外を行わない（後述「5. Infrastructure層設計」参照）|

**②からの補足**: `SearchByPublisherRole`・`FindByIDAndPublisherRole`・`Delete`・`SearchByHighSchoolTarget`・`IsPublisherDeactivated`という具体的なメソッド名・シグネチャは②に明記がなく、②「11. Repository設計」に記載された責務（発信者ロールによる絞り込み検索、削除、高校ID絞り込み検索、発信者アカウントの無効化状態の参照）を実装のために具体化したものである（推測。教師お知らせ機能③文書が外部参照Repositoryのメソッドシグネチャを推測で補ったのと同様の対応）。

### HighSchoolRepository（`domain/repository/high_school_repository.go`、master-data Context（共通マスタ参照機能）提供・外部依存として利用）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByID`|`(ctx context.Context, id uint)`|`(*HighSchool, error)`|対象高校の存在確認（②「12. UseCase設計」ListHighSchoolAnnouncementsUseCaseの「対象高校の存在確認」に対応）|

- 保持しない責務: 高校データの作成・更新・削除
- 判断根拠: ②「3. Bounded Context」の「School Context（高校情報）: 高校別お知らせ参照における対象高校の存在確認に依存する」を、本Contextが定義する参照専用Interfaceとして具体化したものである。アーキテクチャ規約「6. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従う。

**②からの補足**: ②「3. Bounded Context」がいう「School Context（高校情報）」の実体は、本タスクで並行して確定した共通マスタ参照機能_Go実装仕様書が所有するmaster-data Context（`high_schools`テーブルの正規の所有者。共通マスタ参照機能_Go実装仕様書「15. GORM / DBクエリ設計」参照）である。管理者高校学年参照機能（school-directory Context）は`high_schools`テーブルを参照するのみで所有はしておらず、Transaction Script採用のためRepository Interfaceも公開していない（管理者高校学年参照機能_Go実装仕様書「3. Domain層設計」）ため、参照先としては不適切である。Domain Model採用の本Contextは、master-data Contextが公開するapplication層関数（`ExistsHighSchool` / `FindHighSchoolByID`）を呼び出す実装を、自ら定義する`HighSchoolRepository` Interfaceの背後に置く（依存性逆転。アーキテクチャ規約「2. レイヤー責務と依存方向」）。これにより`high_schools`テーブルへの直接クエリは行わない。

## Domain Service（本書では利用しない）

②「8. Domain Service」の判断どおり、`AnnouncementTargetingPolicy`・`AnnouncementVisibilityPolicy`はいずれも本機能では利用しない。管理者向けの対象指定は常に`all_users`固定であり、権限・所属校に応じた妥当性判定という可変のルールが存在しないためである（②の判断根拠をそのまま踏襲）。本書で新たなDomain Serviceは追加しない。

## Domain Error（本書で新規追加）

`domain/errors/errors.go`（教師お知らせ機能_Go実装仕様書が作成済みのファイル）に、以下を追加する。

|エラー変数|発生条件|
|-|-|
|`ErrTargetTypeNotAllowed`|管理者作成時にall_users以外の対象を指定しようとした場合（②「17. Error設計」の「管理者作成時にall_users以外の対象を指定しようとした場合（推測）」に対応）|

教師お知らせ機能_Go実装仕様書が定義した`ErrInvalidStatusTransition` / `ErrScheduledAtNotFuture` / `ErrEmptyTargets` / `ErrInvalidTargetCriteria`は、本機能でもそのまま発生しうる（状態遷移・予約日時ルールは共通のため）。`ErrTargetPermissionViolation` / `ErrTargetSchoolMismatch`（`AnnouncementTargetingPolicy`由来）は、本機能ではPolicyを呼び出さないため発生しない。

---

# 4. Application層設計

## DTO（Command / Query）

- struct名: `ListAdminAnnouncementsQuery`（配置: `application/dto/list_admin_announcements.go`）
  - フィールド: `Keyword string`, `Status string`, `Page dto.PageRequest`（`PageRequest`は教師お知らせ機能_Go実装仕様書が定義済みの型を再利用する）
  - 区分: Query
- struct名: `AdminAnnouncementListItem`
  - フィールド: `ID uint`, `Title string`, `Status string`, `TargetType string`, `PublishedAt *time.Time`, `ScheduledAt *time.Time`, `CreatedAt time.Time`, `PublisherName string`
  - 区分: 出力（②「19. API仕様」の「announcements（id, title, status, target_type, published_at, scheduled_at, created_at, publisher）」に対応）
- struct名: `ListAdminAnnouncementsResult`
  - フィールド: `Items []AdminAnnouncementListItem`, `PageInfo dto.PageInfo`
  - 区分: 出力
- struct名: `ShowAdminAnnouncementQuery`（配置: `application/dto/show_admin_announcement.go`）
  - フィールド: `AnnouncementID uint`
  - 区分: Query
- struct名: `AdminAnnouncementDetail`
  - フィールド: `ID uint`, `Title string`, `Content string`, `Status string`, `TargetType string`, `PublishedAt *time.Time`, `ScheduledAt *time.Time`, `CreatedAt time.Time`, `PublisherName string`
  - 区分: 出力
- struct名: `CreateAdminAnnouncementCommand`（配置: `application/dto/create_admin_announcement.go`）
  - フィールド: `CurrentAdminID uint`, `Title string`, `Content string`, `Status string`, `ScheduledAt *time.Time`
  - 区分: Command（②「19. API仕様」の「announcement[title]（必須）, announcement[content]（必須）, announcement[status]（必須）, announcement[scheduled_at]（scheduled指定時必須）」に対応）
- struct名: `UpdateAdminAnnouncementCommand`（配置: `application/dto/update_admin_announcement.go`）
  - フィールド: `CurrentAdminID uint`, `AnnouncementID uint`, `Title *string`, `Content *string`, `Status *string`, `ScheduledAt *time.Time`
  - 区分: Command（フィールドはいずれも任意。②「19. API仕様」の「announcement[title]/[content]/[status]/[scheduled_at]（いずれも任意）」に対応）
- struct名: `DeleteAdminAnnouncementCommand`
  - フィールド: `CurrentAdminID uint`, `AnnouncementID uint`
  - 区分: Command
- struct名: `PublishAdminAnnouncementCommand`
  - フィールド: `CurrentAdminID uint`, `AnnouncementID uint`
  - 区分: Command
- struct名: `ListHighSchoolAnnouncementsQuery`（配置: `application/dto/list_high_school_announcements.go`）
  - フィールド: `HighSchoolID uint`, `Page dto.PageRequest`
  - 区分: Query
- struct名: `HighSchoolAnnouncementListItem`
  - フィールド: `ID uint`, `Title string`, `Content string`, `Status string`, `PublishedAt *time.Time`, `ScheduledAt *time.Time`, `CreatedAt time.Time`, `PublisherName string`, `Targets []string`
  - 区分: 出力（②「19. API仕様」の「announcements（id, title, content, status, published_at, scheduled_at, created_at, publisher, targets）」に対応）
- struct名: `ListHighSchoolAnnouncementsResult`
  - フィールド: `Items []HighSchoolAnnouncementListItem`, `PageInfo dto.PageInfo`
  - 区分: 出力

**②からの補足**: 各DTOのフィールド構成は②「19. API仕様」のResponse定義から導出した。`PublisherName`（発信者名）の解決方法（`AnnouncementRepository`実装がJOINで取得するか、別途User Contextを参照するか）は②に明記がないため、教師お知らせ機能③文書と同様、Repository実装がJOINクエリで取得する構成とした（推測）。

## UseCase

### ListAdminAnnouncementsUseCase（`application/usecase/list_admin_announcements_usecase.go`）

- struct名: `ListAdminAnnouncementsUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`（教師お知らせ機能_Go実装仕様書で定義済みのInterfaceを、本書が追加した`SearchByPublisherRole`メソッドとあわせて利用する）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ListAdminAnnouncementsQuery) (dto.ListAdminAnnouncementsResult, error)`
- 処理ステップ:
  1. `query.Status`が指定されている場合、`valueobject.NewAnnouncementStatus(query.Status)`で検証する（不正な値の場合は絞り込み条件として無視する。②「15. Validation設計」のDomain節に対応する判断。②に明記のない挙動のため「推測」）
  2. `AnnouncementRepository.SearchByPublisherRole(ctx, query.Keyword, status, query.Page)`を呼び出す
  3. 取得結果を`AdminAnnouncementListItem`一覧へ変換して返す
- トランザクション境界: なし（②「14. Transaction設計」により読み取りのみ）
- 発生しうるApplication Error: なし

### ShowAdminAnnouncementUseCase（`application/usecase/show_admin_announcement_usecase.go`）

- struct名: `ShowAdminAnnouncementUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ShowAdminAnnouncementQuery) (dto.AdminAnnouncementDetail, error)`
- 処理ステップ:
  1. `AnnouncementRepository.FindByIDAndPublisherRole(ctx, query.AnnouncementID)`で対象を取得する
  2. 取得できない場合、`ErrAnnouncementNotFound`（教師お知らせ機能_Go実装仕様書が定義済みのApplication Errorを再利用する）を返す
  3. `AdminAnnouncementDetail`を組み立てて返す
- トランザクション境界: なし
- 発生しうるApplication Error: `ErrAnnouncementNotFound`

### CreateAdminAnnouncementUseCase（`application/usecase/create_admin_announcement_usecase.go`）

- struct名: `CreateAdminAnnouncementUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.CreateAdminAnnouncementCommand) (dto.CreateAdminAnnouncementResult, error)`
- 処理ステップ（②「12. UseCase設計」の「教師向けのCreateAnnouncementUseCaseと異なり、対象指定の妥当性判定（AnnouncementTargetingPolicy）を経由せず、TargetCriteria（all_users）を直接生成する」）:
  1. `valueobject.NewTargetCriteria("all_users", nil, nil, nil)`で対象指定（全ユーザー固定）を生成する
  2. `[]*entity.AnnouncementTarget{ target }`を組み立てる（`entity.NewAnnouncementTarget(criteria)`を1件のみ呼び出す）
  3. `entity.NewAnnouncement(cmd.CurrentAdminID, cmd.Title, cmd.Content, targets)`でAggregateを生成する（生成直後の状態は`draft`固定。②「6. Entity設計」の記載どおり教師向けと共通のEntityロジックを利用する）
  4. `cmd.Status`が`scheduled`の場合、`announcement.Schedule(*cmd.ScheduledAt)`を呼び出す（未来日時でない場合は`ErrScheduledAtNotFuture`）
  5. `cmd.Status`が`published`の場合、`announcement.Publish()`を呼び出す
  6. `AnnouncementRepository.Create(ctx, announcement)`で永続化する（Announcement本体とAnnouncementTarget（1件）を1トランザクションとして扱う。詳細は「8. Transaction実装方針」）
  7. `dto.CreateAdminAnnouncementResult`を返す
- トランザクション境界: Announcement本体とAnnouncementTarget（all_users 1件）の作成を1トランザクションとする（②「14. Transaction設計」）
- 発生しうるDomain Error: `ErrScheduledAtNotFuture`, `ErrInvalidStatusTransition`（③からの補足: `status`に不正な値が指定された場合、`valueobject.NewAnnouncementStatus`が返すエラーとして扱う）

### UpdateAdminAnnouncementUseCase（`application/usecase/update_admin_announcement_usecase.go`）

- struct名: `UpdateAdminAnnouncementUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.UpdateAdminAnnouncementCommand) (dto.UpdateAdminAnnouncementResult, error)`
- 処理ステップ（②「12. UseCase設計」「16. Authorization設計」の記載どおり、発信者ロール確認に加えて操作権限確認を行う）:
  1. `AnnouncementRepository.FindByIDAndPublisherRole(ctx, cmd.AnnouncementID)`で対象を取得する
  2. 取得できない場合、`ErrAnnouncementNotFound`を返す
  3. 操作権限を確認する（「10. Authorization実装方針」参照）。`announcement.IsOwnedBy(cmd.CurrentAdminID)`が`false`の場合は`AnnouncementRepository.IsPublisherDeactivated(ctx, announcement.PublisherID())`を呼び出し、それも`false`（発信者が無効化されていない）であれば`ErrAnnouncementOperationForbidden`を返す。この確認は、以降の状態確認（Entityによる配信済み判定）より先に行う
  4. `cmd.Title` / `cmd.Content`が指定されている場合、対象Entityのタイトル・本文を更新する（②「6. Entity設計」に更新用methodの明記はないため、実装時にEntityへ`UpdateContent(title, content string) error`相当のメソッド追加が必要になる可能性がある。詳細は「14. ②からの補足事項」参照）
  5. `cmd.Status`が指定されている場合、`scheduled`なら`Schedule`、`published`なら`Publish`を呼び出す（Entity側で許可されない遷移は`ErrInvalidStatusTransition`として拒否される。②「17. Error設計」の「配信済みのお知らせを更新・削除・再配信しようとする」は、Entity側が`published`状態からの`Schedule`/`Publish`呼び出しを拒否することで実現する）
  6. `AnnouncementRepository.Update(ctx, announcement)`で永続化する
  7. `dto.UpdateAdminAnnouncementResult`を返す
- トランザクション境界: Announcementの更新を1トランザクションとする（②「14. Transaction設計」）
- 発生しうるApplication Error: `ErrAnnouncementNotFound`, `ErrAnnouncementOperationForbidden`
- 発生しうるDomain Error: `ErrInvalidStatusTransition`, `ErrScheduledAtNotFuture`

**②からの補足**: タイトル・本文の更新について、教師お知らせ機能_Go実装仕様書のAnnouncement Entityは「作成後の更新APIは存在しないため更新用methodは設けない」と明記しており、`UpdateAnnouncementStatusUseCase`という状態のみを更新するUseCaseしか定義していない。一方、②「12. UseCase設計」は管理者向けUpdateAnnouncementUseCaseの入力として「title/content/status/scheduled_at（いずれも任意）」を明記しており、内容の更新も対象に含めている。この差異は②同士（教師お知らせ機能②と管理者お知らせ管理機能②）の間に存在するものであり、本書では管理者向け②の記載を優先し、Announcement Entityに`UpdateContent`相当のメソッド追加が必要になる可能性がある点を推測として明記する。既存Entityへのメソッド追加は「教師お知らせ機能_Go実装仕様書の内容を変更しない」という本書の前提に対する例外的な拡張であり、教師向けの既存メソッド（`Schedule`/`Publish`等）には影響しない加算的な変更として扱う。

### DeleteAdminAnnouncementUseCase（`application/usecase/delete_admin_announcement_usecase.go`）

- struct名: `DeleteAdminAnnouncementUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.DeleteAdminAnnouncementCommand) (dto.DeleteAdminAnnouncementResult, error)`
- 処理ステップ:
  1. `AnnouncementRepository.FindByIDAndPublisherRole(ctx, cmd.AnnouncementID)`で対象を取得する
  2. 取得できない場合、`ErrAnnouncementNotFound`を返す
  3. 操作権限を確認する（Update UseCaseの手順3と同じ。「10. Authorization実装方針」参照）。権限がない場合は`ErrAnnouncementOperationForbidden`を返す。この確認は、以降の配信済み判定より先に行う
  4. 対象の`Status()`が`published`の場合、`ErrCannotDeletePublished`（本書で新規追加。②「17. Error設計」の「配信済みのお知らせを更新・削除・再配信しようとする」に対応）を返す
  5. `AnnouncementRepository.Delete(ctx, cmd.AnnouncementID)`で削除する
  6. `dto.DeleteAdminAnnouncementResult`を返す
- トランザクション境界: Announcement本体とAnnouncementTargetの削除を1トランザクションとする（②「14. Transaction設計」）
- 発生しうるApplication Error: `ErrAnnouncementNotFound`, `ErrAnnouncementOperationForbidden`
- 発生しうるDomain Error: `ErrCannotDeletePublished`

### PublishAdminAnnouncementUseCase（`application/usecase/publish_admin_announcement_usecase.go`）

- struct名: `PublishAdminAnnouncementUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd dto.PublishAdminAnnouncementCommand) (dto.PublishAdminAnnouncementResult, error)`
- 処理ステップ:
  1. `AnnouncementRepository.FindByIDAndPublisherRole(ctx, cmd.AnnouncementID)`で対象を取得する
  2. 取得できない場合、`ErrAnnouncementNotFound`を返す
  3. 操作権限を確認する（Update UseCaseの手順3と同じ。「10. Authorization実装方針」参照）。権限がない場合は`ErrAnnouncementOperationForbidden`を返す。この確認は、以降の配信済み判定より先に行う
  4. `announcement.Publish()`を呼び出す（既にpublishedの場合は`ErrInvalidStatusTransition`が返る。②「12. UseCase設計」の「既に配信済みのお知らせへの再配信を防ぐ判定はAnnouncement Entityが担う」）
  5. `AnnouncementRepository.Update(ctx, announcement)`で永続化する
  6. `dto.PublishAdminAnnouncementResult`を返す
- トランザクション境界: 状態更新（published_atの確定、scheduled_atのクリアを含む）を1トランザクションとする（②「14. Transaction設計」）
- 発生しうるApplication Error: `ErrAnnouncementNotFound`, `ErrAnnouncementOperationForbidden`
- 発生しうるDomain Error: `ErrInvalidStatusTransition`

### ListHighSchoolAnnouncementsUseCase（`application/usecase/list_high_school_announcements_usecase.go`）

- struct名: `ListHighSchoolAnnouncementsUseCase`
- コンストラクタが受け取る依存: `AnnouncementRepository`, `HighSchoolRepository`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query dto.ListHighSchoolAnnouncementsQuery) (dto.ListHighSchoolAnnouncementsResult, error)`
- 処理ステップ:
  1. `HighSchoolRepository.FindByID(ctx, query.HighSchoolID)`で対象高校の存在を確認する
  2. 取得できない場合、`ErrHighSchoolNotFound`（本書で新規追加）を返す
  3. `AnnouncementRepository.SearchByHighSchoolTarget(ctx, query.HighSchoolID, query.Page)`を呼び出す（発信者ロールによる絞り込みは行わない。②「12. UseCase設計」の「発信者が教師であるお知らせも含めて検索する必要があるため、発信者ロールによる絞り込みは行わない」）
  4. `HighSchoolAnnouncementListItem`一覧へ変換して返す
- トランザクション境界: なし（②「14. Transaction設計」）
- 発生しうるApplication Error: `ErrHighSchoolNotFound`

---

# 5. Infrastructure層設計

## Repository実装（既存実装へのメソッド追加）

### AnnouncementRepository実装（`infrastructure/repository/announcement_repository.go`、既存実装への追記）

教師お知らせ機能_Go実装仕様書が定義した実装（package `gormrepo`内の`AnnouncementRepository`）に、以下のメソッド実装を追加する。

|メソッド|発行するクエリ内容|
|-|-|
|`SearchByPublisherRole`|`publisher_id`が管理者ロールのユーザーであること（`users`と`user_roles`の結合による絞り込み）を条件に、`keyword`が指定されていれば`title LIKE ?`、`status`が指定されていれば一致条件を付加する。`created_at`降順、`page.Page`/`page.PerPage`によるOFFSET/LIMITページング。`users`とJoinして発信者名を取得する（②「21. DB操作仕様」）|
|`FindByIDAndPublisherRole`|`id`一致かつ`publisher_id`が管理者ロールのユーザーであることの条件で1件取得する（`FindByIDAndOwner`との違いは`publisher_id = 特定の1人`ではなく`publisher_idのロールがadmin`である点）。発信者が無効化されている場合も取得対象に含める（`users`とのJoinで論理削除済みユーザーを除外しない）|
|`IsPublisherDeactivated`|`users`から`id = publisherID`のレコードを取得し、`deleted_at`が設定されているかを返す。無効化（論理削除）済みのユーザーを判定対象とするため、GORMの論理削除による自動除外を行わない（`Unscoped()`を用いる、または`deleted_at`を明示的に参照する）|
|`Delete`|`id`一致条件で`announcement_targets`を削除した後、`announcements`を削除する（Aggregate全体を1トランザクション内で削除する。詳細は「8. Transaction実装方針」）|
|`SearchByHighSchoolTarget`|`announcement_targets`とJoinし、`target_type = 'by_school' AND high_school_id = ?`、または`target_type = 'by_grade'`で当該高校に属する学年、または`target_type = 'by_user'`で当該高校に所属するユーザーのいずれかに一致するお知らせを検索する（発信者ロールによる絞り込みは行わない）。`created_at`降順、ページングを行う|

**②からの補足**: `SearchByHighSchoolTarget`の具体的な結合条件（`by_grade`/`by_user`経由で高校所属を辿る具体的なJOIN方法）は①未提供のため参照不可。②「21. DB操作仕様」の「`announcement_targets`のhigh_school_id（またはそれに紐づくgrade_id/user_id経由の高校所属）が指定高校と一致するもの」という記載から実装方針を導出したが、実際のテーブル結合（`grades.high_school_id`・`users.high_school_id`等）は実装時に既存スキーマを確認して確定する必要がある（推測）。

### HighSchoolRepository実装（`infrastructure/repository/high_school_repository.go`）

- 実装struct名: 非公開struct（例: `highSchoolRepository`）+ コンストラクタ`NewHighSchoolRepository(db *gorm.DB) repository.HighSchoolRepository`
- `high_schools`テーブルはmaster-data Context（共通マスタ参照機能）が正規の所有者であるため、本Contextでは対応するGORMモデルを定義しない
- `FindByID(ctx, id)`: `masterdata.FindHighSchoolByID(ctx, db, id)`（共通マスタ参照機能_Go実装仕様書「6. Application層設計」）を呼び出し、戻り値の`*masterdata.HighSchoolDetail`（`ID` / `Name`）を本Contextの`HighSchool`へ変換する。対象が存在しない場合は`masterdata.FindHighSchoolByID`が返す`(nil, nil)`を受けて`nil`を返す

## 外部連携実装

対象外。②に本機能でのMail・Cache・Queue連携要件の記載はない。予約状態のお知らせを配信予定日時に自動的に配信する非同期処理は、②「18. Domain Event」の記載どおり現行実装に存在せず、本書でも実装対象に含めない。

---

# 6. Presentation層設計

## Handler

### AdminAnnouncementHandler（`presentation/handler/admin_announcement_handler.go`）

- struct名: `AdminAnnouncementHandler`
- 対応する呼び出し先: `ListAdminAnnouncementsUseCase`, `ShowAdminAnnouncementUseCase`, `CreateAdminAnnouncementUseCase`, `UpdateAdminAnnouncementUseCase`, `DeleteAdminAnnouncementUseCase`, `PublishAdminAnnouncementUseCase`
- メソッド一覧:
  |メソッド|HTTPメソッド|パス|
  |-|-|-|
  |`List`|GET|`/api/v1/admin/announcements`|
  |`Show`|GET|`/api/v1/admin/announcements/:id`|
  |`Create`|POST|`/api/v1/admin/announcements`|
  |`Update`|PATCH|`/api/v1/admin/announcements/:id`|
  |`Delete`|DELETE|`/api/v1/admin/announcements/:id`|
  |`Publish`|POST|`/api/v1/admin/announcements/:id/publish`|
- 処理順序:
  - `List`: クエリパラメータ（`q`, `status`, `page`, `per_page`）を`request.AdminAnnouncementListRequest`にバインド → 型チェック → `ListAdminAnnouncementsUseCase.Execute`を呼び出す → `response.AdminAnnouncementListResponse`へ変換し200で返す
  - `Show`: パスパラメータ`id`をバインド → `ShowAdminAnnouncementUseCase.Execute`を呼び出す → `ErrAnnouncementNotFound`の場合は404に変換 → 成功時は`response.AdminAnnouncementDetailResponse`へ変換し200で返す
  - `Create`: リクエストボディ（`announcement[title]`/`[content]`/`[status]`/`[scheduled_at]`）を`request.AdminAnnouncementCreateRequest`にバインド → 必須・フォーマット検証 → `CreateAdminAnnouncementUseCase.Execute`を呼び出す → 成功時は`{"message": "お知らせを作成しました。"}`を200で返す（②「19. API仕様」）
  - `Update`: パスパラメータ`id`＋リクエストボディを`request.AdminAnnouncementUpdateRequest`にバインド → 認証済みユーザーのIDを`CurrentAdminID`として`UpdateAdminAnnouncementUseCase.Execute`を呼び出す → `ErrAnnouncementNotFound`は404、`ErrAnnouncementOperationForbidden`は403に変換 → 成功時は`{"message": "お知らせを更新しました。"}`を200で返す
  - `Delete`: パスパラメータ`id`をバインド → 認証済みユーザーのIDを`CurrentAdminID`として`DeleteAdminAnnouncementUseCase.Execute`を呼び出す → `ErrAnnouncementNotFound`は404、`ErrAnnouncementOperationForbidden`は403に変換 → 成功時は204（本文なし）を返す
  - `Publish`: パスパラメータ`id`をバインド → 認証済みユーザーのIDを`CurrentAdminID`として`PublishAdminAnnouncementUseCase.Execute`を呼び出す → `ErrAnnouncementNotFound`は404、`ErrAnnouncementOperationForbidden`は403に変換 → 成功時は`{"message": "お知らせを配信しました。"}`を200で返す

### AdminHighSchoolAnnouncementHandler（`presentation/handler/admin_high_school_announcement_handler.go`）

- struct名: `AdminHighSchoolAnnouncementHandler`
- 対応する呼び出し先: `ListHighSchoolAnnouncementsUseCase`
- メソッド一覧:
  |メソッド|HTTPメソッド|パス|
  |-|-|-|
  |`List`|GET|`/api/v1/admin/high_schools/:high_school_id/announcements`|
- 処理順序: パスパラメータ`high_school_id`＋クエリパラメータ`page`をバインド → `ListHighSchoolAnnouncementsUseCase.Execute`を呼び出す → `ErrHighSchoolNotFound`の場合は404に変換 → 成功時は`response.HighSchoolAnnouncementListResponse`へ変換し200で返す

## Request / Response DTO

- struct名: `AdminAnnouncementListRequest`（`presentation/request/admin_announcement_request.go`）
  - フィールド: `Q string`（`form:"q"`）, `Status string`（`form:"status"`）, `Page int`（`form:"page"`）, `PerPage int`（`form:"per_page"`）
  - バリデーションタグ／チェック内容: いずれも`binding:"omitempty"`。`status`は文字列としての型チェックのみで、許可値判定はDomain層（`AnnouncementStatus`）に委ねる
- struct名: `AdminAnnouncementCreateRequest`
  - フィールド: `Announcement AdminAnnouncementCreateBody`（`json:"announcement" binding:"required"`）
  - `AdminAnnouncementCreateBody`のフィールド: `Title string`（`json:"title" binding:"required,max=255"`）, `Content string`（`json:"content" binding:"required,max=10000"`）, `Status string`（`json:"status" binding:"required,oneof=draft scheduled published"`）, `ScheduledAt *time.Time`（`json:"scheduled_at" binding:"required_if=Status scheduled"`）
  - バリデーションタグ／チェック内容: ②「15. Validation設計」の「title: 必須、255文字以内／content: 必須、10,000文字以内／status: draft/scheduled/publishedのいずれか」「scheduled_at: status=scheduled指定時は必須」に対応
- struct名: `AdminAnnouncementUpdateRequest`
  - フィールド: `Announcement AdminAnnouncementUpdateBody`（`json:"announcement" binding:"required"`）
  - `AdminAnnouncementUpdateBody`のフィールド: `Title *string`（`json:"title" binding:"omitempty,max=255"`）, `Content *string`（`json:"content" binding:"omitempty,max=10000"`）, `Status *string`（`json:"status" binding:"omitempty,oneof=draft scheduled published"`）, `ScheduledAt *time.Time`（`json:"scheduled_at"`）
  - バリデーションタグ／チェック内容: いずれのフィールドも任意（②「19. API仕様」の「いずれも任意」に対応）
- struct名: `AdminAnnouncementListItemResponse`（`presentation/response/admin_announcement_response.go`）
  - フィールド: `ID uint`（json: `id`）, `Title string`（json: `title`）, `Status string`（json: `status`）, `TargetType string`（json: `target_type`）, `PublishedAt *time.Time`（json: `published_at`）, `ScheduledAt *time.Time`（json: `scheduled_at`）, `CreatedAt time.Time`（json: `created_at`）, `Publisher string`（json: `publisher`）
- struct名: `AdminAnnouncementListResponse`
  - フィールド: `Announcements []AdminAnnouncementListItemResponse`（json: `announcements`）, `Meta MetaResponse`（json: `meta`）
- struct名: `AdminAnnouncementDetailResponse`
  - フィールド: `ID uint`, `Title string`, `Content string`, `Status string`, `TargetType string`, `PublishedAt *time.Time`, `ScheduledAt *time.Time`, `CreatedAt time.Time`, `Publisher string`
- struct名: `HighSchoolAnnouncementListItemResponse`
  - フィールド: `ID uint`, `Title string`, `Content string`, `Status string`, `PublishedAt *time.Time`, `ScheduledAt *time.Time`, `CreatedAt time.Time`, `Publisher string`, `Targets []string`（json: `targets`）
- struct名: `HighSchoolAnnouncementListResponse`
  - フィールド: `Announcements []HighSchoolAnnouncementListItemResponse`, `Meta MetaResponse`

## Routing

`presentation/routes.go`（教師お知らせ機能_Go実装仕様書で作成済みのファイル）へ、以下のルート登録を追加する。

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/admin/announcements`|`AdminAnnouncementHandler.List`|
|GET|`/api/v1/admin/announcements/:id`|`AdminAnnouncementHandler.Show`|
|POST|`/api/v1/admin/announcements`|`AdminAnnouncementHandler.Create`|
|PATCH|`/api/v1/admin/announcements/:id`|`AdminAnnouncementHandler.Update`|
|DELETE|`/api/v1/admin/announcements/:id`|`AdminAnnouncementHandler.Delete`|
|POST|`/api/v1/admin/announcements/:id/publish`|`AdminAnnouncementHandler.Publish`|
|GET|`/api/v1/admin/high_schools/:high_school_id/announcements`|`AdminHighSchoolAnnouncementHandler.List`|

全ルートに、adminロールを要求する認証Middlewareを適用する（②「16. Authorization設計」）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/admin/announcements|AdminAnnouncementHandler.List|AdminAnnouncementListRequest|AdminAnnouncementListResponse|200|
|GET|/api/v1/admin/announcements/:id|AdminAnnouncementHandler.Show|-（パスパラメータのみ）|AdminAnnouncementDetailResponse|200|
|POST|/api/v1/admin/announcements|AdminAnnouncementHandler.Create|AdminAnnouncementCreateRequest|`{"message": string}`|200|
|PATCH|/api/v1/admin/announcements/:id|AdminAnnouncementHandler.Update|AdminAnnouncementUpdateRequest|`{"message": string}`|200|
|DELETE|/api/v1/admin/announcements/:id|AdminAnnouncementHandler.Delete|-（パスパラメータのみ）|-（本文なし）|204|
|POST|/api/v1/admin/announcements/:id/publish|AdminAnnouncementHandler.Publish|-（パスパラメータのみ）|`{"message": string}`|200|
|GET|/api/v1/admin/high_schools/:high_school_id/announcements|AdminHighSchoolAnnouncementHandler.List|パスパラメータ`high_school_id`, クエリ`page`|HighSchoolAnnouncementListResponse|200|

## Errorケース

|Endpoint|条件|Status Code|Error内容|
|-|-|-|-|
|Show/Update/Delete/Publish|対象お知らせが存在しない、または発信者が管理者ロールでない|404|`ErrAnnouncementNotFound`|
|Update/Delete/Publish|発信者本人でない管理者による操作（発信者が無効化されている場合を除く）。配信済みかどうかの判定より先に行われる|403|`ErrAnnouncementOperationForbidden`（メッセージ「この操作を行う権限がありません」）|
|Create|title/content/status未入力・形式不正|422|Request DTOバリデーションエラー|
|Create/Update|scheduled指定時にscheduled_atが未来日時でない|422|`ErrScheduledAtNotFuture`|
|Update|不正な状態遷移（published状態への不正な操作を含む）|422|`ErrInvalidStatusTransition`|
|Delete/Update/Publish|配信済み（published）のお知らせへの操作|422|`ErrInvalidStatusTransition`（更新・配信時）／`ErrCannotDeletePublished`（削除時）|
|高校別お知らせ参照|対象高校が存在しない|404|`ErrHighSchoolNotFound`|
|全Endpoint共通|未認証|401|認証エラー（②に明記なし。推測）|
|全Endpoint共通|adminロールでない|403|認可エラー（②に明記なし。推測）|
|全Endpoint共通|予期せぬDB接続失敗等|500|Infrastructure Error|

---

# 8. Transaction実装方針

## Transaction開始箇所

- 書き込みを伴うUseCase（`CreateAdminAnnouncementUseCase` / `UpdateAdminAnnouncementUseCase` / `DeleteAdminAnnouncementUseCase` / `PublishAdminAnnouncementUseCase`）の実行中、教師お知らせ機能_Go実装仕様書と同一の方針（Repository実装内でGORMトランザクションを開始する）に従う（教師お知らせ機能③「8. Transaction実装方針」の「UseCaseからGORMへの直接依存（依存方向違反）を避けるため、実装上はRepository実装内でトランザクションを完結させる方式とした」を踏襲）

## Transaction終了箇所

- `Create`: `announcements`への1件INSERTと`announcement_targets`への1件（all_users）INSERTが成功した時点でコミットする
- `Update`: `announcements`の内容・状態・日時UPDATEが成功した時点でコミットする
- `Delete`: `announcement_targets`の削除・`announcements`の削除が成功した時点でコミットする
- `Publish`（内部的には`Update`を呼び出す）: 状態更新が成功した時点でコミットする

## 複数Repositoryにまたがる場合の扱い

- `ListHighSchoolAnnouncementsUseCase`は`HighSchoolRepository`（読み取り専用）と`AnnouncementRepository`（読み取り専用）を呼び出すが、いずれも参照のみであるためトランザクション制御は行わない
- 本機能内でAnnouncementRepository以外に書き込みを行うRepositoryは存在しない

②「14. Transaction設計」に明記されているとおり、1回の業務操作（作成・更新・削除・配信）に対してAnnouncementとAnnouncementTargetの整合性を保つため、UseCase単位でトランザクション境界を統一する。

---

# 9. Validation実装方針

## Presentation

- `AdminAnnouncementCreateRequest`でのチェック内容: `title`必須・255文字以内、`content`必須・10,000文字以内、`status`が`draft`/`scheduled`/`published`のいずれか、`scheduled_at`は`status=scheduled`指定時必須（②「15. Validation設計」）
- `AdminAnnouncementUpdateRequest`でのチェック内容: いずれのフィールドも任意。指定された場合のみ同様の形式チェックを行う
- `AdminAnnouncementListRequest`でのチェック内容: `q`/`status`/`page`/`per_page`の型チェックのみ

## 業務ルール検証

Domain Model採用のため、②の記載どおりEntity／Value Object生成時に検証する。

- `valueobject.NewAnnouncementStatus`: draft/scheduled/published以外の値を拒否する
- `valueobject.NewScheduledAt`: `status=scheduled`指定時の未来日時チェック
- `entity.Announcement.Schedule` / `Publish`: 状態遷移が許可された組み合わせかどうか（published状態からの遷移を含め拒否する）
- `valueobject.NewTargetCriteria("all_users", nil, nil, nil)`: 管理者作成時は常にこの呼び出しのみを行い、他の対象種別を生成しない（②「6. Entity設計」の「AnnouncementTargetの追加される責務」に対応。UseCase側の実装によって「all_users以外を許容しない」制約を保証する）

## 責務分離

②「15. Validation設計」の記載どおり、Presentationは「入力の形式が正しいか」を担当し、Domainは「業務的に妥当か（対象固定・状態遷移・発信者スコープ）」を担当する。

---

# 10. Authorization実装方針

## Middlewareで行う処理

- 認証済みユーザーを特定し、`admin`ロールであることを確認する（②「16. Authorization設計」）

## Handlerで行う処理

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定は持たせない

## UseCaseで行う処理

- 一覧・詳細・更新・削除・配信は、発信者が管理者ロールであるお知らせ全体をスコープとする（`AnnouncementRepository.SearchByPublisherRole` / `FindByIDAndPublisherRole`が担う。`FindByIDAndOwner`は使用しない）
- 更新・削除・配信は、スコープ確認に加えて操作権限確認を行う（②「16. Authorization設計」）。`FindByIDAndPublisherRole`で取得した`announcement`に対し、次の順で判定する
  1. `announcement.IsOwnedBy(cmd.CurrentAdminID)`が`true`であれば許可する（既存Entityのメソッドをそのまま再利用する）
  2. `false`の場合は`AnnouncementRepository.IsPublisherDeactivated(ctx, announcement.PublisherID())`を呼び出し、`true`（発信者が無効化されている）であれば許可する
  3. いずれにも該当しなければ`ErrAnnouncementOperationForbidden`を返す
- 操作権限確認は、Entityによる配信済みかどうかの判定（`Publish` / 削除時の`Status()`確認）より先に行う。他の管理者が発信した配信済みのお知らせへの更新・削除・配信は`ErrAnnouncementOperationForbidden`（403）となり、発信者本人による配信済みのお知らせへの操作は権限確認を通過して`ErrInvalidStatusTransition` / `ErrCannotDeletePublished`（422）となる
- 一覧・詳細は操作権限確認の対象外であり、他の管理者が発信したお知らせも参照できる
- 高校別お知らせ参照は、発信者のロールを問わず、指定高校を対象に含むお知らせ全体をスコープとする

## Domainで行う処理

- `Announcement` Entity（`Schedule`/`Publish`）が、許可されない状態遷移・published状態への操作を拒否する

## 判断理由

②「16. Authorization設計」の判断理由（「誰がアクセスできるか」という認証・ロール確認はMiddleware、「どの範囲のお知らせを操作できるか」というスコープ確認はUseCase、「業務上許される操作か」という判定はDomain）をそのまま踏襲する。参照は「管理者ロールが発信者であること」という広いスコープで可能であり、更新・削除・配信は発信者本人（発信者が無効化されている場合は他の管理者）に限られる。発信者アカウントの無効化状態はAnnouncement Entityが保持しない情報であるため、EntityではなくUseCaseで判定する。

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

- UseCaseは、教師お知らせ機能_Go実装仕様書と同じ方針（Domain Errorをラップし直さず、そのまま呼び出し元へ伝播させる）に従う
- Application Error（`ErrAnnouncementNotFound`は教師お知らせ機能_Go実装仕様書が定義済みのものを再利用する。`ErrHighSchoolNotFound`・`ErrAnnouncementOperationForbidden`は本書で新規に`application/usecase/errors.go`へ追加する）

## Application Error → HTTPレスポンスへの変換方針

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrInvalidStatusTransition`|Domain（既存Entity）|422|
|`ErrScheduledAtNotFuture`|Domain（既存Value Object）|422|
|`ErrEmptyTargets`|Domain（既存Entity）|422（管理者作成時はTargetCriteriaを1件固定で生成するため実質的には発生しない防御的エラー）|
|`ErrInvalidTargetCriteria`|Domain（既存Value Object）|422|
|`ErrCannotDeletePublished`|Domain（本書で新規追加）|422|
|`ErrAnnouncementNotFound`|Application（既存）|404|
|`ErrAnnouncementOperationForbidden`|Application（本書で新規追加）|403|
|`ErrHighSchoolNotFound`|Application（本書で新規追加）|404|
|Request DTOバリデーションエラー|Presentation|422|
|DB接続失敗・クエリ失敗|Infrastructure|500|

Handler側で`errors.Is`による判定を行い、上記対応表に従いHTTPステータスへ変換する。

## Infrastructure Errorのハンドリング方針

Repository実装が返すDBエラー（接続失敗・クエリ失敗）は`fmt.Errorf`でラップしてUseCase・Handlerへ伝播させ、Handlerで未分類のエラーとして一律500に変換する。

---

# 12. GORM / DBクエリ設計

②「20. DB設計方針」のとおり、`announcements`/`announcement_targets`テーブルはお知らせ機能・教師お知らせ機能と共有し、本機能単独でのスキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

- `gormmodel.AnnouncementModel` ⇔ `announcements`テーブル（教師お知らせ機能_Go実装仕様書で定義済みのモデルをそのまま利用する。新規モデル定義は行わない）
- `gormmodel.AnnouncementTargetModel` ⇔ `announcement_targets`テーブル（同上）
- `high_schools`テーブルに対応するGORMモデルは本Contextでは定義しない（master-data Context所有。`HighSchoolRepository`実装はmaster-data Contextが公開する`FindHighSchoolByID`関数呼び出しに委譲する。「5. Infrastructure層設計」参照）

## 主要クエリの条件・ソート・ページネーション方針

- `SearchByPublisherRole`: `users`・`user_roles`とJoinし発信者が管理者ロールであることを条件化。`keyword`指定時`title LIKE ?`、`status`指定時は一致条件。`created_at`降順、ページング（②「21. DB操作仕様」）
- `FindByIDAndPublisherRole`: `id`一致＋発信者が管理者ロールであることの条件
- `Delete`: `announcement_targets`削除後に`announcements`削除
- `SearchByHighSchoolTarget`: `announcement_targets`経由で指定高校を対象に含むものを検索（発信者ロールを問わない）。`created_at`降順、ページング
- `HighSchoolRepository.FindByID`: master-data Contextの`FindHighSchoolByID`関数呼び出しに委譲（本Contextでは`id`一致のクエリを直接発行しない）

SQL文そのものは記載しない。

## 既存Schemaに対する変更

②20章のとおり変更なし。

---

# 13. テストケース設計

②「22. テスト戦略」で採用パターンがDomain Modelであるため、区分はそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|`Announcement.Schedule` / `Publish`（既存Entityの再利用確認）|管理者作成シナリオ（発信者=管理者、対象=all_users固定）でも、教師お知らせ機能で検証済みの状態遷移ルールが同様に機能すること|
|`valueobject.NewTargetCriteria("all_users", nil, nil, nil)`|全ユーザー固定の対象指定が正しく生成されること|

## UseCase Test

|対象|テストケース|
|-|-|
|`ListAdminAnnouncementsUseCase`|`keyword`/`status`による絞り込みが正しく反映されること／他の管理者が作成したお知らせも一覧に含まれること（②の重点検証項目）|
|`ShowAdminAnnouncementUseCase`|発信者が管理者ロールであるお知らせを取得できること／存在しない、または教師発信のお知らせでは`ErrAnnouncementNotFound`を返すこと|
|`CreateAdminAnnouncementUseCase`|作成されたお知らせの対象が常に`all_users`1件であること／`status=scheduled`指定時に`scheduled_at`が未来日時でない場合エラーになること|
|`UpdateAdminAnnouncementUseCase`|内容・状態の更新が正しく反映されること／published状態のお知らせへの更新が拒否されること／発信者本人による更新が許可されること／他の管理者が発信したお知らせの更新が`ErrAnnouncementOperationForbidden`で拒否されること／発信者が無効化されている場合は他の管理者による更新が許可されること／他の管理者が発信した配信済みのお知らせの更新が`ErrInvalidStatusTransition`ではなく`ErrAnnouncementOperationForbidden`となること|
|`DeleteAdminAnnouncementUseCase`|未配信のお知らせが削除できること／published状態のお知らせの削除が`ErrCannotDeletePublished`で拒否されること／他の管理者が発信したお知らせの削除が`ErrAnnouncementOperationForbidden`で拒否されること／発信者が無効化されている場合は他の管理者による削除が許可されること|
|`PublishAdminAnnouncementUseCase`|draft/scheduledから配信できること／既に配信済みの場合にエラーになること／他の管理者が発信したお知らせの配信が`ErrAnnouncementOperationForbidden`で拒否されること／発信者が無効化されている場合は他の管理者による配信が許可されること|
|`ListHighSchoolAnnouncementsUseCase`|指定高校を対象に含むお知らせが、発信者ロールを問わず（教師作成分も含め）取得できること（②の重点検証項目）／対象高校が存在しない場合に`ErrHighSchoolNotFound`を返すこと|

## Repository Test

|対象|テストケース|
|-|-|
|`AnnouncementRepository.SearchByPublisherRole`|発信者ロール絞り込み・キーワード・状態絞り込み・ページネーションの正確性|
|`AnnouncementRepository.FindByIDAndPublisherRole`|管理者発信のお知らせのみ取得できること（発信者が無効化されていても取得できること）|
|`AnnouncementRepository.IsPublisherDeactivated`|論理削除済みの発信者で`true`、有効な発信者で`false`が返ること|
|`AnnouncementRepository.Delete`|Announcement本体とAnnouncementTargetの両方が削除されること|
|`AnnouncementRepository.SearchByHighSchoolTarget`|指定高校を対象に含むお知らせが発信者ロールを問わず取得できること|
|`HighSchoolRepository.FindByID`|`masterdata.FindHighSchoolByID`への呼び出しが正しく行われること／戻り値の`HighSchoolDetail`が`HighSchool`へ正しく変換されること／存在しない高校IDで`nil`が返ること|

## Handler Test

|対象|テストケース|
|-|-|
|`AdminAnnouncementHandler.Create`|title/content/status未入力・形式不正で422が返ること／正常系で200が返ること|
|`AdminAnnouncementHandler.Update`|不正な状態遷移で422が返ること／存在しないIDで404が返ること／`ErrAnnouncementOperationForbidden`で403が返ること|
|`AdminAnnouncementHandler.Delete`|配信済みお知らせの削除で422が返ること／`ErrAnnouncementOperationForbidden`で403が返ること／成功時に204が返ること|
|`AdminAnnouncementHandler.Publish`|配信済みお知らせの再配信で422が返ること／`ErrAnnouncementOperationForbidden`で403が返ること|
|`AdminHighSchoolAnnouncementHandler.List`|存在しない高校IDで404が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|作成→更新→配信→削除試行|配信済み後の更新・削除・再配信がいずれも一貫して拒否されること（②の重点検証項目）|
|高校別お知らせ参照|教師作成のお知らせも含めて正しく返却されることを確認すること（②の重点検証項目）|
|認可|admin以外のロールでアクセスした場合に403が返ること／未認証の場合に401が返ること／他の管理者が発信したお知らせの更新・削除・配信が403となり、お知らせが変更されないこと／発信者が無効化されている場合は他の管理者が更新・削除・配信できること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に整理する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|`AnnouncementRepository`の`SearchByPublisherRole` / `FindByIDAndPublisherRole` / `Delete` / `SearchByHighSchoolTarget`という具体的なメソッド名・シグネチャ|②「11. Repository設計」は責務のみを記載しており、メソッド名・引数構成までは記載がないため、教師お知らせ機能③文書が採用した推測方針と同様の方法で補った|推測|
|`HighSchoolRepository`を本Context（announcement）内に新規定義し、`high_schools`テーブルへ直接クエリする実装とした|school-directory Context（管理者高校学年参照機能）がTransaction Script採用でRepository Interfaceを持たない構造のため、Domain Model採用の本ContextはRepositoryのInterfaceを自ら定義し、依存性逆転の原則を維持する必要がある。管理者ダッシュボード機能③文書・管理者コース・単元参照機能③文書が採用した「参照先Contextの③実装詳細が確認できない場合、読み取り専用のクエリを本Context側で独自定義する」という暫定方針と同一の判断を踏襲した|推測。school-directory Context側の実装詳細確認後に見直しの余地あり|
|`UpdateAdminAnnouncementUseCase`がタイトル・本文の更新を扱うため、既存のAnnouncement Entityに`UpdateContent`相当のメソッド追加が必要になる可能性がある点を明記した|教師お知らせ機能②は更新APIを状態遷移のみと定義しているのに対し、管理者お知らせ管理機能②はtitle/contentの更新も入力に含めている。この②同士の記述差を、既存Entityへの加算的なメソッド追加（教師向けの既存メソッドには影響しない）として整理した|②同士の記述差異を整合させるための判断（推測を含む）|
|`ErrCannotDeletePublished`という新規Domain Errorを追加した|②「17. Error設計」は「配信済みのお知らせを更新・削除・再配信の試行」を1つのDomain Errorとして記載しているが、削除操作はEntityの状態遷移メソッド（Schedule/Publish）を経由しないため、削除固有のエラーとして区別する必要があった|推測|
|更新・削除・配信の操作権限確認を、Announcement Entityへ新メソッドを追加せず、既存の`IsOwnedBy` / `PublisherID`と、`AnnouncementRepository`への`IsPublisherDeactivated`メソッド追加、および新規Application Error `ErrAnnouncementOperationForbidden`（403）で実現した|②「16. Authorization設計」は操作権限確認をUseCaseに配置すると定めているが、具体的なメソッド名・エラー名・発信者無効化状態の取得方法は②に明記がない。発信者アカウントの無効化状態はUser Context由来の情報でありAnnouncement Aggregateが保持しないため、Entity（教師お知らせ機能③が定義する「正」）の不変条件・メソッドは変更せず、Repository経由の参照で補った|推測|
|`ErrHighSchoolNotFound`を`application/usecase/errors.go`に新規定義した|②「17. Error設計」がApplication Errorとして「高校別お知らせ参照における対象高校が存在しない」を明記している一方、具体的な変数名・配置場所までは指定していないため|推測|
|Response DTOの発信者名（`Publisher`/`PublisherName`）の解決方法をRepository実装のJOINクエリとした|②に解決方法の明記がなく、教師お知らせ機能③文書が同様の情報（発信者情報）についてRepository実装のJOINを前提とした構成を採っていることに倣った|推測|
|`SearchByHighSchoolTarget`の具体的な結合条件（`by_grade`/`by_user`経由で高校所属を辿る方法）を確定しなかった|①（Rails実装の具体的なクエリ）が未提供のため参照不可。②「21. DB操作仕様」の記載から大枠の方針のみを導出した|推測（要・既存DBスキーマ確認）|

上記以外の設計判断（Bounded Context・Aggregate・Entity・Value Object・UseCase構成・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API仕様・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、教師お知らせ機能_Go実装仕様書が定義済みのAnnouncement/AnnouncementTarget Aggregateの設計にも変更を加えていない。
