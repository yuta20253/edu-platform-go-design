# 認証機能 Go実装仕様書（4. パスワードリセット）

---

# 1. 本書の範囲

## 担当するエンドポイント

|Method|Path|Handler|概要|
|-|-|-|-|
|POST|`/api/v1/password_resets`|`PasswordResetHandler.Request`|リセットトークンを発行し、メールを送る。退会済みユーザーは対象外とし、結果は常に同一のメッセージを返す|
|PATCH|`/api/v1/password_resets`|`PasswordResetHandler.Change`|トークンを消費してパスワードを更新する。招待メールのリンクからの初回設定もこの経路|
|GET|`/api/v1/password_resets/verify`|`PasswordResetHandler.Verify`|リセットトークンの有効性を確認する|

## 担当するファイル

```
internal/authentication/domain/entity/password_reset_token.go
internal/authentication/domain/event/password_reset_token_issued.go
internal/authentication/domain/service/password_reset_lifecycle_policy.go
internal/authentication/domain/valueobject/reset_token_validity_period.go
internal/authentication/application/dto/password_reset_dto.go
internal/authentication/application/usecase/change_password_usecase.go
internal/authentication/application/usecase/request_password_reset_usecase.go
internal/authentication/application/usecase/verify_reset_token_usecase.go
internal/authentication/infrastructure/mail/auth_notifier.go
internal/authentication/infrastructure/security/secure_token_generator.go
internal/authentication/presentation/handler/password_reset_handler.go
internal/authentication/presentation/request/password_reset_request.go
```

実装12ファイル、テスト込みで約20。

## 本書が前提とする文書

- `認証機能_Go実装仕様書_1_基盤.md`: Domain（`Account` Entity・`Email`・`RawPassword`・`StudentNumber`・`AccountRepository`・`UserRoleRepository`・`CredentialVerificationService`・`PasswordHasher`・Domain Error）、共通Infrastructure（`AccountRepositoryImpl`・`UserRoleRepositoryImpl`・`BcryptPasswordHasher`・GORMモデル・`UserResponse`・`MessageResponse`・`routes.go`）、およびTransaction・Validation・Authorization・Error・GORMの実装方針
- 基盤編を実装・マージしたうえで本書の実装に着手する

本書では、基盤編に記載した実装方針を繰り返さない。

---

# 2. Domain層設計（本書の担当）

基盤編に記載したDomainは繰り返さない。本書は本スライスだけが使うDomainを扱う。

## Entity

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

### ResetTokenValidityPeriod（`domain/valueobject/reset_token_validity_period.go`）

- struct名: `ResetTokenValidityPeriod`
- フィールド: `duration time.Duration`（非公開）
- 生成時に検証するルール: 発行からの有効期間（②により、Devise標準設定を踏襲する前提、具体的な期間は②でも「推測」）
- 公開メソッド: `NewResetTokenValidityPeriod(d time.Duration) ResetTokenValidityPeriod` / `(p ResetTokenValidityPeriod) IsExpired(issuedAt, now time.Time) bool`

> **②からの補足**: 具体的な有効期間の値は②でも未確定（推測）。本書では型・判定メソッドのシグネチャのみを定義し、具体的な時間値は実装時にRails現行設定（`config.reset_password_within`相当）を①側で確認のうえ確定する必要がある（①未提供のため参照不可）。

## Domain Service

### PasswordResetLifecyclePolicy（`domain/service/password_reset_lifecycle_policy.go`）

- struct名: `PasswordResetLifecyclePolicy`
- メソッドシグネチャ: `(p PasswordResetLifecyclePolicy) CanConsume(token *entity.PasswordResetToken, now time.Time, period valueobject.ResetTokenValidityPeriod) error`
- 責務: リセットトークンが現在の状態から見て消費（パスワード更新）可能かを判定する。トークン不一致・未発行・期限切れをそれぞれ区別したDomain Errorを返す

> **②からの補足**: ②「6. Entity設計」の「PasswordResetToken」節は、リセット対象ユーザーの検索において退会済みユーザーを除外することも本Policyの前提条件の一部として言及している。本書では、この退会済み除外は`PasswordResetLifecyclePolicy`の判定メソッド内で行うのではなく、`AccountRepository.FindByEmailExcludingDeleted`によるクエリレベルでのスコープ限定として実装する（③3章のRepository Interface参照）。理由は、退会済み判定はAccountの属性（`DeletedAt`）のみで完結し、GORMの論理削除機構（`gorm.DeletedAt`、Gorm規約「0. 本プロジェクトでの採用方針」）によって検索クエリの時点で自然に除外できるため、Policyに判定ロジックを重複させる必要がないという実装判断である（②の記載同士の整合を取るための判断）。

## Domain Event

### PasswordResetTokenIssued（`domain/event/password_reset_token_issued.go`）

- イベントstruct名: `PasswordResetTokenIssued`
- 保持するフィールド: `AccountID uint`, `Email string`, `Token string`, `IssuedAt time.Time`
- 発火元: `RequestPasswordResetUseCase`（対象ユーザーが存在しリセットトークンが発行された時点）

---

# 3. Application層設計

## DTO（Command / Query）

|struct名|フィールド|区分|
|-|-|-|
|`RequestPasswordResetCommand`|`Email string`|Command|
|`RequestPasswordResetResult`|`Message string`|出力（常に固定の成功メッセージ）|
|`ChangePasswordCommand`|`ResetPasswordToken string`, `Password string`, `PasswordConfirmation string`|Command|
|`ChangePasswordResult`|`Message string`|出力|
|`VerifyResetTokenQuery`|`ResetPasswordToken string`|Query|
|`VerifyResetTokenResult`|`Valid bool`|出力|

`HighSchoolInfo{ID uint, Name string}`, `GradeInfo{ID uint, Year int, DisplayName string}`, `PersonalInfoView`/`AddressView`はprofile Context側の参照結果をそのまま転記する構造（フィールド詳細はprofile Contextの③文書に委ねる）。

## UseCase

### RequestPasswordResetUseCase（`application/usecase/request_password_reset_usecase.go`）

- struct名: `RequestPasswordResetUseCase`
- コンストラクタが受け取る依存: `AccountRepository`, `ResetTokenGenerator`（Interface、後述）, `TransactionManager`, `AuthNotifier`
- 公開メソッド: `(u *RequestPasswordResetUseCase) Execute(ctx context.Context, cmd dto.RequestPasswordResetCommand) (dto.RequestPasswordResetResult, error)`
- 処理ステップ:
  1. `valueobject.NewEmail`で入力形式を検証する
  2. `AccountRepository.FindByEmailExcludingDeleted`で対象アカウントを検索する（退会済みは対象外、②「11. Repository設計」）
  3. 存在する場合のみ、`ResetTokenGenerator.Generate`でトークンを生成し、`TransactionManager.WithinTransaction`内で`AccountRepository.SaveResetToken`を実行する
  4. 存在する場合、`AuthNotifier.NotifyPasswordResetRequested(ctx, event.PasswordResetTokenIssued{...})`を呼び出す
  5. 存在有無に関わらず、常に同一の`RequestPasswordResetResult{Message: "..."}`を返す
- トランザクション境界: ユーザーが存在する場合のみ、トークン発行を1トランザクションとする
- 発生しうるApplication Error: なし（②の方針により、内部的な失敗理由は外部に露出させず常に成功として扱う。Infrastructure Errorのみ5xxとして伝播する）

> **②からの補足**: `ResetTokenGenerator`はセキュリティ用途の乱数生成であり、コーディング規約「23. `crypto/rand`」に従い`crypto/rand`ベースで実装する（`infrastructure/security/secure_token_generator.go`）。

### ChangePasswordUseCase（`application/usecase/change_password_usecase.go`）

- struct名: `ChangePasswordUseCase`
- コンストラクタが受け取る依存: `AccountRepository`, `UserRoleRepository`, `*service.PasswordResetLifecyclePolicy`, `PasswordHasher`, `TransactionManager`
- 公開メソッド: `(u *ChangePasswordUseCase) Execute(ctx context.Context, cmd dto.ChangePasswordCommand) (dto.ChangePasswordResult, error)`
- 処理ステップ:
  1. `valueobject.NewRawPassword`で入力を検証し、確認用パスワードとの一致を検証する
  2. `AccountRepository.FindByResetToken`で対象アカウントを検索する
  3. `PasswordResetLifecyclePolicy.CanConsume`でトークンの有効性を判定する
  4. `UserRoleRepository.FindByID(ctx, account.UserRoleID)`でアカウントのロール名を取得する（ロールが見つからない場合はInfrastructure Error扱い。アカウントのロールIDはマスタに存在する前提）
  5. `PasswordHasher.Hash`で新パスワードをハッシュ化する
  6. `account.CompleteInvitation(roleName)`で招待待ちを解消する（管理者・教員・生徒の`PasswordResetRequired`が偽になる）
  7. `TransactionManager.WithinTransaction`内で`AccountRepository.UpdatePasswordAndConsumeResetToken(ctx, account.ID, newPasswordHash, account.PasswordResetRequired)`を実行する（パスワードハッシュ更新・トークン消費・招待状態の反映を1操作で行う）
  8. `ChangePasswordResult`を返す
- トランザクション境界: トークン検証からパスワード更新・トークン消費・招待待ちの解消（教員・生徒の場合）までを1トランザクションとする
- 発生しうるApplication Error: `ErrResetTokenNotFound`, `ErrResetTokenExpired`, `ErrResetTokenNotIssued`, `ErrPasswordConfirmationMismatch`

> **②からの補足**: 招待メールのリンクからのパスワード設定は、パスワードリセットと同じトークンの消費であるため、`ChangePasswordUseCase`が招待待ち→招待完了の遷移を担う（②「10. 状態遷移図」「12. UseCase設計」）。招待待ちの解消の対象ロールは、Rails現行の`Auth::ChangePasswordService`に従い管理者・教員・生徒である。既に招待完了のアカウント（招待とは無関係の通常のパスワードリセット）では、`PasswordResetRequired`は偽のまま変化しない。②の「トークン有効性検証からパスワード更新・トークン消費までを1トランザクション」という記載は、招待待ちの解消を含む範囲として実装する。新たなDomain Errorは追加しない。

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

# 4. Infrastructure層設計（本書の担当）

## 認証技術アダプタ（本書の担当）

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`SecureTokenGenerator`（`infrastructure/security/secure_token_generator.go`）|`RequestPasswordResetUseCase`（`ResetTokenGenerator`）、`LogoutUseCase`（`JTIGenerator`）|コーディング規約「23. `crypto/rand`」に従い`crypto/rand`を用いる|

`SecureTokenGenerator`はセッション編の`LogoutUseCase`も`JTIGenerator`として使う。セッション編を先に実装する場合はそちらで作成してよい。

## 外部連携実装（本書の担当）

### 通知（`infrastructure/mail/auth_notifier.go`）

- struct名: `AuthNotifier`（`application/usecase`層で定義するインターフェースの実装）
- インターフェース定義（`application/usecase`層、利用側で定義。コーディング規約「7. インターフェース」）:

```go
type AuthNotifier interface {
    NotifyPasswordResetRequested(ctx context.Context, evt event.PasswordResetTokenIssued) error
    NotifyAccountActivated(ctx context.Context, evt event.AccountActivated) error
}
```

- 実装方針: アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の「ベストエフォートで良い処理」区分に従い、`jobs`テーブルを経由せず、各メソッド内で`go func() { ... }()`によりgoroutineを直接起動して`teacher-notification` Context（メール送信基盤）が公開する参照手段を呼び出す。`ctx`はリクエストの生存期間に紐づけず`context.Background()`を使用する（規約13章の例に準拠）。送信失敗はログ出力のみで、呼び出し元（UseCase）へはエラーを伝播させない

`cache/`・`queue/`は対象外（②に該当要件の記載なし。非同期メール送信は上記のとおり規約13章の「ベストエフォート処理」で扱うため`jobs`テーブル基盤は不要）。

---

---

# 5. Presentation層設計

## Handler

### PasswordResetHandler（`presentation/handler/password_reset_handler.go`）

- struct名: `PasswordResetHandler`
- 対応する呼び出し先: `*usecase.RequestPasswordResetUseCase`, `*usecase.ChangePasswordUseCase`, `*usecase.VerifyResetTokenUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`RequestReset`|POST|`/api/v1/password/reset/request`|
|`ChangePassword`|PATCH|`/api/v1/password/reset`|
|`VerifyResetToken`|POST|`/api/v1/password/verify`|

- 各メソッド処理順序: リクエストバインド → Presentation Validation（型・必須・メール形式チェック）→ 対応するUseCase呼び出し → `response.MessageResponse`（または`VerifyResetToken`用レスポンス）へ変換して返却

## Request DTO

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`PasswordResetRequestRequest`|`Email string`|`binding:"required,email"`|
|`ChangePasswordRequest`|`ResetPasswordToken string`, `Password string`, `PasswordConfirmation string`|`binding:"required"`|
|`VerifyResetTokenRequest`|`ResetPasswordToken string`|`binding:"required"`|

`GetCurrentUser`（`GET /api/v1/me`）はパラメータを持たないためRequest DTOを設けない。

Response DTO（`UserResponse`・`MessageResponse`）は基盤編「6. Infrastructure層設計（共通）」の「共通のResponse DTO」に従う。

---

# 6. API仕様

## Errorケース

|Endpoint|条件|Status Code|Error内容|
|-|-|-|-|

---

---

# 7. テストケース設計（本書の担当分）

Domain Test・Repository Testのうち基盤編が担当する分は基盤編に記載する。本書は本スライスのUseCase・Handlerのテストを扱う。
