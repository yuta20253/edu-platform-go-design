# クラス編成機能 Go実装仕様書 2 Infrastructure

---

## 本書の位置づけ（分割）

本機能の実装ファイルは40であり、1つの③が扱う30〜35を超えるため、③自動生成プロンプト「2. ディレクトリ構成」の「文書の分割」に従って分割している。Handlerが`school_class_handler.go`の1つのみでスライスに割れないため、**レイヤーで割っている。**

|文書|担当|実装ファイル数|テスト込みの目安|
|-|-|-|-|
|`クラス編成機能_Go実装仕様書_1_Domain.md`|Domain層|14|約23|
|`クラス編成機能_Go実装仕様書_2_Infrastructure.md`|Infrastructure層|9|約14|
|`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`|Application層・Presentation層|17|約25|

1つ目 → 2つ目 → 3つ目の順に実装すれば、各PRは単体でコンパイルが通る。Infrastructure層はDomain層のRepository Interfaceを実装し、Application層・Presentation層はその両方に依存するためである。

機能全体のファイル一覧と、どの文書がどのファイルを担当するかは「2. ディレクトリ構成」に記載する。

---

# 1. 機能概要

## 機能概要

学校の学年・クラス（組）の構成を管理する機能である。教師がクラスの新設・改名・削除を「申請」として提出し、他職員操作権限（`manage_other_teachers`）を持つ教師が承認・却下することでクラス構成（`school_classes`）へ反映する、承認ワークフロー型の機能である。承認された申請の内容は、申請レコードの更新と同一の処理単位でクラスデータへ反映される。削除申請は、対象クラスに在籍する生徒・所属する教員がいないことを条件とする。更新系操作（承認・却下・取消）は楽観ロック（`lock_version`）による競合検出を伴う。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により、本機能は **Domain Model** を採用する。

- `SchoolClassRequest`は「pending→approved/rejected/cancelled」という明示的な状態遷移ルールを持ち、承認・却下は他職員操作権限を持つ教師のみが行え、かつ申請者自身は自分の申請を承認・却下できないという当事者制約を持つ
- 承認時には、申請レコードの状態更新と、申請区分（新設・改名・削除）に応じたクラスデータへの反映を「一つの処理としてまとめて行い、途中で失敗した場合は両方とも反映されない」という、複数Entityにまたがる整合性ルールが存在する
- 削除申請は「対象クラスに在籍する生徒・所属する教員がいないこと」という、他Context（在籍・所属情報）を参照する業務ルールを満たさなければ受け付けられない
- 更新系操作には楽観ロックによる競合検出が必須である
- 申請区分ごとに必要な入力項目の組み合わせルールが異なる

以上の理由からTransaction Script・Active Record・Event Sourcingは不採用としている（②「4. 設計パターン」「24. 採用しなかった設計」）。本書はこの判断を変更しない。

## 本書が対象とする実装範囲

- Bounded Context: `school-class`
- 対象UseCase: `ListGradesUseCase` / `ListSchoolClassesUseCase` / `ShowSchoolClassUseCase` / `ListSchoolClassRequestsUseCase` / `ShowSchoolClassRequestUseCase` / `CreateSchoolClassRequestUseCase` / `ProcessSchoolClassRequestUseCase` / `CancelSchoolClassRequestUseCase`
- 規約「3. 設計パターンごとの構造適用方針」のDomain Model構造（`domain`/`application`/`infrastructure`/`presentation`のフルレイヤー構成）で実装する
- ①Rails現行仕様書（クラス編成機能_Rails現行仕様書）の記載を、各APIの入力項目・処理内容・業務ルール・レスポンス・Errorケースの根拠とする

---

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `school-class`（②「3. Bounded Context」）

## ②で採用した設計パターン

- Domain Model（②「4. 設計パターン」）

アーキテクチャ規約「4. Bounded Context構成」の「ディレクトリ構成」の命名規則（Context名のkebab-caseを`_`区切りに置き換え、短縮しない）により、ディレクトリ名は`internal/schoolclass/`となる。

## 作成するディレクトリ一覧

```
internal/schoolclass/
├── context.go                    # NewContext（依存関係の組み立て）
├── dependency.go                 # 他Contextへの依存を宣言するインターフェース
├── domain/
│   ├── entity/
│   ├── valueobject/
│   ├── repository/
│   ├── service/
│   ├── event/
│   └── errors/
├── application/
│   ├── dto/
│   └── usecase/
├── infrastructure/
│   ├── persistence/
│   │   └── gorm/
│   ├── repository/
│   └── queue/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

Domain Modelを採用するため、Contextディレクトリ直下には`context.go`と`dependency.go`のみを置き、他は`domain/`等のサブディレクトリに配置する（アーキテクチャ規約「4. Bounded Context構成」の「ディレクトリ構成」）。他Contextの参照手段のためのインターフェース（`ClassOccupancyRepository`・`TeacherPermissionRepository`・`SchoolClassRequestEventPublisher`）は`dependency.go`に置き、`domain/repository/`には本Contextが所有するデータのRepository Interfaceのみを置く。

- `domain/specification/`: 対象外（②に該当する複雑な仕様判定パターンの記載がなく、業務ルールはDomain Service（3種のPolicy）として②8章に明記されているため）
- `infrastructure/mail/`・`infrastructure/cache/`: 対象外（②に該当する外部連携要件の記載がないため）

## 作成するファイル一覧

```
internal/schoolclass/domain/entity/grade.go
internal/schoolclass/domain/entity/school_class.go
internal/schoolclass/domain/entity/school_class_request.go

internal/schoolclass/domain/valueobject/school_class_request_status.go
internal/schoolclass/domain/valueobject/school_class_request_action.go
internal/schoolclass/domain/valueobject/lock_version.go

internal/schoolclass/context.go
internal/schoolclass/dependency.go

internal/schoolclass/domain/repository/school_class_repository.go
internal/schoolclass/domain/repository/school_class_request_repository.go

internal/schoolclass/domain/service/school_class_request_reflection_policy.go
internal/schoolclass/domain/service/school_class_deletion_eligibility_policy.go
internal/schoolclass/domain/service/duplicate_pending_request_policy.go

internal/schoolclass/domain/event/events.go

internal/schoolclass/domain/errors/errors.go

internal/schoolclass/application/dto/grade_dto.go
internal/schoolclass/application/dto/school_class_dto.go
internal/schoolclass/application/dto/school_class_request_dto.go

internal/schoolclass/application/usecase/list_grades_usecase.go
internal/schoolclass/application/usecase/list_school_classes_usecase.go
internal/schoolclass/application/usecase/show_school_class_usecase.go
internal/schoolclass/application/usecase/list_school_class_requests_usecase.go
internal/schoolclass/application/usecase/show_school_class_request_usecase.go
internal/schoolclass/application/usecase/create_school_class_request_usecase.go
internal/schoolclass/application/usecase/process_school_class_request_usecase.go
internal/schoolclass/application/usecase/cancel_school_class_request_usecase.go
internal/schoolclass/application/usecase/errors.go

internal/schoolclass/infrastructure/gormrepo/grade_model.go
internal/schoolclass/infrastructure/gormrepo/school_class_model.go
internal/schoolclass/infrastructure/gormrepo/teacher_school_class_model.go
internal/schoolclass/infrastructure/gormrepo/school_class_request_model.go

internal/schoolclass/infrastructure/gormrepo/school_class_repository.go
internal/schoolclass/infrastructure/gormrepo/school_class_request_repository.go
internal/schoolclass/infrastructure/gormrepo/class_occupancy_repository.go
internal/schoolclass/infrastructure/gormrepo/teacher_permission_repository.go

internal/schoolclass/infrastructure/queue/school_class_request_event_publisher.go

internal/schoolclass/presentation/handler/school_class_handler.go
internal/schoolclass/presentation/request/school_class_request.go
internal/schoolclass/presentation/response/school_class_response.go
internal/schoolclass/presentation/routes.go
```

**②からの補足**: `ClassOccupancyRepository`（在籍生徒・所属教員の参照）・`TeacherPermissionRepository`（他職員操作権限の確認）は、②「11. Repository設計」の「外部参照Repository（在籍する生徒・所属する教員）」および②「16. Authorization設計」の記載を、アーキテクチャ規約「5. Context間連携ルール」に従い具体的なRepository Interfaceとして具体化したものである。`TeacherPermissionRepository`は、`teacher-permission` Contextが公開する参照手段に対応する（`teacher-permission`③「3. Domain層設計」の`TeacherPermissionRepository`は同Contextの内部実装であり、本Contextからは参照しない）。`ClassOccupancyRepository`のうち、在籍生徒の確認は`user` Contextが所有する`users`が対象であり、`user`②が`CountUsers`を公開しているが、その③が未作成のため、対象テーブルへの直接クエリで暫定対応する。所属教員の確認は`users`ではなく`teacher_school_classes`が対象であり、このテーブルを所有・管理するContextは、現行の②群に定義されていない（書き込む機能が現行の対象範囲に存在しない参照専用の依存。②「3. Bounded Context」）。

---

---

## 本書が担当するファイル

|文書|担当するファイル|
|-|-|
|`_1_Domain.md`|`dependency.go`、`domain/`配下のすべて（entity 3・valueobject 3・repository 2・service 3・event 1・errors 1）|
|`_2_Infrastructure.md`|`infrastructure/`配下のすべて（gormrepo 8・queue 1）|
|`_3_Application・Presentation.md`|`context.go`、`application/`配下のすべて（dto 3・usecase 9）、`presentation/`配下のすべて（4）|

上の一覧は機能全体のファイルである。本書が担当するのは上記のうち該当する行のみで、他の文書が担当するファイルは本書では定義しない。

---

# 3. Domain層設計

`クラス編成機能_Go実装仕様書_1_Domain.md`「3. Domain層設計」に記載する。本書のRepository実装は、同書で定義したRepository Interfaceを実装する。

---

# 4. クラス図

`クラス編成機能_Go実装仕様書_1_Domain.md`「4. クラス図」に記載する。

---

# 5. 状態遷移図

`クラス編成機能_Go実装仕様書_1_Domain.md`「5. 状態遷移図」に記載する。

---

# 6. Application層設計

`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`「6. Application層設計」に記載する。

---

# 7. シーケンス図・処理フロー図

`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`「7. シーケンス図・処理フロー図」に記載する。

---

# 8. Infrastructure層設計

## Repository実装

### SchoolClassRepository実装（`infrastructure/gormrepo/school_class_repository.go`）

- 実装struct名: 非公開struct（例: `schoolClassRepository`）+ コンストラクタ`NewSchoolClassRepository`
- 対応するGORMモデル: `gormmodel.GradeModel`（テーブル`grades`）, `gormmodel.SchoolClassModel`（テーブル`school_classes`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`ListGrades`|`high_school_id = ?`|全件取得（②21章：ページネーション不要）|
|`ListClassesByHighSchool`|`grades`と`school_classes`を`grade_id`で結合し、`high_school_id = ?`|学年IDをキーとしたmapへ変換して返す|
|`FindByID`|`id = ?`（`school_classes`）。同校確認は`grades`との結合、または取得後の`grade.HighSchoolID`比較で行う|-|
|`FindGradeByID`|`id = ? AND high_school_id = ?`（`grades`）|-|
|`Create`|`school_classes`へ1件INSERT|-|
|`Update`|`id = ?`条件で`name`を更新|-|
|`Delete`|`teacher_school_classes`を`school_class_id = ?`条件でDELETEしたうえで、`id = ?`条件でDELETE（②20章：物理削除／論理削除いずれかは推測、17章参照）|同一トランザクション内で実行する|

- Entity ⇔ GORMモデルの変換方針: 各モデルから対応Entityへの非公開変換関数（`toEntity`）、逆方向（`fromEntity`）をrepository実装内に用意する

### SchoolClassRequestRepository実装（`infrastructure/gormrepo/school_class_request_repository.go`）

- 実装struct名: 非公開struct（例: `schoolClassRequestRepository`）+ コンストラクタ`NewSchoolClassRequestRepository`
- 対応するGORMモデル: `gormmodel.SchoolClassRequestModel`（テーブル`school_class_requests`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`Create`|`school_class_requests`へ1件INSERT|-|
|`FindByID`|`id = ?`かつ、`grades`との結合または事前取得した`high_school_id`との比較による同校確認|Gorm規約「5. アソシエーション」の方針どおり明示的な結合／個別クエリを用いる|
|`Update`|`id = ?`条件で`status`・`approver_id`・`approved_at`・`cancelled_at`・`reason`を更新。`lock_version`列との一致条件をGORMの`optimisticlock.Version`機構で自動付与する（Gorm規約「12. 楽観ロック（Optimistic Locking）」）。`RowsAffected == 0`の場合は`domainerror.ErrOptimisticLockConflict`を返す|-|
|`ExistsPendingBySchoolClassID`|`school_class_id = ? AND status = 'pending'`の件数確認|-|
|`Search`|`grades`を結合して`grades.high_school_id = ?`で絞り込み、`Status`・`ApplicantID`の指定があれば条件に加える。`created_at DESC, id DESC`で並べ、`Page`・`PerPage`に応じたオフセット・件数制限を適用する。あわせて総件数を取得する。対象クラス・申請者・承認者・学年は、N+1を避けるため1回の結合または事前取得でまとめて読み込む|Rails現行の`includes(:school_class, :applicant, :approver, :grade)`に相当する（①「5. API / 処理詳細」）|

- Entity ⇔ GORMモデルの変換方針: `LockVersion`（Domain、プレーンな`int64`）⇔`optimisticlock.Version`（GORM）の相互変換をrepository実装内の変換関数で行う（Gorm規約「12. 楽観ロック（Optimistic Locking）」の「Domain EntityとGORMモデルの分離」方針）

### ClassOccupancyRepository実装（`infrastructure/gormrepo/class_occupancy_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewClassOccupancyRepository`
- 対応するGORMモデル: `user` Context所有の`users`テーブル（`school_class_id`列を参照専用で読み取る最小フィールド定義）、`teacher_school_classes`テーブル（`school_class_id`列を参照専用で読み取る最小フィールド定義。所有するContextは現行の②群に定義されていない。17章参照）。`user`②の③が未作成のため、`users`は`user`の`CountUsers`を呼び出さず、対象テーブルへの直接クエリで暫定対応する。`user`の③が整備された時点で、`CountUsers`（クラス指定・無効化済みを含む）の呼び出しへの置き換えを検討する
- クエリ内容:
  - `HasEnrolledStudents`: `users`の`school_class_id = ?`の件数確認（`deleted_at`による絞り込みは行わず、無効化済みも含める）
  - `HasAssignedTeachers`: `teacher_school_classes`の`school_class_id = ?`の件数確認（`role`による絞り込みは行わない）
  - `ListEnrolledStudents`: `users`の`school_class_id = ?`で`id`・`name`・`name_kana`を取得し、`name_kana`昇順で並べる（①「5. API / 処理詳細」。Rails現行の`object.users.sort_by(&:name_kana)`に相当する。`deleted_at`による絞り込みは行わない）
  - `ListAssignedTeachers`: `teacher_school_classes`の`school_class_id = ?`で`user_id`・`role`を取得し、`users`と結合して教員の`name`を得る（①「5. API / 処理詳細」。並び順はRails現行が指定しないため`teacher_school_classes.id`昇順で固定する（推測。順序が不定にならないようにするための実装判断））

### TeacherPermissionRepository実装（`infrastructure/gormrepo/teacher_permission_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewTeacherPermissionRepository`
- 対応するGORMモデル: `teacher_permissions`テーブル（参照専用の最小フィールド定義。`teacher-permission` Contextのモデルとは別に本Context用に定義する）
- クエリ内容: `user_id = ?`で1件取得し、`manage_other_teachers`列を返す。レコードが存在しない場合は`false`を返す（権限なしとして扱う。①「8. 権限制御」）

## 外部連携実装

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`SchoolClassRequestEventPublisher`実装（`infrastructure/queue/school_class_request_event_publisher.go`）|`CreateSchoolClassRequestUseCase.PublishSubmitted`, `ProcessSchoolClassRequestUseCase.PublishResolved`|アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の「ベストエフォートで良い処理」として実装する（②18章「採用理由」）。`jobs`テーブルを経由せず、UseCaseのコミット確定後に`go func() { ctx := context.Background(); ... }()`でgoroutineを起動し、`announcement` Contextが公開する通知作成手段（お知らせ機能と共通の仕組み。正確なpackage pathは②に明記がなく「推測」）を呼び出す|

Mail・Cacheは対象外。

---

---

# 9. Presentation層設計

`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`「9. Presentation層設計」に記載する。

---

# 10. API仕様

`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`「10. API仕様」に記載する。

---

# 11. Transaction実装方針

`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`「11. Transaction実装方針」に記載する。本書のRepository実装は、`gormtx.DBFromContext(ctx, r.db)`で接続を取得する。

---

# 12. Validation実装方針

`クラス編成機能_Go実装仕様書_1_Domain.md`「12. Validation実装方針」（業務ルール検証）と、`_3_Application・Presentation.md`「12. Validation実装方針」（Presentation）に記載する。

---

# 13. Authorization実装方針

`クラス編成機能_Go実装仕様書_3_Application・Presentation.md`「13. Authorization実装方針」に記載する。

---

# 14. Error実装方針

機能全体で共通のため、`クラス編成機能_Go実装仕様書_1_Domain.md`「14. Error実装方針」に記載する。本書のRepository実装が返すInfrastructure Errorの扱いも同節に従う。

---

# 15. GORM / DBクエリ設計

②「20. DB設計方針」により、既存Rails DBをそのまま継続利用し、スキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

### GradeModel（`infrastructure/gormrepo/grade_model.go`）

- 対応テーブル: `grades`（参照専用の最小フィールド定義: `ID` / `HighSchoolID` / `Year`）

### SchoolClassModel（`infrastructure/gormrepo/school_class_model.go`）

- 対応テーブル: `school_classes`

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`GradeID`|`grade_id`||
|`Name`|`name`||
|`CreatedAt`|`created_at`|Gorm規約のタイムスタンプ自動トラッキング|
|`UpdatedAt`|`updated_at`||

### TeacherSchoolClassModel（`infrastructure/gormrepo/teacher_school_class_model.go`）

- 対応テーブル: `teacher_school_classes`（`ID` / `UserID` / `SchoolClassID` / `Role`の最小フィールド定義。`ClassOccupancyRepository`による参照と、`SchoolClassRepository.Delete`による対象クラスの行の削除にのみ用いる。このテーブルを所有・管理するContextは現行の②群に定義されておらず、本Contextからは、書き込む機能が現行の対象範囲に存在しない参照専用の依存として扱う。削除はクラスの削除に付随する連動削除のみである）

### SchoolClassRequestModel（`infrastructure/gormrepo/school_class_request_model.go`）

- 対応テーブル: `school_class_requests`

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`SchoolClassID`|`school_class_id`|NULL許容（新設時）|
|`GradeID`|`grade_id`||
|`ApplicantID`|`applicant_id`||
|`ApproverID`|`approver_id`|NULL許容|
|`Action`|`action`|文字列として保存し、Repository実装内で`valueobject.SchoolClassRequestAction`と相互変換する|
|`Status`|`status`|同上（`valueobject.SchoolClassRequestStatus`）|
|`Name`|`name`|NULL許容|
|`Reason`|`reason`|NULL許容|
|`ApprovedAt`|`approved_at`|NULL許容|
|`CancelledAt`|`cancelled_at`|NULL許容|
|`LockVersion`|`lock_version`|`optimisticlock.Version`型（Gorm規約「12. 楽観ロック（Optimistic Locking）」）|
|`CreatedAt`|`created_at`||
|`UpdatedAt`|`updated_at`||

## 主要クエリの条件・ソート・ページネーション方針

|Repository|メソッド|対象テーブル|条件|結合|
|-|-|-|-|-|
|`SchoolClassRepository`|`ListGrades`|`grades`|`high_school_id = ?`|不要|
|`SchoolClassRepository`|`ListClassesByHighSchool`|`grades`, `school_classes`|`high_school_id = ?`|`grade_id`で結合|
|`SchoolClassRepository`|`FindByID`|`school_classes`|`id = ?`|同校確認のため`grades`参照|
|`SchoolClassRepository`|`Delete`|`teacher_school_classes`, `school_classes`|`teacher_school_classes.school_class_id = ?`の削除、続いて`school_classes.id = ?`の削除（同一トランザクション）|不要|
|`ClassOccupancyRepository`|`HasEnrolledStudents`|`users`|`school_class_id = ?`（`deleted_at`による絞り込みなし）|不要|
|`ClassOccupancyRepository`|`HasAssignedTeachers`|`teacher_school_classes`|`school_class_id = ?`（`role`による絞り込みなし）|不要|
|`SchoolClassRequestRepository`|`FindByID`|`school_class_requests`|`id = ?`|同校確認のため`grades`参照|
|`SchoolClassRequestRepository`|`ExistsPendingBySchoolClassID`|`school_class_requests`|`school_class_id = ? AND status = 'pending'`|不要|

ページネーション・ソートはいずれも不要（②21章：学年・クラスとも全件取得が前提、申請一覧取得APIはRails現行仕様に存在しない）。SQL文そのものは記載しない。

## 既存Schemaへの変更

②20章「変更なし」のとおり、本機能によるスキーマ変更は行わない。

---

---

# 16. テストケース設計
本書が担当するレイヤーのテストのみを記載する。他の区分は各担当文書に記載する。

## Domain Test

対象外（本書はInfrastructure層を担当する）。Domain Test は担当する文書に記載する。

## UseCase Test

対象外（本書はInfrastructure層を担当する）。UseCase Test は担当する文書に記載する。

## Repository Test

|対象|テストケース|
|-|-|
|`SchoolClassRepository`|学年一覧・学年別クラス一覧・クラス詳細が同校スコープで正しく取得されること／承認反映（作成・更新・削除）が正しく実行されること／クラスの削除時に、対象クラスの`teacher_school_classes`の行も同じトランザクションで削除され、他のクラスの行が残ること|
|`ClassOccupancyRepository`|`HasEnrolledStudents`は、無効化済み（`deleted_at`あり）のユーザーのみが在籍している場合も真を返すこと／`HasAssignedTeachers`は、`teacher_school_classes`に担任・副担任のいずれの行がある場合も真を返し、行がない場合に偽を返すこと|
|`SchoolClassRequestRepository`|申請の作成・同校スコープでの取得が正しく動作すること／楽観ロック競合時に`ErrOptimisticLockConflict`を返すこと／重複申請の存在確認が正しく動作すること／`Search`が高校・ステータス・申請者で絞り込み、`created_at`降順・`id`降順で並び、ページネーションと総件数が正しいこと／対象クラス・申請者・承認者が未設定の場合に`nil`で返ること|
|`ClassOccupancyRepository`（表示用）|`ListEnrolledStudents`が氏名カナ昇順で返ること／`ListAssignedTeachers`が`role`を含めて返ること|

## Handler Test

対象外（本書はInfrastructure層を担当する）。Handler Test は担当する文書に記載する。

## Integration Test

対象外（本書はInfrastructure層を担当する）。Integration Test は担当する文書に記載する。

---

# 17. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に列挙する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ディレクトリ名・package名を`schoolclass`とした|②のContext名は`school-class`のみで、ディレクトリ名の明記がない。アーキテクチャ規約「4. Bounded Context構成」の「Contextディレクトリの命名」（Context名から`-`を取り除いた連結形）に従って変換した|規約に基づく（推測ではない）|
|`ClassOccupancyRepository`・`TeacherPermissionRepository`を新設した。`ClassOccupancyRepository`の在籍生徒の確認は`user` Contextが所有する`users`（③が未作成のため暫定で直接クエリ）、所属教員の確認は`teacher_school_classes`（所有するContextが現行の②群に未定義の参照専用の依存）を対象とし、`TeacherPermissionRepository`は`teacher-permission` Contextが公開する参照手段として扱った|②「11. Repository設計」「16. Authorization設計」が依存関係を明記しているが、具体的なInterface名・配置場所までは規定していない。`teacher_school_classes`を所有するContextは、現行の②群のいずれにも定義されていない|推測|
|クラスの削除の承認時に、対象クラスの`teacher_school_classes`の行を、クラスの削除より前に同一トランザクションで削除する|Rails現行の`SchoolClass`が`has_many :teacher_school_classes, dependent: :destroy`でクラスの削除時に行を連動削除しており、DBの外部キー制約（`teacher_school_classes.school_class_id`→`school_classes`）もあるため。②「3. Bounded Context」に記載した|推測ではない（Rails現行の挙動に基づく）|
|`SchoolClassRequestEventPublisher`の`announcement` Context呼び出し先の正確なpackage path|②「3. Bounded Context」「18. Domain Event」は`announcement` Contextへの依存を明記するが、当該Contextの②/③文書（お知らせ機能）との具体的な接続方法は本タスクの対象外文書に依存する|推測|
|`SchoolClass.Delete`（Repository）を物理削除とするか論理削除とするかを確定していない|②20章はスキーマ変更なしとするのみで、削除方式（`deleted_at`利用の有無）を明記していない。Gorm規約「0. 本プロジェクトでの採用方針」は原則論理削除だが、クラス自体に削除申請というワークフローが既にあるため、物理削除の可能性も残る|推測（実装時に①または業務要件を確認する必要がある）|
|`CancelSchoolClassRequest`で申請者本人でない場合のHTTP Statusを403とした|②「17. Error設計」のエラー仕様表に本ケースの明示的なStatus Code記載がない（承認・却下の自己承認禁止と同様に扱った）|推測|
|`teacher_permissions`テーブルを本Context用に独立してモデル定義する構成とした（教師教員管理機能側と物理テーブルは共有するが、GORMモデル定義は別途持つ）|アーキテクチャ規約「5. Context間連携ルール」の「相手Contextの内部実装に直接依存しない」方針、および認証機能等の先例に倣った|推測|
|`school_class_requests`の`lock_version`列がGorm規約の`optimisticlock.Version`型でそのまま扱える前提とした|②「4. 設計パターン」「17. Validation設計」は楽観ロックの採用のみを明記し、具体的なカラム型までは規定していない。既存Rails実装（`lock_version`という命名自体がRailsの`optimistic_locking`慣習に由来）を踏まえた判断|推測|

上記以外の設計判断（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
