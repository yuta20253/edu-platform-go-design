# ダッシュボード機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒のホーム画面に表示する、期限が近い目標（Goal）を最大5件取得する参照専用機能である。生徒自身の目標を`due_date`昇順で取得し、上位5件に絞り込んで返却する。状態変更を伴わない読み取り専用の集計処理である。

## 採用設計パターンとその理由（②からの要約）

- 採用パターン: **Transaction Script**
- 理由（②「4. 設計パターン」より）:
  - 業務ルールが「期限が近い順で最大5件取得する」という単純な並び替え・件数制限のみであり、状態管理・状態遷移が存在しない
  - 本機能側では目標の状態を変更しない（読み取り専用）
  - 将来的に他の集計項目（タスク進捗率等）が追加される場合も、独立したTransaction Scriptとして追加すればよく、既存ロジックへの影響がない
  - 入力（current user）に対する出力（上位5件の目標）を検証するだけでテストが完結する

## 本書が対象とする実装範囲

- 対象Bounded Context: `dashboard`（生徒向けダッシュボード）
- 対象エンドポイント: `GET /api/v1/student/dashboard`
- Transaction Script構造（アーキテクチャ規約「4. 設計パターンごとの構造適用方針」）に基づく、application層の関数、infrastructure層のクエリ関数、presentation層のHandler/Response/Routingの実装単位までを具体化する
- goal-management（目標管理機能）自体の実装仕様（目標の作成・更新・状態遷移等）は対象外。本書はgoal-managementが管理するデータを読み取り専用で参照する側の実装のみを扱う
- ①Rails実装（Controller/Query Object/Serializer等の詳細実装）は本タスクでは未提供のため、実装仕様の根拠として参照していない（「①未提供のため参照不可」として扱う）

---

# 2. ディレクトリ構成

- 対象Bounded Context名: `dashboard`
- ②で採用した設計パターン: **Transaction Script**
- 採用パターンに対応する構造: アーキテクチャ規約「4. 設計パターンごとの構造適用方針」の Transaction Script構成をそのまま適用する（domain層・usecase層・Repository Interfaceは設けない）

## 作成するディレクトリ一覧

```
internal/dashboard/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    ├── response/
    └── （routes.go はpresentation直下）
```

## 作成するファイル一覧

```
internal/dashboard/application/list_recent_goals.go
internal/dashboard/infrastructure/goal_query.go
internal/dashboard/presentation/handler/dashboard_handler.go
internal/dashboard/presentation/response/dashboard_response.go
internal/dashboard/presentation/routes.go
```

**②からの補足**: `domain/`・`presentation/request/`は本機能では作成しない。リクエストパラメータが存在しない（②「12. Validation設計」）ため、Request DTOも設けない。ファイル名・package構成は②に明記がないため、コーディング規約「4. パッケージ設計」およびアーキテクチャ規約「9. 命名規約」（Transaction Scriptの関数は`動詞+対象`）に従って補った（推測ではなく規約適用による決定）。

---

# 3. Domain層設計

**対象外（Transaction Script採用のため、Domain層を設けない）**。

理由: ②「6. Entity設計」「7. Value Object設計」「8. Domain Service」「9. Repository設計」のいずれも、本機能では独自のEntity・Value Object・Domain Service・Repository Interfaceを持たないと判断されている。業務ルールの検証・処理ステップは「4. Application層設計」に記載する。

---

# 4. Application層設計

**実装単位の読み替え**: 本機能はTransaction Script採用のため、UseCase（struct）は設けない。②「10. UseCase設計」の`GetDashboard`（業務操作としての設計意図）は、`application/`直下に置く関数`ListRecentGoals`として実装する。

**②からの補足**: 関数名`ListRecentGoals`は、②本文の「GetDashboard」という業務操作名から、アーキテクチャ規約「9. 命名規約」の「Transaction Scriptの関数は`動詞+対象`の形式にする（例：`ListRecentGoals`）」に従って命名した。この命名例はアーキテクチャ規約およびGo実装仕様書自動生成プロンプトの出力フォーマット例そのものであり、②の業務内容（生徒の目標一覧を取得する）と矛盾しない。

## DTO（Command / Query）

|struct名|フィールドと型|Command / Query|
|-|-|-|
|`GoalSummary`|`ID uint`, `Title string`, `Description string`, `Status string`, `DueDate time.Time`|Query（出力）|

- `ListRecentGoals`関数への入力は`userID uint`（current userのID）のみであり、専用の入力DTO structは設けない
- `GoalSummary`は`application/`パッケージ内に定義し、infrastructure層のクエリ結果をこの型に変換して返す

**②からの補足**: `GoalSummary`のフィールド構成は、②「16. API互換方針」のResponse定義（id, title, description, status, due_date）から導出した。②本文にDTOのフィールド型までの明記はないため、実装のために補った。

## UseCase（Transaction Script関数として記載）

### `ListRecentGoals`

- 関数シグネチャ: `func ListRecentGoals(ctx context.Context, db *gorm.DB, userID uint) ([]GoalSummary, error)`
- 処理ステップ（呼び出し順序。ロジックそのものは書かない）:
  1. `infrastructure`層のクエリ関数（`FindRecentGoalsByUserID`）を、`userID`と上限件数（5件）を指定して呼び出す
  2. 取得したレコードを`GoalSummary`のスライスへ変換する
  3. infrastructure層からのエラーをApplication Errorとしてラップして返す（詳細は「11. Error実装方針」）
- 呼び出すinfrastructure関数: `infrastructure.FindRecentGoalsByUserID`
- トランザクション境界: 不要（読み取りのみ。②「11. Transaction設計」に基づく。詳細は「8. Transaction実装方針」）
- 発生しうるApplication Error: `ErrFetchRecentGoalsFailed`（infrastructure層のエラーを包んだ取得失敗エラー）

**②からの補足**: 上限件数「5」は②「1. 機能概要」「16. API互換方針」に明記された業務ルールであり、実装時は`application`パッケージ内の定数（例：`MaxDashboardGoals = 5`）として定義することを推測として補った。定数名・配置場所は②に記載がないための実装判断である。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

### `FindRecentGoalsByUserID`

- 関数シグネチャ: `func FindRecentGoalsByUserID(ctx context.Context, db *gorm.DB, userID uint, limit int) ([]GoalRecord, error)`
- 発行するクエリ内容:
  - 対象テーブル: `goals`
  - 絞り込み条件: `user_id = ?`（current userのID）
  - ソート順: `due_date`昇順
  - 件数制限: `limit`件（application層から5を渡す）
  - SQL文そのものは記載しない。GORMのメソッドチェーン（Where条件・Order・Limit相当）で組み立てる方針とする

### GORMモデル（読み取り専用）

- struct名: `GoalRecord`
- 対応テーブル: `goals`（既存Rails DBのテーブルをそのまま利用。②「17. DB設計方針」よりSchema変更なし）
- 保持フィールド: `ID`, `UserID`, `Title`, `Description`, `Status`, `DueDate`（表示に必要な属性のみ。②「6. Entity設計」の「表示に必要な属性（id, title, description, due_date, status）」に対応）

**②からの補足**: `GoalRecord`をdashboard機能側で独自に定義する方針は、②およびアーキテクチャ規約「6. Context間連携ルール」（「相手Contextの内部Entity・Value Object・Infrastructure実装に直接依存しない」）に基づく判断である。goal-management側が公開する参照専用の関数・モデルが存在する場合はそれを利用すべきだが、goal-management機能の②/③文書は本タスクで参照しておらず、公開インターフェースの詳細が不明であるため、dashboard機能のinfrastructure層に読み取り専用のGORMモデルを独自定義し、`goals`テーブルへ直接クエリする構成とした。これは推測であり、goal-management側の実装仕様書を確認したうえで、公開関数呼び出しへの置き換えを検討する余地がある。

## 外部連携実装

対象外。②に記載の通り、Mail・Cache・Queue等の外部連携は本機能に含まれない。

---

# 6. Presentation層設計

## Handler

- struct名: `DashboardHandler`
- 対応する呼び出し先: `application.ListRecentGoals`関数
- コンストラクタ: `func NewDashboardHandler(db *gorm.DB) *DashboardHandler`（`db`はHandlerが保持し、application関数呼び出し時に渡す）

### メソッド一覧

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`GetDashboard`|GET|`/api/v1/student/dashboard`|

### 処理順序

1. Middlewareが設定したcontextからcurrent user（生徒）を取得し、`userID`を抽出する
2. リクエストパラメータの入力バインド・Validationは行わない（②「12. Validation設計」の通りパラメータが存在しないため）
3. `application.ListRecentGoals(ctx, db, userID)`を呼び出す
4. 取得した`[]application.GoalSummary`を`DashboardResponse`へ変換する
5. エラー発生時は「11. Error実装方針」の対応表に従いHTTPレスポンスへ変換する
6. 正常時は200 OKで`DashboardResponse`をJSONとして返す

**②からの補足**: Transaction Script採用のためUseCase層を経由しない。本来UseCaseが担う「current userのuser_idを検索条件へ渡す」手順（②「13. Authorization設計」のUseCase節に相当）は、Handlerの処理順序の中で`application.ListRecentGoals`呼び出し時の引数として明記した。

## Request / Response DTO

- Request DTO: 対象外（②「12. Validation設計」の通り、リクエストパラメータが存在しないため作成しない）

### Response DTO

|struct名|フィールドと型|備考|
|-|-|-|
|`DashboardResponse`|`Goals []GoalResponse`|ルートレスポンス|
|`GoalResponse`|`ID uint`, `Title string`, `Description string`, `Status string`, `DueDate time.Time`|②「16. API互換方針」のResponse項目に対応|

バリデーションタグ: Response DTOであるため入力バリデーションタグは付与しない。JSONタグ（`json:"id"`等）はコーディング規約・Gorm規約に反しない範囲でsnake_caseを付与する（②からの補足: JSONフィールド名の具体的な表記は②に明記がないため、既存API互換方針に合わせsnake_caseとした）。

## Routing

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/student/dashboard`|`DashboardHandler.GetDashboard`|

`presentation/routes.go`に`RegisterRoutes(rg *gin.RouterGroup, h *DashboardHandler)`のようなルート登録関数を置く方針とする（②からの補足: ルーティング登録関数の具体的な形は②に記載がないため、Ginを前提とした一般的な構成として補った）。認証・ロールチェックのMiddlewareは、authenticationを扱う別Bounded Context側の実装であり本書の対象外とする。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|`/api/v1/student/dashboard`|`DashboardHandler.GetDashboard`|なし|`DashboardResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証（トークン不備等）|401|認証エラー（Middleware側で処理。本機能のHandler到達前に弾かれる）|
|studentロールでない|403|認可エラー（Middleware側で処理）|
|`ListRecentGoals`実行時にDB接続失敗等が発生|500|Application Error/Infrastructure Errorに基づく想定外エラー（②「16. API互換方針」の「想定外エラー時のみ既存形式に準じたレスポンスを返す」に対応）|

**②からの補足**: 401/403はauthenticationを扱う別Contextの責務であり、②「13. Authorization設計」のMiddleware節に記載された「役割がstudentであることを確認する」を実現する具体的なStatus Codeは②に明記がないため、一般的なHTTP認可エラーの慣例として補った。

---

# 8. Transaction実装方針

②「11. Transaction設計」の通り、Transaction開始位置・終了位置ともに「なし」である。

- Transaction開始箇所: 対象外（状態変更を伴わない読み取り専用処理のため）
- Transaction終了箇所: 対象外
- 複数関数にまたがる場合の扱い: 本機能は`ListRecentGoals`単体から`FindRecentGoalsByUserID`単体を呼び出すのみであり、複数のinfrastructure関数を組み合わせる処理は存在しない。よってTransaction境界の調整は不要である

---

# 9. Validation実装方針

## Presentation

- ②「12. Validation設計」の通り、リクエストパラメータが存在しないため型チェック・必須チェック・フォーマットチェックのいずれも対象外

## 業務ルール検証

Transaction Script採用時のため、`application/list_recent_goals.go`内のガード節で検証する内容を記載する。

- ②「12. Validation設計」の通り、業務ルール・状態チェック・整合性チェックは「特になし」
- 前提条件はcurrent userの識別のみであり、`userID`が有効な値であることはHandler側でcontextから取得できることをもって満たされる（②「Middleware」の認証処理に委ねる）

**②からの補足**: `userID`の未設定（context取得失敗）時の扱いは②に明記がない。Handler側で取得失敗時は500（Application Error）として扱う方針を推測として補った。

---

# 10. Authorization実装方針

## Middleware

- 認証済みユーザーを特定し、contextへ保持する
- 役割がstudentであることを確認する

（②「13. Authorization設計」のMiddleware節をそのまま踏襲。具体的な実装はauthenticationを扱う別Bounded Context側の責務であり、本機能側では利用するのみとする）

## Handler

- 業務権限判定は持たせない（②の通り）
- contextからcurrent userの`userID`を取得し、`application.ListRecentGoals`へ引き渡す

## application関数（Transaction Script採用時）

- `ListRecentGoals`は、渡された`userID`を`FindRecentGoalsByUserID`の検索条件として使うことで、自分自身の目標のみを取得できるようにする（②「13. Authorization設計」のUseCase節を、Transaction Script構造に読み替えたもの）

## 判断理由（②より）

本機能はcurrent userのスコープに閉じた単純な参照であるため、認証確認と検索条件スコープ化のみで十分であり、追加の認可レイヤーを設ける必要はない。

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

②「14. Error設計」の通り、本機能では業務ルール違反がほぼ発生しないためDomain Errorは原則使用しない。Application Errorのみを中心としたシンプルな構成とする。

- `application`パッケージ内に`var ErrFetchRecentGoalsFailed = errors.New("...")`のようなsentinel errorを定義し、infrastructure層のエラーを`fmt.Errorf("...: %w", err)`でラップする方針とする（コーディング規約「6. エラーハンドリング」の`%w`ラップ方針に準拠）

## Application Error → HTTPレスポンスへの変換方針

- Handler側で`ListRecentGoals`のエラーを判定し、500 Internal Server Errorとして既存API互換のエラーレスポンス形式で返す

## Infrastructure Errorのハンドリング方針

- DB接続失敗等のGORMエラーは、infrastructure層の`FindRecentGoalsByUserID`内で`%w`によりラップし、application層へ伝播させる

|Error種別|発生層|HTTP Status|
|-|-|-|
|Application Error（`ErrFetchRecentGoalsFailed`）|Application（`ListRecentGoals`）|500|
|Infrastructure Error（DB接続失敗等）|Infrastructure（`FindRecentGoalsByUserID`）|500（Application Errorにラップされた上で変換）|

**②からの補足**: sentinel errorの具体的な変数名・エラーメッセージ文言は②に記載がないため実装のために補った。Domain Errorに相当する層はTransaction Script構造のため設けない（アーキテクチャ規約「8. 横断的関心事の置き場所」のError変換方針に基づく）。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」の通り、既存Rails DBを継続利用し、Schema変更は行わない。

## 利用するGORMモデルとテーブルの対応

|GORMモデル|テーブル|備考|
|-|-|-|
|`GoalRecord`|`goals`|ダッシュボード表示に必要な属性（id, user_id, title, description, status, due_date）のみを保持する読み取り専用モデル|

## 主要クエリの条件・ソート・ページネーション方針

- 条件: `user_id = ?`（current userのID）
- ソート: `due_date`昇順
- 件数制限: 上位5件（`LIMIT`相当。ページネーションは行わない。②に追加のページング要件の記載はない）

SQL文そのものはここに記載しない。GORMのメソッドチェーンで組み立てる。

## 既存Schemaへの変更

- ②の通り変更なし。`goals`テーブルの構造変更提案は行わない

---

# 13. テストケース設計

②「18. テスト戦略」を、Transaction Script読み替えルール（「Domain Test」「Repository Test」は対象外、「UseCase Test」→「Application関数 Test」）に従って具体化する。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test（②の「UseCase Test」を読み替え）

|対象|テストケース|
|-|-|
|`ListRecentGoals`|指定した`userID`に紐づく目標のみが返り、他ユーザーの目標が含まれないこと|
|`ListRecentGoals`|返却される目標が`due_date`昇順に並んでいること|
|`ListRecentGoals`|対象目標が6件以上存在する場合でも、返却件数が5件に絞り込まれること|
|`ListRecentGoals`|対象目標が0件の場合、空スライスが返ること（エラーにならないこと）|
|`ListRecentGoals`|`FindRecentGoalsByUserID`がエラーを返した場合、`ErrFetchRecentGoalsFailed`にラップされて返ること|

## Repository Test

対象外（Transaction Script採用のため、Repository層を設けない）。②「18. テスト戦略」記載の「検索条件（user_id絞り込み・ソート順・件数制限）の正確性」の検証観点は、上記「Application関数 Test」に統合して検証する。

## Handler Test

|対象|テストケース|
|-|-|
|`DashboardHandler.GetDashboard`|認証済みユーザーに対して200 OKと目標一覧のJSONが返ること|
|`DashboardHandler.GetDashboard`|`application.ListRecentGoals`がエラーを返した場合、500が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/student/dashboard`|エンドポイント経由でダッシュボードの目標一覧（最大5件、due_date昇順）が正しく取得できること|
|`GET /api/v1/student/dashboard`|未認証・非studentロールでアクセスした場合に401/403が返ること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|application関数名を`ListRecentGoals`とした|②の業務操作名「GetDashboard」から、アーキテクチャ規約「9. 命名規約」（Transaction Scriptの関数は動詞+対象）に従って命名した。同規約・自動生成プロンプトの構成例と一致する命名である|規約適用による決定（推測ではない）|
|`GoalSummary`（application DTO）・`GoalRecord`（infrastructure GORMモデル）のフィールド構成|②「16. API互換方針」のResponse項目（id, title, description, status, due_date）および②「6. Entity設計」の記載から導出した|規約・②記載事項からの導出（推測ではない）|
|`GoalRecord`をdashboard機能側に独自定義し、`goals`テーブルへ直接クエリする構成とした|goal-management側が公開する参照専用インターフェースの詳細（②/③）を本タスクでは参照しておらず、アーキテクチャ規約「6. Context間連携ルール」を踏まえた暫定的な実装方針として補った|推測。goal-management側の実装仕様確認後に見直しの余地あり|
|上限件数5件を`application`パッケージ内の定数（例：`MaxDashboardGoals`）として定義する方針|②に定数化の指示はないが、業務ルール変更時の影響範囲を限定する（②「2. 設計方針」の保守性方針）ために補った|推測|
|Application Errorのsentinel error名（`ErrFetchRecentGoalsFailed`）・エラーメッセージ文言|②「14. Error設計」はApplication Errorの責務のみを定義しており、具体的な変数名・文言は記載がないため、コーディング規約「6. エラーハンドリング」に沿って補った|推測|
|Response DTOのJSONタグ表記（snake_case）|②はレスポンス項目名（id, title, description, status, due_date）のみを定義しており、JSONタグの具体的な表記規則は明記がないため、既存API互換方針に合わせて補った|推測|
|Handler構成（`DashboardHandler`が`*gorm.DB`を保持し、application関数へ渡す）・ルート登録関数の形|②にPresentation層の実装レベルの構成記載がないため、Ginを前提とした一般的な構成として補った|推測|
|401/403のHTTP Status Code対応|②「13. Authorization設計」のMiddleware節（認証確認・役割確認）に対応する具体的なStatus Codeの明記がないため、一般的なHTTP認可エラーの慣例として補った|推測|

---
