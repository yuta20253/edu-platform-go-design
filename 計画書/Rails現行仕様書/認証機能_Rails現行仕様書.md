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

---

# 3. 業務フロー

1. ログイン時、ユーザーは `email` と `password` を送信する。
2. システムは入力項目を検証し、ユーザー存在とパスワード照合を行う。
3. 認証に成功した場合、JWT を生成して `access_token` Cookie に設定し、認証済みユーザー情報を返却する。
4. ログアウト時はユーザーの `jti` を更新し、Cookie を削除してセッションを終了する。
5. 新規登録時は `user_role_name` を検証し、必要に応じて `high_school_id` / `grade_id` をチェックしてユーザーを作成する。
6. パスワードリセットリクエストでは、該当するユーザーにリセットトークン付きメールを送信する。
7. パスワード変更では、トークンとパスワード確認を検証し、パスワードを更新する。
8. トークン検証では、リセットトークンが存在し、有効期限内かを確認する。

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
  - `user.grade_id` (integer, required for `student` / `teacher`)

### 処理内容

1. `Auth::SignUpForm` で入力項目を検証する。
2. `Auth::SignUpService` で次の処理を実行する。
   - `UserRole.find_by(name: user_role_name)` を検索し、存在しない場合は `SignUpError` を返す。
   - `student` または `teacher` の場合、`high_school_id` と `grade_id` を検証し、存在しない場合は `SignUpError` を返す。
   - `User.create!` でユーザーを作成する。
3. `CurrentUserSerializer` で作成したユーザー情報を返却する。

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

1. `User.find_by(email: params[:email])` でユーザー検索を行う。
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

# 6. データモデル

## `users`

- 役割: ユーザー認証とログイン情報を保持する。
- 主なカラム: `email`, `encrypted_password`, `jti`, `reset_password_token`, `reset_password_sent_at`, `user_role_id`, `high_school_id`, `grade_id`
- リレーション: `belongs_to :user_role`, `belongs_to :high_school`, `belongs_to :grade`

## `user_roles`

- 役割: ユーザーロールを管理する。
- 主なカラム: `name`

---

# 7. 権限制御

- `SessionsController#create` と `RegistrationsController#create` は認証不要。
- `PasswordResetsController` の `create`, `update`, `verify` も認証不要。
- `SessionsController#destroy` は `current_user` が存在することが前提。
- JWT 認証は `access_token` Cookie を利用する。

---

# 8. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::SessionsController`, `Api::V1::RegistrationsController`, `Api::V1::PasswordResetsController`|
|Form|`Auth::LoginForm`, `Auth::SignUpForm`, `Auth::PasswordResetForm`|
|Service|`Auth::LoginService`, `Auth::SignUpService`, `Auth::ResetPasswordService`, `Auth::ChangePasswordService`|
|Serializer|`CurrentUserSerializer`|
