# 認証機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - ユーザーのログイン、ログアウト、新規登録、パスワードリセットを提供し、認証状態を管理する。
- システム上の役割
  - ユーザーがメールアドレスとパスワードでログインできるようにする。
  - JWT を HTTP Only Cookie でクライアントに渡す。
  - 新規ユーザー登録時にユーザーロールや学校情報を正しく検証してユーザーを作成する。
  - パスワードリセットリクエストを受け付け、メール送信をトリガーする。
  - リセットトークンの有効性を検証し、パスワード変更を完了する。
- 利用者
  - `student` / `teacher` / `admin` ロールのユーザー

---

# 2. 機能一覧

|操作|概要|
|-|-|
|ログイン|メールアドレスとパスワードで認証し、JWT を Cookie に保存する|
|ログアウト|JWT を無効化し、Cookie を削除する|
|新規登録|role に応じたユーザーを作成する|
|パスワードリセットリクエスト|メールアドレスにリセットメールを送信する|
|パスワード変更|リセットトークンでパスワードを更新する|
|トークン検証|リセットトークンの有効期限を確認する|
|ログイン中ユーザー情報取得|現在ログインしているユーザー自身の情報を取得する|

ログイン中ユーザー自身の基本情報・個人情報・住所を更新する操作は「プロフィール管理機能」で提供される。

---

# 3. 業務フロー

1. ログイン時、ユーザーは `email` と `password` を送信する。
2. システムは入力項目を検証し、ユーザー存在とパスワード照合を行う。
3. 認証に成功した場合、JWT を生成して `access_token` Cookie に設定し、認証済みユーザー情報を返却する。
4. ログアウト時はユーザーの `jti` を更新し、Cookie を削除してセッションを終了する。
5. 新規登録時は `user_role_name` を検証し、必要に応じて `high_school_id` / `grade_id` をチェックしてユーザーを作成する。生徒が生徒コード（`student_number`）を入力した場合は、新規ユーザーを作成する代わりに、学校側で事前に作成済みのアカウントを本人のものとして有効化する。
6. パスワードリセットリクエストでは、退会（論理削除）していないユーザーの中から該当するユーザーを検索し、リセットトークン付きメールを送信する。
7. パスワード変更では、トークンとパスワード確認を検証し、パスワードを更新する。
8. トークン検証では、リセットトークンが存在し、有効期限内かを確認する。
9. ログイン中ユーザー情報取得では、Cookie の JWT から特定したユーザー自身の情報を取得して返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::SessionsController`|`create`|POST|`/api/v1/user/login`|ログイン処理|
|`Api::V1::SessionsController`|`destroy`|DELETE|`/api/v1/user/logout`|ログアウト処理|
|`Api::V1::RegistrationsController`|`create`|POST|`/api/v1/student/signup`|生徒登録|
|`Api::V1::RegistrationsController`|`create`|POST|`/api/v1/teacher/signup`|教師登録|
|`Api::V1::RegistrationsController`|`create`|POST|`/api/v1/admin/signup`|管理者登録|
|`Api::V1::PasswordResetsController`|`create`|POST|`/api/v1/password/reset/request`|パスワードリセットメール送信|
|`Api::V1::PasswordResetsController`|`update`|PATCH|`/api/v1/password/reset`|パスワード更新|
|`Api::V1::PasswordResetsController`|`verify`|POST|`/api/v1/password/verify`|リセットトークン検証|
|`Api::V1::UsersController`|`show`|GET|`/api/v1/me`|ログイン中ユーザー自身の情報を取得|

---

# 5. API / 処理詳細

## `Api::V1::SessionsController#create`

### 概要

メールアドレスとパスワードでログインし、JWT を HTTP Only Cookie に保存して現在のユーザー情報を返却する。

### Request

- Body
  - `email` (string, required)
  - `password` (string, required)

### 処理内容

1. `Auth::LoginForm` で `email` と `password` の存在とフォーマットを検証する。
2. `Auth::LoginService` でユーザー検索とパスワード検証を行う。
3. 認証成功時、`sign_in(user, store: false)` を実行して Devise JWT を生成する。
4. `request.env['warden-jwt_auth.token']` から取得したトークンを Cookie `access_token` に設定する。
   - `httponly: true`
   - `secure: Rails.env.production?`
   - `same_site: :lax`
   - `path: '/'`
   - `expires: 1.day.from_now`
5. `CurrentUserSerializer` でユーザー情報を返却する。

### Response

- 200 OK
- `user`:
  - `id`, `name`, `name_kana`, `email`, `profile_completed`
  - `user_personal_info`, `user_role`, `high_school`, `address`, `grade`

### Errorケース

- 422 Unprocessable Content
  - 入力検証エラー
- 401 Unauthorized
  - `Auth::LoginService::LoginError`（メールアドレスまたはパスワードが違います）

---

## `Api::V1::SessionsController#destroy`

### 概要

ログアウト処理。ユーザーの `jti` を更新して既存 JWT を無効化し、`access_token` Cookie を削除する。

### 処理内容

1. `current_user&.update!(jti: SecureRandom.uuid)` で JWT の識別子を更新する。
2. `cookies.delete(:access_token, path: '/')` で Cookie を削除する。
3. `ログアウトしました。` メッセージを返却する。

### Response

- 200 OK
- `message`: `ログアウトしました。`

---

## `Api::V1::RegistrationsController#create`

### 概要

ユーザーを新規登録する。登録は `student`, `teacher`, `admin` いずれのエンドポイントからも同一処理を実行する。

### Request

- Body
  - `user.email` (string, required)
  - `user.name` (string, optional)
  - `user.name_kana` (string, optional)
  - `user.password` (string, required)
  - `user.password_confirmation` (string, required)
  - `user.user_role_name` (string, required)
  - `user.high_school_id` (integer, required for `student` / `teacher`)
  - `user.grade_id` (integer, required for `student` かつ `student_number` 未入力の場合 / `teacher`)
  - `user.student_number` (string, optional。生徒が学校から発行された生徒コードを持つ場合に入力する)

### 処理内容

1. `Auth::SignUpForm` で入力項目を検証する。
   - `name_kana` はカタカナ形式であることを検証する。
   - `student` かつ選択した高校が生徒コード管理対象校（`csv_managed`）の場合、`student_number` の入力を必須とする。
   - `student_number` が入力されている場合、所定のフォーマット（学校コードとコード本体をハイフンで連結した形式）であることを検証する。
2. `Auth::SignUpService` で次の処理を実行する。
   - `UserRole.find_by(name: user_role_name)` を検索し、存在しない場合は `SignUpError` を返す。
   - 生徒が `student_number` を入力した場合は新規ユーザーを作成せず、生徒コードに対応する既存ユーザー（学校側で事前登録済みの仮アカウント）を検索し、生徒コードに含まれる学校コードと選択した高校が一致すること、対象ユーザーが未有効化（`password_reset_required` が true）であることを確認したうえで、入力内容で更新し `activated_at` を設定してアカウントを有効化する。有効化に伴い、元のメールアドレス宛にアカウント有効化完了のメールを送信する。
   - 上記に該当しない `student` または `teacher` の場合、`high_school_id` と `grade_id` を検証し、存在しない場合は `SignUpError` を返したうえで `User.create!` によりユーザーを新規作成する。この通常登録（自己登録）では、有効化日時（`activated_at`）は設定されない（未設定のまま作成される）。`activated_at` が設定されるのは、上記の生徒コードによる仮アカウントの有効化のときだけである。
3. `CurrentUserSerializer` で登録（または有効化）されたユーザー情報を返却する。

### Response

- 201 Created
- `user`:
  - `id`, `name`, `name_kana`, `email`, `profile_completed`
  - `user_personal_info`, `user_role`, `high_school`, `address`, `grade`

### Errorケース

- 422 Unprocessable Content
  - `Auth::SignUpService::SignUpError`
  - ActiveRecord バリデーションエラー
- 500 Internal Server Error
  - 予期せぬエラー

---

## `Api::V1::PasswordResetsController#create`

### 概要

パスワードリセットリクエストを受け取り、該当ユーザーにリセットメール送信をトリガーする。

### Request

- Body
  - `email` (string, required)

### 処理内容

1. 退会（論理削除）していないユーザーの中から、`email` に一致するユーザーを検索する。
2. `Auth::ResetPasswordService` を呼び出し、ユーザーが存在する場合はリセットトークンを生成して `SendResetPasswordEmailJob.perform_later` を実行する。
3. `パスワード変更メールを送信しました。` メッセージを返却する。
4. 例外発生時も同じメッセージを返却し、情報漏洩を防止する。

### Response

- 200 OK
- `message`: `パスワード変更メールを送信しました。`

---

## `Api::V1::PasswordResetsController#update`

### 概要

リセットトークンを使ってパスワードを更新する。

### Request

- Body
  - `password_reset.reset_password_token` (string, required)
  - `password_reset.password` (string, required)
  - `password_reset.password_confirmation` (string, required)

### 処理内容

1. `Auth::PasswordResetForm` で入力値を検証する。
2. `Auth::ChangePasswordService` で `User.reset_password_by_token` を呼び出す。
3. 正常に更新できた場合は `パスワードを更新しました。` を返却する。

### Response

- 200 OK
- `message`: `パスワードを更新しました。`

### Errorケース

- 422 Unprocessable Content
  - `ValidationError`（トークン不正、パスワード未入力、確認パスワード不一致など）

---

## `Api::V1::PasswordResetsController#verify`

### 概要

リセットトークンの有効性を確認する。

### Request

- Body
  - `reset_password_token` (string, required)

### 処理内容

1. `User.with_reset_password_token` でトークンに紐づくユーザーを検索する。
2. ユーザーが存在しない場合、`ユーザーが見つかりません。` を返す。
3. `reset_password_period_valid?` が false の場合、`トークンの有効期限が切れています。` を返す。
4. 有効な場合、`トークンは有効です。` を返す。

### Response

- 200 OK
- `message`: `トークンは有効です。`

---

## `Api::V1::UsersController#show`

### 概要

ログイン中のユーザー自身の情報を取得する。プロフィール編集画面の初期表示など、自分自身の登録情報を確認する用途で利用される。プロフィール情報の更新は「プロフィール管理機能」で提供される。

### Request

- パラメータなし（Cookie の JWT から対象ユーザーを特定する）

### 処理内容

1. Cookie の JWT から認証済みユーザーを特定する。
2. 個人情報・ロール・高校・住所（都道府県含む）を合わせて取得する。
3. `CurrentUserSerializer` でユーザー情報を返却する。

### Response

- 200 OK
- `user`:
  - `id`, `name`, `name_kana`, `email`, `profile_completed`
  - `user_personal_info`, `user_role`, `high_school`, `address`, `grade`

### Errorケース

- 401 Unauthorized
  - 未ログインの場合

---

# 6. データモデル

## `users`

- 役割: ユーザー認証とログイン情報を保持する。
- 主なカラム: `email`, `encrypted_password`, `jti`, `reset_password_token`, `reset_password_sent_at`, `user_role_id`, `high_school_id`, `grade_id`, `address_id`, `student_number`, `password_reset_required`, `activated_at`, `deleted_at`
- リレーション: `belongs_to :user_role`, `belongs_to :high_school`, `belongs_to :grade`, `belongs_to :address`, `has_one :user_personal_info`
- 業務ルール: `deleted_at` が設定されたユーザーは論理削除済みとして扱われ、パスワードリセット等の検索対象から除外される。

## `user_roles`

- 役割: ユーザーロールを管理する。
- 主なカラム: `name`

## `user_personal_infos`

- 役割: ユーザーの個人情報（電話番号・生年月日・性別）を保持する。
- 主なカラム: `user_id`, `phone_number`, `birthday`, `gender`
- リレーション: `belongs_to :user`
- ログイン・新規登録・ログイン中ユーザー情報取得のレスポンスに含まれる。詳細な更新仕様は「プロフィール管理機能」を参照。

## `addresses` / `prefectures` / `high_schools` / `grades`

- 役割: ユーザーが選択する住所・都道府県・在籍高校・学年の情報を保持する。
- ログイン・新規登録・ログイン中ユーザー情報取得のレスポンスに、ユーザーが紐づく住所・高校・学年の情報として含まれる。
- 都道府県・住所・高校・学年の一覧取得や検索の仕様は「共通マスタ参照機能」を、ユーザー自身の住所選択・更新の仕様は「プロフィール管理機能」を参照。

---

# 7. 権限制御

- `SessionsController#create` と `RegistrationsController#create` は認証不要。
- `PasswordResetsController` の `create`, `update`, `verify` も認証不要。
- `SessionsController#destroy` は `current_user` が存在することが前提。
- `UsersController#show`（`GET /api/v1/me`）は認証が必須で、ログイン中のユーザー自身の情報のみ取得できる。
- JWT 認証は `access_token` Cookie を利用する。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::SessionsController`, `Api::V1::RegistrationsController`, `Api::V1::PasswordResetsController`, `Api::V1::UsersController`|
|Form|`Auth::LoginForm`, `Auth::SignUpForm`, `Auth::PasswordResetForm`|
|Service|`Auth::LoginService`, `Auth::SignUpService`, `Auth::ResetPasswordService`, `Auth::ChangePasswordService`|
|Serializer|`CurrentUserSerializer`|
|Model|`User`, `UserRole`, `UserPersonalInfo`, `Address`, `Prefecture`, `HighSchool`, `Grade`|
