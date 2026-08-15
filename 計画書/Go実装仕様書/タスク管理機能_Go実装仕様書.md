# タスク管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒が自分の学習タスクを登録・参照・更新できる機能である。一覧・詳細・作成・更新の4操作を提供し、タスクに紐づく目標（Goal）・単元（Unit）情報を含めて扱う（②「1. 機能概要」要約）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により **Active Record** を採用する。

- 主要操作がCRUD中心で、作成・取得・更新の流れが明確である
- タスクと単元との関連付けは、データの整合性を保つために集約しやすい
- Handlerが直接Storeを呼び出し、認可・入力検証をHandler／Store側に集約することで、データアクセスの振る舞いを一元化しやすい

状態遷移（not_started / in_progress / completed）は存在するが、複雑なビジネスルールが多いわけではないため、Domain Model・Transaction Script・Event Sourcingはいずれも採用しない（②「4. 設計パターン」）。

本書は、`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」の **Active Record** 節の構造（`model.go` 相当のstruct定義＋`store.go` 相当のStore、usecase層なし、Repository Interfaceの分離なし）に従って実装レベルへ落とし込む。

## 本書が対象とする実装範囲

- Bounded Context: `task-management`（②「3. Bounded Context」）
- 対象エンドポイント: タスク一覧取得・詳細取得・作成・更新の4API（②「16. API互換方針」）
- 対象外: Goal・Unit・Userそのものの管理機能（他Contextの責務。②「3. Bounded Context」他Contextとの依存関係を参照）
- ①Rails実装の詳細は本タスクでは提供されていないため、参照が必要な箇所は「①未提供のため参照不可」として扱う

---

# 2. ディレクトリ構成

## 対象Bounded Context名

`task-management`

`規約/アーキテクチャ規約.md`「5. Bounded Context構成」の機能Context一覧表では `task-management` は上位ドメイン `Task` に属し、対応する内部ディレクトリ名は明記されていない。②にも明記がないため、規約2章のプロジェクト全体構成に列挙された `internal/task/` を本Contextのディレクトリとして採用する（**②からの補足・推測**。詳細は「14. ②からの補足事項」参照）。

## ②で採用した設計パターン

Active Record（②「4. 設計パターン」）

## 採用パターンに対応する構造

`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」Active Record節に従い、domain/infrastructureのレイヤー分離およびusecase層を設けない。Entity相当のstructと永続化操作（Store）を同一package（`internal/task`）に置く。

作成するディレクトリ一覧:

```
internal/task/
internal/task/presentation/
internal/task/presentation/handler/
internal/task/presentation/request/
internal/task/presentation/response/
```

## 作成するファイル一覧

```
internal/task/task.go                          # Task struct・TaskStatus型・Validate()等
internal/task/task_unit_link.go                 # TaskUnitLink struct
internal/task/task_unit_relation_policy.go       # 単元紐付け追加・削除可否判定（②「8. Domain Service」相当）
internal/task/dependency.go                     # 他Context参照用インターフェース定義（GoalOwnershipChecker / UnitReferenceChecker）
internal/task/errors.go                          # struct/Storeが返すエラー変数定義
internal/task/task_store.go                      # TaskStore
internal/task/task_unit_link_store.go            # TaskUnitLinkStore
internal/task/presentation/handler/task_handler.go
internal/task/presentation/request/task_request.go
internal/task/presentation/response/task_response.go
internal/task/presentation/routes.go
```

`domain/` `application/` `infrastructure/` の各ディレクトリはActive Record採用のため作成しない（規約4章）。

---

# 3. Domain層設計

**実装上の位置づけ**: 本機能はActive Record採用のため、domain層のディレクトリ分離は行わない。以下は②「6〜9章」の設計意図を、Active Record構造（Model＝struct＋メソッド、Store＝永続化）に落とし込んだものである（規約4章 Active Record節「『Value Object』『Repository Interface』『Domain Service』は原則『対象外』とし、検証ルールはModelのメソッドとして記載する」に従う）。

## Model（Entity相当）

### Task（`internal/task/task.go`）

②「6. Entity設計」Taskの責務・②「7. Value Object設計」TaskStatus/DueDate/Priorityの独自ルールを、同一package内のstruct・型・メソッドとして統合する。

フィールド:

|フィールド|型|意味|
|-|-|-|
|ID|uint|タスクID（主キー）|
|UserID|uint|所有者（生徒）のユーザーID|
|GoalID|uint|紐づく目標ID（②「12. Validation設計」の「指定されたgoal_idが自分の目標かどうか」の検証対象）|
|Title|string|タイトル|
|Content|string|内容|
|DueDate|time.Time|期限（②「7. Value Object設計」DueDateの意図を、比較・表示用メソッドで代替。詳細は「14. ②からの補足事項」参照）|
|Priority|string|優先度の正規化後の値（②「7. Value Object設計」Priorityの正規化ルールを`NormalizePriority`関数で実現）|
|Memo|string|メモ|
|Status|TaskStatus|タスク状態|
|CompletedAt|*time.Time|完了日時。completed以外ではnil（②「7. Value Object設計」TaskStatusの独自ルール）|
|CreatedAt|time.Time|作成日時（GORM自動設定。Gorm規約「タイムスタンプのトラッキング」）|
|UpdatedAt|time.Time|更新日時（GORM自動設定）|

`TaskStatus`型（string基底の独自型）:

```go
type TaskStatus string

const (
    TaskStatusNotStarted TaskStatus = "not_started"
    TaskStatusInProgress TaskStatus = "in_progress"
    TaskStatusCompleted  TaskStatus = "completed"
)
```

公開method一覧（シグネチャのみ。実装ロジックは記載しない）:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTask`|`(userID, goalID uint, title, content string, dueDate time.Time, priority string, memo string) (*Task, error)`|`(*Task, error)`|新規Task生成時の不変条件（必須項目・初期status=not_started）を保証するファクトリ|
|`(t *Task) Validate() error`|なし|`error`|title等必須項目・値域の妥当性を検証する（②「12. Validation設計」Domainの一部）|
|`(t *Task) ChangeStatus(status TaskStatus) error`|`status TaskStatus`|`error`|status遷移の妥当性判定と、completed時のCompletedAt設定／非completed時のクリアを行う（②「7. Value Object設計」TaskStatus独自ルール、②「14. Error設計」不正な状態遷移の判定元）|
|`(t Task) IsOwnedBy(userID uint) bool`|`userID uint`|`bool`|所有者判定（②「13. Authorization設計」の業務的判定を補助）|
|`(t Task) IsOverdue(now time.Time) bool`|`now time.Time`|`bool`|期限比較（②「7. Value Object設計」DueDateの「期限の比較演算を一元化する」を実現）|
|`(t Task) DueDateDisplay() string`|なし|`string`|画面表示用フォーマット変換（②「7. Value Object設計」DueDateの「画面表示用のフォーマットと永続化用の値を分ける」を実現）|
|`NormalizePriority(input any) (string, error)`|`input any`|`(string, error)`|文字列・整数いずれの入力も受け取り、ドメイン上一貫した値へ正規化する（②「7. Value Object設計」Priority独自ルール。パッケージレベル関数。正規化後の具体的な値集合は②に明記がないため「14. ②からの補足事項」参照）|

不変条件（`NewTask`で保証する内容）:

- UserID・GoalID・Titleは空値を許容しない
- 生成直後のStatusは常に`TaskStatusNotStarted`
- 生成直後のCompletedAtは常にnil

### TaskUnitLink（`internal/task/task_unit_link.go`）

②「6. Entity設計」TaskUnitLinkの責務を反映する。

フィールド:

|フィールド|型|意味|
|-|-|-|
|ID|uint|関連ID（主キー）|
|TaskID|uint|紐づくタスクID|
|UnitID|uint|紐づく単元ID|
|CreatedAt|time.Time|作成日時（GORM自動設定）|

公開method一覧:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`NewTaskUnitLink`|`(taskID, unitID uint) (*TaskUnitLink, error)`|`(*TaskUnitLink, error)`|taskID・unitIDが0でないことを保証するファクトリ|

## Value Object

規約4章「Active Record」節により、Value Objectを独立した層としては設けない（原則「対象外」）。②「7. Value Object設計」で定義されたTaskStatus／DueDate／Priorityの独自ルールは、上記「Model」節のとおりTaskのフィールド型・メソッドとしてすべて実現している。②「Value Objectを採用しないもの」（タイトル・内容・メモ）についても同様に単純な`string`フィールドとして扱う。

## Repository Interface

規約4章「Active Record」節により、domain層にRepository Interfaceを定義しない（原則「対象外」）。②「9. Repository設計」の責務は「5. Infrastructure層設計」節のStoreとして実装する。

なお、②「9. Repository設計」のGoalStore・UnitStoreは、goal-management Context・curriculum(Unit) Context側が公開する参照手段であり、task-managementがこれらのデータを所有・実装するものではない（`規約/アーキテクチャ規約.md`「6. Context間連携ルール」）。task-management側では、コーディング規約「5. インターフェース」（利用側でインターフェースを定義する）に従い、`internal/task/dependency.go`に以下の参照用interfaceを定義する。

```go
// GoalOwnershipChecker は goal-management Context が公開する
// 「目標が指定の生徒に属するか」の参照手段を表す。
type GoalOwnershipChecker interface {
    ExistsForStudent(ctx context.Context, goalID, userID uint) (bool, error)
}

// UnitReferenceChecker は curriculum(Unit) Context が公開する
// 単元の存在確認・学習開始状態確認の参照手段を表す。
type UnitReferenceChecker interface {
    Exists(ctx context.Context, unitID uint) (bool, error)
    IsStarted(ctx context.Context, unitID, userID uint) (bool, error)
}
```

これらのinterfaceの実装（goal-management Context・curriculum Context側のStore）は本書の対象外である（**②からの補足**。詳細は「14. ②からの補足事項」参照）。

## Domain Service

②「8. Domain Service」TaskUnitRelationPolicyを、独立したstruct/interfaceではなく、`internal/task/task_unit_relation_policy.go`内のpackageレベル関数として実装する（規約4章 Active Record節に従いDomain Serviceは原則「対象外」とするための構造上の読み替え。判断根拠は「14. ②からの補足事項」参照）。

|関数|引数|戻り値|責務|
|-|-|-|-|
|`CanRemoveTaskUnitLink`|`unitStarted bool`|`bool`|学習開始済み単元の紐付けを削除できるかを判定する（②「8. Domain Service」の判定内容そのもの。`unitStarted`は`UnitReferenceChecker.IsStarted`の結果をHandler側で取得して渡す）|

## Domain Event

②「15. Domain Event」により、本機能ではDomain Eventを採用しない。「対象外」とする。

## Domain Error

規約4章「Active Record」節に従い、「struct/Storeが返すエラー」として`internal/task/errors.go`に`sentinel error`を定義する。

|変数名|発生条件|対応する②の記載|
|-|-|-|
|`ErrTaskNotFound`|指定IDのTaskが存在しない、または所有者が一致しない|②「14. Error設計」タスク未存在|
|`ErrInvalidStatusTransition`|`Task.ChangeStatus`で許容されない遷移が指定された|②「14. Error設計」不正な状態遷移|
|`ErrGoalNotOwned`|指定goal_idが自分の目標でない|②「14. Error設計」所有者外の目標参照|
|`ErrUnitNotFound`|指定unit_idが存在しない|②「12. Validation設計」フォーマットチェック・整合性チェック|
|`ErrUnitLinkRemovalNotAllowed`|学習開始済み単元の紐付け削除を試みた|②「14. Error設計」学習開始済み単元の削除試行|

---

# 4. Application層設計

**実装上の位置づけ**: 本機能はActive Record採用のためusecase層を設けない。「対象外（Active Record採用のため、usecase層を設けない）」とする。②「10. UseCase設計」に記載された4つの業務操作（ListTasks／ShowTask／CreateTask／UpdateTask）は、Handlerが`internal/task`のStore・Modelを直接呼び出す処理として実装する。処理順序は「6. Presentation層設計」のHandler処理順序に統合して記載する。

## DTO（Command / Query）

DTOはPresentation層のRequest/Response DTOとして「6. Presentation層設計」にまとめて記載する（Active Record採用のため、Application層独自のCommand/Query DTOは設けない）。

---

# 5. Infrastructure層設計

**実装上の位置づけ**: 本機能はActive Record採用のためinfrastructure層のディレクトリ分離は行わない。「3. Domain層設計」で定義したModelと同一package（`internal/task`）にStoreを置く。

## Store実装

### TaskStore（`internal/task/task_store.go`）

- struct名: `TaskStore`
- 対応するGORMモデル: `Task`（同一struct。GORMタグは付与せず、規約のデフォルト命名規則（フィールド名→snake_caseカラム名）に委ねる。「12. GORM/DBクエリ設計」参照）

コンストラクタ:

```go
func NewTaskStore(db *gorm.DB) *TaskStore
```

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindByIDForUser`|`(ctx context.Context, id, userID uint)`|`(*Task, error)`|`id`と`user_id`の一致で1件取得する所有者スコープ検索（②「9. Repository設計」所有者スコープでの検索）。該当なしは`ErrTaskNotFound`を返す|
|`FindAllByUser`|`(ctx context.Context, userID uint, status *TaskStatus, page, perPage int)`|`([]*Task, int64, error)`|`user_id`一致を必須条件とし、`status`指定時のみ絞り込みを追加、`due_date`昇順ソート、`page`/`perPage`によるOFFSET/LIMITページネーションを行う（②「9. Repository設計」保持する検索機能）。件数取得はCOUNTクエリを別途発行する|
|`CreateWithUnitLinks`|`(ctx context.Context, t *Task, unitIDs []uint)`|`error`|Task作成とTaskUnitLink一括追加を1トランザクションで実行する（②「10. UseCase設計」CreateTaskのトランザクション範囲。「8. Transaction実装方針」参照）|
|`UpdateWithUnitLinks`|`(ctx context.Context, t *Task, addUnitIDs, removeUnitIDs []uint)`|`error`|Task更新と、指定された追加・削除対象のTaskUnitLink同期を1トランザクションで実行する（②「10. UseCase設計」UpdateTaskのトランザクション範囲。削除可否の判定は事前にHandler側で完了している前提とする。「8. Transaction実装方針」参照）|

保持しない責務（②「9. Repository設計」保持しない責務を踏襲）: 業務ルール判定・認可判定・状態遷移の判断・単元紐付けの削除可否判定はStoreに持たせない。

### TaskUnitLinkStore（`internal/task/task_unit_link_store.go`）

- struct名: `TaskUnitLinkStore`
- 対応するGORMモデル: `TaskUnitLink`（テーブル名`task_units`。「12. GORM/DBクエリ設計」参照）

コンストラクタ:

```go
func NewTaskUnitLinkStore(db *gorm.DB) *TaskUnitLinkStore
```

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindUnitIDsByTaskID`|`(ctx context.Context, taskID uint)`|`([]uint, error)`|`task_id`一致で紐づく`unit_id`一覧を取得する（②「9. Repository設計」タスクIDによる関連単元取得）|
|`Add`|`(ctx context.Context, taskID uint, unitIDs []uint)`|`error`|指定unitIDs分のTaskUnitLinkレコードを一括作成する（②「9. Repository設計」紐付けの追加）|
|`Remove`|`(ctx context.Context, taskID uint, unitIDs []uint)`|`error`|`task_id`一致かつ`unit_id IN (...)`条件でレコードを削除する（②「9. Repository設計」紐付けの削除）|

`TaskStore.CreateWithUnitLinks`／`UpdateWithUnitLinks`は、トランザクション用の`*gorm.DB`を`NewTaskUnitLinkStore`に渡して同一トランザクション内でTaskUnitLinkStoreのメソッドを呼び出す（Entity ⇔ GORMモデルの変換は不要。同一structをそのまま永続化する）。

Entity ⇔ GORMモデルの変換方針: Active Record採用のため、`Task`・`TaskUnitLink`のstructそのものをGORMの操作対象として扱う。変換処理は設けない。

## 外部連携実装

②にMail・Cache・Queue等の外部連携に関する記載はない。「対象外」とする。

---

# 6. Presentation層設計

## Handler

### TaskHandler（`internal/task/presentation/handler/task_handler.go`）

- struct名: `TaskHandler`
- 依存: `*task.TaskStore`、`*task.TaskUnitLinkStore`、`task.GoalOwnershipChecker`、`task.UnitReferenceChecker`（コンストラクタで注入する）

```go
func NewTaskHandler(
    taskStore *task.TaskStore,
    taskUnitLinkStore *task.TaskUnitLinkStore,
    goalChecker task.GoalOwnershipChecker,
    unitChecker task.UnitReferenceChecker,
) *TaskHandler
```

メソッド一覧（HTTPメソッド・パスとの対応は「7. API仕様」参照）:

|メソッド|対応API|
|-|-|
|`(h *TaskHandler) ListTasks(c *gin.Context)`|GET /api/v1/student/tasks|
|`(h *TaskHandler) ShowTask(c *gin.Context)`|GET /api/v1/student/tasks/:id|
|`(h *TaskHandler) CreateTask(c *gin.Context)`|POST /api/v1/student/tasks|
|`(h *TaskHandler) UpdateTask(c *gin.Context)`|PATCH /api/v1/student/tasks/:id|

Active Record採用のためUseCase層を経由しない。以下、②「10. UseCase設計」で「Handler処理」として記載された業務操作の呼び出し順序を、権限チェック・呼び出し順序を含めてHandlerの処理順序として記載する（規約8章 横断的関心事の置き場所に基づき、認可（所有権・業務権限）は該当Storeまたは本Handlerで行う）。

#### ListTasks 処理順序

1. Middlewareで設定済みのcurrent user（student）をcontextから取得する
2. クエリパラメータ（`status`・`page`）をRequest DTOにバインドし、型・フォーマットを検証する
3. `TaskStore.FindAllByUser`をcurrent userのuser_idでスコープして呼び出す（②「10. UseCase設計」ListTasks 呼び出すStore: TaskStore）
4. 取得結果をResponse DTOへ変換して返す

#### ShowTask 処理順序

1. current userを取得する
2. パスパラメータ`id`をバインドする
3. `TaskStore.FindByIDForUser`をcurrent userのuser_idでスコープして呼び出す。存在しない場合は`ErrTaskNotFound`（②「10. UseCase設計」ShowTask 判断根拠：所有者外アクセスは404として扱う）
4. `TaskUnitLinkStore.FindUnitIDsByTaskID`で関連単元IDを取得する（②「10. UseCase設計」ShowTask 呼び出すStore: TaskUnitLinkStore）
5. 取得結果をResponse DTOへ変換して返す

#### CreateTask 処理順序

1. current userを取得する
2. Request DTOへバインドし、型・必須・フォーマットを検証する（②「12. Validation設計」Presentation）
3. `GoalOwnershipChecker.ExistsForStudent`で指定goal_idが自分の目標か確認する。該当しない場合は`ErrGoalNotOwned`（②「12. Validation設計」業務ルール：指定されたgoal_idが自分の目標かどうか）
4. 指定された各unit_idについて`UnitReferenceChecker.Exists`で存在確認する。存在しない場合は`ErrUnitNotFound`
5. `task.NormalizePriority`でpriorityを正規化する
6. `task.NewTask`でTaskを生成し、`Task.Validate()`で業務ルールを検証する（②「12. Validation設計」Domain）
7. `TaskStore.CreateWithUnitLinks`をTaskと確認済みunit_ids一覧で呼び出す（②「10. UseCase設計」CreateTask 呼び出すStore：GoalStore→UnitStore→TaskStore→TaskUnitLinkStoreの順に対応）
8. 作成結果をResponse DTOへ変換して返す

#### UpdateTask 処理順序

1. current userを取得する
2. パスパラメータ`id`とRequest DTOをバインドし、型・必須・フォーマットを検証する
3. `TaskStore.FindByIDForUser`で対象Taskを取得する。存在しない場合は`ErrTaskNotFound`
4. status変更が含まれる場合、`Task.ChangeStatus`で遷移の妥当性とCompletedAtの整合性を検証する（②「12. Validation設計」状態チェック）
5. unit_idsの差分（追加対象・削除対象）を算出する
6. 削除対象の各unit_idについて`UnitReferenceChecker.IsStarted`で学習開始状態を取得し、`task.CanRemoveTaskUnitLink`で削除可否を判定する。削除不可の場合は`ErrUnitLinkRemovalNotAllowed`（②「12. Validation設計」整合性チェック：学習開始済み単元を削除しようとした場合の禁止判定、②「8. Domain Service」TaskUnitRelationPolicy）
7. 追加対象の各unit_idについて`UnitReferenceChecker.Exists`で存在確認する
8. `TaskStore.UpdateWithUnitLinks`をTask・追加対象unit_ids・削除対象unit_idsで呼び出す（②「10. UseCase設計」UpdateTask 呼び出すStore：TaskStore・TaskUnitLinkStore・UnitStoreに対応）
9. 更新結果をResponse DTOへ変換して返す

## Request / Response DTO

### Request（`internal/task/presentation/request/task_request.go`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`ListTasksQuery`|`Status string`、`Page int`|`Status`は`binding:"omitempty,oneof=not_started in_progress completed"`。`Page`は`binding:"omitempty,min=1"`|
|`CreateTaskRequest`|`GoalID uint`、`Title string`、`Content string`、`DueDate string`、`Priority any`、`Memo string`、`UnitIDs []uint`|`GoalID`・`Title`・`DueDate`は`binding:"required"`。`DueDate`は日付フォーマット検証（②「12. Validation設計」フォーマットチェック：日付形式）。`Priority`は文字列・数値いずれの形式も受け付ける（②「7. Value Object設計」Priority独自仕様。具体的な許容値は②に明記がないため「14. ②からの補足事項」参照）。`UnitIDs`は配列形式チェック（②「12. Validation設計」フォーマットチェック：unit_idsの配列形式）|
|`UpdateTaskRequest`|`Title *string`、`Content *string`、`DueDate *string`、`Priority any`、`Memo *string`、`Status *string`、`UnitIDs *[]uint`|部分更新のためポインタ型で「未指定」を表現する。`Status`指定時は`binding:"omitempty,oneof=not_started in_progress completed"`|

### Response（`internal/task/presentation/response/task_response.go`）

|struct名|フィールドと型|
|-|-|
|`TaskResponse`|`ID uint`、`GoalID uint`、`Title string`、`Content string`、`DueDate string`、`Priority string`、`Memo string`、`Status string`、`CompletedAt *string`、`UnitIDs []uint`、`CreatedAt string`、`UpdatedAt string`|
|`TaskListResponse`|`Tasks []TaskResponse`、`Page PageInfo`|
|`PageInfo`|`CurrentPage int`、`PerPage int`、`TotalCount int64`、`TotalPages int`|

`DueDate`・`CompletedAt`・`CreatedAt`・`UpdatedAt`は`Task.DueDateDisplay()`等を用いて画面表示用フォーマットへ変換する。EntityであるTaskをそのまま返さず、必ずResponse DTOへ変換する（規約「7. データフロー」）。

## Routing

`internal/task/presentation/routes.go`

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/student/tasks|TaskHandler.ListTasks|
|GET|/api/v1/student/tasks/:id|TaskHandler.ShowTask|
|POST|/api/v1/student/tasks|TaskHandler.CreateTask|
|PATCH|/api/v1/student/tasks/:id|TaskHandler.UpdateTask|

いずれのルートも認証Middleware（本人確認）・認可Middleware（student roleチェック）を経由する（②「13. Authorization設計」Middleware、「10. Authorization実装方針」参照）。

---

# 7. API仕様

②「16. API互換方針」に基づき、Rails現行仕様と同一のエンドポイントを維持する。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/student/tasks|ListTasks|`ListTasksQuery`（クエリパラメータ）|`TaskListResponse`|200|
|GET|/api/v1/student/tasks/:id|ShowTask|パスパラメータ`id`|`TaskResponse`|200|
|POST|/api/v1/student/tasks|CreateTask|`CreateTaskRequest`|`TaskResponse`|201（②「16. API互換方針」Status Code：作成・更新成功の扱いは既存仕様に合わせて統一する、とあり具体的な使い分けは①未提供のため参照不可。作成=201、更新=200と推測。「14. ②からの補足事項」参照）|
|PATCH|/api/v1/student/tasks/:id|UpdateTask|パスパラメータ`id` + `UpdateTaskRequest`|`TaskResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401|認証エラー（Middleware）|
|student以外のロール|403|認可エラー（Middleware）|
|Request DTOの型・必須・フォーマット不正|422|Presentation Validationエラー（②「16. API互換方針」422: 入力・業務ルール違反）|
|指定goal_idが自分の目標でない（`ErrGoalNotOwned`）|422|業務ルール違反|
|指定unit_idが存在しない（`ErrUnitNotFound`）|422|業務ルール違反|
|不正な状態遷移（`ErrInvalidStatusTransition`）|422|業務ルール違反|
|学習開始済み単元の削除試行（`ErrUnitLinkRemovalNotAllowed`）|422|業務ルール違反|
|対象タスク不存在／所有者不一致（`ErrTaskNotFound`）|404|②「16. API互換方針」404: 対象タスク不存在|
|DB接続失敗等のInfrastructure Error|500|内部エラー|

---

# 8. Transaction実装方針

②「11. Transaction設計」を実装単位に落とし込む。

## Transaction開始箇所

- Active Record採用のため、`TaskStore.CreateWithUnitLinks`・`TaskStore.UpdateWithUnitLinks`メソッド内で`db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })`を用いてトランザクションを開始する（②「11. Transaction設計」Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する）
- `ListTasks`・`ShowTask`はトランザクションを使用しない（②「11. Transaction設計」）

## Transaction終了箇所（Commit / Rollback条件）

- `CreateWithUnitLinks`: Task作成とTaskUnitLink一括追加がすべて成功した時点でコミットする。いずれかが失敗した場合はロールバックする
- `UpdateWithUnitLinks`: Task更新とTaskUnitLinkの追加・削除同期がすべて成功した時点でコミットする。いずれかが失敗した場合はロールバックする

## 複数Storeにまたがる場合の扱い

`TaskStore.CreateWithUnitLinks`／`UpdateWithUnitLinks`は、トランザクション用の`tx *gorm.DB`を用いて`TaskUnitLinkStore`を`NewTaskUnitLinkStore(tx)`で生成し、同一トランザクション内でTaskUnitLinkStoreのメソッドを呼び出す。GoalOwnershipChecker・UnitReferenceCheckerによる確認は、トランザクション開始前（Handler側）で完了させ、トランザクション内では行わない（②「9. Repository設計」GoalStore/UnitStoreは「タスクの作成可否判断そのもの」を保持しないとされている点と整合）。

---

# 9. Validation実装方針

②「12. Validation設計」を実装レベルに落とし込む。

## Presentation

- 型チェック: `binding`タグによるRequest DTOの型検証
- 必須チェック: `CreateTaskRequest`の`GoalID`・`Title`・`DueDate`に`binding:"required"`
- フォーマットチェック: `DueDate`の日付形式検証、`Status`の`oneof`検証、`UnitIDs`の配列形式検証

## 業務ルール検証（Active Record採用時: Modelのメソッドで検証する内容）

- `Task.Validate()`: title等必須項目・値域の妥当性
- `Task.ChangeStatus()`: 状態遷移の妥当性、completed時のCompletedAt整合性（②「12. Validation設計」状態チェック）
- `NormalizePriority()`: priority入力値の正規化・妥当性

## 業務ルール検証（Model単体では完結しないため、Handlerで実行する内容）

goal_idの所有権確認（`GoalOwnershipChecker`）、unit_idの存在確認（`UnitReferenceChecker.Exists`）、学習開始済み単元の削除禁止判定（`UnitReferenceChecker.IsStarted` + `CanRemoveTaskUnitLink`）は、他Contextのデータを参照する業務ルールであり、Task Model単体では検証できないため、Handlerが呼び出し元となって検証する（②「12. Validation設計」業務ルール・整合性チェックの実装先を、Active Record構造に合わせて具体化したもの。**②からの補足**。詳細は「14. ②からの補足事項」参照）。

## 責務分離

②の方針どおり、Presentationは「入力が正しいか」、Model／Handlerは「業務的に妥当か」を担当する。

---

# 10. Authorization実装方針

②「13. Authorization設計」を実装レベルに落とし込む。

## Middleware

- JWT等の検証を行い、current userをcontextに格納する（規約「8. 横断的関心事の置き場所」認証）
- ロールがstudentであることを確認する（②「13. Authorization設計」Middleware）

## Handler

- Middlewareが認証・ロール認可を行うため、Handler自体は個別の業務権限判定を持たない（②「13. Authorization設計」Handler）
- ただし、Active Record採用によりUseCase層がないため、所有者スコープでの取得（`TaskStore.FindByIDForUser`・`FindAllByUser`にuser_idを渡す）と、goal_id所有権確認（`GoalOwnershipChecker`）はHandlerが呼び出す（②「13. Authorization設計」UseCaseの記載を、Active Record構造上Handlerに読み替え）

## Store／Model

- `TaskStore`の検索メソッドは、渡されたuser_idによるスコープを常に条件へ含める（②「13. Authorization設計」Domain：所有者外のアクセスが行われないようにする、を実装レベルで反映）
- Model（`Task`）は`IsOwnedBy`メソッドを提供するのみで、認可の主体としては扱わない

---

# 11. Error実装方針

②「14. Error設計」を実装レベルに落とし込む。

## Domain Error → Application Errorへの変換方針

Active Record採用のためDomain Error層は独立させず、「3. Domain層設計」の`errors.go`で定義したsentinel error（`ErrTaskNotFound`等）をそのままApplication Error相当として扱う（規約8章 横断的関心事の置き場所「Transaction Script/Active Record採用時はDomain Errorに相当する層がないため、関数・struct側で発生したエラーをApplication Error相当として扱う」）。

## Application Error → HTTPレスポンスへの変換方針

Handlerが`errors.Is`でsentinel errorを判定し、対応するHTTP Status Codeへ変換する。

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrTaskNotFound`|Store|404|
|`ErrGoalNotOwned`|Handler（GoalOwnershipChecker経由）|422|
|`ErrUnitNotFound`|Handler（UnitReferenceChecker経由）|422|
|`ErrInvalidStatusTransition`|Model（Task.ChangeStatus）|422|
|`ErrUnitLinkRemovalNotAllowed`|Handler（CanRemoveTaskUnitLink経由）|422|
|Request DTOバインド／バリデーションエラー|Presentation|422|
|未認証|Middleware|401|
|ロール不一致|Middleware|403|
|上記以外（DB接続失敗等）|Store／Infrastructure|500|

## Infrastructure Errorのハンドリング方針

GORMが返すDB接続エラー等は、sentinel errorとして特別扱いせず、Storeからそのまま呼び出し元へ返し、Handlerでハンドリングされなかった場合は共通のエラーハンドリングMiddleware（本機能固有の設計ではないため対象外。`shared/`側の既存実装に従う）で500として応答する。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」により、既存Rails DBをそのまま継続利用し、Schema変更は行わない。

## 利用するGORMモデルとテーブルの対応

|Goモデル|テーブル名|備考|
|-|-|-|
|`Task`|`tasks`|GORMのデフォルト命名規則（struct名の複数形snake_case）で一致するため、`TableName()`のオーバーライドは不要（Gorm規約「複数形のテーブル名」）|
|`TaskUnitLink`|`task_units`|struct名のデフォルト複数形は`task_unit_links`だが、②「17. DB設計方針」の記載どおり実際のテーブル名は`task_units`であるため、`Tabler`インターフェース（`func (TaskUnitLink) TableName() string { return "task_units" }`）による明示的な上書きが必要（Gorm規約「テーブル名」。**②からの補足**：②はテーブル名を明記しているのみで、GoモデルとGORM命名規則の不一致には触れていないため、③側で気づいて反映した）|

## 主要クエリの条件・ソート・ページネーション方針

- `TaskStore.FindAllByUser`: `user_id`一致必須、`status`指定時のみ絞り込み、`due_date`昇順、`page`/`perPage`によるOFFSET/LIMIT（②「9. Repository設計」保持する検索機能）
- `TaskStore.FindByIDForUser`: `id`かつ`user_id`一致
- `TaskUnitLinkStore.FindUnitIDsByTaskID`: `task_id`一致
- `TaskUnitLinkStore.Add`/`Remove`: `task_id`と`unit_id`の組み合わせによる一括作成・削除

SQL文そのものは本書に記載しない。

## 既存Schemaに対する変更

②「17. DB設計方針」により変更なし。追加スキーマは不要。

---

# 13. テストケース設計

②「18. テスト戦略」を、Active Record採用時の区分（規約4章／指示書「Active Record採用時: 『Domain Test』→『Model Test』、『UseCase Test』は『対象外』、『Repository Test』→『Store Test』」）に読み替えて具体化する。

## Model Test（②「Domain Test」からの読み替え）

|対象|テストケース|
|-|-|
|`Task.ChangeStatus`|not_started→in_progress→completedの正常遷移でCompletedAtが設定されること|
|`Task.ChangeStatus`|completedから他状態への遷移でCompletedAtがクリアされること|
|`Task.ChangeStatus`|許容されない遷移で`ErrInvalidStatusTransition`相当のエラーが返ること|
|`Task.Validate`|title未設定等の必須項目欠落を検出すること|
|`NormalizePriority`|文字列入力・数値入力それぞれで正規化された値が得られること（許容値は「14. ②からの補足事項」参照）|
|`CanRemoveTaskUnitLink`|学習開始済み（true）の場合にfalseを返すこと、未開始（false）の場合にtrueを返すこと|

## UseCase Test

対象外（Active Record採用のため、usecase層を設けない）。

## Store Test（②「Repository Test」からの読み替え）

|対象|テストケース|
|-|-|
|`TaskStore.FindAllByUser`|status絞り込み・due_date昇順・ページネーションが正しく機能すること|
|`TaskStore.FindByIDForUser`|所有者が異なるタスクは取得できず`ErrTaskNotFound`となること|
|`TaskStore.CreateWithUnitLinks`|Task作成とTaskUnitLink追加が同一トランザクションで成功すること|
|`TaskStore.CreateWithUnitLinks`|TaskUnitLink追加が失敗した場合にTask作成もロールバックされること|
|`TaskStore.UpdateWithUnitLinks`|追加・削除対象unit_idsが正しく反映されること|
|`TaskUnitLinkStore.FindUnitIDsByTaskID`|指定task_idに紐づくunit_id一覧が取得できること|

## Handler Test

|対象|テストケース|
|-|-|
|`ListTasks`|status・pageクエリパラメータのバリデーション結果とHTTPステータスの対応|
|`ShowTask`|存在しないtask idで404が返ること|
|`CreateTask`|必須項目欠落時に422が返ること|
|`CreateTask`|goal_idが他人の目標の場合に422が返ること|
|`UpdateTask`|学習開始済み単元の削除を試みた場合に422が返ること|
|全Handler|未認証・非student roleでのアクセスが401/403となること|

## Integration Test

|対象|テストケース|
|-|-|
|タスク作成〜一覧取得〜詳細取得〜更新|一連のエンドポイント呼び出しが正常に完了し、レスポンスの整合性が保たれること（②「18. テスト戦略」Integration Test）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測か否か|
|-|-|-|
|Bounded Context `task-management` の内部ディレクトリ名を`internal/task`とした|②はContext名（task-management）のみを記載し、内部ディレクトリ名を明記していない。`規約/アーキテクチャ規約.md`「2. ディレクトリ構成」のプロジェクト全体構成に列挙された`internal/task/`を、5章の上位ドメイン「Task」およびタスク管理機能の対応として採用した|推測（規約の記載からの妥当な読み替えだが、②文書内に明示的な対応付けの記載はない）|
|GoalStore・UnitStoreをtask-management内で実装せず、task package側に参照用interface（`GoalOwnershipChecker`／`UnitReferenceChecker`）を定義し、実装はgoal-management／curriculum Context側に委ねる構成とした|②「9. Repository設計」はGoalStore・UnitStoreをtask-managementの節内で記載しているが、`規約/アーキテクチャ規約.md`「6. Context間連携ルール」により他Contextの内部実装に直接依存できないため、コーディング規約「5. インターフェース」（利用側での定義）に従って整理した。②の「責務」「保持しない責務」の記載内容自体は変更していない|補足（規約の適用による構造上の具体化であり、推測ではなく規約遵守のための判断）|
|②「8. Domain Service」TaskUnitRelationPolicyを、独立したstruct/interfaceではなく`internal/task`パッケージ内のpackageレベル関数`CanRemoveTaskUnitLink`として実装した|`規約/アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」Active Record節により、Domain Serviceに相当する独立層は原則「対象外」とされているため。判定内容そのものは②の記載を変更していない|補足（規約適用）|
|②「7. Value Object設計」TaskStatus／DueDate／Priorityを独立したValue Object structではなく、Taskの型・メソッドとして実装した|同上、規約4章Active Record節により、Value Objectは原則「対象外」とされているため|補足（規約適用）|
|`TaskUnitLink`のGORMテーブル名を`task_units`として`TableName()`で明示的に上書きする必要がある点|②「17. DB設計方針」はテーブル名`task_units`を利用する旨のみ記載しており、Go構造体`TaskUnitLink`のデフォルト複数形（`task_unit_links`）との不一致には触れていない。Gorm規約「テーブル名」に基づき③側で明示した|補足（Gorm規約の適用によりテーブル不一致を検出したもの）|
|`Priority`の正規化後の具体的な許容値集合（例："low"/"medium"/"high"等）|②「7. Value Object設計」は「文字列・整数の両方で受け取る」「一貫した値へ正規化する」とのみ記載し、具体的なマッピングルール・値集合を明記していない。①未提供のため参照不可|推測（実装着手前に要件確認が必要）|
|`GoalID`が必須項目（nullを許容しない）である前提とした|②「12. Validation設計」に「指定されたgoal_idが自分の目標かどうか」という検証項目があることから必須と推測したが、②はnull許容性を明記していない。①未提供のため参照不可|推測|
|作成時のHTTP Status Codeを201とした（②は「作成・更新成功の扱いは既存仕様に合わせて統一する」とのみ記載）|②「16. API互換方針」Status Codeの記載が「201/200: 作成・更新成功の扱いは既存仕様に合わせて統一する」と曖昧であり、①未提供のため実際のRails挙動を参照できない。REST慣例に基づき作成=201、更新=200と仮置きした|推測（実装着手前にRails現行仕様の確認を推奨）|
|`TaskStore.CreateWithUnitLinks`／`UpdateWithUnitLinks`というメソッド名・トランザクションの起点をStoreの単一メソッドに集約する具体的な設計|②「11. Transaction設計」は「Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する」という方針のみを記載しており、具体的なメソッド名・分割単位は指定していない|補足（②の方針を実装可能な粒度に具体化したもの）|

