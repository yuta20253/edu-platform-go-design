# 分析機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒の学習進捗・成績を分析し、`analytics[type]`パラメータで指定された分析タイプ（`task_completion`・`understanding_score`・`grade_average`・`course_rank`・`unit_rank`）に応じた集計結果を返却する、読み取り専用の機能である（②「1. 機能概要」）。

## 採用設計パターンとその理由（②からの要約）

②「4. 設計パターン」により、本機能は **Transaction Script** を採用する。

- 各分析タイプは「既存データの読み取り→集計→結果返却」という手続き型の処理であり、状態を持つEntityが振る舞う必要がない
- 分析対象データ（タスク・成績・理解度）はいずれも他Context（task-management／Grade・Assessment Context／curriculum）が所有するデータであり、本Contextは参照・集計にのみ責務を持つ
- 分析タイプごとに独立した処理単位とすることで、新しい分析タイプの追加が既存の分析タイプに影響しない構造とする（②「2. 設計方針」）

Active Record・Domain Model・Event Sourcingは、②「4. 設計パターン 採用しなかったパターン」「20. 採用しなかった設計」のとおり不採用である。

なお②は、Transaction Script採用を前提としつつも、`AnalyticsType`・`CompletionRate`・`UnderstandingScore`という3つのValue Objectと、`RankCalculationService`という1つのDomain Serviceを明示的な設計判断として採用している（②「7. Value Object設計」「8. Domain Service」）。本書はこれらの概念を変更せず維持するが、規約`アーキテクチャ規約.md`「11. ①②③文書との関係」の「②文書内のRepository/UseCase等の記載は概念的な設計意図であり、実装構造への変換（層をどこまで分離するか）は③で行う」という方針に基づき、Transaction Script構造（domain/ディレクトリを作らない）の範囲内でこれらを実装する位置づけを本書で確定する。詳細は「3. Domain層設計」「4. Application層設計」に記載する。

## 本書が対象とする実装範囲

本書は、②で確定した設計（Bounded Context・設計パターン・Aggregate・Value Object・Domain Service・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略）を変更せず、Goでの具体的なコード構成（package構成・型定義・関数シグネチャ・クエリ内容）に落とし込むことを目的とする。

規約`アーキテクチャ規約.md`「4. 設計パターンごとの構造適用方針」の「Transaction Script」節に従い、`{context}/application`・`infrastructure`・`presentation`の構造を適用し、domain層・usecase層（struct）・Repository Interfaceは設けない。

①Rails実装詳細は本タスクでは提供されていないため、①の実装コードそのものを根拠とする記載は行わない（①未提供のため参照不可）。②に明記された「Rails現行仕様の要約」の範囲でのみ言及する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- Context名（②）: `learning-analytics`
- ディレクトリ名: `internal/analytics`

  > **②からの補足**: ②にはディレクトリ名の明記がない。アーキテクチャ規約「5. Bounded Context構成 命名規則」の「Context名とディレクトリ名が一致しない場合は②文書内で対応関係を明記する」「英単語1語または短いスネークケースにする」に従い、`learning-analytics`を1語に短縮した（認証機能の②③文書における`authentication`→`internal/auth`の前例に倣う判断。推測）。

## ②で採用した設計パターン

Transaction Script

## 作成するディレクトリ一覧

規約「4. 設計パターンごとの構造適用方針」の「Transaction Script」構造に従う。domain層・usecase層（struct）・Repository Interfaceは作成しない。

```
internal/analytics/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

## 作成するファイル一覧

```
internal/analytics/application/analytics_dto.go
internal/analytics/application/completion_rate.go
internal/analytics/application/understanding_score.go
internal/analytics/application/get_task_completion_analytics.go
internal/analytics/application/get_understanding_score_analytics.go
internal/analytics/application/get_grade_average_analytics.go
internal/analytics/application/get_course_rank_analytics.go
internal/analytics/application/get_unit_rank_analytics.go
internal/analytics/application/rank_calculation.go
internal/analytics/application/errors.go

internal/analytics/infrastructure/task_completion_query.go
internal/analytics/infrastructure/understanding_score_query.go
internal/analytics/infrastructure/grade_query.go
internal/analytics/infrastructure/rank_query.go

internal/analytics/presentation/request/analytics_request.go
internal/analytics/presentation/request/analytics_type.go
internal/analytics/presentation/response/analytics_response.go
internal/analytics/presentation/handler/analytics_handler.go
internal/analytics/presentation/routes.go
```

> **②からの補足**: ファイル分割の粒度（1関数1ファイルにするか等）は②に明記がない。コーディング規約の「パッケージ名はディレクトリ名と一致させる」方針の範囲内で、分析タイプ単位・関心事単位に分割した（推測）。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層（`domain/`ディレクトリ）を設けない。規約「4. 設計パターンごとの構造適用方針」の「Transaction Script」節に従う）。

ただし②「7. Value Object設計」「8. Domain Service」は、Transaction Script採用を前提としつつも`AnalyticsType`・`CompletionRate`・`UnderstandingScore`・`RankCalculationService`という4つの概念を明示的な設計判断として採用しており、本書はこの判断を変更しない。各概念の実装位置は以下のとおりとし、具体的な型定義・メソッド一覧は「4. Application層設計」に記載する。

## Value Object（実装位置の整理）

|②での概念|実装位置|判断根拠|
|-|-|-|
|`AnalyticsType`|`presentation/request/analytics_type.go`|②「12. Validation設計」がAnalyticsTypeの許容値チェックをPresentation層の責務として明記しているため（②内の記載同士を整合させた判断であり、新たな推測ではない）|
|`CompletionRate`|`application/completion_rate.go`|タスク完了率の集計結果に対する型付け・範囲検証であり、UseCase（Transaction Script関数）の出力値の意味付けにあたるため、application層に置く（配置場所自体は②に明記がなく推測）|
|`UnderstandingScore`|`application/understanding_score.go`|同上（理解度スコアの集計結果に対する型付け・範囲検証）|

`AnalyticsType`・`CompletionRate`・`UnderstandingScore`はいずれも、規約「4. 設計パターンごとの構造適用方針」の「Transaction Script」節が定める「Value Object等の型は作らず、関数内のガード節で行う」という原則から見ると例外的な位置づけになる。②が明示的にこれらをValue Objectとして採用と判断根拠を記載しているため（②「7. Value Object設計」）、本書ではこの判断を維持しつつ、構造としては最小限（コンストラクタ関数＋型を持つだけの単純な型）に留め、domain層やInterfaceによる抽象化は追加しない。

## Domain Service（実装位置の整理）

|②での概念|実装位置|判断根拠|
|-|-|-|
|`RankCalculationService`|`application/rank_calculation.go`内の非公開関数|禁止事項「Transaction Script/Active Record採用機能に対してDomain Model相当のusecase層・Repository Interface・依存性逆転を不要に追加する」を避けるため、struct＋interfaceによるDIは行わず、`GetCourseRankAnalytics`・`GetUnitRankAnalytics`の両関数から呼び出す非公開の共有関数として実装する（規約に基づく判断。推測ではない）|

## Repository Interface

対象外（Transaction Script採用のため、Repository Interfaceをdomain層に定義しない）。②「9. Repository設計」に記載された`TaskCompletionRepository`・`UnderstandingScoreRepository`・`GradeRepository`・`RankRepository`の設計意図は、「5. Infrastructure層設計」のinfrastructure関数として実装する。

## Domain Event

対象外。②「15. Domain Event」により不要と判断されている。

## Domain Error

対象外（Transaction Script採用のためDomain Errorに相当する層を設けない。規約「8. 横断的関心事の置き場所」の「Transaction Script/Active Record採用時はDomain Errorに相当する層がないため、関数・struct側で発生したエラーをApplication Error相当として扱う」に従う）。②「14. Error設計」のDomain Errorに相当する内容（順位算出時の範囲指定不整合等の業務ルール違反）は、`application/errors.go`のセンチネルエラーとして「11. Error実装方針」に記載する。

---

# 4. Application層設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、UseCase層（struct）を設けない。1つの業務操作を1つの関数として`application/`直下に置く（規約「4. 設計パターンごとの構造適用方針」）。

## DTO（Query）

本機能は読み取り専用のためCommand DTOは存在せず、すべてQueryに属する。

### 入力（Query）

|struct名|フィールドと型|属性|
|-|-|-|
|`TaskCompletionQuery`|`UserID uint64` / `CourseID *uint64` / `UnitID *uint64`|Query|
|`UnderstandingScoreQuery`|`UserID uint64` / `CourseID *uint64` / `UnitID *uint64`|Query|
|`GradeAverageQuery`|`UserID uint64` / `CourseID *uint64` / `UnitID *uint64`|Query|
|`CourseRankQuery`|`UserID uint64` / `CourseID *uint64`|Query|
|`UnitRankQuery`|`UserID uint64` / `UnitID *uint64`|Query|

> **②からの補足**: course_rank/unit_rankの`CourseID`/`UnitID`をポインタ型（任意）としたのは、②「12. Validation設計」がその必須チェックを「Domain（業務ルール）」として位置づけているためであり（Presentation層の必須チェック対象としていない）、Transaction Script採用時は「9. Validation実装方針」のとおりapplication関数内のガード節で検証する（②内の記載を整合させた判断）。

### 出力（Query結果）

|struct名|フィールドと型|備考|
|-|-|-|
|`TaskCompletionResult`|`Rate CompletionRate` / `CompletedCount int` / `TotalCount int`||
|`UnderstandingScoreResult`|`Score UnderstandingScore`||
|`GradeAverageResult`|`Average float64`|②「7. Value Object設計」はCompletionRate/UnderstandingScoreのみをVO対象としており、成績平均は対象に含まれていないため、plain `float64`のまま扱う（②に根拠のない業務ルールを追加しないための判断）|
|`RankResult`|`Rank int` / `TotalCount int`|course_rank/unit_rank共通（②「10. UseCase設計」が両者で順位算出ロジックを共有する方針のため。struct共通化自体は推測）|

## Application関数

### GetTaskCompletionAnalytics（`application/get_task_completion_analytics.go`）

- 関数シグネチャ: `func GetTaskCompletionAnalytics(ctx context.Context, q TaskCompletionQuery) (TaskCompletionResult, error)`
- 処理ステップ:
  1. infrastructure層の`FetchTaskCompletionSummary`を呼び出し、完了数・総数を取得する
  2. 完了率を算出し、`NewCompletionRate`で範囲検証したうえで`TaskCompletionResult`を組み立てる
  3. infrastructure層のエラーをラップして返す
- 呼び出すinfrastructure関数: `FetchTaskCompletionSummary`
- 発生しうるApplication Error: infrastructure由来のエラー（DB接続失敗等）

  > **②からの補足**: 総数（対象タスク数）が0件の場合の完了率の扱い（0%として扱うかエラーとするか）は②に明記がない。本書では0%として扱う実装判断とする（推測）。

### GetUnderstandingScoreAnalytics（`application/get_understanding_score_analytics.go`）

- 関数シグネチャ: `func GetUnderstandingScoreAnalytics(ctx context.Context, q UnderstandingScoreQuery) (UnderstandingScoreResult, error)`
- 処理ステップ:
  1. infrastructure層の`FetchUnderstandingScoreSummary`を呼び出し、理解度スコアの集計値を取得する
  2. `NewUnderstandingScore`で範囲検証したうえで`UnderstandingScoreResult`を組み立てる
  3. infrastructure層のエラーをラップして返す
- 呼び出すinfrastructure関数: `FetchUnderstandingScoreSummary`
- 発生しうるApplication Error: infrastructure由来のエラー

### GetGradeAverageAnalytics（`application/get_grade_average_analytics.go`）

- 関数シグネチャ: `func GetGradeAverageAnalytics(ctx context.Context, q GradeAverageQuery) (GradeAverageResult, error)`
- 処理ステップ:
  1. infrastructure層の`FetchGradeSummary`を呼び出し、成績平均を取得する
  2. `GradeAverageResult`を組み立てる
  3. infrastructure層のエラーをラップして返す
- 呼び出すinfrastructure関数: `FetchGradeSummary`
- 発生しうるApplication Error: infrastructure由来のエラー

  > **②からの補足**: 対象成績データが0件の場合の扱いが②に明記がない。本書では平均0として扱う実装判断とする（推測）。

### GetCourseRankAnalytics（`application/get_course_rank_analytics.go`）

- 関数シグネチャ: `func GetCourseRankAnalytics(ctx context.Context, q CourseRankQuery) (RankResult, error)`
- 処理ステップ:
  1. ガード節: `q.CourseID`が`nil`の場合、`ErrCourseIDRequired`を返す（②「12. Validation設計」Domainの「course_rankの場合にcourse_idが指定されているかの整合性チェック」に対応）
  2. infrastructure層の`CourseExists`を呼び出し、course_idの実在確認を行う。存在しない場合`ErrCourseNotFound`を返す（②「12. Validation設計」Domainの整合性チェックに対応）
  3. infrastructure層の`FetchCourseRankCohort`を呼び出し、対象コース内の比較対象データ（他生徒を含む集計値）を取得する
  4. 「順位算出（共有ロジック）」節の`calculateRank`にユーザーIDと比較対象データを渡し、順位・母数を算出する
  5. `RankResult`を組み立てて返す
- 呼び出すinfrastructure関数: `CourseExists`, `FetchCourseRankCohort`
- 発生しうるApplication Error: `ErrCourseIDRequired`, `ErrCourseNotFound`, infrastructure由来のエラー

### GetUnitRankAnalytics（`application/get_unit_rank_analytics.go`）

- 関数シグネチャ: `func GetUnitRankAnalytics(ctx context.Context, q UnitRankQuery) (RankResult, error)`
- 処理ステップ: `GetCourseRankAnalytics`と同様の構造で、`UnitID`のガード節（`ErrUnitIDRequired`）・`UnitExists`による実在確認（`ErrUnitNotFound`）・`FetchUnitRankCohort`によるデータ取得・`calculateRank`による順位算出を行う
- 呼び出すinfrastructure関数: `UnitExists`, `FetchUnitRankCohort`
- 発生しうるApplication Error: `ErrUnitIDRequired`, `ErrUnitNotFound`, infrastructure由来のエラー

## 順位算出（共有ロジック、②のRankCalculationServiceに対応）

- 関数シグネチャ: `func calculateRank(entries []rankEntry, targetUserID uint64) (RankResult, error)`（`application/rank_calculation.go`内の非公開関数）
- `rankEntry`は非公開struct（`UserID uint64` / `Score float64`）とし、infrastructure層から受け取った比較対象データを保持する。他生徒の氏名・メールアドレス等の個人を特定できる情報は含めない（②「13. Authorization設計」Domainの「個人を特定できる情報を結果に含めない」というルールに対応）
- 処理ステップ（ロジックそのものは記載しない）:
  1. `entries`をスコア降順に並べ替える
  2. `targetUserID`に一致する要素の順位（1始まり）を求める
  3. 対象ユーザーが`entries`に含まれない場合はエラーを返す
  4. `RankResult{Rank, TotalCount: len(entries)}`を返す
- `GetCourseRankAnalytics`・`GetUnitRankAnalytics`の双方から呼び出され、順位算出ロジックの重複を避ける（②「8. Domain Service 判断根拠」に対応）

  > **②からの補足**: course_rank/unit_rankで比較対象とする指標（理解度スコアか成績平均か、あるいは両者を統合した指標か）が②に明記されていない（②「9. Repository設計」RankRepositoryは「成績/理解度データ」とのみ記載）。①未提供のため参照不可であり、本書でも指標を確定できない。infrastructure層の`FetchCourseRankCohort`/`FetchUnitRankCohort`が返す`float64`スコアの算出元は、curriculum/Grade・Assessment Context側の③文書確定後に確認する必要がある（推測不可・要確認事項として「14. ②からの補足事項」にも記載する）。

## トランザクション境界

②「11. Transaction設計」により、すべての関数は読み取り専用でありトランザクションは使用しない。詳細は「8. Transaction実装方針」を参照。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

**実装上の位置づけ**: ②「9. Repository設計」に記載された各Repositoryの設計意図（管理対象・責務・検索機能）を、`infrastructure/`直下の関数として直接実装する。

### task_completion_query.go

- 関数シグネチャ: `func FetchTaskCompletionSummary(ctx context.Context, userID uint64, courseID, unitID *uint64) (TaskCompletionSummary, error)`
- 発行するクエリ内容: task-management Contextが所有するタスクテーブルに対し、`user_id`で絞り込み、`course_id`/`unit_id`が指定されている場合はさらに絞り込んだうえで、完了タスク数・対象タスク総数を集計する（②「9. Repository設計」TaskCompletionRepositoryの「保持する検索機能: user_idによる絞り込み、完了・未完了タスク数の集計」に対応）
- 戻り値: `TaskCompletionSummary{CompletedCount int; TotalCount int}`（非公開struct、application層の`TaskCompletionResult`とは別に定義する内部集計値）

### understanding_score_query.go

- 関数シグネチャ: `func FetchUnderstandingScoreSummary(ctx context.Context, userID uint64, courseID, unitID *uint64) (float64, error)`
- 発行するクエリ内容: Grade/Assessment Contextが所有する理解度スコアデータに対し、`user_id`・`course_id`/`unit_id`で絞り込み、平均値を集計する（②「9. Repository設計」UnderstandingScoreRepositoryに対応）

  > **②からの補足**: 理解度スコアの算出元テーブル・カラムは②自身が「推測」と明記しており（②「3. Bounded Context 他Contextとの依存関係」）、①未提供のため本書でも確定できない（②の推測を維持）。

### grade_query.go

- 関数シグネチャ: `func FetchGradeSummary(ctx context.Context, userID uint64, courseID, unitID *uint64) (float64, error)`
- 発行するクエリ内容: Grade/Assessment Contextが所有する成績データに対し、`user_id`・`course_id`/`unit_id`（推測: ②「9. Repository設計」GradeRepositoryが「期間・科目等による絞り込み（推測）」としているため、詳細な絞り込み条件は本書でも確定しない）で絞り込み、平均値を集計する

### rank_query.go

- 関数シグネチャ:
  - `func CourseExists(ctx context.Context, courseID uint64) (bool, error)`
  - `func UnitExists(ctx context.Context, unitID uint64) (bool, error)`
  - `func FetchCourseRankCohort(ctx context.Context, courseID uint64) ([]rankEntry, error)`
  - `func FetchUnitRankCohort(ctx context.Context, unitID uint64) ([]rankEntry, error)`
- 発行するクエリ内容:
  - `CourseExists`/`UnitExists`: curriculum Contextが所有するコース／単元テーブルに対する存在確認（②「12. Validation設計」Domainの整合性チェックに対応）
  - `FetchCourseRankCohort`/`FetchUnitRankCohort`: 指定コース／単元に属する生徒群の比較対象スコアを、`user_id`とスコアの組として取得する（②「9. Repository設計」RankRepositoryの「対象生徒とその他生徒群の比較対象データ取得」に対応、および「保持しない責務: 他生徒の個人情報の外部公開」を踏まえ、氏名等は取得しない）

  > **②からの補足**: `CourseExists`/`UnitExists`はcurriculum Context、比較対象データの取得はGrade/Assessment Contextの参照手段を呼び出す想定である。規約「6. Context間連携ルール」の「相手Contextが公開する参照手段を呼び出す」方針に従うが、各Context側の公開関数の具体的なシグネチャは②未記載（かつcurriculum・Grade/Assessment Context側の③文書が本タスクでは未確認）のため、本書では呼び出し方針のみを記載し、確定的な関数シグネチャの整合は各Context側の③文書確定後に確認する必要がある（推測）。

## 外部連携実装

対象外。②に本機能でのMail・Cache・Queue利用の記載はない。

---

# 6. Presentation層設計

## Handler

### AnalyticsHandler（`presentation/handler/analytics_handler.go`）

- struct名: `AnalyticsHandler`
- 対応する呼び出し先: `application/`の各関数（`GetTaskCompletionAnalytics`等）
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`GetAnalytics`|GET|`/api/v1/student/analytics`|

- 処理順序（Transaction Script採用のため、本来UseCaseが担う手順も含めて記載する）:
  1. Middlewareが設定した`current_user`（`user_id`）を`*gin.Context`から取得する
  2. クエリパラメータ`analytics[type]`・`course_id`・`unit_id`を`AnalyticsRequest`へバインドする
  3. `AnalyticsRequest`のバリデーション（必須チェック・型チェック）を実行する。失敗時は422相当のエラーレスポンスを返す

     > **②からの補足**: 既存エンドポイントのクエリパラメータ名`analytics[type]`はブラケット記法であり、Ginの`ShouldBindQuery`によるタグベースの自動バインドが期待どおり動作するかは規約・②いずれにも明記がない。本書では`c.Query("analytics[type]")`による明示的な取得を実装方針として補足する（推測、フレームワーク挙動に基づく実装判断）。
  4. `AnalyticsType`の許容値チェック（`ParseAnalyticsType`）を実行する。不正な値の場合は400を返す（②「16. API互換方針」Status Codeの「400: 分析タイプが不正」に対応）
  5. `AnalyticsType`の値に応じて、対応する`application/`関数を呼び出す（`user_id`・`course_id`・`unit_id`をQuery DTOに詰めて渡す）
  6. 呼び出し先が返すエラーを「11. Error実装方針」の変換表に従いHTTP Statusへ変換する
  7. 成功時は分析タイプに応じたResponse DTOへ変換し、200で返す

## Request / Response DTO

### Request

|struct名|フィールドと型|バリデーション|
|-|-|-|
|`AnalyticsRequest`|`Type string` / `CourseID *uint64` / `UnitID *uint64`|`Type`: 必須（`binding:"required"`）。`CourseID`/`UnitID`: 数値変換可能であることのみ（型チェック）、必須指定はしない|

- `AnalyticsType`型（`presentation/request/analytics_type.go`）:
  - 型定義: `type AnalyticsType string`
  - 定数: `AnalyticsTypeTaskCompletion` / `AnalyticsTypeUnderstandingScore` / `AnalyticsTypeGradeAverage` / `AnalyticsTypeCourseRank` / `AnalyticsTypeUnitRank`（値は②「1. 機能概要」記載の5種の文字列そのまま）
  - 公開関数: `func ParseAnalyticsType(s string) (AnalyticsType, error)`（許容値チェック。不正な値は`ErrInvalidAnalyticsType`を返す）

### Response

|struct名|フィールドと型|対応する分析タイプ|
|-|-|-|
|`TaskCompletionResponse`|`Type string` / `CompletionRate float64` / `CompletedCount int` / `TotalCount int`|`task_completion`|
|`UnderstandingScoreResponse`|`Type string` / `Score float64`|`understanding_score`|
|`GradeAverageResponse`|`Type string` / `Average float64`|`grade_average`|
|`RankResponse`|`Type string` / `Rank int` / `TotalCount int`|`course_rank` / `unit_rank`|

> **②からの補足**: ②「16. API互換方針」は「分析タイプごとの結果ペイロードは、既存仕様に近い構造を維持する」とのみ記載しており、フィールド名・型までは規定していない。①未提供のため、既存Railsレスポンスのフィールド名をそのまま踏襲する根拠がない。本書では②「6. Entity設計」「7. Value Object設計」に記載された概念（完了率・スコア・平均値・順位）をそのままフィールド化した（推測）。

## Routing

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/student/analytics|`AnalyticsHandler.GetAnalytics`|

`routes.go`にてstudentロール向けルートグループへ登録し、認証・ロールチェックMiddlewareを適用する（②「13. Authorization設計」Middlewareに対応）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/student/analytics|AnalyticsHandler.GetAnalytics|AnalyticsRequest|TaskCompletionResponse \| UnderstandingScoreResponse \| GradeAverageResponse \| RankResponse（`type`により分岐）|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|未認証（`current_user`未設定）|401|Middlewareレベルで拒否|
|studentロール以外|403|Middlewareレベルで拒否（②「13. Authorization設計」Middlewareの役割チェックに対応）|
|`analytics[type]`未指定|400|Request DTOバリデーションエラー|
|`analytics[type]`が許容値以外|400|`ErrInvalidAnalyticsType`（②「16. API互換方針」Status Codeの「400: 分析タイプが不正」に対応。Rails現行仕様の例外発生ではなく明示的な400として扱う）|
|`course_id`/`unit_id`が数値変換不可|400|Request DTOバリデーションエラー|
|`course_rank`指定時に`course_id`未指定|400|`ErrCourseIDRequired`|
|`unit_rank`指定時に`unit_id`未指定|400|`ErrUnitIDRequired`|
|指定`course_id`が存在しない|404|`ErrCourseNotFound`（②「16. API互換方針」Status Codeの「404: 指定course_id/unit_idが存在しない」に対応）|
|指定`unit_id`が存在しない|404|`ErrUnitNotFound`|
|infrastructure層の障害（DB接続失敗等）|500|未分類のInfrastructure Error|

---

# 8. Transaction実装方針

## Transaction開始箇所

なし。②「11. Transaction設計」により、すべてのapplication関数は読み取り専用でありトランザクションを開始しない。

## Transaction終了箇所（Commit / Rollback条件）

該当なし（状態変更を伴わないため）。

## 複数infrastructure関数にまたがる場合の扱い

`GetCourseRankAnalytics`・`GetUnitRankAnalytics`は`CourseExists`/`UnitExists`と`FetchCourseRankCohort`/`FetchUnitRankCohort`という複数のinfrastructure関数を順に呼び出すが、いずれも読み取りのみであり、整合性保証のためのトランザクションは不要である（②「11. Transaction設計 理由」に対応）。各呼び出しは独立したクエリとして順次実行する。

---

# 9. Validation実装方針

## Presentation

- 型チェック: `course_id`/`unit_id`が数値であることを`AnalyticsRequest`のバインド時に検証する
- 必須チェック: `analytics[type]`が必須項目であることを`binding:"required"`で検証する
- フォーマットチェック: `ParseAnalyticsType`により、`analytics[type]`が5種の許容値のいずれかに合致するかを検証する（②「12. Validation設計」Presentationに対応）

## 業務ルール検証（Transaction Script採用のためapplication関数内のガード節）

- `GetCourseRankAnalytics`: `CourseID`が`nil`の場合にガード節で`ErrCourseIDRequired`を返す
- `GetUnitRankAnalytics`: `UnitID`が`nil`の場合にガード節で`ErrUnitIDRequired`を返す
- `GetCourseRankAnalytics`/`GetUnitRankAnalytics`: `CourseExists`/`UnitExists`による実在確認を行い、存在しない場合は`ErrCourseNotFound`/`ErrUnitNotFound`を返す

（②「12. Validation設計」Domainの「course_rank/unit_rankの場合にcourse_id/unit_idが指定されているかの整合性チェック」「指定されたcourse_id/unit_idが実在するかの確認」に対応。規約「9. Validation実装方針」のTransaction Script区分「application関数内のガード節で検証する内容」に従う）

---

# 10. Authorization実装方針

②「13. Authorization設計」をそのまま実装レベルに落とし込む。

## Middleware

- JWT等から認証済みユーザーを特定し、`user_id`を`*gin.Context`に保持する
- 役割がstudentであることを確認する（不一致時403）

## Handler

- リクエストパラメータの受け取りと、`AnalyticsType`に応じたapplication関数の呼び分けを行う
- 具体的な業務権限判定は持たせない

## Application関数（Transaction Script採用のためUseCase相当）

- すべての関数が`current_user`の`user_id`をすべての検索条件（Query DTO）に付与し、自分自身の学習データのみを対象とする
- `GetCourseRankAnalytics`/`GetUnitRankAnalytics`は、他生徒の個人情報を直接返却せず、対象生徒自身の順位情報（`Rank`・`TotalCount`）のみを`RankResult`として出力する

## 順位算出ロジック（②のDomainに対応）

- `calculateRank`は、他生徒の集計データを比較には利用するが、個人を特定できる情報（氏名・メールアドレス等）を結果に含めない（②「13. Authorization設計 Domain」に対応。Transaction Script採用のためdomain層は設けず、application層の非公開関数としてこのルールを実装する）

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

対象外（Transaction Script採用のためDomain Errorに相当する層はない）。②「14. Error設計」のDomain Errorに相当する業務ルール違反（順位算出時の範囲指定不整合）は、`application/errors.go`にセンチネルエラーとして定義し、Application Error相当として直接扱う（規約「8. 横断的関心事の置き場所」に従う）。

## Application Error → HTTPレスポンスへの変換方針

Handler層で`errors.Is`によりエラー種別を判定し、以下のStatus Codeへ変換する。変換ロジックは共通のエラーマッピング処理に集約し、Handler内で重複させない。

|Error種別|発生層|HTTP Status|
|-|-|-|
|Request DTOバリデーションエラー（`analytics[type]`未指定・`course_id`/`unit_id`型不正）|Presentation|400|
|`ErrInvalidAnalyticsType`|Presentation|400|
|`ErrCourseIDRequired` / `ErrUnitIDRequired`|Application|400|
|`ErrCourseNotFound` / `ErrUnitNotFound`|Application|404|
|未認証|Middleware|401|
|studentロール以外|Middleware|403|
|DB接続失敗・クエリ失敗|Infrastructure|500|
|他Context参照手段の呼び出し失敗（curriculum／Grade・Assessment Context連携エラー）|Infrastructure|500|

## Infrastructure Errorのハンドリング方針

infrastructure層で発生したエラーは`fmt.Errorf`でラップしてapplication層へ伝播させ、Handler層で未分類のエラーとして500に変換する。業務エラー（`ErrCourseIDRequired`等）と技術的障害（Infrastructure Error）は`errors.Is`で明確に区別する（コーディング規約「6. エラーハンドリング」に従う）。

---

# 12. GORM / DBクエリ設計

②「17. DB設計方針」により、既存Rails DBをそのまま継続利用し、スキーマ変更は行わない。

## 利用するGORMモデルとテーブルの対応

本Context自身は永続化対象を持たないため、独自のGORMモデルを新規定義しない。infrastructure関数は他Contextが所有する既存モデルを参照する想定とする。

|参照先|モデル（想定）|所有Context|備考|
|-|-|-|-|
|タスク完了状況|既存タスクモデル|task-management|詳細フィールドは②未記載のため、task-management Context側の③文書確定後に確認する（推測）|
|理解度スコア|既存理解度スコアモデル|Grade/Assessment Context|Grade/Assessment Contextは規約「5. Bounded Context構成」の正式なContext一覧に未掲載であり、②自身が「推測」と明記している（②「3. Bounded Context」）。本書もこの推測を維持する|
|成績|既存成績モデル|Grade/Assessment Context|同上|
|コース／単元|既存コース・単元モデル|curriculum|存在確認（`CourseExists`/`UnitExists`）に利用|

## 主要クエリの条件・ソート・ページネーション方針

|操作|条件|ソート|ページネーション|
|-|-|-|-|
|`FetchTaskCompletionSummary`|`user_id = ?`、`course_id`/`unit_id`指定時は追加条件|不要（集計のみ）|不要|
|`FetchUnderstandingScoreSummary`|`user_id = ?`、`course_id`/`unit_id`指定時は追加条件|不要（平均集計）|不要|
|`FetchGradeSummary`|`user_id = ?`、`course_id`/`unit_id`指定時は追加条件|不要（平均集計）|不要|
|`CourseExists`/`UnitExists`|`id = ?`|不要|不要|
|`FetchCourseRankCohort`|`course_id = ?`|スコア降順（順位算出のため。Go側で並べ替える想定、SQL側ORDER BYの要否は実装時に確定）|不要（対象コース内の全件を取得する想定）|
|`FetchUnitRankCohort`|`unit_id = ?`|同上|同上|

本機能に一覧表示用のページネーションは存在しない（②に一覧UIの記載がないため）。

## 既存Schemaへの変更

②「17. DB設計方針」により変更なし。SQL文そのものは本書に記載しない。

---

# 13. テストケース設計

②「18. テスト戦略」を、規約「4. 設計パターンごとの構造適用方針」および出力フォーマットのTransaction Script区分（「Domain Test」「Repository Test」は対象外、「UseCase Test」→「Application関数 Test」）に従って読み替える。

## Domain Test

対象外（Transaction Script採用のためDomain層を設けない）。`AnalyticsType`・`CompletionRate`・`UnderstandingScore`の妥当性判定・順位算出ロジックの検証は、以下の「Application関数 Test」「Handler Test」に含める。

## Application関数 Test

|対象|テストケース|
|-|-|
|`GetTaskCompletionAnalytics`|正しい集計結果が返る／対象タスクが0件の場合の完了率の扱い／`user_id`スコープが正しく適用される|
|`GetUnderstandingScoreAnalytics`|正しい集計結果が返る／`CompletionRate`/`UnderstandingScore`の範囲検証（不正値でエラーになること）|
|`GetGradeAverageAnalytics`|正しい平均値が返る／対象成績が0件の場合の扱い|
|`GetCourseRankAnalytics`|`CourseID`未指定時に`ErrCourseIDRequired`になる／存在しない`course_id`で`ErrCourseNotFound`になる／正しい順位・母数が返る|
|`GetUnitRankAnalytics`|`GetCourseRankAnalytics`と同様の観点（`ErrUnitIDRequired`・`ErrUnitNotFound`・正しい順位）|
|`calculateRank`（順位算出共有ロジック）|対象ユーザーの順位が正しく算出される／同点の扱い／対象ユーザーが比較対象データに含まれない場合にエラーになる／結果に他生徒の個人情報が含まれない|

## Repository Test

対象外（Transaction Script採用のためRepository層を設けない）。infrastructure関数のクエリ内容の検証は「Integration Test」に含める。

## Handler Test

|対象|テストケース|
|-|-|
|`AnalyticsHandler.GetAnalytics`|`analytics[type]`未指定時に400を返す／許容値以外の`type`指定時に400を返す（`ErrInvalidAnalyticsType`）／`course_id`/`unit_id`が数値変換不可の場合に400を返す／未認証時に401を返す／studentロール以外で403を返す／各`type`に応じたapplication関数への呼び分けが正しいこと／各`type`に応じたResponse DTOへの変換が正しいこと|

## Integration Test

|対象|テストケース|
|-|-|
|エンドポイント経由の各分析タイプ取得|`task_completion`/`understanding_score`/`grade_average`/`course_rank`/`unit_rank`それぞれで正しい結果が200で取得できること（②「18. テスト戦略」Integration Testに対応）|
|不正な`type`指定|400エラーが返ること|
|`course_rank`/`unit_rank`で`course_id`/`unit_id`未指定・不存在|400／404エラーが返ること|
|infrastructure関数のクエリ内容|`user_id`スコープ・`course_id`/`unit_id`絞り込みが実データに対して正しく適用されること（Repository Testの対象外化に伴い、DBアクセスを伴う検証としてここに含める）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に列挙する。

|判断した内容|判断理由|推測かどうか|
|-|-|-|
|ディレクトリ名を`internal/analytics`とした|②のContext名は`learning-analytics`のみで、ディレクトリ名の明記がない。アーキテクチャ規約「5. 命名規則」に従い短縮した（認証機能の前例に倣う）|推測|
|`AnalyticsType`・`CompletionRate`・`UnderstandingScore`・`RankCalculationService`（②のVO/Domain Service）の実装位置を、domain層ではなく`presentation/request`・`application`直下の単純な型・関数として確定した|規約「11. ①②③文書との関係」の「②文書内のRepository/UseCase等の記載は概念的な設計意図であり、実装構造への変換は③で行う」方針、および禁止事項「TS/AR採用機能へのDomain Model相当の過剰な構造追加の禁止」に基づく判断|規約に基づく判断（推測ではない）|
|`AnalyticsType`を`presentation/request`層に配置した|②「12. Validation設計」がAnalyticsTypeの許容値チェックをPresentation層の責務として明記しているため|②内の記載を整合させた判断（推測ではない）|
|`RankCalculationService`をstruct＋interfaceではなく、`application/rank_calculation.go`内の非公開共有関数として実装した|禁止事項「TS/AR採用機能へのDomain Model相当のusecase層・Repository Interface・依存性逆転の不要な追加」を避けるための判断|規約に基づく判断（推測ではない）|
|course_rank/unit_rankの順位算出で比較する指標（理解度スコアか成績平均か、統合指標か）を確定しなかった|②「9. Repository設計」RankRepositoryが「成績/理解度データ」とのみ記載し、指標を確定していない。①未提供のため参照不可|①未提供のため参照不可（推測不可・要確認事項）|
|タスク完了率0/0時・成績データ0件時のデフォルト値を0として扱う実装判断とした|②に該当ケースの取り扱いの明記がないため|推測|
|`analytics[type]`のブラケット記法クエリパラメータを、Ginの構造体タグ自動バインドではなく`c.Query("analytics[type]")`による明示的な取得で実装する方針とした|Ginの標準的なタグベースバインドがブラケット記法キーを確実に処理できるかについて②・規約いずれにも記載がない技術的判断であるため|推測|
|Response DTOのフィールド名（`CompletionRate`, `Score`, `Average`, `Rank`, `TotalCount`等）を②「6. Entity設計」「7. Value Object設計」の概念名からそのまま決定した|②「16. API互換方針」は「既存仕様に近い構造を維持する」とのみ記載し、フィールド名までは規定していない。①未提供のため既存Railsレスポンスのフィールド名を根拠にできない|推測|
|`GradeAverageResult`にVOを設けずplain `float64`のまま扱った|②「7. Value Object設計」がVO対象をCompletionRate/UnderstandingScoreに限定しており、成績平均をVO化する記載がないため、②に根拠のない業務ルールを追加しない方針を優先した|②の記載範囲を維持した判断（推測ではない）|
|`FetchCourseRankCohort`/`FetchUnitRankCohort`が返すスコアの算出元（Grade/Assessment Context側の具体的なモデル・カラム）を確定しなかった|Grade/Assessment Context自体が規約「5. Bounded Context構成」の正式なContext一覧に未掲載であり、②自身が「推測」と明記している事項を引き継いでいる|①未提供のため参照不可（②の推測を維持）|

上記以外の設計判断（Bounded Context・設計パターン・Aggregate・Repository・UseCase・Transaction境界・Authorization方針・Error設計の基本方針・API互換方針・DB方針・テスト戦略の基本方針）はすべて②の記載をそのまま踏襲しており、変更・追加した業務ルールはない。
