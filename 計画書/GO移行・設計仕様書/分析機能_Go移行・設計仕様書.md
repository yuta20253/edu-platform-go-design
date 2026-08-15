# 分析機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

生徒の学習進捗・成績を分析し、指定された分析タイプに応じた結果を返却する機能である。Rails現行仕様では、`analytics[type]`パラメータによって`task_completion`・`understanding_score`・`grade_average`・`course_rank`・`unit_rank`のいずれかの分析処理を呼び分け、対応する分析結果をJSONで返却する。

## 利用者

- 生徒ユーザー（student ロール）
- 自分自身の学習データに基づく分析結果のみ取得可能

## 業務上の目的

- 生徒が自身の学習進捗（タスク完了率・理解度・成績平均）を把握できるようにする
- コース・単元単位での相対的な順位（ランク）を把握できるようにする

---

# 2. 設計方針

本機能は分析タイプごとに異なる集計ロジックを持つ参照機能である。Rails現行仕様では単一のServiceクラスが型に応じて内部処理を分岐しているが、Go設計では分析タイプごとに責務を独立させ、新しい分析タイプの追加が既存ロジックに影響しない構造とする。

- 責務分離: 分析タイプごとの集計ロジックを独立させ、型判定と集計処理を混在させない
- 保守性: 分析タイプの追加・変更が既存の分析ロジックに影響しない構造とする
- テスト容易性: 分析タイプごとに独立してテストできるようにする
- API互換性: 既存エンドポイント・リクエストパラメータ（`analytics[type]`, `course_id`, `unit_id`）を維持する
- 拡張性: 将来的な分析タイプの追加を、既存コードの変更なしに追加できるようにする

---

# 3. Bounded Context

## Context名

- learning-analytics（学習分析）

## Contextの責務

- タスク完了率・理解度スコア・成績平均・コース内順位・単元内順位の算出
- 分析タイプに応じた集計結果の提供

## 他Contextとの依存関係

- task-management: タスク完了状況の参照に依存する
- Grade/Assessment Context: 理解度スコア・成績データの参照に依存する（推測: Rails現行仕様書に理解度スコア・成績の算出元テーブルの詳細記載がないため、既存の学習履歴・成績関連テーブルを参照する前提とする）
- curriculum: コース・単元単位でのランク算出範囲の特定に依存する

## 依存する理由

学習分析Contextは、タスク・成績・理解度といった学習データそのものを所有するのではなく、他Contextが管理するデータを集計・加工して分析結果を導出する性質の機能である。分析結果を生成するためには、複数Contextのデータを横断的に参照する必要がある。

---

# 4. 設計パターン

## 採用パターン

Transaction Script

## 判断根拠

本機能は5種類の分析タイプそれぞれについて、既存データ（タスク・成績・理解度・順位）を読み取り、集計した結果を返却するのみであり、状態を保持したり状態遷移を管理したりする業務ルールは存在しない。各分析タイプは独立した集計手続きとして表現でき、複数の分析タイプが1つのEntityの振る舞いとして共通化されるべき理由もない。

- 業務ルール: 分析タイプごとに集計方法が異なるが、いずれも「既存データの読み取り→集計→結果返却」という手続き型の処理であり、Entityが状態を持って振る舞う必要がない
- 状態管理: 本機能側では分析対象データを変更しない（読み取り専用）
- 将来拡張: 分析タイプが増えても、それぞれ独立したUseCase（Transaction Script）として追加でき、既存の分析タイプに影響を与えない。これはRails現行仕様のような単一Serviceクラスへの分岐追加よりも変更影響が小さい
- テスト容易性: 分析タイプごとに入力（current user, course_id, unit_id）と出力（集計結果）を独立して検証できる

## 採用しなかったパターン

### Active Record

分析結果はいずれも派生的な集計値であり、永続化される対象ではない。モデルに保存・更新責務を持たせる意味がない。

### Domain Model

分析対象（タスク完了率・理解度・成績・順位）はいずれも他Contextが管理するデータの読み取り結果であり、本Context固有のEntityが状態を持って振る舞うことはない。集計ロジックは手続き的な処理であり、Entityへ振る舞いを集約するメリットが小さい。

### Event Sourcing

分析結果の履歴管理・再構築要件は現行仕様に存在しない。

---

# 5. Aggregate設計

不要と判断する。

理由: 本機能は複数Contextのデータを読み取り専用で集計するのみであり、書き込みの整合性を保証する境界が存在しない。分析タイプごとの集計結果はAggregateではなく、UseCaseの出力値（読み取り専用の集計結果）として扱う。

---

# 6. Entity設計

本機能では永続化対象となる独自のEntityを新規に定義しない。各分析タイプが参照するデータは、それぞれの所有Context（task-management、Grade/Assessment Context、curriculum）のEntityを読み取り専用で参照する。

## AnalyticsResult（分析結果、読み取り専用の集計値）

- 役割: 分析タイプごとの集計結果を表す
- ライフサイクル: UseCase実行のたびに生成され、永続化されない
- 状態変化: なし
- 保持する責務: 分析タイプに応じた集計値（完了率・スコア・平均値・順位など）の保持
- 判断根拠: 分析結果は保存されるドメイン概念ではなく、都度算出される出力値であるため、Entityというよりも集計結果を表すデータのまとまりとして扱う

---

# 7. Value Object設計

## AnalyticsType

- 採用理由: `task_completion`・`understanding_score`・`grade_average`・`course_rank`・`unit_rank`という許容値を型として明示し、不正な分析タイプの指定を早期に検出するため
- 独自ルール: 上記5種類のいずれかのみを許容する。不正な値は許容しない
- Entity属性ではなくValue Objectにする理由: 分析タイプはリクエストパラメータからUseCase解決までの間で参照される値であり、Entityの属性としてではなく、入力値の妥当性・分岐条件を表す独立した概念として扱う方が適切であるため

## CompletionRate / UnderstandingScore（比率・スコア値）

- 採用理由: 完了率・理解度スコアはいずれも0〜100（または0〜1）の範囲を持つ値であり、単純な数値型のまま扱うと範囲外の値が混入するリスクがある
- 独自ルール: 有効範囲内の値のみを許容する
- Entity属性ではなくValue Objectにする理由: 集計結果としての意味（比率・スコアという業務的な意味）を型で明示し、表示フォーマットの統一を図るため

## Value Objectを採用しないもの

- course_id・unit_id: 単なる識別子であり、独自の業務ルールを持たないためValue Object化は不要とする

---

# 8. Domain Service

## RankCalculationService

- 責務: コース単位・単元単位での相対順位（course_rank, unit_rank）を算出する
- Entityへ持たせない理由: 順位算出は特定のEntity単体の属性ではなく、対象生徒と他の生徒群の成績・理解度データを横断的に比較する処理であり、単一Entityの振る舞いとして表現できないため
- 判断根拠: course_rankとunit_rankは対象範囲（コース単位か単元単位か）が異なるだけで、順位算出のロジック自体は共通化できるため、UseCase間で重複させずDomain Serviceとして共通化する

## 追加で必要としないService

- task_completion・understanding_score・grade_averageはそれぞれ単一の集計処理（合計・平均の算出）であり、複数Entityをまたぐ判定ロジックを含まないため、個別のDomain Serviceを設けず、UseCase内の手続きとして扱う

---

# 9. Repository設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、Repository Interfaceをdomain層に定義しない。以下は永続化・検索責務の設計意図であり、実装時はinfrastructure層の関数として直接実装する(規約: アーキテクチャ規約.md「4. 設計パターンごとの構造適用方針」)。

## TaskCompletionRepository

- 管理対象: タスク完了状況（task-managementのデータを参照）
- 責務: 生徒のタスク完了率算出に必要な集計データの取得
- 保持する検索機能: user_idによる絞り込み、完了・未完了タスク数の集計
- 保持しない責務: タスクの作成・更新
- 判断根拠: タスクの永続化責務はtask-managementにあり、本Contextでは集計に必要な読み取りのみを担当するため

## UnderstandingScoreRepository

- 管理対象: 理解度スコアデータ（推測: Grade/Assessment Contextのデータを参照）
- 責務: 生徒の理解度スコア集計データの取得
- 保持する検索機能: user_idによる絞り込み、course_id/unit_idによる範囲絞り込み
- 保持しない責務: 理解度スコアの作成・更新
- 判断根拠: 理解度データの生成元は別Contextの責務であり、本Contextでは集計のための参照に徹するため

## GradeRepository

- 管理対象: 成績データ（推測: Grade/Assessment Contextのデータを参照）
- 責務: 生徒の成績平均算出に必要な集計データの取得
- 保持する検索機能: user_idによる絞り込み、期間・科目等による絞り込み（推測: 現行仕様書に詳細な絞り込み条件の記載がないため、既存の成績データ構造に準じる）
- 保持しない責務: 成績の作成・更新
- 判断根拠: 成績データの生成元は別Contextの責務であり、本Contextでは集計のための参照に徹するため

## RankRepository

- 管理対象: 順位算出に必要な集計データ（コース・単元単位の成績/理解度データ）
- 責務: 指定コース・単元における対象生徒の相対順位算出に必要なデータの取得
- 保持する検索機能: course_id/unit_idによる範囲指定、対象生徒とその他生徒群の比較対象データ取得
- 保持しない責務: 順位算出ロジックそのもの（RankCalculationServiceが担当）、他生徒の個人情報の外部公開
- 判断根拠: 順位算出には他生徒の集計データとの比較が必要だが、Repositoryはデータ取得のみに徹し、算出ロジックはDomain Serviceに委ねるため

---

# 10. UseCase設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、UseCase層(struct)を設けない。以下は業務操作の設計意図であり、実装時はapplication層の関数として直接実装する。

## GetTaskCompletionAnalytics

- 目的: 生徒のタスク完了率を取得する
- 入力: current user, course_id（任意）, unit_id（任意）
- 出力: タスク完了率の分析結果
- トランザクション範囲: 読み取りのみ、トランザクションは不要
- 呼び出す関数: タスク完了率集計関数(TaskCompletionRepositoryの設計意図をinfrastructure層の関数として実装)
- 判断根拠: 単一の集計処理であり、他の分析タイプと処理を共有する必要がないため

## GetUnderstandingScoreAnalytics

- 目的: 生徒の理解度スコアを取得する
- 入力: current user, course_id（任意）, unit_id（任意）
- 出力: 理解度スコアの分析結果
- トランザクション範囲: 読み取りのみ
- 呼び出す関数: 理解度スコア集計関数(UnderstandingScoreRepositoryの設計意図をinfrastructure層の関数として実装)
- 判断根拠: 単一の集計処理であり、独立した関数として扱うことでテスト・保守が容易になるため

## GetGradeAverageAnalytics

- 目的: 生徒の成績平均を取得する
- 入力: current user, course_id（任意）, unit_id（任意）
- 出力: 成績平均の分析結果
- トランザクション範囲: 読み取りのみ
- 呼び出す関数: 成績平均集計関数(GradeRepositoryの設計意図をinfrastructure層の関数として実装)
- 判断根拠: 単一の集計処理であり、独立した関数として扱うことでテスト・保守が容易になるため

## GetCourseRankAnalytics

- 目的: 指定コース内での生徒の相対順位を取得する
- 入力: current user, course_id（必須）
- 出力: コース内順位の分析結果
- トランザクション範囲: 読み取りのみ
- 呼び出す関数: 順位算出用データ取得関数（RankRepositoryの設計意図をinfrastructure層の関数として実装。RankCalculationServiceを利用）
- 判断根拠: 順位算出ロジックをunit_rankと共有するため、RankCalculationServiceを介して処理する

## GetUnitRankAnalytics

- 目的: 指定単元内での生徒の相対順位を取得する
- 入力: current user, unit_id（必須）
- 出力: 単元内順位の分析結果
- トランザクション範囲: 読み取りのみ
- 呼び出す関数: 順位算出用データ取得関数（RankRepositoryの設計意図をinfrastructure層の関数として実装。RankCalculationServiceを利用）
- 判断根拠: 順位算出ロジックをcourse_rankと共有するため、RankCalculationServiceを介して処理する

---

# 11. Transaction設計

## Transaction開始位置

- なし

## Transaction終了位置

- なし

## 理由

すべてのUseCaseが読み取り専用の集計処理であり、状態を変更しないため、トランザクションによる整合性保証は不要である。

---

# 12. Validation設計

## Presentation

- 型チェック: `analytics[type]`が文字列であること、course_id/unit_idが整数であることを検証する
- 必須チェック: `analytics[type]`が必須項目であることを検証する
- フォーマットチェック: AnalyticsTypeの許容値（task_completion / understanding_score / grade_average / course_rank / unit_rank）に合致するかを検証する

## Domain

- 業務ルール: course_rank/unit_rankの場合にcourse_id/unit_idが指定されているかの整合性チェック
- 状態チェック: 特になし（読み取り専用のため）
- 整合性チェック: 指定されたcourse_id/unit_idが実在するかの確認

## 責務分離

- Presentationは「typeが許容値かどうか」「必須パラメータが揃っているか」という入力形式の妥当性を担当する
- Domainは「course_rank/unit_rank算出に必要な範囲指定が業務的に成立するか」を担当する
- これにより、Rails現行仕様で発生していた「サービス内部でのInvalidAnalyticsTypeError」のような、処理途中での例外発生を避け、Presentation層での早期検証によって問題を切り分けられるようにする

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、コンテキストへ保持する
- 役割がstudentであることを確認する

## Handler

- リクエストパラメータの受け取りとAnalyticsTypeに応じたapplication関数の呼び分けを行う
- 具体的な業務権限判定は持たせない

## UseCase

- current userのuser_idをすべての検索関数の条件に付与し、自分自身の学習データのみを対象とする
- course_rank/unit_rankにおいても、他生徒の個人情報を直接返却せず、対象生徒自身の順位情報のみを出力する

## Domain

- RankCalculationServiceは、他生徒の集計データを比較には利用するが、個人を特定できる情報を結果に含めないというルールを持つ

## 判断理由

分析機能は他生徒のデータと比較する処理を含むため、認可を「自分のデータのみ取得可能」という単純なスコープ制御だけでなく、「他生徒の情報を結果に漏らさない」という業務ルールとしてRankCalculationService・該当するapplication関数の双方に明示する必要がある。

---

# 14. Error設計

## Domain Error

責務: 順位算出時の範囲指定不整合など、業務ルール違反を表現する

## Application Error

責務: 指定されたcourse_id/unit_idが存在しない場合や、集計処理自体の失敗を表現する

## Infrastructure Error

責務: DB接続失敗など、永続化層の技術的な障害を表現する

判断理由: Rails現行仕様では`InvalidAnalyticsTypeError`が処理の途中で例外として発生する構造になっているが、Go設計ではAnalyticsTypeの妥当性をPresentation層で先に検証することで、型不正はDomain Error/Application Errorに到達する前に排除する。これにより、例外駆動の制御フローに依存しない設計とする。

---

# 15. Domain Event

不要と判断する。

理由: 本機能は参照のみであり、分析結果の取得に伴って他処理へ通知すべき副作用が現行仕様に存在しないため。

---

# 16. API互換方針

## URL

- GET /api/v1/student/analytics

現行仕様のエンドポイントを維持する。

## HTTP Method

- 既存仕様どおりGETのみを維持する

## Request

- `analytics[type]`（必須）, `course_id`（任意）, `unit_id`（任意）を維持する

## Response

- 分析タイプごとの結果ペイロードは、既存仕様に近い構造を維持する

## Status Code

- 200: 取得成功
- 400: 分析タイプが不正、または必須パラメータ不足（Rails現行仕様では例外発生だが、Go設計ではPresentation層での検証により明示的な400として扱う）
- 404: 指定course_id/unit_idが存在しない

## Error Response

- 既存のエラーレスポンス形式を踏襲しつつ、分析タイプ不正時のエラーメッセージを明確化する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 現行の学習履歴・タスク・成績関連テーブルを参照することで本機能の要件を満たせるため（推測: 各分析タイプの算出元テーブルの詳細はRails現行仕様書に明記されていないため、既存モデル構造に基づき集計可能であることを前提とする）

## 変更を提案しない理由

- 本機能は既存データの集計・参照のみであり、現行スキーマに構造的な問題は現行仕様書から確認できないため

---

# 18. テスト戦略

## Domain Test

- 目的: AnalyticsType・CompletionRate・UnderstandingScoreの妥当性判定、RankCalculationServiceの順位算出ロジックを検証する

## UseCase Test

- 目的: 分析タイプごとのUseCaseが正しい集計結果を返すこと、自分自身のデータのみを対象とすることを検証する

## Repository Test

- 目的: 各Repositoryの検索条件（user_idスコープ・course_id/unit_id絞り込み）の正確性を検証する

## Handler Test

- 目的: `analytics[type]`のバリデーション結果とHTTPステータスへの変換、UseCase解決の正しさを検証する

## Integration Test

- 目的: エンドポイント経由で各分析タイプの結果が正しく取得できること、不正なtype指定時に適切なエラーが返ることを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller | Handler | HTTP入力の受け取りとtypeに応じたapplication関数の呼び分け、レスポンス整形に限定する |
| Service（Student::AnalyticsService） | 分析タイプごとの独立したapplication関数群 | 単一Serviceでの型分岐をやめ、タイプごとに独立した処理単位とする(Transaction Script採用のためusecase層は設けない) |
| Analysis（TaskCompletion, UnderstandingScore, GradeAverage, Rank） | application関数（+ RankCalculationServiceなどのDomain Service） | 分析ロジックを関数単位に分離し、共通化が必要な順位算出のみDomain Serviceとして抽出する |

---

# 20. 採用しなかった設計

## Domain Model

- 採用しなかった理由: 分析対象データはいずれも他Contextが管理する情報の集計結果であり、本Context固有のEntityが状態を持って振る舞う必要がないため
- 将来的に採用する可能性: 分析結果自体に対する評価・コメント機能など、状態を持つ機能が追加された場合は再検討する

## Active Record

- 採用しなかった理由: 分析結果は永続化されない派生値であり、モデルに保存・更新責務を持たせる意義がないため
- 将来的に採用する可能性: 分析結果をスナップショットとして保存する要件が発生した場合に再検討する

## Event Sourcing

- 採用しなかった理由: 分析結果の履歴管理・再構築要件が現行仕様に存在しないため
- 将来的に採用する可能性: 分析結果の推移を時系列で追跡する要件が発生した場合に検討する

## Rails現行仕様のような単一Serviceによる型分岐

- 採用しなかった理由: 単一のServiceクラスに全分析タイプの分岐ロジックを集約すると、新しい分析タイプの追加のたびに既存クラスを変更するリスクが高まり、既存の分析タイプへの影響が発生しやすいため
- 将来的に採用する可能性: 分析タイプ数が極めて少なく、共通処理が大部分を占めるようになった場合は、統合したUseCase構造への再検討もあり得る

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Transaction Script | 各分析タイプが状態を持たない読み取り専用の集計処理であるため |
| Aggregate | 未採用 | 他Contextデータの読み取り専用集計であり、整合性境界が不要なため |
| Transaction境界 | 未使用 | 状態変更を伴わないため |
| Domain Event | 未採用 | 他処理への通知要件がない |
| Value Object | 一部採用 | 分析タイプ・比率・スコアの許容値と意味を型として明示したいため |
| Domain Service | RankCalculationServiceのみ採用 | course_rank/unit_rankで共通する順位算出ロジックを重複させないため |
| 処理構成 | 分析タイプごとに独立したapplication関数 | 型分岐を単一Serviceに集約せず、変更影響を局所化するため |

---

# 設計差分管理

## Rails現行仕様

- `Student::AnalyticsService`が`type`パラメータに応じて内部で処理を分岐し、対応する分析クラス（TaskCompletion, UnderstandingScore, GradeAverage, Rank）を呼び出している
- 不正な`type`が指定された場合、`InvalidAnalyticsTypeError`が処理の途中で例外として発生する可能性がある

## Go設計での変更内容

- 分析タイプごとに独立したUseCaseを用意し、Presentation層でtypeに応じたUseCase解決を行う
- AnalyticsTypeの妥当性検証をPresentation層で先に行い、不正な値は業務処理に到達する前に400として扱う
- course_rank/unit_rankで共通する順位算出ロジックをRankCalculationServiceとして抽出する

## 変更理由

- 単一Serviceに型分岐ロジックを集約する設計は、分析タイプの追加のたびに既存クラスへの変更が必要となり、変更影響が広がりやすい。タイプごとに独立したUseCaseとすることで、新規分析タイプの追加を既存コードの変更なしに行えるようにする
- 例外駆動でのエラー検出ではなく、Presentation層での事前検証によって不正な入力を早期に検出し、エラーハンドリングを一貫させる

## 影響範囲

- フロントエンドから見たAPIの外部仕様（エンドポイント・リクエストパラメータ）は変更しない
- 不正なtype指定時のレスポンス（HTTPステータス・エラーメッセージ）が、例外由来の挙動から明示的な400エラーへ変わる可能性があるため、フロントエンド側のエラーハンドリングとの整合を確認する必要がある
- 既存DBスキーマは維持するため、データ移行は不要

