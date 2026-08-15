# 管理者ダッシュボード機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

管理者がシステム全体の概況（ユーザー種別ごとの人数・総問題数・直近のCSVインポート履歴）を把握できるようにする、読み取り専用の集計表示機能である。単一の表示操作（GET）のみを提供し、状態変更を伴わない。

## 採用設計パターンとその理由（②からの要約）

- 採用パターン: **Transaction Script**
- 理由（②「4. 設計パターン」より）:
  - 状態を持つEntityが存在せず、業務ルールは「複数Contextの集計を実行し、まとめて返す」だけの手続き的処理で表現できる
  - 集計項目が増えても、application層の関数内の手続きに処理を追加するだけで対応できる
  - 各集計処理を個別に検証すれば十分であり、Entityに複雑な振る舞いを持たせる必要がない
- 採用しなかったパターン（②「4. 設計パターン」より）: Active Record（作成・更新対象のEntityが存在しないため不適合）、Domain Model（状態やライフサイクルを持つEntityが存在しないため意義がない）、Event Sourcing（状態変化の記録・再構築要件がなく過剰設計）

## 本書が対象とする実装範囲

- 対象Bounded Context: `admin-dashboard`（②「3. Bounded Context」）
- 対象エンドポイント: `GET /api/v1/admin/dashboard`
- Transaction Script構造（アーキテクチャ規約「4. 設計パターンごとの構造適用方針」）に基づく、application層の関数、infrastructure層のクエリ関数、presentation層のHandler/Response/Routingの実装単位までを具体化する
- User Context（ユーザー種別ごとの人数）・Question Context（総問題数）・Import Context（直近インポート履歴）自体が管理するデータの作成・更新・状態遷移の実装仕様は対象外。本書はこれらのContextが保持するデータを読み取り専用で参照し、集約して返す側の実装のみを扱う
- ①Rails実装（Controller/Serializer等の詳細実装）は本タスクでは未提供のため、実装仕様の根拠として参照していない（「①未提供のため参照不可」として扱う）

---

# 2. ディレクトリ構成

- 対象Bounded Context名: `admin-dashboard`
- ②で採用した設計パターン: **Transaction Script**
- 採用パターンに対応する構造: アーキテクチャ規約「4. 設計パターンごとの構造適用方針」の Transaction Script構成をそのまま適用する（domain層・usecase層・Repository Interfaceは設けない）

## 作成するディレクトリ一覧

```
internal/admin_dashboard/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    ├── response/
    └── （routes.go はpresentation直下）
```

## 作成するファイル一覧

```
internal/admin_dashboard/application/get_dashboard_summary.go
internal/admin_dashboard/infrastructure/user_query.go
internal/admin_dashboard/infrastructure/question_query.go
internal/admin_dashboard/infrastructure/import_history_query.go
internal/admin_dashboard/presentation/handler/admin_dashboard_handler.go
internal/admin_dashboard/presentation/response/admin_dashboard_response.go
internal/admin_dashboard/presentation/routes.go
```

**②からの補足**: `internal/`配下のディレクトリ名は、②のContext名`admin-dashboard`をアーキテクチャ規約「9. 命名規約」（kebab-caseのContext名を短いスネークケースへ変換する）に従い`internal/admin_dashboard`とした。対応関係は②に明記がないため規約適用による判断である。`domain/`・`presentation/request/`は本機能では作成しない。リクエストパラメータが存在しない（②「12. Validation設計」）ため、Request DTOも設けない。infrastructure層を3ファイルに分割した構成は、②「9. Repository設計」（UserRepository / QuestionRepository / ImportHistoryRepository）および②「10. UseCase設計」（「呼び出す関数: ユーザー種別件数集計関数, 総問題数集計関数, 直近インポート履歴取得関数」）に直接対応しており、②に根拠がある構成である。

---

# 3. Domain層設計

**対象外（Transaction Script採用のため、Domain層を設けない）**。

理由: ②「6. Entity設計」「7. Value Object設計」「8. Domain Service」「9. Repository設計」のいずれも、本機能では独自のEntity・Value Object・Domain Service・Repository Interfaceを持たないと判断されている（②「6. Entity設計」の`DashboardSummary`は「業務的な意味を持つEntityというよりも出力データ構造に近い性質」と明記されている）。業務ルールの検証・処理ステップは「4. Application層設計」に記載する。

---

# 4. Application層設計

**実装単位の読み替え**: 本機能はTransaction Script採用のため、UseCase（struct）は設けない。②「10. UseCase設計」の`GetDashboardSummary`（業務操作としての設計意図）は、`application/`直下に置く関数`GetDashboardSummary`として実装する。

**②からの補足**: 関数名は②本文に記載された業務操作名`GetDashboardSummary`をそのまま採用した。アーキテクチャ規約「9. 命名規約」の「Transaction Scriptの関数は`動詞+対象`の形式にする」にも合致するため、名称変更は行っていない。

## DTO（Command / Query）

|struct名|フィールドと型|Command / Query|
|-|-|-|
|`DashboardSummary`|`StudentCount int`, `TeacherCount int`, `AdminCount int`, `TotalQuestions int`, `RecentImports []ImportHistorySummary`|Query（出力）|
|`ImportHistorySummary`|`ID uint`, `FileName string`, `Status string`, `SuccessCount int`, `ErrorCount int`, `TotalCount int`, `CreatedAt time.Time`|Query（出力）|

- `GetDashboardSummary`関数への入力は無し（current admin userの識別は認可のみに用いられ、集計条件には使われない。②「10. UseCase設計」の「入力: current admin user（パラメータなし）」「13. Authorization設計」の「特別なスコープ限定は行わない」に対応）
- `DashboardSummary` / `ImportHistorySummary`は`application/`パッケージ内に定義し、各infrastructure関数の戻り値をこれらの型へ変換して組み立てる

**②からの補足**: 上記2つのDTOのフィールド構成は、②「16. API互換方針」のResponse定義（`stats`のstudent_count/teacher_count/admin_count/total_questions、`recent_imports`のid/file_name/status/success_count/error_count/total_count/created_at）から導出した。②本文にDTOのフィールド型までの明記はないため、実装のために補った。

## UseCase（Transaction Script関数として記載）

### `GetDashboardSummary`

- 関数シグネチャ: `func GetDashboardSummary(ctx context.Context, db *gorm.DB) (DashboardSummary, error)`
- 処理ステップ（呼び出し順序。ロジックそのものは書かない）:
  1. `infrastructure`層の`CountUsersByRole`を呼び出し、ロール別ユーザー数を取得する
  2. `infrastructure`層の`CountTotalQuestions`を呼び出し、総問題数を取得する
  3. `infrastructure`層の`FindRecentImportHistories`を、上限件数（5件）を指定して呼び出し、直近インポート履歴を取得する
  4. 上記3つの結果を`DashboardSummary`へ組み立てる
  5. いずれかのinfrastructure呼び出しでエラーが発生した場合、Application Errorとしてラップして返す（詳細は「11. Error実装方針」）
- 呼び出すinfrastructure関数: `infrastructure.CountUsersByRole`, `infrastructure.CountTotalQuestions`, `infrastructure.FindRecentImportHistories`
- トランザクション境界: 不要（読み取りのみ。②「11. Transaction設計」に基づく。詳細は「8. Transaction実装方針」）
- 発生しうるApplication Error: `ErrCountUsersByRoleFailed`, `ErrCountTotalQuestionsFailed`, `ErrFetchRecentImportHistoriesFailed`（各infrastructure呼び出しの失敗を個別に表すエラー）

**②からの補足**: 直近インポート履歴の上限件数「5」は②「9. Repository設計」（ImportHistoryRepositoryの「直近5件のインポート履歴取得」）に明記された業務ルールであり、実装時は`application`パッケージ内の定数（例：`MaxRecentImports = 5`）として定義することを推測として補った。定数名・配置場所は②に記載がないための実装判断である。3つのApplication Errorをinfrastructure呼び出し単位で分割した設計は、②「9. Repository設計」が3つの責務（UserRepository / QuestionRepository / ImportHistoryRepository）に分けて記載していることに合わせた判断であり、②「14. Error設計」自体は個別のエラー名までは定めていないため推測を含む。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

### `CountUsersByRole`

- 関数シグネチャ: `func CountUsersByRole(ctx context.Context, db *gorm.DB) (UserRoleCounts, error)`
- 発行するクエリ内容:
  - 対象テーブル: `users`（ロール情報を保持する`user_roles`との関連を含む。②「17. DB設計方針」の「既存の users / user_roles ... テーブルで集計要件を満たしている」に対応）
  - 集計内容: ロール（student / teacher / admin）ごとの件数集計
  - SQL文そのものは記載しない。GORMのメソッドチェーン（Group・Countまたは集計用クエリ）で組み立てる方針とする
- 戻り値struct: `UserRoleCounts`（`StudentCount int`, `TeacherCount int`, `AdminCount int`）

**②からの補足**: `users`/`user_roles`の具体的な結合方法（1ユーザーが複数ロールを持てるか、ロールが1カラムか別テーブルか）は②に明記がない。②「17. DB設計方針」がテーブル名のみを列挙しているため、ロール別件数集計を実現する具体的な結合・グルーピング方法は実装時にスキーマを確認して判断する必要がある事項として推測にとどめる。

### `CountTotalQuestions`

- 関数シグネチャ: `func CountTotalQuestions(ctx context.Context, db *gorm.DB) (int, error)`
- 発行するクエリ内容:
  - 対象テーブル: `questions`
  - 集計内容: 全件数のカウント（絞り込み条件なし。②「13. Authorization設計」の「管理者はシステム全体を集計対象とする」に対応し、校・学年等によるスコープ限定は行わない）
  - SQL文そのものは記載しない

**②からの補足**: 総問題数の集計対象テーブルを`questions`と推測した。②「17. DB設計方針」は`users` / `user_roles` / `import_histories`のみを列挙しており、Question Context側のテーブル名（②「3. Bounded Context」で総問題数の集計依存先として言及されている）が明記されていない。アーキテクチャ規約「5. Bounded Context構成」の上位ドメイン`Learning`（問題・復習テスト等）を参照し、一般的な命名から`questions`と判断したが、これは推測であり、Learning Context側の②/③文書で正式なテーブル名を確認する必要がある。

### `FindRecentImportHistories`

- 関数シグネチャ: `func FindRecentImportHistories(ctx context.Context, db *gorm.DB, limit int) ([]ImportHistoryRecord, error)`
- 発行するクエリ内容:
  - 対象テーブル: `import_histories`
  - ソート順: 作成日時（`created_at`）降順（②「9. Repository設計」の「作成日時降順」に対応）
  - 件数制限: `limit`件（application層から5を渡す）
  - SQL文そのものは記載しない。GORMのメソッドチェーン（Order・Limit相当）で組み立てる方針とする

## GORMモデル（読み取り専用）

|struct名|対応テーブル|保持フィールド|
|-|-|-|
|`UserRoleCounts`|`users` / `user_roles`（集計結果を保持する非永続構造体）|`StudentCount int`, `TeacherCount int`, `AdminCount int`|
|`ImportHistoryRecord`|`import_histories`|`ID uint`, `FileName string`, `Status string`, `SuccessCount int`, `ErrorCount int`, `TotalCount int`, `CreatedAt time.Time`（②「16. API互換方針」のrecent_imports項目に対応する属性のみを保持する読み取り専用モデル）|

既存Rails DBのテーブルをそのまま利用し、Schema変更は行わない（②「17. DB設計方針」）。

**②からの補足**: `UserRoleCounts`は特定テーブルに1対1対応するGORMモデルではなく、集計クエリの戻り値を受けるための非永続構造体である。この位置づけは②に明記がないため実装のために補った。また、User/Question/Import各Contextが読み取り専用の公開参照関数・モデルを既に用意している場合はそれを利用すべきだが、それらのContext自体の③実装仕様書は本タスクで参照しておらず、公開インターフェースの詳細が不明であるため、admin_dashboard機能のinfrastructure層に読み取り専用のGORMモデル・クエリ関数を独自定義し、対象テーブルへ直接クエリする構成とした（アーキテクチャ規約「6. Context間連携ルール」の「相手Contextの内部Entity・Value Object・Infrastructure実装に直接依存しない」を踏まえつつ、参照手段が未確認のための暫定判断）。これは推測であり、各Context側の実装仕様書を確認したうえで、公開関数呼び出しへの置き換えを検討する余地がある。

## 外部連携実装

対象外。②に記載の通り、Mail・Cache・Queue等の外部連携は本機能に含まれない。

---

# 6. Presentation層設計

## Handler

- struct名: `AdminDashboardHandler`
- 対応する呼び出し先: `application.GetDashboardSummary`関数
- コンストラクタ: `func NewAdminDashboardHandler(db *gorm.DB) *AdminDashboardHandler`（`db`はHandlerが保持し、application関数呼び出し時に渡す）

### メソッド一覧

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`GetDashboard`|GET|`/api/v1/admin/dashboard`|

### 処理順序

1. Middlewareが認証済みユーザーを特定し、adminロールであることを確認済みであることを前提とする（②「13. Authorization設計」のMiddleware節）
2. リクエストパラメータの入力バインド・Validationは行わない（②「12. Validation設計」の通りパラメータが存在しないため）
3. `application.GetDashboardSummary(ctx, db)`を呼び出す
4. 取得した`application.DashboardSummary`を`DashboardResponse`へ変換する
5. エラー発生時は「11. Error実装方針」の対応表に従いHTTPレスポンスへ変換する
6. 正常時は200 OKで`DashboardResponse`をJSONとして返す

**②からの補足**: Transaction Script採用のためUseCase層を経由しない。②「13. Authorization設計」の「UseCase」節（「特別なスコープ限定は行わない」）に相当する手順は、Handlerが`application.GetDashboardSummary`を追加の絞り込み条件なしで呼び出す、という処理順序の中で表現した。

## Request / Response DTO

- Request DTO: 対象外（②「12. Validation設計」の通り、リクエストパラメータが存在しないため作成しない）

### Response DTO

|struct名|フィールドと型|備考|
|-|-|-|
|`DashboardResponse`|`Stats StatsResponse`, `RecentImports []RecentImportResponse`|ルートレスポンス|
|`StatsResponse`|`StudentCount int`, `TeacherCount int`, `AdminCount int`, `TotalQuestions int`|②「16. API互換方針」の`stats`に対応|
|`RecentImportResponse`|`ID uint`, `FileName string`, `Status string`, `SuccessCount int`, `ErrorCount int`, `TotalCount int`, `CreatedAt time.Time`|②「16. API互換方針」の`recent_imports`に対応|

バリデーションタグ: Response DTOであるため入力バリデーションタグは付与しない。JSONタグは②「16. API互換方針」に明記されたフィールド名（`stats`, `student_count`, `teacher_count`, `admin_count`, `total_questions`, `recent_imports`, `id`, `file_name`, `status`, `success_count`, `error_count`, `total_count`, `created_at`）をそのままsnake_caseで付与する。

## Routing

|Method|Path|Handler|
|-|-|-|
|GET|`/api/v1/admin/dashboard`|`AdminDashboardHandler.GetDashboard`|

`presentation/routes.go`に`RegisterRoutes(rg *gin.RouterGroup, h *AdminDashboardHandler)`のようなルート登録関数を置く方針とする（②からの補足: ルーティング登録関数の具体的な形は②に記載がないため、Ginを前提とした一般的な構成として補った）。認証・adminロールチェックのMiddlewareは、authenticationを扱う別Bounded Context側の実装であり本書の対象外とする。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|`/api/v1/admin/dashboard`|`AdminDashboardHandler.GetDashboard`|なし|`DashboardResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証（トークン不備等）|401|認証エラー（Middleware側で処理。本機能のHandler到達前に弾かれる）|
|adminロールでない|403|認可エラー（Middleware側で処理）|
|`GetDashboardSummary`実行時にDB接続失敗等が発生|500|Application Error/Infrastructure Errorに基づく想定外エラー（②「16. API互換方針」の「障害時は既存パターンに合わせる」に対応）|

**②からの補足**: 401/403はauthenticationを扱う別Contextの責務であり、②「13. Authorization設計」のMiddleware節に記載された「adminロールであることを確認する」を実現する具体的なStatus Codeは②に明記がないため、一般的なHTTP認可エラーの慣例として補った。

---

# 8. Transaction実装方針

②「11. Transaction設計」の通り、Transaction開始位置・終了位置ともに「使用しない」「該当なし」である。

- Transaction開始箇所: 対象外（状態変更を伴わない読み取り専用処理のため）
- Transaction終了箇所: 対象外
- 複数関数にまたがる場合の扱い: `GetDashboardSummary`は`CountUsersByRole` / `CountTotalQuestions` / `FindRecentImportHistories`の3つのinfrastructure関数を呼び出すが、いずれも独立した参照系クエリであり、更新を伴わないため整合性確保のためのトランザクション調整は不要である。3つの呼び出し結果が同一時点のスナップショットであることまでは保証しない（②に厳密な整合性要件の記載がないため、読み取り専用集計として許容する）

---

# 9. Validation実装方針

## Presentation

- ②「12. Validation設計」の通り、リクエストパラメータが存在しないため型チェック・必須チェック・フォーマットチェックのいずれも対象外

## 業務ルール検証

Transaction Script採用時のため、`application/get_dashboard_summary.go`内のガード節で検証する内容を記載する。

- ②「12. Validation設計」の通り、業務ルール（Domain）は「該当なし」
- 本機能では入力検証の対象がなく、認可（adminロール確認）のみが主な制御対象となる（②「12. Validation設計」の「責務分離」に対応）

---

# 10. Authorization実装方針

## Middleware

- 認証済みユーザーを特定し、contextへ保持する
- adminロールであることを確認する

（②「13. Authorization設計」のMiddleware節をそのまま踏襲。具体的な実装はauthenticationを扱う別Bounded Context側の責務であり、本機能側では利用するのみとする）

## Handler

- 具体的な業務権限判定は持たせない（②の通り）
- ルーティング層でAPIの入口を担当し、認証失敗時のHTTP応答を整える

## application関数（Transaction Script採用時）

- `GetDashboardSummary`は、②「13. Authorization設計」のUseCase節（「特別なスコープ限定は行わない（管理者はシステム全体を集計対象とする）」）の通り、絞り込み条件を持たずシステム全体を対象に集計する

## 判断理由（②より）

管理者はシステム全体を対象とするため、生徒・教師向け機能のような所有者スコープの絞り込みが不要であり、ロール確認のみで十分である（②「13. Authorization設計」の判断理由をそのまま踏襲）。

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

②「14. Error設計」の通り、本機能は読み取り専用であるためDomain Errorは実質的に発生しない。Application Errorのみを中心としたシンプルな構成とする。

- `application`パッケージ内に、infrastructure呼び出し単位で以下のsentinel errorを定義し、infrastructure層のエラーを`fmt.Errorf("...: %w", err)`でラップする方針とする（コーディング規約「6. エラーハンドリング」の`%w`ラップ方針に準拠）
  - `ErrCountUsersByRoleFailed`
  - `ErrCountTotalQuestionsFailed`
  - `ErrFetchRecentImportHistoriesFailed`

## Application Error → HTTPレスポンスへの変換方針

- Handler側で`GetDashboardSummary`のエラーを判定し、500 Internal Server Errorとして既存API互換のエラーレスポンス形式で返す

## Infrastructure Errorのハンドリング方針

- DB接続失敗等のGORMエラーは、infrastructure層の各関数（`CountUsersByRole` / `CountTotalQuestions` / `FindRecentImportHistories`）内で`%w`によりラップし、application層へ伝播させる

|Error種別|発生層|HTTP Status|
|-|-|-|
|Application Error（`ErrCountUsersByRoleFailed`等）|Application（`GetDashboardSummary`）|500|
|Infrastructure Error（DB接続失敗等）|Infrastructure（各クエリ関数）|500（Application Errorにラップされた上で変換）|

**②からの補足**: sentinel errorの具体的な変数名・エラーメッセージ文言、および3つに分割する粒度は②に記載がないため実装のために補った。Domain Errorに相当する層はTransaction Script構造のため設けない（アーキテクチャ規約「8. 横断的関心事の置き場所」のError変換方針に基づく）。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」の通り、既存Rails DBを継続利用し、Schema変更は行わない。

## 利用するGORMモデルとテーブルの対応

|GORMモデル/構造体|テーブル|備考|
|-|-|-|
|`UserRoleCounts`|`users` / `user_roles`|ロール別件数の集計結果を保持する非永続構造体|
|`ImportHistoryRecord`|`import_histories`|ダッシュボード表示に必要な属性（id, file_name, status, success_count, error_count, total_count, created_at）のみを保持する読み取り専用モデル|
|（総問題数集計）|`questions`（②に明記なく推測。詳細は「5. Infrastructure層設計」の補足参照）|専用モデルは設けず、件数のみを返す|

## 主要クエリの条件・ソート・ページネーション方針

- ロール別ユーザー数集計: 絞り込み条件なし（全ユーザー対象）、ロールごとにグルーピングして件数を算出
- 総問題数集計: 絞り込み条件なし（全問題対象）、全件数のカウント
- 直近インポート履歴: 絞り込み条件なし、`created_at`降順、上位5件（`LIMIT`相当。ページネーションは行わない。②に追加のページング要件の記載はない）

SQL文そのものはここに記載しない。GORMのメソッドチェーンで組み立てる。

## 既存Schemaへの変更

- ②の通り変更なし。`users` / `user_roles` / `import_histories` / `questions`（推測）のいずれについても構造変更提案は行わない

---

# 13. テストケース設計

②「18. テスト戦略」を、Transaction Script読み替えルール（「Domain Test」「Repository Test」は対象外、「UseCase Test」→「Application関数 Test」）に従って具体化する。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test（②の「UseCase Test」を読み替え）

|対象|テストケース|
|-|-|
|`GetDashboardSummary`|`CountUsersByRole`の結果（student/teacher/admin件数）が`DashboardSummary`の各カウントフィールドへ正しく組み立てられること|
|`GetDashboardSummary`|`CountTotalQuestions`の結果が`DashboardSummary.TotalQuestions`へ正しく設定されること|
|`GetDashboardSummary`|`FindRecentImportHistories`の結果が`DashboardSummary.RecentImports`へ正しく組み立てられること|
|`GetDashboardSummary`|`FindRecentImportHistories`の呼び出し時に上限件数として5が渡されること|
|`GetDashboardSummary`|いずれかのinfrastructure関数がエラーを返した場合、対応するApplication Errorにラップされて返ること|
|`CountUsersByRole`|ロール（student/teacher/admin）ごとの件数集計が正確であること|
|`CountTotalQuestions`|総問題数の集計が正確であること|
|`FindRecentImportHistories`|直近5件が`created_at`降順で取得されること|
|`FindRecentImportHistories`|インポート履歴が6件以上存在する場合でも返却件数が5件に絞り込まれること|
|`FindRecentImportHistories`|インポート履歴が0件の場合、空スライスが返ること（エラーにならないこと）|

## Repository Test

対象外（Transaction Script採用のため、Repository層を設けない）。②「18. テスト戦略」記載の「ロール別集計、総問題数集計、直近インポート履歴取得の正確性」の検証観点は、上記「Application関数 Test」の`CountUsersByRole` / `CountTotalQuestions` / `FindRecentImportHistories`の行に統合して検証する。

## Handler Test

|対象|テストケース|
|-|-|
|`AdminDashboardHandler.GetDashboard`|認証済みadminユーザーに対して200 OKと`DashboardResponse`のJSONが返ること|
|`AdminDashboardHandler.GetDashboard`|`application.GetDashboardSummary`がエラーを返した場合、500が返ること|
|`AdminDashboardHandler.GetDashboard`|レスポンスのJSONキーが②「16. API互換方針」のフィールド名（stats/student_count等、recent_imports/id等）と一致すること|

## Integration Test

|対象|テストケース|
|-|-|
|`GET /api/v1/admin/dashboard`|エンドポイント経由でstats（student_count/teacher_count/admin_count/total_questions）とrecent_imports（最大5件、created_at降順）が正しく取得できること|
|`GET /api/v1/admin/dashboard`|未認証・非adminロールでアクセスした場合に401/403が返ること|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に記載する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|`internal/`配下のディレクトリ名を`internal/admin_dashboard`とした|②のContext名`admin-dashboard`とディレクトリ名の対応関係が②に明記がないため、アーキテクチャ規約「9. 命名規約」の変換ルール（kebab-case→スネークケース）に基づき判断した|規約適用による決定（推測ではない）|
|infrastructure層を`user_query.go` / `question_query.go` / `import_history_query.go`の3ファイルに分割した|②「9. Repository設計」（UserRepository / QuestionRepository / ImportHistoryRepository）および②「10. UseCase設計」の「呼び出す関数」の記載に直接対応する構成である|②記載事項からの導出（推測ではない）|
|`DashboardSummary` / `ImportHistorySummary`（application DTO）、`UserRoleCounts` / `ImportHistoryRecord`（infrastructure構造体）のフィールド構成|②「16. API互換方針」のResponse項目（stats/recent_importsの各フィールド）から導出した|規約・②記載事項からの導出（推測ではない）|
|総問題数の集計対象テーブルを`questions`と推測した|②「17. DB設計方針」は`users` / `user_roles` / `import_histories`のみを列挙しており、②「3. Bounded Context」で言及されているQuestion Context側のテーブル名が明記されていないため、アーキテクチャ規約「5. Bounded Context構成」の上位ドメイン`Learning`を参照し一般的な命名として補った|推測。Learning Context側の②/③文書で正式なテーブル名の確認が必要|
|`users` / `user_roles`の具体的な結合方法（ロールが別テーブルか1カラムか、1ユーザーが複数ロールを持てるか）|②「17. DB設計方針」はテーブル名のみを列挙しており、集計を実現する具体的な結合・グルーピング方法までは明記がないため、実装時にスキーマを確認する前提の暫定方針として補った|推測|
|User/Question/Import各Contextの公開参照関数を経由せず、admin_dashboard機能のinfrastructure層で対象テーブルへ直接クエリする構成とした|各Context自体の③実装仕様書を本タスクでは参照しておらず、公開インターフェースの詳細が不明であるため、アーキテクチャ規約「6. Context間連携ルール」を踏まえた暫定的な実装方針として補った|推測。各Context側の実装仕様確認後に見直しの余地あり|
|直近インポート履歴の上限件数5件を`application`パッケージ内の定数（例：`MaxRecentImports`）として定義する方針|②「9. Repository設計」に明記された上限件数「5」を、実装上は定数化することで業務ルール変更時の影響範囲を限定する（②「2. 設計方針」の保守性方針）ために補った|推測（定数化そのものは②の数値に基づく実装判断）|
|Application Errorのsentinel error名（`ErrCountUsersByRoleFailed`等3種）・エラーメッセージ文言|②「14. Error設計」はApplication Errorの責務のみを定義しており、具体的な変数名・文言、3分割の粒度は記載がないため、コーディング規約「6. エラーハンドリング」に沿って補った|推測|
|Handler構成（`AdminDashboardHandler`が`*gorm.DB`を保持し、application関数へ渡す）・ルート登録関数の形|②にPresentation層の実装レベルの構成記載がないため、Ginを前提とした一般的な構成として補った|推測|
|401/403のHTTP Status Code対応|②「13. Authorization設計」のMiddleware節（認証確認・adminロール確認）に対応する具体的なStatus Codeの明記がないため、一般的なHTTP認可エラーの慣例として補った|推測|
