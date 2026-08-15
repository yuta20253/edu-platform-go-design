# 認証機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

ユーザーのログイン・ログアウト・新規登録（student/teacher/admin）・パスワードリセットを提供し、認証状態（セッションの有効性を表す `jti`、パスワードリセットトークンの有効性）を管理する機能である。JWTをHTTP Only CookieでクライアントAPI互換性を維持したまま提供する。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により、本機能は **Domain Model** を採用する。

- Account（資格情報・`jti`）、PasswordResetToken（発行・有効・期限切れ・消費済み）という状態遷移を持つ概念が中心にあること
- パスワード照合・ロール別登録要件・リセットトークン有効期限判定という複数の業務ルールが複数UseCaseにまたがって再利用されること
- セキュリティ上重要なロジックを型・サービスとして明示し、レビュー可能性とテスト容易性を高める必要があること

上記の理由からTransaction Script・Active Record・Event Sourcingは採用せず、Domain Modelを採用している（詳細は②「4. 設計パターン」参照）。

## 本書が対象とする実装範囲

本書は、②で確定した設計（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略）を変更せず、Goでの具体的なコード構成（package構成・struct定義・interfaceメソッドシグネチャ・クエリ内容）に落とし込むことを目的とする。

規約`アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」の「Domain Model」節に従い、`{context}/domain`・`application`・`infrastructure`・`presentation`のフルレイヤー構成を適用する。

①Rails実装詳細は本タスクでは提供されていないため、①の実装コードそのものを根拠とする記載は行わない（①未提供のため参照不可）。②に明記された「Rails現行仕様の要約」の範囲でのみ言及する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名（②）: `authentication`
- ディレクトリ名: `internal/auth`

  > **②からの補足**: ②にはディレクトリ名の明記がない。アーキテクチャ規約「5. Bounded Context構成 命名規則」に従い、Context名`authentication`を英単語1語のディレクトリ名に短縮したものであり、他機能（`internal/task`, `internal/school`等）の命名慣習に合わせた実装判断である（推測）。

## ②で採用した設計パターン

Domain Model

## 作成するディレクトリ一覧

```
internal/auth/
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
│   ├── security/
│   └── mail/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

`domain/specification/`・`infrastructure/cache/`・`infrastructure/queue/`は本機能では対象外（②に該当する業務ルール・キャッシュ要件・独自の非同期実行基盤の必要性の記載がないため）。

> **②からの補足**: `infrastructure/security/`は規約の標準ディレクトリ例（`persistence/gorm`, `repository`, `mail`, `cache`, `queue`）には含まれない。②「Railsとの責務対応」で「Devise + warden-jwt → Infrastructure（トークン発行アダプタ）」と記載されているため、パスワードハッシュ照合・JWT発行という技術アダプタの置き場所として新設した（推測。②に具体的なディレクトリ名の指定はない）。

## 作成するファイル一覧

```
internal/auth/domain/entity/account.go
internal/auth/domain/entity/password_reset_token.go

internal/auth/domain/valueobject/email.go
internal/auth/domain/valueobject/raw_password.go
internal/auth/domain/valueobject/reset_token_validity_period.go
internal/auth/domain/valueobject/signup_role_requirement.go

internal/auth/domain/repository/account_repository.go
internal/auth/domain/repository/user_role_repository.go
internal/auth/domain/repository/high_school_repository.go
internal/auth/domain/repository/grade_repository.go

internal/auth/domain/service/credential_verification_service.go
internal/auth/domain/service/password_hasher.go
internal/auth/domain/service/registration_eligibility_policy.go
internal/auth/domain/service/password_reset_lifecycle_policy.go

internal/auth/domain/event/password_reset_token_issued.go

internal/auth/domain/errors/errors.go

internal/auth/application/dto/login_dto.go
internal/auth/application/dto/logout_dto.go
internal/auth/application/dto/register_dto.go
internal/auth/application/dto/password_reset_dto.go

internal/auth/application/usecase/login_usecase.go
internal/auth/application/usecase/logout_usecase.go
internal/auth/application/usecase/register_usecase.go
internal/auth/application/usecase/request_password_reset_usecase.go
internal/auth/application/usecase/change_password_usecase.go
internal/auth/application/usecase/verify_reset_token_usecase.go

internal/auth/infrastructure/persistence/gorm/user_model.go

internal/auth/infrastructure/repository/account_repository.go
internal/auth/infrastructure/repository/user_role_repository.go
internal/auth/infrastructure/repository/high_school_repository.go
internal/auth/infrastructure/repository/grade_repository.go

internal/auth/infrastructure/security/bcrypt_password_hasher.go
internal/auth/infrastructure/security/jwt_token_issuer.go
internal/auth/infrastructure/security/secure_token_generator.go

internal/auth/infrastructure/mail/password_reset_event_handler.go

internal/auth/presentation/handler/session_handler.go
internal/auth/presentation/handler/registration_handler.go
internal/auth/presentation/handler/password_reset_handler.go

internal/auth/presentation/request/session_request.go
internal/auth/presentation/request/registration_request.go
internal/auth/presentation/request/password_reset_request.go

internal/auth/presentation/response/session_response.go
internal/auth/presentation/response/registration_response.go
internal/auth/presentation/response/message_response.go

internal/auth/presentation/routes.go
```

---

# 3. Domain層設計

## Entity

### Account（`domain/entity/account.go`）

- struct名: `Account`
- フィールド:

|フィールド|型|意味|
|-|-|-|
|`ID`|`uint`|アカウント（`users`テーブル行）の識別子|
|`Email`|`valueobject.Email`|ログインに用いるメールアドレス|
|`PasswordHash`|`string`|保存済みパスワードハッシュ（ハッシュアルゴリズム自体はInfrastructure層`PasswordHasher`が担う）|
|`UserRoleID`|`uint`|ロール（student/teacher/admin）を示す識別子|
|`JTI`|`string`|現在有効なセッション識別子|
|`HighSchoolID`|`*uint`|所属高校ID（student/teacherのみ設定、adminはnil）|
|`GradeID`|`*uint`|学年ID（student/teacherのみ設定、adminはnil）|
|`ResetToken`|`*entity.PasswordResetToken`|発行中のパスワードリセットトークン（未発行時はnil）|

- 公開メソッド一覧（引数・戻り値のみ。ロジックは記述しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewAccount`|`(id uint, email valueobject.Email, passwordHash string, userRoleID uint, jti string, highSchoolID, gradeID *uint) (*Account, error)`|`(*Account, error)`|不変条件を満たしたAccountを生成するファクトリ|
|`RotateJTI`|`(newJTI string)`|`error`|ログアウト時に`jti`をローテーションし、以前のセッションを無効化する|
|`UpdatePasswordHash`|`(newHash string)`|`error`|パスワードリセット成功時にハッシュを更新する|
|`IssuePasswordResetToken`|`(token string, issuedAt time.Time)`|`error`|リセットトークンを発行し保持する|
|`ConsumePasswordResetToken`|`()`|`error`|リセットトークンを消費済み（nil化）にする|
|`HasResetToken`|`()`|`bool`|リセットトークンが発行済みかを判定する|

- 不変条件（ファクトリで保証する内容）:
  - `Email`は`valueobject.Email`型としてのみ保持され、生成時に形式検証済みであることが保証される
  - `PasswordHash`は空文字を許容しない
  - `UserRoleID`は0を許容しない（登録時にUserRoleRepositoryで存在確認済みの値のみを渡す）

> **②からの補足**: ②「5. Aggregate設計」はAccountのAggregate境界から「プロフィール情報（氏名・個人情報・住所等）」を明示的に除外している。一方、RegisterUseCaseの入力には`name`, `name_kana`が含まれる（②「10. UseCase設計」）。本書では、Account Entity自体は認証関連フィールドのみを保持し、`name`/`name_kana`はEntity生成時の状態としては持たせず、Repository層への作成入力（`CreateAccountParams`、後述）としてのみ受け渡す設計とする。これは②のAggregate境界方針をそのままGo実装に反映した結果であり、②に明記のない実装判断のため補足する（推測ではなく、②の記載同士の整合を取るための判断）。

### PasswordResetToken（`domain/entity/password_reset_token.go`）

- struct名: `PasswordResetToken`
- フィールド:

|フィールド|型|意味|
|-|-|-|
|`Token`|`string`|発行されたリセットトークン文字列|
|`IssuedAt`|`time.Time`|トークン発行日時|

- 公開メソッド一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewPasswordResetToken`|`(token string, issuedAt time.Time) (*PasswordResetToken, error)`|`(*PasswordResetToken, error)`|発行直後のトークンを生成するファクトリ|
|`IsValid`|`(now time.Time, period valueobject.ResetTokenValidityPeriod) bool`|`bool`|現在有効（期限内）かどうかを判定する|

- 不変条件: `Token`は空文字を許容しない。

## Value Object

### Email（`domain/valueobject/email.go`）

- struct名: `Email`
- フィールド: `value string`（非公開）
- 生成時に検証するルール: メールアドレス形式であること
- 公開メソッド: `NewEmail(raw string) (Email, error)` / `(e Email) String() string`

### RawPassword（`domain/valueobject/raw_password.go`）

- struct名: `RawPassword`
- フィールド: `value string`（非公開）
- 生成時に検証するルール: 登録時の最小文字数等のポリシー（②「7. Value Object設計」により、具体的な文字数はRails現行のDevise標準ポリシーを踏襲する前提。②で「推測」と明記されているため本書でも数値を確定しない）
- 公開メソッド: `NewRawPassword(raw string) (RawPassword, error)` / `(p RawPassword) Matches(confirmation RawPassword) bool` / `(p RawPassword) String() string`

> **②からの補足**: 具体的な最小文字数は②でも「推測」とされ確定していない。本書でも数値は確定せず、実装時にRails現行DBのバリデーション実態を別途確認する必要がある旨を明記する（①未提供のため参照不可）。

### ResetTokenValidityPeriod（`domain/valueobject/reset_token_validity_period.go`）

- struct名: `ResetTokenValidityPeriod`
- フィールド: `duration time.Duration`（非公開）
- 生成時に検証するルール: 発行からの有効期間（②により、Devise標準設定を踏襲する前提、具体的な期間は②でも「推測」）
- 公開メソッド: `NewResetTokenValidityPeriod(d time.Duration) ResetTokenValidityPeriod` / `(p ResetTokenValidityPeriod) IsExpired(issuedAt, now time.Time) bool`

> **②からの補足**: 具体的な有効期間の値は②でも未確定（推測）。本書では型・判定メソッドのシグネチャのみを定義し、具体的な時間値は実装時にRails現行設定（`config.reset_password_within`相当）を①側で確認のうえ確定する必要がある（①未提供のため参照不可）。

### SignUpRoleRequirement（`domain/valueobject/signup_role_requirement.go`）

- struct名: `SignUpRoleRequirement`
- フィールド: `roleName string`（非公開）
- 生成時に検証するルール: ロール名（`student`/`teacher`/`admin`）に応じて高校ID・学年IDが必須かを判定するルールを保持する
- 公開メソッド: `NewSignUpRoleRequirement(roleName string) SignUpRoleRequirement` / `(r SignUpRoleRequirement) RequiresHighSchool() bool` / `(r SignUpRoleRequirement) RequiresGrade() bool`

## Value Objectを採用しないもの

②「7. Value Object設計」の方針どおり、`jti`は単なるUUID文字列として`Account.JTI string`にそのまま保持し、Value Object化しない。

## Repository Interface

### AccountRepository（`domain/repository/account_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByEmail`|`(ctx context.Context, email valueobject.Email)`|`(*entity.Account, error)`|メールアドレスによる一意検索（ログイン・パスワードリセットリクエスト時）|
|`FindByResetToken`|`(ctx context.Context, token string)`|`(*entity.Account, error)`|リセットトークンによる検索（トークン検証・パスワード更新時）|
|`Create`|`(ctx context.Context, params CreateAccountParams)`|`(*entity.Account, error)`|アカウントの新規作成（登録時）。`CreateAccountParams`にName/NameKanaを含む|
|`UpdateJTI`|`(ctx context.Context, accountID uint, newJTI string)`|`error`|`jti`の更新（ログアウト時）|
|`SaveResetToken`|`(ctx context.Context, accountID uint, token string, issuedAt time.Time)`|`error`|リセットトークン発行情報の保存|
|`UpdatePasswordAndConsumeResetToken`|`(ctx context.Context, accountID uint, newPasswordHash string)`|`error`|パスワードハッシュ更新とリセットトークン消費（クリア）を1操作で行う|

`CreateAccountParams`のフィールド: `Email valueobject.Email`, `PasswordHash string`, `UserRoleID uint`, `HighSchoolID *uint`, `GradeID *uint`, `Name string`, `NameKana string`

### UserRoleRepository（`domain/repository/user_role_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByName`|`(ctx context.Context, name string)`|`(id uint, found bool, err error)`|ロール名による存在確認とID取得|

### HighSchoolRepository（`domain/repository/high_school_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`Exists`|`(ctx context.Context, id uint)`|`(bool, error)`|登録時に指定された`high_school_id`の存在確認（参照専用）|

### GradeRepository（`domain/repository/grade_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`Exists`|`(ctx context.Context, id uint)`|`(bool, error)`|登録時に指定された`grade_id`の存在確認（参照専用）|

## Domain Service

### CredentialVerificationService（`domain/service/credential_verification_service.go`）

- struct名: `CredentialVerificationService`
- 依存: `PasswordHasher`（同ファイルまたは`password_hasher.go`で定義するinterface）
- メソッドシグネチャ:
  - `NewCredentialVerificationService(hasher PasswordHasher) *CredentialVerificationService`
  - `(s *CredentialVerificationService) Verify(account *entity.Account, raw valueobject.RawPassword) error`
- 責務: 入力された`RawPassword`とAccountのパスワードハッシュを照合し、失敗時は`domain/errors.ErrInvalidCredentials`相当を返す

### PasswordHasher（`domain/service/password_hasher.go`）

> **②からの補足**: ②「Railsとの責務対応」の「Infrastructure（トークン発行アダプタ）」記載を根拠に、コーディング規約「5. インターフェース（利用側で定義する）」に従い、CredentialVerificationServiceの利用側であるdomain/serviceパッケージにinterfaceを定義した（推測。②に配置場所の明記はない）。

- interface名: `PasswordHasher`
- メソッドシグネチャ:
  - `Hash(raw string) (string, error)`
  - `Verify(hash, raw string) (bool, error)`

### RegistrationEligibilityPolicy（`domain/service/registration_eligibility_policy.go`）

- struct名: `RegistrationEligibilityPolicy`
- メソッドシグネチャ: `(p RegistrationEligibilityPolicy) Validate(requirement valueobject.SignUpRoleRequirement, highSchoolID, gradeID *uint, highSchoolExists, gradeExists bool) error`
- 責務: ロール別要件（学生・教員は高校・学年必須、管理者は不要）を満たしているかを判定する

### PasswordResetLifecyclePolicy（`domain/service/password_reset_lifecycle_policy.go`）

- struct名: `PasswordResetLifecyclePolicy`
- メソッドシグネチャ: `(p PasswordResetLifecyclePolicy) CanConsume(token *entity.PasswordResetToken, now time.Time, period valueobject.ResetTokenValidityPeriod) error`
- 責務: リセットトークンが現在の状態から見て消費（パスワード更新）可能かを判定する。トークン不一致・未発行・期限切れをそれぞれ区別したDomain Errorを返す

## Domain Event

### PasswordResetTokenIssued（`domain/event/password_reset_token_issued.go`）

- イベントstruct名: `PasswordResetTokenIssued`
- 保持するフィールド: `AccountID uint`, `Email string`, `Token string`, `IssuedAt time.Time`
- 発火元: `RequestPasswordResetUseCase`（対象ユーザーが存在しトークンが発行された時点）

> **②からの補足**: イベントのディスパッチ機構（同期呼び出し／Goチャネル／外部キューへのpublish等）は②に明記がない。本書ではイベントstructの定義のみを示し、実際の購読・非同期送信の実装方式（`infrastructure/mail/password_reset_event_handler.go`がどうトリガーされるか）はUseCase側からハンドラを直接呼び出す構成（後述4章）とする。将来的にイベントバスを導入する場合も、このイベントstructをそのまま利用できる（推測）。

## Domain Error

`domain/errors/errors.go`に、`errors.New`によるセンチネルエラー変数として定義する（②「14. Error設計」のDomain Errorに対応）。

|変数名|発生条件|
|-|-|
|`ErrInvalidCredentials`|メールアドレスまたはパスワードが不一致（ユーザー不存在の場合も同一エラーとして扱う）|
|`ErrInvalidRole`|指定されたロール名が存在しない|
|`ErrRegistrationRequirementNotMet`|ロール別必須項目（高校・学年）の欠如、または指定IDが実在しない|
|`ErrPasswordConfirmationMismatch`|パスワードと確認用パスワードが一致しない|
|`ErrResetTokenNotFound`|リセットトークンに一致するアカウントが存在しない|
|`ErrResetTokenExpired`|リセットトークンが有効期限切れ|
|`ErrResetTokenNotIssued`|リセットトークンが発行されていない状態での消費操作|

---

# 4. Application層設計

## DTO（Command / Query）

|struct名|フィールド|区分|
|-|-|-|
|`LoginQuery`|`Email string`, `Password string`|Query|
|`LoginResult`|`AccountID uint`, `Email string`, `UserRoleID uint`, `Token string`, `ExpiresAt time.Time`|出力|
|`LogoutCommand`|`AccountID uint`|Command|
|`LogoutResult`|`Message string`|出力|
|`RegisterCommand`|`Email string`, `Password string`, `PasswordConfirmation string`, `Name string`, `NameKana string`, `RoleName string`, `HighSchoolID *uint`, `GradeID *uint`|Command|
|`RegisterResult`|`AccountID uint`, `Email string`, `UserRoleID uint`, `HighSchoolID *uint`, `GradeID *uint`|出力|
|`RequestPasswordResetCommand`|`Email string`|Command|
|`RequestPasswordResetResult`|`Message string`|出力（常に固定の成功メッセージ）|
|`ChangePasswordCommand`|`ResetPasswordToken string`, `Password string`, `PasswordConfirmation string`|Command|
|`ChangePasswordResult`|`Message string`|出力|
|`VerifyResetTokenQuery`|`ResetPasswordToken string`|Query|
|`VerifyResetTokenResult`|`Valid bool`|出力|

## UseCase

### LoginUseCase（`application/usecase/login_usecase.go`）

- struct名: `LoginUseCase`
- コンストラクタが受け取る依存: `AccountRepository`（Interface）, `*service.CredentialVerificationService`, `TokenIssuer`（本ファイル内で定義するInterface。後述の②からの補足参照）
- 公開メソッド: `(u *LoginUseCase) Execute(ctx context.Context, query dto.LoginQuery) (dto.LoginResult, error)`
- 処理ステップ:
  1. `valueobject.NewEmail` / `NewRawPassword`で入力を検証する
  2. `AccountRepository.FindByEmail`で対象アカウントを取得する（存在しない場合も次のステップで同一エラーに集約する）
  3. `CredentialVerificationService.Verify`で照合する
  4. `TokenIssuer.Issue`でJWTを発行する
  5. `LoginResult`を組み立てて返す
- トランザクション境界: なし（②「11. Transaction設計」により読み取りのみ）
- 発生しうるApplication Error: `ErrInvalidCredentials`（アカウント不存在・パスワード不一致のいずれも同一エラーとして扱い、ユーザー列挙を防ぐ）

> **②からの補足**: `TokenIssuer`はJWT発行という技術的関心事のためのInterfaceであり、②「Railsとの責務対応」の「Devise + warden-jwt → Infrastructure（トークン発行アダプタ）」を根拠に、利用側であるLoginUseCase側（application層）で定義する（コーディング規約「5. インターフェース」準拠）。メソッドシグネチャ: `Issue(ctx context.Context, accountID uint, jti string, userRoleID uint) (token string, expiresAt time.Time, err error)`。実装は`infrastructure/security/jwt_token_issuer.go`に置く。

### LogoutUseCase（`application/usecase/logout_usecase.go`）

- struct名: `LogoutUseCase`
- コンストラクタが受け取る依存: `AccountRepository`, `JTIGenerator`（Interface、後述）, `TransactionManager`（後述8章）
- 公開メソッド: `(u *LogoutUseCase) Execute(ctx context.Context, cmd dto.LogoutCommand) (dto.LogoutResult, error)`
- 処理ステップ:
  1. `JTIGenerator.Generate`で新しい`jti`を生成する
  2. `TransactionManager`内で`AccountRepository.UpdateJTI`を実行する
  3. `LogoutResult`を返す
- トランザクション境界: `UpdateJTI`実行を1トランザクションとする（②「11. Transaction設計」）
- 発生しうるApplication Error: Infrastructure Error（DB更新失敗）

### RegisterUseCase（`application/usecase/register_usecase.go`）

- struct名: `RegisterUseCase`
- コンストラクタが受け取る依存: `UserRoleRepository`, `HighSchoolRepository`, `GradeRepository`, `AccountRepository`, `*service.RegistrationEligibilityPolicy`, `PasswordHasher`, `TransactionManager`
- 公開メソッド: `(u *RegisterUseCase) Execute(ctx context.Context, cmd dto.RegisterCommand) (dto.RegisterResult, error)`
- 処理ステップ:
  1. `valueobject.NewEmail` / `NewRawPassword`で入力形式を検証し、`RawPassword.Matches`でパスワード確認一致を検証する
  2. `valueobject.NewSignUpRoleRequirement(cmd.RoleName)`でロール別要件を取得する
  3. `UserRoleRepository.FindByName`でロールの存在確認・ID取得を行う
  4. 必要に応じて`HighSchoolRepository.Exists` / `GradeRepository.Exists`を呼び出す
  5. `RegistrationEligibilityPolicy.Validate`で登録要件充足を判定する
  6. `PasswordHasher.Hash`でパスワードをハッシュ化する
  7. `TransactionManager`内で`AccountRepository.Create`を実行する
  8. `RegisterResult`を返す
- トランザクション境界: ロール・高校・学年の存在確認からアカウント作成完了までを1トランザクションとする（②「11. Transaction設計」）
- 発生しうるApplication Error: `ErrInvalidRole`, `ErrRegistrationRequirementNotMet`, `ErrPasswordConfirmationMismatch`

### RequestPasswordResetUseCase（`application/usecase/request_password_reset_usecase.go`）

- struct名: `RequestPasswordResetUseCase`
- コンストラクタが受け取る依存: `AccountRepository`, `ResetTokenGenerator`（Interface、後述）, `*service.PasswordResetLifecyclePolicy`不使用（発行時は不要）, `TransactionManager`, `PasswordResetEventPublisher`（Interface、後述）
- 公開メソッド: `(u *RequestPasswordResetUseCase) Execute(ctx context.Context, cmd dto.RequestPasswordResetCommand) (dto.RequestPasswordResetResult, error)`
- 処理ステップ:
  1. `valueobject.NewEmail`で入力形式を検証する
  2. `AccountRepository.FindByEmail`で対象アカウントを検索する
  3. 存在する場合のみ、`ResetTokenGenerator.Generate`でトークンを生成し、`TransactionManager`内で`AccountRepository.SaveResetToken`を実行する
  4. 存在する場合、`PasswordResetEventPublisher.Publish`で`event.PasswordResetTokenIssued`を発行する
  5. 存在有無に関わらず、常に同一の`RequestPasswordResetResult{Message: "..."}`を返す
- トランザクション境界: ユーザーが存在する場合のみ、トークン発行を1トランザクションとする（②「11. Transaction設計」）
- 発生しうるApplication Error: なし（②の方針により、内部的な失敗理由は外部に露出させず常に成功として扱う。Infrastructure Errorのみ5xxとして伝播する）

> **②からの補足**: `ResetTokenGenerator`はセキュリティ用途の乱数生成であり、コーディング規約「17. crypto/rand」に従い`crypto/rand`ベースで実装する（`infrastructure/security/secure_token_generator.go`）。②にはこのInterface名の明記はないため、命名は本書での実装判断である（推測）。`PasswordResetEventPublisher`は`event.PasswordResetTokenIssued`を`infrastructure/mail/password_reset_event_handler.go`に受け渡すためのInterfaceで、メソッドシグネチャは`Publish(ctx context.Context, event event.PasswordResetTokenIssued) error`。

### ChangePasswordUseCase（`application/usecase/change_password_usecase.go`）

- struct名: `ChangePasswordUseCase`
- コンストラクタが受け取る依存: `AccountRepository`, `*service.PasswordResetLifecyclePolicy`, `PasswordHasher`, `TransactionManager`
- 公開メソッド: `(u *ChangePasswordUseCase) Execute(ctx context.Context, cmd dto.ChangePasswordCommand) (dto.ChangePasswordResult, error)`
- 処理ステップ:
  1. `valueobject.NewRawPassword`で入力を検証し、確認用パスワードとの一致を検証する
  2. `AccountRepository.FindByResetToken`で対象アカウントを検索する
  3. `PasswordResetLifecyclePolicy.CanConsume`でトークンの有効性を判定する
  4. `PasswordHasher.Hash`で新パスワードをハッシュ化する
  5. `TransactionManager`内で`AccountRepository.UpdatePasswordAndConsumeResetToken`を実行する
  6. `ChangePasswordResult`を返す
- トランザクション境界: トークン検証からパスワード更新・トークン消費までを1トランザクションとする（②「11. Transaction設計」）
- 発生しうるApplication Error: `ErrResetTokenNotFound`, `ErrResetTokenExpired`, `ErrResetTokenNotIssued`, `ErrPasswordConfirmationMismatch`

### VerifyResetTokenUseCase（`application/usecase/verify_reset_token_usecase.go`）

- struct名: `VerifyResetTokenUseCase`
- コンストラクタが受け取る依存: `AccountRepository`, `*service.PasswordResetLifecyclePolicy`
- 公開メソッド: `(u *VerifyResetTokenUseCase) Execute(ctx context.Context, query dto.VerifyResetTokenQuery) (dto.VerifyResetTokenResult, error)`
- 処理ステップ:
  1. `AccountRepository.FindByResetToken`で対象アカウントを検索する
  2. 見つからない場合は`Valid: false`を返す
  3. 見つかった場合、`PasswordResetLifecyclePolicy.CanConsume`相当の読み取り専用判定を行う
  4. `VerifyResetTokenResult`を返す
- トランザクション境界: なし（読み取りのみ）
- 発生しうるApplication Error: なし（不正・期限切れは`Valid: false`として結果に含める）

---

# 5. Infrastructure層設計

## Repository実装

### AccountRepositoryImpl（`infrastructure/repository/account_repository.go`）

- 実装struct名: `AccountRepositoryImpl`（package名`gormrepo`等、規約「9. 命名規約」により`Impl`接尾辞を付けずpackage名で区別する方針に従うなら`account_repository.go`内structは`accountRepository`のように非公開にしてもよい。本書では規約9章「実装側は`〇〇RepositoryImpl`のような接尾辞を付けない」に従い、非公開struct + コンストラクタ`NewAccountRepository`で公開する構成とする）
- 対応するGORMモデル: `gormmodel.UserModel`（`infrastructure/persistence/gorm/user_model.go`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`FindByEmail`|`email = ?`で1件検索|`RecordNotFound`は`domain/errors.ErrInvalidCredentials`相当に変換せず、そのまま「見つからない」ことをUseCase側の判断に委ねる（呼び出し元でエラー種別を決定）|
|`FindByResetToken`|`reset_password_token = ?`で1件検索|
|`Create`|`users`テーブルへ1件INSERT相当の作成|`CreateAccountParams`の全フィールドを反映する|
|`UpdateJTI`|`id = ?`条件で`jti`カラムのみ更新|
|`SaveResetToken`|`id = ?`条件で`reset_password_token`, `reset_password_sent_at`を更新|
|`UpdatePasswordAndConsumeResetToken`|`id = ?`条件で`encrypted_password`を更新し、同時に`reset_password_token`, `reset_password_sent_at`をNULLへ更新|

- Entity ⇔ GORMモデルの変換方針: `gormmodel.UserModel`から`entity.Account` / `entity.PasswordResetToken`への変換関数（`toEntity`）、逆方向の変換関数（`toModel`）をrepository実装内の非公開関数として用意する。`Email`変換時は`valueobject.NewEmail`を通し、DBに不正な形式が入っていた場合はInfrastructure Errorとして扱う。

### UserRoleRepositoryImpl / HighSchoolRepositoryImpl / GradeRepositoryImpl

- いずれも対応するGORMモデル（`gormmodel.UserRoleModel` / `gormmodel.HighSchoolModel` / `gormmodel.GradeModel`。ただし後二者は School Context側の既存モデルを参照専用で利用する想定）に対し、`id = ?`（または`name = ?`）による存在確認クエリのみを発行する。業務ルール判定は持たない（②「9. Repository設計」の方針どおり）。

> **②からの補足**: HighSchoolModel/GradeModelの具体的な定義場所（School Contextのモデルをそのまま参照するか、認証コンテキスト側に読み取り専用の複製定義を置くか）は②に明記がない。アーキテクチャ規約「6. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従い、School Context側が公開するRepository参照系メソッドを利用することを原則とし、詳細な結線方法はSchool Context側の③文書と合わせて確定する（推測）。

## 外部連携実装

### 認証技術アダプタ（②からの補足）

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`BcryptPasswordHasher`（`infrastructure/security/bcrypt_password_hasher.go`）|`domain/service.CredentialVerificationService`、`RegisterUseCase`、`ChangePasswordUseCase`|`domain/service.PasswordHasher`インターフェースを実装する。ハッシュアルゴリズムの選定（bcrypt等）は①未提供のため参照不可。既存ユーザーの再ログインに影響しないよう、Rails現行のハッシュ方式と一致させる必要がある旨を②「設計差分管理」が既に指摘しており、本書ではその制約を踏襲する|
|`JWTTokenIssuer`（`infrastructure/security/jwt_token_issuer.go`）|`LoginUseCase`|`application/usecase.TokenIssuer`インターフェースを実装する。JWTのクレーム（`sub`, `jti`, `role`等）・署名鍵管理は②「16. API互換方針」のCookie仕様（`access_token`, `httponly`, `secure`, `same_site: lax`, `path: /`, 有効期限1日）と整合させる|
|`SecureTokenGenerator`（`infrastructure/security/secure_token_generator.go`）|`RequestPasswordResetUseCase`（`ResetTokenGenerator`）、`LogoutUseCase`（`JTIGenerator`）|コーディング規約「17. crypto/rand」に従い`crypto/rand`を用いる|

### メール送信（`infrastructure/mail/password_reset_event_handler.go`）

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`PasswordResetEventHandler`|`RequestPasswordResetUseCase`（`PasswordResetEventPublisher`経由）|`event.PasswordResetTokenIssued`を受け取り、Notification Context（メール送信基盤）が公開する参照手段を呼び出してリセットメール送信を依頼する（アーキテクチャ規約「6. Context間連携ルール」準拠。Notification Context内部のメール実装には直接依存しない）|

`cache/`・`queue/`は対象外（②に該当要件の記載なし）。

---

# 6. Presentation層設計

## Handler

### SessionHandler（`presentation/handler/session_handler.go`）

- struct名: `SessionHandler`
- 対応する呼び出し先: `*usecase.LoginUseCase`, `*usecase.LogoutUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`Login`|POST|`/api/v1/user/login`|
|`Logout`|DELETE|`/api/v1/user/logout`|

- `Login`処理順序: リクエストバインド（`request.LoginRequest`）→ Presentation Validation（`binding`タグによる型・必須・メール形式チェック）→ `LoginUseCase.Execute`呼び出し → 成功時、Cookie（`access_token`）を`httponly`/`secure`/`same_site: lax`/`path: /`/有効期限1日で設定 → `response.LoginResponse`へ変換して返却
- `Logout`処理順序: Middlewareで設定済みの`current_user`（AccountID）を取得 → `LogoutUseCase.Execute`呼び出し → Cookie（`access_token`）削除 → `response.MessageResponse`を返却

## RegistrationHandler（`presentation/handler/registration_handler.go`）

- struct名: `RegistrationHandler`
- 対応する呼び出し先: `*usecase.RegisterUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|ロール（固定値としてUseCaseへ渡す）|
|-|-|-|-|
|`SignUpStudent`|POST|`/api/v1/student/signup`|`student`|
|`SignUpTeacher`|POST|`/api/v1/teacher/signup`|`teacher`|
|`SignUpAdmin`|POST|`/api/v1/admin/signup`|`admin`|

- 処理順序（3メソッド共通）: リクエストバインド（`request.SignUpRequest`、ネストされた`user`キー構造を維持）→ Presentation Validation（`binding`タグによる型・必須チェック。`high_school_id`/`grade_id`のロール別必須判定は本来UseCase内の`RegistrationEligibilityPolicy`が担うため、Handlerでは追加の必須判定を行わない）→ ロール名を固定値として`dto.RegisterCommand`に設定し`RegisterUseCase.Execute`呼び出し → `response.RegisterResponse`へ変換して返却（Status 201）

> **②からの補足**: レスポンス本文には`profile_completed`, `user_personal_info`, `user_role`, `high_school`, `address`, `grade`等、認証コンテキストのAggregate外のプロフィール情報が含まれる（②「16. API互換方針」）。これらは②「5. Aggregate設計」により認証コンテキストの責務外であり、規約「12. 今後の課題」にも記載のとおりUser Context自体の②文書がまだ存在しない。本書では、これらのフィールドをHandler層で「User Context公開参照インターフェース（`UserProfileReader`、概念のみ）」から取得し`response.RegisterResponse`にマージする構成を示すが、`UserProfileReader`の具体的なメソッドシグネチャ・実装はUser Context側の②文書整備後に確定するものとし、本書では確定させない（推測、②に根拠のない詳細実装を新設しないための意図的な留保）。

## PasswordResetHandler（`presentation/handler/password_reset_handler.go`）

- struct名: `PasswordResetHandler`
- 対応する呼び出し先: `*usecase.RequestPasswordResetUseCase`, `*usecase.ChangePasswordUseCase`, `*usecase.VerifyResetTokenUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`RequestReset`|POST|`/api/v1/password/reset/request`|
|`ChangePassword`|PATCH|`/api/v1/password/reset`|
|`VerifyResetToken`|POST|`/api/v1/password/verify`|

- 各メソッド処理順序: リクエストバインド → Presentation Validation（型・必須・メール形式チェック）→ 対応するUseCase呼び出し → `response.MessageResponse`（または`VerifyResetToken`用レスポンス）へ変換して返却

## Request / Response DTO

### Request（`presentation/request/`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`LoginRequest`|`Email string`, `Password string`|`binding:"required,email"` / `binding:"required"`|
|`SignUpRequest`|`User SignUpUserParams`|`binding:"required"`|
|`SignUpUserParams`|`Email string`, `Password string`, `PasswordConfirmation string`, `Name string`, `NameKana string`, `HighSchoolID *uint`, `GradeID *uint`|`Email`: `binding:"required,email"`。`Password`/`PasswordConfirmation`/`Name`/`NameKana`: `binding:"required"`。`HighSchoolID`/`GradeID`: ロール別必須判定はDomain層（`RegistrationEligibilityPolicy`）に委ねるため、Presentationでは型チェックのみ|
|`PasswordResetRequestRequest`|`Email string`|`binding:"required,email"`|
|`ChangePasswordRequest`|`ResetPasswordToken string`, `Password string`, `PasswordConfirmation string`|`binding:"required"`|
|`VerifyResetTokenRequest`|`ResetPasswordToken string`|`binding:"required"`|

### Response（`presentation/response/`）

|struct名|フィールドと型|
|-|-|
|`LoginResponse`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `ProfileCompleted bool`, `UserPersonalInfo any`, `UserRole any`, `HighSchool any`, `Address any`, `Grade any`（②「16. API互換方針」のレスポンス構造を維持。ネストされた各フィールドの具体的な型は、User/School各Contextの③文書確定後に確定する。②からの補足）|
|`RegisterResponse`|`LoginResponse`と同一構造（②により登録成功時のレスポンス構造もユーザー情報構造を踏襲）|
|`MessageResponse`|`Message string`|

## Routing（`presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|POST|`/api/v1/user/login`|`SessionHandler.Login`|
|DELETE|`/api/v1/user/logout`|`SessionHandler.Logout`（認証Middleware必須）|
|POST|`/api/v1/student/signup`|`RegistrationHandler.SignUpStudent`|
|POST|`/api/v1/teacher/signup`|`RegistrationHandler.SignUpTeacher`|
|POST|`/api/v1/admin/signup`|`RegistrationHandler.SignUpAdmin`|
|POST|`/api/v1/password/reset/request`|`PasswordResetHandler.RequestReset`|
|PATCH|`/api/v1/password/reset`|`PasswordResetHandler.ChangePassword`|
|POST|`/api/v1/password/verify`|`PasswordResetHandler.VerifyResetToken`|

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|POST|/api/v1/user/login|SessionHandler.Login|LoginRequest|LoginResponse|200|
|DELETE|/api/v1/user/logout|SessionHandler.Logout|-（Cookie経由）|MessageResponse|200|
|POST|/api/v1/student/signup|RegistrationHandler.SignUpStudent|SignUpRequest|RegisterResponse|201|
|POST|/api/v1/teacher/signup|RegistrationHandler.SignUpTeacher|SignUpRequest|RegisterResponse|201|
|POST|/api/v1/admin/signup|RegistrationHandler.SignUpAdmin|SignUpRequest|RegisterResponse|201|
|POST|/api/v1/password/reset/request|PasswordResetHandler.RequestReset|PasswordResetRequestRequest|MessageResponse|200|
|PATCH|/api/v1/password/reset|PasswordResetHandler.ChangePassword|ChangePasswordRequest|MessageResponse|200|
|POST|/api/v1/password/verify|PasswordResetHandler.VerifyResetToken|VerifyResetTokenRequest|MessageResponse|200|

## Errorケース

|Endpoint|条件|Status Code|Error内容|
|-|-|-|-|
|login|メールアドレス/パスワード不一致・アカウント不存在|401|`ErrInvalidCredentials`（同一メッセージ）|
|login|入力形式不正|422|Request DTOバリデーションエラー|
|logout|`current_user`未設定（未認証）|401|Middlewareレベルで拒否|
|signup（全ロール）|登録要件違反（ロール不正・高校/学年未指定または不実在）|422|`ErrInvalidRole` / `ErrRegistrationRequirementNotMet`|
|signup（全ロール）|パスワード確認不一致|422|`ErrPasswordConfirmationMismatch`|
|signup（全ロール）|予期せぬエラー|500|Infrastructure Error|
|password/reset/request|任意（常に同一成功レスポンス）|200|-（②の方針によりユーザー存在有無を問わず成功として扱う）|
|password/reset（PATCH）|トークン不正・未発行|422|`ErrResetTokenNotFound` / `ErrResetTokenNotIssued`|
|password/reset（PATCH）|トークン期限切れ|422|`ErrResetTokenExpired`|
|password/reset（PATCH）|パスワード確認不一致|422|`ErrPasswordConfirmationMismatch`|
|password/verify|トークン不正・期限切れ|200|`Valid: false`をレスポンスに含める（②に個別エラーコードの記載がないため、成功レスポンス内で真偽値として表現する。推測）|

---

# 8. Transaction実装方針

## Transaction開始箇所

②「11. Transaction設計」により、UseCaseの開始時（読み取り専用UseCaseを除く）にトランザクションを開始する。

> **②からの補足**: 具体的な実装機構（GORMの`db.Transaction(func(tx *gorm.DB) error {...})`をどの層がラップするか）は②に明記がない。本書では、UseCaseがGORMに直接依存しない（アーキテクチャ規約「3. レイヤー責務と依存方向」の禁止事項）ことを守るため、`TransactionManager`インターフェースを導入する（推測）。

- interface名: `TransactionManager`（定義場所は利用側であるapplication層、例: `application/usecase/transaction_manager.go`）
- メソッドシグネチャ: `WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error`
- 実装: `infrastructure/repository`配下でGORMの`*gorm.DB.Transaction`を用いて実装し、`fn`に渡す`ctx`にトランザクション用の`*gorm.DB`を保持させる（Repository実装側が`ctx`からトランザクション用DBを取得する）

## Transaction終了箇所（Commit / Rollback条件）

|UseCase|終了箇所|
|-|-|
|LogoutUseCase|`AccountRepository.UpdateJTI`完了時点でCommit、エラー時Rollback|
|RegisterUseCase|`AccountRepository.Create`完了時点でCommit、途中の存在確認失敗・作成失敗時はRollback|
|RequestPasswordResetUseCase|`AccountRepository.SaveResetToken`完了時点でCommit（対象ユーザーが存在する場合のみトランザクションを使用）|
|ChangePasswordUseCase|`AccountRepository.UpdatePasswordAndConsumeResetToken`完了時点でCommit、トークン検証失敗時はトランザクションを開始しない|
|LoginUseCase / VerifyResetTokenUseCase|トランザクションを使用しない（読み取りのみ）|

## 複数Repositoryにまたがる場合の扱い

RegisterUseCaseは`UserRoleRepository` / `HighSchoolRepository` / `GradeRepository`（参照系、トランザクション不要）と`AccountRepository.Create`（書き込み）を扱うが、②「11. Transaction設計」の方針どおり、参照確認から作成までを1つの`TransactionManager.WithinTransaction`スコープ内で実行し、途中の不整合（不正なロール・高校・学年の組み合わせでの作成）を防ぐ。

---

# 9. Validation実装方針

## Presentation

- 型チェック: 各Request DTOの`binding`タグ（Ginの`ShouldBindJSON`使用）で、`email`, `password`, `user.*`, `password_reset.*`等の型を検証する
- 必須チェック: ログイン（`email`, `password`）、登録（`email`, `password`, `password_confirmation`, `name`, `name_kana`）、リセット系（`email`, `reset_password_token`, `password`, `password_confirmation`）の必須項目を`binding:"required"`で検証する
- フォーマットチェック: `email`は`binding:"email"`でメールアドレス形式を検証する

## 業務ルール検証（Domain Model採用のため②の記載どおり）

- Entity／Value Object生成時に検証する内容: `valueobject.NewEmail`（形式）、`valueobject.NewRawPassword`（最小文字数等）、`entity.NewAccount`（不変条件）
- UseCase内で判定する業務ルール:
  - `RegisterUseCase`: `RegistrationEligibilityPolicy.Validate`によるロール別必須項目（高校・学年）の判定
  - `ChangePasswordUseCase` / `VerifyResetTokenUseCase`: `PasswordResetLifecyclePolicy.CanConsume`によるトークン有効性判定
  - 全登録・リセット系: `RawPassword.Matches`によるパスワード確認一致判定

---

# 10. Authorization実装方針

②「13. Authorization設計」をそのまま実装レベルに落とし込む。

## Middleware

- Cookie（`access_token`）からJWTを取得し、署名検証・`jti`の一致確認を行って`current_user`（AccountID等）をGinの`*gin.Context`に設定する
- ログイン・登録（student/teacher/admin）・パスワードリセット系（request/reset/verify）エンドポイントは認証チェック対象外とする（Ginのルートグループを分離し、Middlewareを適用しない）
- ログアウトエンドポイントのみ、`current_user`の存在を前提とするMiddlewareを適用する

## Handler

- Cookie（`access_token`）のHTTP Only Cookie設定・削除操作を担当する
- 資格情報の照合やトークンの有効性判定は持たせない（UseCaseへ委譲する）

## UseCase

- `LogoutUseCase`は`current_user`（AccountID）が存在することを前提としてセッション無効化を実行する
- `RegisterUseCase` / パスワードリセット系UseCaseは、認証済みユーザーの有無に依存しない業務ルールの検証のみを行う

## Domain

- `Account` / `PasswordResetToken`が、資格情報照合・セッション無効化・トークン消費という「本人確認そのもの」に関わる判定を担う（`CredentialVerificationService`, `PasswordResetLifecyclePolicy`経由）

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

各UseCaseは、`domain/errors`のセンチネルエラーを`errors.Is`で判定し、必要に応じてApplication層独自のラップ（`fmt.Errorf("...: %w", err)`、コーディング規約「6. エラーのラップ」準拠）を行った上でHandlerへ伝播させる。`RequestPasswordResetUseCase`のみ、内部的な「対象ユーザー不存在」を外部エラーとして伝播させず、常に固定の成功結果（`dto.RequestPasswordResetResult`）に変換する（②「14. Error設計」のApplication Errorの方針どおり）。

## Application Error → HTTPレスポンスへの変換方針

Handler層で`errors.Is`によりDomain Errorの種別を判定し、以下のStatus Codeへ変換する。変換ロジック自体は共通のエラーマッピング関数（例: `presentation/handler`配下の非公開ヘルパー）に集約し、各Handlerで重複させない。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrInvalidCredentials`|Domain|401|
|`ErrInvalidRole`|Domain|422|
|`ErrRegistrationRequirementNotMet`|Domain|422|
|`ErrPasswordConfirmationMismatch`|Domain|422|
|`ErrResetTokenNotFound`|Domain|422|
|`ErrResetTokenExpired`|Domain|422|
|`ErrResetTokenNotIssued`|Domain|422|
|Request DTOバリデーションエラー|Presentation|422|
|DB接続失敗・クエリ失敗|Infrastructure|500|
|JWT署名・発行失敗|Infrastructure|500|
|メール送信基盤連携失敗（`PasswordResetEventHandler`）|Infrastructure|非同期処理のためHTTPレスポンスには影響させない（②の方針上、リセットリクエスト自体は常に成功として返す）|

## Infrastructure Errorのハンドリング方針

Infrastructure層（Repository実装・security・mail）で発生したエラーは、`fmt.Errorf`でラップしてApplication層へ伝播させ、Handler層で未分類のエラーとして500に変換する。業務エラー（Domain Error）と技術的障害（Infrastructure Error）を`errors.Is` / `errors.As`で明確に区別する。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」により、既存Rails DBをそのまま継続利用し、スキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

### UserModel（`infrastructure/persistence/gorm/user_model.go`）

- 対応テーブル: `users`（既存Railsスキーマ）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー（Gorm規約のデフォルト`ID`フィールドをそのまま利用）|
|`Email`|`email`|
|`EncryptedPassword`|`encrypted_password`|Rails Devise由来のカラム名をそのまま踏襲するため、フィールド名と規約上のsnake_case変換結果が一致する|
|`JTI`|`jti`|Gorm規約のデフォルトsnake_case変換では連続大文字の扱いが曖昧になり得るため、`gorm:"column:jti"`タグで明示的に指定する（Gorm規約「カラム名」の`column`タグ上書き方針に従う）|
|`ResetPasswordToken`|`reset_password_token`|
|`ResetPasswordSentAt`|`reset_password_sent_at`|
|`UserRoleID`|`user_role_id`|
|`HighSchoolID`|`high_school_id`|
|`GradeID`|`grade_id`|
|`Name`|`name`|
|`NameKana`|`name_kana`|
|`CreatedAt`|`created_at`|Gorm規約のタイムスタンプ自動トラッキングに従う|
|`UpdatedAt`|`updated_at`|同上|

- テーブル名: 構造体名`UserModel`はGorm規約のデフォルト複数形変換では`user_models`となり実テーブル名`users`と一致しないため、`Tabler`インターフェース（`func (UserModel) TableName() string { return "users" }`）を実装してテーブル名を明示的に指定する（Gorm規約「テーブル名」節に従う）。

### UserRoleModel / HighSchoolModel / GradeModel

- `UserRoleModel`は`user_roles`テーブル（存在確認のみに利用するため`ID`, `Name`程度の最小フィールドで定義する）
- `HighSchoolModel` / `GradeModel`はSchool Context側の既存モデル定義を参照する想定（5章参照。詳細はSchool Context側の③文書に委ねる）

## 主要クエリの条件・ソート・ページネーション方針

|操作|条件|ソート|ページネーション|
|-|-|-|-|
|`FindByEmail`|`email = ?`|不要（一意検索）|不要|
|`FindByResetToken`|`reset_password_token = ?`|不要（一意検索）|不要|
|`UserRoleRepository.FindByName`|`name = ?`|不要|不要|
|`HighSchoolRepository.Exists` / `GradeRepository.Exists`|`id = ?`|不要|不要|

本機能に一覧取得・集計クエリは存在しないため、ソート・ページネーション方針は対象外。

## 既存Schemaへの変更

②「17. DB設計方針」により変更なし。SQL文そのものは本書に記載しない。

---

# 13. テストケース設計

②「18. テスト戦略」をDomain Model採用時の区分のまま、具体的なテストケース単位に落とし込む。

## Domain Test

|対象|テストケース|
|-|-|
|`CredentialVerificationService.Verify`|パスワード一致時に成功する／不一致時に`ErrInvalidCredentials`相当を返す|
|`PasswordResetToken.IsValid`|発行直後は有効／有効期間経過後は無効と判定する|
|`ResetTokenValidityPeriod.IsExpired`|境界値（有効期間ちょうど）での判定|
|`RegistrationEligibilityPolicy.Validate`|student/teacherで高校・学年未指定時にエラー、adminで高校・学年未指定でも成功する|
|`SignUpRoleRequirement.RequiresHighSchool` / `RequiresGrade`|ロールごとの真偽値が②の要件どおりであること|
|`RawPassword.Matches`|一致／不一致の判定|
|`Email.NewEmail`|不正な形式でエラーになること|
|`PasswordResetLifecyclePolicy.CanConsume`|未発行・期限切れ・有効の3状態での判定|

## UseCase Test

|対象|テストケース|
|-|-|
|`LoginUseCase`|正しい資格情報でトークンが発行される／誤ったパスワードで`ErrInvalidCredentials`になる／存在しないメールアドレスでも同一エラーになる|
|`LogoutUseCase`|`jti`が更新されること|
|`RegisterUseCase`|student/teacherで高校・学年ID不足時にエラーになる／adminで高校・学年ID未指定でも成功する／ロール不正時にエラーになる|
|`RequestPasswordResetUseCase`|ユーザーが存在する場合にトークンが発行されイベントが発行される／存在しない場合でも同一の成功結果が返る（②の重点検証項目）|
|`ChangePasswordUseCase`|有効なトークンでパスワードが更新されトークンが消費される／期限切れトークンでエラーになる／トークン不一致でエラーになる|
|`VerifyResetTokenUseCase`|有効トークンで`Valid: true`／無効・期限切れで`Valid: false`|

## Repository Test

|対象|テストケース|
|-|-|
|`AccountRepositoryImpl.FindByEmail`|存在する/しないメールアドレスでの検索結果|
|`AccountRepositoryImpl.FindByResetToken`|存在する/しないトークンでの検索結果|
|`AccountRepositoryImpl.UpdateJTI`|更新後に`jti`が反映されること|
|`AccountRepositoryImpl.UpdatePasswordAndConsumeResetToken`|パスワードハッシュ更新とトークンクリアが同時に反映されること|
|`UserRoleRepositoryImpl.FindByName` / `HighSchoolRepositoryImpl.Exists` / `GradeRepositoryImpl.Exists`|存在する/しないIDでの判定結果|

## Handler Test

|対象|テストケース|
|-|-|
|`SessionHandler.Login`|バリデーションエラー時に422を返す／成功時にCookieが設定され200を返す|
|`SessionHandler.Logout`|未認証時に401を返す／成功時にCookieが削除され200を返す|
|`RegistrationHandler.SignUp*`|各ロールで必須項目欠如時に422を返す／成功時に201を返す|
|`PasswordResetHandler.RequestReset`|存在しないメールアドレスでも200・固定メッセージを返す|
|`PasswordResetHandler.ChangePassword`|不正トークンで422を返す／成功時に200を返す|
|`PasswordResetHandler.VerifyResetToken`|有効/無効トークンでの`Valid`値の違い|

## Integration Test

|対象|テストケース|
|-|-|
|ログイン→Cookie発行→ログアウト|ログアウト後、同一JWTでの再アクセスが認証エラーになること（`jti`不一致の確認）|
|ロール別登録|student/teacher/adminそれぞれで正常に登録が完了すること|
|パスワードリセット一連フロー|リクエスト→トークン検証→変更の一連の流れが正常に完了し、消費済みトークンの再利用が拒否されること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に列挙する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ディレクトリ名を`internal/auth`とした|②のContext名は`authentication`のみで、ディレクトリ名の明記がない。アーキテクチャ規約「5. 命名規則」に従い短縮した|推測|
|Account Entityに`Name`/`NameKana`を持たせず、Repository作成入力（`CreateAccountParams`）のみで受け渡す構成とした|②「5. Aggregate設計」がプロフィール情報をAggregate外としている方針を、RegisterUseCaseの入力要件と矛盾なく実装するための判断|②の記載同士の整合を取るための判断（推測ではない）|
|`PasswordHasher` / `TokenIssuer` / `JTIGenerator` / `ResetTokenGenerator` / `PasswordResetEventPublisher`という技術アダプタ用Interfaceを新設し、配置場所（domain/serviceまたはapplication/usecase）を決定した|②「Railsとの責務対応」がInfrastructure（トークン発行アダプタ）の必要性を示すのみで、Interface名・配置場所までは規定していない。コーディング規約「5. インターフェース」の「利用側で定義する」方針に従って配置した|推測|
|`TransactionManager`インターフェースによるトランザクション制御方式を導入した|②「11. Transaction設計」はトランザクション境界（開始・終了タイミング）のみを定めており、UseCaseがGORMに直接依存しないための実装機構は指定していない。アーキテクチャ規約「3. レイヤー責務と依存方向」の禁止事項（UseCaseがInfrastructure具象実装に依存しない）を満たすための判断|推測|
|`RawPassword`の最小文字数、`ResetTokenValidityPeriod`の具体的な有効期間の値を確定しなかった|②自身がこれらを「推測」と明記し、Devise標準ポリシーを踏襲する前提としているのみで具体的な数値がない。①（Rails実装詳細）が未提供のため、本書でも数値を確定できない|①未提供のため参照不可（②の推測を維持）|
|`LoginResponse` / `RegisterResponse`のプロフィール関連フィールド（`user_personal_info`, `high_school`, `address`, `grade`等）の取得方法として、`UserProfileReader`という概念的なUser Context参照ポートを提示するに留めた|②「5. Aggregate設計」により認証コンテキストの責務外である一方、②「16. API互換方針」のレスポンス構造にはこれらが含まれる。アーキテクチャ規約「12. 今後の課題」がUser Context自体の②文書未整備を認めているため、詳細実装はUser Context側の文書確定後とした|推測（②に根拠のない新規実装を先取りしないための意図的な留保）|
|HighSchoolModel / GradeModelの定義場所をSchool Context側の既存モデル参照とした|②「9. Repository設計」はHighSchool/GradeRepositoryを「参照専用」とするのみで、モデル定義の所在は指定していない。アーキテクチャ規約「6. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従った|推測|
|`VerifyResetToken`のレスポンスを個別エラーコードではなく`Valid: bool`を含む200レスポンスとした|②「7. API仕様」相当のStatus Code一覧に`password/verify`固有のエラーステータスの明記がないため、既存の確認系エンドポイントの一般的な設計として判断した|推測|

上記以外の設計判断（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
