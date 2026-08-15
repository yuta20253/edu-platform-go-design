# 目標管理機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒が自分の学習目標（Goal）を登録・参照・更新する機能である。提供する操作は一覧・詳細（紐づくタスク一覧を含む）・作成・更新の4つ。Goalは`status`（not_started / in_progress / completed）を保持するが、②Go移行・設計仕様書に明記されているとおり、本機能のAPI内には状態遷移ロジック（遷移条件・遷移禁止ルール等）は存在しない。`status`は値として保持・表示されるのみである。

## 採用設計パターンとその理由（②からの要約）

- 採用パターン: **Active Record**（②「4. 設計パターン」）
- 判断根拠: 目標データの主要操作は一覧・詳細・作成・更新というCRUD中心の業務であり、`status`属性を持つものの現行仕様に遷移ルールが存在しないため、Domain Model採用基準（状態遷移ルールが存在する場合に採用）を満たさない。所有権チェックや期限（DueDate）の妥当性検証をEntity相当のstructに寄せることで保守性を確保する方が、手続き型のTransaction Scriptより適切と判断されている
- 採用しなかったパターン: Transaction Script（バリデーション以上のルールが将来増えると手続きが肥大化しやすいため）、Domain Model（遷移ルールがなく過剰設計になるため）、Event Sourcing（状態遷移イベントの再構築要件がないため）

## 本書が対象とする実装範囲

②「9. Repository設計」〜「18. テスト戦略」で示された、goal-management Bounded ContextにおけるGoalの一覧・詳細・作成・更新のActive Record実装（`model.go`相当のstruct定義、`store.go`相当の永続化処理、Presentation層のHandler/Request/Response/Routing）。①Rails実装の詳細は未提供のため参照できず、本書では言及箇所を「①未提供のため参照不可」として明示する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- goal-management（②「3. Bounded Context」）

## ②で採用した設計パターン

- Active Record（②「4. 設計パターン」）

## ディレクトリ・パッケージ名について（②からの補足）

②にはGo実装上のディレクトリ名（`internal/`配下のパッケージ名）の明記がない。アーキテクチャ規約「5. Bounded Context構成 命名規則」に基づき、既存の`task-management`→`internal/task`の対応関係に倣い、`goal-management`→`internal/goal`とする。**この対応は②に明記がないため推測である。**

## 作成するディレクトリ一覧

アーキテクチャ規約「4. 設計パターンごとの構造適用方針」のActive Record構造に従う。domain/infrastructureのレイヤー分離・usecase層は設けない。

```
internal/goal/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── (routes.go はディレクトリ直下)
```

## 作成するファイル一覧

```
internal/goal/goal.go              # Goal struct（Model）+ Validate()
internal/goal/goal_status.go       # GoalStatus型（許容値の型定義）
internal/goal/due_date.go          # DueDate型（保存値/表示フォーマットの一元化）
internal/goal/errors.go            # struct/Storeが返すエラー定義
internal/goal/goal_store.go        # GoalStore（永続化・検索）
internal/goal/task_ref.go          # TaskRef struct（参照専用の読み取りモデル）
internal/goal/task_store.go        # TaskStore（参照専用の永続化・検索）
internal/goal/presentation/handler/goal_handler.go
internal/goal/presentation/request/goal_request.go
internal/goal/presentation/response/goal_response.go
internal/goal/presentation/routes.go
```

**②からの補足**: ②の構造適用方針の例示は`model.go`/`store.go`という単一ファイルの雛形だが、本機能はGoal（Model）とTask参照（読み取り専用モデル）という2種類のstructを扱うため、責務ごとにファイルを分割している。これはActive Record構造（domain/infrastructureレイヤーを分離しない、同一package内に置く）という②の設計方針自体は変更せず、同一package内でのファイル分割という実装レベルの整理にとどまる。

---

# 3. Domain層設計

**実装上の位置づけ**: 本機能はActive Record採用のため、domain層・infrastructure層のレイヤー分離を行わない。以下は「Entity」の代わりに「Model（Entity相当）」として、`internal/goal`パッケージ直下のstructとして扱う（アーキテクチャ規約「4. 設計パターンごとの構造適用方針」Active Record節）。

## Model（Entity相当）

### Goal（`goal.go`）

②「6. Entity設計」「7. Value Object設計」に基づくGoalのGoフィールド定義。

|フィールド名|型|意味|
|-|-|-|
|ID|uint|Goalの識別子|
|UserID|uint|所有者（生徒）のユーザーID。②「Authorization設計」の所有権チェックの材料|
|Title|string|目標タイトル。必須項目（②「12. Validation設計」）|
|Description|string|目標詳細|
|DueDate|DueDate|期限。②「7. Value Object設計」のDueDateを型として保持|
|Status|GoalStatus|状態値。②「7. Value Object設計」のGoalStatusを型として保持。遷移ロジックは持たない|
|CreatedAt|time.Time|作成日時|
|UpdatedAt|time.Time|更新日時|

公開メソッド:

|メソッド|引数|戻り値|責務|
|-|-|-|-|
|`(g *Goal) Validate()`|なし|`error`|Title必須チェック、DueDateの妥当性、Statusの許容値チェックを行う（②「12. Validation設計」Domain節に対応）。ロジックの実装内容は本書では規定しない|

不変条件: Titleが空でないこと、DueDateが妥当な値であること、Statusが許容値（GoalStatus）の範囲内であること。②に明記のとおり、遷移の妥当性（前の状態から今の状態へ遷移してよいか）は検証対象に含めない。

**②からの補足**: `Status`をAPIの作成・更新リクエストに含めるかどうかは②に明記がない。②「4. 設計パターン」で「statusの変更ロジックはこの機能のAPI内では直接定義されていない」とされていることから、本書では作成時にStatusを内部で初期値（`not_started`）として設定し、更新リクエストの入力対象には含めない前提とする。**これは②に明記がないための推測であり、実装前に①（Rails実装）またはプロダクトオーナーへの確認を推奨する。**

### GoalStatus（`goal_status.go`）

②「7. Value Object設計」のGoalStatusに対応する型。Active Record方針上、独立したValue Object層は設けず、Goalの構成要素としてpackage内の付随型として扱う。

|項目|内容|
|-|-|
|型定義|`type GoalStatus string`|
|許容値|`not_started` / `in_progress` / `completed`（②に定義された3値の定数として表現）|
|公開メソッド|`IsValid() bool`（許容値内かどうかを判定する。遷移ルールは持たない）|

### DueDate（`due_date.go`）

②「7. Value Object設計」のDueDateに対応する型。同様に独立層は設けず、Goalの構成要素として扱う。

|項目|内容|
|-|-|
|保持するフィールド|内部にtime.Time相当の値を1つ保持する（保存値）|
|生成方法|文字列から生成するコンストラクタ相当の関数（引数・戻り値のみ規定し、パース処理の実装内容は本書では規定しない）|
|公開メソッド|`String() string`（`YYYY/MM/DD`形式の表示用文字列を返す。②「7. Value Object設計」に基づく）、比較用メソッド（Before/After等、期限の前後比較を提供する。②に「比較演算を提供する」とのみ記載があり、具体的なメソッド名は本書での補足）|

**②からの補足**: DueDateの生成時に受け取る入力文字列のフォーマット（リクエストで送信される形式）は②に明記がない。②が明示するのは表示用フォーマット（`YYYY/MM/DD`）のみである。①未提供のため参照不可。本書ではRequest DTO側でISO 8601形式（`YYYY-MM-DD`）を受け取り、DueDate生成時に変換する前提を置くが、**これは推測であり実装前に確認が必要**。

## Value Object

対象外（Active Record採用のため独立したValue Object層は設けない）。②が設計したGoalStatus・DueDateは、上記のとおりGoalと同一package内の付随型として扱う。

## Repository Interface

対象外（Active Record採用のため、domain層にRepository Interfaceを定義しない。永続化・検索はEntity相当のstructと同一package内のStoreとして「5. Infrastructure層設計」に記載する）。

## Domain Service

対象外。②「8. Domain Service」のとおり、複数Entityを横断する業務ルールは現状存在しないため不要と判断されている。

## Domain Event

対象外。②「15. Domain Event」のとおり、非同期の副作用が現行仕様に存在しないため採用しない。

## Domain Error（struct/Storeが返すエラー）

②「14. Error設計」Domain Errorに対応する内容を、`errors.go`内のセンチネルエラー（`error`型の変数）として定義する方針とする。

|エラー|発生条件|
|-|-|
|不正なDueDateを表すエラー|DueDateの生成・Validate()時に、期限として不正な値が渡された場合|
|不正なStatusを表すエラー|Validate()時に、GoalStatusが許容値外の場合|
|対象Goal不存在を表すエラー|GoalStoreが指定ID・所有者に一致するGoalを取得できなかった場合（②「9. Repository設計」・「14. Error設計」Application Errorの404相当に対応する材料として、Store層で定義する）|

---

# 4. Application層設計

対象外（Active Record採用のため、usecase層を設けない。②「10. UseCase設計」に記載されたListGoals/ShowGoal/CreateGoal/UpdateGoalの業務操作は、HandlerがGoalStore/TaskStoreを直接呼び出す処理として実装する。処理順序は「6. Presentation層設計」のHandler処理順序に統合して記載する）。

---

# 5. Infrastructure層設計

**実装上の位置づけ**: 本機能はActive Record採用のため、Repository Interfaceの実装という形は取らない。②「9. Repository設計」で定義された責務を、Model（Goal / TaskRef）と同一package内のStoreとして実装する。

## Store実装

### GoalStore（`goal_store.go`）

|項目|内容|
|-|-|
|struct名|`GoalStore`|
|対応するGORMモデル|`Goal`struct自体をGORMモデルとして扱う（Modelと同一structをGORMタグ付きで扱う方針。詳細は「12. GORM / DBクエリ設計」参照）|
|保持するフィールド|`db *gorm.DB`|
|コンストラクタ|`NewGoalStore(db *gorm.DB) *GoalStore`|

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindByUserID`|`ctx context.Context, userID uint, page, perPage int`|`([]Goal, int64, error)`|user_idによる絞り込み、due_date昇順ソート、ページネーション（LIMIT/OFFSET相当）、および総件数取得（②「9. Repository設計」の検索機能に対応）|
|`FindByIDAndUserID`|`ctx context.Context, id, userID uint`|`(*Goal, error)`|id・user_idの両方に一致する1件取得。一致しない場合はGoal不存在エラー（他ユーザーの目標を含む404相当。②「13. Authorization設計」の所有権チェックをクエリ条件として実現する）|
|`Create`|`ctx context.Context, goal *Goal`|`error`|Goalレコードの新規作成|
|`Update`|`ctx context.Context, goal *Goal`|`error`|id（および所有権確認済みのuser_id）に一致するGoalレコードの更新|

②「9. Repository設計」に明記のとおり、業務ルール判定・認可判定はStoreに持たせない。所有権チェックは「id・user_idが一致するレコードを取得できるか」というクエリ条件として表現し、判定ロジック自体はHandler側の処理順序で扱う。

### TaskStore（参照用、`task_store.go`）

|項目|内容|
|-|-|
|struct名|`TaskStore`|
|対応するGORMモデル|`TaskRef`（読み取り専用モデル。実体のタスク管理は別Context（task-management）の責務であり、本Contextでは参照専用の投影として扱う）|
|保持するフィールド|`db *gorm.DB`|
|コンストラクタ|`NewTaskStore(db *gorm.DB) *TaskStore`|

メソッド一覧:

|メソッド|引数|戻り値|発行するクエリ内容|
|-|-|-|-|
|`FindByGoalID`|`ctx context.Context, goalID uint`|`([]TaskRef, error)`|指定goal_idに紐づくタスク一覧の取得（②「9. Repository設計」TaskStoreの責務どおり読み取り専用）|

②「9. Repository設計」のとおり、TaskStoreはタスクの作成・更新責務を持たない。

**②からの補足**: アーキテクチャ規約「6. Context間連携ルール」では、他Contextのデータ利用時は相手Contextが公開する参照手段（Storeの参照系メソッド）を呼び出す方針が示されている。②「9. Repository設計」は本Context内にTaskStore（参照用）を配置する設計をすでに決定しているため、本書もその決定をそのまま踏襲する。将来的にtask-management Context側が公開APIとして参照系メソッドを整備した場合は、TaskStoreの実装をそちらの呼び出しに置き換える余地がある点を留意事項として補足する。

## 外部連携実装

対象外。②にMail・Cache・Queue等の外部連携は記載されていない。

---

# 6. Presentation層設計

## Handler

|項目|内容|
|-|-|
|struct名|`GoalHandler`|
|対応する呼び出し先|`GoalStore`、`TaskStore`（Active Record採用のためUseCaseを経由せず直接呼び出す）|
|保持するフィールド|`goalStore *GoalStore`, `taskStore *TaskStore`|
|コンストラクタ|`NewGoalHandler(goalStore *GoalStore, taskStore *TaskStore) *GoalHandler`|

メソッド一覧（HTTPメソッド・パス対応）:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`ListGoals`|GET|/api/v1/student/goals|
|`ShowGoal`|GET|/api/v1/student/goals/:id|
|`CreateGoal`|POST|/api/v1/student/goals|
|`UpdateGoal`|PATCH|/api/v1/student/goals/:id|

処理順序（UseCase層を経由しない分、本来UseCaseが担う手順をここに明記する）:

### ListGoals

1. `req.Context()`からcurrent user（②「13. Authorization設計」で特定済みのstudentユーザー）を取得する
2. クエリパラメータ（page, per_page）をバインドする
3. `goalStore.FindByUserID`をcurrent userのIDで呼び出す（②「10. UseCase設計」ListGoalsの入出力に対応）
4. 取得したGoal一覧をResponse DTOへ変換する
5. 200 OKで返す

### ShowGoal

1. current userを取得する
2. パスパラメータ`id`をバインドする
3. `goalStore.FindByIDAndUserID`をid・current user IDで呼び出す（②「13. Authorization設計」の所有権チェックに対応。一致しない場合はGoal不存在エラー→404）
4. `taskStore.FindByGoalID`で紐づくタスク一覧を取得する（②「10. UseCase設計」ShowGoalの出力に対応）
5. Goal詳細とタスク一覧をResponse DTOへ変換する
6. 200 OKで返す

### CreateGoal

1. current userを取得する
2. Request DTO（CreateGoalRequest）へのバインドとPresentationレベルのバリデーションを行う
3. Request DTOからGoal（Model）を組み立てる（UserIDはcurrent userのIDを設定し、Statusは初期値を設定する。②からの補足: Statusの初期値設定は②に明記なし、推測）
4. `goal.Validate()`で業務ルール検証を行う
5. `goalStore.Create`を呼び出す（②「11. Transaction設計」のとおりGoal作成を1トランザクションとして扱う）
6. 作成結果（id）をResponse DTOへ変換する
7. 成功時のステータスコードで返す（②からの補足: 具体的なステータスコードは②に明記がなく推測。「7. API仕様」参照）

### UpdateGoal

1. current userを取得する
2. パスパラメータ`id`とRequest DTO（UpdateGoalRequest）へのバインドとPresentationレベルのバリデーションを行う
3. `goalStore.FindByIDAndUserID`で対象Goalを取得する（所有権チェック。一致しない場合はGoal不存在エラー→404。②「13. Authorization設計」の「取得・更新対象の目標が自分のものかどうかをHandler/Store側で判断する」に対応）
4. 取得したGoalにRequest DTOの内容を反映する
5. `goal.Validate()`で業務ルール検証を行う
6. `goalStore.Update`を呼び出す（②「11. Transaction設計」のとおりGoal更新を1トランザクションとして扱う）
7. 更新結果（id）をResponse DTOへ変換する
8. 成功時のステータスコードで返す

## Request / Response DTO

### Request（`presentation/request/goal_request.go`）

|struct名|フィールド|型|バリデーションタグ／チェック内容|
|-|-|-|-|
|`CreateGoalRequest`|Title|string|`binding:"required"`（②「12. Validation設計」必須チェック）|
||Description|string|なし|
||DueDate|string|`binding:"required"`＋フォーマットチェック（②「12. Validation設計」フォーマットチェック。具体的な入力フォーマットは②未記載につき「3. Domain層設計」の補足のとおり推測）|
|`UpdateGoalRequest`|Title|string|`binding:"required"`|
||Description|string|なし|
||DueDate|string|`binding:"required"`＋フォーマットチェック|

**②からの補足**: UpdateGoalRequestの必須項目をCreateGoalRequestと同一とした。②「12. Validation設計」はPresentation章でtitle/due_dateの必須チェックを一般的に述べているのみで、作成と更新でチェック内容を分ける記載はないため、両者を同一とする判断は推測である。

### Response（`presentation/response/goal_response.go`）

|struct名|フィールド|型|
|-|-|-|
|`GoalResponse`|ID|uint|
||Title|string|
||Description|string|
||Status|string|
||DueDate|string（`YYYY/MM/DD`形式。②「16. API互換方針」）|
|`GoalListResponse`|Goals|`[]GoalResponse`|
||Page|int|
||PerPage|int|
||TotalCount|int64|
|`TaskRefResponse`|ID|uint|
||Title|string|
||Status|string|
|`GoalDetailResponse`|GoalResponse|埋め込み|
||Tasks|`[]TaskRefResponse`|
|`CreateGoalResponse`|ID|uint|
|`UpdateGoalResponse`|ID|uint|
|`ErrorResponse`|Errors|`[]string`|

**②からの補足**:
- `GoalListResponse`のページ情報フィールド（Page/PerPage/TotalCount）は②「10. UseCase設計」に「出力: 目標一覧とページ情報」とあるのみで具体的な構造の記載がなく、①未提供のため参照不可。本書での構成は推測である。
- `TaskRefResponse`のフィールド（ID/Title/Status）は②「16. API互換方針」に「紐づくタスク一覧（tasks）を含む構造を維持する」とあるのみで、詳細フィールドの記載がなく、①未提供のため参照不可。本書での構成は推測であり、実装前にタスク管理機能側のレスポンス定義（存在すれば）と①の実際のシリアライザ仕様を確認することを推奨する。
- `ErrorResponse`の構造は②「16. API互換方針」に「既存のerrors形式をそのまま踏襲するか、Goの実装に合わせて再構成する」とあり、いずれを取るか②では確定していない。本書では「Goの実装に合わせて再構成する」を仮に選択した構造とし、**推測である**。

## Routing（`presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/student/goals|GoalHandler.ListGoals|
|GET|/api/v1/student/goals/:id|GoalHandler.ShowGoal|
|POST|/api/v1/student/goals|GoalHandler.CreateGoal|
|PATCH|/api/v1/student/goals/:id|GoalHandler.UpdateGoal|

ルートは`/api/v1/student`グループ配下に登録し、認証・student役割チェックのMiddleware（②「13. Authorization設計」Middleware節）をグループ単位で適用する。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/student/goals|ListGoals|クエリ: page, per_page|GoalListResponse|200|
|GET|/api/v1/student/goals/:id|ShowGoal|パス: id|GoalDetailResponse|200|
|POST|/api/v1/student/goals|CreateGoal|CreateGoalRequest|CreateGoalResponse|201（②に明記なし。作成成功として一般的な値を推測で採用。実際は①または既存仕様に合わせて確認要）|
|PATCH|/api/v1/student/goals/:id|UpdateGoal|パス: id、Body: UpdateGoalRequest|UpdateGoalResponse|200（②に明記なし。推測）|

各Endpointのエラーケース:

### GET /api/v1/student/goals

|条件|Status Code|Error内容|
|-|-|-|
|未認証|401（②「13. Authorization設計」Middleware節に基づく一般的な扱い。具体的なステータスは②に明記なく推測）|認証エラー|
|studentロールでない|403（同上、推測）|権限エラー|

### GET /api/v1/student/goals/:id

|条件|Status Code|Error内容|
|-|-|-|
|対象Goalが存在しない、または他ユーザーのGoalである|404|Goal不存在エラー（②「14. Error設計」Application Errorに対応）|

### POST /api/v1/student/goals

|条件|Status Code|Error内容|
|-|-|-|
|title/due_date未入力、フォーマット不正|422|バリデーションエラー（②「16. API互換方針」Status Code節）|

### PATCH /api/v1/student/goals/:id

|条件|Status Code|Error内容|
|-|-|-|
|対象Goalが存在しない、または他ユーザーのGoalである|404|Goal不存在エラー|
|title/due_date未入力、フォーマット不正|422|バリデーションエラー|

---

# 8. Transaction実装方針

## Transaction開始位置

②「11. Transaction設計」のとおり、Handlerの処理単位に対応するStoreメソッド内でトランザクションを開始する。具体的には`GoalStore.Create`および`GoalStore.Update`メソッド内でトランザクションを開始する。

## Transaction終了位置

- `GoalStore.Create`: Goalの保存が完了した時点でコミットする。保存に失敗した場合はロールバックする
- `GoalStore.Update`: Goalの更新が完了した時点でコミットする。更新に失敗した場合はロールバックする
- `GoalStore.FindByUserID` / `GoalStore.FindByIDAndUserID` / `TaskStore.FindByGoalID`: 読み取りのみのためトランザクションを使用しない（②「11. Transaction設計」）

## 複数Storeにまたがる場合の扱い

②「9. Repository設計」「11. Transaction設計」のとおり、Goal単体の作成・更新であり複数テーブルにまたがる整合性維持は不要なため、GoalStoreとTaskStoreを1つのトランザクションにまとめる処理は行わない。ShowGoalでのTaskStore呼び出しは読み取り専用の参照であり、GoalStoreの処理とは独立して扱う。

---

# 9. Validation実装方針

## Presentation

- `CreateGoalRequest` / `UpdateGoalRequest`のバインド時に、Ginのbindingタグで型チェック・必須チェックを行う（Title, DueDateの`required`）
- DueDateの日付フォーマットチェックは、バインド後の追加チェック処理として行う（②「12. Validation設計」フォーマットチェックに対応。具体的な検証方法・入力フォーマットは②に明記がなく、「6. Presentation層設計」の補足のとおり推測）

## 業務ルール検証

Active Record採用のため、Goal（Model）の`Validate()`メソッドで検証する。

- Titleが空でないこと
- DueDateが妥当な値であること（②「7. Value Object設計」DueDateの独自ルールに対応）
- Statusが許容値（GoalStatus.IsValid()）の範囲内であること。②に明記のとおり、遷移の妥当性は検証しない
- 所有権チェック（他ユーザーのGoalへのアクセス・更新禁止）は、Validate()ではなくStoreの検索条件（`FindByIDAndUserID`）として実現する（②「12. Validation設計」Domain節の「業務ルール: 他ユーザーの目標へのアクセス・更新を禁止する」に対応。②「9. Repository設計」で「業務ルール判定はStoreに持たせない」とされているため、判定そのものはHandler側の処理順序に含め、Storeはその判定に必要なクエリ条件のみを提供する）

---

# 10. Authorization実装方針

- Middlewareで行う処理: JWT等による認証済みユーザーの特定とcontextへの格納、役割（student）であることの確認（②「13. Authorization設計」Middleware節）
- Handlerで行う処理: contextからcurrent userを取り出し、Store呼び出し時のuser_idとして使用する。具体的な業務権限判定はHandler自体には持たせず、Store呼び出しの条件として委譲する（②「13. Authorization設計」Handler節）
- Store／Modelで行う処理（Active Record採用時）: `GoalStore.FindByIDAndUserID`がid・user_idの両方一致を検索条件とすることで所有権チェックを実現する。Goal（Model）自体はUserIDフィールドを保持し、所有権確認の材料を提供するのみで、判定ロジックは持たない（②「13. Authorization設計」Domain節「Goal Entityが所有者（user_id）を保持し、Handler側からの所有権確認の材料を提供する」に対応）

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

`errors.go`で定義したセンチネルエラー（Goal不存在・不正なDueDate・不正なStatus）を、Handlerで判定してHTTPレスポンスへ変換する。②にはDomain ErrorとApplication Errorの明確な変換関数の記載はないため、Active Record構成上はStoreが返すエラーをHandler側で直接HTTPステータスへマッピングする（②「14. Error設計」の責務分離をActive Record構造に合わせて簡略化したもの）。

## Application Error → HTTPレスポンスへの変換方針

Handlerが各エラーの種類を判定し、対応するステータスコードとErrorResponseを返す。

## Infrastructure Errorのハンドリング方針

DB接続失敗・永続化失敗（GORMが返すその他のエラー）は、Goal不存在エラー等の既知エラーと区別し、500番台のステータスとして扱う（②「14. Error設計」Infrastructure Error「永続化層の失敗をドメインに漏らさず、技術的な障害として切り分ける」に対応）。

|Error種別|発生層|HTTP Status|
|-|-|-|
|Goal不存在（他ユーザーのGoalを含む）|Store|404|
|バリデーションエラー（Presentation: 型・必須・フォーマット）|Handler（Request DTOバインド）|422|
|バリデーションエラー（業務ルール: Goal.Validate()）|Model|422|
|認証エラー|Middleware|401（②に明記なく推測）|
|認可エラー（役割不一致）|Middleware|403（②に明記なく推測）|
|DB接続失敗・永続化失敗|Store（Infrastructure相当）|500|

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」のとおり、既存Rails DBを継続利用し、スキーマ変更は行わない。既存の`goals`テーブルを利用する。

## 利用するGORMモデルとテーブルの対応

- `Goal`struct自体をGORMモデルとして扱う（Active Record方針上、Entity相当のstructとGORMモデルを分離しない）。Gorm規約の命名規約に従い、struct名`Goal`は既定でテーブル名`goals`に対応する（`TableName()`のオーバーライドは不要）
- カラム対応はGorm規約の既定命名規則（フィールド名のsnake_case）に従う。`ID`→`id`、`UserID`→`user_id`、`Title`→`title`、`Description`→`description`、`DueDate`→`due_date`、`Status`→`status`、`CreatedAt`→`created_at`、`UpdatedAt`→`updated_at`
- `TaskRef`struct（読み取り専用モデル）は、参照先の`tasks`テーブル（task-management Context側が所有するスキーマ）に対応する読み取り専用モデルとして扱う。書き込みは行わない

**②からの補足（GORMとValue Objectの対応）**: `DueDate`・`GoalStatus`は独立したValue Object層を持たず、Goalのフィールド型として存在する。GORMがこれらのカスタム型を直接カラムとして読み書きできるようにする実装方針（`database/sql`の`Scanner`/`Valuer`インターフェースの実装、またはtime.Time/stringとの相互変換）が必要になるが、②にはこの変換方式の明記がない。本書では以下を推測として補足する。

- `GoalStatus`は`string`を基底型とする型のため、GORMは追加実装なしに文字列カラムとして扱える
- `DueDate`はtime.Timeを内部に保持する独自structのため、`Scanner`/`Valuer`の実装、またはGoalStore内でtime.Time⇔DueDateの変換を行う方針のいずれかが必要になる。具体的な採用方式は②に明記がなく、実装時の判断に委ねる

## 主要クエリの条件・ソート・ページネーション方針

- `GoalStore.FindByUserID`: `user_id`で絞り込み、`due_date`昇順でソートし、page・per_pageに基づくページネーション（②「9. Repository設計」の検索機能に対応）を行う
- `GoalStore.FindByIDAndUserID`: `id`と`user_id`の両方に一致する1件を取得する
- `TaskStore.FindByGoalID`: `goal_id`で絞り込む

## 既存Schemaへの変更

②「17. DB設計方針」のとおり、スキーマ変更は提案されていない。本書でも既存`goals`テーブルをそのまま利用する前提とする。

---

# 13. テストケース設計

②「18. テスト戦略」を、Active Record採用に合わせて読み替える（アーキテクチャ規約「4. 設計パターンごとの構造適用方針」の読み替えルールに従い、Domain Test→Model Test、UseCase Test→対象外、Repository Test→Store Test）。

## Model Test

|対象|テストケース|
|-|-|
|GoalStatus.IsValid|許容値（not_started / in_progress / completed）でtrueを返すこと|
|GoalStatus.IsValid|許容値以外の文字列でfalseを返すこと|
|DueDateの生成|正しい形式の入力から正常に生成できること|
|DueDateの生成|不正な形式の入力でエラーになること|
|DueDate.String|`YYYY/MM/DD`形式の文字列を返すこと|
|Goal.Validate|Titleが空の場合にエラーになること|
|Goal.Validate|DueDateが不正な場合にエラーになること|
|Goal.Validate|Statusが許容値外の場合にエラーになること|
|Goal.Validate|すべての項目が妥当な場合にエラーにならないこと|

## UseCase Test

対象外（Active Record採用のため、usecase層を設けない）。

## Store Test

|対象|テストケース|
|-|-|
|GoalStore.FindByUserID|指定user_idのGoalのみ取得できること|
|GoalStore.FindByUserID|due_date昇順でソートされること|
|GoalStore.FindByUserID|ページネーションが正しく機能すること（件数・ページ境界）|
|GoalStore.FindByIDAndUserID|所有者本人のGoalが取得できること|
|GoalStore.FindByIDAndUserID|他ユーザーのGoalを指定した場合にGoal不存在エラーが返ること|
|GoalStore.Create|Goalが正しく作成され、IDが採番されること|
|GoalStore.Update|既存Goalの内容が正しく更新されること|
|TaskStore.FindByGoalID|指定goal_idに紐づくタスク一覧が取得できること|
|TaskStore.FindByGoalID|紐づくタスクがない場合に空のスライスが返ること|

## Handler Test

|対象|テストケース|
|-|-|
|GoalHandler.ListGoals|正常系: 目標一覧とページ情報が200で返ること|
|GoalHandler.ShowGoal|正常系: 目標詳細とタスク一覧が200で返ること|
|GoalHandler.ShowGoal|異常系: 存在しない・他ユーザーのGoalを指定した場合に404が返ること|
|GoalHandler.CreateGoal|正常系: 目標が作成されIDを含むレスポンスが返ること|
|GoalHandler.CreateGoal|異常系: title/due_date未入力時に422が返ること|
|GoalHandler.UpdateGoal|正常系: 目標が更新されること|
|GoalHandler.UpdateGoal|異常系: 存在しない・他ユーザーのGoalを指定した場合に404が返ること|
|GoalHandler.UpdateGoal|異常系: title/due_date未入力時に422が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|一覧〜詳細〜作成〜更新の一連の流れ|エンドポイント経由で一覧・詳細・作成・更新が正常に動作すること（②「18. テスト戦略」Integration Test）|
|認可|未認証・studentロール以外のユーザーがアクセスした場合に拒否されること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|Bounded Context名`goal-management`に対応する`internal/`配下のディレクトリ名を`internal/goal`とした|アーキテクチャ規約「5. Bounded Context構成」の命名規則、および既存の`task-management`→`internal/task`の対応関係からの類推|推測|
|Active Record構造の雛形（`model.go`/`store.go`）を、Goal用・TaskRef用に複数ファイル（`goal.go`/`goal_status.go`/`due_date.go`/`errors.go`/`goal_store.go`/`task_ref.go`/`task_store.go`）に分割した|②が扱う対象（Goal本体・参照専用のTask）が2種類あり、責務ごとにファイルを分けた方が可読性が高いと判断したため。Active Record構造（レイヤー分離をしない、同一package内に置く）という設計方針自体は変更していない|実装レベルの整理（②の設計判断とは矛盾しない）|
|作成・更新リクエストにStatusフィールドを含めず、作成時にStatusを内部で初期値（not_started）に設定する|②「4. 設計パターン」に「statusの変更ロジックはこの機能のAPI内では直接定義されていない」とあることから、Statusをユーザー入力の対象外とする方が現行仕様に忠実と判断したため|推測|
|DueDateのリクエスト入力フォーマットをISO 8601形式（`YYYY-MM-DD`）と仮定し、表示用フォーマット（`YYYY/MM/DD`）と分離した|②はDueDateの表示用フォーマットのみを明記しており、入力フォーマットの記載がないため。①未提供のため参照不可|推測|
|GoalListResponseのページ情報フィールド（Page/PerPage/TotalCount）の構造|②は「目標一覧とページ情報」とのみ記載し、具体的なフィールド構成がないため。①未提供のため参照不可|推測|
|TaskRefResponse（詳細画面の紐づくタスク一覧）のフィールド構成（ID/Title/Status）|②は「紐づくタスク一覧（tasks）を含む構造を維持する」とのみ記載し、詳細フィールドの記載がないため。①未提供のため参照不可|推測|
|ErrorResponseの構造（`Errors []string`）|②「16. API互換方針」に「既存のerrors形式をそのまま踏襲するか、Goの実装に合わせて再構成する」とあり、いずれかが確定していないため、「Goの実装に合わせて再構成する」を仮に採用した|推測|
|作成成功時のHTTPステータスを201、更新成功時を200とした|②「16. API互換方針」に「作成・更新成功時のステータスは既存仕様に合わせて統一する」とあるのみで具体的な値の記載がなく、①未提供のため参照不可|推測|
|認証エラー・認可エラーのHTTPステータスをそれぞれ401・403とした|②「13. Authorization設計」に具体的なステータスコードの記載がないため、一般的なHTTP実践に基づく|推測|
|DueDateのGORM永続化方式（Scanner/Valuer実装か、Store内での相互変換か）を確定していない|②にはValue ObjectとGORMモデルの変換方式についての明記がなく、実装時の判断に委ねる必要があるため|推測（方式の選択肢を示すにとどめ、確定は実装時判断とした）|

上記以外の項目（Bounded Context・Aggregate・Entity/Model構成・Repository（Store）責務・UseCase（Handler処理）の内容・Transaction境界・Validation方針・Authorization方針・Error設計・API互換方針・DB方針・テスト戦略の大枠）は②の記載内容をそのまま実装レベルに落とし込んだものであり、追加の判断は行っていない。
