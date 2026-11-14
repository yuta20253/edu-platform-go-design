# 管理者分析機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者が、生徒の学習状況をKPI・日別推移・正答率ワーストランキング・高校別利用状況・コンテンツ網羅率の観点で把握できるようにすること。
- システム上の役割
  - 期間・高校・教科で絞り込んだ上で、解答履歴・学習ログ・単元・問題・コースのデータを集計して返す。
  - 本機能はデータを所有せず、他機能が管理するデータを読み取り専用で集約する(管理者ダッシュボード機能と同じ位置づけ)。
- 利用者
  - `admin` ロールのユーザー(管理者)

---

# 2. 機能一覧

|操作|概要|
|-|-|
|表示|指定した期間・絞り込み条件で、KPI(現在の期間と直前の同じ長さの期間の比較)、日別推移、正答率が低い単元・設問のランキング、高校別の利用状況、コンテンツ網羅率を取得する|

---

# 3. 業務フロー

1. 管理者が分析画面を開く、または期間・高校・教科の絞り込みを変更する。
2. システムは期間(`from`/`to`)を検証する。未指定の場合は当日を含む直近30日間を既定の期間とする。
3. 指定した高校・教科で対象データを絞り込む。対象となる生徒は、招待受諾済みかつ有効(論理削除されていない)な生徒に限る。
4. 指定した期間のKPI(アクティブな生徒数・解答数・正答率・学習時間)を、現在の期間と、同じ日数だけ直前に遡った期間の両方で集計する。
5. 指定した期間の日別の活動状況を、記録が無い日も0件として全日分集計する。
6. 指定した期間で解答件数が一定数以上ある単元・設問のうち、正答率が低いものを上位10件抽出する。
7. 生徒が1人以上在籍する高校ごとの利用状況(在籍数・アクティブな生徒数・アクティブ率・解答数・正答率)を集計する。
8. 単元・問題の在庫状況(問題が無い単元数、問題はあるが指定期間内に解答が無い単元数)を集計する。
9. 集計結果と集計条件(期間・閾値など)をあわせて返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::AnalyticsController`|show|GET|`/api/v1/admin/analytics`|KPI・日別推移・正答率ワーストランキング・高校別利用状況・コンテンツ網羅率を取得する|

---

# 5. API / 処理詳細

## Admin::AnalyticsController#show

### 概要

指定した期間・絞り込み条件(高校・教科)で、管理者向けの学習状況の集計結果を取得する。

---

### Request

|項目|必須|型|説明|
|-|-|-|-|
|from|任意|string(`YYYY-MM-DD`)|集計期間の開始日。未指定時は`to`から29日前|
|to|任意|string(`YYYY-MM-DD`)|集計期間の終了日。未指定時は当日|
|high_school_id|任意|integer|指定した高校に在籍する生徒のみに絞り込む|
|subject_id|任意|integer|指定した教科に属するコース・単元のみに絞り込む(コンテンツ網羅率を除く)|

---

### 処理内容

1. `from`・`to`・`high_school_id`・`subject_id`を検証する。
2. 検証済みの期間・絞り込み条件で、KPI・日別推移・ワーストランキング・高校別利用状況・コンテンツ網羅率をそれぞれ集計する。
3. 集計結果と集計条件をレスポンスとして返す。

---

### 業務ルール

- 集計対象の生徒は「招待受諾済みかつ有効(論理削除されていない)」な生徒に限る。招待中(パスワード未設定)の生徒・論理削除された生徒は対象外。
- 期間は開始日・終了日とも両端を含む。解答時刻・学習開始時刻はデータベース上はUTCで保存されているが、日本時間(JST)の暦日に変換したうえで期間に当てはめる。
- 期間を指定しない場合、終了日は当日、開始日は当日から29日前になる(当日を含めて30日間)。
- 開始日は終了日より後の日付にできない。
- 指定できる期間は366日以内。
- KPIの「直前の期間」は、指定した期間と同じ日数だけ、指定した期間の開始日の前日までを遡った期間になる(例: 8/13〜9/11の30日間を指定した場合、直前の期間は7/14〜8/12)。
- 正答率が低い単元・設問のランキングは、指定期間内の解答件数が20件以上のものだけを対象とし、正答率の低い順に最大10件を返す(同率の場合は単元ID・設問IDの昇順)。論理削除された単元・問題・コースは対象外。
- 高校別利用状況は、生徒が1人以上在籍する高校のみを対象とし、アクティブ率の低い順(利用されていない高校が上位)に返す(同率の場合は高校IDの昇順)。
- コンテンツ網羅率(`content_coverage`)は、高校・教科による絞り込みの影響を受けない、コンテンツ全体に対する指標である。指定した期間は「期間内に解答が無い単元」の判定にのみ使う。論理削除された単元・コースは対象外。
- `high_school_id`・`subject_id`に該当するデータが存在しない場合でも404にはならず、該当する集計結果が0件・空配列で返る。

---

### Database変更

- なし(参照のみ)。

---

### Response

|項目|説明|
|-|-|
|`kpis`|`active_student_count`・`answer_count`・`accuracy_rate`・`study_minutes`の4項目。各項目は`current`(指定期間)・`previous`(直前の同じ長さの期間)を持つ|
|`daily_activity`|指定期間の日ごとの配列。各要素は`date`・`active_student_count`・`answer_count`・`accuracy_rate`|
|`low_accuracy_units`|正答率が低い単元の配列(最大10件)。`unit_id`・`unit_name`・`course_id`・`course_name`・`subject_name`・`answer_count`・`accuracy_rate`|
|`low_accuracy_questions`|正答率が低い設問の配列(最大10件)。`question_id`・`question_text`(80文字で切り詰め)・`unit_id`・`unit_name`・`course_id`・`answer_count`・`accuracy_rate`|
|`high_school_usage`|高校別の利用状況の配列。`high_school_id`・`high_school_name`・`student_count`・`active_student_count`・`active_rate`・`answer_count`・`accuracy_rate`|
|`content_coverage`|`total_units`・`units_without_questions`・`units_without_answers`|
|`meta`|`from`・`to`・`previous_from`・`previous_to`・`min_answer_count`(20)・`ranking_limit`(10)・`max_range_days`(366)・`generated_at`(集計実行時刻)|

正答率・アクティブ率は、分母となる件数が0件の場合は0、それ以外は小数第1位までの百分率(%)で返す。

---

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|未認証|401|-|
|管理者以外のロールでアクセス|403|-|
|`from`・`to`が日付として解釈できない|422|`{"errors": [...]}`。対象項目は`from`または`to`、メッセージは「は正しい日付形式(YYYY-MM-DD)で指定してください」|
|開始日が終了日より後|422|メッセージは「開始日は終了日より前の日付を指定してください」|
|指定した期間が366日を超える|422|メッセージは「指定できる期間は366日以内です」|

---

# 6. データモデル

本機能が所有するテーブルはない。以下はすべて他機能が所有するテーブルの参照である。

|テーブル|役割|本機能で使う主なカラム|
|-|-|-|
|`question_histories`|問題の解答履歴。問題解答機能が所有|`user_id`, `course_id`, `unit_id`, `question_id`, `is_correct`, `answered_at`, `deleted_at`|
|`study_logs`|学習ログ。学習ログ機能が所有|`user_id`, `unit_id`, `started_at`, `duration_minutes`, `deleted_at`|
|`units`|単元。教材が所有|`id`, `course_id`, `unit_name`, `deleted_at`|
|`questions`|設問。教材が所有|`id`, `unit_id`, `question_text`, `deleted_at`|
|`courses`|コース。教材が所有|`id`, `subject_id`, `level_name`, `deleted_at`|
|`subjects`|教科。教材が所有|`id`, `name`|
|`high_schools`|高校。共通マスタ参照機能が所有|`id`, `name`|
|`users`|ユーザー|`id`, `high_school_id`, `password_reset_required`, `deleted_at`|

リレーション:

- `question_histories`は`users`・`courses`・`units`・`questions`に属する。`course_id`・`unit_id`・`question_id`を履歴自身が保持しているため、単元・設問から辿らずに集計できる。
- `study_logs`は`users`・`units`に属する。
- `subjects` → `courses` → `units` → `questions`の階層が、ランキング集計(単元・設問)の表示項目(`course_name`・`subject_name`)の組み立てに対応する。
- `users`は1つの`high_schools`に属する(高校別利用状況・高校による絞り込みで使う)。

参照する側の論理削除の扱い:

- `question_histories`・`study_logs`は、いずれも論理削除済みを除外して参照される。
- `units`・`questions`・`courses`は、いずれも論理削除済みを除外して参照される。
- `users`(集計対象の生徒)は、論理削除済みを除外し、かつ招待受諾済み(`password_reset_required`が偽)のものに限る。

---

# 7. 状態管理

- なし。本機能が扱うデータはいずれも状態を持たず、リクエストごとに集計結果を算出するのみである。

---

# 8. 権限制御

|操作|認証|ロール|
|-|-|-|
|分析データ取得|必要|`admin`(管理者)のみ|

管理者以外のロール(生徒・教員)でアクセスした場合、他の管理者向けAPIと共通の権限判定によって拒否される。

---

# 9. 非同期処理

- なし。集計はリクエスト時に同期で実行される。
- キャッシュ・事前集計(バッチ)は使用していない。常に実データ(`question_histories`・`study_logs`等)を直接集計する。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::AnalyticsController`|
|Form|`Admin::AnalyticsFilterForm`|
|Query|`Admin::AnalyticsQuery`|
|Query(concern)|`AnalyticsRankable`, `AnalyticsHighSchoolUsable`, `AnalyticsDailyActivityable`, `AnalyticsContentCoverable`|
|Policy|`AdminServicePolicy`|
|参照する他機能のModel|`QuestionHistory`, `StudyLog`, `Unit`, `Question`, `Course`, `Subject`, `HighSchool`, `User`|
|Serializer|使用しない(Queryが集計結果のHashを直接組み立てて返す)|

注意:

この情報は現在実装との対応確認用である。新システム設計へRails構造をそのまま引き継ぐ目的ではない。
