# ユーザー基盤機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

`user` Contextは、ユーザー情報の参照の公開と、招待経路でのアカウント作成（生徒・教員・管理者）、生徒番号の発行、招待メールの送信依頼を担う。エンドユーザーが直接操作する画面・APIを持たず、他のBounded Contextからの内部呼び出しのみを受ける（②「1. 機能概要」）。

## 本書の範囲

**本書は、他Contextが呼び出す公開操作の定義のみを対象とする。** 具体的には、関数のシグネチャ、入力・出力の型、返すエラー、トランザクションの引き継ぎ方式を確定させる。

他Contextの③（認証機能・教師生徒参照機能・生徒CSVインポート機能・教員招待通知機能・管理者管理者ユーザー機能・教師教員管理機能・管理者教員管理機能）が本Contextの操作を呼ぶ記述を持つため、呼び出し側が参照できる境界を先に定める。

本Contextの内部実装（infrastructure層のクエリ、GORMモデル、招待メールの文面、テストケース設計）は本書では扱わない。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により **Transaction Script** を採用する。

- 本Contextの操作は、参照（検索条件の組み立てと取得）と、作成（入力検証・行の作成・付随する依頼の登録）という手続き的な処理で構成される
- 状態遷移を持つのは招待状態（`password_reset_required`）だが、その遷移の責務は`authentication`にあり（認証②「10. 状態遷移図」）、本Contextは作成時の初期値を設定するのみである
- 集約・Repository Interface・UseCase層を持ち込むだけの複雑さがない

---

# 2. ディレクトリ構成

Context名`user`に対応するディレクトリ名は`internal/user/`、package名は`user`となる（アーキテクチャ規約「4. Bounded Context構成」の「ディレクトリ構成」）。Transaction Script採用のため、同規約「3. 設計パターンごとの構造適用方針」のTransaction Scriptの構造に従う。

```
internal/user/
├── context.go                    # NewContext（依存関係の組み立て）
├── dependency.go                 # 他Contextへの依存を宣言するインターフェース
├── application/
│   ├── get_user_attributes.go    # 参照操作
│   ├── find_user_by_email.go
│   ├── search_users.go
│   ├── count_users.go
│   ├── create_student_account.go # 作成操作
│   ├── create_teacher_account.go
│   ├── create_admin_account.go
│   ├── issue_student_number.go   # 生徒番号の発行・書式検証
│   ├── request_invitation_email.go
│   └── dto.go                    # 入出力の型
├── infrastructure/
│   ├── user_query.go             # 参照のDBアクセス
│   ├── user_command.go           # 作成・更新のDBアクセス
│   └── job.go                    # 招待メール送信依頼の`jobs`への登録
└── errors.go                     # 本Contextが返すエラー
```

`presentation/`は作成しない。本ContextはHTTPエンドポイントを持たない（②「1. 機能概要」）。

## 他Contextへの依存（`dependency.go`）

```go
package user

// TokenIssuer は authentication が公開するパスワード設定用トークンの発行操作である。
// トークンの状態遷移（発行・期限・消費）は authentication が所有する（②「論点と決定」3）。
type TokenIssuer interface {
    IssueInvitationToken(ctx context.Context, userID uint) (rawToken string, err error)
}

// SchoolCodeReader は master-data が公開する高校の学校コードの取得操作である。
// 生徒番号の発行に使う（②「論点と決定」10）。
type SchoolCodeReader interface {
    FindSchoolCode(ctx context.Context, highSchoolID uint) (string, error)
}

// Mailer は招待メールの送信基盤である。
type Mailer interface {
    SendInvitationEmail(ctx context.Context, to string, rawToken string) error
}
```

パスワードのハッシュ化は`internal/shared/password`を直接importする。Contextへの依存にはしない（②「論点と決定」3）。

---

# 3. 公開操作の定義

## 入出力の共通型

```go
// UserAttributes は本Contextが公開するユーザーの基本属性である。
// 公開する項目は②「12. UseCase設計」の参照操作の出力に対応する。
type UserAttributes struct {
    ID            uint
    Email         string
    Name          string
    NameKana      string
    Role          Role   // admin / student / teacher / guardian
    HighSchoolID  *uint
    GradeID       *uint
    SchoolClassID *uint
    StudentNumber *string
    // InvitationCompleted は password_reset_required が偽であることを表す。
    InvitationCompleted bool
    // Deactivated は deleted_at が設定されていることを表す。
    Deactivated bool
}

// DeactivatedScope は無効化済みユーザーの扱いである。
// 既定値を持たせず、呼び出し側に必ず指定させる（②「論点と決定」8）。
type DeactivatedScope int

const (
    ActiveOnly DeactivatedScope = iota + 1 // 有効のみ
    IncludeDeactivated                      // 無効化済みを含む
)
```

`DeactivatedScope`にゼロ値を割り当てないのは、指定の欠落を実行時に検出できるようにするためである。ゼロ値を`ActiveOnly`にすると、指定を忘れた呼び出しが黙って通る。

## 参照操作

```go
// GetUserAttributes は指定したユーザーの基本属性を取得する。
// 見つからないIDは結果に含めない（エラーにしない）。無効化済みのユーザーも返す。
func (c *Context) GetUserAttributes(ctx context.Context, userIDs []uint) ([]UserAttributes, error)

// FindUserByEmail はメールアドレスに一致するユーザーを1件取得する。
// 照合前に前後の空白を除去し小文字化する。該当しない場合は found が false になる。
func (c *Context) FindUserByEmail(ctx context.Context, email string) (attrs UserAttributes, found bool, err error)

// SearchUsers は条件に一致するユーザーの一覧を ID 昇順で取得する。
// ページネーションは行わない。条件が1つも指定されていない場合はエラーを返す。
func (c *Context) SearchUsers(ctx context.Context, cond SearchCondition) ([]UserAttributes, error)

// CountUsers は条件に一致するユーザーの人数を数える。
// 人数が0のグループは結果に含めない（Rails現行の group.count と同じ）。
func (c *Context) CountUsers(ctx context.Context, cond SearchCondition, unit AggregateUnit) ([]CountResult, error)

type SearchCondition struct {
    IDs                 []uint
    Roles               []Role
    HighSchoolID        *uint
    GradeID             *uint
    SchoolClassID       *uint
    InvitationCompleted *bool
    ExcludeIDs          []uint
    Deactivated         DeactivatedScope // 必須
}

type AggregateUnit int

const (
    AggregateNone AggregateUnit = iota + 1
    AggregateByRole
    AggregateByHighSchool
    AggregateByGrade
)

type CountResult struct {
    // Key は集計単位に応じたID。AggregateNone の場合は 0。
    Key   uint
    Count int64
}
```

`FindUserByEmail`が「存在しない」をエラーではなく`found`で返すのは、呼び出し側（教師生徒参照のメール重複判定など）が「存在しないこと」を正常系として扱うためである。

## 作成操作

```go
// CreateStudentAccount は生徒アカウント本体を作成し、招待メールの送信を依頼する。
// 生徒番号は本Contextが発行する。常に招待待ちで作成し、作成時に招待メールの送信を依頼する。
func (c *Context) CreateStudentAccount(ctx context.Context, in CreateStudentAccountInput) (UserAttributes, error)

type CreateStudentAccountInput struct {
    Name          string
    NameKana      string
    Email         string
    HighSchoolID  uint
    GradeID       uint
    SchoolClassID *uint // 任意
}

// CreateTeacherAccount は教員アカウント本体を、指定された初期状態で作成する。
// PendingInvitation と SendInvitationEmail は呼び出し側が必ず指定する（②「12. UseCase設計」の経路別の指定値）。
func (c *Context) CreateTeacherAccount(ctx context.Context, in CreateTeacherAccountInput) (UserAttributes, error)

type CreateTeacherAccountInput struct {
    Name     string
    NameKana string
    Email    string
    HighSchoolID uint
    GradeID  *uint // 任意
    // PendingInvitation は招待待ち状態で作成するかである。
    PendingInvitation *bool // 必須（nil はエラー）
    // SendInvitationEmail は作成時に招待メールの送信を依頼するかである。
    SendInvitationEmail *bool // 必須（nil はエラー）
}

// CreateAdminAccount は管理者アカウント本体を、招待待ちで作成し招待メールの送信を依頼する。
// Name が空の場合はメールアドレスの @ より前の部分を氏名とする。
// 氏名カナ・所属校・学年・クラス・生徒番号は設定しない。
func (c *Context) CreateAdminAccount(ctx context.Context, in CreateAdminAccountInput) (UserAttributes, error)

type CreateAdminAccountInput struct {
    Name  string // 任意
    Email string
}
```

`PendingInvitation`・`SendInvitationEmail`をポインタにして`nil`をエラー扱いにするのは、`bool`のゼロ値（false）と「指定なし」を区別するためである。Rails現行では教師経路と管理者経路でこの2つの組み合わせが異なり、既定値を置くとどちらかの経路が黙って誤った状態で作成される。

## 生徒番号

```go
// IssueStudentNumber は生徒に一意の生徒番号を割り当てる。
// 書式は「高校の学校コード-8桁の英数字大文字」である（②「7. Value Object設計」）。
// 既存生徒の更新経路（生徒CSVインポート）も、発行規則を複製せずこの操作を呼ぶ（②「論点と決定」6）。
func (c *Context) IssueStudentNumber(ctx context.Context, highSchoolID uint) (string, error)

// ValidateStudentNumberFormat は生徒番号の書式を検証する。
// authentication と account-linking は、書式を自前で定義せずこの操作を呼ぶ（②「論点と決定」7）。
func ValidateStudentNumberFormat(studentNumber string) error
```

`ValidateStudentNumberFormat`をメソッドではなくpackageレベルの関数にするのは、書式検証がDB接続も他Contextへの依存も必要としないためである。呼び出し側が`Context`を組み立てずに使える。

## 招待メールの送信依頼

```go
// DeliveryMode は招待メールの実行方式である。既定値を持たせず、必ず指定させる。
type DeliveryMode int

const (
    // DeliveryEnqueue は jobs へ依頼を登録し、送信をワーカーに任せる方式である。
    // 呼び出し側のトランザクションに参加し、ロールバック時は依頼も取り消される。
    DeliveryEnqueue DeliveryMode = iota + 1
    // DeliveryImmediate はその場で送信し、結果を返す方式である。
    // 呼び出し側のトランザクションの外で呼ぶ。
    DeliveryImmediate
)

// InvitationResult は即時方式の送信結果である。送信失敗はエラーではなく結果として返す。
type InvitationResult int

const (
    InvitationSent InvitationResult = iota + 1 // 送信成功
    InvitationFailed                            // 送信失敗（トークン発行またはメール送信基盤の失敗）
    InvitationSkipped                           // 送信対象外（対象が存在しない、または無効化済み）
)

// RequestInvitationEmail は指定したユーザーへ、パスワード設定用のリンクを含む招待メールを送る。
// DeliveryEnqueue の場合、戻り値の InvitationResult は意味を持たない（受付の完了のみ）。
func (c *Context) RequestInvitationEmail(ctx context.Context, userID uint, mode DeliveryMode) (InvitationResult, error)
```

即時方式が送信失敗をエラーではなく`InvitationResult`で返すのは、`teacher-notification`が失敗を業務結果として`TeacherNotification`へ記録する必要があるためである（②「12. UseCase設計」）。エラーで返すと、呼び出し側が「記録すべき失敗」と「処理を中断すべき障害」を区別できない。

本操作は認可を行わない。「その教員へ送ってよいか」（他職員操作権限・同校・招待未完了）の判定は呼び出し元の責務である（②「16. Authorization設計」）。

---

# 4. トランザクションの引き継ぎ

本Contextの作成操作は、呼び出し側のトランザクションに参加する必要がある。アーキテクチャ規約「11. Transaction実装パターン（TransactionManager）」の「呼び出し側トランザクションの引き継ぎ」に従い、`context.Context`で引き継ぐ（②「論点と決定」9）。

```go
// Good（本Contextの作成操作）
func (c *Context) CreateStudentAccount(ctx context.Context, in CreateStudentAccountInput) (UserAttributes, error) {
    var attrs UserAttributes

    err := transaction.RunInTx(ctx, c.db, func(tx *gorm.DB) error {
        // 行の作成と、jobs への招待メール送信依頼の登録を同じトランザクションで行う
        return nil
    })

    return attrs, err
}
```

`transaction.RunInTx`は、`ctx`にトランザクションがあればそれに参加し、なければ新しく開始する（規約11章）。呼び出し元の採用パターン（教員管理はActive Record、教師生徒参照はTransaction Script、生徒CSVインポートはDomain Model）によらず、引き継ぎの方式は同一である。

`RequestInvitationEmail`の即時方式は、この引き継ぎの対象外である。呼び出し側のトランザクションの外で呼ぶ（コミット前に送信してはならない）。

---

# 5. `users`への書き込み範囲

`users`は複数のContextが列ごとに異なる責務で書き込む共有テーブルである（②「20. DB設計方針」、アーキテクチャ規約の方針）。**本Contextは、自分が所有する列だけを対象に更新する。** 構造体全体を渡す`Save`を使わない。

```go
// Good
// 作成時に本Contextが設定する列だけを指定する
return tx.Model(&userModel{}).Create(map[string]any{
    "email":                   in.Email,
    "name":                    in.Name,
    "name_kana":               in.NameKana,
    "user_role_id":            roleID,
    "high_school_id":          in.HighSchoolID,
    "grade_id":                in.GradeID,
    "school_class_id":         in.SchoolClassID,
    "student_number":          studentNumber,
    "encrypted_password":      hashed,
    "jti":                     jti,
    "password_reset_required": true,
}).Error
```

```go
// Bad
// 構造体全体を保存している。読み取り時点の値で他Contextの列（address_id・activated_at など）を
// 上書きし、その間に他Contextが行った更新を消す
return tx.Save(&u).Error
```

`users`には楽観ロック用の列がなく、既存Rails DBを継続利用するためスキーマ変更も行わない（Gorm規約「7. マイグレーション」）。したがってGorm規約「12. 楽観ロック（Optimistic Locking）」の方式は使えず、他Contextの列を巻き込まない部分更新が唯一の防御手段である。

本Contextが作成時に設定する列と、作成後に他Contextが更新する列の対応は②「20. DB設計方針」の表に従う。

---

# 6. 本書で扱わない範囲

次の内容は本書では定めない。本Contextの内部実装を具体化する際に追加する。

- infrastructure層のクエリの実装（`SearchUsers`の条件の組み立て、`CountUsers`のグループ集計）
- GORMモデルの定義とタグ
- 招待メールの文面・件名・リンクの構成
- 生徒番号の衝突時の再試行回数と上限の扱い
- テストケース設計

---

# 7. ②からの補足事項

|補足した内容|理由|区分|
|-|-|-|
|`DeactivatedScope`・`DeliveryMode`にゼロ値を割り当てず、`iota + 1`から始める|②は「無効化の扱い」「実行方式」を必須指定としているが、Goの`int`型のゼロ値は0であり、指定の欠落と有効な値を区別できない。1から始めることで、指定忘れを実行時に検出できる|補足（②の必須指定という方針をGoで表現するための判断）|
|`CreateTeacherAccountInput`の`PendingInvitation`・`SendInvitationEmail`をポインタにする|②は両方を必須指定としている。`bool`では「false」と「指定なし」が区別できず、既定値を置くと教師経路・管理者経路のいずれかが黙って誤った状態で作成される|補足（同上）|
|`FindUserByEmail`が「存在しない」を`found`で返す|②は出力を「ユーザーの基本属性、または『存在しない』」としている。呼び出し側は存在しないことを正常系として扱うため、エラーではなく真偽値で返す|補足（②の出力の定義をGoのシグネチャに落としたもの）|
|`ValidateStudentNumberFormat`をpackageレベルの関数にする|書式検証はDB接続も他Contextへの依存も必要としない。`Context`の組み立てを呼び出し側に要求しないため|補足|
|`CountResult.Key`を集計単位に応じたIDとし、`AggregateNone`では0とする|②は出力を「集計の単位が『なし』なら人数1件、それ以外ならグループごとの人数」としている。単位ごとに別の型を用意せず1つの型で表現するため|補足|
