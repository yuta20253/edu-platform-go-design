# 問題解答機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

生徒がタスクに紐づく単元の問題に回答し、回答結果を保存・更新し、提出時にタスク状態を更新する機能である。Rails現行仕様では、問題一覧取得・解答登録・解答更新・確認表示・提出という5つの操作を提供している。

## 利用者

- 学生ユーザー（student ロール）
- 自分のタスク・単元・問題に紐づく操作のみ可能

## 業務上の目的

- 生徒が学習単元に対する理解度を確認しながら回答できるようにする
- 回答履歴を蓄積し、再解答・確認表示に反映する
- 提出時にタスクの進捗状態を更新し、学習の完了状況を可視化する

---

# 2. 設計方針

この機能は、Rails実装の構造をそのまま移すのではなく、問題解答という業務に集中したGo設計とする。

主な設計思想は以下のとおりである。

- 責務分離: 入力検証・正誤判定・履歴保存・タスク状態更新を分離する
- 保守性: 解答判定や提出状態更新のルールをドメイン側に集約する
- テスト容易性: 問題の正誤判定と提出状態の遷移をユースケース単位で検証しやすくする
- 拡張性: 今後、選択式だけでなくテキスト解答や複数解答形式へ拡張しやすい構造にする
- API互換性: 既存APIの主要なエンドポイントとレスポンスの意味は維持する

---

# 3. Bounded Context

## Context名

- question-answering

## Contextの責務

- 問題の一覧取得
- 回答の登録・更新
- 正誤判定
- 回答履歴の保存
- 提出時のタスク進捗更新

## 他Contextとの依存関係

- Task Context: タスクの所有権・存在確認・状態更新に依存する
- Unit Context: 単元の存在とタスクとの紐づき確認に依存する
- Question Context: 問題・選択肢・ヒントの参照に依存する
- User Context: 認証済み生徒の識別に依存する

## 依存する理由

本機能は単に「問題を表示する」だけでなく、タスク・単元・問題の関係性が前提となるため。解答機能自体は回答・履歴・判定を中心に扱い、他情報は参照・確認のために依存する。

---

# 4. 設計パターン

## 採用パターン

Domain Model

## 判断根拠

本機能は、問題の正誤判定・回答履歴の更新・提出時の進捗状態決定という、学習成果に直結する業務ルールを含むため、業務領域としては中核に近い。単なるCRUDではなく、回答内容・正誤結果・履歴の整合性を一貫して扱う必要があるため、ドメイン概念を明確にしておく価値が高い。

このため、業務領域の分類に基づけば、回答履歴・正誤結果・状態遷移を一つのドメインとして扱うDomain Modelが自然である。特に以下の理由がある。

- 回答履歴は単なるデータではなく、学習成果と進捗の意味を持つ
- 問題・選択肢・タスク・単元の関係を踏まえ、正誤判定と履歴更新を一貫して扱う必要がある
- 提出時の状態遷移は、回答の集約結果に基づいて決定されるため、ドメインルールとして明示する価値がある

したがって、業務領域が中核であり、ルールの意味が重要であることを踏まえると、Domain Modelを採用するのが妥当である。

## 採用しなかったパターン

### Transaction Script

- 手続き型に寄りすぎるため、回答履歴・正誤判定・提出状態のルールがユースケースに偏りやすい
- ドメインルールが散在し、将来の拡張時に扱いづらくなる可能性がある
- 中核業務としての意味を持つルールを、手続き中心で表現すると保守性が低下しやすい

### Active Record寄りの設計

- 単なる永続化中心に寄りすぎると、回答履歴の意味や正誤判定ルールがモデルに散りやすい
- 中核業務としての評価ロジックと状態遷移を十分に表現しにくい
- ドメイン上の意味を明示するためには、Entityに責務を持たせる方が適切である

### Event Sourcing

- 提出や解答が他システムに通知される要件がなく、イベント履歴の再構築が必要ない
- 現状の業務では過剰な設計である

---

# 5. Aggregate設計

本機能では、Aggregateを大きく分ける必要はない。回答履歴単位を中心に扱い、タスク・単元・問題との関係は参照・検証対象として扱う。

## Aggregate Root

- QuestionHistory

## Aggregateに含めるEntity

- QuestionHistory
- AnswerResult（回答結果として扱う）

## Aggregate境界

- QuestionHistoryが回答内容・正誤・解答時刻・対象関係を一貫して扱う単位とする
- タスクや単元はAggregateの外部参照として扱い、回答履歴の整合性を保つための条件として利用する

## 整合性を保証する単位

- 解答登録・更新時に、履歴の作成・更新と正誤判定の結果を一貫して処理する

理由: 1つの操作で回答内容と判定結果の整合性を保つためである。

---

# 6. Entity設計

## QuestionHistory

- 役割: 生徒の解答履歴の中心的な概念
- ライフサイクル: 作成 → 更新 → 再判定
- 状態変化: 未回答 → 回答済み → 再解答済み
- 保持する責務:
  - 回答内容と判定結果を保持する
  - 解答時間・解説閲覧フラグなどの補助情報を保持する
  - 既存履歴がある場合の更新ルールを管理する
- 判断根拠: 回答履歴自体が本機能の主要な業務情報であるため

## AnswerResult

- 役割: 正誤判定の結果を表す概念
- ライフサイクル: 判定時に生成
- 状態変化: correct / incorrect
- 保持する責務:
  - 正誤結果を保持する
  - 回答内容との整合を管理する
- 判断根拠: 判定結果は単なるフラグではなく、業務上意味を持つため

## Task

- 役割: 提出時の状態更新対象として参照される
- 判断根拠: 本機能の中心は回答履歴であり、Taskは回答処理の外部参照として扱う

---

# 7. Value Object設計

## AnswerStatus

- 採用理由: 正誤を文字列やboolean単位で扱うより、意味を持つvalueとして明示した方が保守しやすい
- 独自ルール:
  - correct / incorrect のいずれかのみ許容する
  - 既存履歴の再判定時も同じ状態へ正規化する
- Entity属性ではなくValue Objectにする理由: 判定結果は単なる真偽ではなく、業務的に意味のある状態だから

## AnsweredAt

- 採用理由: 回答時刻は表示・比較・ソートに使われるため、型として扱うと扱いやすい
- 独自ルール:
  - 日付時刻の整合性を保証する
- Entity属性ではなくValue Objectにする理由: 時刻の意味をドメインで明示できるため

## Value Objectを採用しないもの

- 選択肢ID・問題ID・タスクID: 識別子として単純な値であり、追加ルールがないためValue Object化は不要とする

---

# 8. Domain Service

## AnswerEvaluationPolicy

- 責務: 問題の正誤判定ロジックをまとめて扱う
- Entityへ持たせない理由: 正誤判定は問題・選択肢・回答内容の組み合わせに依存し、QuestionHistory単体だけで完結しないため
- 判断根拠: 判定ルールは独立した業務ポリシーとして扱う方が明確であるため

## SubmissionProgressPolicy

- 責務: 提出時にタスク状態をどのように決定するかを判定する
- Entityへ持たせない理由: 提出時の状態決定は回答履歴の集合とタスクの状況に依存し、単一Entityに閉じ込めにくいため
- 判断根拠: 進捗判定は複数情報を横断するルールとして扱う方が自然だから

---

# 9. Repository設計

## QuestionHistoryRepository

- 管理対象: QuestionHistory
- 責務:
  - 解答履歴の保存
  - 既存履歴の取得
  - 既存履歴の更新
  - 指定問題に対する履歴取得
- 保持する検索機能:
  - user_id / task_id / unit_id / question_idによる検索
- 保持しない責務:
  - 正誤判定
  - 提出状態の決定
  - 認可判定
- 判断根拠: 永続化と検索に集中させ、業務ロジックを持たせないため

## QuestionRepository

- 管理対象: Question
- 責務:
  - 問題の存在確認
  - 問題に紐づく選択肢・ヒントの取得
- 保持する検索機能:
  - unit_id / task_idに紐づく問題取得
- 保持しない責務:
  - 回答の正誤判定そのもの
- 判断根拠: 問題情報の参照に特化させるため

## QuestionChoiceRepository

- 管理対象: QuestionChoice
- 責務:
  - 選択肢の存在確認
  - 選択肢が対象問題に属するか確認
- 保持しない責務:
  - 正誤判定の最終決定
- 判断根拠: 選択肢の参照と妥当性確認に集中させるため

## TaskRepository

- 管理対象: Task
- 責務:
  - 指定タスクの取得
  - タスク状態の更新
- 保持しない責務:
  - 提出状態の判断ロジック
- 判断根拠: 永続化と状態更新の責務に限定するため

---

# 10. UseCase設計

## ListQuestionsUseCase

- 目的: タスク・単元に紐づく問題一覧を取得する
- 入力: current user, task id, unit id
- 出力: 問題一覧と既回答状態
- トランザクション範囲: 読み取りのみ
- 呼び出すRepository:
  - QuestionRepository
  - QuestionHistoryRepository
- 判断根拠: 関連問題と履歴の照会をまとめて行うため

## CreateAnswerUseCase

- 目的: 問題への回答を登録する
- 入力: current user, task id, unit id, question id, answer payload
- 出力: 判定結果と保存結果
- トランザクション範囲: 履歴保存を1トランザクションで扱う
- 呼び出すRepository:
  - QuestionRepository
  - QuestionChoiceRepository
  - QuestionHistoryRepository
- 判断根拠: 回答内容の妥当性確認と履歴保存を一貫して行うため

## UpdateAnswerUseCase

- 目的: 既存解答を更新し、再判定する
- 入力: current user, task id, unit id, question id, answer payload
- 出力: 更新された判定結果
- トランザクション範囲: 既存履歴更新を1トランザクションで扱う
- 呼び出すRepository:
  - QuestionHistoryRepository
  - QuestionChoiceRepository
- 判断根拠: 再解答時には既存履歴を更新し、判定結果も反映するため

## ShowConfirmationUseCase

- 目的: 確認表示用の回答状況を取得する
- 入力: current user, task id, unit id
- 出力: 問題ごとの回答状況
- トランザクション範囲: 読み取りのみ
- 呼び出すRepository:
  - QuestionRepository
  - QuestionHistoryRepository
- 判断根拠: 問題と履歴を照合して確認表示に必要な状態を返すため

## SubmitTaskUseCase

- 目的: タスク提出時に進捗状態を更新する
- 入力: current user, task id
- 出力: 提出後のタスク状態
- トランザクション範囲: タスク状態更新を1トランザクションで扱う
- 呼び出すRepository:
  - TaskRepository
  - QuestionHistoryRepository
- 判断根拠: 回答履歴の集合に基づいて、タスク状態を一貫して更新するため

---

# 11. Transaction設計

## Transaction開始位置

- UseCaseの開始時にトランザクションを開始する

## Transaction終了位置

- CreateAnswerUseCase / UpdateAnswerUseCase / SubmitTaskUseCase は、保存または更新が完了した時点でコミットする
- ListQuestionsUseCase / ShowConfirmationUseCase は読み取りのみのため、トランザクションを使用しない

## 理由

- 1つの業務操作に対して、回答履歴とタスク状態の整合性を保つため
- UseCase単位で処理の境界を明確にし、テスト・保守のしやすさを確保するため

---

# 12. Validation設計

## Presentation

- 型チェック: task_id / unit_id / question_id / question_choice_id の型と必須項目を検証する
- 必須チェック: 回答登録・更新時に必要な識別子を検証する
- フォーマットチェック: answer_text / time_spent_sec / explanation_viewed の形式を検証する

## Domain

- 業務ルール: 指定された問題・選択肢・タスク・単元が正しく紐づいているかを検証する
- 状態チェック: 既存履歴がある場合に更新対象として妥当かを確認する
- 整合性チェック: 回答が対象問題の選択肢であること、提出時に回答済み状態が反映されることを確認する

## 責務分離

- Presentationは「入力が適切か」を担当する
- Domainは「業務上妥当か」を担当する
- これにより、HTTP依存の検証と業務ルールを分離できる

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、studentロールであることを確認する

## Handler

- APIエントリポイントで認証失敗時のレスポンスを整える
- 業務権限の判定は持たせない

## UseCase

- current userのタスク・単元・問題に対する所有権を前提に処理を実行する
- 取得・更新対象が自分のタスクに属するかを確認する

## Domain

- 回答履歴や提出処理に対して、所有者外の操作が行われないようにする
- ただし、認可の本体はUseCase側に寄せる

## 判断理由

認可はHTTPレベル・業務レベル・ドメインレベルで責務を分けることで、権限ルールの変更に強い構造とするためである

---

# 14. Error設計

## Domain Error

- 責務: ドメインルール違反を表現する
- 例: 選択肢が対象問題に属さない、既存履歴が存在しない、提出状態の遷移が不正である
- 判断理由: 業務ルール違反をアプリケーション層に漏らさず、ドメイン側で明示的に扱うため

## Application Error

- 責務: ユースケース実行時の失敗を表現する
- 例: 対象タスクが存在しない、対象問題が見つからない、保存処理の失敗
- 判断理由: ユースケースの失敗理由をHTTPレスポンスに変換しやすくするため

## Infrastructure Error

- 責務: DB接続失敗・永続化失敗・外部依存の不整合を表現する
- 判断理由: 永続化層の失敗をドメインに漏らさず、技術的な障害として切り分けるため

---

# 15. Domain Event

本機能では現時点でDomain Eventを採用しない。理由は、回答登録・提出に対して他処理へ通知するような非同期副作用が明示されていないためである。

将来的に、回答完了や提出完了をトリガーに学習履歴や通知を送る要件が発生した場合はイベント化を検討する。

---

# 16. API互換方針

## URL

- Rails現行仕様と同じエンドポイントを維持する
  - GET /api/v1/student/tasks/:task_id/units/:unit_id/questions
  - POST /api/v1/student/answers
  - PATCH /api/v1/student/answers
  - GET /api/v1/student/tasks/:task_id/units/:unit_id/confirmation
  - PATCH /api/v1/student/tasks/:task_id/submission

## HTTP Method

- 既存仕様どおりに維持する

## Request

- Railsのパラメータ形式は維持しつつ、Go側では入力DTOとして吸収する
- 既存のtask_id / unit_id / question_id / question_choice_idの意味は維持する

## Response

- 取得・登録・更新・提出の成功時に、Rails現行仕様と同等の意味を持つレスポンスを返す
- 失敗時は422/404のステータスを維持する

## Status Code

- 200: 取得成功
- 422: 入力・業務ルール違反
- 404: 対象データ不存在

## Error Response

- 既存のerrors構造を意識しつつ、Goの実装に合わせて整形する
- フロントエンド互換性を優先する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 現行仕様でquestion_historiesを利用しており、移行対象の業務要件を満たしているため
- 追加スキーマは現時点では不要であり、既存データとの整合性を維持する方が安全である

## 変更を提案しない理由

- 今回の移行対象は回答履歴と提出状態更新であり、業務ルールが複雑化するまではスキーマ拡張は不要である

---

# 18. テスト戦略

## Domain Test

- 目的: 正誤判定ルール、再解答時の更新ルール、提出状態の遷移を検証する

## UseCase Test

- 目的: ListQuestionsUseCase / CreateAnswerUseCase / UpdateAnswerUseCase / SubmitTaskUseCase の業務動作を検証する

## Repository Test

- 目的: QuestionHistoryRepository / QuestionRepository / QuestionChoiceRepository / TaskRepository の永続化・検索の正確性を検証する

## Handler Test

- 目的: API入力のバリデーション結果とHTTPステータスの変換を検証する

## Integration Test

- 目的: エンドポイント経由で問題一覧取得・回答登録・更新・提出が一貫して動作することを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller | Handler | HTTP入力の受け取りとレスポンス整形に限定する |
| Form Object | Request DTO + Validation | 入力検証をPresentation層で分離する |
| Service | UseCase | 業務処理の起点として扱う |
| Model | Entity + Repository + Active Record-like behavior | 業務ルールはEntity、永続化と関連付けはRepository/モデル責務として整理する |
| Serializer | Presenter / Response DTO | 画面に返すレスポンス整形を分離する |
| Judge Service | Domain Service | 正誤判定や提出状態判断をドメインポリシーとして扱う |

---

# 20. 採用しなかった設計

## Transaction Script

- 採用しなかった理由: ルールが散在しやすく、中核業務としての意味を持つ状態遷移を明示しにくいため
- 将来的に採用する可能性: 機能が極めて単純な場合には、簡略化の観点から再検討できる

## Event Sourcing

- 採用しなかった理由: 現状には履歴再構築やイベント再生の要件がないため
- 将来的に採用する可能性: 学習履歴分析や監査要件が強化された場合に有効な可能性がある

## Active Record寄りの設計

- 採用しなかった理由: モデル中心に業務ルールを寄せると、回答判定と提出状態更新が分散しやすいため
- 将来的に採用する可能性: 機能が極めて単純な場合には、簡略化の観点から再検討できる

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Domain Model | 中核業務であり、正誤判定・回答履歴・提出状態の整合性をドメインとして扱うため |
| Aggregate | QuestionHistory単位 | 回答履歴と判定結果の整合性を保つ単位として十分 |
| Transaction境界 | UseCase単位 | 1業務処理と整合性保証の単位として自然 |
| Domain Event | 未採用 | 現時点で他処理への通知要件がない |
| Value Object | 一部採用 | 正誤や回答時刻の意味を明示したいため |
| Authorization | UseCase + Middleware | 認証と業務権限を分離しやすい |

---

# 設計差分管理

## Rails現行仕様

- Form ObjectとServiceがControllerから呼ばれ、入力検証と判定処理が分散している
- モデル側に一部の業務ルールが寄りやすい

## Go設計での変更内容

- 入力検証はPresentation層に寄せる
- 正誤判定と提出状態判定はDomain Service / UseCaseに集約する
- 認可はMiddlewareとUseCaseで管理する

## 変更理由

- Railsの実装構造をそのままGoに写すと、責務が曖昧になりやすいため
- Goでは責務を明確に分けた方が、テストと保守性に優れるため

## 影響範囲

- フロントエンドから見たAPIの外部仕様は概ね維持するが、内部構造はGoらしい責務分割に変更する
- 既存DBスキーマは維持するため、データ移行やマイグレーションの追加は不要
