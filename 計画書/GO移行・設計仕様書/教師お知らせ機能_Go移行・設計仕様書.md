# 教師お知らせ機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

教師が対象ユーザー向けにお知らせを作成・公開・閲覧できる機能である。Rails現行仕様では、お知らせを下書き（draft）として作成し、公開予約（scheduled）や公開（published）へ状態を進める運用となっている。お知らせには「誰に見せるか」を表す対象（全ユーザー・権限別・学年別・学校別・個人別）を複数指定でき、閲覧側では対象条件に自身が合致するお知らせのみが表示される。

## 利用者

- `teacher` ロールのユーザー（教師）
- 作成者本人のみが自身のお知らせを更新できる

## 業務上の目的

- 教師から生徒・教員などへ向けた情報発信を、下書き・予約・公開という段階を踏んで安全に行う
- 対象を柔軟に指定し、必要な範囲にのみお知らせを届ける
- 誤った状態遷移（例: 公開済みを下書きへ戻す等）を防ぎ、公開情報の一貫性を保つ

---

# 2. 設計方針

- 責務分離: HTTP入出力、入力形式検証、対象指定の業務検証、状態遷移ルール、永続化を明確に分離する
- 保守性: 「draft → scheduled → published」という状態遷移ルールと、公開日時の整合性ルールをドメイン側に集約し、Controller/Formに散らばっている現行ロジックを一箇所に統合する
- テスト容易性: 状態遷移や対象指定ルールをドメイン層で単体テスト可能な形にする
- API互換性: 既存フロントエンドとの接続を維持するため、エンドポイント・リクエスト構造・レスポンス構造は概ね維持する
- 拡張性: 将来的に対象条件の種類が増えた場合や、公開時の通知連携が必要になった場合に対応しやすい構造とする

---

# 3. Bounded Context

## Context名

- announcement（お知らせ管理コンテキスト）

## Contextの責務

- お知らせ本体（タイトル・本文・状態・公開日時）のライフサイクル管理
- お知らせ対象（誰に届けるか）の管理
- 教師が閲覧可能なお知らせの絞り込み
- 自身が作成したお知らせの管理

## 他Contextとの依存関係

- User Context: 発信者（publisher）および閲覧者の識別、`high_school_id` / `user_role_id` / `grade_id` の参照に依存する
- School Context（学年・学校情報）: 対象指定時の学年・学校の存在確認、同校判定に依存する
- Teacher Permission Context: 対象指定時、`own_grade` 権限を持つ教師が自身の学年以外を指定できないようにする制約判定に依存する
- Teacher Dashboard Context: ダッシュボード側から本Contextへ、公開中お知らせの参照が行われる（本Contextが提供する側）

## 依存する理由

お知らせの対象指定・閲覧制御は、教師自身の所属学校・学年・権限情報を前提として成立する。これらの情報はお知らせContextの外側（User/School/Teacher Permission Context）が真正な情報源であるため、お知らせContextはそれらを参照のみ行い、自身では保持しない。

---

# 4. 設計パターン

## 採用パターン

Domain Model

## 判断根拠

お知らせ（Announcement）は、Rails実装において明示的な状態遷移ルール（`draft → scheduled/published`、`scheduled → published`、`published → 遷移不可`）と、状態に付随する整合性ルール（`scheduled` では `scheduled_at` が未来日時であること、`published` 遷移時に `published_at` を自動設定すること）を持つ。これは単なるCRUD対象ではなく、状態そのものに業務的な意味と制約があるエンティティである。

また、対象指定（AnnouncementTarget）についても、「`own_grade` 権限の教師は自身の学年のみ指定できる」「指定先の学年・ユーザーは同校でなければならない」といった、単純な入力チェックでは表現しきれない業務ルールが存在する。これらは複数のエンティティ（発信者の権限、対象の学年・ユーザー）にまたがる判断であり、手続き的に書くとロジックが分散しやすい。

以上より、以下の理由からDomain Modelを採用する。

- Entityが明確な状態（draft/scheduled/published）を持つ
- 状態遷移ルールが存在し、不正な遷移を防ぐ必要がある
- 対象指定に関する複数の業務ルールがAnnouncement/AnnouncementTargetに関連する
- 将来的に対象条件の種類や通知連携などの拡張が見込まれる
- 状態遷移・対象ルールをEntity/Domain Serviceに集約することで、UseCase・Handlerが薄く保守しやすくなる

## 採用しなかったパターン

### Transaction Script

- 状態遷移ルールや対象指定ルールをUseCase内の手続きとして書くと、同種の判定が作成・更新の両方に重複しやすい
- 業務ルールが「今回はこう書いた」という手続きに埋没し、再利用・テストがしにくくなる

### Active Record

- 状態遷移の正しさを保証する責務をモデルの単純な属性更新に任せると、不正な状態遷移が容易に発生しうる
- 対象指定の業務ルール（同校制約・学年スコープ制約）はモデルの属性検証だけでは表現しづらく、責務が肥大化しやすい

### Event Sourcing

- 現行仕様は「現在の状態」を管理すれば業務要件を満たしており、状態遷移の全履歴を再構築する要件は明示されていない
- 監査目的での変更履歴管理が必要になった場合に検討すべきであり、現時点では過剰な設計である

---

# 5. Aggregate設計

## Aggregate Root

- Announcement

## Aggregateに含めるEntity

- Announcement（本体）
- AnnouncementTarget（対象指定、複数件）

## Aggregate境界

- お知らせ本体とその対象指定群を1つの整合性単位として扱う
- 対象指定は必ずお知らせに従属し、単独で存在しない
- 対象先として参照される学年・ユーザー・学校・権限情報はAggregateの外部参照であり、Aggregateには含めない

## 整合性を保証する単位

- 作成時: Announcement本体と対象指定群を同時に整合させる（対象未指定は作成不可）
- 更新時: 状態遷移と公開日時の整合性を1つの単位として保証する

理由: 対象指定はお知らせの一部であり、お知らせ抜きに対象指定だけが存在することは業務上あり得ないため、1つのAggregateとして扱うことでデータの整合性を保証しやすくなる。

---

# 6. Entity設計

## Announcement

- 役割: お知らせ本体を表す中心的なドメイン概念
- ライフサイクル: 作成（draft）→ 予約（scheduled）または直接公開（published）→ 公開（published）
- 状態変化:
  - draft → scheduled
  - draft → published
  - scheduled → published
  - published からは遷移不可
- 保持する責務:
  - タイトル・本文・発信者・状態・公開日時・予約日時を保持する
  - 許可された状態遷移のみを受け付ける
  - `scheduled` 状態では公開予定日時が未来日時であることを保証する
  - `published` へ遷移した際に公開日時を確定させる
- 判断根拠: 状態そのものが業務的な意味を持ち、遷移の正しさを保証する責務が本Entityに強く関連するため

## AnnouncementTarget

- 役割: お知らせの対象条件（誰に届けるか）を表す概念
- ライフサイクル: お知らせ作成時に登録され、以降は変更されない（現行仕様では対象の更新APIはない）
- 状態変化: なし（作成のみ）
- 保持する責務:
  - 対象種別（全ユーザー/権限別/学年別/学校別/個人別）と、それに応じた参照先（学年・権限・学校・ユーザー）を保持する
  - 対象種別に応じて必要な参照先が過不足なく設定されていることを保持する
- 判断根拠: 対象種別ごとに必要な参照先が異なり、単なる外部キーの集合ではなく業務的な組み合わせルールを持つため

## 参照専用の外部概念（Grade / User / TeacherPermission / HighSchool）

- 役割: 対象指定や閲覧制御の判定材料として参照する
- 判断根拠: これらは他Contextが真正な管理主体であり、お知らせContextでは書き込みを行わない参照専用の情報として扱う

---

# 7. Value Object設計

## AnnouncementStatus

- 採用理由: 状態を文字列のまま扱うと、許可されない遷移（例: published → draft）が実装のあちこちで再チェックされ、抜け漏れが起きやすいため
- 独自ルール:
  - draft / scheduled / published のいずれかのみ許容する
  - 遷移可能な状態の組み合わせ（draft→scheduled/published、scheduled→published）のみ許可する
- Entity属性ではなくValue Objectにする理由: 状態遷移という業務ルールそのものを型として表現し、Announcement Entityから遷移判定ロジックを再利用しやすくするため

## ScheduledAt / PublishedAt

- 採用理由: 単なる日時ではなく、「予約時は未来日時が必須」「公開時は自動確定される」という業務ルールを伴うため
- 独自ルール:
  - `scheduled` 状態のときのみ未来日時であることを検証する
  - `published` への遷移時に未設定であれば現在時刻を設定する
- Entity属性ではなくValue Objectにする理由: 日時の妥当性判定ロジックを型に閉じ込め、Entity本体の判断ロジックを簡潔に保つため

## TargetCriteria（対象種別＋参照先の組）

- 採用理由: 対象種別（all_users/by_role/by_grade/by_school/by_user）ごとに必要な参照先の組み合わせが異なり、誤った組み合わせを防ぐ必要があるため
- 独自ルール:
  - `by_role` / `by_grade` は権限（user_role_id）の指定を要する
  - `by_grade` は学年（grade_id）の指定を要する
  - `by_user` はユーザー（user_id）の指定を要する
  - `by_school` は発信者の所属校を参照する
- Entity属性ではなくValue Objectにする理由: 種別と参照先の整合性を単体で検証できるようにし、AnnouncementTarget Entityの責務を「保持」に集中させるため

## Value Objectを採用しないもの

- タイトル・本文: 文字列としての意味が強く、独自の業務ルールを持たないため、Value Object化は不要とする

---

# 8. Domain Service

## AnnouncementTargetingPolicy

- 責務: 対象指定（TargetCriteria群）が、発信者の権限・所属校を踏まえて妥当かどうかを判定する
  - `own_grade` 権限の教師は、自身の学年以外を `by_grade` で指定できない
  - `by_grade` で指定する学年、`by_user` で指定するユーザーは、発信者と同校でなければならない
- Entityへ持たせない理由: 判定には発信者の権限情報（Teacher Permission Context）、参照先の学年・ユーザー情報（School/User Context）という、AnnouncementもAnnouncementTargetも単体では保持しない外部情報が必要なため
- 判断根拠: 複数Context・複数Entityにまたがる業務ルールをUseCaseに直接書くと、作成処理に業務ロジックが埋没するため、独立したDomain Serviceとして切り出し、テストと再利用を容易にする

## AnnouncementVisibilityPolicy

- 責務: 教師が閲覧可能なお知らせかどうか（対象条件に自身が合致するか）を判定するための条件を表現する
- Entityへ持たせない理由: 判定は「閲覧者側の属性」と「お知らせ側の対象条件」の突き合わせであり、Announcement Entity単体の責務ではなく、検索条件として扱う方が自然なため
- 判断根拠: 現行のRails実装では `for_user` スコープとしてSQL条件に変換されており、Goでも同様に検索条件（Repositoryへの問い合わせ条件）として表現する方が、大量データに対する効率的な絞り込みと整合する。したがってこのポリシーはドメインルールの「意味」を定義する役割にとどめ、実際の絞り込み実行はRepositoryに委ねる

---

# 9. Repository設計

## AnnouncementRepository

- 管理対象: Announcement, AnnouncementTarget
- 責務:
  - 教師が閲覧可能な公開中お知らせの検索（対象条件による絞り込み）
  - 自身が作成したお知らせの検索
  - お知らせ本体と対象指定群の作成
  - お知らせの状態・公開予定日時の更新
- 保持する検索機能:
  - 閲覧可能条件による絞り込み（AnnouncementVisibilityPolicyの条件を反映）
  - 作成者による絞り込み（authoredタブ）
  - 公開状態による絞り込み
  - 公開日時降順のページネーション
  - 上位N件取得（ダッシュボード向け）
- 保持しない責務:
  - 状態遷移の可否判定
  - 対象指定の妥当性判定
- 判断根拠: 永続化と検索条件の実行に責務を限定し、業務ルールの判断はEntity/Domain Serviceに残すため

## 外部参照Repository（Grade / User / UserRole / TeacherPermission）

- 管理対象: 各Contextが所有するエンティティ（お知らせContextからは参照のみ）
- 責務:
  - 対象指定の妥当性判定に必要な存在確認・同校確認・権限確認のための参照を提供する
- 保持しない責務:
  - 対象指定の可否そのものの判断（AnnouncementTargetingPolicyが担う）
- 判断根拠: 他Contextの所有物に対する書き込みを行わず、参照のみに責務を限定するため

---

# 10. UseCase設計

## ListAnnouncementsUseCase

- 目的: `authored` タブでは自身が作成したお知らせ一覧、それ以外では閲覧可能な公開中お知らせ一覧を取得する
- 入力: current user, tab, page
- 出力: お知らせ一覧とページ情報
- トランザクション範囲: 読み取りのみ、トランザクションは不要
- 呼び出すRepository:
  - AnnouncementRepository
- 判断根拠: 単一条件での検索・ページネーションのみであり、書き込みを伴わないため

## ShowAnnouncementUseCase

- 目的: 指定お知らせの詳細を取得する
- 入力: current user, announcement id
- 出力: お知らせ詳細（対象情報・発信者情報を含む）
- トランザクション範囲: 読み取りのみ
- 呼び出すRepository:
  - AnnouncementRepository
- 判断根拠: 公開中かつ閲覧可能なお知らせのみを対象とする単純な参照処理であるため

## CreateAnnouncementUseCase

- 目的: お知らせを下書きとして作成し、対象指定を登録する
- 入力: current user, title, content, target criteria list
- 出力: 作成結果
- トランザクション範囲: Announcement本体とAnnouncementTarget群の作成を1トランザクションで扱う
- 呼び出すRepository:
  - AnnouncementRepository（作成）
  - Grade / User / UserRole 参照Repository（対象妥当性判定のための存在確認・同校確認）
- 判断根拠: AnnouncementTargetingPolicyによる判定結果を踏まえ、お知らせ本体と対象指定を一貫して作成する必要があるため

## UpdateAnnouncementStatusUseCase

- 目的: 自身が作成したお知らせの状態・公開予定日時を更新する
- 入力: current user, announcement id, 更新後status, scheduled_at
- 出力: 更新結果
- トランザクション範囲: Announcementの状態更新を1トランザクションで扱う
- 呼び出すRepository:
  - AnnouncementRepository
- 判断根拠: 状態遷移の可否はAnnouncement Entityが判定し、UseCaseは所有者確認と永続化のみを担当するため

---

# 11. Transaction設計

## Transaction開始位置

- UseCaseの開始時にトランザクションを開始する（書き込みを伴うUseCaseのみ）

## Transaction終了位置

- CreateAnnouncementUseCase では、Announcement本体と全対象指定の作成が完了した時点でコミットする
- UpdateAnnouncementStatusUseCase では、状態更新が完了した時点でコミットする
- ListAnnouncementsUseCase / ShowAnnouncementUseCase ではトランザクションを使用しない

## 理由

- 1回の業務操作（作成・更新）に対して、お知らせと対象指定の整合性を保つため
- UseCase単位で境界を統一することで、テスト・保守のしやすさを確保するため

---

# 12. Validation設計

## Presentation

- 型チェック: HTTP入力の型（文字列・配列・真偽値等）を検証する
- 必須チェック: title / content / announcement_targets（配列かつ非空）を検証する
- フォーマットチェック: target_typeが定義済みの値であるかの形式的な検証、scheduled_atの日付形式検証

## Domain

- 業務ルール: `own_grade` 権限の教師が自身の学年以外を指定していないか、対象の学年・ユーザーが同校かどうか（AnnouncementTargetingPolicy）
- 状態チェック: 状態遷移が許可された組み合わせかどうか（Announcement Entity）
- 整合性チェック: `scheduled` 状態での公開予定日時の未来日時チェック、`published` 遷移時の公開日時確定

## 責務分離

- Presentationは「入力の形式が正しいか」を担当する
- Domainは「業務的に妥当か（誰に配信してよいか、状態を進めてよいか）」を担当する
- これにより、フロントエンドの入力仕様変更と業務ルール変更を独立して扱える

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、`teacher` ロールであることを確認する

## Handler

- ルーティングとHTTP入出力の変換のみを担当し、業務権限判定は持たせない

## UseCase

- `authored` タブでは自身が作成したお知らせのみを対象とする
- 更新は自身が作成したお知らせのみに限定する（所有者確認）

## Domain

- AnnouncementTargetingPolicyにおいて、`own_grade` 権限の教師が指定できる対象範囲を制限する
- Announcement Entityにおいて、許可されない状態遷移を拒否する

## 判断理由

「誰がアクセスできるか」という認証・ロール確認はMiddlewareに、「自分のリソースか」という所有権確認はUseCaseに、「業務上許される操作か」という判定はDomainに配置することで、権限体系の変更に対して影響範囲を限定できる。

---

# 14. Error設計

## Domain Error

責務: 業務ルール違反を表現する

- 不正な状態遷移
- 対象指定の権限逸脱（own_grade制約違反）
- 対象指定の同校制約違反
- 予約日時が未来日時でない

## Application Error

責務: ユースケース実行時の失敗を表現する

- 対象お知らせが存在しない、または閲覧不可
- 自身が作成したお知らせでない（更新時）
- 参照先（学年・ユーザー等）が存在しない

## Infrastructure Error

責務: DB接続・永続化・外部Context参照時の技術的失敗を表現する

## 判断理由

業務ルール違反（Domain）とリソース未存在・所有権違反（Application）を区別することで、HTTPステータス変換（422 / 404）を一貫した基準で行える。

---

# 15. Domain Event

本機能では現時点でDomain Eventを採用しない。理由は、お知らせの作成・状態変更に対して、他処理（通知送信等）への非同期の波及がRails現行仕様上明示されていないためである。

将来的に、公開時に対象ユーザーへの通知（プッシュ通知・メール等）を送る要件が追加された場合は、「AnnouncementPublished」イベントの導入を検討する余地がある（推測）。

---

# 16. API互換方針

## URL

- Rails現行仕様と同じエンドポイントを維持する
  - GET /api/v1/teacher/announcements
  - GET /api/v1/teacher/announcements/:id
  - POST /api/v1/teacher/announcements
  - PATCH /api/v1/teacher/announcements/:id

## HTTP Method

- 既存仕様どおりに維持する

## Request

- `announcement[title]` / `announcement[content]` / `announcement[announcement_targets]` の意味は維持しつつ、Go側では入力DTOとして吸収する
- `tab` / `page` はクエリパラメータとして維持する

## Response

- 一覧・詳細・作成・更新のレスポンス構造はRails現行仕様に近い意味を維持する
- `AnnouncementSerializer` / `AuthoredAnnouncementSerializer` に相当するレスポンス整形をPresentation層で行う

## Status Code

- 200: 取得・更新成功
- 422: 入力・業務ルール違反
- 404: 対象お知らせ不存在または閲覧不可

## Error Response

- 既存のerrors形式を踏襲し、フロントエンド互換性を優先する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 現行の `announcements` / `announcement_targets` テーブルは、状態遷移・対象指定という業務要件を満たしており、追加のスキーマ変更は不要である
- 既存データとの整合性を維持する方が安全である

---

# 18. テスト戦略

## Domain Test

- 目的: Announcementの状態遷移ルール、公開日時の整合性、AnnouncementTargetingPolicyの対象妥当性判定を検証する

## UseCase Test

- 目的: ListAnnouncementsUseCase / ShowAnnouncementUseCase / CreateAnnouncementUseCase / UpdateAnnouncementStatusUseCase の業務振る舞いを検証する

## Repository Test

- 目的: AnnouncementRepositoryによる閲覧可能条件の絞り込み、作成者絞り込み、ページネーションの正確性を検証する

## Handler Test

- 目的: API入力のバリデーション結果とHTTPステータスの変換を検証する

## Integration Test

- 目的: エンドポイント経由での作成・状態更新・一覧取得が正しく連携して動作することを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller | Handler | HTTP入出力のみを担当する |
| Form（CreateAnnouncementForm） | Request DTO + Presentation Validation | 入力形式の検証を分離する |
| Form内の業務ルール検証（own_grade/同校チェック） | AnnouncementTargetingPolicy（Domain Service） | 業務ルールをドメイン層に集約する |
| Service（CreateAnnouncementService） | UseCase | 作成処理の起点として扱う |
| Model（Announcement）の状態遷移バリデーション | Announcement Entity | 状態遷移ルールをEntityの責務として明確化する |
| Model（AnnouncementTarget） | AnnouncementTarget Entity | 対象指定の保持責務として整理する |
| Scope（for_user） | AnnouncementVisibilityPolicy + Repository検索条件 | 閲覧条件の意味をドメインで定義し、実行はRepositoryに委ねる |
| Serializer | Presenter / Response DTO | レスポンス整形をPresentation層に分離する |

---

# 20. 採用しなかった設計

## Transaction Script

- 採用しなかった理由: 状態遷移と対象指定ルールが複数の判定にまたがり、手続き型で書くとロジックが重複・分散しやすいため
- 将来的に採用する可能性: 機能が大幅に単純化された場合には再検討できるが、現状は見込みが低い

## Active Record

- 採用しなかった理由: 状態遷移の妥当性・対象指定の業務ルールをモデル属性の検証だけに任せると、責務が肥大化し保守性が低下するため
- 将来的に採用する可能性: 状態・対象ルールが撤廃され単純な掲示板的機能になった場合は検討の余地がある

## Event Sourcing

- 採用しなかった理由: 状態遷移の履歴管理・監査要件が現時点で存在しないため
- 将来的に採用する可能性: 公開履歴の監査要件が生じた場合に有効な可能性がある

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Domain Model | 状態遷移ルールと対象指定の業務ルールが明確に存在するため |
| Aggregate | Announcement + AnnouncementTarget | 対象指定はお知らせに従属し、単独では存在しないため |
| Transaction境界 | UseCase単位 | 作成・更新それぞれが1つの整合性単位であるため |
| Domain Service | AnnouncementTargetingPolicy / AnnouncementVisibilityPolicy | 複数Context・複数Entityにまたがる判定を分離するため |
| Domain Event | 未採用 | 現時点で他処理への非同期通知要件がない |
| Value Object | 採用（Status/日時/対象条件） | 状態遷移と対象条件の妥当性を型として表現するため |
| Authorization | Middleware + UseCase + Domain | 認証・所有権確認・業務ルール判定を層ごとに分離するため |

---

# 設計差分管理

## Rails現行仕様

- CreateAnnouncementForm（入力検証＋一部業務ルール）とCreateAnnouncementService（永続化）にロジックが分散している
- Announcementモデル内に状態遷移バリデーションが実装されている
- 対象指定の妥当性判定（own_grade制約・同校制約）がForm内のprivateメソッドとして実装されている

## Go設計での変更内容

- 入力形式検証をPresentation層に寄せる
- 対象指定の業務ルール判定をAnnouncementTargetingPolicy（Domain Service）に集約する
- 状態遷移ルールをAnnouncement Entityの責務として明確化する
- 永続化と検索条件の実行をAnnouncementRepositoryに集約する

## 変更理由

- Rails実装ではForm/Service/Modelに業務ルールが分散しており、ルール変更時の影響範囲が把握しにくい
- Go設計では、状態遷移はEntity、対象指定の妥当性はDomain Serviceに責務を集約することで、業務ルール変更時の修正箇所を明確にし、保守性・テスト容易性を高める

## 影響範囲

- フロントエンドから見たAPI外部仕様は維持する
- 既存DBスキーマは維持するため、データ移行は不要
