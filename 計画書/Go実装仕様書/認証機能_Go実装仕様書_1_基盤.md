# 認証機能 Go実装仕様書（1. 基盤）

---

# 1. 機能概要

## 機能概要

ユーザーのログイン・ログアウト・新規登録（student/teacher/admin）・パスワードリセット・ログイン中ユーザー自身の基礎情報取得（`GET /api/v1/me`）を提供し、認証状態（セッションの有効性を表す`jti`、パスワードリセットトークンの有効性、招待状態、仮アカウントの有効化状態）を管理する機能である。JWTをHTTP Only Cookieで、クライアントAPI互換性を維持したまま提供する。新規登録では、生徒が学校発行の生徒コード（`student_number`）を入力した場合、新規アカウント作成の代わりに、`user` Contextの`CreateStudentAccount`が事前に作成した仮アカウント（生徒CSVインポート・教師による生徒の単体登録で作成されたもの）を本人のものとして有効化する分岐を持つ。パスワードリセット対象の検索は、退会（論理削除）済みユーザーを除外して行う。

招待による生徒・教員・管理者アカウントの作成（仮パスワードの発行・招待待ちの初期状態・招待メールの送信依頼）は`user` Contextの責務であり、本機能は招待による作成の起点ではない。本機能が`users`の行を作るのは、本人による自己登録（サインアップ）のみである。一方、招待メールのリンクからのパスワード設定は、パスワードリセットと同じトークンの消費として本機能が受け付け、設定に成功した管理者・教員・生徒は招待待ちから招待完了（`password_reset_required`が偽）へ遷移する（②「3. Bounded Context」「10. 状態遷移図」）。

## 採用設計パターンとその理由（②からの要約）

②Go移行・設計仕様書「4. 設計パターン」により、本機能は **Domain Model** を採用する。

- Account（資格情報・`jti`・招待状態・仮アカウント有効化状態）、PasswordResetToken（発行・有効・期限切れ・消費済み）という状態遷移を持つ概念が中心にあること
- パスワード照合・ロール別登録要件・仮アカウント有効化要件・リセットトークン有効期限判定・退会済みユーザー除外という複数の業務ルールが複数UseCaseにまたがって再利用されること
- セキュリティ上重要なロジックを型・サービスとして明示し、レビュー可能性とテスト容易性を高める必要があること

上記の理由からTransaction Script・Active Record・Event Sourcingは採用せず、Domain Modelを採用している（詳細は②「4. 設計パターン」参照）。

## 本書群が対象とする実装範囲

本書は、②で確定した設計（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略）を変更せず、Goでの具体的なコード構成（package構成・struct定義・interfaceメソッドシグネチャ・クエリ内容）に落とし込むことを目的とする。②は25節構成へ全面刷新済みであり、本書はその内容（退会済み除外、生徒コード有効化分岐、StudentNumber Value Object、GetCurrentUserUseCase、profile/master-data Contextへの参照）を実装レベルに反映する。

規約`アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」の「Domain Model」節に従い、`{context}/domain`・`application`・`infrastructure`・`presentation`のフルレイヤー構成を適用する。TransactionManager・AppErrorの実装パターンは、②個別の推測に頼らず、規約「11. Transaction実装パターン（TransactionManager）」「12. Error変換パターン（AppError）」で標準化された内容にそのまま従う。

①Rails実装詳細は本タスクでは提供されていないため、①の実装コードそのものを根拠とする記載は行わない（①未提供のため参照不可）。②に明記された「Rails現行仕様の要約」の範囲でのみ言及する。

---

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名（②）: `authentication`
- ディレクトリ名: `internal/authentication`

  > **ディレクトリ名について**: アーキテクチャ規約「4. Bounded Context構成」の「ディレクトリ構成」の命名規則（Context名のkebab-caseを`_`区切りに置き換え、短縮しない）により、Context名 `authentication` に対応するディレクトリ名は `internal/authentication/` となる（1語のため変換なし）。

## ②で採用した設計パターン

Domain Model

## 文書の担当

本機能の実装は54ファイルで、1文書あたり30〜35ファイル（テスト込み）に収めるため4文書に分割する（③自動生成プロンプト「2. ディレクトリ構成」の「文書の分割」）。

|文書|担当するエンドポイント|実装|テスト込みの目安|
|-|-|-|-|
|`認証機能_Go実装仕様書_1_基盤.md`（本書）|なし（Domain・共通Infrastructure・実装方針）|19|約31|
|`認証機能_Go実装仕様書_2_セッション.md`|`POST /api/v1/login`・`DELETE /api/v1/logout`・`GET /api/v1/me`|11|約19|
|`認証機能_Go実装仕様書_3_登録.md`|`POST /api/v1/sign_up`（通常登録・仮アカウント有効化）|12|約20|
|`認証機能_Go実装仕様書_4_パスワードリセット.md`|`POST /api/v1/password_resets`・`PATCH /api/v1/password_resets`・`GET /api/v1/password_resets/verify`|12|約20|

基盤編 → 各スライス編の順に実装すれば、各PRは単体でコンパイルが通る。スライス編の間に依存はないため、2つ目以降の順序は問わない。

Transaction・Validation・Authorization・Error・GORMの実装方針は機能全体で共通であるため本書に置く。スライス編では繰り返さず、本書を参照する。


## 作成するディレクトリ一覧

```
internal/authentication/
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

`domain/specification/`・`infrastructure/cache/`・`infrastructure/queue/`は本機能では対象外（②に該当する業務ルール・キャッシュ要件・独自の非同期実行基盤の必要性の記載がない。非同期送信はアーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の「ベストエフォート処理（goroutine起動）」で扱い、`jobs`テーブル基盤は使用しないため`infrastructure/queue/`も不要）。

`infrastructure/security/`は規約の標準ディレクトリ例（`persistence/gorm`, `repository`, `mail`, `cache`, `queue`）には含まれないが、②「23. Railsとの責務対応」の「Devise + warden-jwt → Infrastructure（トークン発行アダプタ）」に基づき、パスワードハッシュ照合・JWT発行という技術アダプタの置き場所として新設する（②からの補足、推測）。

## 作成するファイル一覧（機能全体）

### 基盤編（本書）の担当

```
internal/authentication/domain/entity/account.go
internal/authentication/domain/errors/errors.go
internal/authentication/domain/repository/account_repository.go
internal/authentication/domain/repository/user_role_repository.go
internal/authentication/domain/service/credential_verification_service.go
internal/authentication/domain/service/password_hasher.go
internal/authentication/domain/valueobject/email.go
internal/authentication/domain/valueobject/raw_password.go
internal/authentication/domain/valueobject/student_number.go
internal/authentication/application/apperror/apperror.go
internal/authentication/application/usecase/transaction_manager.go
internal/authentication/infrastructure/persistence/gorm/user_model.go
internal/authentication/infrastructure/repository/account_repository.go
internal/authentication/infrastructure/repository/user_role_repository.go
internal/authentication/infrastructure/repository/transaction_manager.go
internal/authentication/infrastructure/security/bcrypt_password_hasher.go
internal/authentication/presentation/response/message_response.go
internal/authentication/presentation/response/user_response.go
internal/authentication/presentation/routes.go
```

`user_response.go`は、ログイン・登録・`GET /api/v1/me`の3経路が同形式のレスポンスを返すため（②「19. API仕様」）、スライス編ではなく本書の担当とする。

### セッション編の担当

```
internal/authentication/application/dto/get_current_user_dto.go
internal/authentication/application/dto/login_dto.go
internal/authentication/application/dto/logout_dto.go
internal/authentication/application/usecase/get_current_user_usecase.go
internal/authentication/application/usecase/login_usecase.go
internal/authentication/application/usecase/logout_usecase.go
internal/authentication/infrastructure/repository/profile_reader.go
internal/authentication/infrastructure/security/jwt_token_issuer.go
internal/authentication/presentation/handler/current_user_handler.go
internal/authentication/presentation/handler/session_handler.go
internal/authentication/presentation/request/session_request.go
```

### 登録編の担当

```
internal/authentication/domain/event/account_activated.go
internal/authentication/domain/repository/grade_repository.go
internal/authentication/domain/repository/high_school_repository.go
internal/authentication/domain/service/provisional_account_activation_policy.go
internal/authentication/domain/service/registration_eligibility_policy.go
internal/authentication/domain/valueobject/signup_role_requirement.go
internal/authentication/application/dto/register_dto.go
internal/authentication/application/usecase/register_usecase.go
internal/authentication/infrastructure/repository/grade_repository.go
internal/authentication/infrastructure/repository/high_school_repository.go
internal/authentication/presentation/handler/registration_handler.go
internal/authentication/presentation/request/registration_request.go
```

### パスワードリセット編の担当

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


---

# 3. Domain層設計

本書は、複数のスライス編が使うDomainを扱う。スライス固有のDomain（`PasswordResetToken` Entity・`SignUpRoleRequirement`・`ResetTokenValidityPeriod`・各Domain Service・Domain Event）は、各スライス編で定義する。

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
|`GradeID`|`*uint`|学年ID（student/teacherかつ生徒コード管理対象校でない場合のみ設定）|
|`StudentNumber`|`*valueobject.StudentNumber`|生徒コード（生徒コード管理対象校の生徒のみ設定、他はnil）|
|`PasswordResetRequired`|`bool`|`true`の間は「招待待ち」、`false`の間は「招待完了」を示す。`true`かつ`ActivatedAt`未設定の間は「仮登録」状態でもある（②「6. Entity設計」「10. 状態遷移図」）|
|`ActivatedAt`|`*time.Time`|仮アカウントが有効化された日時。生徒コードによる仮アカウントの有効化（`Activate`）でのみ設定し、通常登録（自己登録）では設定しない（未設定=nilのまま。Rails現行の自己登録が`activated_at`を設定しないことに合わせる。`user`②「20章」と同じ整理）|
|`DeletedAt`|`*time.Time`|論理削除日時（退会済みの場合に設定。②「11. Repository設計」の退会済み除外判定に用いる）|
|`ResetToken`|`*entity.PasswordResetToken`|発行中のパスワードリセットトークン（未発行時はnil）|

- 公開メソッド一覧（引数・戻り値のみ。ロジックは記述しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewAccount`|`(id uint, email valueobject.Email, passwordHash string, userRoleID uint, jti string, highSchoolID, gradeID *uint, studentNumber *valueobject.StudentNumber, passwordResetRequired bool, activatedAt *time.Time, deletedAt *time.Time) (*Account, error)`|`(*Account, error)`|不変条件を満たしたAccountを生成するファクトリ|
|`RotateJTI`|`(newJTI string)`|`error`|ログアウト時に`jti`をローテーションし、以前のセッションを無効化する|
|`UpdatePasswordHash`|`(newHash string)`|`error`|パスワードリセット成功時にハッシュを更新する|
|`IssuePasswordResetToken`|`(token string, issuedAt time.Time)`|`error`|リセットトークンを発行し保持する|
|`ConsumePasswordResetToken`|`()`|`error`|リセットトークンを消費済み（nil化）にする|
|`HasResetToken`|`()`|`bool`|リセットトークンが発行済みかを判定する|
|`IsProvisional`|`()`|`bool`|`PasswordResetRequired`が真かつ`ActivatedAt`が未設定かどうかを判定する（②「10. 状態遷移図」の遷移条件をそのまま反映）|
|`IsDeleted`|`()`|`bool`|退会（論理削除）済みかどうかを判定する|
|`IsInvitationPending`|`()`|`bool`|`PasswordResetRequired`が真（招待待ち）かどうかを判定する（②「10. 状態遷移図」の招待状態）|
|`CompleteInvitation`|`(roleName string)`|`error`|パスワードの設定に成功したときに、招待待ちを解消して招待完了へ遷移させる。`roleName`が`teacher`または`student`の場合のみ`PasswordResetRequired`を偽にし、それ以外（`admin`を含む）は変更しない。既に招待完了（偽）の場合も変化しない。`roleName`が空の場合はエラーを返す（②「6. Entity設計」「10. 状態遷移図」）|
|`Activate`|`(now time.Time)`|`error`|仮登録状態から有効化済み状態へ遷移する（`IsProvisional()`が`false`の場合はエラーを返す）。氏名・氏名カナ等プロフィール項目の反映は本メソッドの責務としない（後述の②からの補足を参照）|

- 不変条件（ファクトリで保証する内容）:
  - `Email`は`valueobject.Email`型としてのみ保持され、生成時に形式検証済みであることが保証される
  - `PasswordHash`は空文字を許容しない
  - `UserRoleID`は0を許容しない（登録時にUserRoleRepositoryで存在確認済みの値のみを渡す）
  - `PasswordResetRequired`が`true`かつ`ActivatedAt`が非nilという組み合わせ（有効化済みなのに仮登録要求フラグが残っている状態）は不変条件違反として`NewAccount`がエラーを返す

> **②からの補足**: ②「5. Aggregate設計」はAccountのAggregate境界から「プロフィール情報（氏名・個人情報・住所等）」を明示的に除外している一方、②「6. Entity設計」はAccountの責務として「有効化時に入力内容（氏名・氏名カナ等）で自身を更新する」を挙げており、両者は文言上ややテンションがある。本書ではAggregate境界（5章）の方を優先し、`Account`構造体自体には`Name`/`NameKana`フィールドを持たせない。「自身を更新する」という6章の記述は、`Activate()`によるドメイン状態（`ActivatedAt`・`PasswordResetRequired`）の遷移と、氏名・氏名カナという非Aggregate項目のRepository経由での永続化（`AccountRepository.ActivateProvisionalAccount`の`ActivateAccountParams`、後述）が1つのUseCase処理内で一体的に行われることとして実装する。旧版の`RegisterUseCase`（通常登録）における`CreateAccountParams`と同じ考え方を、仮アカウント有効化にも適用したものであり、②の記載同士（5章のAggregate境界と6章の責務記述）を矛盾なく実装に落とし込むための判断である（推測ではなく、②の記載同士の整合を取るための判断）。

> **②からの補足**: ②「6. Entity設計」「10. 状態遷移図」は、パスワード設定の成功時に、管理者・教員・生徒の`password_reset_required`を偽にする招待待ち→招待完了の遷移を、Accountの責務として定めている。`Account`はロール名ではなく`UserRoleID`のみを保持するため、`CompleteInvitation`はロール名を引数に取り、UseCaseが`UserRoleRepository.FindByID`で取得した値を渡す構成とした。ロール名は`student` / `teacher` / `admin`の文字列（`SignUpRoleRequirement`と同じ値）とする。`Activate`（生徒コードによる有効化）も`PasswordResetRequired`を偽にするが、こちらは仮登録からの遷移としてロールを問わず`IsProvisional()`を前提とし、`CompleteInvitation`とは別の遷移である（②の記載を実装に落とし込むための判断。推測）。

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

### StudentNumber（`domain/valueobject/student_number.go`）

- struct名: `StudentNumber`
- フィールド: `schoolCode string`（非公開）, `body string`（非公開）
- 生成時に検証するルール: 「学校コード-コード本体」のハイフン区切り形式であること（②「7. Value Object設計」）
- 公開メソッド:
  - `NewStudentNumber(raw string) (StudentNumber, error)`
  - `(s StudentNumber) SchoolCode() string`
  - `(s StudentNumber) String() string`
  - `(s StudentNumber) MatchesSchoolCode(schoolCode string) bool`

> **②からの補足**: ②「15. Validation設計」の「バリデーション仕様」表では、`student_number`のフォーマットチェック自体はPresentation層の責務とされている。本Value Objectは、Presentationのチェックをすり抜けた不正な形式がDomain層に到達しないための多重防御（defense in depth）として、Domain側でも独立して形式検証を行う構成とする。フォーマット不正時のエラーはEmail/RawPasswordと同様、Domain Errorのセンチネル変数としては定義せず、`NewStudentNumber`が返す通常のerrorとして扱う（②に個別のセンチネルエラー名の指定がないため、Email/RawPasswordの既存方針に合わせた実装判断。推測）。

②「7. Value Object設計」の方針どおり、`jti`は単なるUUID文字列として`Account.JTI string`にそのまま保持し、Value Object化しない。

## Repository Interface

### AccountRepository（`domain/repository/account_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByID`|`(ctx context.Context, id uint)`|`(*entity.Account, error)`|IDによる一意検索（`GetCurrentUserUseCase`用）|
|`FindByEmail`|`(ctx context.Context, email valueobject.Email)`|`(*entity.Account, error)`|メールアドレスによる一意検索（ログイン時。②の方針により退会済みも検索対象に含める、後述の②からの補足参照）|
|`FindByEmailExcludingDeleted`|`(ctx context.Context, email valueobject.Email)`|`(*entity.Account, error)`|メールアドレスによる一意検索（パスワードリセットリクエスト時。退会（論理削除）済みアカウントを除外する。②「11. Repository設計」の退会済み除外方針）|
|`FindByResetToken`|`(ctx context.Context, token string)`|`(*entity.Account, error)`|リセットトークンによる検索（トークン検証・パスワード更新時）|
|`FindByStudentNumber`|`(ctx context.Context, studentNumber string)`|`(*entity.Account, error)`|生徒コードによる仮アカウント検索（新規登録時の仮アカウント特定。②「11. Repository設計」）|
|`Create`|`(ctx context.Context, params CreateAccountParams)`|`(*entity.Account, error)`|アカウントの新規作成（通常登録＝本人による自己登録時。招待による作成は`user` Contextが行い、本メソッドは行わない）。`CreateAccountParams`にName/NameKanaを含む|
|`ActivateProvisionalAccount`|`(ctx context.Context, accountID uint, params ActivateAccountParams)`|`error`|仮アカウントを入力内容で更新し、`activated_at`を設定する（②「12. UseCase設計」の仮アカウント有効化フロー）|
|`UpdateJTI`|`(ctx context.Context, accountID uint, newJTI string)`|`error`|`jti`の更新（ログアウト時）|
|`SaveResetToken`|`(ctx context.Context, accountID uint, token string, issuedAt time.Time)`|`error`|リセットトークン発行情報の保存|
|`UpdatePasswordAndConsumeResetToken`|`(ctx context.Context, accountID uint, newPasswordHash string, passwordResetRequired bool)`|`error`|パスワードハッシュ更新とリセットトークン消費（クリア）に加え、招待状態（`passwordResetRequired`。`Account.CompleteInvitation`適用後の値）の反映を1操作で行う|

`CreateAccountParams`のフィールド: `Email valueobject.Email`, `PasswordHash string`, `UserRoleID uint`, `HighSchoolID *uint`, `GradeID *uint`, `Name string`, `NameKana string`

`ActivateAccountParams`のフィールド: `Email valueobject.Email`, `PasswordHash string`, `Name string`, `NameKana string`, `ActivatedAt time.Time`

### UserRoleRepository（`domain/repository/user_role_repository.go`）

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`FindByName`|`(ctx context.Context, name string)`|`(id uint, found bool, err error)`|ロール名による存在確認とID取得（登録時）|
|`FindByID`|`(ctx context.Context, id uint)`|`(name string, found bool, err error)`|IDによるロール名取得（`GetCurrentUserUseCase`用）|

`HighSchoolRepository`・`GradeRepository`は登録編のみが使うため、登録編で定義する。

## Domain Service

### CredentialVerificationService（`domain/service/credential_verification_service.go`）

- struct名: `CredentialVerificationService`
- 依存: `PasswordHasher`（同ファイルまたは`password_hasher.go`で定義するinterface）
- メソッドシグネチャ:
  - `NewCredentialVerificationService(hasher PasswordHasher) *CredentialVerificationService`
  - `(s *CredentialVerificationService) Verify(account *entity.Account, raw valueobject.RawPassword) error`
- 責務: 入力された`RawPassword`とAccountのパスワードハッシュを照合し、失敗時は`domain/errors.ErrInvalidCredentials`相当を返す

### PasswordHasher（`domain/service/password_hasher.go`）

- interface名: `PasswordHasher`
- メソッドシグネチャ:
  - `Hash(raw string) (string, error)`
  - `Verify(hash, raw string) (bool, error)`

> **②からの補足**: コーディング規約「7. インターフェース」の「利用側で定義する」方針に従い、利用側であるdomain/serviceパッケージにinterfaceを定義した（②に配置場所の明記はないため推測）。

## Domain Error

`domain/errors/errors.go`に、`errors.New`によるセンチネルエラー変数として定義する（②「17. Error設計」のDomain Errorに対応）。

|変数名|発生条件|
|-|-|
|`ErrInvalidCredentials`|メールアドレスまたはパスワードが不一致（ユーザー不存在の場合も同一エラーとして扱う）|
|`ErrInvalidRole`|指定されたロール名が存在しない|
|`ErrRegistrationRequirementNotMet`|ロール別必須項目（高校・学年・生徒コード）の欠如、または指定IDが実在しない|
|`ErrPasswordConfirmationMismatch`|パスワードと確認用パスワードが一致しない|
|`ErrResetTokenNotFound`|リセットトークンに一致するアカウントが存在しない|
|`ErrResetTokenExpired`|リセットトークンが有効期限切れ|
|`ErrResetTokenNotIssued`|リセットトークンが発行されていない状態での消費操作|
|`ErrProvisionalAccountNotFound`|生徒コードに一致する仮アカウントが存在しない|
|`ErrProvisionalAccountAlreadyActivated`|生徒コードに一致するアカウントが既に有効化済み|
|`ErrStudentNumberSchoolMismatch`|生徒コードに含まれる学校コードが選択高校の`school_code`と不一致|

---

---

# 4. クラス図

```mermaid
classDiagram
    class Account {
        -uint ID
        -valueobject.Email Email
        -string PasswordHash
        -uint UserRoleID
        -string JTI
        -*uint HighSchoolID
        -*uint GradeID
        -*valueobject.StudentNumber StudentNumber
        -bool PasswordResetRequired
        -*time.Time ActivatedAt
        -*time.Time DeletedAt
        -*entity.PasswordResetToken ResetToken
        +NewAccount(...) (*Account, error)
        +RotateJTI(newJTI string) error
        +UpdatePasswordHash(newHash string) error
        +IssuePasswordResetToken(token string, issuedAt time.Time) error
        +ConsumePasswordResetToken() error
        +HasResetToken() bool
        +IsProvisional() bool
        +IsDeleted() bool
        +IsInvitationPending() bool
        +CompleteInvitation(roleName string) error
        +Activate(now time.Time) error
    }
    class PasswordResetToken {
        -string Token
        -time.Time IssuedAt
        +NewPasswordResetToken(token string, issuedAt time.Time) (*PasswordResetToken, error)
        +IsValid(now time.Time, period ResetTokenValidityPeriod) bool
    }
    class Email {
        <<ValueObject>>
        -string value
        +NewEmail(raw string) (Email, error)
        +String() string
    }
    class RawPassword {
        <<ValueObject>>
        -string value
        +NewRawPassword(raw string) (RawPassword, error)
        +Matches(confirmation RawPassword) bool
    }
    class ResetTokenValidityPeriod {
        <<ValueObject>>
        -time.Duration duration
        +IsExpired(issuedAt, now time.Time) bool
    }
    class SignUpRoleRequirement {
        <<ValueObject>>
        -string roleName
        +RequiresHighSchool() bool
        +RequiresGrade(highSchoolIsCSVManaged bool) bool
        +RequiresStudentNumber(highSchoolIsCSVManaged bool) bool
    }
    class StudentNumber {
        <<ValueObject>>
        -string schoolCode
        -string body
        +NewStudentNumber(raw string) (StudentNumber, error)
        +SchoolCode() string
        +MatchesSchoolCode(schoolCode string) bool
    }
    class AccountRepository {
        <<interface>>
        +FindByID(ctx, id) (*Account, error)
        +FindByEmail(ctx, email) (*Account, error)
        +FindByEmailExcludingDeleted(ctx, email) (*Account, error)
        +FindByResetToken(ctx, token) (*Account, error)
        +FindByStudentNumber(ctx, studentNumber) (*Account, error)
        +Create(ctx, params) (*Account, error)
        +ActivateProvisionalAccount(ctx, accountID, params) error
        +UpdateJTI(ctx, accountID, newJTI) error
        +SaveResetToken(ctx, accountID, token, issuedAt) error
        +UpdatePasswordAndConsumeResetToken(ctx, accountID, newHash, passwordResetRequired) error
    }
    class UserRoleRepository {
        <<interface>>
        +FindByName(ctx, name) (uint, bool, error)
        +FindByID(ctx, id) (string, bool, error)
    }
    class HighSchoolRepository {
        <<interface>>
        +Exists(ctx, id) (bool, error)
        +FindByID(ctx, id) (*HighSchoolSummary, error)
    }
    class GradeRepository {
        <<interface>>
        +Exists(ctx, id) (bool, error)
        +FindByID(ctx, id) (*GradeSummary, error)
    }
    class CredentialVerificationService {
        <<DomainService>>
        +Verify(account, rawPassword) error
    }
    class RegistrationEligibilityPolicy {
        <<DomainService>>
        +Validate(requirement, highSchoolID, gradeID, studentNumber, highSchoolExists, gradeExists, isCSVManaged) error
    }
    class ProvisionalAccountActivationPolicy {
        <<DomainService>>
        +Authorize(account, studentNumber, highSchoolSchoolCode) error
    }
    class PasswordResetLifecyclePolicy {
        <<DomainService>>
        +CanConsume(token, now, period) error
    }

    Account "1" *-- "0..1" PasswordResetToken : 保持
    Account --> Email : 保持
    Account --> StudentNumber : 保持(任意)
    PasswordResetToken --> ResetTokenValidityPeriod : 有効期間を参照
    CredentialVerificationService ..> Account : 照合
    RegistrationEligibilityPolicy ..> SignUpRoleRequirement : 判定に利用
    ProvisionalAccountActivationPolicy ..> StudentNumber : 学校コード照合
    ProvisionalAccountActivationPolicy ..> Account : 仮アカウント状態判定
    PasswordResetLifecyclePolicy ..> PasswordResetToken : 状態判定
    AccountRepository ..> Account : 入出力
```

②のクラス図（②9章）は業務概念の関係を示すのみだが、本図はGoのstruct/interfaceとして実際に定義するフィールド・メソッドシグネチャまで具体化した。UserRole/HighSchool/Grade・profile Context参照（`ProfileReader`）はRepository/Interfaceの依存関係として6章（Application層設計）・8章（Infrastructure層設計）で扱う。

---

---

# 5. 状態遷移図

## Account（招待状態）

```mermaid
stateDiagram-v2
    [*] --> InvitationPending : 招待による作成(本Context外。userの作成操作)
    [*] --> InvitationCompleted : NewAccount(通常登録=自己登録)
    InvitationPending --> InvitationCompleted : CompleteInvitation(roleName)(ChangePasswordUseCase。管理者・教員・生徒)
    InvitationPending --> InvitationCompleted : Activate()(RegisterUseCaseの仮アカウント有効化分岐)

    note right of InvitationPending
        管理者は、パスワードを設定しても
        招待待ちのまま遷移しない
    end note
```

遷移条件:

- `InvitationPending`は`Account.IsInvitationPending()`（`PasswordResetRequired == true`）が`true`であることに対応する。`user` Contextが招待による作成時に設定するものであり、招待による作成の起点は本Contextではない
- `InvitationCompleted`は`PasswordResetRequired == false`であることに対応する。通常登録（自己登録）で作成されたアカウントと、招待待ちにしない指定で`user` Contextが作成した教員は、この状態で始まる
- `InvitationPending`から`InvitationCompleted`への遷移は次の2つである
  - `Account.CompleteInvitation(roleName)`: `ChangePasswordUseCase`が、トークンの有効性判定の後、永続化の前に、アカウントのロール名（`UserRoleRepository.FindByID`で取得）を渡して呼ぶ。`roleName`が`teacher`または`student`の場合のみ`PasswordResetRequired`を偽にし、`admin`の場合は変更しない。結果は`AccountRepository.UpdatePasswordAndConsumeResetToken`の`passwordResetRequired`引数として、パスワードハッシュ更新・トークン消費と同一のトランザクションで永続化される。パスワードリセットと招待メールのリンクからのパスワード設定は同じ操作であるため、どちらの経路でもこの遷移が起きる
  - `Account.Activate(now)`: 下記「Account（仮登録・有効化状態）」の遷移
- `InvitationCompleted`から`InvitationPending`へ戻る遷移はない
- `Provisional`（下記）は、`InvitationPending`のうち`ActivatedAt == nil`のアカウントである。`CompleteInvitation`によって`InvitationCompleted`になったアカウントは、`ActivatedAt == nil`でも`PasswordResetRequired == false`のため`Provisional`ではなく、`ProvisionalAccountActivationPolicy.Authorize`が`ErrProvisionalAccountAlreadyActivated`を返す（②「6. Entity設計」）

## Account（仮登録・有効化状態）

```mermaid
stateDiagram-v2
    [*] --> Active : NewAccount(通常登録, ActivatedAtは未設定のまま)
    [*] --> Provisional : 生徒アカウントの招待による作成(本Context外。userのCreateStudentAccount)
    Provisional --> Active : Activate()(RegisterUseCaseの仮アカウント有効化分岐)
```

遷移条件:

- `Provisional`は`Account.IsProvisional()`（`PasswordResetRequired == true && ActivatedAt == nil`）が`true`であることに対応する
- `Active`は`Account.IsProvisional()`が`false`であることに対応する。通常登録（自己登録）で作成されたアカウントは`ActivatedAt == nil`のまま`Active`で始まり（`AccountRepository.Create`は`activated_at`を設定しない）、`Activate(now)`を経たアカウントは`ActivatedAt`が設定された`Active`になる。`Active`かどうかを`ActivatedAt`の有無で判定してはならない
- `Provisional`から`Active`への遷移は`Account.Activate(now)`が担い、`ProvisionalAccountActivationPolicy.Authorize`による事前判定（学校コード一致等）を通過した場合のみ呼び出される
- `Provisional`状態のアカウントの作成自体（`[*] --> Provisional`）は本Contextの操作範囲外であり、`user` Contextの`CreateStudentAccount`（生徒CSVインポート・教師による生徒の単体登録が呼ぶ）が担う

## PasswordResetToken

```mermaid
stateDiagram-v2
    [*] --> Issued : IssuePasswordResetToken()
    Issued --> Expired : ResetTokenValidityPeriod.IsExpired()がtrueと判定
    Issued --> Consumed : ConsumePasswordResetToken()
    Expired --> [*]
    Consumed --> [*]
```

遷移条件:

- `Issued`は`RequestPasswordResetUseCase`が対象アカウントを`AccountRepository.FindByEmailExcludingDeleted`で発見できた場合にのみ`Account.IssuePasswordResetToken`経由で生成される
- `Expired`は`PasswordResetToken.IsValid` / `PasswordResetLifecyclePolicy.CanConsume`が発行から`ResetTokenValidityPeriod`を超えたと判定した時点で扱われる（永続化された専用ステータス値は持たない）
- `Consumed`は`Account.ConsumePasswordResetToken()`によりトークン・発行日時をクリアすることで表現される

`jti`は単純な値のローテーションであり、複数の状態を持つ概念ではないため状態遷移図としては可視化しない。

---

---

# 6. Infrastructure層設計（共通）

本書は、複数のスライス編が使うInfrastructureを扱う。スライス固有のもの（`JWTTokenIssuer`・`SecureTokenGenerator`・`ProfileReaderImpl`・`HighSchoolRepositoryImpl`・`GradeRepositoryImpl`・`AuthNotifier`）は、各スライス編で定義する。

## Repository実装

### AccountRepositoryImpl（`infrastructure/repository/account_repository.go`）

- 実装struct名: 非公開struct（例: `accountRepository`）+ コンストラクタ`NewAccountRepository(db *gorm.DB) repository.AccountRepository`（規約「8. 命名規約（アーキテクチャレベル）」により`Impl`接尾辞を付けない）
- 対応するGORMモデル: `gormmodel.UserModel`（`infrastructure/persistence/gorm/user_model.go`）
- 各メソッドで発行するクエリ内容:

|メソッド|条件|備考|
|-|-|-|
|`FindByID`|`id = ?`で1件検索|
|`FindByEmail`|`email = ?`で1件検索|論理削除済み行も対象に含めるため`Unscoped()`を明示的に付与する（後述の②からの補足参照）|
|`FindByEmailExcludingDeleted`|`email = ?`で1件検索|`Unscoped()`を付けない（GORM標準の`gorm.DeletedAt`により`deleted_at IS NULL`が自動付与され、退会済みが自然に除外される）|
|`FindByResetToken`|`reset_password_token = ?`で1件検索|
|`FindByStudentNumber`|`student_number = ?`で1件検索|
|`Create`|`users`テーブルへ1件INSERT相当の作成|`CreateAccountParams`の全フィールドを反映する。`activated_at`は設定しない（未設定のまま作成する）。自己登録のみで使用する（招待による作成は`user` Contextが行う）|
|`ActivateProvisionalAccount`|`id = ?`条件で`email`・`encrypted_password`・`name`・`name_kana`・`activated_at`・`password_reset_required`（falseへ）を更新|
|`UpdateJTI`|`id = ?`条件で`jti`カラムのみ更新|
|`SaveResetToken`|`id = ?`条件で`reset_password_token`, `reset_password_sent_at`を更新|
|`UpdatePasswordAndConsumeResetToken`|`id = ?`条件で`encrypted_password`を更新し、同時に`reset_password_token`, `reset_password_sent_at`をNULLへ更新し、引数`passwordResetRequired`の値を`password_reset_required`へ反映する|`ChangePasswordUseCase`が`Account.CompleteInvitation`適用後の値を渡す（教員・生徒は偽。管理者は現在の値のまま）。3つの更新は1回のUPDATEで行う|

- Entity ⇔ GORMモデルの変換方針: `gormmodel.UserModel`から`entity.Account` / `entity.PasswordResetToken`への変換関数（`toEntity`）、逆方向の変換関数（`toModel`）をrepository実装内の非公開関数として用意する。`Email`変換時は`valueobject.NewEmail`、`StudentNumber`変換時は`valueobject.NewStudentNumber`を通し、DBに不正な形式が入っていた場合はInfrastructure Errorとして扱う。

> **②からの補足**: ログイン用`FindByEmail`が退会済みアカウントを対象に含めるかどうかは②「11. Repository設計」でも明確化されていない（「ログイン時の検索についても同様に退会済みを除外すべきかは現行仕様書に明記がないため、本書ではパスワードリセット対象検索に限定して扱う」）。本書では、Gorm規約「0. 本プロジェクトでの採用方針」の論理削除方針（`gorm.DeletedAt`によりFind系は標準で`deleted_at IS NULL`を自動付与）を踏まえ、ログイン検索は現状の挙動（退会済みを対象から除外しない）を変更しないために`Unscoped()`を明示的に付与し、パスワードリセット対象検索（`FindByEmailExcludingDeleted`）のみGORM標準の自動除外に委ねる設計とした。これは②が意図的に未決定としている論点にGo実装として最小限の解を与えるための判断であり、①未提供のため最終的な挙動確認はできない（推測）。

### UserRoleRepositoryImpl

- 対応するGORMモデル: `gormmodel.UserRoleModel`（本Contextが所有する`user_roles`テーブル）に対し、`id = ?`（または`name = ?`）による存在確認・ロール名取得クエリのみを発行する。業務ルール判定は持たない。

## 認証技術アダプタ

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`BcryptPasswordHasher`（`infrastructure/security/bcrypt_password_hasher.go`）|`domain/service.CredentialVerificationService`、`RegisterUseCase`、`ChangePasswordUseCase`|`domain/service.PasswordHasher`インターフェースを実装する。ハッシュアルゴリズムの選定（bcrypt等）は①未提供のため参照不可。既存ユーザーの再ログインに影響しないよう、Rails現行のハッシュ方式と一致させる必要がある|

`JWTTokenIssuer`はセッション編、`SecureTokenGenerator`はセッション編（`JTIGenerator`）とパスワードリセット編（`ResetTokenGenerator`）で扱う。

## 共通のResponse DTO

|struct名|フィールドと型|
|-|-|
|`UserResponse`|`ID uint`, `Name string`, `NameKana string`, `Email string`, `ProfileCompleted bool`, `UserPersonalInfo any`, `UserRole any`, `HighSchool any`, `Address any`, `Grade any`（②「19. API仕様」のレスポンス構造を維持。ログイン・登録・`GET /api/v1/me`のすべてで共通のレスポンス構造として利用する。ネストされた各フィールドの具体的な型はprofile/master-data各Contextの③文書と合わせて確定する）|
|`MessageResponse`|`Message string`|

> **②からの補足**: 旧版は`LoginResponse` / `RegisterResponse`を別structとして定義していたが、②「19. API仕様」が「ログイン・登録・`GET /api/v1/me`のResponseはすべて同形式のユーザー情報一式」と明記しているため、`UserResponse`という単一のstructに統合した（②の記載をより正確に反映するための実装判断）。

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
|GET|`/api/v1/me`|`CurrentUserHandler.GetCurrentUser`（認証Middleware必須）|

---

---

# 7. Transaction実装方針

## Transaction開始箇所

②「14. Transaction設計」により、UseCaseの開始時（読み取り専用UseCaseを除く）にトランザクションを開始する。実装機構はアーキテクチャ規約「11. Transaction実装パターン（TransactionManager）」の標準パターンにそのまま従う（本機能個別の推測を要しない）。

- interface名: `TransactionManager`（`application/usecase/transaction_manager.go`）
- メソッドシグネチャ: `WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error`
- 実装: `infrastructure/repository/transaction_manager.go`でGORMの`*gorm.DB.Transaction`を用いて実装し、`fn`に渡す`ctx`にトランザクション用の`*gorm.DB`を保持させる（各Repository実装はこの`ctx`からトランザクション用DBを取得する）

## Transaction終了箇所（Commit / Rollback条件）

|UseCase|終了箇所|
|-|-|
|LogoutUseCase|`AccountRepository.UpdateJTI`完了時点でCommit、エラー時Rollback|
|RegisterUseCase（通常登録パス）|`AccountRepository.Create`完了時点でCommit、途中の存在確認失敗・作成失敗時はRollback|
|RegisterUseCase（仮アカウント有効化パス）|`AccountRepository.ActivateProvisionalAccount`完了時点でCommit、`ProvisionalAccountActivationPolicy.Authorize`違反・対象未検出時はRollback（`AuthNotifier`呼び出しはCommit後に行い、トランザクションのスコープ外とする）|
|RequestPasswordResetUseCase|`AccountRepository.SaveResetToken`完了時点でCommit（対象ユーザーが存在する場合のみトランザクションを使用）|
|ChangePasswordUseCase|`AccountRepository.UpdatePasswordAndConsumeResetToken`（パスワードハッシュ更新・トークン消費・招待状態の反映）完了時点でCommit、トークン検証失敗時はトランザクションを開始しない|
|LoginUseCase / VerifyResetTokenUseCase / GetCurrentUserUseCase|トランザクションを使用しない（読み取りのみ）|

## 複数Repositoryにまたがる場合の扱い

- RegisterUseCase（通常登録パス）は`UserRoleRepository` / `HighSchoolRepository` / `GradeRepository`（参照系、トランザクション不要）と`AccountRepository.Create`（書き込み）を扱うが、参照確認から作成までを1つの`TransactionManager.WithinTransaction`スコープ内で実行し、途中の不整合（不正なロール・高校・学年の組み合わせでの作成）を防ぐ
- RegisterUseCase（仮アカウント有効化パス）は`HighSchoolRepository.FindByID`（学校コード取得、トランザクション不要な参照）を先に実行した上で、`AccountRepository.FindByStudentNumber`から`ActivateProvisionalAccount`までを1つの`TransactionManager.WithinTransaction`スコープ内で実行する
- ChangePasswordUseCaseは`UserRoleRepository.FindByID`（ロール名の取得、参照のためトランザクション不要）を先に実行し、`account.CompleteInvitation`の結果を、`AccountRepository.UpdatePasswordAndConsumeResetToken`の1回の呼び出しでパスワードハッシュ更新・トークン消費とともに永続化する。`users`への書き込みは1操作であるため、パスワードだけが更新され招待待ちが残る中間状態は生じない

---

---

# 8. Validation実装方針

## Presentation

|フィールド|struct名|バリデーションタグ|エラーメッセージ方針|
|-|-|-|-|
|`Email`|`LoginRequest`|`binding:"required,email"`|「入力内容を確認してください」|
|`Password`|`LoginRequest`|`binding:"required"`|「入力内容を確認してください」|
|`Email`|`SignUpUserParams`|`binding:"required,email"`|「メールアドレスの形式が不正です」|
|`Password`/`PasswordConfirmation`/`Name`/`NameKana`|`SignUpUserParams`|`binding:"required"`|「入力内容を確認してください」|
|`StudentNumber`|`SignUpUserParams`|`binding:"omitempty"`＋カスタム形式チェック|「生徒コードの形式が不正です」|
|`Email`|`PasswordResetRequestRequest`|`binding:"required,email"`|「メールアドレスの形式が不正です」|
|`ResetPasswordToken`/`Password`/`PasswordConfirmation`|`ChangePasswordRequest`|`binding:"required"`|「入力内容を確認してください」|
|`ResetPasswordToken`|`VerifyResetTokenRequest`|`binding:"required"`|「入力内容を確認してください」|

## 業務ルール検証

- Entity／Value Object生成時に検証する内容: `valueobject.NewEmail`（形式）、`valueobject.NewRawPassword`（最小文字数等）、`valueobject.NewStudentNumber`（ハイフン区切り形式）、`entity.NewAccount`（不変条件）
- UseCase内で判定する業務ルール:
  - `RegisterUseCase`（通常登録）: `RegistrationEligibilityPolicy.Validate`によるロール別必須項目（高校・学年・生徒コード）の判定
  - `RegisterUseCase`（仮アカウント有効化）: `ProvisionalAccountActivationPolicy.Authorize`による有効化可否判定
  - `ChangePasswordUseCase` / `VerifyResetTokenUseCase`: `PasswordResetLifecyclePolicy.CanConsume`によるトークン有効性判定
  - 全登録・リセット系: `RawPassword.Matches`によるパスワード確認一致判定

---

---

# 9. Authorization実装方針

②「16. Authorization設計」をそのまま実装レベルに落とし込む。

## Middleware

- Cookie（`access_token`）からJWTを取得し、署名検証・`jti`の一致確認を行って`current_user`（AccountID等）をGinの`*gin.Context`に設定する
- ログイン・登録（student/teacher/admin）・パスワードリセット系（request/reset/verify）エンドポイントは認証チェック対象外とする（Ginのルートグループを分離し、Middlewareを適用しない）
- ログアウト・`GET /api/v1/me`のエンドポイントのみ、`current_user`の存在を前提とするMiddlewareを適用する

## Handler

- Cookie（`access_token`）のHTTP Only Cookie設定・削除操作を担当する
- 資格情報の照合、トークンの有効性判定、仮アカウント有効化可否判定は持たせない（UseCaseへ委譲する）

## UseCase

- `LogoutUseCase`は`current_user`（AccountID）が存在することを前提としてセッション無効化を実行する
- `RegisterUseCase` / パスワードリセット系UseCaseは、認証済みユーザーの有無に依存しない業務ルールの検証のみを行う
- `GetCurrentUserUseCase`は`current_user`を前提とし、常に本人自身の情報のみを取得対象とする（他ユーザーの情報は取得できない）

## Domain

- `Account` / `PasswordResetToken`が、資格情報照合・セッション無効化・トークン消費・仮アカウント有効化可否という「本人確認そのもの」に関わる判定を担う（`CredentialVerificationService`, `PasswordResetLifecyclePolicy`, `ProvisionalAccountActivationPolicy`経由）

---

---

# 10. Error実装方針

## Domain Error → Application Errorへの変換方針

アーキテクチャ規約「12. Error変換パターン（AppError）」の標準パターンに従う。各UseCaseは、`domain/errors`のセンチネルエラーを`errors.Is`で判定し、`application/apperror`の`AppError`を実装した型へ変換して返す。Domain Error自体にStatusCode/LogLevelを持たせない。`RequestPasswordResetUseCase`のみ、内部的な「対象ユーザー不存在」を外部エラーとして伝播させず、常に固定の成功結果（`dto.RequestPasswordResetResult`）に変換する（②「17. Error設計」のApplication Errorの方針どおり）。

## Application Error → HTTPレスポンスへの変換方針

Gin規約「8. エラーハンドリングミドルウェア」の集中エラーハンドリングミドルウェアに変換ロジックを集約する。Handlerは`c.Error(err)`でエラーを登録するのみとし、個々のHandlerでStatus Codeを判定しない。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrInvalidCredentials`|Domain|401|
|`ErrInvalidRole`|Domain|422|
|`ErrRegistrationRequirementNotMet`|Domain|422|
|`ErrPasswordConfirmationMismatch`|Domain|422|
|`ErrResetTokenNotFound`|Domain|422|
|`ErrResetTokenExpired`|Domain|422|
|`ErrResetTokenNotIssued`|Domain|422|
|`ErrProvisionalAccountNotFound`|Domain|422|
|`ErrProvisionalAccountAlreadyActivated`|Domain|422|
|`ErrStudentNumberSchoolMismatch`|Domain|422|
|`current_user`未設定（`GET /api/v1/me`・logout）|Middleware|401|
|Request DTOバリデーションエラー|Presentation|422|
|DB接続失敗・クエリ失敗|Infrastructure|500|
|JWT署名・発行失敗|Infrastructure|500|
|メール送信基盤連携失敗（`AuthNotifier`）|Infrastructure|非同期処理のためHTTPレスポンスには影響させない（規約13章のベストエフォート方針。エラーはログ出力のみ）|

## Infrastructure Errorのハンドリング方針

Infrastructure層（Repository実装・security・mail）で発生したエラーは、`fmt.Errorf`でラップしてApplication層へ伝播させ、Presentation層側の集中エラーハンドリングミドルウェアで未分類のエラーとして500に変換する。業務エラー（Domain Error）と技術的障害（Infrastructure Error）を`errors.Is` / `errors.As`で明確に区別する（コーディング規約「18. エラーハンドリング」）。

---

---

# 11. GORM / DBクエリ設計

②「20. DB設計方針」により、既存Rails DBをそのまま継続利用し、スキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

### UserModel（`infrastructure/persistence/gorm/user_model.go`）

- 対応テーブル: `users`（既存Railsスキーマ）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー（Gorm規約のデフォルト`ID`フィールドをそのまま利用）|
|`Email`|`email`|
|`EncryptedPassword`|`encrypted_password`|Rails Devise由来のカラム名をそのまま踏襲するため、フィールド名と規約上のsnake_case変換結果が一致する|
|`JTI`|`jti`|Gorm規約のデフォルトsnake_case変換では連続大文字の扱いが曖昧になり得るため、`gorm:"column:jti"`タグで明示的に指定する|
|`ResetPasswordToken`|`reset_password_token`|
|`ResetPasswordSentAt`|`reset_password_sent_at`|
|`UserRoleID`|`user_role_id`|
|`HighSchoolID`|`high_school_id`|
|`GradeID`|`grade_id`|
|`StudentNumber`|`student_number`|`*string`（未設定時NULL）|
|`PasswordResetRequired`|`password_reset_required`|`bool`|
|`ActivatedAt`|`activated_at`|`*time.Time`|
|`Name`|`name`|
|`NameKana`|`name_kana`|
|`DeletedAt`|`deleted_at`|`gorm.DeletedAt`型。Gorm規約「0. 本プロジェクトでの採用方針」の論理削除方針に従う|
|`CreatedAt`|`created_at`|Gorm規約のタイムスタンプ自動トラッキングに従う|
|`UpdatedAt`|`updated_at`|同上|

- テーブル名: 構造体名`UserModel`はGorm規約のデフォルト複数形変換では`user_models`となり実テーブル名`users`と一致しないため、`Tabler`インターフェース（`func (UserModel) TableName() string { return "users" }`）を実装してテーブル名を明示的に指定する。

### UserRoleModel

- `UserRoleModel`は`user_roles`テーブル（存在確認・ロール名取得のみに利用するため`ID`, `Name`程度の最小フィールドで定義する）。`high_schools` / `grades`テーブルに対応するGORMモデルは、master-data Contextが正規の所有者であるため本Contextでは定義しない（8章参照）。

## 主要クエリの条件・ソート・ページネーション方針

|Repository/Store|メソッド|対象テーブル|条件|結合|
|-|-|-|-|-|
|AccountRepository|`FindByID`|users|`id = ?`|なし|
|AccountRepository|`FindByEmail`|users|`email = ?`（`Unscoped()`付与、退会済みも対象）|なし|
|AccountRepository|`FindByEmailExcludingDeleted`|users|`email = ?`（GORM標準の`deleted_at IS NULL`自動付与）|なし|
|AccountRepository|`FindByResetToken`|users|`reset_password_token = ?`|なし|
|AccountRepository|`FindByStudentNumber`|users|`student_number = ?`|なし|
|UserRoleRepository|`FindByName` / `FindByID`|user_roles|`name = ?` / `id = ?`|なし|
|HighSchoolRepository / GradeRepository|`Exists` / `FindByID`|（master-data Context経由。本Contextはhigh_schools / gradesへ直接クエリしない）|`masterdata.ExistsHighSchool` / `masterdata.FindHighSchoolByID` / `masterdata.ExistsGrade` / `masterdata.FindGradeByID`の呼び出し|なし|

AccountRepository・UserRoleRepositoryはいずれも一意検索または存在確認のみであり、ページネーション・ソートは不要（本機能に一覧取得・集計クエリは存在しない）。HighSchoolRepository / GradeRepositoryはSQLを直接発行せず、master-data Context側のapplication層関数呼び出しに委譲する。

## 既存Schemaへの変更

②「20. DB設計方針」により変更なし。SQL文そのものは本書に記載しない。

---

---

# 12. テストケース設計（本書の担当分）

## Domain Test

|対象|テストケース|
|-|-|
|`CredentialVerificationService.Verify`|パスワード一致時に成功する／不一致時に`ErrInvalidCredentials`相当を返す|
|`PasswordResetToken.IsValid`|発行直後は有効／有効期間経過後は無効と判定する|
|`ResetTokenValidityPeriod.IsExpired`|境界値（有効期間ちょうど）での判定|
|`RegistrationEligibilityPolicy.Validate`|student/teacherで高校・学年未指定時にエラー、adminで高校・学年未指定でも成功する、生徒コード管理対象校のstudentで`grade_id`の代わりに`student_number`が必須になる|
|`SignUpRoleRequirement.RequiresHighSchool` / `RequiresGrade` / `RequiresStudentNumber`|ロール・`csv_managed`区分ごとの真偽値が②の要件どおりであること|
|`StudentNumber.NewStudentNumber` / `MatchesSchoolCode`|ハイフン区切り形式の正当性判定、学校コードの一致／不一致判定|
|`RawPassword.Matches`|一致／不一致の判定|
|`Email.NewEmail`|不正な形式でエラーになること|
|`PasswordResetLifecyclePolicy.CanConsume`|未発行・期限切れ・有効の3状態での判定|
|`Account.IsProvisional`|`PasswordResetRequired`と`ActivatedAt`の組み合わせによる判定|
|`Account.Activate`|仮登録状態からの遷移成功／既に有効化済みの場合はエラーになること|
|`Account.IsInvitationPending`|`PasswordResetRequired`が真で`true`、偽で`false`を返すこと|
|`Account.CompleteInvitation`|`teacher` / `student`で`PasswordResetRequired`が偽になる／`admin`では真のまま変化しない／既に偽の場合は偽のまま変化しない／`roleName`が空の場合はエラーになる／`ActivatedAt`は変更されない|
|`ProvisionalAccountActivationPolicy.Authorize`|仮アカウント不存在・既に有効化済み・学校コード不一致のそれぞれでエラーが区別されること|

## Repository Test

|対象|テストケース|
|-|-|
|`AccountRepositoryImpl.FindByEmail`|退会済みアカウントも検索対象に含まれること|
|`AccountRepositoryImpl.FindByEmailExcludingDeleted`|退会済みアカウントが検索対象から除外されること|
|`AccountRepositoryImpl.FindByStudentNumber`|存在する/しない生徒コードでの検索結果|
|`AccountRepositoryImpl.ActivateProvisionalAccount`|氏名・氏名カナ・パスワードハッシュ・`activated_at`・`password_reset_required`が正しく更新されること|
|`AccountRepositoryImpl.UpdateJTI`|更新後に`jti`が反映されること|
|`AccountRepositoryImpl.UpdatePasswordAndConsumeResetToken`|パスワードハッシュ更新とトークンクリアが同時に反映されること／`passwordResetRequired`に偽を渡した場合に`password_reset_required`が偽へ更新され、真を渡した場合は真のまま維持されること|
|`UserRoleRepositoryImpl.FindByName` / `FindByID`、`HighSchoolRepositoryImpl.Exists` / `FindByID`、`GradeRepositoryImpl.Exists` / `FindByID`|存在する/しないIDでの判定結果|

スライス編が担当するUseCase Test・Handler Test・Integration Testは、各スライス編に記載する。

---

# 13. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に列挙する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ディレクトリ名を`internal/authentication`とした|アーキテクチャ規約「4. Bounded Context構成」の「ディレクトリ構成」の命名規則（Context名のkebab-caseを`_`区切りに置き換え、短縮しない）による。Context名から機械的に決まるため、②文書での対応付けは不要|規約適用による決定（推測ではない）|
|Account Entityに`Name`/`NameKana`を持たせず、Repository作成・有効化入力（`CreateAccountParams` / `ActivateAccountParams`）のみで受け渡す構成とした|②「5. Aggregate設計」がプロフィール情報をAggregate外としている方針と、②「6. Entity設計」の「有効化時に入力内容で自身を更新する」という責務記述の両方を、矛盾なく実装に落とし込むための判断|②の記載同士の整合を取るための判断（推測ではない）|
|`SignUpRoleRequirement.RequiresGrade` / `RequiresStudentNumber`が`highSchoolIsCSVManaged bool`を引数に取る構成とした|②「7. Value Object設計」の「生徒コード管理対象校の場合、`grade_id`の代わりに`student_number`を必須とする」分岐を表現するために必要だが、②はメソッドシグネチャまでは規定していない|推測|
|`StudentNumber`の形式不正をDomain Errorのセンチネル変数ではなく通常のerrorとして扱った|②「15. Validation設計」がフォーマットチェックをPresentation層の責務としており、Domain側は多重防御として位置づけたため。既存の`Email`/`RawPassword`の扱いと一貫させた|推測|
|`ProvisionalAccountActivationPolicy.Authorize`が`account *entity.Account`の`nil`を許容する引数設計とした|「該当する仮アカウントが見つからない」ケースの判定をPolicy内に一元化するための実装判断。②「8. Domain Service」はシグネチャを規定していない|推測|
|`PasswordResetTokenIssued` / `AccountActivated`イベントの実際のディスパッチを、`jobs`テーブルではなくgoroutine直接起動（`AuthNotifier`）で実装した|アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の「処理の分類」表が、パスワードリセットメール送信・有効化完了メール送信を「ベストエフォートで良い処理」に分類しているため、規約が定める標準パターンにそのまま従った|規約に基づく判断（推測ではない）|
|`TransactionManager` / `AppError`の実装パターンを個別に推測せず、アーキテクチャ規約「11章」「12章」の標準パターンをそのまま採用した|規約制定（2026-08）により、旧版が個別に「推測」としていた内容がプロジェクト全体の標準として確定したため|規約に基づく判断（推測ではない）|
|`RawPassword`の最小文字数、`ResetTokenValidityPeriod`の具体的な有効期間の値を確定しなかった|②自身がこれらを「推測」と明記し、Devise標準ポリシーを踏襲する前提としているのみで具体的な数値がない。①未提供のため、本書でも数値を確定できない|①未提供のため参照不可（②の推測を維持）|
|ログイン用`FindByEmail`は退会済みアカウントも対象に含め（`Unscoped()`使用）、パスワードリセット用`FindByEmailExcludingDeleted`のみGORM標準の論理削除自動除外に委ねる設計とした|②「11. Repository設計」がログイン時の退会済み除外要否を明示的に未決定としているため、現状の挙動を変えない（除外しない）解釈を採用した|推測|
|`ProfileReader`インターフェース（profile Context参照）を`application/usecase`層に具体的なメソッドシグネチャとして定義した|旧版は「概念のみに留め確定させない」としていたが、本タスクで並行してprofile Context側の③文書（プロフィール管理機能_Go実装仕様書）を作成するため、参照可能なデータ構造を踏まえて具体化した|推測（profile Context側の③文書と合わせて最終確定する）|
|HighSchoolRepositoryImpl / GradeRepositoryImplを、GORMモデルの直接参照ではなくmaster-data Context（共通マスタ参照機能）が公開する`ExistsHighSchool` / `FindHighSchoolByID` / `ExistsGrade` / `FindGradeByID`関数の呼び出しとして実装した|②「11. Repository設計」はHighSchool/GradeRepositoryを「参照専用」とするのみで実装方式までは指定していない。アーキテクチャ規約「5. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従い、本タスクで並行して確定した共通マスタ参照機能_Go実装仕様書側の関数シグネチャに合わせて実装方式を確定した|②からの補足（③間の整合を取るための具体化であり、新しい業務ルールの追加ではない）|
|`Account.CompleteInvitation`が`roleName string`を引数に取り、`ChangePasswordUseCase`が`UserRoleRepository.FindByID`で取得したロール名を渡す構成とした|②「6. Entity設計」「10. 状態遷移図」が、パスワード設定の成功時に管理者・教員・生徒の招待待ちを解消する遷移をAccountの責務と定めているが、`Account`は`UserRoleID`のみを保持し、②はメソッドシグネチャを規定していないため|推測（②の遷移規則を実装に落とし込むための判断）|
|`AccountRepository.UpdatePasswordAndConsumeResetToken`が`passwordResetRequired bool`引数を受け取り、パスワードハッシュ更新・トークン消費・招待状態の反映を1操作とした（`NewAccount`も`passwordResetRequired`引数を受け取る）|②「5. Aggregate設計」「14. Transaction設計」が、パスワード更新・トークン消費・招待待ちの解消を1トランザクションと定めているため。Account EntityのフィールドにあるPasswordResetRequiredを、ファクトリと永続化の双方で受け渡す必要がある|②の記載を実装に落とし込むための判断（推測）|
|`VerifyResetToken`のレスポンスを個別エラーコードではなく`Valid: bool`を含む200レスポンスとした|②のStatus Code一覧に`password/verify`固有のエラーステータスの明記がないため、既存の確認系エンドポイントの一般的な設計として判断した|推測|
|`LoginResponse` / `RegisterResponse`を`UserResponse`という単一structに統合した|②「19. API仕様」がログイン・登録・`GET /api/v1/me`のレスポンスをすべて同形式としているため、旧版の別struct定義を統合した|②の記載をより正確に反映するための判断（推測ではない）|

上記以外の設計判断（Bounded Context・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
