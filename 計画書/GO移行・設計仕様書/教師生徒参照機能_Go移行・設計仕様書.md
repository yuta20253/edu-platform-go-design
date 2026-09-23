# 教師生徒参照機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

教師が同校の生徒一覧および生徒詳細を参照し、あわせて生徒アカウントを1件ずつ新規登録できる機能である。Rails現行仕様では、一覧取得・詳細取得に加え、氏名・氏名（カナ）・メールアドレス・学年・クラスを指定した生徒アカウントの新規作成（create）を提供する。一覧・詳細取得では、担当学年権限を持つ教師は担当学年の生徒のみに閲覧範囲が制限される。新規作成の作成経路は、生徒CSVインポート機能と共通のアカウント作成処理（仮パスワード発行・招待メール送信を含む）を利用する。

## 利用者

- 教師ユーザー（teacher ロール）
- 同校の生徒のみ参照・登録可能。担当学年権限がある場合は参照は担当学年の生徒のみに制限される（新規登録は担当学年権限の有無を問わず同校の教師であれば実行できる）

## 業務上の目的

- 教師が担当する生徒の状況を把握できるようにする
- 学校・学年という権限範囲に基づいて、閲覧できる生徒を適切に制限する
- 教師が生徒アカウントを個別に登録できるようにし、着任・転入時など少人数の追加登録に対応する
- 生徒データの参照・個別登録の窓口として提供し、既存データの更新は行わない

---

# 2. 設計方針

本機能は「参照」を中心としつつ、生徒アカウントの個別新規登録（create）を1操作だけ持つ機能であり、Rails実装の構造をそのまま移すのではなく、権限に基づく検索条件の組み立て・レスポンス整形と、登録時の同校妥当性検証に責務を絞ったGo設計とする。生徒アカウント本体の作成処理（仮パスワード発行・生徒番号採番・招待メール送信）は`user` Contextの`CreateStudentAccount`が担うため、本機能内に重複実装しない。

主な設計思想は以下のとおりである。

- 責務分離: HTTP・検索条件構築・権限スコープ・登録時の同校妥当性検証・レスポンス整形を分離する
- 保守性: 学校・学年による絞り込み条件を一箇所に集約し、権限ルール変更の影響範囲を限定する。生徒アカウント作成自体の業務ルール（仮パスワード発行・生徒番号採番・招待メール送信等）は`user` Contextに委譲し、本機能側の変更範囲を「誰が・どの学年/クラスに登録してよいか」に限定する
- テスト容易性: 検索条件の組み立てとページングをユースケース単位で検証できるようにする。新規登録時の同校妥当性検証も独立して検証できるようにする
- 拡張性: 将来的な検索条件（氏名検索等）の追加に対応しやすい構造とする
- API互換性: 既存エンドポイント・レスポンス構造（`students` + `meta`）およびcreateエンドポイントを維持する

---

# 3. Bounded Context

## Context名

- student-directory

## Contextの責務

- 教師視点での生徒情報の参照（一覧・詳細）
- 学校・学年スコープに基づく絞り込み
- 生徒アカウントの個別新規登録受付（同校の学年・クラスに対する登録であることの妥当性検証）

## 他Contextとの依存関係

- User Context（`user`。②は`ユーザー基盤機能_Go移行・設計仕様書.md`）: 生徒アカウント本体の作成を、公開操作`CreateStudentAccount`の直接の呼び出しで依頼する。入力は氏名・氏名カナ・メールアドレス・所属校ID・学年ID・クラスIDで、招待待ち・作成時の招待メール送信は固定（入力に持たない）、生徒番号は`user`が発行する。学年・クラスの実在と整合は、本Contextが検証済みの内容を渡す（`user`②「15. Validation設計」）。生徒・学年・所属校の一覧・詳細の参照は、ページネーション・結合を伴うため、本Context自身が参照モデルを持つ（`user`②「3. Bounded Context」）
- School/Grade Context: 登録先の学年・クラスが教師の所属校に属することの確認に依存する

## 依存する理由

生徒データそのものはUser Contextが正として管理する情報であり、本機能はその情報を教師の権限範囲に応じて参照するビューを提供するに過ぎない。新規登録についても、本Contextが担うのは「同校の教師が、同校の学年・クラスに対して登録しようとしているか」という業務権限・スコープの検証までであり、生徒アカウントそのものの作成（仮パスワード発行・生徒番号採番・招待メール送信）は`user` Contextの`CreateStudentAccount`へ委譲する。Rails現行では、`Teacher::CreateStudentForm`が入力検証・同校妥当性確認を行い、`Student::CreateStudentService`（`Common::CreateUserService`を継承）がアカウントを作成しており、この分担をそのまま維持する。これにより、アカウント作成の詳細ルールが変更された場合の影響範囲を`user`の1箇所に閉じ込め、生徒CSVインポート機能（`student-import`。同じ操作を呼ぶ）との間でロジックが重複・乖離することを防ぐ。本Contextと`student-import`の間に依存関係はない。

---

# 4. 設計パターン

## 採用パターン

Transaction Script

## 判断根拠

本機能は一覧取得・詳細取得を中心とした参照系機能に、生徒アカウントの新規登録が1操作加わったものである。新規登録が加わったことで書き込みが発生するものの、以下の理由からTransaction Scriptの水準を超えないと判断する。

- 一覧・詳細取得の業務ルールは「同校であること」「担当学年権限がある場合は担当学年のみ」という2条件の絞り込みに限定され、状態遷移・状態管理は存在しない
- 新規登録における本Context自身の業務ルールは「指定した学年・クラスが教師の所属校に属し、かつクラスが指定学年に属すること」という同校妥当性検証に限定される。氏名カナ・メール形式の入力検証は本Contextの入力検証（Presentation）が行い、仮パスワード発行・生徒番号採番・招待メール送信といった作成本体の複雑な処理は`user`の`CreateStudentAccount`へ委譲するため、本Context側にはEntityへ振る舞いを集約するほどの複雑さが生まれない
- 新規登録によって「同じデータ構造を複数の関数で扱う」状態にはなるが、それぞれの関数（一覧・詳細・新規登録）が扱う業務ルールは単純な条件判定にとどまり、Active Recordへの格上げの目安（規約3章）である「複雑な業務ルール・状態遷移の発生」には該当しない
- 検索条件構築・ページングおよび登録時の同校妥当性検証という、いずれも手続き的な処理が中心であり、Entityに振る舞いを持たせる必要性が薄い
- 将来的な拡張（検索条件の追加、登録項目の追加）があっても、手続き的な条件追加で十分に対応可能である

以上より、参照系機能に新規登録が加わった現状の複雑さに対しては、引き続きTransaction Scriptが適していると判断する。

## 採用しなかったパターン

### Active Record

新規登録操作が加わったことで作成責務自体は存在するが、その作成処理の実体（仮パスワード発行・生徒番号採番・招待メール送信）は`user`の`CreateStudentAccount`へ委譲するため、本Context側にモデル（struct）へ保存・更新ロジックを持たせる必要がない。同校妥当性検証という単純な条件判定のみであれば、structへ振る舞いを持たせるActive Recordの構造を導入するメリットが薄く、不採用とする。

### Domain Model

生徒Entityは本機能内では状態やライフサイクルを持たず、複数の業務ルールが絡む振る舞いも存在しない。Entityに責務を集約するメリットがなく、DDDを目的化した過剰設計になるため不採用とする。

### Event Sourcing

状態変化や他処理への通知が発生しない参照専用機能であり、過剰設計であるため不採用とする。

---

# 5. Aggregate設計

不要と判断する。

理由: 本機能は`users`への書き込み（作成・更新）を自身では行わない（新規登録の書き込みは`user`の`CreateStudentAccount`が行う）ため、整合性を保証すべき書き込み単位（Aggregate）が存在せず、Aggregateとして境界を定義する必要がない。

---

# 6. Entity設計

## Student

- 役割: 教師が参照・登録する生徒情報を表す概念
- ライフサイクル: 新規登録リクエストとして受け付けられ（本Context側では同校妥当性検証のみ行う）、実際のアカウント生成は`user` Contextの`CreateStudentAccount`に委ねられる。以降の参照は一覧・詳細取得のみで、本機能内での更新・削除は行わない
- 状態変化: 本Context内では状態を持たない（作成後の状態管理はUser Contextの責務）
- 保持する責務: 生徒の基本情報（氏名・かな氏名・学年・所属校）を表現する。新規登録時は入力された氏名・氏名カナ・メールアドレス・学年・クラスを保持し、`user`の`CreateStudentAccount`へ引き渡す
- 判断根拠: 生徒情報の永続的な管理・状態はUser Contextが管理する実体であり、本機能では読み取り専用の表現と、新規登録リクエストの受け皿として扱うため、作成処理そのものの責務は持たせない

## Grade / SchoolClass / HighSchool（参照対象）

- 役割: 絞り込み条件、および新規登録時の同校妥当性検証の判定材料として利用される外部概念
- 判断根拠: 本機能の中心はStudentの参照・登録受付であり、Grade/SchoolClass/HighSchoolは条件判定用の参照情報として扱う。SchoolClassは、生徒単体新規登録（学年・クラス指定）の追加に伴い新たに参照対象として加わった

---

# 7. Value Object設計

## GradeScope

- 採用理由: 「担当学年権限の有無」と「対象学年ID」の組み合わせを検索条件として明示的に扱うため
- 独自ルール:
  - 担当学年権限がない場合は学年による絞り込みを行わない
  - 権限がある場合は、対象学年IDと一致する生徒のみを対象とする
- Entity属性ではなくValue Objectにする理由: この絞り込み条件はStudent Entityの属性ではなく、教師の権限に基づく検索パラメータであるため、Entityとは独立した概念として扱う方が責務が明確になる

## StudentEnrollmentTarget（学年・クラスの組）

- 採用理由: 新規登録時に指定される学年ID・クラスIDの組み合わせは、「教師の所属校に属する学年であること」「クラスがその学年に属すること」という2つの整合性ルールを同時に満たす必要があり、単純な外部キーの集合として扱うべきではないため
- 独自ルール:
  - `grade_id` は登録する教師の所属高校に属する学年でなければならない
  - `school_class_id` は、指定された `grade_id` の学年に属するクラスでなければならない
- Entity属性ではなくValue Objectにする理由: 学年・クラスの組み合わせ妥当性を単体で検証できるようにし、Student Entityの責務を「登録リクエストの保持」に集中させるため。教師教員管理機能におけるGradeScope（担当学年の許容値検証）と役割は類似するが、対象が「閲覧範囲」ではなく「登録先」である点が異なるため別のValue Objectとして扱う

## Value Objectを採用しないもの

- 氏名・かな氏名・メールアドレス: 形式検証（氏名カナのカタカナ形式・メールアドレス形式）は入力検証（Presentation。12章）で行い、文字数・メールアドレスの一意性は`user`の`CreateStudentAccount`が検証するため、本Context内では独自のValue Objectとして持たない

---

# 8. Domain Service

## StudentEnrollmentPolicy

- 責務: 新規登録時に指定された学年ID・クラスIDが、登録する教師の所属校に属し、かつクラスが指定学年に属しているかどうかを判定する
- Entityへ持たせない理由: 判定にはGrade/SchoolClassという、Student Entity単体では保持しない外部情報が必要なため
- 判断根拠: 教師教員管理機能におけるTeacherGradeAssignmentPolicyと同様、同校制約の判定は単一のEntityの責務に寄せると不自然であり、独立したDomain Serviceとして切り出すことで判定ロジックの所在を明確にする

## 追加で必要としないもの

- 一覧・詳細取得の絞り込みロジックはGradeScope（Value Object）とapplication関数内の検索条件組み立てで表現できる範囲に収まり、複数Entityを横断する業務ルールが存在しないため、Domain Serviceとして独立させる必要がない

---

# 9. Repository設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、Repository Interfaceをdomain層に定義しない。以下は永続化・検索責務の設計意図であり、実装時はinfrastructure層の関数として直接実装する(規約: アーキテクチャ規約.md「3. 設計パターンごとの構造適用方針」)。

## StudentRepository

- 管理対象: Student（User Contextの生徒情報への参照）
- 責務:
  - 同校生徒の一覧取得
  - 指定IDの生徒取得
  - 新規生徒アカウント作成の依頼（`user` Contextの`CreateStudentAccount`の呼び出し）
- 保持する検索機能:
  - high_school_idによる絞り込み
  - grade_idによる絞り込み
  - name_kana順ソート
  - ページネーション（1ページあたりの件数を可変指定可能。未指定時のデフォルト件数・上限件数を持つ）
- 保持しない責務:
  - 権限判定そのもの（絞り込み条件の適用は呼び出し元のapplication関数から受け取る）
  - 生徒アカウント作成の実処理（仮パスワード発行・生徒番号採番・招待メール送信は`user`の`CreateStudentAccount`の責務）
- 判断根拠: 検索・参照および作成依頼の受け渡しに特化させ、権限判定ロジックと作成の実処理は、それぞれ呼び出し元のapplication関数・`user`の`CreateStudentAccount`に置くことで責務を分離するため

## GradeRepository / SchoolClassRepository（School/Grade Context提供・参照専用）

- 管理対象: Grade, SchoolClass（参照専用）
- 責務: 新規登録時、指定された学年・クラスが教師の所属校の学年・クラスであり、クラスが指定学年に属することの存在確認
- 保持しない責務: 学年・クラス自体の作成・更新（クラス編成機能の責務）
- 判断根拠: 登録先の妥当性検証に必要な参照確認に限定するため

---

# 10. UseCase設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、UseCase層(struct)を設けない。以下は業務操作の設計意図であり、実装時はapplication層の関数として直接実装する。

## ListStudents

- 目的: 教師が閲覧可能な同校生徒一覧を取得する
- 入力: current teacher, page, per_page
- 出力: 生徒一覧、ページ情報
- トランザクション範囲: 読み取りのみ、トランザクション不要
- 呼び出す関数: 生徒一覧検索関数
- 判断根拠: 権限に基づく絞り込み条件と、リクエストごとに可変のページング条件を組み立てた上で一覧を取得することが主目的であるため

## ShowStudent

- 目的: 指定生徒の詳細を取得する
- 入力: current teacher, student id
- 出力: 生徒詳細情報
- トランザクション範囲: 読み取りのみ
- 呼び出す関数: 生徒詳細取得関数
- 判断根拠: 権限範囲外の生徒へのアクセスを防止しつつ、詳細情報を取得するため

## CreateStudent

- 目的: 教師が同校の生徒アカウントを1件新規登録する
- 入力: current teacher, name, name_kana, email, grade_id, school_class_id
- 出力: 作成結果
- トランザクション範囲: 本Context自身の書き込みはなく、アカウントの保存と招待メール送信依頼の登録は`user`の`CreateStudentAccount`が1つのトランザクションで扱う（呼び出し側がトランザクションを開始していなければ`user`が開始する）。学年・クラスの事前確認は読み取りのみで、このトランザクションには含めない
- 呼び出す関数:
  - GradeRepository / SchoolClassRepository（登録先の学年・クラスが同校であることの存在確認。StudentEnrollmentPolicyの判定材料）
  - StudentRepository（`user`の`CreateStudentAccount`への作成依頼。入力は氏名・氏名カナ・メールアドレス・所属校ID（操作者の所属校）・学年ID・クラスID）
- 判断根拠: StudentEnrollmentPolicyによる同校妥当性検証を行った上で、実際のアカウント生成処理は`user`の`CreateStudentAccount`に委ねることで、本機能側の責務を登録可否の判定と作成依頼の起点に限定するため。Rails現行の`Teacher::CreateStudentForm`→`Student::CreateStudentService`の流れと同じ分担である

---

# 11. Transaction設計

## Transaction開始位置

- ListStudents / ShowStudentでは使用しない
- CreateStudentでは、本Contextはトランザクションを開始しない。`user`の`CreateStudentAccount`が、アカウントの保存と招待メール送信依頼の登録を1つのトランザクションとして開始する

## Transaction終了位置

- CreateStudentでは、`user`の`CreateStudentAccount`が生徒アカウントの作成を完了した時点で、`user`側でコミットする

## 理由

一覧・詳細取得はデータ変更を一切伴わない読み取り専用処理であるため、トランザクション管理は不要である。一方、新規登録は「アカウントは作成されたが招待メール送信の起点となる記録が残らない」といった不整合を避ける必要があるが、本Contextが書き込む対象はなく、アカウントの保存と招待メール送信依頼の登録は`user`の`CreateStudentAccount`が同一トランザクションで扱う（`user`②「14. Transaction設計」）。学年・クラスの同校妥当性検証は読み取りのみの事前確認であり、書き込みとは独立している（Rails現行でも、`Teacher::CreateStudentForm`の検証はトランザクションの外で、`Common::CreateUserService`が自身のトランザクションを持つ）。

---

# 12. Validation設計

## Presentation

- 型チェック: page / per_pageパラメータが整数であることを検証する
- 必須チェック: 詳細取得時のid（route由来）、新規登録時のname / name_kana / email / grade_id / school_class_idを検証する
- フォーマットチェック: 新規登録時のメールアドレス形式、氏名カナのカタカナ形式（Rails現行の`NameValidatable`。`user`は氏名カナの形式を検証しないため、本Contextが行う）、per_pageの上限値（100件）を超えていないかの形式的な検証

## Domain

- 業務ルール: 新規登録時、指定された学年・クラスが登録する教師の所属校に属し、クラスが指定学年に属しているか（StudentEnrollmentPolicy）
- 状態チェック: 該当なし（一覧・詳細取得の絞り込み条件はapplication関数内で組み立てるのみで、入力値そのものに対する業務ルール違反は発生しない）
- 整合性チェック: 新規登録時のメールアドレスが既存アカウントに使用されていないか（`user`の`CreateStudentAccount`の責務。無効化済みを含む全アカウントが対象。本Contextは呼び出しの起点となる）

## 責務分離

- Presentationは「入力形式が正しいか」を担当する
- UseCase/Domainは「権限スコープに基づく絞り込みが正しく適用されているか」「登録先の学年・クラスが同校として妥当か」を担当する

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、teacherロールであることを確認する

## Handler

- ルーティング層でAPIの入口を担当し、認証失敗時のHTTP応答を整える
- 具体的な権限判定は持たせない

## UseCase

- current_userの所属校IDによる絞り込みを適用する（一覧・詳細取得）
- 担当学年権限がある場合はgrade_idによる絞り込みを適用する（一覧・詳細取得）
- 対象生徒が権限範囲外である場合は、存在しない場合と同様に扱う
- 新規登録時、current_userの所属校を基準に登録先の学年・クラスの同校妥当性を検証する。新規登録は担当学年権限の有無を問わず、同校の教師であれば実行できる（Rails現行仕様どおり、教員新規作成のような追加の業務権限チェックは課さない）

## Domain

- GradeScope（Value Object）が一覧・詳細取得の絞り込み条件の妥当性を表現する
- StudentEnrollmentPolicyが新規登録時の学年・クラスの同校妥当性を判定する
- 認可の本体は該当するapplication関数側に寄せ、Domainは条件表現・判定ロジックの提供に留める

## 判断理由

権限のロジック（同校・担当学年）は単純な条件分岐であり、複雑なドメインルールではないため、該当するapplication関数に一元化する。Domain層はその表現をValue Object/Domain Serviceとして提供するに留めることで、責務を過度に分散させない。新規登録に「他職員操作権限」のような追加の業務権限チェックが不要である点は、教師教員管理機能（教員の新規作成には他職員操作権限が必要）との明確な違いであり、Rails現行仕様の権限制御節にもその旨が明記されている。

---

# 14. Error設計

## Domain Error

責務: 業務ルール違反の表現。本機能では、権限範囲外の生徒への直接アクセス試行、新規登録時の学年・クラス同校不整合（StudentEnrollmentPolicy違反）を表現する

## Application Error

責務: ユースケース実行時の失敗を表現する。対象生徒が存在しない、または権限範囲外（学校・学年不一致）である場合の未検出エラーのほか、新規登録時のメールアドレス重複（`user`の`CreateStudentAccount`が返すValidationエラー）を含む

## Infrastructure Error

責務: DB接続失敗等の技術的な障害を表現する

## 判断理由

権限範囲外アクセスと未存在を区別せず「見えない」扱いとすることで、生徒IDの存在自体を第三者に推測させない設計とする（Rails現行仕様と同様、いずれも404として扱う）。業務ルール違反・ユースケース失敗・技術的失敗を分離することで、HTTPレスポンスへの変換を明確にする。新規登録時のメールアドレス重複は、本Context自身が判定するルールではなく`user`の`CreateStudentAccount`からの結果を受け取るものであるため、Domain Errorではなく Application Errorとして扱う。

## エラー仕様

|業務シナリオ|エラー種別|発生層|想定するHTTPステータス|
|-|-|-|-|
|対象生徒が存在しない、または権限範囲外|NotFound（Application Error）|UseCase|404|
|新規登録の学年・クラスが同校でない、またはクラスが指定学年に属さない|Validation（Domain Error）|Domain（StudentEnrollmentPolicy）|422|
|新規登録のメールアドレスが既存アカウント（他校・生徒以外・無効化済みを含む）で使用済み|Validation（Application Error）|Application（`user`の`CreateStudentAccount`からの結果）|422|
|入力形式が不正（必須項目欠如・メール形式不正等）|Validation|Presentation|422|

---

# 15. Domain Event

不要と判断する。

理由: 一覧・詳細取得は参照のみであり、他処理へ通知すべき状態変化が一切発生しない。新規登録についても、招待メール送信は`user`の`CreateStudentAccount`の責務であり、本Context自身が複数の後続処理へ波及する通知の起点になるわけではないため、Domain Eventを導入するメリットが薄い。

---

# 16. API互換方針

## URL

- Rails現行仕様と同じエンドポイントを維持する
  - GET /api/v1/teacher/students
  - GET /api/v1/teacher/students/:id
  - POST /api/v1/teacher/students

## HTTP Method

- 既存仕様どおりに維持する

## Request

- 一覧取得: `page`（ページ番号、未指定時は1ページ目）、`per_page`（1ページあたりの件数。未指定時は10件、最大100件）の意味を維持する
- 新規登録: `name` / `name_kana` / `email` / `grade_id` / `school_class_id` はGo側で入力DTOとして吸収する

## Response

- 一覧: `students`（一覧）+ `meta`（current_page/total_pages/total_count/per_page）の構造を維持する
- 詳細: 生徒の基本情報と関連情報を維持する
- 新規登録: `message`（「生徒の新規作成に成功しました。」相当）を維持する

## Status Code

- 200: 取得成功
- 201: 新規登録成功
- 404: 対象生徒不存在（権限範囲外を含む）
- 422: 入力・業務ルール違反（新規登録時）

## Error Response

- 既存のerrors形式を踏襲する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 参照および新規登録のみの機能であり、既存の users / grades / school_classes / high_schools テーブルで業務要件を満たしているため

---

# 18. テスト戦略

## Domain Test

- 目的: GradeScopeによる絞り込み条件の組み立てロジック、StudentEnrollmentPolicyによる学年・クラスの同校妥当性判定ロジックを検証する

## UseCase Test

- 目的: 権限（同校・担当学年）に応じた一覧・詳細取得の挙動、CreateStudentにおける同校妥当性検証と`user`の`CreateStudentAccount`の呼び出し（入力の受け渡し・メールアドレス重複の結果の変換）の挙動を検証する

## Repository Test

- 目的: 検索条件・可変ページングの正確性、GradeRepository/SchoolClassRepositoryによる存在確認の正確性を検証する

## Handler Test

- 目的: page / per_pageパラメータおよび新規登録の入力バリデーションと、HTTPステータス変換を検証する

## Integration Test

- 目的: エンドポイント経由で一覧・詳細取得が権限どおりに動作すること、新規登録が同校妥当性検証を経て`user`の`CreateStudentAccount`へ連携され、招待待ちの生徒アカウントが作成されることを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller | Handler | HTTP入力の受け取りとレスポンス整形に限定する |
| Query（Teacher::StudentsQuery） | infrastructure関数 + application関数内の検索条件組み立て | 検索条件の構築とスコープ適用を分離する(Transaction Script採用のためusecase層・Repository Interfaceは設けない) |
| Form（Teacher::CreateStudentForm） | Request DTO + Presentation Validation | 入力形式の検証を分離する |
| Form内の同校妥当性検証 | StudentEnrollmentPolicy（Domain Service） | 業務ルールをドメイン層に集約する |
| Service（Student::CreateStudentService, Common::CreateUserService） | `user` Contextの`CreateStudentAccount` | 仮パスワード発行・生徒番号採番・招待メール送信という実処理は`user`が担い、本Context（単体登録）は直接呼ぶ。生徒CSVインポート機能も同じ操作を呼ぶ |
| Serializer（StudentSerializer） | Presenter / Response DTO | レスポンス整形を分離する |
| Model（User） | Entity（Student）+ Repository | データ表現と永続化アクセスの責務を分離する |

---

# 20. 採用しなかった設計

## Active Record

- 採用しなかった理由: 新規登録操作は加わったが、アカウント作成の実処理は`user`の`CreateStudentAccount`へ委譲するため、本Context側のstructに保存・更新ロジックを持たせる必要がない
- 将来的に採用する可能性: 教師による生徒情報の更新（氏名変更等）や生徒メモ登録など、本Context自身が保存・更新ロジックを持つ操作が追加された場合は再検討する余地がある

## Domain Model

- 採用しなかった理由: 生徒Entityが状態・振る舞いを持たず、複雑な業務ルールも存在しないため
- 将来的に採用する可能性: 現時点では想定しない

## Event Sourcing

- 採用しなかった理由: 通知すべき状態変化が存在しないため
- 将来的に採用する可能性: 現時点では想定しない

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Transaction Script | 参照が中心で、新規登録が加わっても本Context自身の業務ルールは同校妥当性検証にとどまり、複雑な状態管理を持たないため |
| Aggregate | 不要 | 整合性を保証すべき書き込み単位（アカウント生成の実処理）は`user`に委譲するため |
| Transaction境界 | 本Contextは開始しない（新規登録のトランザクションは`user`の`CreateStudentAccount`が持つ） | 本Context自身の書き込みがなく、アカウントの保存と招待メール送信依頼の登録の整合性は`user`が保つため |
| Domain Event | 未採用 | 招待メール送信は`user`の`CreateStudentAccount`の責務であり、本Context自身が非同期通知の起点にならないため |
| Value Object | GradeScope（絞り込み）/ StudentEnrollmentTarget（登録先の同校妥当性）を採用 | 権限に基づく絞り込み条件と、登録先学年・クラスの整合性ルールをそれぞれ明示するため |
| Domain Service | StudentEnrollmentPolicy | 学年・クラスの同校妥当性判定がStudent Entity単体の責務に収まらないため |
| Authorization | application関数 + Middleware | ロール確認と業務スコープ判定を分離して管理しやすくするため。新規登録には教員新規作成のような追加の業務権限チェックを課さない（Rails現行仕様どおり） |
| Context間連携 | `user` Contextの`CreateStudentAccount`を直接呼ぶ | 仮パスワード発行・生徒番号採番・招待メール送信のロジックを重複実装しないため。Rails現行の`Teacher::CreateStudentForm`→`Student::CreateStudentService`の流れと整合する |

---

# 設計差分管理

## Rails現行仕様

- Query（Teacher::StudentsQuery）とSerializerがControllerから直接利用され、権限による絞り込みロジックがController/Query間に分散している
- 一覧取得は`per_page`パラメータで1ページあたりの件数を可変に指定でき、未指定時は10件・上限100件である
- 生徒単体新規登録（create）は、Teacher::CreateStudentFormが入力検証・同校妥当性確認を行い、Student::CreateStudentService / Common::CreateUserServiceが、生徒CSVインポート機能の新規行と共通のアカウント作成処理（仮パスワード発行・招待メール送信を含む）を実行する

## Go設計での変更内容

- 絞り込み条件（GradeScope）をapplication関数に集約する
- Repositoryは検索条件の適用（クエリ実行）、および可変ページングの実行のみを担当する
- 新規登録における同校妥当性検証をStudentEnrollmentPolicy（Domain Service）に集約する
- 生徒アカウント作成の実処理（仮パスワード発行・生徒番号採番・招待メール送信）は本Context内に実装せず、`user` Contextの`CreateStudentAccount`を直接呼び出す構成とする

## 変更理由

- 権限ロジックの変更に強い構造とし、テスト容易性を高めるため
- Rails実装が単体登録・CSV一括登録の双方で同じService（Student::CreateStudentService, Common::CreateUserService）を共有している構造を踏襲し、Go設計でもアカウント作成ロジックを`user`に置いて二重実装を避けることで、仮パスワード発行や招待メール送信などのルール変更時の影響範囲を1箇所に限定する

## 影響範囲

- フロントエンドから見た外部APIの互換性は維持する（新規登録エンドポイントの追加を含む）
- 既存DBスキーマは維持するため、データ移行やマイグレーションの追加は不要
- `user`の`CreateStudentAccount`の規則（生徒は常に招待待ち・作成時に招待メール送信、生徒番号の発行、メールアドレス重複の扱い）が、本機能の新規登録処理に適用される。生徒CSVインポート機能（student-import）も同じ操作を呼ぶが、両Contextの間に依存関係はない
