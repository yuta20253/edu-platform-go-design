# 管理者管理者ユーザー機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者（`admin`ロールのユーザー）自身が、管理者アカウントの検索・ページング付き一覧参照、詳細参照、新規作成、既存管理者のプロフィール・個人情報更新、論理削除を行う機能である。新規作成のうち、管理者アカウント本体の作成（仮パスワード・招待待ち・招待メールの送信依頼・氏名未入力時の補完）は`user` Contextの`CreateAdminAccount`が担い、本機能はその呼び出しと、作成結果の管理者詳細としての返却を担う（②「3. Bounded Context」）。削除操作には「自分自身は削除できない」「削除後も有効な管理者が1人以上残ること（最後の管理者は削除できない）」という2つの安全弁ルールが適用される。

また、管理者プロフィール更新時の入力補助として、都道府県・市区町村・町域から住所候補を検索するエンドポイントを提供する。この住所候補検索は本機能が住所マスタを所有して行うものではなく、`master-data`（共通マスタ参照）Contextが提供する参照手段を、管理者向けの利用経路として中継するものである（②「3. Bounded Context」）。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により、本機能は **Domain Model** を採用する。

- 管理者アカウントが「有効」「削除済み（論理削除）」という状態を持ち、状態遷移（有効→削除済み）に「自分自身でないこと」「削除後も有効な管理者が1人以上残ること」という複数の前提条件が伴うこと
- 「最後の管理者かどうか」の判定が、対象アカウント単体の属性だけでなく、システム全体の有効管理者数という集約横断の情報に依存すること
- 電話番号の桁数、性別の許容値、生年月日の未来日付禁止という複数の業務ルールが更新時に絡むこと
- 削除可否判定ロジックを独立したDomain Serviceとして切り出すことで、将来の権限段階追加等の変更に対する耐性とテスト容易性を確保する必要があること

上記の理由からTransaction Script・Active Record・Event Sourcingは採用せず、Domain Modelを採用している（詳細は②「4. 設計パターン」参照）。

## 本書が対象とする実装範囲

本書は、②で確定した設計（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API仕様・DB方針・テスト戦略）を変更せず、Goでの具体的なコード構成（package構成・struct定義・interfaceメソッドシグネチャ・クエリ内容）に落とし込むことを目的とする。

規約`アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」の「Domain Model」節に従い、`{context}/domain`・`application`・`infrastructure`・`presentation`のフルレイヤー構成を適用する。

本書は②「12. UseCase設計」のCreateAdminUseCaseのうち、`user` Contextの`CreateAdminAccount`を呼び出す部分（外部依存`AdminAccountCreator`）を、本Contextの実装範囲として定める。`CreateAdminAccount`自体の実装は`user` Contextの③が扱い、本書では扱わない。

本書は②「12. UseCase設計」のSearchAddressCandidatesUseCase（住所候補検索）も実装対象に含める。同UseCaseはmaster-data Contextへの薄い中継処理であり、本機能のDomain Model採用判断（②「4. 設計パターン」）に寄与するものではないが、②が本Contextの責務として明記するAPI（`GET /api/v1/admin/addresses`）であるため、本書の実装範囲に含める。

①Rails実装詳細は本タスクでは提供されていないため、①の実装コードそのものを根拠とする記載は行わない（①未提供のため参照不可）。②に明記された「Rails現行仕様の要約」の範囲でのみ言及する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名（②）: `admin-account-management`
- ディレクトリ名: `internal/admin_account`

  > **②からの補足**: ②にはディレクトリ名の明記がない。アーキテクチャ規約「4. Bounded Context構成」の命名規則（Context名はkebab-case、`internal/`配下は英単語1語または短いスネークケース）に従い、`admin-account-management`を短縮した（推測。`認証機能`が`authentication`→`internal/auth`と短縮した先例に倣う）。

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
- `domain/event/`: 対象外（②「18. Domain Event」により、本機能では現時点でDomain Eventを採用しないと明記されているため）
- `infrastructure/mail/`・`infrastructure/cache/`・`infrastructure/queue/`: 対象外（②に該当する外部連携要件の記載がないため。管理者の作成に伴う招待メールの送信依頼は、`user` Contextの`CreateAdminAccount`の内部で規約13の`jobs`テーブルへ登録されるため、本Contextは`jobs`を扱わない。②「18. Domain Event」が将来的な連携ポイント（削除時のJWT無効化）として記録している内容は「推測」かつ「現行仕様には明記されていない」とされており、本書でも実装対象としない）

## 作成するファイル一覧

```
internal/admin_account/domain/entity/admin_account.go
internal/admin_account/domain/entity/admin_personal_info.go

internal/admin_account/domain/valueobject/phone_number.go
internal/admin_account/domain/valueobject/birthday.go
internal/admin_account/domain/valueobject/gender.go

internal/admin_account/domain/repository/admin_account_repository.go
internal/admin_account/domain/repository/admin_personal_info_repository.go
internal/admin_account/domain/repository/admin_account_creator.go
internal/admin_account/domain/repository/address_lookup.go

internal/admin_account/domain/service/admin_deletion_policy.go

internal/admin_account/domain/errors/errors.go

internal/admin_account/application/dto/list_admins_dto.go
internal/admin_account/application/dto/show_admin_dto.go
internal/admin_account/application/dto/create_admin_dto.go
internal/admin_account/application/dto/update_admin_dto.go
internal/admin_account/application/dto/delete_admin_dto.go
internal/admin_account/application/dto/search_address_candidates_dto.go

internal/admin_account/application/usecase/transaction_manager.go
internal/admin_account/application/usecase/list_admins_usecase.go
internal/admin_account/application/usecase/show_admin_usecase.go
internal/admin_account/application/usecase/create_admin_usecase.go
internal/admin_account/application/usecase/update_admin_usecase.go
internal/admin_account/application/usecase/delete_admin_usecase.go
internal/admin_account/application/usecase/search_address_candidates_usecase.go
internal/admin_account/application/usecase/errors.go

internal/admin_account/infrastructure/persistence/gorm/user_model.go
internal/admin_account/infrastructure/persistence/gorm/user_personal_info_model.go

internal/admin_account/infrastructure/repository/admin_account_repository.go
internal/admin_account/infrastructure/repository/admin_personal_info_repository.go
internal/admin_account/infrastructure/repository/admin_account_creator.go
internal/admin_account/infrastructure/repository/address_lookup.go

internal/admin_account/presentation/handler/admin_handler.go
internal/admin_account/presentation/handler/address_handler.go

internal/admin_account/presentation/request/admin_request.go

internal/admin_account/presentation/response/admin_response.go

internal/admin_account/presentation/routes.go
```

> **②からの補足**: `domain/repository/admin_account_creator.go`・`infrastructure/repository/admin_account_creator.go`は、②「3. Bounded Context」「11. Repository設計」が定める`user` Contextへの依存（管理者アカウントの作成）を実装レベルで具体化したものである。本Contextは`AdminAccountCreator`というInterfaceを自ら定義し（アーキテクチャ規約「2. レイヤー責務と依存方向」の依存性逆転）、その実装は`users`テーブルへ直接書き込まず、`user` Contextが公開する`CreateAdminAccount`関数を呼び出す構成とする（アーキテクチャ規約「5. Context間連携ルール」）。管理者の作成で`user_role_id`を解決する必要はなくなるため、ロール名からロールIDを解決する専用のRepositoryは持たない。一覧・詳細・件数取得での管理者ロールによる絞り込みは、`AdminAccountRepository`の実装が`user_roles`テーブルとの結合で行う（②「21. DB操作仕様」）。
>
> `domain/repository/address_lookup.go`・`infrastructure/repository/address_lookup.go`・`application/usecase/search_address_candidates_usecase.go`・`presentation/handler/address_handler.go`は、②「3. Bounded Context」「11. Repository設計」が定めるmaster-data Contextへの依存（住所候補検索）を実装レベルで具体化したものである。本Contextは`AddressLookup`という参照専用Interfaceを自ら定義し（アーキテクチャ規約「2. レイヤー責務と依存方向」の依存性逆転）、その実装（`infrastructure/repository/address_lookup.go`）は`addresses`/`prefectures`テーブルへ直接クエリせず、master-data Context（共通マスタ参照機能）が公開する`SearchAddresses`関数（共通マスタ参照機能_Go実装仕様書「6. Application層設計」）を呼び出す構成とする（アーキテクチャ規約「5. Context間連携ルール」）。

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
|`Name`|`string`|氏名（作成時に氏名が未入力だった場合の補完は`user` Contextが行うため、本Contextは補完後の値をそのまま保持する）|
|`NameKana`|`string`|フリガナ|
|`AddressID`|`*uint`|住所（Address Context）への外部参照。②「5. Aggregate設計」によりAggregate外部参照として識別子のみを保持する|
|`UserRoleID`|`uint`|ロール識別子（本Contextが扱う対象は常に"admin"ロールのIDのみ）|
|`CreatedAt`|`time.Time`|作成日時（参照専用。管理者詳細のレスポンスで返す）|
|`UpdatedAt`|`time.Time`|更新日時（参照専用。管理者詳細のレスポンスで返す）|
|`DeletedAt`|`*time.Time`|論理削除日時。`nil`の場合は有効、値が設定されている場合は削除済み|

- 公開メソッド一覧（引数・戻り値のみ。ロジックは記述しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewAdminAccount`|`(id uint, email string, name string, nameKana string, addressID *uint, userRoleID uint, createdAt time.Time, updatedAt time.Time, deletedAt *time.Time) (*AdminAccount, error)`|`(*AdminAccount, error)`|不変条件を満たしたAdminAccountを生成するファクトリ（永続化層からの復元にも使用する。アカウントの新規作成は`user` Contextが行うため、本ファクトリは`user`が作成済みの行の復元に用いる）|
|`UpdateProfile`|`(name string, nameKana string, addressID *uint)`|`error`|基本情報（表示名・フリガナ・住所参照）を更新する|
|`MarkDeleted`|`(deletedAt time.Time)`|`error`|論理削除状態へ遷移させる。既に削除済みの場合は`ErrAdminAlreadyDeleted`を返す|
|`IsDeleted`|`()`|`bool`|削除済みかどうかを判定する|

- 不変条件（ファクトリで保証する内容）:
  - `Email`は空文字を許容しない
  - `UserRoleID`は0を許容しない

> **②からの補足**: ②「6. Entity設計」は削除実行時にAdminAccountが「事前に判定された削除可否結果を反映して状態を遷移させる」とし、判定ロジック自体は保持しないとしている。本書では、削除可否判定（自分自身／最後の管理者）は`AdminDeletionPolicy`が担い、`MarkDeleted`は「既に削除済みでないか」という状態遷移そのものの整合性（②「15. Validation設計」の「状態チェック」）のみを保証する構成とした。②の記載同士（Domain Serviceへの判定集約とEntityの状態チェック責務）を矛盾なく実装するための判断であり、新しい業務ルールの追加ではない。

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
|`NewAdminPersonalInfo`|`(userID uint, phone *valueobject.PhoneNumber, birthday *valueobject.Birthday, gender *valueobject.Gender) (*AdminPersonalInfo, error)`|`(*AdminPersonalInfo, error)`|個人情報を生成するファクトリ（各項目は未入力＝`nil`を許容する。永続化層からの復元では、一部の項目が`nil`の行も生成される。UpdateAdminUseCaseが新規作成する場合は、3項目のうち少なくとも1つが指定されている場合に限る）|
|`UpdatePersonalInfo`|`(phone *valueobject.PhoneNumber, birthday *valueobject.Birthday, gender *valueobject.Gender)`|`error`|電話番号・生年月日・性別を更新する|

- 不変条件: `UserID`は0を許容しない。各Value Objectフィールドは、値が設定される場合は必ずValue Objectのコンストラクタを経由した検証済みの値であることが保証される（`nil`は許容する）。

- 生成・作成のタイミング（②「6. Entity設計」）: 管理者の作成時（CreateAdminUseCase）にはAdminPersonalInfoを作成しない。UpdateAdminUseCaseが、個人情報が未登録の管理者に対して、電話番号・生年月日・性別のいずれかが指定された場合に限り新規作成する。3項目がすべて未指定の更新では作成しない（空のAdminPersonalInfoは作らない）。Rails現行の`Admin::AdminForm`が、既存の個人情報がなく個人情報も未入力の場合は空のレコードを作らないことに合わせる（Rails現行を根拠とする）

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

## Value Objectを採用しないもの

②「7. Value Object設計」の方針どおり、メールアドレスは本機能に固有の複雑な業務ルールを持たないため、`AdminAccount.Email string`にそのまま保持し、Value Object化しない。形式検証はPresentation層で行う。住所（`address_id`）も同様に、本Context内で独自の業務ルールを持たない単なる外部参照IDとして扱う。氏名も、未入力時の補完を`user` Contextが担う（②「7. Value Object設計」）ため、Value Object化せず`string`として保持する。

## 外部参照Entity（master-data Context提供）

### AddressCandidate

- struct名: `AddressCandidate`（`domain/repository/address_lookup.go`に、`AddressLookup` Interfaceの戻り値型として定義する。本Contextが所有するAggregateには含めない）
- フィールド: `ID uint`, `PostalCode string`, `City string`, `Town string`, `PrefectureName string`（②「6. Entity設計」の「表示・選択に必要な最小限の情報（id・郵便番号・市区町村・町域・都道府県名）」に対応）
- ライフサイクル・状態変化: 本Contextでは扱わない。作成・更新・削除の責務はmaster-data Contextが持つ（②の記載どおり）

## Repository Interface

### AdminAccountRepository（`domain/repository/admin_account_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`List`|`(ctx context.Context, params ListAdminAccountsParams)`|`([]*entity.AdminAccount, int64, error)`|検索キーワード・ページングを適用した一覧取得と総件数取得（②「11. Repository設計」）|
|`FindByID`|`(ctx context.Context, id uint)`|`(*entity.AdminAccount, error)`|特定管理者の取得（作成直後の再取得を含む。`CreatedAt` / `UpdatedAt`を返す）。存在しない場合は`(nil, nil)`を返し、種別判定はUseCase側に委ねる|
|`Update`|`(ctx context.Context, account *entity.AdminAccount)`|`error`|基本情報の更新|
|`Delete`|`(ctx context.Context, id uint, deletedAt time.Time)`|`error`|論理削除の実行（`deleted_at`の更新）|
|`CountActive`|`(ctx context.Context)`|`(int64, error)`|有効な管理者の件数取得（削除可否判定に使用）|

`ListAdminAccountsParams`のフィールド: `Keyword string`, `Page int`, `PerPage int`

- 保持しない責務: 削除可否の判定、管理者アカウントの作成（`user` Contextの`CreateAdminAccount`が行う。下記`AdminAccountCreator`を経由する）、名前のデフォルト補完ルール（②「11. Repository設計」の方針どおり）

### AdminPersonalInfoRepository（`domain/repository/admin_personal_info_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByUserID`|`(ctx context.Context, userID uint)`|`(*entity.AdminPersonalInfo, error)`|`user_id`による個人情報の取得。存在しない場合は`(nil, nil)`|
|`Create`|`(ctx context.Context, info *entity.AdminPersonalInfo)`|`error`|個人情報の作成（UpdateAdminUseCaseが、未登録かつ個人情報が1項目以上指定された場合のみ呼ぶ。管理者の作成時には呼ばない）|
|`Update`|`(ctx context.Context, info *entity.AdminPersonalInfo)`|`error`|個人情報の更新|

- 保持しない責務: 電話番号・生年月日・性別のバリデーション（Value Object側で表現済みの検証の再確認に留める。②「11. Repository設計」）

### AdminAccountCreator（`domain/repository/admin_account_creator.go`、user Context提供・外部依存として利用）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`Create`|`(ctx context.Context, params CreateAdminAccountParams)`|`(CreatedAdminAccount, error)`|`user` Contextの`CreateAdminAccount`を呼び、管理者アカウントを招待待ちで作成して招待メールの送信を依頼する（②「11. Repository設計」）。`ctx`に呼び出し側のトランザクションが保持されている場合は、それに参加する|

`CreateAdminAccountParams`のフィールド: `Name string`（任意。未入力ならメールアドレスの`@`より前が氏名になる。補完は`user` Contextが行う）, `Email string`
`CreatedAdminAccount`のフィールド: `ID uint`, `Name string`, `Email string`（`user` Contextの作成結果の基本属性のうち、本Contextが使うもの。`created_at` / `updated_at`は含まれない）

- 保持しない責務: 仮パスワードの発行・招待待ちの設定・招待メールの送信依頼・氏名の補完（すべて`user` Contextが行う）、作成結果の`created_at` / `updated_at`の提供（`AdminAccountRepository.FindByID`で再取得する）
- 返すエラー: `user` ContextのValidationエラーを、本Contextの`ErrAdminEmailAlreadyUsed` / `ErrInvalidAdminAccountInput`（下記Domain Error）へ変換して返す。`user` Contextの内部エラーは、Infrastructure Errorとしてラップして返す
- 判断根拠: ②「11. Repository設計」の「AdminAccountCreator（user Context提供、外部依存として利用）」を、本Context自身が定義するInterfaceとして具体化したものである。実装は、`user` Contextが公開する`CreateAdminAccount`の呼び出しに限る

### AddressLookup（`domain/repository/address_lookup.go`、master-data Context提供・外部依存として利用）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`Search`|`(ctx context.Context, params AddressSearchParams)`|`([]AddressCandidate, error)`|都道府県ID（必須）・市区町村・町域による住所候補の検索（②「11. Repository設計」）|

`AddressSearchParams`のフィールド: `PrefectureID uint`, `City string`, `Town string`

- 保持しない責務: 住所データの作成・更新・削除（②の記載どおり、住所マスタの変更は本機能のスコープ外）
- 判断根拠: ②「11. Repository設計」の「住所検索は共通マスタ参照機能が提供する参照手段であり、本Contextはこれを外部依存として呼び出す立場に留める」を、本Context自身が定義する参照専用Interfaceとして具体化したものである。アーキテクチャ規約「5. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従う

## Domain Service

### AdminDeletionPolicy（`domain/service/admin_deletion_policy.go`）

- struct名: `AdminDeletionPolicy`（状態を持たない。コンストラクタ`NewAdminDeletionPolicy() *AdminDeletionPolicy`は依存を受け取らない）
- 公開メソッド: `(p *AdminDeletionPolicy) Authorize(currentAdminID uint, targetAdminID uint, activeAdminCount int64) error`
- 責務: 削除対象が操作者自身でないこと、削除後も有効な管理者が1人以上残ることの2条件を判定し、いずれかに違反する場合は対応するDomain Errorを返す（②「8. Domain Service」）

## Domain Event

対象外。②「18. Domain Event」により、本機能では現時点でDomain Eventを採用しない。②は「管理者アカウントが削除された場合、認証コンテキスト側で該当ユーザーのセッション（JWT）を無効化する必要がある可能性がある」ことを将来の連携ポイントとして記録しているが、「現行仕様には明記されていない」「推測」と明記されているため、本書では`AdminAccountDeleted`等のイベントstruct・発火処理を実装対象に含めない。

## Domain Error

`domain/errors/errors.go`に、`errors.New`によるセンチネルエラー変数として定義する（②「17. Error設計」のDomain Errorに対応。認証機能の実装仕様と同一の方式を踏襲する）。`ErrAdminEmailAlreadyUsed` / `ErrInvalidAdminAccountInput`は、`user` Contextが返すValidationエラーを本Contextの語彙へ変換したもの（②「17. Error設計」のApplication Errorに相当）であり、`AdminAccountCreator`の実装（infrastructure層）が返すため、infrastructure層から参照できるdomain層に置く。

|変数名|発生条件|
|-|-|
|`ErrCannotDeleteSelf`|削除対象が操作者自身である|
|`ErrLastAdminCannotBeDeleted`|削除後に有効な管理者が0人になる（最後の管理者の削除試行）|
|`ErrAdminAlreadyDeleted`|既に削除済みの管理者に対して`MarkDeleted`が呼ばれた|
|`ErrInvalidPhoneNumber`|電話番号が数字のみ10〜11桁の形式を満たさない|
|`ErrInvalidBirthday`|生年月日が未来日付である|
|`ErrInvalidGender`|性別が許容値のいずれにも一致しない|
|`ErrAdminEmailAlreadyUsed`|`AdminAccountCreator.Create`が、`user` Contextから「メールアドレスが既存アカウント（無効化済みを含む）で使用済み」のValidationエラーを受け取った|
|`ErrInvalidAdminAccountInput`|`AdminAccountCreator.Create`が、`user` Contextから上記以外のValidationエラー（メールアドレスの形式不正・氏名の文字数超過）を受け取った|

---

# 4. Application層設計

## DTO（Command / Query）

|struct名|フィールドと型|区分|
|-|-|-|
|`ListAdminsQuery`|`CurrentAdminID uint`, `Keyword string`, `Page int`, `PerPage int`|Query|
|`AdminSummary`|`ID uint`, `Name string`, `Email string`|出力（一覧項目）|
|`ListAdminsResult`|`Admins []AdminSummary`, `Page int`, `PerPage int`, `TotalCount int64`|出力|
|`ShowAdminQuery`|`CurrentAdminID uint`, `AdminID uint`|Query|
|`ShowAdminResult`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `CreatedAt time.Time`, `UpdatedAt time.Time`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|出力|
|`CreateAdminCommand`|`CurrentAdminID uint`, `Name string`, `Email string`|Command|
|`CreateAdminResult`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `CreatedAt time.Time`, `UpdatedAt time.Time`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|出力（作成直後に再取得した管理者詳細。ShowAdminResultと同じ内容）|
|`UpdateAdminCommand`|`CurrentAdminID uint`, `AdminID uint`, `Name string`, `NameKana string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|Command|
|`UpdateAdminResult`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `CreatedAt time.Time`, `UpdatedAt time.Time`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *time.Time`, `Gender *string`|出力|
|`DeleteAdminCommand`|`CurrentAdminID uint`, `AdminID uint`|Command|
|`DeleteAdminResult`|（フィールドなし。成功したことのみを表す）|出力|
|`SearchAddressCandidatesQuery`|`CurrentAdminID uint`, `PrefectureID uint`, `City string`, `Town string`|Query（②「12. UseCase設計」の「入力: current admin, prefecture_id（必須）, city（任意）, town（任意）」に対応）|
|`AddressCandidateResult`|`ID uint`, `PostalCode string`, `City string`, `Town string`, `PrefectureName string`|出力（一覧項目）|
|`SearchAddressCandidatesResult`|`Candidates []AddressCandidateResult`|出力|

> **②からの補足**: `UpdateAdminCommand`が`Email`を含まない構成とした。②「12. UseCase設計」のCreateAdminUseCase入力にのみ`email`が明記され、UpdateAdminUseCase入力は「current admin, admin id, update request」と抽象的にしか記載がない。②「19. API仕様」のRequestパラメータ一覧には`email`が含まれるが、一覧・詳細・作成・更新のいずれで使われるかは区別されていない。本書ではメールアドレスの変更自体が②のどのUseCaseにも明示的な業務ルールとして記載されていないため、更新対象から除外した（推測。実装時に更新可否を②側で再確認する必要がある）。

## UseCase

### ListAdminsUseCase（`application/usecase/list_admins_usecase.go`）

- struct名: `ListAdminsUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`（Interface）
- 公開メソッド: `(u *ListAdminsUseCase) Execute(ctx context.Context, query dto.ListAdminsQuery) (dto.ListAdminsResult, error)`
- 処理ステップ:
  1. `PerPage`の正規化を行う（②「11. Repository設計」「12. UseCase設計」により、未指定・0以下の場合は既定値25件、100を超える場合は100件に丸める）
  2. `AdminAccountRepository.List`で検索・ページングを適用した一覧と総件数を取得する
  3. `ListAdminsResult`を組み立てて返す
- トランザクション境界: なし（②「14. Transaction設計」により読み取りのみ）
- 発生しうるApplication Error: なし（Infrastructure Errorのみ）

> `PerPage`の既定値25件・上限100件は、Rails現行（`Api::V1::Admin::AdminsController`の`DEFAULT_PER_PAGE`と、共通の`sanitized_per_page`・上限`MAX_PER_PAGE`）を根拠とする（①「管理者管理者ユーザー機能」参照）。並び順は`created_at`の降順（新しい順）である（13章のクエリ方針参照）。

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
- コンストラクタが受け取る依存: `AdminAccountCreator`（Interface、user Context提供）, `AdminAccountRepository`, `TransactionManager`
- 公開メソッド: `(u *CreateAdminUseCase) Execute(ctx context.Context, cmd dto.CreateAdminCommand) (dto.CreateAdminResult, error)`
- 処理ステップ:
  1. 管理者ロールによる操作であることは、呼び出し前にMiddlewareが認可済みである（10章）。UseCaseは`cmd.CurrentAdminID`を認可のためには使わない
  2. `TransactionManager.WithinTransaction`内で以下を実行する:
     a. `AdminAccountCreator.Create`で、`user` Contextの`CreateAdminAccount`を呼び、管理者アカウントを作成する（氏名（任意）・メールアドレスを渡す。氏名が未入力の場合の補完・仮パスワードの発行・招待待ちの設定・招待メールの送信依頼の登録は`user` Contextが行う）
     b. `AdminAccountRepository.FindByID`で、作成された管理者を再取得する。`user` Contextの作成結果に`created_at` / `updated_at`が含まれず、Rails現行の作成レスポンス（`Admin::AdminDetailSerializer`）が`created_at` / `updated_at`を含むため、再取得して補う（`AdminAccountCreator`の結果のIDを用いる）。取得できない場合は、作成直後の不整合であり内部エラーとして扱う
  3. `CreateAdminResult`を、再取得した`AdminAccount`から組み立てて返す（`AdminPersonalInfo`は作成しないため、`AddressID` / `PhoneNumber` / `Birthday` / `Gender`はすべて`nil`となる）
- トランザクション境界: `AdminAccountCreator.Create`・`AdminAccountRepository.FindByID`を1トランザクションとする。`AdminAccountCreator.Create`は`ctx`に保持されたトランザクションに参加するため、`user` Contextが登録する招待メールの送信依頼も同じトランザクションに含まれ、ロールバック時は取り消される
- 発生しうるApplication Error: なし（Infrastructure Errorのみ）
- 発生しうるDomain Error: `ErrAdminEmailAlreadyUsed`, `ErrInvalidAdminAccountInput`（`user` ContextのValidationエラーの変換）

> **②からの補足**: ②「14. Transaction設計」は「CreateAdminUseCaseでは、`CreateAdminAccount`の呼び出し・作成結果の再取得が完了した時点でコミットする」としている。`user` Contextの作成操作が呼び出し側のトランザクションに参加する方式（`context.Context`を介した引き継ぎ）は、`user`②の未解決の論点9であり、`user`の③で確定した方式に、`AdminAccountCreator`の実装を合わせる（本Contextの`TransactionManager`が`ctx`に保持するトランザクションを、`user`の関数へ`ctx`として渡す前提。規約11の方式）。

### UpdateAdminUseCase（`application/usecase/update_admin_usecase.go`）

- struct名: `UpdateAdminUseCase`
- コンストラクタが受け取る依存: `AdminAccountRepository`, `AdminPersonalInfoRepository`, `TransactionManager`
- 公開メソッド: `(u *UpdateAdminUseCase) Execute(ctx context.Context, cmd dto.UpdateAdminCommand) (dto.UpdateAdminResult, error)`
- 処理ステップ:
  1. `AdminAccountRepository.FindByID`で対象管理者を取得する。取得できない場合は`ErrAdminNotFound`を返す
  2. `cmd.Name`（氏名。更新時は必須。Presentationで検証済み）を、そのまま氏名として扱う
  3. `cmd.PhoneNumber` / `cmd.Birthday` / `cmd.Gender`が指定されている場合のみ、それぞれ`valueobject.NewPhoneNumber` / `valueobject.NewBirthday` / `valueobject.NewGender`で検証する
  4. `account.UpdateProfile`で基本情報を更新する
  5. `AdminPersonalInfoRepository.FindByUserID`で既存の個人情報を取得し、次のとおり扱う（電話番号・生年月日・性別の「指定なし」は、`nil`または空文字とする）
     - 既存がある場合: `UpdatePersonalInfo`で3項目を更新入力の値で置き換える（指定のない項目は`nil`になる）
     - 既存がなく、3項目のいずれかが指定されている場合: `entity.NewAdminPersonalInfo`で新規に生成する
     - 既存がなく、3項目がすべて指定なしの場合: `AdminPersonalInfo`を作成せず、個人情報に関する永続化を行わない
  6. `TransactionManager.WithinTransaction`内で`AdminAccountRepository.Update`を実行し、手順5で既存があった場合は`AdminPersonalInfoRepository.Update`、新規に生成した場合は`AdminPersonalInfoRepository.Create`を実行する
  7. `UpdateAdminResult`を組み立てて返す
- トランザクション境界: AdminAccountとAdminPersonalInfoの更新（新規作成を含む）を1トランザクションとする（②「14. Transaction設計」）
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
- トランザクション境界: 有効管理者数の取得から削除実行までを1トランザクションとする（②「14. Transaction設計」。「最後の管理者」ルールの競合を防ぐため）
- 発生しうるApplication Error: `ErrAdminNotFound`
- 発生しうるDomain Error: `ErrCannotDeleteSelf`, `ErrLastAdminCannotBeDeleted`, `ErrAdminAlreadyDeleted`

### SearchAddressCandidatesUseCase（`application/usecase/search_address_candidates_usecase.go`）

- struct名: `SearchAddressCandidatesUseCase`
- コンストラクタが受け取る依存: `AddressLookup`（Interface、master-data Context提供）
- 公開メソッド: `(u *SearchAddressCandidatesUseCase) Execute(ctx context.Context, query dto.SearchAddressCandidatesQuery) (dto.SearchAddressCandidatesResult, error)`
- 処理ステップ:
  1. `query.PrefectureID`が0（未指定）の場合、`ErrPrefectureIDRequired`を返す（②「12. UseCase設計」「15. Validation設計」の「都道府県は必須です。」に対応）
  2. `AddressLookup.Search`へ`AddressSearchParams{PrefectureID, City, Town}`を渡して住所候補を検索する
  3. 検索結果を`SearchAddressCandidatesResult`へ変換して返す
- トランザクション境界: なし（②「14. Transaction設計」の「住所候補検索はmaster-data Contextへの問い合わせのみであり、データ変更を伴わないためトランザクション不要」に対応）
- 発生しうるApplication Error: `ErrPrefectureIDRequired`
- 判断根拠: ②「12. UseCase設計」が「本UseCaseはmaster-data Contextへの問い合わせを管理者向けに中継するのみであり、住所検索そのものの業務ルール（都道府県必須等）はmaster-data Context側の責務である」と明記しているとおり、本UseCaseはAddressLookupを呼び出すだけの薄い構成とする。ただし「都道府県必須」というPresentation/Application境界の入力検証自体は、Handlerではなく本UseCase側のガード節として実装する（②「15. Validation設計」の「Presentation: 必須チェック: ...住所候補検索時のprefecture_idの必須性を検証する」の記載を踏まえ、Request DTOレベルの型チェックはPresentationで行い、業務的な必須判定はUseCaseのガード節で行う二重の検証とする）

---

# 5. Infrastructure層設計

## Repository実装

### AdminAccountRepository実装（`infrastructure/repository/admin_account_repository.go`）

- 実装struct名: 非公開struct（例: `adminAccountRepository`）+ コンストラクタ`NewAdminAccountRepository`で公開する構成（規約「9. 命名規約」の「`Impl`接尾辞を付けない」方針に従う。認証機能の実装仕様と同一の方式）
- 対応するGORMモデル: `gormmodel.UserModel`（`infrastructure/persistence/gorm/user_model.go`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`List`|`user_roles`との結合による管理者ロール（`user_roles.name = 'admin'`）の絞り込みに加え、`Keyword`が指定されていれば`name LIKE ?`または`email LIKE ?`（②「11. Repository設計」の「キーワード検索（名前・メールアドレス等）」）。`Page`/`PerPage`によるOFFSET/LIMIT。総件数はCOUNTクエリで別途取得|GORM標準の論理削除機構により`deleted_at IS NULL`が自動的に条件へ付与される（後述「②からの補足」参照）|
|`FindByID`|`id = ?`かつ管理者ロール（`user_roles`との結合）で1件検索。`created_at` / `updated_at`も取得する|同上、削除済みは自動的に除外される。作成直後の再取得（CreateAdminUseCase）でも用いる|
|`Update`|`id = ?`条件で`name`, `name_kana`, `address_id`を更新|
|`Delete`|`id = ?`条件で`deleted_at`を現在時刻へ更新（論理削除）|
|`CountActive`|管理者ロール（`user_roles`との結合）かつ`deleted_at IS NULL`の件数を取得|

- Entity ⇔ GORMモデルの変換方針: `gormmodel.UserModel`から`entity.AdminAccount`への変換関数（`toEntity`）、更新用の変換関数（`toModel`）をrepository実装内の非公開関数として用意する。`Name`は`string`のまま受け渡す。

> **②からの補足**: ②「20. DB設計方針」は`users`の`deleted_at`による論理削除を前提とするが、実装機構（GORMの`gorm.DeletedAt`型フィールドによる標準の論理削除機能を用いるか、明示的な`WHERE deleted_at IS NULL`条件を都度付与するか）は②・Gorm規約のいずれにも明記がない。本書ではGORM標準の論理削除機構（`gorm.DeletedAt`型フィールド）を採用し、`Delete`実行時に`deleted_at`が自動設定され、`List`/`FindByID`等の標準検索メソッドが自動的に削除済みレコードを除外する構成とする（推測。Gorm規約に反しない一般的なGORM機能であり、②が前提とする論理削除仕様を実現する）。この結果、Domain層の`ErrAdminAlreadyDeleted`は主に直接`MarkDeleted`を呼び出す経路の防御的チェックとして機能し、通常のHTTP経由の操作では「対象不存在」（`ErrAdminNotFound`）として先に検出される。

### AdminPersonalInfoRepository実装（`infrastructure/repository/admin_personal_info_repository.go`）

- 実装struct名: 非公開struct（例: `adminPersonalInfoRepository`）+ コンストラクタ`NewAdminPersonalInfoRepository`
- 対応するGORMモデル: `gormmodel.UserPersonalInfoModel`（`infrastructure/persistence/gorm/user_personal_info_model.go`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`FindByUserID`|`user_id = ?`で1件検索|存在しない場合は`(nil, nil)`を返す|
|`Create`|`user_personal_infos`テーブルへ1件作成。UpdateAdminUseCaseが、未登録かつ個人情報が1項目以上指定された場合に呼ぶ。指定のない項目（`nil`）のカラムはNULLのまま作成する|
|`Update`|`user_id = ?`条件で`phone_number`, `birthday`, `gender`を更新|

- Entity ⇔ GORMモデルの変換方針: `PhoneNumber`/`Birthday`/`Gender`各Value Objectと、GORMモデルのnull許容カラム（`*string`/`*time.Time`/`*string`）との相互変換関数を用意する。DB上に値が存在する場合のみ対応するValue Objectコンストラクタで検証し直す。不正な形式が保存されていた場合はInfrastructure Errorとして扱う。

### AdminAccountCreator実装（`infrastructure/repository/admin_account_creator.go`）

- 実装struct名: 非公開struct（例: `adminAccountCreator`）+ コンストラクタ`NewAdminAccountCreator(...) repository.AdminAccountCreator`
- `users`テーブルへは直接書き込まず、`user` Contextが公開する`CreateAdminAccount`関数（`user`②「12. UseCase設計」）を呼び出す。`ctx`は、`TransactionManager`が保持するトランザクションを含んだまま、`user`の関数へ渡す
- 変換方針: `AdminAccountCreator.Create`の`CreateAdminAccountParams`（`Name` / `Email`）を`CreateAdminAccount`の入力（氏名・メールアドレス）へ、戻り値の基本属性（`id`・氏名・メールアドレス）を`CreatedAdminAccount`へ変換する。`user` ContextのValidationエラーは`ErrAdminEmailAlreadyUsed` / `ErrInvalidAdminAccountInput`へ、内部エラーは`fmt.Errorf`でラップしてInfrastructure Errorとして返す
- `user` Contextの③が確定した際、公開関数のシグネチャ・エラーの区別方法に合わせる（14章）

### AddressLookup実装（`infrastructure/repository/address_lookup.go`）

- 実装struct名: 非公開struct（例: `addressLookup`）+ コンストラクタ`NewAddressLookup(db *gorm.DB) repository.AddressLookup`
- `addresses`・`prefectures`テーブルはmaster-data Context（共通マスタ参照機能）が正規の所有者であるため、本Contextでは対応するGORMモデルを定義せず、`AddressLookup.Search`は`masterdata.SearchAddresses(ctx, db, prefectureID, city, town)`（共通マスタ参照機能_Go実装仕様書「6. Application層設計」）を呼び出す
- 変換方針: `masterdata.SearchAddresses`の戻り値`[]masterdata.Address`（`ID` / `PostalCode` / `City` / `Town` / `Prefecture.Name`）を、本Contextの`domain/repository.AddressCandidate`（`ID` / `PostalCode` / `City` / `Town` / `PrefectureName`）へ変換する

## 外部連携実装

対象外。②に本機能でのMail・Cache・Queue連携要件の記載はない。②「18. Domain Event」が記録する将来的なJWT無効化連携（推測、現行仕様に非明記）は本書の実装対象に含めない。

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
- `Create`処理順序: リクエストバインド（`request.AdminCreateRequest`）→ Presentation Validation（`email`必須・形式チェック）→ `CreateAdminUseCase.Execute`呼び出し（内部で`user` Contextの`CreateAdminAccount`を呼ぶ。管理者ロールの認可は、ルーティングに適用済みのMiddlewareが済ませている）→ `response.AdminDetailResponse`（作成結果）へ変換して返却（②「19. API仕様」により201）
- `Update`処理順序: パスパラメータ（`id`）＋リクエストバインド（`request.AdminUpdateRequest`）→ Presentation Validation（氏名の必須・電話番号・生年月日の形式チェック）→ `UpdateAdminUseCase.Execute`呼び出し → `response.AdminDetailResponse`へ変換して返却（200）
- `Delete`処理順序: パスパラメータ（`id`）バインド → Middlewareが設定した`current_admin`のIDを`dto.DeleteAdminCommand.CurrentAdminID`へ設定 → `DeleteAdminUseCase.Execute`呼び出し → 成功時は204（本文なし）を返却。Handler自身は削除可否判定を持たない（②「16. Authorization設計」の「Handler: 具体的な削除可否判定は持たせない」方針に従う）

### AddressHandler（`presentation/handler/address_handler.go`）

- struct名: `AddressHandler`
- 対応する呼び出し先: `*usecase.SearchAddressCandidatesUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`Search`|GET|`/api/v1/admin/addresses`|

- `Search`処理順序: クエリパラメータバインド（`request.AddressSearchRequest`）→ Presentation Validation（`prefecture_id`必須・型チェック）→ Middlewareが設定した`current_admin`のIDを`dto.SearchAddressCandidatesQuery.CurrentAdminID`へ設定 → `SearchAddressCandidatesUseCase.Execute`呼び出し → `response.AddressSearchResponse`へ変換して返却（200）。`prefecture_id`未指定時は400（②「19. API仕様」参照）

## Request / Response DTO

### Request（`presentation/request/admin_request.go`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`AdminListRequest`|`Q string`, `Page int`, `PerPage int`|クエリパラメータ。`binding:"omitempty"`。型のみチェックし、`PerPage`の上限制御はApplication層（`ListAdminsUseCase`）で行う|
|`AdminCreateRequest`|`Name string`, `Email string`|`Email`: `binding:"required,email"`（②「15. Validation設計」の「必須チェック: 作成時の`email`の必須性」）。`Name`は`binding:"omitempty"`（未入力を許容し、氏名の補完は`user` Contextの`CreateAdminAccount`が担う）|
|`AdminUpdateRequest`|`Name string`, `NameKana string`, `AddressID *uint`, `PhoneNumber *string`, `Birthday *string`, `Gender *string`|`Name`: `binding:"required"`（更新時の氏名は必須。②「15. Validation設計」）。`PhoneNumber`: `binding:"omitempty,numeric"`（数字形式チェック）。`Birthday`: `binding:"omitempty,datetime=2006-01-02"`（日付形式チェック）。桁数・未来日付・許容値そのものの検証はDomain層のValue Objectが行う（②「15. Validation設計」の責務分離方針）|
|`AddressSearchRequest`|`PrefectureID uint`, `City string`, `Town string`|`PrefectureID`: `binding:"required"`（②「15. Validation設計」の「必須チェック: ...住所候補検索時のprefecture_idの必須性」に対応）。`City`/`Town`: `binding:"omitempty"`|

### Response（`presentation/response/admin_response.go`）

|struct名|フィールドと型|
|-|-|
|`AdminSummaryResponse`|`ID uint`, `Name string`, `Email string`|
|`AdminListResponse`|`Admins []AdminSummaryResponse`, `Page int`, `PerPage int`, `TotalCount int64`|
|`AdminDetailResponse`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `CreatedAt string`, `UpdatedAt string`, `ActivityLog []any`（Rails現行の`activity_log`。常に空の配列。Handlerのレスポンス変換で空のスライスを設定し、JSONで`[]`となるようにする（`nil`にしない）。UseCase・Domainには持ち込まない）, `AddressID *uint`, `PhoneNumber *string`, `Birthday *string`, `Gender *string`|
|`AddressCandidateResponse`|`ID uint`, `PostalCode string`, `City string`, `Town string`, `Prefecture string`（②「19. API仕様」の「住所候補一覧（id, postal_code, city, town, prefecture）」に対応）|
|`AddressSearchResponse`|`Addresses []AddressCandidateResponse`|

## Routing（`presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/admin/admins`|`AdminHandler.List`|
|GET|`/api/v1/admin/admins/:id`|`AdminHandler.Show`|
|POST|`/api/v1/admin/admins`|`AdminHandler.Create`|
|PATCH|`/api/v1/admin/admins/:id`|`AdminHandler.Update`|
|DELETE|`/api/v1/admin/admins/:id`|`AdminHandler.Delete`|
|GET|`/api/v1/admin/addresses`|`AddressHandler.Search`|

全ルートに、adminロールを要求する認証Middlewareを適用する（②「16. Authorization設計」）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/admin/admins|AdminHandler.List|AdminListRequest|AdminListResponse|200|
|GET|/api/v1/admin/admins/:id|AdminHandler.Show|-（パスパラメータのみ）|AdminDetailResponse|200|
|POST|/api/v1/admin/admins|AdminHandler.Create|AdminCreateRequest|AdminDetailResponse|201|
|PATCH|/api/v1/admin/admins/:id|AdminHandler.Update|AdminUpdateRequest|AdminDetailResponse|200|
|DELETE|/api/v1/admin/admins/:id|AdminHandler.Delete|-（パスパラメータのみ）|-（本文なし）|204|
|GET|/api/v1/admin/addresses|AddressHandler.Search|AddressSearchRequest|AddressSearchResponse|200|

## Errorケース

|Endpoint|条件|Status Code|Error内容|
|-|-|-|-|
|List|-|200|該当なし（不正なページ番号等は上限・下限に丸めて処理し、②に明記のないエラー化は行わない。推測）|
|Show|対象管理者が存在しない、または削除済み|404|`ErrAdminNotFound`|
|Create|`email`未入力・形式不正|422|Request DTOバリデーションエラー|
|Create|`email`が既存アカウント（無効化済みを含む）で使用済み|422|`ErrAdminEmailAlreadyUsed`（`user` Contextの`CreateAdminAccount`が返すValidationエラーの変換）|
|Create|`Name`が100文字を超える等、`user` ContextのValidationエラー|422|`ErrInvalidAdminAccountInput`|
|Update|`Name`未入力|422|Request DTOバリデーションエラー|
|Update|対象管理者が存在しない|404|`ErrAdminNotFound`|
|Update|電話番号が10〜11桁の数字でない|422|`ErrInvalidPhoneNumber`|
|Update|生年月日が未来日付|422|`ErrInvalidBirthday`|
|Update|性別が許容値外|422|`ErrInvalidGender`|
|Delete|対象管理者が存在しない|404|`ErrAdminNotFound`|
|Delete|削除対象が操作者自身|422|`ErrCannotDeleteSelf`（②「19. API仕様」の「自分自身は削除できません」メッセージを踏襲）|
|Delete|削除により有効な管理者が0人になる|422|`ErrLastAdminCannotBeDeleted`（②「19. API仕様」の「最後の管理者は削除できません」メッセージを踏襲）|
|Search|`prefecture_id`未指定|400|`ErrPrefectureIDRequired`（②「19. API仕様」の「400（prefecture_id未指定）」、エラーメッセージ「都道府県は必須です。」を踏襲）|
|全Endpoint共通|予期せぬDB接続失敗等|500|Infrastructure Error|

---

# 8. Transaction実装方針

## Transaction開始箇所

②「14. Transaction設計」により、UseCaseの開始時（読み取り専用UseCaseを除く）にトランザクションを開始する。

> **②からの補足**: 具体的な実装機構は②に明記がない。アーキテクチャ規約「2. レイヤー責務と依存方向」の禁止事項（UseCaseがInfrastructure具象実装に依存しない）を満たすため、認証機能の実装仕様と同一の方式で`TransactionManager`インターフェースを導入する（推測）。

- interface名: `TransactionManager`（定義場所: `application/usecase/transaction_manager.go`、利用側であるapplication層で定義する。コーディング規約「5. インターフェース」準拠）
- メソッドシグネチャ: `WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error`
- 実装: `infrastructure/repository`配下でGORMの`*gorm.DB.Transaction`を用いて実装し、`fn`に渡す`ctx`にトランザクション用の`*gorm.DB`を保持させる

## Transaction終了箇所（Commit / Rollback条件）

|UseCase|終了箇所|
|-|-|
|ListAdminsUseCase / ShowAdminUseCase / SearchAddressCandidatesUseCase|トランザクションを使用しない（読み取りのみ）|
|CreateAdminUseCase|`AdminAccountCreator.Create`・`AdminAccountRepository.FindByID`（再取得）が完了した時点でCommit、いずれかの失敗（`CreateAdminAccount`のValidationエラーを含む）でRollback。`CreateAdminAccount`が登録した招待メールの送信依頼（`jobs`）も、Rollback時は取り消される|
|UpdateAdminUseCase|`AdminAccountRepository.Update`と、個人情報の`AdminPersonalInfoRepository.Update`または`Create`（個人情報の永続化が必要な場合のみ）が完了した時点でCommit、いずれかの失敗でRollback|
|DeleteAdminUseCase|`AdminAccountRepository.CountActive`取得から`AdminAccountRepository.Delete`完了までを終えた時点でCommit、判定違反・失敗時はRollback|

## 複数Repositoryにまたがる場合の扱い

CreateAdminUseCaseは`AdminAccountCreator`（`user` Contextの`CreateAdminAccount`。`users`の作成と招待メールの送信依頼の登録）・`AdminAccountRepository`（再取得）を、UpdateAdminUseCaseは`AdminAccountRepository`と`AdminPersonalInfoRepository`の両方を、1つの`TransactionManager.WithinTransaction`スコープ内で呼び出し、`users`と`user_personal_infos`の不整合な中間状態（②「5. Aggregate設計」）を防ぐ。DeleteAdminUseCaseは`AdminAccountRepository`のみを扱うが、`CountActive`取得と`Delete`実行の間に他の削除操作が割り込むと「最後の管理者」ルールが破られる可能性があるため、同一トランザクション内で実行する（②「14. Transaction設計」の理由）。

> **②からの補足**: 同時実行時の排他制御（例: 行ロック）の具体的な要否・方式は②に明記がない。トランザクション分離レベルまたは明示的ロックの必要性は、実装時にDBの同時実行特性を踏まえて別途確認する必要がある（推測）。

---

# 9. Validation実装方針

## Presentation

- 型チェック: 各Request DTOの`binding`タグ（Ginの`ShouldBind`使用）で型を検証する
- 必須チェック: `AdminCreateRequest.Email`の必須性を`binding:"required"`で検証する（②「15. Validation設計」）
- フォーマットチェック: `email`は`binding:"email"`、`phone_number`は`binding:"numeric"`、`birthday`は`binding:"datetime=2006-01-02"`で形式を検証する

## 業務ルール検証（Domain Model採用のため②の記載どおり）

- Entity／Value Object生成時に検証する内容:
  - `valueobject.NewPhoneNumber`: 電話番号の桁数（10〜11桁）
  - `valueobject.NewBirthday`: 生年月日の未来日付禁止
  - `valueobject.NewGender`: 性別の許容値
  - `entity.AdminAccount.MarkDeleted`: 削除対象が既に削除済みでないかの状態チェック
- `CreateAdminUseCase`が呼ぶ`user` Contextの`CreateAdminAccount`が検証する内容: 氏名の補完（未入力なら メールアドレスの`@`より前）・氏名の文字数・メールアドレスの形式と一意性（②「15. Validation設計」）。本Contextはこれらを再実装しない
- UseCase内で判定する業務ルール:
  - `ListAdminsUseCase`: `PerPage`の正規化（未指定・0以下は既定値25件、100件を上限に丸める）
  - `UpdateAdminUseCase`: 個人情報が未登録で3項目がすべて指定なしの場合は`AdminPersonalInfo`を作成しない
  - `DeleteAdminUseCase`: `AdminDeletionPolicy.Authorize`による「自分自身でないこと」「最後の管理者でないこと」の整合性チェック
  - `SearchAddressCandidatesUseCase`: `prefecture_id`の必須チェック（②「15. Validation設計」の「都道府県必須」という業務ルールの実体はmaster-data Context側が定義するものであり、本Contextはそれをそのまま呼び出す）

---

# 10. Authorization実装方針

②「16. Authorization設計」をそのまま実装レベルに落とし込む。

## Middleware

- Cookie/Header等から認証済みユーザーを特定し、`current_admin`としてGinの`*gin.Context`に保持する
- 役割が"admin"であることを確認する（役割不一致の場合は401または403で拒否。具体的なステータスコードは②に明記がないため、既存の他機能の認証Middleware実装方針に合わせる。推測）

## Handler

- ルーティング層でAPIの入口を担当する
- 具体的な削除可否判定は持たせない（②の方針どおり、`current_admin`のIDを`dto.DeleteAdminCommand.CurrentAdminID`へ橋渡しするのみ）

## UseCase

- `CreateAdminUseCase`は、Middlewareで管理者ロールの認可が済んだ状態で、`user` Contextの`CreateAdminAccount`を呼ぶ。`CreateAdminAccount`は認可を行わない（`user`②「16. Authorization設計」）
- `DeleteAdminUseCase`が、操作者（`cmd.CurrentAdminID`）のIDを削除対象（`cmd.AdminID`）と比較する材料として`AdminDeletionPolicy`へ渡す
- 有効管理者数を`AdminAccountRepository.CountActive`から取得し、`AdminDeletionPolicy`へ渡す

## Domain

- `AdminDeletionPolicy`が「自分自身か」「最後の管理者か」という業務レベルの認可判定を行う
- `AdminAccount`は判定結果を受けて`MarkDeleted`で状態遷移を実行するのみで、判定ロジック自体は保持しない
- `SearchAddressCandidatesUseCase`はadminロールであれば誰でも利用できる補助機能であり、追加の業務レベル認可判定を持たない（②「16. Authorization設計」の判断理由に対応）

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

各UseCaseは、`domain/errors`のセンチネルエラーを`errors.Is`で判定し、必要に応じてApplication層独自のラップ（`fmt.Errorf("...: %w", err)`、コーディング規約「6. エラーのラップ」準拠）を行った上でHandlerへ伝播させる。「対象管理者未存在」「住所候補検索のprefecture_id未指定」は②「17. Error設計」によりApplication Errorとして扱うため、`application/usecase/errors.go`に`ErrAdminNotFound` / `ErrPrefectureIDRequired`をセンチネルエラー変数として定義する（②に配置場所の明記はないため推測）。

## Application Error → HTTPレスポンスへの変換方針

Handler層で`errors.Is`によりエラー種別を判定し、以下のStatus Codeへ変換する。変換ロジックは共通のエラーマッピング関数（`presentation/handler`配下の非公開ヘルパー）に集約し、各Handlerで重複させない。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrAdminNotFound`|Application|404|
|`ErrAdminEmailAlreadyUsed`|Application（`user` ContextのValidationエラーの変換）|422|
|`ErrInvalidAdminAccountInput`|Application（`user` ContextのValidationエラーの変換）|422|
|`ErrCannotDeleteSelf`|Domain|422|
|`ErrLastAdminCannotBeDeleted`|Domain|422|
|`ErrAdminAlreadyDeleted`|Domain|422（通常はGORM標準の論理削除機構により`ErrAdminNotFound`として先に検出される。5章「②からの補足」参照）|
|`ErrInvalidPhoneNumber`|Domain|422|
|`ErrInvalidBirthday`|Domain|422|
|`ErrInvalidGender`|Domain|422|
|`ErrPrefectureIDRequired`|Application|400|
|Request DTOバリデーションエラー|Presentation|422|
|DB接続失敗・クエリ失敗|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

Infrastructure層（Repository実装）で発生したエラーは、`fmt.Errorf`でラップしてApplication層へ伝播させ、Handler層で未分類のエラーとして500に変換する。業務エラー（Domain Error）と技術的障害（Infrastructure Error）を`errors.Is` / `errors.As`で明確に区別する。

---

# 12. GORM / DBクエリ設計

②「20. DB設計方針」により、既存Rails DBをそのまま継続利用し、スキーマ変更は行わない。

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

### `user_roles`テーブルについて

管理者ロールによる絞り込みは、`AdminAccountRepository`の実装が`users`と`user_roles`の結合（`user_roles.name = 'admin'`）で行う（②「21. DB操作仕様」）。`user_roles`に対応するGORMモデルは定義しない。管理者の作成（`users`への書き込み）は`user` Contextが行うため、本ContextでロールIDを解決する必要はない。

### `addresses` / `prefectures`テーブルについて

`addresses`・`prefectures`テーブルはmaster-data Context（共通マスタ参照機能）が正規の所有者であり（共通マスタ参照機能_Go実装仕様書「15. GORM / DBクエリ設計」参照）、本Contextでは対応するGORMモデルを定義しない。`AddressLookup`実装（8章）はmaster-data Contextが公開する`SearchAddresses`関数を呼び出すのみであり、SQLを直接発行しない。

## 主要クエリの条件・ソート・ページネーション方針

|操作|条件|ソート|ページネーション|
|-|-|-|-|
|`AdminAccountRepository.List`|管理者ロール（`user_roles`との結合）＋キーワード指定時`name LIKE ?`または`email LIKE ?`|`created_at`の降順（新しい順）。Rails現行の`AdminsQuery#order_default`と同じ。追加のタイブレーク条件は設けない（Rails現行に合わせる）|`page`/`per_page`（既定値25件・上限100件）によるOFFSET/LIMIT|
|`AdminAccountRepository.FindByID`|`id = ?`かつ管理者ロール（`user_roles`との結合）|不要（一意検索）|不要|
|`AdminAccountRepository.CountActive`|管理者ロール（`user_roles`との結合）かつ`deleted_at IS NULL`|不要|不要|
|`AdminPersonalInfoRepository.FindByUserID`|`user_id = ?`|不要（一意検索）|不要|
|`AdminAccountCreator.Create`|`users`へは書き込まない（`user` Contextの`CreateAdminAccount`の呼び出しに委譲）|不要|不要|
|`AddressLookup.Search`|master-data Contextの`SearchAddresses`関数呼び出しに委譲（`prefecture_id`必須＋`city`/`town`部分一致・`prefectures`とのJoinはmaster-data Context側の実装に従う）|master-data Context側の実装に従う（本Contextでは指定しない）|不要（②「21. DB操作仕様」の「該当件数が少数であることを前提とした一覧返却」）|

## 既存Schemaへの変更

②「20. DB設計方針」により変更なし。SQL文そのものは本書に記載しない。

---

# 13. テストケース設計

②「22. テスト戦略」をDomain Model採用時の区分のまま、具体的なテストケース単位に落とし込む。

## Domain Test

|対象|テストケース|
|-|-|
|`AdminDeletionPolicy.Authorize`|削除対象が操作者自身の場合に`ErrCannotDeleteSelf`を返す／有効管理者数が1人の場合に`ErrLastAdminCannotBeDeleted`を返す／自分自身でなく2人以上有効な場合に成功する（②の重点検証項目）|
|`PhoneNumber.NewPhoneNumber`|10桁・11桁の数字で成功する／9桁以下・12桁以上・数字以外を含む場合に`ErrInvalidPhoneNumber`相当を返す|
|`Birthday.NewBirthday`|基準時刻以前の日付で成功する／基準時刻より未来の日付で`ErrInvalidBirthday`相当を返す|
|`Gender.NewGender`|許容値で成功する／許容値外で`ErrInvalidGender`相当を返す|
|`AdminAccount.MarkDeleted`|未削除状態から削除済みへの遷移が成功する／既に削除済みの状態で呼び出すと`ErrAdminAlreadyDeleted`を返す|

## UseCase Test

|対象|テストケース|
|-|-|
|`ListAdminsUseCase`|キーワード検索が反映される／`PerPage`が100を超える指定時に100件へ丸められる／`PerPage`が未指定・0以下の場合に25件になる|
|`ShowAdminUseCase`|存在する管理者IDで個人情報を含む詳細が返る／存在しない管理者IDで`ErrAdminNotFound`を返す|
|`CreateAdminUseCase`|`AdminAccountCreator.Create`へ氏名（任意）・メールアドレスがそのまま渡される（氏名の補完は`user` Contextが行うため、本UseCaseでは検証しない）／`AdminPersonalInfo`が作成されない（`AdminPersonalInfoRepository.Create`が呼ばれない）／作成後に`AdminAccountRepository.FindByID`で再取得され、`CreatedAt` / `UpdatedAt`を含む結果が返る／`AdminAccountCreator.Create`が`ErrAdminEmailAlreadyUsed`を返した場合に、そのエラーがそのまま返る／再取得できなかった場合に内部エラーになる|
|`UpdateAdminUseCase`|氏名を含む基本情報・個人情報が正しく更新される（既存の個人情報がある場合は3項目が入力値で置き換わる）／個人情報が未登録で3項目がすべて指定なしの場合に`AdminPersonalInfo`が作成されない／個人情報が未登録で3項目のいずれかが指定された場合に`AdminPersonalInfo`が作成される／存在しない管理者IDで`ErrAdminNotFound`を返す／不正な電話番号・生年月日・性別でそれぞれ対応するDomain Errorを返す|
|`DeleteAdminUseCase`|最後の管理者を削除しようとした場合にエラーになる／自分自身を削除しようとした場合にエラーになる／条件を満たす場合に正常に削除される（②の重点検証項目）|
|`SearchAddressCandidatesUseCase`|`prefecture_id`未指定時に`ErrPrefectureIDRequired`を返す／`AddressLookup.Search`へ正しく中継されること（②の「master-data ContextのAddressLookupへ正しく中継されることを検証する」に対応）|

## Repository Test

|対象|テストケース|
|-|-|
|`AdminAccountRepository.List`|キーワード検索・ページング・総件数取得の正確性／`created_at`の降順で並ぶこと|
|`AdminAccountRepository.FindByID`|存在する/しない管理者IDでの検索結果、削除済み管理者が除外されること、`created_at` / `updated_at`が返ること|
|`AdminAccountRepository.CountActive`|有効な管理者のみがカウントされ、削除済みが除外されること|
|`AdminAccountRepository.Delete`|`deleted_at`が正しく反映されること|
|`AdminPersonalInfoRepository.FindByUserID` / `Create` / `Update`|個人情報の作成・更新・NULL許容項目の取り扱いの正確性|
|`AdminAccountCreator.Create`|`user` Contextの`CreateAdminAccount`へ氏名・メールアドレスが正しく渡されること／戻り値が`CreatedAdminAccount`へ変換されること／`user`のメール重複・その他のValidationエラーが`ErrAdminEmailAlreadyUsed` / `ErrInvalidAdminAccountInput`へ変換されること／`user`の内部エラーがラップされて返ること（`CreateAdminAccount`自体の規則は`user` Contextのテストで検証済みとし、本Contextでは呼び出しと変換の検証に限定する）|
|`AddressLookup.Search`|`masterdata.SearchAddresses`への引数（`prefectureID`/`city`/`town`）が正しく渡されること／戻り値（`masterdata.Address`）が`AddressCandidate`へ正しく変換されること（`prefecture_id`絞り込み・`city`/`town`部分一致・都道府県名の結合自体の正確性はmaster-data Context側のInfrastructure関数Testで検証済みのため、本Contextでは変換ロジックの検証に限定する）|

## Handler Test

|対象|テストケース|
|-|-|
|`AdminHandler.List`|クエリパラメータのバインド結果が正しく`ListAdminsUseCase`へ渡される|
|`AdminHandler.Show`|存在しないIDで404を返す／成功時に200を返し、`activity_log`が空の配列（`[]`）で返る|
|`AdminHandler.Create`|`email`未入力・形式不正で422を返す／メール重複（`ErrAdminEmailAlreadyUsed`）で422を返す／成功時に201を返し、`created_at` / `updated_at`を含む管理者詳細を返す（`activity_log`が空の配列であること）|
|`AdminHandler.Update`|氏名未入力・不正な`phone_number`/`birthday`形式で422を返す／成功時に200を返す|
|`AdminHandler.Delete`|最後の管理者・自分自身の削除試行で422を返す／存在しないIDで404を返す／成功時に204を返す|
|`AddressHandler.Search`|`prefecture_id`未指定で400を返す／成功時に200と住所候補一覧を返す|

## Integration Test

|対象|テストケース|
|-|-|
|一覧→詳細→作成→更新→削除の一連フロー|各操作がエンドポイント経由で正常に完了すること|
|管理者の作成（`user` Contextと組み合わせ）|作成された管理者が招待待ちであること／氏名未入力ならメールアドレスの`@`より前が氏名になること／招待メールの送信依頼（`jobs`）が管理者アカウントと同じトランザクションで登録されること／メール重複で失敗した場合はアカウント・招待メールの送信依頼のいずれも残らないこと／`user_personal_infos`にレコードが作成されないこと／レスポンスに`created_at` / `updated_at`が含まれ、`activity_log`が空の配列であること|
|削除の安全弁ルール|APIレベルで、自分自身の削除・最後の管理者の削除がそれぞれ422で拒否されること（②の重点検証項目）|
|住所候補検索エンドポイント|`GET /api/v1/admin/addresses`が期待どおりの絞り込み結果を返すこと（②「22. テスト戦略」の記載どおり）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に列挙する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ディレクトリ名を`internal/admin_account`とした|②のContext名は`admin-account-management`のみで、ディレクトリ名の明記がない。アーキテクチャ規約「4. Bounded Context構成」の命名規則に従い短縮した（認証機能の先例に倣う）|推測|
|`domain/repository/admin_account_creator.go`を新設し、`user` Contextの`CreateAdminAccount`を呼び出す窓口Interfaceとして扱った（実装は`infrastructure/repository/admin_account_creator.go`）|②「3. Bounded Context」「11. Repository設計」が`user` Contextへの依存（管理者アカウントの作成）を定めているが、具体的なInterface名・配置場所までは規定していない。アーキテクチャ規約「5. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従った|推測|
|`UpdateAdminCommand`からメールアドレスの変更項目を除外した|②「12. UseCase設計」がUpdateAdminUseCase入力を抽象的にしか記載しておらず、メールアドレス変更に関する業務ルールの記載が②のどこにもないため|推測（実装時に②側での再確認が必要）|
|Genderの具体的な許容値（列挙値）を確定しなかった|②「7. Value Object設計」はRails `UserPersonalInfo.genders`への準拠を示すのみで具体的な値を記載していない。①（Rails実装詳細）が本タスクでは未提供のため、値を確定できない|①未提供のため参照不可|
|`Birthday`の生成時に基準時刻（`now`）を引数として明示的に受け取る構成とした|②は「未来日付を許容しない」というルールのみを定め、テスト容易性のための基準時刻の扱いは規定していない。認証機能の`ResetTokenValidityPeriod.IsExpired`と同様の構成とした|推測|
|`TransactionManager`インターフェースによるトランザクション制御方式を導入した|②「14. Transaction設計」はトランザクション境界（開始・終了タイミング）のみを定めており、UseCaseがGORMに直接依存しないための実装機構は指定していない。アーキテクチャ規約「2. レイヤー責務と依存方向」の禁止事項を満たすための判断。認証機能の実装仕様と同一方式とした|推測|
|CreateAdminUseCaseにおいて、`CreateAdminAccount`の呼び出し・作成結果の再取得を1トランザクションに含めた|②「14. Transaction設計」の記載に従い、作成時も`users`・招待メールの送信依頼（`jobs`。`user` Contextが登録）を1トランザクションとした。呼び出し側トランザクションの引き継ぎ方式は`user`②の未解決の論点9であり、`user`の③の確定に合わせる|②の記載どおり（引き継ぎ方式は`user`③の確定待ち）|
|`user` Contextが返すValidationエラーのうち、メール重複を`ErrAdminEmailAlreadyUsed`、その他（メール形式不正・氏名の文字数超過）を`ErrInvalidAdminAccountInput`へ変換する構成とした|②「17. Error設計」が、`user`のValidationエラーを本Contextのエラーとして返すことを定めている。`user`②は、メール重複とその他のValidationエラーの区別方法をGoの型としては定めていないため、`user`の③で確定した区別方法に合わせる|推測（`user`③の確定待ち）|
|作成直後の再取得のため、AdminAccountに`CreatedAt` / `UpdatedAt`を持たせ、管理者詳細のDTO・レスポンスに`created_at` / `updated_at`を含めた|Rails現行の管理者詳細（`Admin::AdminDetailSerializer`）が`created_at` / `updated_at`を含み、作成・詳細・更新のレスポンスで共通に返す。`user`②の作成結果にはこれらが含まれないため、再取得で補う|Rails現行の挙動の反映（推測ではない）|
|更新時（PATCH）の氏名を必須とした（`AdminUpdateRequest.Name`が`binding:"required"`）|Rails現行の`Admin::AdminForm`が、更新時に氏名を必須とする。氏名の補完ルールは`user` Contextに移ったため、更新時に補完で代替せず、入力の必須で扱う|Rails現行の挙動の反映（推測ではない）|
|「対象管理者未存在」を表す`ErrAdminNotFound`を`application/usecase/errors.go`に配置した|②「17. Error設計」がこれをApplication Errorとして明記しているが、具体的な配置ファイルは指定していない|推測|
|GORM標準の論理削除機構（`gorm.DeletedAt`型フィールド）を用いて`deleted_at`を扱う構成とした|②・Gorm規約のいずれも具体的な実装機構を指定していないが、②が前提とする`deleted_at`による論理削除仕様を実現する一般的なGORM機能であるため採用した|推測|
|一覧取得の並び順を`created_at`の降順、`per_page`の既定値を25件とした|②「11. Repository設計」「19. API仕様」に記載のとおり、Rails現行（`AdminsQuery#order_default`、`AdminsController`の`DEFAULT_PER_PAGE`）に合わせる|Rails現行の挙動の反映（推測ではない）|
|`AdminDetailResponse`に`activity_log`（常に空の配列）を含めた|②「19. API仕様」のとおり、Rails現行の`Admin::AdminDetailSerializer`が常に空の配列を返す`activity_log`を持つ。履歴機能は設計しないため、Handlerのレスポンス変換で空のスライスを設定するのみとし、Application・Domain層には持ち込まない|Rails現行の挙動の反映（推測ではない）|
|管理者作成（`POST`）成功時のHTTP Status Codeを201とした|②「19. API仕様」のPOSTの「Status Code」が201であり、Rails現行の`Api::V1::Admin::AdminsController#create`が`status: :created`を返すため|②の明記どおり（推測ではない）|
|`AddressLookup`の実装（`infrastructure/repository/address_lookup.go`）を、master-data Context（共通マスタ参照機能）が公開する`SearchAddresses`関数呼び出しとした|本Context自身は`AddressLookup`という参照専用Interfaceを定義し（Domain Model採用のための依存性逆転）、その実装は本タスクで並行して確定した共通マスタ参照機能_Go実装仕様書側の`SearchAddresses`関数シグネチャに合わせて構成した。アーキテクチャ規約「5. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針どおりであり、`addresses`/`prefectures`テーブルへの直接クエリは行わない|②からの補足（③間の整合を取るための具体化であり、新しい業務ルールの追加ではない）|
|`SearchAddressCandidatesUseCase`が`prefecture_id`必須チェックをガード節として持つ構成とした|②「12. UseCase設計」は本UseCaseを「master-data Contextへの問い合わせを管理者向けに中継するのみ」とする一方、②「15. Validation設計」はPresentation層での必須チェックも明記しており、両者を矛盾なく実装するためPresentation（型チェック）とApplication（業務的な必須判定）の二重チェックとした|②の記載同士を整合させるための判断（新しい業務ルールの追加ではない）|

上記以外の設計判断（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API仕様・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
