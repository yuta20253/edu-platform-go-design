# 分析機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒自身の学習実績（タスクの達成状況・問題の正答率・学習時間・学習の継続日数）を集計し、生徒が自分の到達度や学年内での位置づけを把握できるようにすること。
- システム上の役割
  - 他機能（タスク管理・学習ログ・問題解答）が蓄積した実績データを横断して読み取り、集計結果だけを返す。分析機能がデータを作成・更新することはない。
  - 分析は2系統に分かれる。
    - **指標単体の取得**: 分析タイプを1つ指定し、その指標だけを取得する。期間指定はできず、生徒の全期間の実績を対象とする。
    - **期間サマリの取得**: 指定期間と、その直前の同じ長さの期間を比べられる形で、複数指標をまとめて取得する。
- 利用者
  - `student` ロールのユーザー（生徒）。取得対象は常にログイン中の生徒自身の実績であり、他の生徒の実績を指定して取得する手段はない。

---

# 2. 機能一覧

|操作|概要|
|-|-|
|分析取得|分析タイプを1つ指定し、その指標を全期間の実績から集計して取得する|
|期間サマリ取得|指定期間とその直前期間を比較できる形で、継続日数・タスク達成率・学習時間・教科別正答率をまとめて取得する|

---

# 3. 業務フロー

## 分析取得

1. 生徒が分析タイプ（必要に応じてコースまたは単元の指定）を指定して分析を要求する。
2. システムは分析タイプを判定し、対応する集計処理を選ぶ。
3. 指定が空文字・未指定の場合はタスク達成状況（`task_completion`）として扱う。
4. サポートされていない分析タイプの場合はエラーとして返す。
5. 集計結果を返却する。

## 期間サマリ取得

1. 生徒が集計期間（開始日・終了日）を指定してサマリを要求する。期間の指定は省略できる。
2. システムは期間の指定内容を検証する。不正な場合はエラーとして返す。
3. 省略された場合は「今日を終了日とする7日間」を集計期間とする。
4. 指定期間の直前にある、同じ長さの期間を「比較対象期間」として決める。
5. 継続日数・タスク達成率・学習時間・教科別正答率を、指定期間と比較対象期間のそれぞれで集計する。
6. 集計結果と、実際に使われた期間の情報をあわせて返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::AnalyticsController`|`index`|GET|`/api/v1/student/analytics`|分析タイプを指定して指標を取得|
|`Api::V1::Student::Analytics::SummariesController`|`show`|GET|`/api/v1/student/analytics/summary`|期間サマリを取得|

---

# 5. API / 処理詳細

## `Api::V1::Student::AnalyticsController#index`

### 概要

分析タイプを1つ指定し、ログイン中の生徒の実績を全期間で集計して返す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`analytics[type]`|必須|string|分析タイプ。`task_completion` / `understanding_score` / `grade_average` / `course_rank` / `unit_rank`。空文字・`null` の場合は `task_completion` として扱う|
|`course_id`|任意|integer|`course_rank` のときに順位を求める対象コース|
|`unit_id`|任意|integer|`unit_rank` のときに順位を求める対象単元|

`analytics` キー自体が存在しない場合は、必須パラメータ不足として扱われる。

### 処理内容

1. 分析タイプを受け取り、対応する集計処理を選ぶ。
2. 選ばれた集計処理を実行する。
3. 集計結果をそのまま JSON として返す。

### 業務ルール

#### 共通

- 集計対象はログイン中の生徒自身の実績に限る。
- このAPIに期間指定はなく、常に生徒の全期間の実績を対象とする。
- 割合はすべて百分率（0〜100）で、小数第1位に丸めて返す。分母が0の場合は丸めを行わず `0`（整数）を返す。
- 問題の解答履歴は、論理削除済みの履歴も集計対象に含む。論理削除済みの問題・選択肢に紐づく履歴も含む。
- タスクは、論理削除済みのタスクを集計対象から除く。

#### `task_completion`（タスク達成状況）

- 生徒のタスク全件を分母、うち完了状態のタスクを分子として達成率を求める。
- 期限日が設定されていないタスクも分母に含む。

#### `understanding_score`（単元別理解度）

- 生徒の解答履歴を「教科 → コース → 単元」の3階層にまとめ、単元ごとの正答率（正答数 ÷ 解答数）をスコアとして返す。
- 解答履歴が1件も存在しない教科・コース・単元は、結果に現れない。解答履歴が全くない場合は空の集合を返す。
- 教科・コース・単元の並び順は、解答履歴を取得した順に現れた順序であり、名前順やID順に整列されるわけではない。

#### `grade_average`（学年平均との比較）

- 生徒自身の正答率とタスク達成率を、同じ学年に属する生徒集団の平均と並べて返す。
- 「同じ学年」とは学年の学年番号（年次）が一致することを指し、学校が異なる生徒も比較対象に含む。
- 比較対象集団には生徒自身も含む。退会・無効化された（論理削除済みの）ユーザーも含む。
- 学年の平均正答率は、集団全体の正答数の合計 ÷ 解答数の合計で求める。生徒ごとの正答率を平均したものではない。
- 学年の平均タスク達成率も同様に、集団全体の完了タスク数の合計 ÷ タスク数の合計で求める。
- 比較対象集団が空、または集計対象が0件の場合、平均は `0` を返す。
- ログイン中の生徒に学年が設定されていない場合、この分析タイプは処理を完了できない。

#### `course_rank` / `unit_rank`（コース内順位 / 単元内順位）

- 指定されたコース（または単元）に解答履歴を持つ生徒を母集団とし、生徒ごとの正答率（正答数 ÷ 解答数）が高い順に順位を付ける。
- 母集団は在籍中（論理削除されていない）のユーザーに限る。
- 返すのは自分の順位と母集団の人数のみで、他の生徒の情報は返さない。
- 自分が当該コース・単元に解答履歴を持たない場合、順位は `null` となり、人数だけが返る。
- 対象のコース・単元が指定されていない場合、母集団は0人となり、順位は `null`、人数は `0` となる。
- 正答率が同じ生徒が複数いる場合の順位の決め方は、同率を同順位として扱う仕組みにはなっていない。並べた結果の位置がそのまま順位になる。
- この順位は、`understanding_score` や `grade_average` とは独立した集計であり、それらの結果を合成した指標ではない。

### Database変更

- なし

### Response

分析タイプごとに構造が異なる。HTTP Status は `200`。

#### `task_completion`

|項目|型|説明|
|-|-|-|
|`completed_count`|integer|完了タスク数|
|`total_count`|integer|タスク総数|
|`completion_rate`|number|達成率（%）|

#### `understanding_score`

|項目|型|説明|
|-|-|-|
|`subjects`|array|教科の配列|
|`subjects[].subject_name`|string|教科名|
|`subjects[].courses`|array|コースの配列|
|`subjects[].courses[].level_name`|string|コース名（レベル名）|
|`subjects[].courses[].level_number`|integer|レベル番号|
|`subjects[].courses[].units`|array|単元の配列|
|`subjects[].courses[].units[].unit_name`|string|単元名|
|`subjects[].courses[].units[].score`|number|単元の正答率（%）|

#### `grade_average`

|項目|型|説明|
|-|-|-|
|`correct_rate.my`|number|自分の正答率（%）|
|`correct_rate.average`|number|学年の平均正答率（%）|
|`task_completion_rate.my`|number|自分のタスク達成率（%）|
|`task_completion_rate.average`|number|学年の平均タスク達成率（%）|

#### `course_rank` / `unit_rank`

|項目|型|説明|
|-|-|-|
|`rank`|integer / null|自分の順位。解答履歴がない場合は `null`|
|`total_users`|integer|母集団の人数|

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|サポートされていない分析タイプを指定|400|`{"errors": ["指定された分析タイプ（{指定値}）は存在しません。"]}`|
|`analytics` パラメータ自体が存在しない|400|必須パラメータ不足として扱われ、個別のエラー本文は定義されていない|
|`grade_average` を指定し、生徒に学年が設定されていない|500|集計を完了できない|
|未認証|401|認証されていない|
|`student` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

---

## `Api::V1::Student::Analytics::SummariesController#show`

### 概要

指定期間と、その直前の同じ長さの期間を比較できる形で、学習状況のサマリを返す。

### Request

|項目|必須|型|説明|
|-|-|-|-|
|`from`|任意|string|集計開始日。`YYYY-MM-DD` 形式。省略時は「終了日の6日前」|
|`to`|任意|string|集計終了日。`YYYY-MM-DD` 形式。省略時は「今日」|

### 処理内容

1. 期間指定を検証する。不正な場合はエラーを返す。
2. 省略された項目を既定値で補い、集計期間を確定する。
3. 集計期間の直前にある、同じ長さの期間を比較対象期間として決める。
4. 継続日数・タスク達成率・学習時間・教科別正答率を集計する。
5. 集計結果と、実際に使われた期間情報をあわせて返す。

### 業務ルール

#### 期間の扱い

- `from`・`to` はどちらも集計範囲に含む（両端を含む）。
- 既定値は「今日を終了日とする7日間」。`to` 省略時は今日、`from` 省略時は終了日の6日前となる。
- `from` のみ指定した場合は `to` が今日、`to` のみ指定した場合は `from` が「今日の6日前」となるため、指定した期間と実際の集計期間が一致しない組み合わせが生じうる。
- 期間の日数（両端含む）は最大366日。
- 比較対象期間は「集計開始日の前日」を終了日とし、集計期間と同じ日数をさかのぼった期間とする。
- 日付の判定は日本時間で行う。学習ログ・解答履歴の日時はUTCで保存されているため、集計時に日本時間の1日の範囲（当日の0時00分00秒〜23時59分59秒）へ変換して絞り込む。
- 応答に含まれる日付はすべて `YYYY-MM-DD` 形式の文字列で返す。

#### 継続日数（`streak_days`）

- 学習ログの開始日時（日本時間の日付）を学習した日とみなし、連続して学習した日数を数える。
- 起点は「今日」。今日の学習ログがなければ「昨日」を起点とする。どちらにもなければ `0`。
- 起点から1日ずつ過去にさかのぼり、学習ログのある日が途切れるまで数える。
- **指定期間の影響を受けない。** 期間を指定しても、継続日数は常に今日を基準に算出される。
- 論理削除済みの学習ログは数えない。
- 1日に複数の学習ログがあっても1日として数える。

#### タスク達成率（`task_completion`）

- 集計期間・比較対象期間のそれぞれについて、期限日が期間内にあるタスクを分母、うち完了状態のタスクを分子として達成率を求める。
- 期限日が設定されていないタスクは、期間指定がある以上どちらの期間にも含まれない。
- 期間内に対象タスクが存在しない場合、件数は `0`、達成率は `0` となる。

#### 学習時間（`study_minutes`）

- 学習ログの学習時間（分）を、開始日時の日本時間の日付ごとに合計する。
- 論理削除済みの学習ログは集計しない。学習時間が未記録の学習ログは0分として扱う。
- 集計期間の日数が31日以内のときは日単位（`unit` は `day`）、32日以上のときは週単位（`unit` は `week`）で推移を返す。
- 週単位の場合は、集計開始日から7日ずつ区切って合計する。各区切りの代表日は、その区切りの先頭日とする。期間の日数が7の倍数でない場合、最後の区切りは7日未満となる。
- 推移には学習ログのない日も `0` 分として含める。
- 合計値は集計期間・比較対象期間の両方を返すが、推移（`series`）と単位（`unit`）は集計期間のものだけを返す。

#### 教科別正答率（`subject_progress`）

- 解答日時が期間内にある解答履歴を教科ごとにまとめ、教科ごとの正答率（正答数 ÷ 解答数）を求める。
- 並び順は教科のID昇順。
- 集計期間に解答履歴のない教科は結果に現れない。
- 比較対象期間の正答率は、同じ教科の集計期間の結果に対して付与する。比較対象期間に当該教科の解答履歴がない場合、比較値は `null` となる。
- 集計期間に解答履歴が1件もない場合は空の配列を返す。

### Database変更

- なし

### Response

HTTP Status は `200`。

|項目|型|説明|
|-|-|-|
|`streak_days`|integer|連続学習日数|
|`task_completion.current`|object|集計期間のタスク達成状況（`completed_count` / `total_count` / `completion_rate`）|
|`task_completion.previous`|object|比較対象期間のタスク達成状況（同構造）|
|`study_minutes.unit`|string|推移の単位。`day` または `week`|
|`study_minutes.series`|array|集計期間の学習時間推移|
|`study_minutes.series[].date`|string|代表日（`YYYY-MM-DD`）|
|`study_minutes.series[].minutes`|integer|学習時間（分）|
|`study_minutes.total.current`|integer|集計期間の学習時間合計（分）|
|`study_minutes.total.previous`|integer|比較対象期間の学習時間合計（分）|
|`subject_progress`|array|教科別正答率の配列|
|`subject_progress[].subject_name`|string|教科名|
|`subject_progress[].correct_rate.current`|number|集計期間の正答率（%）|
|`subject_progress[].correct_rate.previous`|number / null|比較対象期間の正答率（%）。履歴がない場合は `null`|
|`meta.from`|string|実際に使われた集計開始日|
|`meta.to`|string|実際に使われた集計終了日|
|`meta.previous_from`|string|比較対象期間の開始日|
|`meta.previous_to`|string|比較対象期間の終了日|

### Errorケース

エラー本文はいずれも `{"errors": [メッセージ, ...]}` の形式で、複数のメッセージが同時に含まれうる。

|条件|HTTP Status|内容|
|-|-|-|
|`from` が `YYYY-MM-DD` 形式でない、または存在しない日付|422|`From は正しい日付を入力してください`|
|`to` が `YYYY-MM-DD` 形式でない、または存在しない日付|422|`To は正しい日付を入力してください`|
|`from` が `to` より後|422|`To はfrom以降の日付を指定してください`|
|期間が366日を超える|422|`取得期間は366日以内で指定してください`|
|未認証|401|認証されていない|
|`student` ロール以外|403|`{"errors": ["この操作を行う権限がありません"]}`|

メッセージ先頭の `From` / `To` は、項目名の日本語訳が定義されていないため、項目名がそのまま英語表記で出力されている。

---

# 6. データモデル

分析機能が所有するテーブルはない。以下はすべて他機能が所有するテーブルの参照である。

|テーブル|役割|分析で使う主なカラム|
|-|-|-|
|`tasks`|学習タスク。タスク管理機能が所有|`user_id`, `status`, `due_date`, `deleted_at`|
|`study_logs`|学習ログ。学習ログ機能が所有|`user_id`, `started_at`, `duration_minutes`, `deleted_at`|
|`question_histories`|問題の解答履歴。問題解答機能が所有|`user_id`, `course_id`, `unit_id`, `is_correct`, `answered_at`, `deleted_at`|
|`units`|単元。教材が所有|`id`, `course_id`, `unit_name`|
|`courses`|コース。教材が所有|`id`, `subject_id`, `level_name`, `level_number`|
|`subjects`|教科。教材が所有|`id`, `name`|
|`users`|ユーザー|`id`, `grade_id`, `deleted_at`|
|`grades`|学年。学校ごとに年次を持つ|`id`, `high_school_id`, `year`|

リレーション:

- `question_histories` は `users`・`courses`・`units` に属する。`course_id`・`unit_id` を履歴自身が保持しているため、単元からコースをたどらずに集計できる。
- `subjects` → `courses` → `units` の階層が、`understanding_score` の3階層集計の構造にそのまま対応する。
- `users` は1つの `grades` に属する。`grades` は学校ごとに独立しており、年次（`year`）は学校をまたいで同じ値を取りうる。

参照する側の論理削除の扱い:

- `tasks` は論理削除済みを常に除外して参照される。
- `study_logs` は論理削除済みを除外して参照される。
- `question_histories` は論理削除済みも含めて参照される。
- `users`（学年平均の母集団）は論理削除済みも含む。`users`（順位の母集団）は論理削除済みを除外する。

---

# 7. 状態管理

- 分析機能自身は状態を持たない。
- 集計で参照する状態は、タスクの進行状態（`not_started` / `in_progress` / `completed`）のみ。`completed` を「完了」として扱い、それ以外を未完了として扱う。

---

# 8. 権限制御

- 利用可能ユーザー: `student` ロールのユーザーのみ。
- 未認証のアクセスは `401`。`student` 以外のロールは `403`。
- 閲覧制限: 集計対象は常にログイン中の生徒自身の実績に限られる。他の生徒を指定する手段はない。
- 順位・学年平均では他の生徒の実績を集計に用いるが、返すのは自分の順位・人数・平均値のみで、他の生徒を特定できる情報は返さない。

---

# 9. 非同期処理

- なし。集計はすべてリクエスト時に同期で実行される。
- 集計結果のキャッシュ・事前集計テーブルも使用していない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::AnalyticsController`, `Api::V1::Student::Analytics::SummariesController`|
|Form|`Student::AnalyticsPeriodForm`|
|Form Concern|`DateRangeValidatable`|
|Service|`Student::AnalyticsService`|
|Service（指標単体）|`Student::Analytics::TaskCompletion`, `Student::Analytics::UnderstandingScore`, `Student::Analytics::GradeAverage`, `Student::Analytics::Rank`|
|Service（期間サマリ）|`Student::Analytics::Summary`, `Student::Analytics::Streak`, `Student::Analytics::StudyMinutes`, `Student::Analytics::SubjectProgress`|
|Service（補助）|`Student::Analytics::TaskStats`, `Student::Analytics::Calculator`|
|Service Concern|`AnalyticsPeriodable`|
|Error|`Student::InvalidAnalyticsTypeError`|
|Model|`Task`, `StudyLog`, `QuestionHistory`, `Unit`, `Course`, `Subject`, `User`, `Grade`|
|Serializer|なし（集計結果の Hash をそのまま JSON 化）|
