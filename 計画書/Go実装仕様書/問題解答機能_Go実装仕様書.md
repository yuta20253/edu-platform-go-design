# 問題解答機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒がタスクに紐づく単元の問題に回答し、回答結果を保存・更新し、提出時にタスク状態を更新する機能である。問題一覧取得・解答登録・解答更新・確認表示・提出という5つの操作を提供する（②1章）。

## 採用設計パターンとその理由（②からの要約）

②4章にて **Domain Model** を採用している。理由は、正誤判定・回答履歴の更新・提出時の進捗状態決定が学習成果に直結する中核業務ルールであり、単なるCRUDではなく回答内容・正誤結果・履歴の整合性を一貫して扱う必要があるためである。Transaction Script（ルールが手続きに散在しやすい）・Active Record寄りの設計（判定ロジックがモデルに散りやすい）・Event Sourcing（イベント再構築の要件がない）はいずれも採用しないと②に明記されている。

本書は②の採用パターン判断を変更しない。アーキテクチャ規約「4. 設計パターンごとの構造適用方針」の「Domain Model」節に従い、`domain/application/infrastructure/presentation` のフルレイヤー構成を適用する。

## 本書が対象とする実装範囲

- Bounded Context: `question-answering`（②3章）
- 対象UseCase: ListQuestionsUseCase / CreateAnswerUseCase / UpdateAnswerUseCase / ShowConfirmationUseCase / SubmitTaskUseCase（②10章）
- 対象API: ②16章に記載の5エンドポイント
- ①Rails実装（Controller/Service/Model等のコード詳細）は本タスクでは提供されていないため、「①未提供のため参照不可」として扱い、必要箇所ではその旨を明記する。

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `question-answering`（②3章）

Context名はkebab-caseだが、アーキテクチャ規約9章により `internal/` 配下は英単語1語または短いスネークケースとする。②には `internal/` 配下のディレクトリ名が明記されていないため、本書では `question_answering` をディレクトリ名として採用する（**②からの補足**。判断理由は14章参照）。

## ②で採用した設計パターン

- Domain Model（②4章）

## 採用パターンに対応する構造

アーキテクチャ規約「2. ディレクトリ構成」「4. 設計パターンごとの構造適用方針」のDomain Model標準構成をそのまま適用する。

## 作成するディレクトリ一覧

```
internal/question_answering/
├── domain/
│   ├── entity/
│   ├── valueobject/
│   ├── repository/
│   ├── service/
│   └── errors/
├── application/
│   ├── dto/
│   ├── command/
│   ├── query/
│   └── usecase/
├── infrastructure/
│   ├── persistence/
│   │   └── gorm/
│   └── repository/
└── presentation/
    ├── handler/
    ├── request/
    ├── response/
    └── routes.go
```

`domain/specification/` `domain/event/` `infrastructure/mail/` `infrastructure/cache/` `infrastructure/queue/` は本機能では **対象外**（②15章「Domain Eventは現時点で採用しない」、非同期通知・メール・キャッシュ・キューの要件が②に記載されていないため）。

## 作成するファイル一覧

```
# Domain層
internal/question_answering/domain/entity/question_history.go
internal/question_answering/domain/entity/answer_result.go
internal/question_answering/domain/entity/question_ref.go
internal/question_answering/domain/entity/question_choice_ref.go
internal/question_answering/domain/entity/task_ref.go
internal/question_answering/domain/valueobject/answer_status.go
internal/question_answering/domain/valueobject/answered_at.go
internal/question_answering/domain/repository/question_history_repository.go
internal/question_answering/domain/repository/question_repository.go
internal/question_answering/domain/repository/question_choice_repository.go
internal/question_answering/domain/repository/task_repository.go
internal/question_answering/domain/service/answer_evaluation_policy.go
internal/question_answering/domain/service/submission_progress_policy.go
internal/question_answering/domain/errors/errors.go

# Application層
internal/question_answering/application/query/list_questions_query.go
internal/question_answering/application/query/show_confirmation_query.go
internal/question_answering/application/command/create_answer_command.go
internal/question_answering/application/command/update_answer_command.go
internal/question_answering/application/command/submit_task_command.go
internal/question_answering/application/dto/question_list_item.go
internal/question_answering/application/dto/confirmation_item.go
internal/question_answering/application/usecase/list_questions_usecase.go
internal/question_answering/application/usecase/create_answer_usecase.go
internal/question_answering/application/usecase/update_answer_usecase.go
internal/question_answering/application/usecase/show_confirmation_usecase.go
internal/question_answering/application/usecase/submit_task_usecase.go

# Infrastructure層
internal/question_answering/infrastructure/persistence/gorm/question_history_model.go
internal/question_answering/infrastructure/repository/question_history_repository_impl.go
internal/question_answering/infrastructure/repository/question_repository_impl.go
internal/question_answering/infrastructure/repository/question_choice_repository_impl.go
internal/question_answering/infrastructure/repository/task_repository_impl.go

# Presentation層
internal/question_answering/presentation/handler/question_handler.go
internal/question_answering/presentation/handler/answer_handler.go
internal/question_answering/presentation/handler/submission_handler.go
internal/question_answering/presentation/request/create_answer_request.go
internal/question_answering/presentation/request/update_answer_request.go
internal/question_answering/presentation/response/question_list_response.go
internal/question_answering/presentation/response/answer_response.go
internal/question_answering/presentation/response/confirmation_response.go
internal/question_answering/presentation/response/submission_response.go
internal/question_answering/presentation/routes.go
```

Handlerを `question_handler.go` / `answer_handler.go` / `submission_handler.go` の3ファイルに分割する構成は②に明記がないため、**②からの補足**として扱う（6章参照）。

---

# 3. Domain層設計

## Entity

### QuestionHistory（Aggregate Root）

②6章・②5章より、回答履歴の中心概念であり、本機能唯一のAggregate Rootである。

- struct名: `QuestionHistory`
- 保持するフィールド:
  - `id uint`: 識別子
  - `userID uint`: 回答した生徒のユーザーID
  - `taskID uint`: 紐づくタスクID
  - `unitID uint`: 紐づく単元ID
  - `questionID uint`: 対象問題ID
  - `questionChoiceID *uint`: 選択した選択肢ID（選択式の場合。テキスト解答の場合はnil）
  - `answerText *string`: テキスト回答内容（テキスト解答形式の場合。選択式の場合はnil）
  - `result *AnswerResult`: 正誤判定結果（Aggregate内Entity）
  - `answeredAt AnsweredAt`: 解答時刻（Value Object）
  - `timeSpentSec int`: 解答所要時間（秒）
  - `explanationViewed bool`: 解説閲覧フラグ
- 各フィールドの意味: ②12章Validation設計に列挙された `task_id / unit_id / question_id / question_choice_id / answer_text / time_spent_sec / explanation_viewed` を根拠に整理した（フィールド一覧自体は**②からの補足**。具体的な型・nil許容の有無は本書で判断した）
- 公開するmethod一覧:
  - `NewQuestionHistory(userID, taskID, unitID, questionID uint, choiceID *uint, answerText *string) (*QuestionHistory, error)`: 新規解答登録時のファクトリ。責務は初期状態（未判定）の履歴を生成すること
  - `ApplyResult(result AnswerResult) error`: 判定結果を反映する。責務は判定結果と回答内容の整合を保証すること（②6章AnswerResultの責務「回答内容との整合を管理する」に対応）
  - `UpdateAnswer(choiceID *uint, answerText *string, timeSpentSec int) error`: 再解答時に回答内容を更新する。責務は既存履歴の更新ルールを管理すること（②6章「既存履歴がある場合の更新ルールを管理する」）
  - `MarkExplanationViewed() error`: 解説閲覧フラグを立てる
  - `Status() AnswerStatus`: 現在の正誤状態を返す参照系メソッド
  - `UserID() uint` / `TaskID() uint` / `UnitID() uint` / `QuestionID() uint`: 所有権確認等に用いる参照系メソッド
- 不変条件（コンストラクタで保証する内容）: `questionChoiceID` と `answerText` は少なくとも一方が設定されていること（②12章「回答が対象問題の選択肢であること」の整合性チェックに対応する入力レベルの不変条件。①未提供のため、選択式・テキスト解答の排他関係の詳細は推測）

### AnswerResult（Aggregate内Entity）

②6章では正誤判定の結果を表すEntityとされ、②7章ではAnswerStatusを正誤を表すValue Objectとしている。両者は矛盾しないよう、AnswerResultをAnswerStatusを内包するEntityとして整理する（**②からの補足**。判断理由は14章参照）。

- struct名: `AnswerResult`
- 保持するフィールド:
  - `id uint`: 識別子
  - `status AnswerStatus`: 正誤状態（Value Object）
  - `judgedAt time.Time`: 判定時刻
- 各フィールドの意味: ②6章「正誤結果を保持する」「回答内容との整合を管理する」に対応
- 公開するmethod一覧:
  - `NewAnswerResult(status AnswerStatus, judgedAt time.Time) (AnswerResult, error)`: 判定時に生成するファクトリ
  - `Status() AnswerStatus`: 正誤状態の参照
  - `IsCorrect() bool`: 正解かどうかの判定
- 不変条件: `status` は `AnswerStatus` の生成規則（correct/incorrectのいずれか）に従うことをVO側で保証する

### Question / QuestionChoice / Task の扱い

②6章ではTaskを「提出時の状態更新対象として参照される」Entityとして挙げている。ただし、②3章「他Contextとの依存関係」・アーキテクチャ規約6章「相手Contextの内部Entity・Value Object・Infrastructure実装に直接依存しない」により、Task・Question・QuestionChoiceは他Bounded Contextが所有するデータである。本書では、これらを **QuestionHistory Aggregateの外部参照用データ型（Ref型）** として、Entityとは区別して定義する（**②からの補足**。詳細は14章参照）。

- `QuestionRef`: `id uint`, `unitID uint`, `taskID uint`, `body string` 等、問題一覧表示・正誤判定に必要な最小限の参照フィールドを持つ読み取り専用データ
- `QuestionChoiceRef`: `id uint`, `questionID uint`, `isCorrect bool` 等、選択肢の妥当性確認・正誤判定に必要な参照フィールドを持つ読み取り専用データ（`isCorrect` は②に明記のない**推測**フィールド。14章参照）
- `TaskRef`: `id uint`, `userID uint`, `status string` 等、所有権確認・状態更新に必要な参照フィールドを持つ読み取り専用データ

これらはEntityのようにmethodで不変条件を強制するものではなく、Repositoryが返す読み取り専用の値である。ドメインルール判定（正誤判定・所有権確認）の入力として、Domain Service・UseCaseに渡す。

## Value Object

### AnswerStatus

- struct名: `AnswerStatus`
- 保持するフィールド: `value string`（非公開。`"correct"` または `"incorrect"` のみ許容）
- 生成時に検証するルール: `correct` / `incorrect` のいずれか以外の値を拒否する（②7章「correct / incorrect のいずれかのみ許容する」）
- 公開するmethod一覧:
  - `NewAnswerStatus(value string) (AnswerStatus, error)`: 生成時に許容値を検証するファクトリ
  - `IsCorrect() bool`
  - `String() string`

### AnsweredAt

- struct名: `AnsweredAt`
- 保持するフィールド: `value time.Time`（非公開）
- 生成時に検証するルール: 日付時刻の整合性を保証する（②7章。具体的な整合性条件は②に明記がなく、ゼロ値でないことの検証等を想定する。**推測**）
- 公開するmethod一覧:
  - `NewAnsweredAt(t time.Time) (AnsweredAt, error)`
  - `Time() time.Time`

## Value Objectを採用しないもの

②7章の通り、選択肢ID・問題ID・タスクIDはValue Object化しない。`uint` 等の基本型のまま扱う。

## Repository Interface

### QuestionHistoryRepository

②9章の責務・検索機能に基づく。

- interface名: `QuestionHistoryRepository`
- メソッドシグネチャ一覧:
  - `FindByUserAndQuestion(ctx context.Context, userID, taskID, unitID, questionID uint) (*entity.QuestionHistory, error)`: user_id/task_id/unit_id/question_idによる既存履歴の取得
  - `FindAllByTaskAndUnit(ctx context.Context, userID, taskID, unitID uint) ([]*entity.QuestionHistory, error)`: 一覧取得・確認表示用の検索
  - `Save(ctx context.Context, history *entity.QuestionHistory) error`: 新規履歴の保存
  - `Update(ctx context.Context, history *entity.QuestionHistory) error`: 既存履歴の更新
- 各メソッドの責務: 永続化と検索に限定し、正誤判定・提出状態の決定・認可判定は持たない（②9章「保持しない責務」）

### QuestionRepository

- interface名: `QuestionRepository`
- メソッドシグネチャ一覧:
  - `FindByID(ctx context.Context, questionID uint) (*entity.QuestionRef, error)`: 問題の存在確認
  - `FindAllByUnitAndTask(ctx context.Context, taskID, unitID uint) ([]*entity.QuestionRef, error)`: unit_id/task_idに紐づく問題取得
- 各メソッドの責務: 問題情報の参照に特化し、回答の正誤判定そのものは持たない（②9章）

### QuestionChoiceRepository

- interface名: `QuestionChoiceRepository`
- メソッドシグネチャ一覧:
  - `FindByID(ctx context.Context, choiceID uint) (*entity.QuestionChoiceRef, error)`: 選択肢の存在確認
  - `ExistsForQuestion(ctx context.Context, questionID, choiceID uint) (bool, error)`: 選択肢が対象問題に属するかの確認
- 各メソッドの責務: 選択肢の参照と妥当性確認に集中させ、正誤判定の最終決定は持たない（②9章）

### TaskRepository

- interface名: `TaskRepository`
- メソッドシグネチャ一覧:
  - `FindByID(ctx context.Context, taskID uint) (*entity.TaskRef, error)`: 指定タスクの取得
  - `UpdateStatus(ctx context.Context, taskID uint, status string) error`: タスク状態の更新
- 各メソッドの責務: 永続化と状態更新の責務に限定し、提出状態の判断ロジックは持たない（②9章）

これら4つのRepository Interfaceは `domain/repository/` に定義する（規約「5. インターフェース」の「利用するパッケージ側で定義する」原則、および本規約アーキテクチャ規約3章の依存性逆転に従う）。

## Domain Service

### AnswerEvaluationPolicy

- struct/interface名: `AnswerEvaluationPolicy`（struct）
- メソッドシグネチャ: `Evaluate(question *entity.QuestionRef, choice *entity.QuestionChoiceRef, answerText *string) (entity.AnswerResult, error)`
- 責務: 問題の正誤判定ロジックをまとめて扱う（②8章）。Entityへ持たせない理由は、判定が問題・選択肢・回答内容の組み合わせに依存し、QuestionHistory単体で完結しないため（②8章）

### SubmissionProgressPolicy

- struct/interface名: `SubmissionProgressPolicy`（struct）
- メソッドシグネチャ: `Determine(histories []*entity.QuestionHistory, task *entity.TaskRef) (string, error)`（戻り値は決定後のタスク状態を表す文字列。具体的な状態値の集合は②に明記がなく、①未提供のため確認不可。**推測**）
- 責務: 提出時にタスク状態をどのように決定するかを判定する（②8章）。回答履歴の集合とタスクの状況に依存し、単一Entityに閉じ込めにくいため独立させる（②8章）

## Domain Error

②14章の分類に基づき、`domain/errors/` に集約する。

- エラー種別ごとの型／変数定義方針: sentinel error（`var Err〇〇 = errors.New(...)`）を基本形とし、必要に応じて `errors.Is` で判定できるカスタムエラー型を用いる。具体的な実装方法（sentinel/カスタム型の使い分け）は②・規約のいずれにも明記がなく、コーディング規約21章「`errors.New`と`fmt.Errorf`の使い分けについて独自ルールを追加しない」に従い、本書では実装者の判断に委ねる方針だけを示す
- 発生条件（②14章）:
  - `ErrChoiceNotBelongToQuestion`: 選択肢が対象問題に属さない
  - `ErrHistoryNotFound`: 既存履歴が存在しない（更新対象の再解答時等）
  - `ErrInvalidSubmissionTransition`: 提出状態の遷移が不正である

---

# 4. Application層設計

Domain Model採用のため、UseCase struct + Repository Interfaceとして記載する（②の採用パターンに対応するアーキテクチャ規約4章の記載どおり）。

## DTO（Command / Query）

`application/query/`, `application/command/`, `application/dto/` に配置する。

- `ListQuestionsQuery`（Query）: `UserID uint`, `TaskID uint`, `UnitID uint`
- `ShowConfirmationQuery`（Query）: `UserID uint`, `TaskID uint`, `UnitID uint`
- `CreateAnswerCommand`（Command）: `UserID uint`, `TaskID uint`, `UnitID uint`, `QuestionID uint`, `QuestionChoiceID *uint`, `AnswerText *string`, `TimeSpentSec int`, `ExplanationViewed bool`
- `UpdateAnswerCommand`（Command）: `CreateAnswerCommand` と同一フィールド構成（更新対象の識別は `UserID/TaskID/UnitID/QuestionID` の組で行う）
- `SubmitTaskCommand`（Command）: `UserID uint`, `TaskID uint`
- `dto.QuestionListItem`: `QuestionID uint`, `Body string`, `Choices []QuestionChoiceItem`, `AlreadyAnswered bool`, `Status *string`（UseCase出力の一部として使用）
- `dto.ConfirmationItem`: `QuestionID uint`, `Status string`, `AnswerText *string`, `QuestionChoiceID *uint`, `ExplanationViewed bool`

フィールド構成は②10章の「入力」「出力」の記述（current user, task id, unit id, question id, answer payload 等）と②12章のRequestフィールドを根拠に具体化した（**②からの補足**。フィールド名・型自体は本書で決定）。

## UseCase

### ListQuestionsUseCase

- struct名: `ListQuestionsUseCase`
- コンストラクタが受け取る依存: `QuestionRepository`, `QuestionHistoryRepository`, `TaskRepository`（②9章の呼び出すRepositoryはQuestionRepository/QuestionHistoryRepositoryのみだが、②13章「UseCaseはタスク・単元・問題に対する所有権を前提に処理を実行する」を満たすため、TaskRepositoryを所有権確認用に追加する。**②からの補足**。詳細は14章）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query query.ListQuestionsQuery) (*dto.QuestionListResult, error)`
- 処理ステップ:
  1. TaskRepositoryでタスクを取得し、`TaskRef.UserID` がクエリの `UserID` と一致するか確認する（所有権チェック）
  2. QuestionRepositoryでtask_id/unit_idに紐づく問題一覧を取得する
  3. QuestionHistoryRepositoryでuser_id/task_id/unit_idに紐づく既存履歴を取得する
  4. 問題一覧と既存履歴を突き合わせ、既回答状態を含む結果を組み立てる
- トランザクション境界: なし（読み取りのみ、②11章）
- 発生しうるApplication Error: タスクが存在しない／所有権がない、対象単元が存在しない

### CreateAnswerUseCase

- struct名: `CreateAnswerUseCase`
- コンストラクタが受け取る依存: `QuestionRepository`, `QuestionChoiceRepository`, `QuestionHistoryRepository`, `TaskRepository`, `AnswerEvaluationPolicy`（TaskRepositoryは所有権確認のための追加。**②からの補足**。14章参照）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd command.CreateAnswerCommand) (*dto.AnswerResultDTO, error)`
- 処理ステップ:
  1. TaskRepositoryでタスク取得・所有権確認
  2. QuestionRepositoryで対象問題の存在確認
  3. QuestionChoiceRepositoryで選択肢が対象問題に属するか確認（選択式の場合）
  4. AnswerEvaluationPolicyで正誤判定を行い`AnswerResult`を生成する
  5. `QuestionHistory`エンティティを生成し、判定結果を反映する
  6. QuestionHistoryRepository.Saveで履歴を保存する
- トランザクション境界: 履歴保存を1トランザクションで扱う（②11章）
- 発生しうるApplication Error: 対象タスクが存在しない、対象問題が見つからない、選択肢が問題に属さない、保存処理の失敗（②14章）

### UpdateAnswerUseCase

- struct名: `UpdateAnswerUseCase`
- コンストラクタが受け取る依存: `QuestionHistoryRepository`, `QuestionChoiceRepository`, `AnswerEvaluationPolicy`（②9章の呼び出すRepository一覧どおり。QuestionRepositoryは②に明記がないため含めない。正誤再判定に問題情報が必要な場合はQuestionChoiceRefのみで完結する前提とする。**推測**。14章参照）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd command.UpdateAnswerCommand) (*dto.AnswerResultDTO, error)`
- 処理ステップ:
  1. QuestionHistoryRepository.FindByUserAndQuestionで既存履歴を取得する（存在しない場合は`ErrHistoryNotFound`）
  2. QuestionChoiceRepositoryで新たな選択肢が対象問題に属するか確認する
  3. AnswerEvaluationPolicyで再判定を行う
  4. `QuestionHistory.UpdateAnswer`で回答内容を更新し、`ApplyResult`で判定結果を反映する
  5. QuestionHistoryRepository.Updateで更新を保存する
- トランザクション境界: 既存履歴更新を1トランザクションで扱う（②11章）
- 発生しうるApplication Error: 既存履歴が存在しない、選択肢が問題に属さない、保存処理の失敗

### ShowConfirmationUseCase

- struct名: `ShowConfirmationUseCase`
- コンストラクタが受け取る依存: `QuestionRepository`, `QuestionHistoryRepository`, `TaskRepository`（TaskRepositoryは所有権確認用の追加。**②からの補足**）
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, query query.ShowConfirmationQuery) (*dto.ConfirmationResult, error)`
- 処理ステップ:
  1. TaskRepositoryでタスク取得・所有権確認
  2. QuestionRepositoryで対象問題一覧を取得する
  3. QuestionHistoryRepositoryで既存履歴を取得する
  4. 問題と履歴を照合し、確認表示用の状態を組み立てる
- トランザクション境界: なし（読み取りのみ、②11章）
- 発生しうるApplication Error: タスクが存在しない／所有権がない

### SubmitTaskUseCase

- struct名: `SubmitTaskUseCase`
- コンストラクタが受け取る依存: `TaskRepository`, `QuestionHistoryRepository`, `SubmissionProgressPolicy`
- 公開メソッドのシグネチャ: `Execute(ctx context.Context, cmd command.SubmitTaskCommand) (*dto.SubmissionResultDTO, error)`
- 処理ステップ:
  1. TaskRepositoryでタスクを取得し、所有権を確認する
  2. QuestionHistoryRepositoryで対象タスクに紐づく回答履歴一覧を取得する
  3. SubmissionProgressPolicyで回答履歴の集合とタスク状況から新しいタスク状態を決定する
  4. TaskRepository.UpdateStatusで状態を更新する
- トランザクション境界: タスク状態更新を1トランザクションで扱う（②11章）
- 発生しうるApplication Error: 対象タスクが存在しない、提出状態の遷移が不正である

---

# 5. Infrastructure層設計

## Repository実装（Domain Model採用時）

### QuestionHistoryRepositoryImpl

- 実装struct名: `QuestionHistoryRepositoryImpl`（package: `infrastructure/repository`）
- 対応するGORMモデル: `gorm.QuestionHistoryModel`（テーブル: `question_histories`。②17章に明記）
- 各メソッドで発行するクエリ内容:
  - `FindByUserAndQuestion`: `user_id`, `task_id`, `unit_id`, `question_id` を条件とした単一レコード取得
  - `FindAllByTaskAndUnit`: `user_id`, `task_id`, `unit_id` を条件とした一覧取得（ソート・ページネーションは②に明記なし。件数が少数のため未使用と想定。**推測**）
  - `Save`: 新規レコードの作成
  - `Update`: 主キー指定での更新
- Entity ⇔ GORMモデルの変換方針: `infrastructure/repository` 内の非公開変換関数（`toEntity` / `toModel`）で相互変換する。GORMモデルはドメイン層に漏らさない

### QuestionRepositoryImpl / QuestionChoiceRepositoryImpl / TaskRepositoryImpl

②3章「Task Context / Unit Context / Question Contextへの依存」、アーキテクチャ規約6章「他Contextのデータを利用する場合は、相手Contextが公開する参照手段を呼び出す」に基づき、これら3つのRepository実装は**question-answering Context自身のテーブルを直接操作するのではなく、依存先Contextが公開する参照手段（Repository/Storeの参照系メソッド、または参照用の関数）を呼び出す**構成とする。

- `TaskRepositoryImpl`: Task Context（アーキテクチャ規約5章の一覧より `task-management` Contextに対応すると推測。**推測**、14章参照）が公開する参照・更新手段を呼び出す
- `QuestionRepositoryImpl` / `QuestionChoiceRepositoryImpl`: ②内で「Question Context」と呼ばれる依存先の実装上の対応Contextが②・アーキテクチャ規約のいずれにも明記されていない。本書では対応関係を確定できないため、**②からの補足**として「依存先Contextが公開する参照手段を呼び出す」という方針のみを示し、具体的なパッケージ・関数名の設計は対象外とする（14章参照）

これらの実装がGORMモデルを直接扱うか、依存先Contextのpackage関数を経由するかは、依存先Context側の②③文書と合わせて確定する必要がある。本書のスコープでは、Repository Interfaceのシグネチャ（3章）までを規定する。

## Store実装（Active Record採用時）

対象外（本機能はDomain Modelを採用しており、Active Record構成のStoreは設けない）。

## Infrastructure関数（Transaction Script採用時）

対象外（本機能はDomain Modelを採用しており、Transaction Script構成のinfrastructure関数は設けない）。

## 外部連携実装

Mail・Cache・Queue等は②に必要とされる記載がないため、**対象外**とする（②15章「Domain Eventを採用しない」、②に非同期処理・外部通知の記載がないことに基づく）。

---

# 6. Presentation層設計

## Handler

②に明記のないHandler分割方針として、機能単位で3つのHandlerに分ける（**②からの補足**）。

### QuestionHandler

- struct名: `QuestionHandler`
- 対応する呼び出し先: `ListQuestionsUseCase`, `ShowConfirmationUseCase`
- メソッド一覧:
  - `ListQuestions(c *gin.Context)`: `GET /api/v1/student/tasks/:task_id/units/:unit_id/questions`
  - `ShowConfirmation(c *gin.Context)`: `GET /api/v1/student/tasks/:task_id/units/:unit_id/confirmation`
- 処理順序: パスパラメータ・current userのバインド → Validation（型・必須） → 対応UseCaseのExecute呼び出し → Response DTOへの変換 → JSONレスポンス返却

### AnswerHandler

- struct名: `AnswerHandler`
- 対応する呼び出し先: `CreateAnswerUseCase`, `UpdateAnswerUseCase`
- メソッド一覧:
  - `CreateAnswer(c *gin.Context)`: `POST /api/v1/student/answers`
  - `UpdateAnswer(c *gin.Context)`: `PATCH /api/v1/student/answers`
- 処理順序: リクエストボディのバインド → Request DTOでのValidation → 対応UseCaseのExecute呼び出し → Response DTOへの変換 → JSONレスポンス返却

### SubmissionHandler

- struct名: `SubmissionHandler`
- 対応する呼び出し先: `SubmitTaskUseCase`
- メソッド一覧:
  - `SubmitTask(c *gin.Context)`: `PATCH /api/v1/student/tasks/:task_id/submission`
- 処理順序: パスパラメータのバインド → Validation → `SubmitTaskUseCase.Execute`呼び出し → Response DTOへの変換 → JSONレスポンス返却

いずれのHandlerも業務ロジック・権限判定の本体は持たず、UseCaseへ委譲する（アーキテクチャ規約3章「Presentationは業務ロジックを持たない」）。

## Request / Response DTO

### CreateAnswerRequest

- struct名: `CreateAnswerRequest`
- フィールドと型: `TaskID uint`, `UnitID uint`, `QuestionID uint`, `QuestionChoiceID *uint`, `AnswerText *string`, `TimeSpentSec int`, `ExplanationViewed bool`
- バリデーションタグ／チェック内容（②12章Presentation節に対応）:
  - `TaskID` / `UnitID` / `QuestionID`: 必須・型チェック（`binding:"required"`）
  - `QuestionChoiceID` / `AnswerText`: いずれか一方が必須（構造体レベルのカスタムバリデーションを想定）
  - `TimeSpentSec`: フォーマットチェック（0以上の整数）
  - `ExplanationViewed`: bool型チェック

### UpdateAnswerRequest

- struct名: `UpdateAnswerRequest`
- フィールドと型: `CreateAnswerRequest` と同一構成
- バリデーションタグ／チェック内容: `CreateAnswerRequest` と同様

### Response DTO

- `QuestionListResponse`: `Questions []QuestionItemResponse`
- `AnswerResponse`: `QuestionHistoryID uint`, `Status string`, `AnsweredAt string`
- `ConfirmationResponse`: `Items []ConfirmationItemResponse`
- `SubmissionResponse`: `TaskID uint`, `Status string`

Response DTOはEntityを直接返さず、UseCaseの出力DTOから変換する（アーキテクチャ規約7章「Entityをそのままレスポンスとして返さない」）。

## Routing

`presentation/routes.go` に以下を登録する。

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/student/tasks/:task_id/units/:unit_id/questions|QuestionHandler.ListQuestions|
|POST|/api/v1/student/answers|AnswerHandler.CreateAnswer|
|PATCH|/api/v1/student/answers|AnswerHandler.UpdateAnswer|
|GET|/api/v1/student/tasks/:task_id/units/:unit_id/confirmation|QuestionHandler.ShowConfirmation|
|PATCH|/api/v1/student/tasks/:task_id/submission|SubmissionHandler.SubmitTask|

これらのルートは、認証・studentロール確認Middlewareを適用したRouterGroup配下に登録する（②13章、10章参照）。

---

# 7. API仕様

②16章のAPI互換方針に基づく実装対象Endpoint一覧。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/student/tasks/:task_id/units/:unit_id/questions|QuestionHandler.ListQuestions|なし（パスパラメータのみ）|QuestionListResponse|200|
|POST|/api/v1/student/answers|AnswerHandler.CreateAnswer|CreateAnswerRequest|AnswerResponse|200（②に成功時ステータスの明記は「200: 取得成功」のみで作成系の明記なし。本書では既存レスポンス互換のため200とする。**推測**。14章参照）|
|PATCH|/api/v1/student/answers|AnswerHandler.UpdateAnswer|UpdateAnswerRequest|AnswerResponse|200|
|GET|/api/v1/student/tasks/:task_id/units/:unit_id/confirmation|QuestionHandler.ShowConfirmation|なし（パスパラメータのみ）|ConfirmationResponse|200|
|PATCH|/api/v1/student/tasks/:task_id/submission|SubmissionHandler.SubmitTask|なし（パスパラメータのみ）|SubmissionResponse|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|認証なし・トークン無効|401（②に明記なし。一般的なMiddleware認証失敗として想定。**推測**）|認証エラー|
|studentロールでない|403（②に明記なし。**推測**）|権限エラー|
|対象タスク・単元・問題が存在しない|404|対象データ不存在（②16章）|
|自分のタスクに属さない対象への操作|404（存在しない扱いとして統一するか403にするかは②に明記なし。**推測**。14章参照）|所有権エラー|
|入力値の型・必須・フォーマット不正|422|入力検証エラー（②16章）|
|選択肢が対象問題に属さない等の業務ルール違反|422|業務ルール違反（②16章、②14章Domain Error）|
|既存履歴が存在しない（更新時）|422または404（②に明記なし。**推測**。本書では404を推奨）|履歴不存在|
|提出状態の遷移が不正|422|状態遷移エラー（②14章）|
|保存処理・DB接続の失敗|500（②に明記なし。一般的なInfrastructure Errorとして想定。**推測**）|内部エラー|

---

# 8. Transaction実装方針

②11章のTransaction境界を実装単位に落とし込む。

## Transaction開始箇所

- UseCase内の、Repository書き込み操作を開始する直前（②11章「UseCaseの開始時にトランザクションを開始する」）で開始する
- 対象: `CreateAnswerUseCase.Execute` / `UpdateAnswerUseCase.Execute` / `SubmitTaskUseCase.Execute`

具体的な実装パターンは規約に定めがないため、一例として、Infrastructure層に `TransactionManager` インターフェース（`WithinTransaction(ctx context.Context, fn func(ctx context.Context) error) error`）を用意し、UseCaseがこれをコンストラクタ依存として受け取って処理全体をラップする方式を示す（**②からの補足**。規約に具体的なトランザクション実装パターンの定めがないため、実装者が別の一貫した方式を採用してもよい）。

## Transaction終了箇所

- コミット条件: 保存・更新（`QuestionHistoryRepository.Save/Update`、`TaskRepository.UpdateStatus`）が正常に完了した時点
- ロールバック条件: Domain Error・Application Errorが発生した時点、またはRepository呼び出しがエラーを返した時点
- `ListQuestionsUseCase` / `ShowConfirmationUseCase` はトランザクションを使用しない（②11章、読み取りのみ）

## 複数Repositoryにまたがる場合の扱い

- `CreateAnswerUseCase`: `QuestionHistoryRepository.Save`のみが書き込み対象であり、`QuestionRepository`/`QuestionChoiceRepository`/`TaskRepository`は参照のみのため、書き込みを伴う`QuestionHistoryRepository`をトランザクションスコープに含める
- `SubmitTaskUseCase`: `TaskRepository.UpdateStatus`と`QuestionHistoryRepository`（参照のみ）にまたがるが、状態更新はTaskRepositoryのみのため、TaskRepositoryの更新操作を単一トランザクションとする
- 他Context（Task/Question）のRepository実装が依存先Contextの永続化機構を経由する場合、トランザクション境界が本Context内で完結しない可能性がある。この点は②に明記がなく、依存先Context側の実装と合わせて確認が必要（**②からの補足**、14章参照）

---

# 9. Validation実装方針

②12章のValidation設計を実装レベルに落とし込む。

## Presentation

- `CreateAnswerRequest` / `UpdateAnswerRequest`（6章参照）で以下を検証する:
  - 型チェック: `task_id` / `unit_id` / `question_id` / `question_choice_id` の型
  - 必須チェック: 回答登録・更新時に必要な識別子（②12章）
  - フォーマットチェック: `answer_text` / `time_spent_sec` / `explanation_viewed` の形式（②12章）

## 業務ルール検証

Domain Model採用のため、Entity／Value Object生成時の検証とUseCase内で判定する業務ルールに分ける。

- Entity/Value Object生成時: `QuestionHistory.NewQuestionHistory`で選択肢ID・回答テキストの排他条件を検証、`AnswerStatus.NewAnswerStatus`でcorrect/incorrectのみ許容、`AnsweredAt.NewAnsweredAt`で日時整合性を検証（3章参照）
- UseCase内: 「指定された問題・選択肢・タスク・単元が正しく紐づいているか」（②12章）を`CreateAnswerUseCase`/`UpdateAnswerUseCase`の処理ステップ内（QuestionRepository/QuestionChoiceRepositoryの参照結果を用いた検証）で判定する。「既存履歴がある場合に更新対象として妥当か」は`UpdateAnswerUseCase`のステップ1（FindByUserAndQuestion）で判定する。「提出時に回答済み状態が反映される」は`SubmissionProgressPolicy`で判定する

## 責務分離

Presentationは「入力が適切か」、Domain（Entity/VO）とApplication（UseCase）は「業務上妥当か」を担当する（②12章の方針を維持）。

---

# 10. Authorization実装方針

②13章のAuthorization設計を実装レベルに落とし込む。

## Middleware

- JWT等による認証済みユーザーの特定と、studentロールであることの確認を行う（②13章）。実装は横断的なMiddleware（本Bounded Context固有の実装ではなく、`shared/`または`auth` Context側で提供されるMiddlewareを利用する想定。②に具体的なMiddleware実装の記載はないため**推測**）

## Handler

- 認証失敗時のレスポンスを整える。業務権限の判定は持たない（②13章）

## UseCase

- current userのタスク・単元・問題に対する所有権を前提に処理を実行する（②13章）。実装としては、各UseCaseの処理ステップの先頭で`TaskRepository.FindByID`により取得した`TaskRef.UserID`とcurrent userのIDを比較する（4章の各UseCase処理ステップ参照）
- 取得・更新対象が自分のタスクに属するかを確認する（②13章）。`QuestionHistoryRepository`の検索条件に`user_id`を含めることで、対象データ自体を自分の範囲に限定する（3章のRepository設計）

## Domain

- 回答履歴や提出処理に対して、所有者外の操作が行われないようにする。ただし認可の本体はUseCase側に寄せる（②13章）。Domain層では所有者IDの比較ロジック自体は持たず、UseCaseから渡された正当なQuestionHistory/TaskRefのみを扱う前提とする

---

# 11. Error実装方針

②14章のError設計を実装レベルに落とし込む。

## Domain Error → Application Errorへの変換方針

- `domain/errors/`で定義したDomain Error（`ErrChoiceNotBelongToQuestion`, `ErrHistoryNotFound`, `ErrInvalidSubmissionTransition`）を、UseCase内で`errors.Is`により判定し、対応するApplication Errorへラップして返す。ラップには`fmt.Errorf`と`%w`を用いる（コーディング規約6章）

## Application Error → HTTPレスポンスへの変換方針

- Handlerまたは共通のエラーハンドリングMiddlewareで、Application Errorの種別に応じてHTTP Status Codeを決定し、統一されたエラーレスポンス形式（②16章「既存のerrors構造を意識しつつGoの実装に合わせて整形する」）に変換する

## Infrastructure Errorのハンドリング方針

- DB接続失敗・永続化失敗等のInfrastructure Errorは、Repository実装内でApplication Error相当（例: 汎用的な「保存処理の失敗」エラー）にラップしてApplication層に伝播させ、ドメインには漏らさない（②14章）

|Error種別|発生層|HTTP Status|
|-|-|-|
|`ErrChoiceNotBelongToQuestion`|Domain|422|
|`ErrHistoryNotFound`|Domain|404|
|`ErrInvalidSubmissionTransition`|Domain|422|
|対象タスク／問題が存在しない|Application|404|
|入力値検証エラー|Presentation|422|
|認証エラー|Presentation（Middleware）|401（**推測**）|
|権限エラー（ロール不一致）|Presentation（Middleware）|403（**推測**）|
|DB接続・永続化failure|Infrastructure|500（**推測**）|

---

# 12. GORM / DBクエリ設計

②17章「既存Rails DBを継続利用、Schema変更なし」に基づく。

## 利用するGORMモデルとテーブルの対応

- `gorm.QuestionHistoryModel` ⇔ `question_histories`テーブル（②17章に明記）
- `Question` / `QuestionChoice` / `Task`のテーブル対応（`questions` / `question_choices` / `tasks`）は②に明記がなく、Gorm規約「構造体名の複数形をテーブル名とする」規約に基づく**推測**である。これらは他Context所有データのため、本Context側で新規GORMモデルを定義せず、依存先Contextが提供する参照手段を経由する想定（5章参照）

## 主要クエリの条件・ソート・ページネーション方針

- `question_histories`検索: `user_id` + `task_id` + `unit_id`（一覧・確認表示用）、または`user_id` + `task_id` + `unit_id` + `question_id`（単一履歴取得用）で絞り込む（②9章「保持する検索機能」）
- ソート・ページネーション: ②に明記がなく、1タスク・1単元あたりの問題数は限定的であることを前提に、本書ではページネーションを設けない方針とする（**推測**）
- GORMモデルの`CreatedAt`/`UpdatedAt`はGorm規約のタイムスタンプ自動トラッキングに従う

## 既存Schemaに対する変更

- ②17章「変更なし」の方針をそのまま維持する。本書でも新規カラム・テーブルの追加提案は行わない

---

# 13. テストケース設計

②18章のテスト戦略をテストケース単位に具体化する。Domain Model採用のため、区分はそのまま使用する。

## Domain Test

|対象|テストケース|
|-|-|
|`AnswerStatus.NewAnswerStatus`|"correct"/"incorrect"を許容し、それ以外の値を拒否すること|
|`AnsweredAt.NewAnsweredAt`|ゼロ値・不正な日時を拒否すること|
|`QuestionHistory.NewQuestionHistory`|選択肢ID・回答テキストが両方nilの場合にエラーとなること|
|`QuestionHistory.UpdateAnswer`|再解答時に既存の回答内容が更新されること|
|`AnswerEvaluationPolicy.Evaluate`|選択式回答が正解選択肢と一致する場合にcorrectとなること／一致しない場合にincorrectとなること|
|`SubmissionProgressPolicy.Determine`|全問回答済みの場合と未回答が残る場合とで、決定される状態が異なること（②に具体的な状態遷移条件の記載がないため、詳細ケースは①未提供につき確認不可。**推測**）|

## UseCase Test

|対象|テストケース|
|-|-|
|`ListQuestionsUseCase`|自分のタスクに紐づく問題一覧と既回答状態が取得できること／他人のタスクの場合にエラーとなること|
|`CreateAnswerUseCase`|正常な選択式回答で履歴が保存され判定結果が返ること／選択肢が対象問題に属さない場合にエラーとなること|
|`UpdateAnswerUseCase`|既存履歴が更新され再判定結果が反映されること／既存履歴が存在しない場合にエラーとなること|
|`ShowConfirmationUseCase`|問題と履歴が正しく突き合わされ確認表示用データが返ること|
|`SubmitTaskUseCase`|回答履歴の集合に基づきタスク状態が更新されること／不正な状態遷移の場合にエラーとなること|

## Repository Test

|対象|テストケース|
|-|-|
|`QuestionHistoryRepositoryImpl.FindByUserAndQuestion`|条件に一致する履歴が取得できること／存在しない場合にnilまたはエラーが返ること|
|`QuestionHistoryRepositoryImpl.FindAllByTaskAndUnit`|user_id/task_id/unit_idで絞り込まれた一覧が取得できること|
|`QuestionHistoryRepositoryImpl.Save` / `Update`|保存・更新後にDB上のレコードが期待どおりになること|
|`QuestionRepositoryImpl` / `QuestionChoiceRepositoryImpl` / `TaskRepositoryImpl`|依存先Contextの参照手段経由で正しくデータが取得できること（依存先Context側の実装に依存するため、本書のスコープでは方針レベルの確認に留まる）|

## Handler Test

|対象|テストケース|
|-|-|
|`AnswerHandler.CreateAnswer`|必須項目欠如時に422が返ること／正常時に200とAnswerResponseが返ること|
|`AnswerHandler.UpdateAnswer`|存在しない履歴に対する更新で404または422が返ること|
|`QuestionHandler.ListQuestions`|認証なしで401（**推測**）が返ること／正常時に200が返ること|
|`SubmissionHandler.SubmitTask`|他人のタスクに対する提出で404が返ること（**推測**）|

## Integration Test

|対象|テストケース|
|-|-|
|問題一覧取得→回答登録→回答更新→確認表示→提出|一連の操作がエンドポイント経由で一貫して動作し、最終的にタスク状態が更新されること（②18章Integration Test方針）|

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容を以下に整理する。①（Rails実装）は本タスクでは未提供のため、参照が必要な箇所も併せて明記する。

|判断した内容|判断理由|推測／補足の別|
|-|-|-|
|Bounded Context名`question-answering`のディレクトリ名を`question_answering`（スネークケース）とする|アーキテクチャ規約9章はディレクトリ名を「英単語1語または短いスネークケース」とするが、複数語Context名の具体的な変換規則は明記がないため|補足（実装のための判断）|
|`QuestionHistory`/`AnswerResult`の具体的なフィールド構成・型|②はEntityの役割・責務を概念的に記述するのみで、フィールド一覧は明記がない。②12章Validation設計に列挙された`task_id/unit_id/question_id/question_choice_id/answer_text/time_spent_sec/explanation_viewed`を手がかりに具体化した|補足（②記載事項からの具体化）|
|`AnswerResult`を`AnswerStatus`（VO）を内包するEntityとして整理|②6章（AnswerResultをEntityとする）と②7章（AnswerStatusを正誤のVOとする）が共存しており、そのままでは正誤情報の持ち方が重複する。両者を矛盾なく統合するため、AnswerResult EntityがAnswerStatus VOを内包する構造とした|補足（②内の記述整合のための構造化）|
|`Question`/`QuestionChoice`/`Task`を本Context内では「他Context所有の参照用データ型（Ref型）」として扱い、Entityとして本Contextのドメインに含めない|アーキテクチャ規約6章「相手Contextの内部Entity・Value Object・Infrastructure実装に直接依存しない」に基づく。②6章はTaskをEntityとして挙げているが、これは概念説明であり実装構造への変換は③（本書）の役割（アーキテクチャ規約11章）|補足|
|Task Contextを`task-management`、Unit Contextを`curriculum`と推測し、Question Contextの実装上の対応Contextは確定できないとした|アーキテクチャ規約5章のBounded Context一覧を根拠とするが、②内の「Question Context」に一致するContext名が一覧に見当たらないため|推測（Task/Unitのみ）・確認不可（Question）|
|`TaskRepositoryImpl`/`QuestionRepositoryImpl`/`QuestionChoiceRepositoryImpl`は依存先Contextの公開参照手段を経由し、本Context内でGORMモデルを直接定義しない|アーキテクチャ規約6章のContext間連携ルールに基づく。②のRepository設計は責務を概念的に記述するのみで、実装連携方式は明記がない|補足|
|`ListQuestionsUseCase`/`ShowConfirmationUseCase`/`CreateAnswerUseCase`にTaskRepositoryを追加で依存させ、タスク所有権を確認する|②9章の「呼び出すRepository」一覧にはTaskRepositoryが明記されていないが、②13章Authorization設計「UseCaseは所有権を前提に処理を実行する」を満たすために必要と判断した。既存のRepository Interface自体を変更するものではない|補足（①未提供のため実装時要確認）|
|`UpdateAnswerUseCase`の再判定はQuestionChoiceRepositoryの情報のみで完結する前提とした（QuestionRepositoryは追加しない）|②9章の呼び出すRepository一覧にQuestionRepositoryが明記されていないため、②の記載どおりに留めた。ただし再判定に問題本体の情報が必要な場合は不足する可能性がある|推測（①未提供のため確認不可）|
|`QuestionChoiceRef`が正解フラグ（`isCorrect`相当）を保持する前提とした|②はAnswerEvaluationPolicyの責務（正誤判定）のみを記述し、正解情報の保持元を明記していない。選択式問題である以上、選択肢側が正解フラグを持つ構成を仮定した|推測（①未提供のため確認不可）|
|GORMモデルのテーブル対応（`questions`/`question_choices`/`tasks`）|②17章では`question_histories`のみ明記されている。他はGorm規約「構造体名の複数形」規約に基づく推測|推測|
|POST/PATCH系エンドポイントの成功時HTTP Status Codeを200とした|②16章は「200: 取得成功」「422」「404」のみを明記し、作成・更新系の成功時コード（200/201の別）は記載がない。①未提供のため確認不可|推測|
|認証エラー401・権限エラー403・所有権エラー404・DB障害500等、②に明記のないStatus Codeの割当|②16章に明記のない一般的なHTTP実装慣行に基づく|推測|
|トランザクション実装をInfrastructure層の`TransactionManager`インターフェース方式で例示した|規約にトランザクションの具体的な実装パターンの定めがないため、一例として提示した。他の一貫した方式を実装者が採用してもよい|補足|
|Handler構成を`QuestionHandler`/`AnswerHandler`/`SubmissionHandler`の3ファイルに分割した|②はHandler構成そのものを規定していないため、エンドポイントの業務内容単位で分割した|補足|

上記以外の②記載内容（採用パターン・Bounded Context・Aggregate構成・Entity/Value Object/Repository/UseCaseの責務・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event方針・API互換方針・DB方針・テスト戦略）については、②の決定をそのまま前提とし、変更していない。
