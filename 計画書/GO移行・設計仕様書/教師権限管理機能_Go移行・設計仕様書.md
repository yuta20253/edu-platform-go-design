# 教師権限管理機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

教師が同校教員の権限（学年閲覧範囲 `grade_scope`、他教員管理権限 `manage_other_teachers`）を確認・更新できる機能である。Rails現行仕様では、`manage_other_teachers` 権限を持つ教師のみが更新操作を許可され、さらに「自分自身の権限は更新できない」「校内で最後の有効教員の権限は更新できない」という組織運用上の保護ルールが存在する。

## 利用者

- `teacher` ロールのユーザー（教師）のうち、`manage_other_teachers` 権限を持つ者（更新操作のみ）
- 一覧・詳細参照は同校教員であれば可能

## 業務上の目的

- 教員体制の変化に応じて、権限（閲覧範囲・管理権限）を柔軟に見直せるようにする
- 誤操作や権限剥奪によって「学校に権限管理者が誰もいなくなる」「本人が自分の権限を誤って変更する」といった運用上の事故を防ぐ

---

# 2. 設計方針

- 責務分離: HTTP入出力、入力形式検証、権限更新に伴う組織運用上の保護ルール、永続化を分離する
- 保守性: 「自己更新禁止」「最後の有効教員保護」という業務上重要な不変条件を、UseCase内の条件分岐として散在させず、ドメイン層に集約する
- テスト容易性: 保護ルールを独立した単位でテストできるようにする
- API互換性: 既存フロントエンドとの接続を維持するため、エンドポイント・レスポンス構造を維持する
- 拡張性: 将来的に権限の種類が増える、保護ルールが追加される場合に対応しやすい構造とする

---

# 3. Bounded Context

## Context名

- teacher-permission（教師権限管理コンテキスト）

## Contextの責務

- 教員権限（`grade_scope` / `manage_other_teachers`）の参照・更新
- 権限更新時の組織運用上の保護ルール（自己更新禁止・最後の有効教員保護）の適用
- 権限更新を実行できる教師かどうかの業務的な判定

## 他Contextとの依存関係

- Teacher Directory Context: 更新対象教員が同校に存在するか、着任時に作成された権限レコードの参照に依存する。教員アカウント自体の作成・一覧管理はTeacher Directory Contextの責務であり、本Contextは権限の参照・更新のみを担う
- User Context: current user（更新操作を行う教師）の識別、所属校情報の参照に依存する

## 依存する理由

権限更新の対象となる教員そのものの存在・同校判定は、教員名簿の真正な管理主体であるTeacher Directory Contextに依存する。本Contextは権限（TeacherPermission）そのものの整合性と保護ルールに責務を集中させ、教員アカウントの管理には関与しない。

---

# 4. 設計パターン

## 採用パターン

Domain Model

## 判断根拠

本機能は、単なる `grade_scope` / `manage_other_teachers` の値更新にとどまらず、更新操作そのものに対して以下の業務上の不変条件（invariant）が課されている。

- 更新者は `manage_other_teachers` 権限を持つ教師でなければならない
- 更新者は自分自身の権限を更新できない
- 更新対象が校内で最後の有効教員である場合は更新できない（校内の権限管理者が不在になることを防ぐ）

このうち特に「最後の有効教員保護」は、対象教員1件の属性だけでは判定できず、同校に所属する他の教員（有効な教員の集合）の状態を踏まえて初めて判定できる、複数エンティティにまたがる業務ルールである。これは単純なCRUD検証（値が範囲内か等）とは性質が異なり、組織としての整合性を守るための保護ルールであるため、Active RecordやTransaction Scriptで都度条件分岐を書くよりも、Domain Model（Entity + Domain Service）としてルールを集約した方が、ルールの見落としを防ぎやすく、保守性が高い。

判断基準に照らすと以下のとおりである。

- ドメインロジックの複雑さ: 単一値の検証ではなく、組織内の他教員の状態を踏まえた判定が必要
- 状態管理の有無: `TeacherPermission` は「誰が何を管理できるか」という重要な状態を保持し、その変更には慎重な保護が必要
- 業務ルールの複雑さ: 自己更新禁止・最後の教員保護という、単なる入力検証を超えた組織運用ルールが存在する
- 将来の拡張性: 今後、権限の種類が増えたり、保護ルールが追加された場合にも、ルールをEntity/Domain Serviceに閉じ込めておく方が変更に強い
- テスト容易性: 保護ルールを独立したDomain Serviceとして切り出すことで、DBや他の全業務フローを経由せずにルール単体を検証できる

## 採用しなかったパターン

### Transaction Script

- 自己更新禁止・最後の教員保護という保護ルールをUseCase内の手続きとして書くと、将来ルールが増えた際に条件分岐が積み重なり、可読性・保守性が低下しやすい
- 組織運用上重要なルールであるにもかかわらず、手続きの中に埋没してしまうリスクがある

### Active Record

- 保護ルールの判定に「同校の他教員の状態」という、対象レコード単体を超えた情報が必要であり、モデルの属性検証だけでは自然に表現しにくい
- モデルに保護ルールを持たせると、モデルが他教員の集合を検索する責務まで抱えることになり、責務が肥大化する

### Event Sourcing

- 権限変更の現在値を正しく保護・更新できれば業務要件を満たしており、変更履歴の全再構築という要件は現行仕様上明示されていない
- 監査ログ要件が生じた場合に検討すべきであり、現時点では過剰な設計である

---

# 5. Aggregate設計

## Aggregate Root

- TeacherPermission

## Aggregateに含めるEntity

- TeacherPermission単体（保護ルール判定に必要な「同校の他教員の有効性」は、Aggregate外部の参照情報として扱う）

## Aggregate境界

- TeacherPermission 1件を整合性の単位とする
- 「校内で最後の有効教員かどうか」の判定に必要な他教員の情報は、Aggregateの外部から取得する参照情報（Domain Serviceが受け取る入力）として扱い、TeacherPermission Aggregateには含めない

## 整合性を保証する単位

- 1回の更新操作において、権限者チェック・自己更新禁止チェック・最後の教員保護チェックをすべて満たした上でのみ、TeacherPermissionの更新を確定する

理由: 校内の教員集合全体を1つのAggregateとして扱うと、同校の教員数が多い場合にAggregateが肥大化し、通常の一覧・詳細参照にも影響が及ぶ。保護ルールの判定は「更新時に一時的に必要となる参照情報」であるため、Aggregateの内部に含めず、Domain Serviceが都度参照する設計とする方が、通常時の参照コストを抑えられる。

---

# 6. Entity設計

## TeacherPermission

- 役割: 教師が持つ権限（学年閲覧範囲・他教員管理権限）を表す中心的なドメイン概念
- ライフサイクル: 教員着任時に初期作成される（Teacher Directory Context側の責務） → 本Contextにおいて参照・更新される
- 状態変化: `grade_scope`（own_grade / all_grades）、`manage_other_teachers`（true / false）の値が更新される。名前付きの状態遷移ではないが、値の変化が他機能（お知らせ対象指定の学年制約等）に影響する重要な状態である
- 保持する責務:
  - `grade_scope` が許容値であることを保持・検証する
  - `manage_other_teachers` が真偽値であることを保持する
  - 保有する権限で「他教員を管理できるか」を自ら判定できるようにする
- 判断根拠: 権限そのものの妥当性検証と、自身が持つ権限の意味（他教員管理可否）を問い合わせる責務は、TeacherPermission自身が持つのが自然であるため

---

# 7. Value Object設計

## GradeScope

- 採用理由: `own_grade` / `all_grades` という定義済みの値以外を許容すべきではないため
- 独自ルール:
  - 定義済みの値以外を許容しない
- Entity属性ではなくValue Objectにする理由: 許容値のチェックを型として表現し、教師教員管理機能（着任時の初期権限設定）とも概念を共有できるようにするため（推測: 両機能はTeacherPermissionという同一概念を扱うため、Value Objectとしての定義を揃えておくことが将来の一貫性維持に資すると考えられる）

## ManageOtherTeachersFlag

- 採用理由: 単なる真偽値ではなく、「この値がtrueであることが、他教員の権限更新操作を行える前提条件である」という業務的な意味を持つため
- 独自ルール:
  - true/false以外の値を許容しない
- Entity属性ではなくValue Objectにする理由: 「他教員を管理できるか」という問い合わせを型の振る舞いとして表現し、TeacherPermission Entityおよび後述のDomain Serviceから再利用しやすくするため

## Value Objectを採用しないもの

- 対象教員のID・氏名: 権限管理の対象を識別する情報であり、TeacherPermissionが直接保持するのではなく、対象教員（Teacher Directory Contextが管理するTeacherAccount）への参照として扱う

---

# 8. Domain Service

## TeacherPermissionUpdateGuard

- 責務: 権限更新操作が組織運用上許可されるかどうかを判定する
  - 更新者が `manage_other_teachers` 権限を持つか
  - 更新者が更新対象本人ではないか（自己更新禁止）
  - 更新対象が校内で最後の有効教員でないか（最後の教員保護）
- Entityへ持たせない理由: 判定には更新者自身の権限情報、対象教員の情報、さらに「同校に所属する他の有効教員の集合」というTeacherPermission単体では保持し得ない情報が必要なため。TeacherPermission Entityに持たせると、Entityが他教員の集合を検索する責務まで抱えることになり、責務が肥大化する
- 判断根拠: 自己更新禁止・最後の教員保護という組織運用上の不変条件は、単一のEntityに閉じた判定ではなく、Aggregateをまたいだ集合的な判定であるため、独立したDomain Serviceとして切り出すことで、ルールの所在を明確にし、テスト・変更を容易にする

## 追加で必要としないService

- `grade_scope` / `manage_other_teachers` の値そのものの妥当性検証は、各Value Objectの責務に閉じており、複数Entityをまたぐ判定ではないため、別途Domain Serviceを設ける必要はない

---

# 9. Repository設計

## TeacherPermissionRepository

- 管理対象: TeacherPermission
- 責務:
  - 同校教員の権限一覧取得（氏名カナ順、ページネーション）
  - 指定教員の権限詳細取得
  - 権限の更新
- 保持する検索機能:
  - 高校IDによる絞り込み
  - 有効教員（論理削除されていない）による絞り込み
  - 氏名カナ順のソート
  - ページネーション
- 保持しない責務:
  - 更新可否の判定（TeacherPermissionUpdateGuardの責務）
- 判断根拠: 永続化と検索条件の実行に責務を限定し、保護ルールの判断はDomain Serviceに残すため

## ActiveTeacherCountRepository

- 管理対象: 同校の有効教員集合（参照専用）
- 責務:
  - 指定教員を除いた、校内の有効教員が他に存在するかどうかを確認する
- 保持する検索機能:
  - 高校IDと除外対象教員IDによる有効教員存在確認
- 保持しない責務:
  - 保護ルールそのものの判断（TeacherPermissionUpdateGuardの責務。本Repositoryは判断に必要な事実の取得のみを行う）
- 判断根拠: 「最後の有効教員かどうか」の判定に必要な事実確認をRepositoryの責務とし、判断そのものはDomain Serviceに委ねることで、Repositoryに業務ロジックを持たせない方針を維持するため

---

# 10. UseCase設計

## ListTeacherPermissionsUseCase

- 目的: 同校教員の権限管理対象一覧を取得する
- 入力: current user, page
- 出力: current userの情報、権限一覧、ページ情報
- トランザクション範囲: 読み取りのみ、トランザクションは不要
- 呼び出すRepository:
  - TeacherPermissionRepository
- 判断根拠: 単純な絞り込み・ページネーションのみであるため

## ShowTeacherPermissionUseCase

- 目的: 指定教員の権限詳細を取得する
- 入力: current user, teacher id
- 出力: 教員の権限詳細
- トランザクション範囲: 読み取りのみ
- 呼び出すRepository:
  - TeacherPermissionRepository
- 判断根拠: 同校教員の権限情報を取得する単純な参照処理であるため

## UpdateTeacherPermissionUseCase

- 目的: 指定教員の権限（`grade_scope` / `manage_other_teachers`）を更新する
- 入力: current user, teacher id, grade_scope, manage_other_teachers
- 出力: 更新結果
- トランザクション範囲: TeacherPermissionの更新を1トランザクションで扱う
- 呼び出すRepository:
  - TeacherPermissionRepository（対象取得・更新）
  - ActiveTeacherCountRepository（最後の教員判定に必要な事実確認）
- 判断根拠: TeacherPermissionUpdateGuardによる保護ルール判定を経てから更新を確定する必要があるため

---

# 11. Transaction設計

## Transaction開始位置

- UpdateTeacherPermissionUseCaseの開始時にトランザクションを開始する

## Transaction終了位置

- 保護ルール判定（TeacherPermissionUpdateGuard）を通過し、TeacherPermissionの更新が完了した時点でコミットする
- ListTeacherPermissionsUseCase / ShowTeacherPermissionUseCase ではトランザクションを使用しない

## 理由

- 権限更新という1つの業務操作に対して、保護ルール判定と更新を一貫した単位で扱うため
- UseCase単位で境界を統一し、保守・テストのしやすさを確保するため

---

# 12. Validation設計

## Presentation

- 型チェック: HTTP入力の型（文字列・真偽値）を検証する
- 必須チェック: grade_scope / manage_other_teachers を検証する
- フォーマットチェック: grade_scopeが文字列として送信されていること等の形式検証

## Domain

- 業務ルール: 更新者が `manage_other_teachers` 権限を持つか（TeacherPermissionUpdateGuard）
- 状態チェック: 自己更新でないか、最後の有効教員でないか（TeacherPermissionUpdateGuard）
- 整合性チェック: grade_scopeが許容値であるか（GradeScope Value Object）、manage_other_teachersが真偽値であるか（ManageOtherTeachersFlag Value Object）

## 責務分離

- Presentationは「入力の形式が正しいか」を担当する
- Domainは「業務的に妥当か（更新してよい権限者か、保護対象でないか）」を担当する
- これにより、入力仕様の変更と組織運用ルールの変更を独立して扱える

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、`teacher` ロールであることを確認する

## Handler

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定は持たせない

## UseCase

- current userの所属校IDを用いて、権限一覧・詳細の取得範囲を同校にスコープする
- 更新時、対象教員が同校であることを確認する

## Domain

- TeacherPermissionUpdateGuardにおいて、以下を判定する
  - 更新者が `manage_other_teachers` 権限を持つか
  - 更新者が更新対象本人でないか
  - 更新対象が校内で最後の有効教員でないか

## 判断理由

「同校教員のみを対象とする」という認証・スコープ確認はMiddleware/UseCaseに配置し、「更新してよい権限者か」「保護対象でないか」という組織運用上の判断はDomain（Guard）に集約することで、権限体系や保護ルールの変更が生じてもDomain層のみの修正で対応できるようにする。

---

# 14. Error設計

## Domain Error

責務: 業務ルール違反を表現する

- 更新権限（`manage_other_teachers`）を持たない
- 自分自身の権限を更新しようとした
- 校内で最後の有効教員の権限を更新しようとした
- grade_scopeが許容値でない

## Application Error

責務: ユースケース実行時の失敗を表現する

- 対象教員が存在しない、または同校でない

## Infrastructure Error

責務: DB接続・永続化失敗等の技術的障害を表現する

## 判断理由

自己更新禁止・最後の教員保護・権限不足といった組織運用上の拒否理由は、単なる入力ミスとは性質が異なる業務ルール違反であるため、Domain Errorとして明確に区別する。これにより、Rails現行仕様のように個別の日本語メッセージ（「自分自身は更新できません」等）をControllerに直接書くのではなく、Domain層でルールごとに意味のあるエラーとして表現し、Presentation層でメッセージへ変換する構造とする。

---

# 15. Domain Event

本機能では現時点でDomain Eventを採用しない。

理由: 権限更新に対して、他処理（通知・監査ログ等）への非同期の波及がRails現行仕様上明示されていないためである。

将来的に、権限変更の監査ログを別途記録する要件や、権限変更を対象教員に通知する要件が追加された場合は、「TeacherPermissionChanged」イベントの導入を検討する余地がある（推測）。

---

# 16. API互換方針

## URL

- Rails現行仕様と同じエンドポイントを維持する
  - GET /api/v1/teacher/permissions
  - GET /api/v1/teacher/permissions/:id
  - PATCH /api/v1/teacher/permissions/:id

## HTTP Method

- 既存仕様どおりに維持する

## Request

- `page` はクエリパラメータとして維持する
- `teacher_permission[grade_scope]` / `teacher_permission[manage_other_teachers]` はGo側で入力DTOとして吸収する

## Response

- 一覧・詳細・更新のレスポンス構造はRails現行仕様に近い意味を維持する

## Status Code

- 200: 取得・更新成功
- 422: 権限不足・保護ルール違反・入力不正
- 404: 対象教員不存在

## Error Response

- 既存のerrors形式（`自分自身は更新できません` 等の日本語メッセージ）を踏襲し、フロントエンド互換性を優先する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 現行の `teacher_permissions` テーブルは、権限の参照・更新という業務要件を満たしており、追加のスキーマ変更は不要である
- 「最後の有効教員」判定は `users.deleted_at` による既存の論理削除の仕組みを利用でき、新たなスキーマは不要である

---

# 18. テスト戦略

## Domain Test

- 目的: GradeScope/ManageOtherTeachersFlagの値検証、TeacherPermissionUpdateGuardによる自己更新禁止・最後の教員保護・権限不足の判定を検証する

## UseCase Test

- 目的: ListTeacherPermissionsUseCase / ShowTeacherPermissionUseCase / UpdateTeacherPermissionUseCaseの業務振る舞いを検証する

## Repository Test

- 目的: TeacherPermissionRepositoryによる同校絞り込み・ページネーション、ActiveTeacherCountRepositoryによる有効教員存在確認の正確性を検証する

## Handler Test

- 目的: API入力のバリデーション結果とHTTPステータスの変換を検証する

## Integration Test

- 目的: エンドポイント経由で、正常更新・自己更新拒否・最後の教員保護・権限不足拒否のそれぞれが正しく動作することを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller | Handler | HTTP入出力のみを担当する |
| Controllerの `require_manage_other_teachers!` | TeacherPermissionUpdateGuard（Domain Service） | 権限判定を業務ルールとしてドメイン層に集約する |
| Controllerの `only_active_teacher?` | ActiveTeacherCountRepository + TeacherPermissionUpdateGuard | 事実確認（Repository）と判断（Domain Service）を分離する |
| Form（UpdatePermissionForm） | Request DTO + Presentation Validation + Value Object | 入力形式検証と値の妥当性検証を分離する |
| Query（TeachersQuery） | TeacherPermissionRepository | 検索条件の実行をRepositoryの責務として整理する |
| Serializer | Presenter / Response DTO | レスポンス整形をPresentation層に分離する |
| Model（TeacherPermission） | TeacherPermission Entity + Value Object | 値の妥当性はEntity/Value Objectに、組織運用ルールはDomain Serviceに分離する |

---

# 20. 採用しなかった設計

## Transaction Script

- 採用しなかった理由: 自己更新禁止・最後の教員保護という保護ルールを手続きの中に書くと、将来のルール追加時に条件分岐が積み重なり保守性が低下するため
- 将来的に採用する可能性: 保護ルールが撤廃され単純な値更新のみになった場合は再検討できるが、現状は見込みが低い

## Active Record

- 採用しなかった理由: 保護ルールの判定に「同校の他教員の集合」という単一レコードを超えた情報が必要であり、モデル単体の検証だけでは責務が肥大化するため
- 将来的に採用する可能性: 保護ルールが撤廃された場合は検討の余地がある

## Event Sourcing

- 採用しなかった理由: 権限変更履歴の再構築・監査要件が現時点で存在しないため
- 将来的に採用する可能性: 権限変更の監査ログ要件が生じた場合に有効な可能性がある

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Domain Model | 自己更新禁止・最後の教員保護という、単一Entityを超えた組織運用上の不変条件が存在するため |
| Aggregate | TeacherPermission単体 | 他教員集合の参照は判定時の外部参照情報として扱い、Aggregateを軽量に保つため |
| Transaction境界 | UseCase単位 | 保護ルール判定と更新を1つの業務操作として保証するため |
| Domain Service | TeacherPermissionUpdateGuard | 複数教員にまたがる保護ルールをEntity単体に持たせず、独立した判定単位とするため |
| Domain Event | 未採用 | 現時点で他処理への通知・監査要件がない |
| Value Object | 採用（GradeScope/ManageOtherTeachersFlag） | 許容値・業務的意味を型として明示し再利用するため |

---

# 設計差分管理

## Rails現行仕様

- Controllerのbefore_actionとアクション内条件分岐（`require_manage_other_teachers!`、`only_active_teacher?`、自己更新チェック）に保護ルールが直接実装されている
- 保護ルール違反時のエラーメッセージがController内にハードコードされている

## Go設計での変更内容

- 権限不足・自己更新禁止・最後の教員保護の判定をTeacherPermissionUpdateGuard（Domain Service）に集約する
- 「最後の有効教員かどうか」の事実確認をActiveTeacherCountRepositoryに分離し、判断ロジックと事実確認を分離する
- 入力形式検証をPresentation層、値の妥当性検証をValue Objectに分離する

## 変更理由

- Rails実装ではController内に保護ルールとエラーメッセージが直接書かれており、ルール変更時にHTTP層まで修正が及ぶ
- Go設計では保護ルールをDomain Serviceに集約することで、HTTP層を変更せずに業務ルールのみを修正できるようにし、保守性・テスト容易性を高める

## 影響範囲

- フロントエンドから見たAPI外部仕様（エラーメッセージ含む）は維持する
- 既存DBスキーマは維持するため、データ移行は不要
