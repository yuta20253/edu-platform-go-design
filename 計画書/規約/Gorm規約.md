# Gorm規約

## 0. 本プロジェクトでの採用方針

以下は、GORM標準の規約（1章以降）を前提として、本プロジェクトで実際に採用する仕様を確定したものである。②Go移行・設計仕様書／③Go実装仕様書がこの方針と異なる判断をする場合は、該当文書内で理由を明記する。

### 主キー

主キーは自動採番の連番整数（`ID uint`）を採用する。UUIDは採用しない。

```go
type Teacher struct {
    ID uint // 自動採番の主キー
    // ...
}
```

### 論理削除（ソフトデリート）

削除操作は原則として論理削除とし、GORM標準の`gorm.DeletedAt`型フィールドを使用する。物理削除（`Unscoped`）は業務要件上明確に必要な場合のみ使用する。

```go
type Teacher struct {
    ID        uint
    Name      string
    DeletedAt gorm.DeletedAt `gorm:"column:deleted_at"`
}
```

`gorm.DeletedAt`型を使うことで、`Delete`実行時に`deleted_at`が自動的に現在時刻へ設定され、`Find`系メソッドは標準で`deleted_at IS NULL`を自動的に条件へ付与する。物理削除が必要な場合のみ`Unscoped()`を明示的に呼び出す。

> 補足: この方針が明記される以前は、③Go実装仕様書側（例：管理者管理者ユーザー機能、教師教員管理機能）が機能ごとに同じ結論を個別に「推測」として導出していた。今後はこの規約を正とし、③文書側での再導出は不要とする。

### カラム名の例外（連続大文字・略語）

`JTI`のような連続大文字を含む略語など、GORMの標準snake_case変換では意図が曖昧になりうるフィールド名は、`column`タグで明示的にカラム名を指定する。

```go
type RefreshToken struct {
    JTI string `gorm:"column:jti"` // 標準変換のみでは意図が曖昧になり得るため明示する
}
```

### リレーションの取得とN+1対策

関連レコードを同時に取得する場合は、GORMの`Preload`（Eager Loading）を使用し、ループ内で都度クエリを発行するN+1を避ける。

```go
// Good
var courses []Course
db.Preload("Units").Find(&courses)
```

```go
// Bad
var courses []Course
db.Find(&courses)
for _, c := range courses {
    db.Where("course_id = ?", c.ID).Find(&c.Units) // N+1
}
```

外部キー列（例：`SchoolID uint`）はEntity側に明示的なフィールドとして保持する。関連データの取得は`Preload`または明示的な`Joins`を用い、暗黙のアソシエーション自動読み込みに業務ロジックの結合判定を委ねすぎない。

### 配置（ディレクトリ構成との対応）

GORM用のモデルstruct・タグ定義は、Infrastructure層に置く（具体的な配置ディレクトリはアーキテクチャ規約のディレクトリ構成が確定次第定める）。Domain Modelパターン採用時、Domain層の`entity/`にはGORMタグを持つstructを直接置かず、Infrastructure層のGORMモデルとDomain Entityを分離する。Active Recordパターン採用時は、アーキテクチャ規約「3. 設計パターンごとの構造適用方針」の簡略構造（`model.go`にGORMタグ付きstructを直接定義）に従う。

---

## 1. 主キーとしての ID

GORMはデフォルトで、テーブルの主キーとして ID という名前のフィールドを使用します。

```go
type User struct {
  ID   string //フィールドはデフォルトでは `ID` という名前のフィールドがプライマリフィールドとして使われます。
  Name string
}
```

他のフィールドを primaryKey タグで主キーとして設定できます

```go
// Set field `UUID` as primary field
type Animal struct {
  ID     int64
  UUID   string `gorm:"primaryKey"`
  Name   string
  Age    int64
}
```

Composite Primary Key も参照してください。

## 2. 複数形のテーブル名

GORMは構造体名をテーブル名としてsnake_casesのように複数形にします。構造体 User の場合、対応するテーブル名は規約により users となります。

### テーブル名

Tabler インターフェイスを実装することで、デフォルトのテーブル名を変更することができます。例：

```go
type Tabler interface {
    TableName() string
}

// TableName overrides the table name used by User to `profiles`
func (User) TableName() string {
  return "profiles"
}
```

注意 メソッドの戻り値はキャッシュされるため、 TableNameは動的な名前を許可していません。動的にテーブル名を変更するには、 Scopes で解決することができます。例:

```go
func UserTable(user User) func (tx *gorm.DB) *gorm.DB {
  return func (tx *gorm.DB) *gorm.DB {
    if user.Admin {
      return tx.Table("admin_users")
    }

    return tx.Table("users")
  }
}

db.Scopes(UserTable(user)).Create(&user)
```

### 一時的に名前を指定する

Tableメソッドで一時的にテーブル名を指定できます。例:

```go
// Create table `deleted_users` with struct User's fields
db.Table("deleted_users").AutoMigrate(&User{})

// Query data from another table
var deletedUsers []User
db.Table("deleted_users").Find(&deletedUsers)
// SELECT * FROM deleted_users;

db.Table("deleted_users").Where("name = ?", "jinzhu").Delete(&User{})
// DELETE FROM deleted_users WHERE name = 'jinzhu';
```

FROM句でサブクエリを使用する方法については、 From SubQuery を参照してください。

### NamingStrategy

GORM allows users to change the default naming conventions by overriding the default NamingStrategy, which is used to build TableName, ColumnName, JoinTableName, RelationshipFKName, CheckerName, IndexName, Check out GORM Config for details

## 3. カラム名

規約に従い、データベースのカラム名はフィールド名のsnake_caseを使用します。

```go
type User struct {
  ID        uint      // column name is `id`
  Name      string    // column name is `name`
  Birthday  time.Time // column name is `birthday`
  CreatedAt time.Time // column name is `created_at`
}
```

column タグか NamingStrategy を利用することでカラム名を上書きできます。

```go
type Animal struct {
  AnimalID int64     `gorm:"column:beast_id"`         // set name to `beast_id`
  Birthday time.Time `gorm:"column:day_of_the_beast"` // set name to `day_of_the_beast`
  Age      int64     `gorm:"column:age_of_the_beast"` // set name to `age_of_the_beast`
}
```

## 4. タイムスタンプのトラッキング

### CreatedAt

CreatedAtフィールドを持つモデルの場合、フィールドの値がゼロ値であれば、レコード作成時に現在時刻が設定されます。

```go
db.Create(&user) // set `CreatedAt` to current time

user2 := User{Name: "jinzhu", CreatedAt: time.Now()}
db.Create(&user2) // user2's `CreatedAt` won't be changed

// To change its value, you could use `Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

autoCreateTime タグを falseに設定すると、タイムスタンプのトラッキングを無効にできます。例：

```go
type User struct {
  CreatedAt time.Time `gorm:"autoCreateTime:false"`
}
```

### UpdatedAt

UpdatedAtフィールドを持つモデルの場合、フィールドの値がゼロ値であれば、レコードの更新時または作成時に現在時刻が設定されます。

```go
db.Save(&user) // set `UpdatedAt` to current time

db.Model(&user).Update("name", "jinzhu") // will set `UpdatedAt` to current time

db.Model(&user).UpdateColumn("name", "jinzhu") // `UpdatedAt` won't be changed

user2 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Create(&user2) // user2's `UpdatedAt` won't be changed when creating

user3 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Save(&user3) // user3's `UpdatedAt` will change to current time when updating
```

autoUpdateTime タグを falseに設定すると、タイムスタンプのトラッキングを無効にできます。例：

```go
type User struct {
  UpdatedAt time.Time `gorm:"autoUpdateTime:false"`
}
```

注意 GORMでは、複数のタイムトラッキング用のフィールドを定義することや、UNIX(ナノ/ミリ)秒でタイムトラッキングすることが可能です。詳細については Models をチェックしてください。

---

## 5. アソシエーション

GORMは外部キーを`{関連モデル名}ID`という規約で自動生成します（例：`User`への`belongsTo`であれば`UserID`）。`many2many`の中間テーブル名は両モデル名を組み合わせた規約名（例：`user_languages`）になります。

```go
type User struct {
  Languages []Language `gorm:"many2many:user_languages;"`
}
```

`foreignKey` / `references`タグで外部キーと参照先カラムを、`many2many`タグで中間テーブル名を上書きできます。

```go
type User struct {
  CompanyID int
  Company   Company `gorm:"foreignKey:CompanyID;references:ID"`
}
```

### 本プロジェクトでの方針

外部キー列（例：`SchoolID uint`）は規約に従い`{関連モデル名}ID`のまま明示的にEntity/GORMモデルへ持たせる。関連データの取得は「0. 本プロジェクトでの採用方針」の「リレーションの取得とN+1対策」のとおり`Preload`または`Joins`を用い、`Association()`メソッドによる暗黙的な関連更新（追加・削除の自動反映）は使用しない。関連の追加・削除は、Repository/Storeのメソッドとして明示的に定義する。

```go
// Good
func (s *TaskStore) AddUnitLinks(ctx context.Context, taskID uint, unitIDs []uint) error {
    // 明示的なメソッドとして提供する
}
```

```go
// Bad
db.Model(&task).Association("Units").Append(units) // 暗黙的な関連操作をRepository外から直接呼び出す
```

---

## 6. Transaction

GORMは`db.Transaction(func(tx *gorm.DB) error {...})`というコールバック形式のトランザクションAPIを提供する。関数がエラーを返すとRollback、`nil`を返すとCommitされる。

```go
err := db.Transaction(func(tx *gorm.DB) error {
  if err := tx.Create(&user1).Error; err != nil {
    return err
  }

  if err := tx.Create(&user2).Error; err != nil {
    return err
  }

  return nil
})
```

トランザクション内では、必ず`db`ではなく引数で渡された`tx`を使用する。`db`を直接使うと、トランザクションの外側で実行されてしまう。

Savepointによる部分的なロールバックには`SavePoint()` / `RollbackTo()`を使用できる。手動制御が必要な場合は`Begin()` / `Commit()` / `Rollback()`を使用するが、`defer`でのpanicリカバリを伴う定型パターンを守る。

GORMはデフォルトで書き込み操作をトランザクションでラップする（データ整合性のため）。トランザクションが不要なことが明確な読み取り専用の初期化などに限り、`gorm.Config{SkipDefaultTransaction: true}`によるパフォーマンス向上（公式ドキュメントによれば約30%）を検討してよいが、既定では無効のままとする。

### 本プロジェクトでの方針

* Active Record採用機能: Storeメソッド内で`db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })`を直接使用する（アーキテクチャ規約「11. Transaction実装パターン（TransactionManager）」の適用範囲のとおり）
* Domain Model採用機能: `TransactionManager`の実装（`infrastructure/repository`配下）が内部で`db.WithContext(ctx).Transaction(...)`をラップする。UseCase側がこのAPIを直接呼び出すことはない

---

## 7. マイグレーション

`AutoMigrate`はテーブル・不足している外部キー・制約・カラム・インデックスを自動作成し、既存カラムのサイズ・精度・NULL許容が変わった場合は型を変更する。**既存の不要なカラムは削除しない**（データ保護のため）。

```go
db.AutoMigrate(&User{})
```

公式ドキュメントは、バージョン管理されたマイグレーションが必要な成熟したプロジェクトには、`AutoMigrate`ではなく専用マイグレーションツール（Atlas等）の使用を推奨している。

### 本プロジェクトでの方針

②Go移行・設計仕様書の前提（既存Rails DBを継続利用する）のとおり、本プロジェクトは既存のRailsマイグレーションによって作成されたスキーマをそのまま利用する。したがって`AutoMigrate`は本番運用では使用しない。Go側での新規テーブル追加・カラム変更が必要になった場合は、Railsのマイグレーション体系（`db/migrate/`）に追加する形で行い、GORM側の`AutoMigrate`で独自にスキーマを変更しない。`AutoMigrate`はローカル環境でのテスト用DBセットアップ等、開発補助用途に限定して使用してよい。

---

## 8. ロガー設定

GORMは既定でスロークエリとエラーを出力するロガーを内蔵する。ログレベルは`Silent` / `Error` / `Warn` / `Info`から選択する。

```go
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  Logger: logger.Default.LogMode(logger.Silent),
})
```

`SlowThreshold`（スロークエリと判定する閾値）、`IgnoreRecordNotFoundError`（レコード未検出エラーを無視するか）、`ParameterizedQueries`（SQLログにパラメータを含めるか）等を設定できる。独自ロガーを使う場合は`LogMode()` / `Info()` / `Warn()` / `Error()` / `Trace()`を実装した`logger.Interface`を用意する。

### 本プロジェクトでの方針

コーディング規約「20. ロギング」の構造化ロギング方針（`log/slog`使用）に合わせ、GORMの`logger.Interface`を実装したアダプタを用意し、`slog.Logger`へブリッジする。アプリケーション本体のログとGORMが出すSQLログを同一の構造化フォーマット・同一の出力先に統一する。`SlowThreshold`は既定値（200ms）から開始し、実測に応じて調整する。

---

## 9. エラーハンドリング

Traditional APIでは、finisherメソッド（`First` / `Find` / `Create`等）の呼び出し後に`.Error`フィールドを確認する。

```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
  // エラー処理
}
```

レコードが見つからない場合は`gorm.ErrRecordNotFound`が返る。判定には文字列比較や`==`ではなく`errors.Is`を使用する（コーディング規約「6. エラーハンドリング」に従う）。

```go
// Good
if errors.Is(err, gorm.ErrRecordNotFound) {
    return nil, apperror.NewNotFound("user not found")
}
```

`gorm.Config{TranslateError: true}`を設定すると、DB固有のエラー（一意制約違反等）を`gorm.ErrDuplicatedKey` / `gorm.ErrForeignKeyViolated`等の共通エラー型に変換できる。

### 本プロジェクトでの方針

`TranslateError: true`を設定し、DB方言に依存しないエラー判定を行う。Repository/Store実装は、GORMのエラー（`gorm.ErrRecordNotFound`・`gorm.ErrDuplicatedKey`等）を`errors.Is`で判定した上で、Domain Error（Domain Model採用時）またはApplication Error相当のsentinel error（Active Record/Transaction Script採用時）に変換して返す。GORMのエラーをそのまま上位層（UseCase/Handler）へ伝播させない。

```go
// Good
func (r *userRepository) FindByID(ctx context.Context, id uint) (*domain.User, error) {
    var m userModel

    if err := r.db.WithContext(ctx).First(&m, id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrUserNotFound
        }

        return nil, fmt.Errorf("ユーザーの取得に失敗しました: %w", err)
    }

    return m.toDomain(), nil
}
```

```go
// Bad
func (r *userRepository) FindByID(ctx context.Context, id uint) (*domain.User, error) {
    var m userModel

    err := r.db.WithContext(ctx).First(&m, id).Error
    return m.toDomain(), err // gorm.ErrRecordNotFoundがそのまま上位層に漏れる
}
```

---

## 10. Hooks

GORMは`BeforeSave` / `BeforeCreate` / `AfterCreate` / `BeforeUpdate` / `AfterUpdate` / `BeforeDelete` / `AfterDelete` / `AfterFind`等のモデルフックを提供する（シグネチャは`func(*gorm.DB) error`）。フックはSave/Delete処理と同一トランザクション内で実行され、エラーを返すとロールバックされる。

### 本プロジェクトでの方針

**Hooksは使用しない。** 理由は以下のとおり。

* Hooksはモデルstruct（Infrastructure層のGORMモデル、Active Record採用時はEntity相当のstruct）に業務ロジックを埋め込む機構であり、アーキテクチャ規約「2. レイヤー責務と依存方向」の禁止事項「Repository実装に業務ルール判定を持たせる」に抵触する
* 業務ルール検証はDomain（Domain Model採用時）またはstruct/関数（Active Record/Transaction Script採用時）で行うという、アーキテクチャ規約「7. 横断的関心事の置き場所」の方針と重複・競合する経路になる

`CreatedAt` / `UpdatedAt`の自動設定、`DeletedAt`による論理削除といったGORM標準機能（Hooksではなく規約に基づく自動化）は、業務ルールではなく永続化の関心事であるため、この禁止の対象外として通常どおり使用する。

---

## 11. Context

`db.WithContext(ctx)`でコンテキストを渡す。goroutine-safeであり、複数のgoroutineから同一の`*gorm.DB`を安全に扱える。

```go
db.WithContext(ctx).Find(&users)
```

複数の操作にまたがって同じコンテキストを使う場合は、Continuous Session Modeとして`tx := db.WithContext(ctx)`を変数に保持し、以降の呼び出しに使い回す。

```go
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```

### 本プロジェクトでの方針

コーディング規約「11. context.Context」のとおり、Repository/Store/DBアクセス関数はすべて第一引数に`ctx context.Context`を受け取り、GORM呼び出しの直前で必ず`db.WithContext(ctx)`を経由する。`ctx`を受け取らずにパッケージ変数の`db`を直接使うメソッドは作らない。

```go
// Good
func (s *TaskStore) FindByID(ctx context.Context, id uint) (*Task, error) {
    var t Task

    if err := s.db.WithContext(ctx).First(&t, id).Error; err != nil {
        return nil, err
    }

    return &t, nil
}
```

```go
// Bad
func (s *TaskStore) FindByID(id uint) (*Task, error) {
    var t Task

    if err := s.db.First(&t, id).Error; err != nil { // ctxを受け取らず、キャンセル・タイムアウトが伝搬しない
        return nil, err
    }

    return &t, nil
}
```

---

## 12. 楽観ロック（Optimistic Locking）

複数ユーザーが同じレコードを同時に更新しうる機能では、`gorm.io/plugin/optimisticlock`を使用する。GORM本体と同じ`go-gorm` Organizationが提供する第一級プラグインであり、サードパーティ実装より優先する。

### 使用方法

対象のGORMモデルに`optimisticlock.Version`型のフィールドを追加する。`Model(&m).Updates(...)`実行時、GORMが自動的に`WHERE version = ?`を付与し、成功時に`version`をインクリメントする。

```go
type taskModel struct {
    ID      uint
    Name    string
    Version optimisticlock.Version // カラム名は規約どおり `version`
}
```

```sql
-- 生成されるSQL（イメージ）
UPDATE `tasks` SET `name`='...', `version`=`version`+1 WHERE `tasks`.`version` = 1 AND `id` = 1
```

### 競合の検出とエラー変換

更新のRowsAffectedが0件の場合、バージョン不一致（他ユーザーによる更新が先に成功した）とみなし、共有の`ErrOptimisticLockConflict`（配置例: `internal/shared/domainerror`）を返す。GORM自身のエラー（`result.Error`）とは区別する。

```go
// Good
func (r *taskRepository) Update(ctx context.Context, t *domain.Task) error {
    m := fromDomain(t) // Versionを含めて変換する

    result := r.db.WithContext(ctx).Model(&m).Updates(m)
    if result.Error != nil {
        return fmt.Errorf("タスクの更新に失敗しました: %w", result.Error)
    }

    if result.RowsAffected == 0 {
        return domainerror.ErrOptimisticLockConflict
    }

    return nil
}
```

```go
// Bad
func (r *taskRepository) Update(ctx context.Context, t *domain.Task) error {
    m := fromDomain(t)

    result := r.db.WithContext(ctx).Model(&m).Updates(m)
    return result.Error // RowsAffected == 0 のケース（競合）を見逃し、更新成功として扱ってしまう
}
```

UseCase（またはTransaction Script/Active Recordの場合は該当関数・Store呼び出し元）は、`errors.Is(err, domainerror.ErrOptimisticLockConflict)`で判定し、アーキテクチャ規約「12. Error変換パターン（AppError）」の`AppError`へ変換する（`StatusCode()`は`http.StatusConflict`（409）とする）。Presentation層側の処理は既存のAppError変換フローをそのまま利用でき、個別対応は不要である。

### Domain EntityとGORMモデルの分離

Domain Model採用機能では、`optimisticlock.Version`型（GORM依存）をDomain Entityに直接持たせない（3章「DomainはGin・GORMのimportを持たない」）。Domain Entityは`Version int64`のようなプレーンな型で保持し、Repository実装（`toDomain` / `fromDomain`）で相互変換する。

```go
// Good
// domain/entity/task.go（Domainはoptimisticlockに依存しない）
type Task struct {
    ID      uint
    Name    string
    Version int64
}
```

Active Record採用機能では、Entity相当のstructとGORMモデルが同一のため、そのまま`optimisticlock.Version`型のフィールドを持たせてよい。

### 適用範囲

すべてのテーブルに一律で`Version`列を追加しない。複数ユーザーが同じレコードを同時に更新しうる機能（例：管理者による同一リソースの並行編集）にのみ適用する。単純な参照専用データや、同時更新が業務上起こり得ないデータには適用しない。

### リトライ方針

競合発生時、アプリケーション側での自動リトライは行わない。409をそのままクライアントへ返し、最新データの再取得・再送信をクライアント側に委ねる。UseCase内での自動リトライが必要な機能がある場合は、その機能のGo実装仕様書で理由とともに個別に定める。
