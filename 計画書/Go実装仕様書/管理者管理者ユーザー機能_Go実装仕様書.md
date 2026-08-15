# 管理者管理者ユーザー機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者（`admin`ロールのユーザー）自身が、管理者アカウントの検索・ページング付き一覧参照、詳細参照、新規作成、既存管理者のプロフィール・個人情報更新、論理削除を行う機能である。削除操作には「自分自身は削除できない」「削除後も有効な管理者が1人以上残ること（最後の管理者は削除できない）」という2つの安全弁ルールが適用される。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により、本機能は **Domain Model** を採用する。

- 管理者アカウントが「有効」「削除済み（論理削除）」という状態を持ち、状態遷移（有効→削除済み）に「自分自身でないこと」「削除後も有効な管理者が1人以上残ること」という複数の前提条件が伴うこと
- 「最後の管理者かどうか」の判定が、対象アカウント単体の属性だけでなく、システム全体の有効管理者数という集約横断の情報に依存すること
- 電話番号の桁数、性別の許容値、生年月日の未来日付禁止、名前未入力時のデフォルト補完という複数の業務ルールが作成・更新時に絡むこと
- 削除可否判定ロジックを独立したDomain Serviceとして切り出すことで、将来の権限段階追加等の変更に対する耐性とテスト容易性を確保する必要があること

上記の理由からTransaction Script・Active Record・Event Sourcingは採用せず、Domain Modelを採用している（詳細は②「4. 設計パターン」参照）。

## 本書が対象とする実装範囲

本書は、②で確定した設計（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略）を変更せず、Goでの具体的なコード構成（package構成・struct定義・interfaceメソッドシグネチャ・クエリ内容）に落とし込むことを目的とする。

規約`アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」の「Domain Model」節に従い、`{context}/domain`・`application`・`infrastructure`・`presentation`のフルレイヤー構成を適用する。

①Rails実装詳細は本タスクでは提供されていないため、①の実装コードそのものを根拠とする記載は行わない（①未提供のため参照不可）。②に明記された「Rails現行仕様の要約」の範囲でのみ言及する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名（②）: `admin-account-management`
- ディレクトリ名: `internal/admin_account`

  > **②からの補足**: ②にはディレクトリ名の明記がない。アーキテクチャ規約「5. Bounded Context構成 命名規則」（Context名はkebab-case、`internal/`配下は英単語1語または短いスネークケース）に従い、`admin-account-management`を短縮した（推測。`認証機能`が`authentication`→`internal/auth`と短縮した先例に倣う）。

## ②で採用した設計パターン

Domain Model

## 作成するディレクトリ一覧

```
internal/admin_account/
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

- `domain/specification/`: 対象外（②に該当する複雑な仕様判定パターンの記載がなく、削除可否判定はDomain Service（`AdminDeletionPolicy`）として②「8. Domain Service」に明記されているため、specificationパターンは使用しない）
- `domain/event/`: 対象外（②「15. Domain Event」により、本機能では現時点でDomain Eventを採用しないと明記されているため）
- `infrastructure/mail/`・`infrastructure/cache/`・`infrastructure/queue/`: 対象外（②に該当する外部連携要件の記載がないため。②「15. Domain Event」が将来的な連携ポイント（削除時のJWT無効化）として記録している内容は「推測」かつ「現行仕様には明記されていない」とされており、本書でも実装対象としない）

## 作成するファイル一覧

```
internal/admin_account/domain/entity/admin_account.go
internal/admin_account/domain/entity/admin_personal_info.go

internal/admin_account/domain/valueobject/phone_number.go
internal/admin_account/domain/valueobject/birthday.go
internal/admin_account/domain/valueobject/gender.go
internal/admin_account/domain/valueobject/admin_display_name.go

internal/admin_account/domain/repository/admin_account_repository.go
internal/admin_account/domain/repository/admin_personal_info_repository.go
internal/admin_account/domain/repository/user_role_repository.go

internal/admin_account/domain/service/admin_deletion_policy.go

internal/admin_account/domain/errors/errors.go

internal/admin_account/application/dto/list_admins_dto.go
internal/admin_account/application/dto/show_admin_dto.go
internal/admin_account/application/dto/create_admin_dto.go
internal/admin_account/application/dto/update_admin_dto.go
internal/admin_account/application/dto/delete_admin_dto.go

internal/admin_account/application/usecase/transaction_manager.go
internal/admin_account/application/usecase/list_admins_usecase.go
internal/admin_account/application/usecase/show_admin_usecase.go
internal/admin_account/application/usecase/create_admin_usecase.go
internal/admin_account/application/usecase/update_admin_usecase.go
internal/admin_account/application/usecase/delete_admin_usecase.go
internal/admin_account/application/usecase/errors.go

internal/admin_account/infrastructure/persistence/gorm/user_model.go
internal/admin_account/infrastructure/persistence/gorm/user_personal_info_model.go

internal/admin_account/infrastructure/repository/admin_account_repository.go
internal/admin_account/infrastructure/repository/admin_personal_info_repository.go
internal/admin_account/infrastructure/repository/user_role_repository.go

internal/admin_account/presentation/handler/admin_handler.go

internal/admin_account/presentation/request/admin_request.go

internal/admin_account/presentation/response/admin_response.go

internal/admin_account/presentation/routes.go
```

> **②からの補足**: `domain/repository/user_role_repository.go`は、②「3. Bounded Context」の「Account/Authentication Context: 管理者アカウントの実体（ログイン用の`User`レコード）の作成・識別に依存する」を実装レベルで具体化したものである。管理者アカウント作成時に`user_role_id`（"admin"ロールのID）を解決するために必要となるが、②「9. Repository設計」には明記がないため本書での判断である（推測）。アーキテクチャ規約「6. Context間連携ルール」に従い、authenticationコンテキストの内部実装には依存せず、本Context自身が定義する参照専用Interfaceとして扱う。

---

# 3. Domain層設計

## Entity

### AdminAccount（`domain/entity/admin_account.go`）

- struct名: `AdminAccount`
- フィールド:

|フィールド|型|意味|
|-|-|-|
|`ID`|`uint`|管理者アカウント（`users`テーブル行）の識別子|
|`Email`|`string`|ログインに用いるメールアドレス（②「7. Value Object設計」の方針により形式検証はPresentation層が担うため、Value Object化しない）|
|`Name`|`valueobject.AdminDisplayName`|表示名（名前未入力時はメールアドレスのローカル部がデフォルトとして解決済みの状態で保持される）|
|`NameKana`|`string`|フリガナ|
|`AddressID`|`*uint`|住所（Address Context）への外部参照。②「5. Aggregate設計」によりAggregate外部参照として識別子のみを保持する|
|`UserRoleID`|`uint`|ロール識別子（本Contextが扱う対象は常に"admin"ロールのIDのみ）|
|`DeletedAt`|`*time.Time`|論理削除日時。`nil`の場合は有効、値が設定されている場合は削除済み|

- 公開メソッド一覧（引数・戻り値のみ。ロジックは記述しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewAdminAccount`|`(id uint, email string, name valueobject.AdminDisplayName, nameKana string, addressID *uint, userRoleID uint, deletedAt *time.Time) (*AdminAccount, error)`|`(*AdminAccount, error)`|不変条件を満たしたAdminAccountを生成するファクトリ（永続化層からの復元にも使用する）|
|`UpdateProfile`|`(name valueobject.AdminDisplayName, nameKana string, addressID *uint)`|`error`|基本情報（表示名・フリガナ・住所参照）を更新する|
|`MarkDeleted`|`(deletedAt time.Time)`|`error`|論理削除状態へ遷移させる。既に削除済みの場合は`ErrAdminAlreadyDeleted`を返す|
|`IsDeleted`|`()`|`bool`|削除済みかどうかを判定する|

- 不変条件（ファクトリで保証する内容）:
  - `Email`は空文字を許容しない
  - `UserRoleID`は0を許容しない

> **②からの補足**: ②「6. Entity設計」は削除実行時にAdminAccountが「事前に判定された削除可否結果を反映して状態を遷移させる」とし、判定ロジック自体は保持しないとしている。本書では、削除可否判定（自分自身／最後の管理者）は`AdminDeletionPolicy`が担い、`MarkDeleted`は「既に削除済みでないか」という状態遷移そのものの整合性（②「12. Validation設計」の「状態チェック」）のみを保証する構成とした。②の記載同士（Domain Serviceへの判定集約とEntityの状態チェック責務）を矛盾なく実装するための判断であり、新しい業務ルールの追加ではない。

### AdminPersonalInfo（`domain/entity/admin_personal_info.go`）

- struct名: `AdminPersonalInfo`
- フィールド:

|フィールド|型|意味|
|-|-|-|
|`UserID`|`uint`|紐づく管理者アカウントのID|
|`PhoneNumber`|`*valueobject.PhoneNumber`|電話番号。未入力の場合は`nil`|
|`Birthday`|`*valueobject.Birthday`|生年月日。未入力の場合は`nil`|
|`Gender`|`*valueobject.Gender`|性別。未入力の場合は`nil`|

- 公開メソッド一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewAdminPersonalInfo`|`(userID uint, phone *valueobject.PhoneNumber, birthday *valueobject.Birthday, gender *valueobject.Gender) (*AdminPersonalInfo, error)`|`(*AdminPersonalInfo, error)`|個人情報を生成するファクトリ（各項目は未入力＝`nil`を許容する）|
|`UpdatePersonalInfo`|`(phone *valueobject.PhoneNumber, birthday *valueobject.Birthday, gender *valueobject.Gender)`|`error`|電話番号・生年月日・性別を更新する|

- 不変条件: `UserID`は0を許容しない。各Value Objectフィールドは、値が設定される場合は必ずValue Objectのコンストラクタを経由した検証済みの値であることが保証される（`nil`は許容する）。

> **②からの補足**: ②「6. Entity設計」はAdminPersonalInfoのライフサイクルを「作成（管理者作成に付随）・更新」としているが、②「10. UseCase設計」のCreateAdminUseCase入力は`current admin, name, email`のみで電話番号・生年月日・性別を含まない。本書では、管理者作成時にAdminPersonalInfoを各項目未設定（`nil`）の状態で生成し、後続のUpdateAdminUseCaseで値を設定する構成とした。②の記載同士（Aggregateとしての付随生成と、Create入力の限定）を整合させるための実装判断である（新しい業務ルールの追加ではない）。

## Value Object

### PhoneNumber（`domain/valueobject/phone_number.go`）

- struct名: `PhoneNumber`
- フィールド: `value string`（非公開）
- 生成時に検証するルール: 数字のみで構成され、10桁または11桁であること（②「7. Value Object設計」）
- 公開メソッド: `NewPhoneNumber(raw string) (PhoneNumber, error)` / `(p PhoneNumber) String() string`

### Birthday（`domain/valueobject/birthday.go`）

- struct名: `Birthday`
- フィールド: `value time.Time`（非公開）
- 生成時に検証するルール: 基準時刻（`now`）より未来の日付を許容しない（②「7. Value Object設計」）
- 公開メソッド: `NewBirthday(raw time.Time, now time.Time) (Birthday, error)` / `(b Birthday) Time() time.Time`

  > **②からの補足**: `now`を引数として明示的に受け取る構成は②に明記がない。基準時刻をコンストラクタ内部で`time.Now()`として固定するとテスト容易性が下がるため、認証機能の`ResetTokenValidityPeriod.IsExpired(issuedAt, now)`と同様に呼び出し側から基準時刻を注入する形とした（推測）。

### Gender（`domain/valueobject/gender.go`）

- struct名: `Gender`
- フィールド: `value string`（非公開）
- 生成時に検証するルール: 定義済みの許容値（Rails `UserPersonalInfo.genders`相当）のいずれかであること（②「7. Value Object設計」）
- 公開メソッド: `NewGender(raw string) (Gender, error)` / `(g Gender) String() string`

  > **②からの補足**: 許容される具体的な列挙値（例: `male`/`female`/`other`等）は②に記載がなく、①（Rails `UserPersonalInfo.genders`の定義）も本タスクでは未提供のため参照不可。本書では許容値のリストを確定させず、コンストラクタが許容値集合に対して検証を行うという構造のみを定義する。実装着手前にRails現行の`genders` enum定義を別途確認する必要がある（①未提供のため参照不可）。

### AdminDisplayName（`domain/valueobject/admin_display_name.go`）

- struct名: `AdminDisplayName`
- フィールド: `value string`（非公開）
- 生成時に検証するルール: 入力が空文字または未指定の場合、メールアドレスの`@`より前の部分を採用する（②「7. Value Object設計」）
- 公開メソッド: `NewAdminDisplayName(rawName string, email string) (AdminDisplayName, error)` / `(n AdminDisplayName) String() string`

## Value Objectを採用しないもの

②「7. Value Object設計」の方針どおり、メールアドレスは本機能に固有の複雑な業務ルールを持たないため、`AdminAccount.Email string`にそのまま保持し、Value Object化しない。形式検証はPresentation層で行う。

## Repository Interface

### AdminAccountRepository（`domain/repository/admin_account_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`List`|`(ctx context.Context, params ListAdminAccountsParams)`|`([]*entity.AdminAccount, int64, error)`|検索キーワード・ページングを適用した一覧取得と総件数取得（②「9. Repository設計」）|
|`FindByID`|`(ctx context.Context, id uint)`|`(*entity.AdminAccount, error)`|特定管理者の取得。存在しない場合は`(nil, nil)`を返し、種別判定はUseCase側に委ねる|
|`Create`|`(ctx context.Context, params CreateAdminAccountParams)`|`(*entity.AdminAccount, error)`|管理者アカウントの新規作成|
|`Update`|`(ctx context.Context, account *entity.AdminAccount)`|`error`|基本情報の更新|
|`Delete`|`(ctx context.Context, id uint, deletedAt time.Time)`|`error`|論理削除の実行（`deleted_at`の更新）|
|`CountActive`|`(ctx context.Context)`|`(int64, error)`|有効な管理者の件数取得（削除可否判定に使用）|

`ListAdminAccountsParams`のフィールド: `Keyword string`, `Page int`, `PerPage int`
`CreateAdminAccountParams`のフィールド: `Email string`, `Name valueobject.AdminDisplayName`, `NameKana string`, `AddressID *uint`

- 保持しない責務: 削除可否の判定、名前のデフォルト補完ルール（②「9. Repository設計」の方針どおり）

### AdminPersonalInfoRepository（`domain/repository/admin_personal_info_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByUserID`|`(ctx context.Context, userID uint)`|`(*entity.AdminPersonalInfo, error)`|`user_id`による個人情報の取得。存在しない場合は`(nil, nil)`|
|`Create`|`(ctx context.Context, info *entity.AdminPersonalInfo)`|`error`|個人情報の作成（管理者作成に付随）|
|`Update`|`(ctx context.Context, info *entity.AdminPersonalInfo)`|`error`|個人情報の更新|

- 保持しない責務: 電話番号・生年月日・性別のバリデーション（Value Object側で表現済みの検証の再確認に留める。②「9. Repository設計」）

### UserRoleRepository（`domain/repository/user_role_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindIDByName`|`(ctx context.Context, name string)`|`(uint, error)`|ロール名（"admin"）からロールIDを解決する（参照専用）|

> **②からの補足**: 本Interfaceは②「9. Repository設計」に明記がなく、②「3. Bounded Context」が定めるAccount/Authentication Contextへの依存を、管理者アカウント作成・一覧絞り込み時の`user_role_id`解決のために具体化したものである（推測）。実装（Infrastructure層）は`user_roles`テーブルを参照専用で読み取る。

## Domain Service

### AdminDeletionPolicy（`domain/service/admin_deletion_policy.go`）

- struct名: `AdminDeletionPolicy`（状態を持たない。コンストラクタ`NewAdminDeletionPolicy() *AdminDeletionPolicy`は依存を受け取らない）
- 公開メソッド: `(p *AdminDeletionPolicy) Authorize(currentAdminID uint, targetAdminID uint, activeAdminCount int64) error`
- 責務: 削除対象が操作者自身でないこと、削除後も有効な管理者が1人以上残ることの2条件を判定し、いずれかに違反する場合は対応するDomain Errorを返す（②「8. Domain Service」）

## Domain Event

対象外。②「15. Domain Event」により、本機能では現時点でDomain Eventを採用しない。②は「管理者アカウントが削除された場合、認証コンテキスト側で該当ユーザーのセッション（JWT）を無効化する必要がある可能性がある」ことを将来の連携ポイントとして記録しているが、「現行仕様には明記されていない」「推測」と明記されているため、本書では`AdminAccountDeleted`等のイベントstruct・発火処理を実装対象に含めない。

## Domain Error

`domain/errors/errors.go`に、`errors.New`によるセンチネルエラー変数として定義する（②「14. Error設計」のDomain Errorに対応。認証機能の実装仕様と同一の方式を踏襲する）。

|変数名|発生条件|
|-|-|
|`ErrCannotDeleteSelf`|削除対象が操作者自身である|
|`ErrLastAdminCannotBeDeleted`|削除後に有効な管理者が0人になる（最後の管理者の削除試行）|
|`ErrAdminAlreadyDeleted`|既に削除済みの管理者に対して`MarkDeleted`が呼ばれた|
|`ErrInvalidPhoneNumber`|電話番号が数字のみ10〜11桁の形式を満たさない|
|`ErrInvalidBirthday`|生年月日が未来日付である|
|`ErrInvalidGender`|性別が許容値のいずれにも一致しない|

---

# 4. Application層設計

## DTO（Command / Query）

|struct名|フィールドと型|区分|
|-|-|-|
|`ListAdminsQuery`|`CurrentAdminID uint`, `Keyword string`, `Page int`, `PerPage int`|Query|
|`AdminSummary`|`ID uint`, `Name string`, `Email string`|出力（一覧項目）|
|`ListAdminsResult`|`Admins []AdminSummary`, `Page int`, `PerPage int`, `TotalCount int64`|出力|
|`ShowAdminQuery`|`CurrentAdminID uint`, `AdminID uint`|Query|
|`ShowAdminResult`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|出力|
|`CreateAdminCommand`|`CurrentAdminID uint`, `Name string`, `Email string`|Command|
|`CreateAdminResult`|`ID uint`, `Name string`, `Email string`|出力|
|`UpdateAdminCommand`|`CurrentAdminID uint`, `AdminID uint`, `Name string`, `NameKana string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|Command|
|`UpdateAdminResult`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|出力|
|`DeleteAdminCommand`|`CurrentAdminID uint`, `AdminID uint`|Command|
|`DeleteAdminResult`|（フィールドなし。成功したことのみを表す）|出力|

> **②からの補足**: `UpdateAdminCommand`が`Email`を含まない構成とした。②「10. UseCase設計」のCreateAdminUseCase入力にのみ`email`が明記され、UpdateAdminUseCase入力は「current admin, admin id, update request」と抽象的にしか記載がない。②「16. API互換方針」のRequestパラメータ一覧には`email`が含まれるが、一覧・詳細・作成・更新のいずれで使われるかは区別されていない。本書ではメールアドレスの変更自体が②のどのUseCaseにも明示的な業務ルールとして記載されていないため、更新対象から除外した（推測。実装時に更新可否を②側で再確認する必要がある）。

## UseCase

### ListAdminsUseCase（`application/usecase/list_admins_usecase.go`）

- struct名: `ListAdminsUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`（Interface）
- 公開メソッド: `(u *ListAdminsUseCase) Execute(ctx context.Context, query dto.ListAdminsQuery) (dto.ListAdminsResult, error)`
- 処理ステップ:
  1. `PerPage`の上限制御を行う（②「9. Repository設計」「10. UseCase設計」により100件を上限とする）
  2. `AdminAccountRepository.List`で検索・ページングを適用した一覧と総件数を取得する
  3. `ListAdminsResult`を組み立てて返す
- トランザクション境界: なし（②「11. Transaction設計」により読み取りのみ）
- 発生しうるApplication Error: なし（Infrastructure Errorのみ）

> **②からの補足**: `PerPage`の具体的なデフォルト値（未指定時に何件とするか）は②に明記がない。上限100件のみが明記されているため、本書では上限制御のみを設計対象とし、デフォルト値は実装時に確認する必要がある旨を14章で補足する（推測）。

### ShowAdminUseCase（`application/usecase/show_admin_usecase.go`）

- struct名: `ShowAdminUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`, `AdminPersonalInfoRepository`
- 公開メソッド: `(u *ShowAdminUseCase) Execute(ctx context.Context, query dto.ShowAdminQuery) (dto.ShowAdminResult, error)`
- 処理ステップ:
  1. `AdminAccountRepository.FindByID`で対象管理者を取得する。取得できない場合（存在しない、または論理削除済みでGORMの標準挙動により除外された場合）は`ErrAdminNotFound`を返す
  2. `AdminPersonalInfoRepository.FindByUserID`で個人情報を取得する（未設定の場合は`nil`のまま扱う）
  3. `ShowAdminResult`を組み立てて返す
- トランザクション境界: なし（読み取りのみ）
- 発生しうるApplication Error: `ErrAdminNotFound`

### CreateAdminUseCase（`application/usecase/create_admin_usecase.go`）

- struct名: `CreateAdminUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`, `AdminPersonalInfoRepository`, `UserRoleRepository`, `TransactionManager`
- 公開メソッド: `(u *CreateAdminUseCase) Execute(ctx context.Context, cmd dto.CreateAdminCommand) (dto.CreateAdminResult, error)`
- 処理ステップ:
  1. `valueobject.NewAdminDisplayName(cmd.Name, cmd.Email)`で表示名を解決する（名前未入力時のデフォルト補完。②「7. Value Object設計」）
  2. `UserRoleRepository.FindIDByName(ctx, "admin")`で"admin"ロールIDを解決する
  3. `TransactionManager.WithinTransaction`内で以下を実行する:
     a. `AdminAccountRepository.Create`でAdminAccountを作成する
     b. `entity.NewAdminPersonalInfo`で未入力状態（電話番号・生年月日・性別すべて`nil`）のAdminPersonalInfoを生成し、`AdminPersonalInfoRepository.Create`で作成する
  4. `CreateAdminResult`を組み立てて返す
- トランザクション境界: AdminAccountとAdminPersonalInfoの作成を1トランザクションとする
- 発生しうるApplication Error: なし（Infrastructure Errorのみ）

> **②からの補足**: ②「11. Transaction設計」は「CreateAdminUseCaseでは、対象データの保存が完了した時点でコミットする」とのみ記載し、AdminAccount単体の保存を指しているように読める。本書では②「5. Aggregate設計」がAdminAccountとAdminPersonalInfoを1つの整合性単位とし、「プロフィール更新時、`users`と`user_personal_infos`への反映を1トランザクションで行う」としていることと矛盾しないよう、作成時の両テーブルへの反映も1トランザクションに含めた（②の記載同士を整合させるための判断。新しい業務ルールの追加ではない）。

### UpdateAdminUseCase（`application/usecase/update_admin_usecase.go`）

- struct名: `UpdateAdminUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`, `AdminPersonalInfoRepository`, `TransactionManager`
- 公開メソッド: `(u *UpdateAdminUseCase) Execute(ctx context.Context, cmd dto.UpdateAdminCommand) (dto.UpdateAdminResult, error)`
- 処理ステップ:
  1. `AdminAccountRepository.FindByID`で対象管理者を取得する。取得できない場合は`ErrAdminNotFound`を返す
  2. `valueobject.NewAdminDisplayName(cmd.Name, account.Email)`で表示名を解決する
  3. `cmd.PhoneNumber` / `cmd.Birthday` / `cmd.Gender`が指定されている場合のみ、それぞれ`valueobject.NewPhoneNumber` / `valueobject.NewBirthday` / `valueobject.NewGender`で検証する
  4. `account.UpdateProfile`で基本情報を更新する
  5. `AdminPersonalInfoRepository.FindByUserID`で既存の個人情報を取得し、`UpdatePersonalInfo`で更新する
  6. `TransactionManager.WithinTransaction`内で`AdminAccountRepository.Update`・`AdminPersonalInfoRepository.Update`を実行する
  7. `UpdateAdminResult`を組み立てて返す
- トランザクション境界: AdminAccountとAdminPersonalInfoの更新を1トランザクションとする（②「11. Transaction設計」）
- 発生しうるApplication Error: `ErrAdminNotFound`
- 発生しうるDomain Error: `ErrInvalidPhoneNumber`, `ErrInvalidBirthday`, `ErrInvalidGender`

### DeleteAdminUseCase（`application/usecase/delete_admin_usecase.go`）

- struct名: `DeleteAdminUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`, `*service.AdminDeletionPolicy`, `TransactionManager`
- 公開メソッド: `(u *DeleteAdminUseCase) Execute(ctx context.Context, cmd dto.DeleteAdminCommand) (dto.DeleteAdminResult, error)`
- 処理ステップ:
  1. `TransactionManager.WithinTransaction`内で以下を実行する:
     a. `AdminAccountRepository.FindByID`で対象管理者を取得する。取得できない場合は`ErrAdminNotFound`を返す
     b. `AdminAccountRepository.CountActive`で有効な管理者数を取得する
     c. `AdminDeletionPolicy.Authorize(cmd.CurrentAdminID, cmd.AdminID, activeAdminCount)`で削除可否を判定する
     d. `target.MarkDeleted(now)`で状態を遷移させる
     e. `AdminAccountRepository.Delete`で`deleted_at`を反映する
  2. `DeleteAdminResult`を返す
- トランザクション境界: 有効管理者数の取得から削除実行までを1トランザクションとする（②「11. Transaction設計」。「最後の管理者」ルールの競合を防ぐため）
- 発生しうるApplication Error: `ErrAdminNotFound`
- 発生しうるDomain Error: `ErrCannotDeleteSelf`, `ErrLastAdminCannotBeDeleted`, `ErrAdminAlreadyDeleted`

---

# 5. Infrastructure層設計

## Repository実装

### AdminAccountRepository実装（`infrastructure/repository/admin_account_repository.go`）

- 実装struct名: 非公開struct（例: `adminAccountRepository`）+ コンストラクタ`NewAdminAccountRepository`で公開する構成（規約「9. 命名規約」の「`Impl`接尾辞を付けない」方針に従う。認証機能の実装仕様と同一の方式）
- 対応するGORMモデル: `gormmodel.UserModel`（`infrastructure/persistence/gorm/user_model.go`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`List`|`user_role_id = ?`（"admin"ロールID）に加え、`Keyword`が指定されていれば`name LIKE ?`または`email LIKE ?`（②「9. Repository設計」の「キーワード検索（名前・メールアドレス等）」）。`Page`/`PerPage`によるOFFSET/LIMIT。総件数はCOUNTクエリで別途取得|GORM標準の論理削除機構により`deleted_at IS NULL`が自動的に条件へ付与される（後述「②からの補足」参照）|
|`FindByID`|`id = ?`かつ`user_role_id = ?`（"admin"ロールID）で1件検索|同上、削除済みは自動的に除外される|
|`Create`|`users`テーブルへ1件作成。`CreateAdminAccountParams`の`Email`, `Name`（`AdminDisplayName.String()`）, `NameKana`, `AddressID`と、解決済みの`UserRoleID`を反映する|
|`Update`|`id = ?`条件で`name`, `name_kana`, `address_id`を更新|
|`Delete`|`id = ?`条件で`deleted_at`を現在時刻へ更新（論理削除）|
|`CountActive`|`user_role_id = ?`（"admin"ロールID）かつ`deleted_at IS NULL`の件数を取得|

- Entity ⇔ GORMモデルの変換方針: `gormmodel.UserModel`から`entity.AdminAccount`への変換関数（`toEntity`）、逆方向の変換関数（`toModel`）をrepository実装内の非公開関数として用意する。`Name`変換時は`valueobject.NewAdminDisplayName`を通す。DBに不正な形式が入っていた場合はInfrastructure Errorとして扱う。

> **②からの補足**: ②「17. DB設計方針」は`users`の`deleted_at`による論理削除を前提とするが、実装機構（GORMの`gorm.DeletedAt`型フィールドによる標準の論理削除機能を用いるか、明示的な`WHERE deleted_at IS NULL`条件を都度付与するか）は②・Gorm規約のいずれにも明記がない。本書ではGORM標準の論理削除機構（`gorm.DeletedAt`型フィールド）を採用し、`Delete`実行時に`deleted_at`が自動設定され、`List`/`FindByID`等の標準検索メソッドが自動的に削除済みレコードを除外する構成とする（推測。Gorm規約に反しない一般的なGORM機能であり、②が前提とする論理削除仕様を実現する）。この結果、Domain層の`ErrAdminAlreadyDeleted`は主に直接`MarkDeleted`を呼び出す経路の防御的チェックとして機能し、通常のHTTP経由の操作では「対象不存在」（`ErrAdminNotFound`）として先に検出される。

### AdminPersonalInfoRepository実装（`infrastructure/repository/admin_personal_info_repository.go`）

- 実装struct名: 非公開struct（例: `adminPersonalInfoRepository`）+ コンストラクタ`NewAdminPersonalInfoRepository`
- 対応するGORMモデル: `gormmodel.UserPersonalInfoModel`（`infrastructure/persistence/gorm/user_personal_info_model.go`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`FindByUserID`|`user_id = ?`で1件検索|存在しない場合は`(nil, nil)`を返す|
|`Create`|`user_personal_infos`テーブルへ1件作成。`PhoneNumber`/`Birthday`/`Gender`が`nil`の場合はカラムをNULLのまま作成する|
|`Update`|`user_id = ?`条件で`phone_number`, `birthday`, `gender`を更新|

- Entity ⇔ GORMモデルの変換方針: `PhoneNumber`/`Birthday`/`Gender`各Value Objectと、GORMモデルのnull許容カラム（`*string`/`*time.Time`/`*string`）との相互変換関数を用意する。DB上に値が存在する場合のみ対応するValue Objectコンストラクタで検証し直す。不正な形式が保存されていた場合はInfrastructure Errorとして扱う。

### UserRoleRepository実装（`infrastructure/repository/user_role_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewUserRoleRepository`
- 対応するGORMモデル: `gormmodel.UserRoleModel`（`user_roles`テーブル、参照専用の最小フィールド定義。authenticationコンテキストが所有するテーブルを参照専用で読み取る）
- クエリ内容: `name = ?`（"admin"）で1件検索し、`ID`を返す

## 外部連携実装

対象外。②に本機能でのMail・Cache・Queue連携要件の記載はない。②「15. Domain Event」が記録する将来的なJWT無効化連携（推測、現行仕様に非明記）は本書の実装対象に含めない。

---

# 6. Presentation層設計

## Handler

### AdminHandler（`presentation/handler/admin_handler.go`）

- struct名: `AdminHandler`
- 対応する呼び出し先: `*usecase.ListAdminsUseCase`, `*usecase.ShowAdminUseCase`, `*usecase.CreateAdminUseCase`, `*usecase.UpdateAdminUseCase`, `*usecase.DeleteAdminUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`List`|GET|`/api/v1/admin/admins`|
|`Show`|GET|`/api/v1/admin/admins/:id`|
|`Create`|POST|`/api/v1/admin/admins`|
|`Update`|PATCH|`/api/v1/admin/admins/:id`|
|`Delete`|DELETE|`/api/v1/admin/admins/:id`|

- `List`処理順序: クエリパラメータバインド（`request.AdminListRequest`）→ Presentation Validation（型チェック）→ Middlewareが設定した`current_admin`のIDを`dto.ListAdminsQuery.CurrentAdminID`へ設定 → `ListAdminsUseCase.Execute`呼び出し → `response.AdminListResponse`へ変換して返却（200）
- `Show`処理順序: パスパラメータ（`id`）バインド → `ShowAdminUseCase.Execute`呼び出し → 存在しない場合は404、成功時は`response.AdminDetailResponse`へ変換して返却（200）
- `Create`処理順序: リクエストバインド（`request.AdminCreateRequest`）→ Presentation Validation（`email`必須・形式チェック）→ `CreateAdminUseCase.Execute`呼び出し → `response.AdminDetailResponse`（作成結果相当）へ変換して返却（②「16. API互換方針」により200）
- `Update`処理順序: パスパラメータ（`id`）＋リクエストバインド（`request.AdminUpdateRequest`）→ Presentation Validation（電話番号・生年月日の形式チェック）→ `UpdateAdminUseCase.Execute`呼び出し → `response.AdminDetailResponse`へ変換して返却（200）
- `Delete`処理順序: パスパラメータ（`id`）バインド → Middlewareが設定した`current_admin`のIDを`dto.DeleteAdminCommand.CurrentAdminID`へ設定 → `DeleteAdminUseCase.Execute`呼び出し → 成功時は204（本文なし）を返却。Handler自身は削除可否判定を持たない（②「13. Authorization設計」の「Handler: 具体的な削除可否判定は持たせない」方針に従う）

## Request / Response DTO

### Request（`presentation/request/admin_request.go`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`AdminListRequest`|`Q string`, `Page int`, `PerPage int`|クエリパラメータ。`binding:"omitempty"`。型のみチェックし、`PerPage`の上限制御はApplication層（`ListAdminsUseCase`）で行う|
|`AdminCreateRequest`|`Name string`, `Email string`|`Email`: `binding:"required,email"`（②「12. Validation設計」の「必須チェック: 作成時の`email`の必須性」）。`Name`は`binding:"omitempty"`（未入力を許容し、デフォルト補完はDomain層が担う）|
|`AdminUpdateRequest`|`Name string`, `NameKana string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *string`, `Gender *string`|`PhoneNumber`: `binding:"omitempty,numeric"`（数字形式チェック）。`Birthday`: `binding:"omitempty,datetime=2006-01-02"`（日付形式チェック）。桁数・未来日付・許容値そのものの検証はDomain層のValue Objectが行う（②「12. Validation設計」の責務分離方針）|

### Response（`presentation/response/admin_response.go`）

|struct名|フィールドと型|
|-|-|
|`AdminSummaryResponse`|`ID uint`, `Name string`, `Email string`|
|`AdminListResponse`|`Admins []AdminSummaryResponse`, `Page int`, `PerPage int`, `TotalCount int64`|
|`AdminDetailResponse`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *string`, `Gender *string`|

## Routing（`presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/admin/admins`|`AdminHandler.List`|
|GET|`/api/v1/admin/admins/:id`|`AdminHandler.Show`|
|POST|`/api/v1/admin/admins`|`AdminHandler.Create`|
|PATCH|`/api/v1/admin/admins/:id`|`AdminHandler.Update`|
|DELETE|`/api/v1/admin/admins/:id`|`AdminHandler.Delete`|

全ルートに、adminロールを要求する認証Middlewareを適用する（②「13. Authorization設計」）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/admin/admins|AdminHandler.List|AdminListRequest|AdminListResponse|200|
|GET|/api/v1/admin/admins/:id|AdminHandler.Show|-（パスパラメータのみ）|AdminDetailResponse|200|
|POST|/api/v1/admin/admins|AdminHandler.Create|AdminCreateRequest|AdminDetailResponse|200|
|PATCH|/api/v1/admin/admins/:id|AdminHandler.Update|AdminUpdateRequest|AdminDetailResponse|200|
|DELETE|/api/v1/admin/admins/:id|AdminHandler.Delete|-（パスパラメータのみ）|-（本文なし）|204|

## Errorケース

|Endpoint|条件|Status Code|Error内容|
|-|-|-|-|
|List|-|200|該当なし（不正なページ番号等は上限・下限に丸めて処理し、②に明記のないエラー化は行わない。推測）|
|Show|対象管理者が存在しない、または削除済み|404|`ErrAdminNotFound`|
|Create|`email`未入力・形式不正|422|Request DTOバリデーションエラー|
|Update|対象管理者が存在しない|404|`ErrAdminNotFound`|
|Update|電話番号が10〜11桁の数字でない|422|`ErrInvalidPhoneNumber`|
|Update|生年月日が未来日付|422|`ErrInvalidBirthday`|
|Update|性別が許容値外|422|`ErrInvalidGender`|
|Delete|対象管理者が存在しない|404|`ErrAdminNotFound`|
|Delete|削除対象が操作者自身|422|`ErrCannotDeleteSelf`（②「16. API互換方針」の「自分自身は削除できません」メッセージを踏襲）|
|Delete|削除により有効な管理者が0人になる|422|`ErrLastAdminCannotBeDeleted`（②「16. API互換方針」の「最後の管理者は削除できません」メッセージを踏襲）|
|全Endpoint共通|予期せぬDB接続失敗等|500|Infrastructure Error|

---

# 8. Transaction実装方針

## Transaction開始箇所

②「11. Transaction設計」により、UseCaseの開始時（読み取り専用UseCaseを除く）にトランザクションを開始する。

> **②からの補足**: 具体的な実装機構は②に明記がない。アーキテクチャ規約「3. レイヤー責務と依存方向」の禁止事項（UseCaseがInfrastructure具象実装に依存しない）を満たすため、認証機能の実装仕様と同一の方式で`TransactionManager`インターフェースを導入する（推測）。

- interface名: `TransactionManager`（定義場所: `application/usecase/transaction_manager.go`、利用側であるapplication層で定義する。コーディング規約「5. インターフェース」準拠）
- メソッドシグネチャ: `WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error`
- 実装: `infrastructure/repository`配下でGORMの`*gorm.DB.Transaction`を用いて実装し、`fn`に渡す`ctx`にトランザクション用の`*gorm.DB`を保持させる

## Transaction終了箇所（Commit / Rollback条件）

|UseCase|終了箇所|
|-|-|
|ListAdminsUseCase / ShowAdminUseCase|トランザクションを使用しない（読み取りのみ）|
|CreateAdminUseCase|`AdminAccountRepository.Create`・`AdminPersonalInfoRepository.Create`が完了した時点でCommit、いずれかの失敗でRollback|
|UpdateAdminUseCase|`AdminAccountRepository.Update`・`AdminPersonalInfoRepository.Update`が完了した時点でCommit、いずれかの失敗でRollback|
|DeleteAdminUseCase|`AdminAccountRepository.CountActive`取得から`AdminAccountRepository.Delete`完了までを終えた時点でCommit、判定違反・失敗時はRollback|

## 複数Repositoryにまたがる場合の扱い

CreateAdminUseCase・UpdateAdminUseCaseは`AdminAccountRepository`と`AdminPersonalInfoRepository`の両方を、1つの`TransactionManager.WithinTransaction`スコープ内で呼び出し、`users`と`user_personal_infos`の不整合な中間状態（②「5. Aggregate設計」）を防ぐ。DeleteAdminUseCaseは`AdminAccountRepository`のみを扱うが、`CountActive`取得と`Delete`実行の間に他の削除操作が割り込むと「最後の管理者」ルールが破られる可能性があるため、同一トランザクション内で実行する（②「11. Transaction設計」の理由）。

> **②からの補足**: 同時実行時の排他制御（例: 行ロック）の具体的な要否・方式は②に明記がない。トランザクション分離レベルまたは明示的ロックの必要性は、実装時にDBの同時実行特性を踏まえて別途確認する必要がある（推測）。

---

# 9. Validation実装方針

## Presentation

- 型チェック: 各Request DTOの`binding`タグ（Ginの`ShouldBind`使用）で型を検証する
- 必須チェック: `AdminCreateRequest.Email`の必須性を`binding:"required"`で検証する（②「12. Validation設計」）
- フォーマットチェック: `email`は`binding:"email"`、`phone_number`は`binding:"numeric"`、`birthday`は`binding:"datetime=2006-01-02"`で形式を検証する

## 業務ルール検証（Domain Model採用のため②の記載どおり）

- Entity／Value Object生成時に検証する内容:
  - `valueobject.NewAdminDisplayName`: 名前未入力時のデフォルト補完
  - `valueobject.NewPhoneNumber`: 電話番号の桁数（10〜11桁）
  - `valueobject.NewBirthday`: 生年月日の未来日付禁止
  - `valueobject.NewGender`: 性別の許容値
  - `entity.AdminAccount.MarkDeleted`: 削除対象が既に削除済みでないかの状態チェック
- UseCase内で判定する業務ルール:
  - `ListAdminsUseCase`: `PerPage`の上限制御（100件）
  - `DeleteAdminUseCase`: `AdminDeletionPolicy.Authorize`による「自分自身でないこと」「最後の管理者でないこと」の整合性チェック

---

# 10. Authorization実装方針

②「13. Authorization設計」をそのまま実装レベルに落とし込む。

## Middleware

- Cookie/Header等から認証済みユーザーを特定し、`current_admin`としてGinの`*gin.Context`に保持する
- 役割が"admin"であることを確認する（役割不一致の場合は401または403で拒否。具体的なステータスコードは②に明記がないため、既存の他機能の認証Middleware実装方針に合わせる。推測）

## Handler

- ルーティング層でAPIの入口を担当する
- 具体的な削除可否判定は持たせない（②の方針どおり、`current_admin`のIDを`dto.DeleteAdminCommand.CurrentAdminID`へ橋渡しするのみ）

## UseCase

- `DeleteAdminUseCase`が、操作者（`cmd.CurrentAdminID`）のIDを削除対象（`cmd.AdminID`）と比較する材料として`AdminDeletionPolicy`へ渡す
- 有効管理者数を`AdminAccountRepository.CountActive`から取得し、`AdminDeletionPolicy`へ渡す

## Domain

- `AdminDeletionPolicy`が「自分自身か」「最後の管理者か」という業務レベルの認可判定を行う
- `AdminAccount`は判定結果を受けて`MarkDeleted`で状態遷移を実行するのみで、判定ロジック自体は保持しない

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

各UseCaseは、`domain/errors`のセンチネルエラーを`errors.Is`で判定し、必要に応じてApplication層独自のラップ（`fmt.Errorf("...: %w", err)`、コーディング規約「6. エラーのラップ」準拠）を行った上でHandlerへ伝播させる。「対象管理者未存在」は②「14. Error設計」によりApplication Errorとして扱うため、`application/usecase/errors.go`に`ErrAdminNotFound`をセンチネルエラー変数として定義する（②に配置場所の明記はないため推測）。

## Application Error → HTTPレスポンスへの変換方針

Handler層で`errors.Is`によりエラー種別を判定し、以下のStatus Codeへ変換する。変換ロジックは共通のエラーマッピング関数（`presentation/handler`配下の非公開ヘルパー）に集約し、各Handlerで重複させない。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrAdminNotFound`|Application|404|
|`ErrCannotDeleteSelf`|Domain|422|
|`ErrLastAdminCannotBeDeleted`|Domain|422|
|`ErrAdminAlreadyDeleted`|Domain|422（通常はGORM標準の論理削除機構により`ErrAdminNotFound`として先に検出される。5章「②からの補足」参照）|
|`ErrInvalidPhoneNumber`|Domain|422|
|`ErrInvalidBirthday`|Domain|422|
|`ErrInvalidGender`|Domain|422|
|Request DTOバリデーションエラー|Presentation|422|
|DB接続失敗・クエリ失敗|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

Infrastructure層（Repository実装）で発生したエラーは、`fmt.Errorf`でラップしてApplication層へ伝播させ、Handler層で未分類のエラーとして500に変換する。業務エラー（Domain Error）と技術的障害（Infrastructure Error）を`errors.Is` / `errors.As`で明確に区別する。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」により、既存Rails DBをそのまま継続利用し、スキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

### UserModel（`infrastructure/persistence/gorm/user_model.go`）

- 対応テーブル: `users`（既存Railsスキーマ。認証機能の`internal/auth/infrastructure/persistence/gorm/user_model.go`と同一テーブルを参照するが、Bounded Context間の独立性を保つため、本Contextは自身のモデル定義を別途保持する（②からの補足。認証機能③文書の先例に倣う）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー（Gorm規約のデフォルト`ID`フィールドをそのまま利用）|
|`Email`|`email`|
|`Name`|`name`|
|`NameKana`|`name_kana`|
|`AddressID`|`address_id`|
|`UserRoleID`|`user_role_id`|
|`DeletedAt`|`deleted_at`|GORM標準の論理削除機構（`gorm.DeletedAt`型）を用いる（5章「②からの補足」参照）|
|`CreatedAt`|`created_at`|Gorm規約のタイムスタンプ自動トラッキングに従う|
|`UpdatedAt`|`updated_at`|同上|

- テーブル名: 構造体名`UserModel`はGorm規約のデフォルト複数形変換では`user_models`となり実テーブル名`users`と一致しないため、`Tabler`インターフェース（`func (UserModel) TableName() string { return "users" }`）を実装してテーブル名を明示的に指定する（Gorm規約「テーブル名」節）。

### UserPersonalInfoModel（`infrastructure/persistence/gorm/user_personal_info_model.go`）

- 対応テーブル: `user_personal_infos`

|フィールド|対応カラム|備考|
|-|-|-|
|`UserID`|`user_id`|
|`PhoneNumber`|`phone_number`|`*string`（NULL許容）|
|`Birthday`|`birthday`|`*time.Time`（NULL許容）|
|`Gender`|`gender`|`*string`（NULL許容）|
|`CreatedAt`|`created_at`|
|`UpdatedAt`|`updated_at`|

- テーブル名: 構造体名から規約どおりのデフォルト複数形変換（`user_personal_infos`）が実テーブル名と一致するため、`Tabler`の実装は不要（推測。①未提供のため実テーブル名の完全一致は確認できていない。実装時にRailsスキーマとの一致を確認する必要がある）。

### UserRoleModel（`infrastructure/repository/user_role_repository.go`内、または`infrastructure/persistence/gorm/`）

- `user_roles`テーブルに対し、存在確認・ID解決のみに利用する最小フィールド（`ID`, `Name`）で定義する

## 主要クエリの条件・ソート・ページネーション方針

|操作|条件|ソート|ページネーション|
|-|-|-|-|
|`AdminAccountRepository.List`|`user_role_id = ?`（admin）＋キーワード指定時`name LIKE ?`または`email LIKE ?`|②に具体的な並び順の明記がない。実装時に確認要（推測。本書では暫定的に`id`昇順等の一般的な順序を想定するが確定しない）|`page`/`per_page`（上限100件）によるOFFSET/LIMIT|
|`AdminAccountRepository.FindByID`|`id = ?`かつ`user_role_id = ?`|不要（一意検索）|不要|
|`AdminAccountRepository.CountActive`|`user_role_id = ?`かつ`deleted_at IS NULL`|不要|不要|
|`AdminPersonalInfoRepository.FindByUserID`|`user_id = ?`|不要（一意検索）|不要|
|`UserRoleRepository.FindIDByName`|`name = ?`|不要|不要|

## 既存Schemaへの変更

②「17. DB設計方針」により変更なし。SQL文そのものは本書に記載しない。

---

# 13. テストケース設計

②「18. テスト戦略」をDomain Model採用時の区分のまま、具体的なテストケース単位に落とし込む。

## Domain Test

|対象|テストケース|
|-|-|
|`AdminDeletionPolicy.Authorize`|削除対象が操作者自身の場合に`ErrCannotDeleteSelf`を返す／有効管理者数が1人の場合に`ErrLastAdminCannotBeDeleted`を返す／自分自身でなく2人以上有効な場合に成功する（②の重点検証項目）|
|`AdminDisplayName.NewAdminDisplayName`|名前が指定された場合そのまま採用される／空文字・未指定の場合にメールアドレスのローカル部が採用される|
|`PhoneNumber.NewPhoneNumber`|10桁・11桁の数字で成功する／9桁以下・12桁以上・数字以外を含む場合に`ErrInvalidPhoneNumber`相当を返す|
|`Birthday.NewBirthday`|基準時刻以前の日付で成功する／基準時刻より未来の日付で`ErrInvalidBirthday`相当を返す|
|`Gender.NewGender`|許容値で成功する／許容値外で`ErrInvalidGender`相当を返す|
|`AdminAccount.MarkDeleted`|未削除状態から削除済みへの遷移が成功する／既に削除済みの状態で呼び出すと`ErrAdminAlreadyDeleted`を返す|

## UseCase Test

|対象|テストケース|
|-|-|
|`ListAdminsUseCase`|キーワード検索が反映される／`PerPage`が100を超える指定時に100件へ丸められる|
|`ShowAdminUseCase`|存在する管理者IDで個人情報を含む詳細が返る／存在しない管理者IDで`ErrAdminNotFound`を返す|
|`CreateAdminUseCase`|名前指定時にそのまま作成される／名前未指定時にメールアドレスのローカル部がデフォルト名として作成される|
|`UpdateAdminUseCase`|基本情報・個人情報が正しく更新される／存在しない管理者IDで`ErrAdminNotFound`を返す／不正な電話番号・生年月日・性別でそれぞれ対応するDomain Errorを返す|
|`DeleteAdminUseCase`|最後の管理者を削除しようとした場合にエラーになる／自分自身を削除しようとした場合にエラーになる／条件を満たす場合に正常に削除される（②の重点検証項目）|

## Repository Test

|対象|テストケース|
|-|-|
|`AdminAccountRepository.List`|キーワード検索・ページング・総件数取得の正確性|
|`AdminAccountRepository.FindByID`|存在する/しない管理者IDでの検索結果、削除済み管理者が除外されること|
|`AdminAccountRepository.CountActive`|有効な管理者のみがカウントされ、削除済みが除外されること|
|`AdminAccountRepository.Delete`|`deleted_at`が正しく反映されること|
|`AdminPersonalInfoRepository.FindByUserID` / `Create` / `Update`|個人情報の作成・更新・NULL許容項目の取り扱いの正確性|

## Handler Test

|対象|テストケース|
|-|-|
|`AdminHandler.List`|クエリパラメータのバインド結果が正しく`ListAdminsUseCase`へ渡される|
|`AdminHandler.Show`|存在しないIDで404を返す／成功時に200を返す|
|`AdminHandler.Create`|`email`未入力・形式不正で422を返す／成功時に200を返す|
|`AdminHandler.Update`|不正な`phone_number`/`birthday`形式で422を返す／成功時に200を返す|
|`AdminHandler.Delete`|最後の管理者・自分自身の削除試行で422を返す／存在しないIDで404を返す／成功時に204を返す|

## Integration Test

|対象|テストケース|
|-|-|
|一覧→詳細→作成→更新→削除の一連フロー|各操作がエンドポイント経由で正常に完了すること|
|削除の安全弁ルール|APIレベルで、自分自身の削除・最後の管理者の削除がそれぞれ422で拒否されること（②の重点検証項目）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に列挙する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ディレクトリ名を`internal/admin_account`とした|②のContext名は`admin-account-management`のみで、ディレクトリ名の明記がない。アーキテクチャ規約「5. 命名規則」に従い短縮した（認証機能の先例に倣う）|推測|
|`domain/repository/user_role_repository.go`を新設し、Authentication Contextが公開する参照手段として扱った|②「3. Bounded Context」がAccount/Authentication Contextへの依存（実体の作成・識別）を明記しているが、具体的なInterface名・配置場所までは規定していない。アーキテクチャ規約「6. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従った|推測|
|CreateAdminUseCase実行時、AdminPersonalInfoを各項目未設定（`nil`）の状態で生成し、後続のUpdateAdminUseCaseで値を設定する構成とした|②「6. Entity設計」のAdminPersonalInfoライフサイクル（作成に付随）と、②「10. UseCase設計」のCreateAdminUseCase入力（name/emailのみ）を矛盾なく実装するための判断|②の記載同士の整合を取るための判断（新しい業務ルールの追加ではない）|
|`UpdateAdminCommand`からメールアドレスの変更項目を除外した|②「10. UseCase設計」がUpdateAdminUseCase入力を抽象的にしか記載しておらず、メールアドレス変更に関する業務ルールの記載が②のどこにもないため|推測（実装時に②側での再確認が必要）|
|Genderの具体的な許容値（列挙値）を確定しなかった|②「7. Value Object設計」はRails `UserPersonalInfo.genders`への準拠を示すのみで具体的な値を記載していない。①（Rails実装詳細）が本タスクでは未提供のため、値を確定できない|①未提供のため参照不可|
|`Birthday`の生成時に基準時刻（`now`）を引数として明示的に受け取る構成とした|②は「未来日付を許容しない」というルールのみを定め、テスト容易性のための基準時刻の扱いは規定していない。認証機能の`ResetTokenValidityPeriod.IsExpired`と同様の構成とした|推測|
|`TransactionManager`インターフェースによるトランザクション制御方式を導入した|②「11. Transaction設計」はトランザクション境界（開始・終了タイミング）のみを定めており、UseCaseがGORMに直接依存しないための実装機構は指定していない。アーキテクチャ規約「3. レイヤー責務と依存方向」の禁止事項を満たすための判断。認証機能の実装仕様と同一方式とした|推測|
|CreateAdminUseCaseにおいて、AdminAccountとAdminPersonalInfoの作成を1トランザクションに含めた|②「11. Transaction設計」はAdminAccount作成のみをコミット単位として記載するが、②「5. Aggregate設計」がAdminAccountとAdminPersonalInfoを1つの整合性単位としているため、作成時も両テーブルへの反映を1トランザクションとした|②の記載同士を整合させるための判断（新しい業務ルールの追加ではない）|
|「対象管理者未存在」を表す`ErrAdminNotFound`を`application/usecase/errors.go`に配置した|②「14. Error設計」がこれをApplication Errorとして明記しているが、具体的な配置ファイルは指定していない|推測|
|GORM標準の論理削除機構（`gorm.DeletedAt`型フィールド）を用いて`deleted_at`を扱う構成とした|②・Gorm規約のいずれも具体的な実装機構を指定していないが、②が前提とする`deleted_at`による論理削除仕様を実現する一般的なGORM機能であるため採用した|推測|
|一覧取得のデフォルト並び順・`per_page`のデフォルト値を確定しなかった|②「9. Repository設計」「10. UseCase設計」は上限（100件）のみを明記し、具体的な並び順・デフォルト件数の記載がないため|推測（実装時に確認要）|
|管理者作成（`POST`）成功時のHTTP Status Codeを200とした|②「16. API互換方針」の「Status Code」節が「200: 取得・作成・更新成功」と明記しているため、REST慣習上の201ではなく②の記載どおり200を採用した|②の明記どおり（推測ではない）|

上記以外の設計判断（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
