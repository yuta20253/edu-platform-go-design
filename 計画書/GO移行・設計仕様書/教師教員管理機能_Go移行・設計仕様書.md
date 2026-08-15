# 教師教員管理機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

教師が同校の教員一覧・詳細を確認し、新規教員アカウントを作成（招待）できる機能である。Rails現行仕様では、教員一覧・詳細の参照に加え、新規教員作成時に `User`（アカウント本体）・`TeacherPermission`（権限）・`TeacherGrade`（担当学年）の3レコードを1つの業務操作として同時に作成している。

## 利用者

- `teacher` ロールのユーザー（教師）

## 業務上の目的

- 教師が同校の教員体制を一覧・詳細から把握できるようにする
- 新規教員をアカウント・権限・担当学年をまとめて矛盾なく登録し、着任準備を整える

---

# 2. 設計方針

- 責務分離: HTTP入出力、入力形式検証、着任時の業務ルール（同校学年制約・氏名形式等）、永続化を分離する
- 保守性: 教員アカウント作成に伴う3レコードの同時作成を1つのUseCaseに集約し、Rails現行仕様のようにFormクラスへロジックが集中しないようにする
- テスト容易性: 一覧・詳細の参照処理と、作成処理の業務ルールをそれぞれ独立して検証できるようにする
- API互換性: 既存フロントエンドとの接続を維持するため、エンドポイント・レスポンス構造を維持する
- 拡張性: 将来的に教員の担当学年が複数になる、招待フローが変わる等の変更に耐えられる構造とする

---

# 3. Bounded Context

## Context名

- teacher-directory（教員名簿・着任管理コンテキスト）

## Contextの責務

- 同校教員の一覧・詳細参照
- 新規教員アカウントの作成（着任時のアカウント・初期権限・担当学年の同時登録）

## 他Contextとの依存関係

- User Context: 教員としてのユーザーアカウント本体（氏名・メールアドレス・ロール）の作成に依存する
- School/Grade Context: 担当学年の存在確認、同校判定に依存する
- Teacher Permission Context: 着任時に初期権限（`grade_scope` / `manage_other_teachers`）を設定する。ただし作成後の権限変更ライフサイクルはTeacher Permission Contextが所有する

## 依存する理由

教員アカウントの作成は、ユーザー登録（User Context）・学年情報（School/Grade Context）という他Contextの情報を前提として成立する。また、初期権限の設定は行うが、以後の権限変更・保護ルール（自己更新禁止・最後の教員保護等）はTeacher Permission Contextの責務であるため、作成時点のみ本Contextが関与し、以降の権限ライフサイクル管理はTeacher Permission Contextへ委譲する。

---

# 4. 設計パターン

## 採用パターン

Active Record

## 判断根拠

本機能が扱う操作は、教員一覧・詳細の参照と、新規教員の作成という、状態遷移を伴わないCRUD中心の業務である。教員アカウント自体に「下書き→有効」のような業務的な状態遷移ルールは存在せず（有効/無効の区別は `deleted_at` による論理削除のみであり、本機能の対象外）、主たる価値はアカウント・権限・担当学年という関連レコードを、業務ルール（氏名形式・同校学年制約等）を満たした上で一貫して作成することにある。

このため、以下の理由からActive Recordを採用する。

- 主要操作が一覧・詳細・作成というCRUD中心の操作であり、複雑な状態遷移が存在しない
- 作成時の業務ルール（氏名カナ形式・メール形式・同校学年制約・grade_scope値チェック）は、入力値の妥当性検証に近く、Entity/値の検証責務として十分に表現できる
- アカウント・権限・担当学年の関連付けは、1回の作成操作内でデータの整合性を保てば足り、複雑なドメイン振る舞いを必要としない

## 採用しなかったパターン

### Transaction Script

- 作成処理には氏名カナ形式・同校学年制約など複数の検証ルールがあり、手続き型に寄せるとUseCase内にルールが埋没し、他の作成系UseCase（他機能）との再利用がしにくくなる
- 検証ルールをEntity/値の責務として持たせた方が、テスト単位を明確にできる

### Domain Model

- 教員アカウント自体に状態遷移や、複数Entityにまたがる複雑な業務ルールの蓄積は現時点で存在しない
- 権限に関する状態的な業務ルール（自己更新禁止・最後の教員保護等）はTeacher Permission Context側で扱うため、本機能側でDomain Modelを導入する必然性が薄い

### Event Sourcing

- 教員作成という単発の操作に対して、履歴の再構築や監査目的のイベント管理は現行仕様上求められていない
- 過剰な設計であるため採用しない

---

# 5. Aggregate設計

## Aggregate Root

- TeacherOnboarding（着任時に作成する一連のレコードをまとめる概念上の作成単位）

## Aggregateに含めるEntity

- TeacherAccount（User相当、教員としての識別情報）
- InitialTeacherPermission（作成時点の初期権限）
- TeacherGradeAssignment（担当学年）

## Aggregate境界

- 新規教員の作成という1回の業務操作の範囲に限定する
- 作成後のTeacherAccountの参照（一覧・詳細）や、InitialTeacherPermissionの以後の変更は、それぞれ別の関心事（参照、Teacher Permission Contextでの更新）として扱い、本Aggregateの範囲外とする

## 整合性を保証する単位

- 教員作成時、アカウント・初期権限・担当学年の3レコードを同時に作成し、一部のみ作成される状態を防ぐ

理由: 教員アカウントは権限・担当学年が揃って初めて意味を持つ着任情報であり、3レコードのいずれかが欠けると業務上不完全な状態になるため、作成時点でのみ整合性を保証するAggregateとして扱う。

---

# 6. Entity設計

## TeacherAccount

- 役割: 同校教員としてのユーザーアカウントを表す
- ライフサイクル: 作成 → 参照（一覧・詳細）。本機能の範囲では更新・削除は扱わない
- 状態変化: 本機能の範囲では状態変化を持たない
- 保持する責務:
  - 氏名・氏名カナ・メールアドレスといった識別情報を保持する
  - 氏名カナがカタカナ形式であることを保持・検証する
- 判断根拠: 教員一覧・詳細・作成の中心となる参照/作成対象であるため

## InitialTeacherPermission

- 役割: 着任時に設定する初期権限（学年閲覧範囲・他教員管理権限）を表す
- ライフサイクル: 教員作成時に1度だけ作成される。以後の変更はTeacher Permission Contextの責務
- 状態変化: 本機能の範囲では持たない（作成のみ）
- 保持する責務:
  - `grade_scope` が許容値であることを保持・検証する
  - `manage_other_teachers` が真偽値であることを保持する
- 判断根拠: 作成時点で不正な権限値が登録されることを防ぐ必要があるため

## TeacherGradeAssignment

- 役割: 教員が担当する学年を表す
- ライフサイクル: 教員作成時に作成される
- 状態変化: 本機能の範囲では持たない
- 保持する責務:
  - 指定学年が発信者と同校であることを前提に紐付けを保持する
- 判断根拠: 担当学年は教員アカウントに従属する情報であり、単独では意味を持たないため

---

# 7. Value Object設計

## NameKana

- 採用理由: 氏名カナは単なる文字列ではなく、カタカナのみで構成されるという形式ルールを持つため
- 独自ルール:
  - カタカナ・長音・中黒・空白のみを許容する
- Entity属性ではなくValue Objectにする理由: 形式検証ロジックを型に閉じ込め、他の作成系機能でも再利用できるようにするため

## Email

- 採用理由: メールアドレスは形式検証が必要な値であり、識別子としても機能するため
- 独自ルール:
  - 一般的なメールアドレス形式であることを検証する
- Entity属性ではなくValue Objectにする理由: 形式検証ロジックをEntityから分離し、他機能（認証等）でも同様の検証ルールを再利用できるようにするため

## GradeScope

- 採用理由: `own_grade` / `all_grades` という許容値が決まっており、任意の整数・文字列を許すべきではないため
- 独自ルール:
  - 定義済みの値以外を許容しない
- Entity属性ではなくValue Objectにする理由: 許容値のチェックを型として表現し、教師権限管理機能（Teacher Permission Context）と概念を共有しやすくするため（推測: 両機能はTeacherPermissionという同一概念を扱うため、Value Objectとしての定義を揃えておくことが将来の一貫性維持に資すると考えられる）

## Value Objectを採用しないもの

- 氏名（漢字表記）: 形式的な制約が明示されておらず、単純な文字列として扱う

---

# 8. Domain Service

## TeacherGradeAssignmentPolicy

- 責務: 指定された学年（grade_id）が、着任させる教師の所属校と一致するかどうかを判定する
- Entityへ持たせない理由: 判定にはTeacherAccount・TeacherGradeAssignment単体の情報だけでなく、School/Grade Contextが管理する学年の所属校情報という外部情報が必要なため
- 判断根拠: 同校制約はTeacherAccount・TeacherGradeAssignmentいずれか一方の責務に寄せると不自然であり、両者の整合性を横断的に判定するDomain Serviceとして切り出すことで、判定ロジックの所在を明確にする

## 追加で必要としないService

- 氏名カナ・メールアドレス・grade_scopeの形式検証は、それぞれValue Objectの責務に閉じており、複数Entityをまたぐ判定ではないため、Domain Serviceを追加で設ける必要はない

---

# 9. Repository設計

**実装上の位置づけ**: 本機能はActive Record採用のため、Repository Interfaceをdomain層に定義しない。以下は永続化・検索責務の設計意図であり、実装時はEntity相当のstructと同一packageに置くStore(例: `〇〇Store`)として直接実装する(規約: アーキテクチャ規約.md「4. 設計パターンごとの構造適用方針」)。

## TeacherDirectoryStore

- 管理対象: TeacherAccount（教員としてのUser）
- 責務:
  - 同校教員の一覧取得（氏名カナ順、ページネーション）
  - 指定教員の詳細取得
  - 新規教員アカウントの作成
- 保持する検索機能:
  - 高校IDによる絞り込み
  - `teacher` ロールによる絞り込み
  - 氏名カナ順のソート
  - ページネーション
- 保持しない責務:
  - 権限・担当学年の管理（それぞれ専用Storeが担当）
  - 作成可否の業務判定
- 判断根拠: 教員アカウントの参照・作成に責務を限定し、権限・担当学年の管理と混在させないため

## TeacherPermissionInitializationStore

- 管理対象: InitialTeacherPermission
- 責務:
  - 着任時の初期権限レコードの作成
- 保持する検索機能:
  - なし（作成専用）
- 保持しない責務:
  - 権限の更新・保護ルール判定（Teacher Permission Contextの責務）
- 判断根拠: 作成時点の初期化のみに責務を限定し、以後の権限管理と役割を分離するため

## TeacherGradeAssignmentStore

- 管理対象: TeacherGradeAssignment
- 責務:
  - 担当学年レコードの作成
  - 指定教員の担当学年取得
- 保持する検索機能:
  - 教員IDによる担当学年取得
- 保持しない責務:
  - 学年の存在確認・同校判定（TeacherGradeAssignmentPolicyおよびGrade参照Storeの責務）
- 判断根拠: 担当学年の永続化に責務を限定するため

## GradeStore（School/Grade Context提供・参照専用）

- 管理対象: Grade（参照専用）
- 責務:
  - 指定学年の存在確認、所属校の取得
- 保持しない責務:
  - 割当可否の最終判断（TeacherGradeAssignmentPolicyの責務）
- 判断根拠: 学年情報の真正な管理はSchool/Grade Contextに残すため

---

# 10. UseCase設計

**実装上の位置づけ**: 本機能はActive Record採用のため、UseCase層(struct)を設けない。以下はHandlerが行う業務操作の設計意図であり、実装時はHandlerがStoreを直接呼び出す処理として実装する。

## ListTeachers(Handler処理)

- 目的: 同校の教員一覧を取得する
- 入力: current user, page
- 出力: current userの情報、教員一覧、ページ情報
- トランザクション範囲: 読み取りのみ、トランザクションは不要
- 呼び出すStore:
  - TeacherDirectoryStore
- 判断根拠: 単純な絞り込み・ページネーションのみであるため

## ShowTeacher(Handler処理)

- 目的: 指定教員の詳細を取得する
- 入力: current user, teacher id
- 出力: 教員詳細（権限・担当学年を含む）
- トランザクション範囲: 読み取りのみ
- 呼び出すStore:
  - TeacherDirectoryStore
  - TeacherGradeAssignmentStore
- 判断根拠: 同校教員の詳細情報を関連情報込みで取得する単純な参照処理であるため

## CreateTeacher(Handler処理)

- 目的: 新規教員アカウントを、初期権限・担当学年とともに作成する
- 入力: current user, name, name_kana, email, grade_id, grade_scope, manage_other_teachers
- 出力: 作成結果
- トランザクション範囲: TeacherAccount・InitialTeacherPermission・TeacherGradeAssignmentの作成を1トランザクションで扱う
- 呼び出すStore:
  - GradeStore（学年の存在・所属校確認）
  - TeacherDirectoryStore（アカウント作成）
  - TeacherPermissionInitializationStore（初期権限作成）
  - TeacherGradeAssignmentStore（担当学年作成）
- 判断根拠: TeacherGradeAssignmentPolicyによる同校判定を踏まえ、3レコードを一貫して作成する必要があるため

---

# 11. Transaction設計

## Transaction開始位置

- CreateTeacherの処理に対応するStoreメソッド内でトランザクションを開始する

## Transaction終了位置

- TeacherAccount・InitialTeacherPermission・TeacherGradeAssignmentの作成が全て完了した時点でコミットする
- ListTeachers / ShowTeacherの処理ではトランザクションを使用しない

## 理由

- 教員着任という1つの業務操作に対して、アカウント・権限・担当学年のいずれかのみが作成される不整合な状態を防ぐため
- Handlerの処理単位で境界を統一し、保守・テストのしやすさを確保するため

---

# 12. Validation設計

## Presentation

- 型チェック: HTTP入力の型（文字列・整数・真偽値）を検証する
- 必須チェック: name / name_kana / email / grade_id / grade_scope / manage_other_teachers を検証する
- フォーマットチェック: メールアドレス形式、氏名カナのカタカナ形式

## Domain

- 業務ルール: 指定学年が同校であるか（TeacherGradeAssignmentPolicy）
- 状態チェック: grade_scopeが許容値であるか（GradeScope Value Object）
- 整合性チェック: manage_other_teachersが真偽値として妥当であるか

## 責務分離

- Presentationは「入力の形式が正しいか」を担当する
- Domainは「業務的に妥当か（同校の学年か、許容される権限値か）」を担当する
- これにより、入力仕様の変更と業務ルールの変更を独立して扱える

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、`teacher` ロールであることを確認する

## Handler

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定は持たせない

## UseCase

- current userの所属校IDを用いて、教員一覧・詳細の取得範囲を同校にスコープする
- 新規教員作成時、指定学年が同校かどうかの判定材料としてcurrent userの所属校を利用する

## Domain

- TeacherGradeAssignmentPolicyにおいて、同校でない学年の割当を拒否する

## 判断理由

「同校教員のみを対象とする」というスコープ制御はHandler側で一貫して適用し、学年割当という具体的な業務ルールはDomain（Policy）側に配置することで、認可のスコープ判定と業務ルール判定を分離する。

---

# 14. Error設計

## Domain Error

責務: 業務ルール違反を表現する

- 指定学年が同校でない
- grade_scopeが許容値でない

## Application Error

責務: ユースケース実行時の失敗を表現する

- 対象教員が存在しない、または同校でない
- 作成処理における関連レコードの不整合

## Infrastructure Error

責務: DB接続・永続化失敗等の技術的障害を表現する

## 判断理由

業務ルール違反（Domain）とリソース未存在（Application）を区別することで、HTTPステータス変換（422 / 404）を一貫した基準で行える。

---

# 15. Domain Event

本機能では現時点でDomain Eventを採用しない。

Rails現行仕様の業務フローには「招待メールを送信して教員アカウントを準備する」という記述があるが、確認した実装（`CreateTeacherForm`）内には明示的なメール送信処理は含まれておらず、具体的な送信タイミング・手段は本資料の調査範囲では特定できなかった（推測: 別途の招待・パスワード再設定フローで扱われている可能性がある）。

そのため、教員作成後のメール送信については、UseCase完了後にInfrastructure層のメール送信アダプタを呼び出す形を基本としつつ、この処理が今後複数の後続処理（監査ログ記録等）に拡張される場合は、「TeacherOnboarded」イベントの導入を再検討する余地がある。ただし現時点では後続処理がメール送信のみと想定されるため、イベント化による間接化のメリットが薄く、採用しない。

---

# 16. API互換方針

## URL

- Rails現行仕様と同じエンドポイントを維持する
  - GET /api/v1/teacher/colleagues
  - GET /api/v1/teacher/colleagues/:id
  - POST /api/v1/teacher/colleagues

## HTTP Method

- 既存仕様どおりに維持する

## Request

- `page` はクエリパラメータとして維持する
- `name` / `name_kana` / `email` / `grade_id` / `grade_scope` / `manage_other_teachers` はGo側で入力DTOとして吸収する

## Response

- 一覧・詳細・作成のレスポンス構造はRails現行仕様に近い意味を維持する

## Status Code

- 200: 取得成功
- 201: 作成成功
- 422: 入力・業務ルール違反
- 404: 対象教員不存在

## Error Response

- 既存のerrors形式を踏襲し、フロントエンド互換性を優先する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 現行の `users` / `teacher_permissions` / `teacher_grades` テーブルは、着任時のアカウント・権限・担当学年登録という業務要件を満たしており、追加のスキーマ変更は不要である

---

# 18. テスト戦略

## Domain Test

- 目的: NameKana/Email/GradeScopeの形式検証、TeacherGradeAssignmentPolicyの同校判定を検証する

## UseCase Test

- 目的: ListTeachersUseCase / ShowTeacherUseCase / CreateTeacherUseCaseの業務振る舞いを検証する

## Repository Test

- 目的: TeacherDirectoryRepositoryによる同校絞り込み・ページネーション、関連Repositoryによる作成処理の正確性を検証する

## Handler Test

- 目的: API入力のバリデーション結果とHTTPステータスの変換を検証する

## Integration Test

- 目的: エンドポイント経由で教員一覧取得・詳細取得・新規作成が正しく連携して動作することを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller | Handler | HTTP入出力のみを担当する |
| Form（CreateTeacherForm） | Request DTO + Presentation Validation + Value Object | 入力形式検証と業務的な値の妥当性検証を分離する |
| Form内のトランザクション処理 | CreateTeacher(Handler処理) + Store | 複数レコードの同時作成をHandler+Storeの責務として明確化する（Active Record採用のためusecase層は設けない） |
| Query（TeachersQuery） | TeacherDirectoryStore | 検索条件の実行をStoreの責務として整理する |
| Serializer | Presenter / Response DTO | レスポンス整形をPresentation層に分離する |
| Model（User, TeacherPermission, TeacherGrade） | TeacherAccount / InitialTeacherPermission / TeacherGradeAssignment（struct）+ 各Store（同一package） | 業務ルールと永続化をstruct/Storeに整理する |

---

# 20. 採用しなかった設計

## Domain Model

- 採用しなかった理由: 教員アカウント自体に状態遷移や複雑な業務ルールの蓄積がなく、CRUD中心の構造で十分に保守可能であるため
- 将来的に採用する可能性: 着任フローが多段階（招待中→アカウント有効化待ち→有効等）の状態遷移を持つようになった場合は再検討の余地がある

## Event Sourcing

- 採用しなかった理由: 着任という単発操作に対する履歴再構築・監査要件が現時点で存在しないため
- 将来的に採用する可能性: 教員の異動・権限変更履歴を追跡する要件が生じた場合に有効な可能性がある

## Transaction Script

- 採用しなかった理由: 作成時の複数の検証ルール（形式・同校制約）を手続きに埋め込むと、再利用性とテスト単位が損なわれるため
- 将来的に採用する可能性: 作成ルールが大幅に単純化された場合は再検討できる

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Active Record | 状態遷移のないCRUD中心の業務であり、値検証中心で十分に表現できるため |
| Aggregate | TeacherOnboarding（作成時のみ） | アカウント・権限・担当学年を同時に作成する整合性単位として必要 |
| Transaction境界 | Handlerの処理単位（作成時のみ、Storeメソッド内） | 3レコードの同時作成という1業務操作の単位と一致するため |
| Domain Event | 未採用 | 後続処理がメール送信程度に限定され、イベント化のメリットが薄いため |
| Value Object | 採用（NameKana/Email/GradeScope） | 形式・許容値のルールを型として明示し再利用するため |
| Domain Service | TeacherGradeAssignmentPolicy | 同校制約の判定がTeacherAccount・TeacherGradeAssignment単体の責務に収まらないため |

---

# 設計差分管理

## Rails現行仕様

- CreateTeacherFormが入力検証・トランザクション制御・User/TeacherPermission/TeacherGrade作成を一括して担っている
- TeachersQueryが検索条件の組み立てを担っている

## Go設計での変更内容

- 入力形式検証をPresentation層、業務的な値検証をValue Object、同校制約の判定をDomain Serviceに分離する
- 複数レコードの同時作成処理をCreateTeacherUseCaseに集約する
- 検索条件の実行をTeacherDirectoryRepositoryに集約する

## 変更理由

- Rails実装ではFormクラスに検証・トランザクション制御・永続化が集中しており、ルール変更時の影響範囲が広い
- Go設計では責務を層ごとに分離することで、変更影響を限定し保守性・テスト容易性を高める

## 影響範囲

- フロントエンドから見たAPI外部仕様は維持する
- 既存DBスキーマは維持するため、データ移行は不要
