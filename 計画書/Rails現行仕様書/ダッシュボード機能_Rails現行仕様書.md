# ダッシュボード機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 生徒のホーム画面に表示する情報を、1回のリクエストでまとめて取得すること。
- システム上の役割
  - 期限が近い目標を最大 5 件、紐づくタスクとあわせて取得する。
  - その生徒の当日の解答件数を取得する。
  - いずれもこの機能が作成・更新するデータではなく、他機能が記録したデータの参照・集約のみを行う。
- 利用者
  - `student` ロールのユーザー（生徒）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|表示|期限が近い目標を最大 5 件と、当日の解答件数をまとめて取得する|

この機能に作成・更新・削除の操作はない。

---

# 3. 業務フロー

1. 生徒がダッシュボード画面を表示する。
2. システムは生徒の目標を期限の昇順で最大 5 件取得する。紐づくタスクも同時に読み込む。
3. 続けて、その生徒の当日の解答件数を取得する。
4. 2と3をまとめて返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Student::DashboardsController`|`show`|GET|`/api/v1/student/dashboard`|期限が近い目標と当日の解答件数を取得|

ルーティングは `resource :dashboard, only: :show` として単数形で定義されており、この1エンドポイントのみである。

---

# 5. API / 処理詳細

## `Api::V1::Student::DashboardsController#show`

### 概要

生徒のホーム画面に表示する、期限が近い目標と当日の解答件数を取得する。

### Request

- なし（パスパラメータ・クエリパラメータ・ボディのいずれも受け取らない）

### 処理内容

1. `current_user.goals` を対象に、期限の昇順（同値は `id` 昇順）で並べ、最大 5 件に絞り、紐づくタスクを同時に読み込む（`GoalsQuery` の `due_soon`・`limit_five`・`includes_tasks`）。
2. その生徒の当日の解答件数を取得する（`Student::TodayAnswerCountQuery`）。件数は **Redis** に保持されており、DB からは取得しない。
3. 1と2を `goals` と `today_answer_count` の2キーにまとめて返却する。

### 業務ルール

- 生徒自身の目標のみ対象にする。
- 期限が近い順で最大 5 件まで取得する。期限が同じ場合は `id` の昇順で並べる。
- 論理削除済み（`deleted_at` が設定済み）の目標は対象に含まれない（`Goal` の `default_scope`）。
- 当日の解答件数は、この機能では**参照のみ**行う。件数を加算するのは問題解答機能である（「9. 非同期処理」）。
- 当日の解答件数の取得に失敗した場合は、エラーとせず `0` を返す（下記）。

### Database変更

- なし

### Response

- 最上位は `goals` と `today_answer_count` の2キーである
- `goals` の各要素は `Student::GoalSerializer`
  - `id`
  - `title`
  - `description`
  - `status`
  - `due_date`（`YYYY/MM/DD` 形式。未設定なら `null`）
  - `tasks`: 紐づくタスクの配列（`TaskSerializer`）。各要素は `id`・`user_id`・`goal_id`・`title`・`content`・`due_date`（`YYYY/MM/DD`）・`priority`・`status`・`completed_at`（`YYYY/MM/DD`）と、`units`（`UnitSerializer`）
- `today_answer_count`: 当日の解答件数（整数）
- ページ情報（`meta`）は返さない。最大 5 件に固定されており、ページングを行わないため

目標の各要素は目標管理機能の一覧と同じシリアライザであり、項目も同一である（「目標管理機能」の「5. API / 処理詳細」`#index`）。

### Errorケース

|条件|HTTP Status|内容|
|-|-|-|
|当日の解答件数の取得に失敗した（Redis の障害）|—|エラーにしない。ログに記録したうえで `0` を返し、200 で応答する|
|未認証|401|認証エラー|
|`student` ロールでない|403|`{"errors": ["この操作を行う権限がありません"]}`|

目標の取得に固有のエラーケースはない。目標が 0 件の場合も `goals` を空配列として 200 を返す。

---

# 6. データモデル

この機能が参照するデータは、いずれも他機能が所有するものである。

## `goals`（参照のみ）

- 役割: 生徒の学習目標を記録するテーブル
- この機能が参照する項目: `id`・`title`・`description`・`status`・`due_date` と、紐づくタスク
- 詳細なカラム定義・リレーション・`default_scope` は「目標管理機能」の「6. データモデル」を参照
- この機能はこのテーブルを更新しない

## `tasks`・`units`（参照のみ）

- 役割: 目標に紐づくタスクと、タスクに紐づく単元
- この機能が参照する項目はレスポンスに含まれるとおり。詳細は「タスク管理機能」の「6. データモデル」を参照
- この機能はこれらのテーブルを更新しない

## 当日の解答件数（Redis）

- 保持先: **Redis**。RDB のテーブルではない
- キー: `student:today_answer_count:{ユーザーID}:{日付}`。日付は ISO 8601 形式（`YYYY-MM-DD`）で、サーバの現在時刻の日付を用いる（`Student::TodayAnswerCountKey`）
- 値: 整数（解答のたびに 1 加算される）
- 有効期限: キーが新規作成された時点で「当日の終わり + 5 分」に設定される。既に有効期限が付いているキーには再設定しない（`expireat` の `nx` オプション）。日付をキーに含めているため、日付が変わると別のキーになり、前日のキーは有効期限で消える
- 取得時の挙動: キーが存在しない場合は `0` として扱う（`nil.to_i`）。Redis の障害時はログに記録して `0` を返す
- この機能は値を加算しない。参照のみ行う

---

# 7. 状態管理

- この機能に状態を持つデータはない。目標・タスクの状態は参照して返すのみで、変更しない。

---

# 8. 権限制御

- `student` ロールのユーザーのみ利用できる（`Api::V1::Student::BaseController` の認可）。
- 目標は `current_user.goals` から取得するため、生徒自身の目標のみ対象となる。
- 当日の解答件数は、キーに操作者のユーザーIDを含めるため、生徒自身の件数のみ取得される。

---

# 9. 非同期処理

- この機能の取得処理に紐づく Job / Mailer は見当たらない。
- ただし、当日の解答件数は**この機能の外で加算される**。生徒が問題に解答した時点で `Student::IncrementTodayAnswerCountService` が呼ばれ、Redis のカウンタを 1 加算する。呼び出し元は `Student::CreateQuestionHistoryForm`（問題解答機能）である。
- 加算は Redis の `INCR` と `EXPIREAT` をパイプラインでまとめて実行する。Redis の障害時はログに記録し、解答の記録自体は続行する（加算の失敗で解答が失敗することはない）。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Student::DashboardsController`|
|Controller（共通）|`ApplicationController`, `Api::V1::Student::BaseController`（生徒向け API の認可）|
|Query|`GoalsQuery`（`due_soon`・`limit_five`・`includes_tasks`。目標管理機能と共用）|
|Query|`Student::TodayAnswerCountQuery`（Redis から当日の解答件数を取得。障害時は `0`）|
|キーの構成|`Student::TodayAnswerCountKey`（`student:today_answer_count:{ユーザーID}:{日付}`）|
|Service（他機能）|`Student::IncrementTodayAnswerCountService`（問題解答機能から呼ばれ、Redis のカウンタを加算する）|
|Model|`Goal`（参照のみ）, `Task`（同）, `Unit`（同）|
|Serializer|`Student::GoalSerializer`（目標管理機能と共用。`tasks` を `TaskSerializer` で含める）|

注意:

この情報は現在実装との対応確認用です。新システム設計へRails構造をそのまま引き継ぐ目的ではありません。
