# 生徒CSVインポート機能 Go実装仕様書（2. Infrastructure）

---

# 1. 機能概要

本書はInfrastructure層（13ファイル、テスト込みで約22）を扱う。

## 本書の位置づけ

本書は`生徒CSVインポート機能_Go実装仕様書_1_Domain.md`（以下「1_Domain」）のスライス編である。

1_Domainが定めるもの（本書では繰り返さない）:

- Domain層のすべて（Entity・Value Object・Repository Interface・Domain Service・Domain Event・Domain Error）
- `dependency.go`（`GradeDirectory`・`SchoolClassDirectory`・`StudentAccountCreator`の3インターフェース）
- `context.go`、およびTransaction・Validation・Authorization・Errorの実装方針

本書が定めるもの:

- Repository実装（`infrastructure/gormrepo/`）・GORMモデル
- `dependency.go`のインターフェースの実装（`infrastructure/contextadapter/`）
- 外部連携実装（ファイルストレージ・非同期ジョブ）
- GORM / DBクエリ設計

1_Domainを実装・マージしたうえで本書の実装に着手する。

---

# 2. ディレクトリ構成

本書では扱わない。1_Domain「2. ディレクトリ構成」の機能全体のファイル一覧のうち、本書が担当するものは以下の13である。

```
internal/studentimport/infrastructure/gormrepo/import_error_model.go
internal/studentimport/infrastructure/gormrepo/import_history_model.go
internal/studentimport/infrastructure/gormrepo/imported_student_model.go
internal/studentimport/infrastructure/gormrepo/import_error_repository.go
internal/studentimport/infrastructure/gormrepo/import_history_repository.go
internal/studentimport/infrastructure/gormrepo/imported_student_repository.go
internal/studentimport/infrastructure/gormrepo/student_account_repository.go
internal/studentimport/infrastructure/contextadapter/grade_directory.go
internal/studentimport/infrastructure/contextadapter/school_class_directory.go
internal/studentimport/infrastructure/contextadapter/student_account_creator.go
internal/studentimport/infrastructure/queue/job_publisher.go
internal/studentimport/infrastructure/queue/worker_handler.go
internal/studentimport/infrastructure/storage/csv_file_storage.go
```

---

# 3. Domain層設計

本書では定義しない。1_Domain「3. Domain層設計」を参照。

---

# 4. クラス図

1_Domain「4. クラス図」を参照。本書はRepository実装・`dependency.go`のインターフェースの実装を追加するのみである。

---

# 5. 状態遷移図

1_Domain「5. 状態遷移図」を参照。本書固有の状態遷移はない。

---

# 6. Application層設計

本書では扱わない。3_Application・Presentation.mdに記載する。

---

# 7. シーケンス図・処理フロー図

本書では扱わない。3_Application・Presentation.mdに記載する。

---

# 8. Infrastructure層設計

## Repository実装（`infrastructure/gormrepo/`）

本Contextが所有するテーブル（`import_histories`・`import_errors`・`imported_students`）と、②が明示した例外として直接アクセスする`users`の限定範囲（`StudentAccountRepository`）をここに置く。

### ImportHistoryRepository実装（`infrastructure/gormrepo/import_history_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewImportHistoryRepository`
- 対応するGORMモデル: `ImportHistoryModel`（テーブル`import_histories`、管理者問題インポート機能と共有）
- 各メソッドで発行するクエリ内容:
  - `Create`: `import_histories`へ1件INSERT（`import_type = 'student'`固定）
  - `Update`: id指定で`status`・`success_count`・`error_count`・`total_count`・`finished_at`を更新するUPDATE
  - `FindByID`: `id`一致条件によるSELECT（1件）
- Entity ⇔ GORMモデルの変換方針: 非公開の変換関数（`toEntity`/`fromEntity`）で行う。`Mode`/`Status`（Value Object）⇔文字列カラムの相互変換もこの関数内で行う

### ImportErrorRepository実装（`infrastructure/gormrepo/import_error_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewImportErrorRepository`
- 対応するGORMモデル: `ImportErrorModel`（テーブル`import_errors`、管理者問題インポート機能と共有）
- 各メソッドで発行するクエリ内容:
  - `CreateBatch`: `import_errors`への複数件一括INSERT
  - `FindByImportHistoryID`: `import_history_id`一致条件によるSELECT（複数件、`row_number`昇順。推測）

### ImportedStudentRepository実装（`infrastructure/gormrepo/imported_student_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewImportedStudentRepository`
- 対応するGORMモデル: `ImportedStudentModel`（テーブル`imported_students`。本Contextが所有する）
- 各メソッドで発行するクエリ内容:
  - `CreateBatch`: `imported_students`への複数件一括INSERT（呼び出し元のトランザクションに参加する。アーキテクチャ規約「11. Transaction実装パターン」の`gormtx.DBFromContext`を用いる）。`import_history_id`と`user_id`の組の一意制約違反は、Infrastructure Errorとして返す（通常は行検証（CSV内メール重複の拒否）により発生しない）
  - `FindByImportHistoryID`: `import_history_id`一致条件によるSELECT（複数件、`id`昇順）
- Entity ⇔ GORMモデルの変換方針: 非公開の変換関数（`toEntity`/`fromEntity`）で行う。`Action`（Value Object）⇔整数カラム（created=0, updated=1）の相互変換もこの関数内で行う

### StudentAccountRepository実装（`infrastructure/gormrepo/student_account_repository.go`）

- 実装struct名: 非公開struct + コンストラクタ`NewStudentAccountRepository`
- 対応するGORMモデル: `UserModel`。`user` Context所有の既存`users`テーブルを、②が明示した例外として参照・更新する（1_Domain「1. 機能概要」）。本Contextでは`UserModel`を自前の射影として定義し、`user` Context側のGORMモデルとは別に保持する（認証機能③文書の先例と同じ扱い）
- 各メソッドで発行するクエリ内容:
  - `FindByEmailInSchool`: `high_school_id = ? AND email = ? AND role = 生徒ロール`によるSELECT（1件）
  - `Update`: `id = ?`条件で氏名・氏名カナ・メールアドレス・学年・学級のみを更新する。`StudentNumber`が非nilの場合のみ`student_number`列も更新する（招待メールは送らず、`password_reset_required`・パスワード・認証状態の列には触れない）

## contextadapter（`dependency.go`のインターフェースの実装）

他Context（`master-data`・`school-class`・`user`）の公開操作へ委譲するアダプタを、`infrastructure/contextadapter/`に置く（アーキテクチャ規約「4. Bounded Context構成」。`gormrepo/`には置かない）。アダプタ自身はGORMに依存せず、相手Contextの公開操作を呼び出すだけである（同「5. Context間連携ルール」）。

### GradeDirectory実装（`infrastructure/contextadapter/grade_directory.go`）

- 実装struct名: 非公開struct（例: `gradeDirectory`）+ コンストラクタ`NewGradeDirectory(masterDataContext *masterdata.Context) studentimport.GradeDirectory`
- 委譲先: `master-data`の`FindGradesByHighSchoolID(ctx, highSchoolID)`（共通マスタ参照機能③_1「公開する操作」）
- 実装: `FindGradesByHighSchoolID`の結果から、学年の表示名（`DisplayName`）が`gradeName`と一致する要素を探し、一致するものがあれば`gradeID`・`found=true`を、なければ`found=false`を返す
- 本アダプタはGORMに依存せず、`grades`へクエリを発行しない

### SchoolClassDirectory実装（`infrastructure/contextadapter/school_class_directory.go`）

- 実装struct名: 非公開struct（例: `schoolClassDirectory`）+ コンストラクタ`NewSchoolClassDirectory(schoolClassContext *schoolclass.Context) studentimport.SchoolClassDirectory`
- 委譲先: `school-class`が公開する学級名による解決操作（**要確認**。クラス編成機能③にまだ定義がない。1_Domain「2. ディレクトリ構成」参照）
- 実装方針: `school-class`側の追随時に確定する公開操作を呼び出し、結果を`schoolClassID`・`found`へ変換する。本アダプタはGORMに依存せず、`school_classes`へクエリを発行しない

### StudentAccountCreator実装（`infrastructure/contextadapter/student_account_creator.go`）

- 実装struct名: 非公開struct（例: `studentAccountCreator`）+ コンストラクタ`NewStudentAccountCreator(userContext *user.Context) studentimport.StudentAccountCreator`
- `CreateStudentAccount`: `user` Contextが公開する`CreateStudentAccount`を呼び出し、`CreateStudentAccountParams`を`user`の入力へ、`user`が返す基本属性を`CreateStudentAccountResult`へ変換する。本Contextでは`users`へINSERTしない（仮パスワードの生成・生徒番号の発行・招待メール送信依頼の登録は`user`が行う）
- `IssueStudentNumber`: `user`の`IssueStudentNumber(ctx, highSchoolID)`をそのまま呼び出す
- トランザクション: `ExecuteStudentImportUseCase`が開始したトランザクションに`user`の操作が参加するよう、`ctx`をそのまま渡す（アーキテクチャ規約「11. Transaction実装パターン（TransactionManager）」の「呼び出し側トランザクションの引き継ぎ」。ユーザー基盤機能③「4. トランザクションの引き継ぎ」のとおり、`user`側は`ctx`にトランザクションがあればそれに参加する）
- 入出力の型・エラー種別の判別（メールアドレス重複のValidation等）: ユーザー基盤機能③「3. 公開操作の定義」のとおり

## 外部連携実装

|実装対象|呼び出し元|実装方針|
|-|-|-|
|`FileStorage`実装（`infrastructure/storage/csv_file_storage.go`）|`StartStudentImportUseCase.Save`, `ExecuteStudentImportUseCase.Read`|CSVファイル本体をディスクまたはオブジェクトストレージへ保存し、`import_histories.file_path`（②20章のSchema変更対象、15章参照）にキー情報を記録する。具体的な保存先（ローカルディスク／S3等）は②に選定基準の記載がなく、管理者問題インポート機能と共通の変更であるため両機能で同一の実装方針を採る（推測）|
|招待メール送信|`user`の`CreateStudentAccount`（内部で`jobs`へ送信依頼を登録）|本Contextでは実装しない。新規行の作成を`user`へ依頼することで、招待メールの送信依頼が同一トランザクションで登録される（②11章）|
|`JobPublisher`実装（`infrastructure/queue/job_publisher.go`）、非同期ワーカー（`infrastructure/queue/worker_handler.go`）|`StartStudentImportUseCase`、`main.go`起動時のワーカー登録|アーキテクチャ規約「13. 非同期ジョブ実行パターン（JobQueue）」の「確実に実行したい処理」として実装する。`jobs`テーブル＋ポーリングワーカーの標準実装を用いる。`jobType = "student_import"`のハンドラを`main.go`のワーカー起動時に登録し、`ExecuteStudentImportUseCase.Execute`を呼び出す|

---

# 9. Presentation層設計

本書では扱わない。3_Application・Presentation.mdに記載する。

---

# 10. API仕様

本書では扱わない。3_Application・Presentation.mdに記載する。

---

# 11. Transaction実装方針

本書では扱わない。1_Domain「11. Transaction実装方針」を参照。`gormrepo`配下のRepository実装は`gormtx.DBFromContext(ctx, r.db)`でトランザクション用のコネクションを取得する。

---

# 12. Validation実装方針

本書では扱わない。1_Domain「12. Validation実装方針」を参照。

---

# 13. Authorization実装方針

本書では扱わない。1_Domain「13. Authorization実装方針」を参照。

---

# 14. Error実装方針

本書では扱わない。1_Domain「14. Error実装方針」を参照。

---

# 15. GORM / DBクエリ設計

## 利用するGORMモデルとテーブルの対応

②「20. DB設計方針」により、既存Rails DBを継続利用しつつ、ファイル参照方法のみSchema変更を行う（管理者問題インポート機能と共通の変更）。

### ImportHistoryModel（`infrastructure/gormrepo/import_history_model.go`）

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

### ImportErrorModel（`infrastructure/gormrepo/import_error_model.go`）

- 対応テーブル: `import_errors`（管理者問題インポート機能と共有）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`ImportHistoryID`|`import_history_id`||
|`RowNumber`|`row_number`||
|`Message`|`message`||

### ImportedStudentModel（`infrastructure/gormrepo/imported_student_model.go`）

- 対応テーブル: `imported_students`（既存Rails DBのテーブル。②「20. DB設計方針」のとおり、Go設計での追加変更はない）

|フィールド|対応カラム|備考|
|-|-|-|
|`ID`|`id`|主キー|
|`ImportHistoryID`|`import_history_id`|`import_histories`への外部キー。`user_id`との組で一意|
|`UserID`|`user_id`|インポートされた生徒のID|
|`Action`|`action`|整数カラム（created=0, updated=1。Rails現行のスキーマ`integer`のまま）。既存Schemaの列型は変更せず、Go側のモデル変換で整数⇔`ImportedStudentAction`を相互変換する（**②からの補足**。推測ではなく、Rails現行のスキーマに基づく）|

### UserModel（`infrastructure/gormrepo/student_account_repository.go`内に定義）

- 対応テーブル: `users`。本Contextが読み書きするサブセット（`id`・`name`・`name_kana`・`email`・`high_school_id`・`grade_id`・`school_class_id`・`student_number`・`user_role_id`）のみを射影する

## 主要クエリの条件・ソート・ページネーション方針

|Repository|メソッド|対象テーブル|条件|結合|
|-|-|-|-|-|
|`ImportHistoryRepository`|`Create`/`Update`/`FindByID`|`import_histories`|`id`一致、または`import_type = 'student'`での絞り込み|不要|
|`ImportErrorRepository`|`CreateBatch`/`FindByImportHistoryID`|`import_errors`|`import_history_id`一致|不要|
|`ImportedStudentRepository`|`CreateBatch`/`FindByImportHistoryID`|`imported_students`|`import_history_id`一致（参照）。`id`昇順|不要|
|`StudentAccountRepository`|`FindByEmailInSchool`/`Update`|`users`|`high_school_id`一致、`email`一致、生徒ロール（照会）、`id`一致（更新）|不要|

`GradeDirectory`・`SchoolClassDirectory`・`StudentAccountCreator`はSQLを直接発行せず、それぞれの所有Context側の公開操作呼び出しに委譲する（「8. Infrastructure層設計」）。

SQL文そのものは記載しない。

## 既存Schemaに対する変更

②「20. DB設計方針」で提案されているとおり、`import_histories`テーブルにファイルの保存先を直接表すカラム（`file_path`、推測）を追加する。この変更は管理者問題インポート機能と共有するテーブルへの変更であり、両機能で整合させる必要がある。既存のRails側ActiveStorageデータからのマイグレーション（移行スクリプト）が必要になる。

---

# 16. テストケース設計（本書の担当分）

## Repository Test

|対象|テストケース|
|-|-|
|`ImportHistoryRepository`/`ImportErrorRepository`|永続化・検索の正確性|
|`ImportedStudentRepository`|`CreateBatch`で記録した成功行が`FindByImportHistoryID`で記録された順に取得できること／同じ履歴に同じ生徒を重複して記録しようとした場合に一意制約違反となること／`action`の整数⇔`ImportedStudentAction`の変換が正しいこと|
|`StudentAccountRepository`|更新（招待メールを送らず`password_reset_required`に触れないこと）・メール一致確認の正確性|

## contextadapter Test

|対象|テストケース|
|-|-|
|`GradeDirectory`|`FindGradesByHighSchoolID`への引数が正しく渡ること／表示名が一致する学年が見つかった場合に`found=true`となること／見つからない場合に`found=false`となること|
|`SchoolClassDirectory`|`school-class`側の公開操作が確定した時点で、引数・戻り値の変換が正しいことを検証する（**要確認**。公開操作の確定が前提）|
|`StudentAccountCreator`|`user`の`CreateStudentAccount`へ入力が正しく変換されて渡され、結果が変換されて返ること（`user`の作成規則そのものは`user`のテストで検証する）／`IssueStudentNumber`の呼び出しパラメータが正しいこと|

---

# 17. ②からの補足事項

②に明記がなく、実装のために本書で補った内容である。1_Domain「17. ②からの補足事項」に記載した内容に加え、本書固有のものを以下に示す。

|判断した内容|判断理由|推測か|
|-|-|-|
|`GradeDirectory`・`SchoolClassDirectory`・`StudentAccountCreator`の実装を`infrastructure/contextadapter/`に置いた|アーキテクチャ規約「4. Bounded Context構成」が他Context依存のインターフェースの実装を`contextadapter/`に置くと定めているため（旧版は`infrastructure/repository/grade_class_resolution_repository.go`等としていた）|推測ではない（規約に従った）|
|`GradeDirectory`の実装先を、`master-data`が公開する`FindGradesByHighSchoolID`とした|共通マスタ参照機能③_1が確定済みで公開する操作であり、新しい公開操作の追加が不要であるため|推測ではない（既存の公開操作で足りる）|
|`StudentAccountRepository`の`UserModel`を、認証機能③と同様に本Context専用の射影として独自定義した|`users`テーブルを複数Contextが異なる役割で参照する既存の先例（認証機能③「15. GORM / DBクエリ設計」）に合わせた|推測ではない（既存③の先例に合わせた）|

上記以外の設計判断は1_Domainの記載をそのまま踏襲する。
