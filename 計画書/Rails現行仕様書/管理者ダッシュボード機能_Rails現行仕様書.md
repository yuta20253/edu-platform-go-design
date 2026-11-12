# 管理者ダッシュボード機能 Rails現行仕様書

---

# 1. 機能概要

- 機能の目的
  - 管理者がシステム全体の状況を1画面で把握できるように、利用者数・問題数・最近の問題インポート状況・最近のお知らせをまとめて表示すること。
- システム上の役割
  - 利用者種別ごとに、招待受諾済みと招待中の人数を集計する。
  - 直近30日以内に学習の記録がある生徒の人数を集計する。
  - 論理削除されていない問題の総数を集計する。
  - 直近の問題インポート履歴と、直近のお知らせを取得する。
  - 本機能はデータを所有せず、他機能が管理するデータを読み取り専用で集約する。
- 利用者
  - `admin` ロールのユーザー（管理者）

---

# 2. 機能一覧

|操作|概要|
|-|-|
|表示|利用者種別ごとの人数（受諾済み・招待中）、アクティブな生徒数、問題の総数、最近の問題インポート履歴（最大5件）、最近のお知らせ（最大3件）を取得する|

---

# 3. 業務フロー

1. 管理者がダッシュボード画面を表示する。
2. システムは、有効な利用者を種別ごとに「招待受諾済み」「招待中」に分けて集計する。
3. 直近30日以内に学習ログまたは解答履歴を記録した、招待受諾済みの生徒の人数を集計する。
4. 論理削除されていない問題の総数を集計する。
5. 問題インポートの履歴を新しい順に最大5件取得する。
6. お知らせを最大3件取得する。
7. 集計結果と、集計の条件（対象期間の日数・集計時刻）をあわせて返却する。

---

# 4. API一覧

|Controller|Action|HTTP Method|Endpoint|概要|
|-|-|-|-|-|
|`Api::V1::Admin::DashboardsController`|`show`|GET|`/api/v1/admin/dashboard`|管理者ダッシュボード情報を取得|

---

# 5. API / 処理詳細

## `Api::V1::Admin::DashboardsController#show`

### 概要

システム全体のユーザー数・問題数、最近の問題インポート状況、最近のお知らせを、管理者用ダッシュボードとして取得する。

### Request

- なし

### 処理内容

1. 有効なユーザー（無効化されていないユーザー）を利用者種別ごとに集計する。種別ごとに「招待受諾済み」と「招待中」の2つの人数を同時に数える。
2. 直近30日以内に学習ログまたは解答履歴を記録した、招待受諾済みの生徒の人数を数える。
3. 論理削除されていない問題の総数を数える。
4. 問題インポートの履歴のうち論理削除されていないものを、作成日時の新しい順（同時刻は ID の降順）で最大5件取得する。
5. お知らせを既定の並び順で最大3件取得する。
6. 集計結果・履歴・お知らせと、集計の条件を表す `meta` を返却する。

### 業務ルール

- 管理者はすべての高校・ユーザーを集計対象とする。
- **人数の集計対象は、無効化されていない（`deleted_at` が未設定の）ユーザーに限る。**
- **招待受諾の判定にはパスワード再設定の要否（`password_reset_required`）を用いる。** 招待時に真となり、本人の初回パスワード変更で偽になる。有効化日時（`activated_at`）は、インポート済み生徒の引き継ぎ経路でしか設定されず、通常の新規登録や教員の初回パスワード変更では未設定のままになるため使わない。
- 「アクティブな生徒」は、招待受諾済みの生徒のうち、直近30日以内に**学習ログの開始**または**問題の解答**を記録した者である。いずれか一方でも該当すれば数える。判定の起点となる時刻は1回の取得の中で共通とし、学習ログと解答履歴で基準をずらさない。
- 問題の総数は論理削除済みを除外する。
- 最近のインポート履歴には問題インポートの履歴のみが現れ、生徒インポートの履歴は含まれない。論理削除済みの履歴も除外する。
- 最近のお知らせは、本文・配信対象・配信者を含まない軽量な項目のみを返す。

### Database変更

- なし

### Response

- **200 OK**
- `stats`
  - `student_count`（招待受諾済みの生徒数）
  - `active_student_count`（直近30日以内に学習ログまたは解答履歴がある招待受諾済みの生徒数）
  - `teacher_count`（招待受諾済みの教員数）
  - `admin_count`（招待受諾済みの管理者数）
  - `total_questions`（論理削除されていない問題の総数）
  - `pending_student_count`（招待中の生徒数）
  - `pending_teacher_count`（招待中の教員数）
- `recent_imports`（最大5件。各要素は `id`, `file_name`, `status`, `success_count`, `error_count`, `total_count`, `created_at`）
- `recent_announcements`（最大3件。各要素は `id`, `title`, `status`, `published_at`, `scheduled_at`, `created_at`）
- `meta`
  - `active_student_period_days`（アクティブ判定の対象期間の日数。30）
  - `generated_at`（集計を行った時刻）

該当するデータがない場合も、`recent_imports`・`recent_announcements` は空の配列を返す。種別の人数は該当者がいなければ0を返す。

### Errorケース

- なし（認証前提）

---

# 6. データモデル

本機能は専用のテーブルを持たず、他機能が管理するテーブルを集計・参照する。

## `import_histories`

- 役割: CSV インポート（問題インポート・生徒インポート）の実行履歴を管理するテーブル
- 主なカラム: `id`, `user_id`, `unit_id`, `file_name`, `status`, `import_type`, `total_count`, `success_count`, `error_count`, `deleted_at`, `created_at`
- リレーション: `belongs_to :user`, `belongs_to :unit`（任意）
- `import_type` によりインポート種別（問題インポート／生徒インポート）を区別しており、ダッシュボードに表示される「最近のインポート履歴」は問題インポートの履歴のみを対象とする。

## `users` / `user_roles`

- 役割: 利用者とその種別。人数の集計対象。
- 本機能が参照する主なカラム: `users.deleted_at`（無効化の判定）、`users.password_reset_required`（招待受諾の判定）、`user_roles.name`（種別。`student` / `teacher` / `admin` / `guardian`）

## `questions`

- 役割: 問題。総数の集計対象。
- 本機能が参照する主なカラム: `deleted_at`（論理削除の判定）

## `study_logs` / `question_histories`

- 役割: 学習ログ・解答履歴。アクティブな生徒の判定に用いる。
- 本機能が参照する主なカラム: `study_logs.started_at` / `question_histories.answered_at`（直近30日の判定）、両テーブルの `deleted_at`（論理削除済みは判定に含めない）、`user_id`

## `announcements`

- 役割: お知らせ。最近のお知らせの取得対象。
- 本機能が参照する主なカラム: `id`, `title`, `status`, `published_at`, `scheduled_at`, `created_at`

---

# 7. 状態管理

本機能は状態を持たず、状態を変更しない。

返す値のうち状態に関わるものは次の2つで、いずれも他機能が管理する状態をそのまま載せたものである。

|項目|状態の意味|管理する機能|
|-|-|-|
|`recent_imports[].status`|インポートの進行状況（`pending` / `processing` / `completed` / `failed`）|管理者問題インポート機能|
|`recent_announcements[].status`|お知らせの公開状況|管理者お知らせ管理機能|

人数の集計で用いる「招待受諾済み」「招待中」も、ユーザーの状態（`password_reset_required`）に基づく区分であり、本機能が変更するものではない。

---

# 8. 権限制御

- `current_user` が `admin` ロールであることが前提。

---

# 9. 非同期処理

なし。本機能は参照のみであり、ジョブ・メール送信を伴わない。

---

# 10. Rails実装対応表

|種類|実装|
|-|-|
|Controller|`Api::V1::Admin::DashboardsController`|
|Query|`Admin::DashboardQuery`（集計の本体。対象期間の日数・取得件数の上限を定数として持つ）、`AnnouncementsQuery`（お知らせの既定の並び順）|
|Serializer|`ImportHistorySerializer`、`Admin::DashboardAnnouncementSerializer`|
|Model|`User`, `UserRole`, `ImportHistory`, `Question`, `StudyLog`, `QuestionHistory`, `Announcement`|

集計の条件を表す定数は `Admin::DashboardQuery` が持つ（アクティブ判定の対象期間30日、インポート履歴5件、お知らせ3件）。対象期間はコンストラクタの引数で差し替えられる形になっている。
