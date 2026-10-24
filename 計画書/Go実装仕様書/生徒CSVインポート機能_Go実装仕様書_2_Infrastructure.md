# 生徒CSVインポート機能 Go実装仕様書（2. Infrastructure）

---

# 1. 本書の範囲

本書はInfrastructure層（13ファイル、テスト込みで約22）を扱う。

## 本書が前提とする文書

- `生徒CSVインポート機能_Go実装仕様書_1_Domain.md`: Domain層（Entity・Value Object・Repository Interface・Domain Service・Domain Event・Domain Error）、およびTransaction・Validation・Authorization・Errorの実装方針
- 1_Domainを実装・マージしたうえで本書の実装に着手する

本書では、1_Domainに記載した実装方針を繰り返さない。担当ファイルの一覧は1_Domain「2. ディレクトリ構成」の「2_Infrastructure の担当」を参照する。

---

# 2. Infrastructure層設計

## Repository実装

### ImportHistoryRepository実装（`infrastructure/repository/import_history_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewImportHistoryRepository`
- 対応するGORMモデル: `gormmodel.ImportHistoryModel`（テーブル`import_histories`、管理者問題インポート機能と共有）
- 各メソッドで発行するクエリ内容:
  - `Create`: `import_histories`へ1件INSERT（`import_type = 'student'`固定）
  - `Update`: id指定で`status`・`success_count`・`error_count`・`total_count`・`finished_at`を更新するUPDATE
  - `FindByID`: `id`一致条件によるSELECT（1件）
- Entity ⇔ GORMモデルの変換方針: 非公開の変換関数（`toEntity`/`fromEntity`）で行う。`Mode`/`Status`（Value Object）⇔文字列カラムの相互変換もこの関数内で行う

### ImportErrorRepository実装（`infrastructure/repository/import_error_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewImportErrorRepository`
- 対応するGORMモデル: `gormmodel.ImportErrorModel`（テーブル`import_errors`、管理者問題インポート機能と共有）
- 各メソッドで発行するクエリ内容:
  - `CreateBatch`: `import_errors`への複数件一括INSERT
  - `FindByImportHistoryID`: `import_history_id`一致条件によるSELECT（複数件、`row_number`昇順。推測）

### ImportedStudentRepository実装（`infrastructure/repository/imported_student_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewImportedStudentRepository`
- 対応するGORMモデル: `gormmodel.ImportedStudentModel`（テーブル`imported_students`。本Contextが所有する）
- 各メソッドで発行するクエリ内容:
  - `CreateBatch`: `imported_students`への複数件一括INSERT（呼び出し元のトランザクションに参加する。アーキテクチャ規約「11. Transaction実装パターン」の`dbFromContext`を用いる）。`import_history_id`と`user_id`の組の一意制約違反は、Infrastructure Errorとして返す（通常は行検証（CSV内メール重複の拒否）により発生しない）
  - `FindByImportHistoryID`: `import_history_id`一致条件によるSELECT（複数件、`id`昇順）
- Entity ⇔ GORMモデルの変換方針: 非公開の変換関数（`toEntity`/`fromEntity`）で行う。`Action`（Value Object）⇔整数カラム（created=0, updated=1）の相互変換もこの関数内で行う

### GradeClassResolutionRepository実装（`infrastructure/repository/grade_class_resolution_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewGradeClassResolutionRepository`
- 対応するGORMモデル: `master-data` Context側の既存`grades`/`school_classes`テーブルを参照専用で読み取る（本Contextでは新規に所有モデルとして定義しない）
- クエリ内容:
  - `ResolveGrade`: `high_school_id = ? AND (学年名に相当するカラム) = ?`によるSELECT（1件）
  - `ResolveClass`: `grade_id = ? AND name = ?`によるSELECT（1件）

### StudentAccountRepository実装（`infrastructure/repository/student_account_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewStudentAccountRepository`
- 対応するGORMモデル: `user` Context所有の既存`users`テーブルを参照・更新する（本Contextでは新規に所有モデルとして定義しない）
- 各メソッドで発行するクエリ内容:
  - `FindByEmailInSchool`: `high_school_id = ? AND email = ? AND role = 生徒ロール`によるSELECT（1件）
  - `Update`: `id = ?`条件で氏名・氏名カナ・メールアドレス・学年・学級のみを更新する（招待メールは送らず、`password_reset_required`・パスワード・認証状態の列には触れない。生徒番号が空の場合は、`user`が公開する`IssueStudentNumber`を呼んで発行した値を同じ更新に含める（`user`②「論点と決定」6、`user`③「3. 公開操作の定義」））

### StudentAccountCreationRepository実装（`infrastructure/repository/student_account_creation_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewStudentAccountCreationRepository`
- 実装内容: `user` Contextが公開する`CreateStudentAccount`を呼び出し、`CreateStudentAccountParams`を`user`の入力へ、`user`が返す基本属性を`CreateStudentAccountResult`へ変換する。本Contextでは`users`へINSERTしない（仮パスワードの生成・生徒番号の発行・招待メール送信依頼の登録は`user`が行う）
- トランザクション: `ExecuteStudentImportUseCase`が開始したトランザクションに`user`の操作が参加するよう、`ctx`をそのまま渡す（アーキテクチャ規約「11. Transaction実装パターン（TransactionManager）」の「呼び出し側トランザクションの引き継ぎ」。`user`③「4. トランザクションの引き継ぎ」のとおり、`user`側は`ctx`にトランザクションがあればそれに参加する）
- 入出力の型・エラー種別の判別（メールアドレス重複のValidation等）: `user`③「3. 公開操作の定義」のとおり

## 外部連携実装

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`FileStorage`実装（`infrastructure/storage/csv_file_storage.go`）|`StartStudentImportUseCase.Save`, `ExecuteStudentImportUseCase.Read`|CSVファイル本体をディスクまたはオブジェクトストレージへ保存し、`import_histories.file_path`（②20章のSchema変更対象、15章参照）にキー情報を記録する。具体的な保存先（ローカルディスク／S3等）は②に選定基準の記載がなく、管理者問題インポート機能と共通の変更であるため両機能で同一の実装方針を採る（推測）|
|招待メール送信|`user`の`CreateStudentAccount`（内部で`jobs`へ送信依頼を登録）|本Contextでは実装しない。新規行の作成を`user`へ依頼することで、招待メールの送信依頼が同一トランザクションで登録される（②11章）|
|`JobPublisher`実装（`infrastructure/queue/job_publisher.go`）、非同期ワーカー（`infrastructure/queue/worker_handler.go`）|`StartStudentImportUseCase`、`main.go`起動時のワーカー登録|アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の「確実に実行したい処理」として実装する。`jobs`テーブル＋ポーリングワーカーの標準実装を用いる。`jobType = "student_import"`のハンドラを`main.go`のワーカー起動時に登録し、`ExecuteStudentImportUseCase.Execute`を呼び出す|

---

---

# 3. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

②「20. DB設計方針」により、既存Rails DBを継続利用しつつ、ファイル参照方法のみSchema変更を行う（管理者問題インポート機能と共通の変更）。

### ImportHistoryModel（`infrastructure/persistence/gorm/import_history_model.go`）

- 対応テーブル: `import_histories`（管理者問題インポート機能と共有）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`UserID`|`user_id`|実行した教師のID|
|`ImportType`|`import_type`|`'student'`固定（管理者問題インポート機能は`'question'`）|
|`Mode`|`mode`|文字列として保存|
|`Status`|`status`|文字列として保存|
|`FileName`|`file_name`||
|`FilePath`|`file_path`（新規カラム）|②20章のSchema変更対象。ファイルの保存パス、またはオブジェクトストレージ上のキーに相当する情報（推測。具体的なカラム名・型は本書で`file_path string`と仮定するが確定しない）|
|`TotalCount`|`total_count`||
|`SuccessCount`|`success_count`||
|`ErrorCount`|`error_count`||
|`StartedAt`|`started_at`||
|`FinishedAt`|`finished_at`|NULL許容|
|`CreatedAt`|`created_at`|GORMの自動タイムスタンプ（Gorm規約）|

### ImportErrorModel（`infrastructure/persistence/gorm/import_error_model.go`）

- 対応テーブル: `import_errors`（管理者問題インポート機能と共有）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`ImportHistoryID`|`import_history_id`||
|`RowNumber`|`row_number`||
|`Message`|`message`||

### ImportedStudentModel（`infrastructure/persistence/gorm/imported_student_model.go`）

- 対応テーブル: `imported_students`（既存Rails DBのテーブル。②「20. DB設計方針」のとおり、Go設計での追加変更はない）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`ImportHistoryID`|`import_history_id`|`import_histories`への外部キー。`user_id`との組で一意|
|`UserID`|`user_id`|インポートされた生徒のID|
|`Action`|`action`|整数カラム（created=0, updated=1。Rails現行のスキーマ`integer`のまま）。既存Schemaの列型は変更せず、Go側のモデル変換で整数⇔`ImportedStudentAction`を相互変換する（**②からの補足**。推測ではなく、Rails現行のスキーマに基づく）|

## 主要クエリの条件・ソート・ページネーション方針

|Repository|メソッド|対象テーブル|条件|結合|
|-|-|-|-|-|
|`ImportHistoryRepository`|`Create`/`Update`/`FindByID`|`import_histories`|`id`一致、または`import_type = 'student'`での絞り込み|不要|
|`ImportErrorRepository`|`CreateBatch`/`FindByImportHistoryID`|`import_errors`|`import_history_id`一致|不要|
|`ImportedStudentRepository`|`CreateBatch`/`FindByImportHistoryID`|`imported_students`|`import_history_id`一致（参照）。`id`昇順|不要|
|`GradeClassResolutionRepository`|`ResolveGrade`/`ResolveClass`|`grades`, `school_classes`|`high_school_id`一致、名称一致、`grade_id`一致|`grades`と`school_classes`の結合（学級が指定学年に属することの確認）|
|`StudentAccountRepository`|`FindByEmailInSchool`/`Update`|`users`|`high_school_id`一致、`email`一致、生徒ロール（照会）、`id`一致（更新）|不要|
|`StudentAccountCreationRepository`|`Create`|（`user`の`CreateStudentAccount`が`users`・`jobs`へ書き込む。本Contextはクエリを発行しない）|—|—|

SQL文そのものは記載しない。

## 既存Schemaに対する変更

②「20. DB設計方針」で提案されているとおり、`import_histories`テーブルにファイルの保存先を直接表すカラム（`file_path`、推測）を追加する。この変更は管理者問題インポート機能と共有するテーブルへの変更であり、両機能で整合させる必要がある。既存のRails側ActiveStorageデータからのマイグレーション（移行スクリプト）が必要になる。

---

---

# 4. テストケース設計（本書の担当分）

## Repository Test

|対象|テストケース|
|-|-|
|`ImportHistoryRepository`/`ImportErrorRepository`|永続化・検索の正確性|
|`ImportedStudentRepository`|`CreateBatch`で記録した成功行が`FindByImportHistoryID`で記録された順に取得できること／同じ履歴に同じ生徒を重複して記録しようとした場合に一意制約違反となること／`action`の整数⇔`ImportedStudentAction`の変換が正しいこと|
|`GradeClassResolutionRepository`|学年名・学級名からの名称解決の正確性（存在しない場合に`nil`を返すこと）|
|`StudentAccountRepository`|更新（招待メールを送らず`password_reset_required`に触れないこと）・メール一致確認の正確性|
|`StudentAccountCreationRepository`|`user`の`CreateStudentAccount`へ入力が正しく変換されて渡され、結果が変換されて返ること（`user`の作成規則そのものは`user`のテストで検証する）|
