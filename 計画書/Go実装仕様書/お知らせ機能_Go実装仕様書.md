# お知らせ機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

生徒が、自分の対象範囲（所属高校・役割・学年・ユーザー単位）に含まれ、かつ`published`状態の公開済みお知らせを一覧・詳細で参照する機能である（②「1. 機能概要」）。お知らせの作成・スケジュール公開（`draft`→`scheduled`→`published`の状態遷移）は、管理者向け機能・非同期ジョブ（`PublishScheduledAnnouncementsJob`相当）の責務であり、本機能（生徒向け参照）のスコープ外である（②「3. Bounded Context」）。

## 採用設計パターンとその理由（②からの要約）

Transaction Script（②「4. 設計パターン」）。

- 一覧取得・詳細取得の2操作のみで構成される参照系機能であり、生徒側で状態を変更する操作が存在しない
- 対象範囲判定は複数条件（所属高校・役割・学年・ユーザー単位）の組み合わせだが、入力（生徒属性）から検索条件を組み立てるだけの処理であり、状態を持つEntityに振る舞いを持たせる必然性がない
- Aggregate・Domain Service・Domain Eventはいずれも不要と判断されている（②「5. Aggregate設計」「8. Domain Service」「15. Domain Event」）

## 本書が対象とする実装範囲

- `internal/announcement`配下における、生徒向けお知らせ一覧取得（`GET /api/v1/student/announcements`）・詳細取得（`GET /api/v1/student/announcements/:id`）の実装
- 上記2操作を実現するapplication層の関数、infrastructure層のDBアクセス関数、presentation層（Handler・Response・Routing）

対象外:

- お知らせの作成・更新・スケジュール公開処理（別機能・非同期ジョブの責務。②「3. Bounded Context」）
- 教師向けお知らせ機能（②「19. Railsとの責務対応」等の記載はannouncement Context全体に触れるが、本書は生徒向け参照機能のみを対象とする）

---

# 2. ディレクトリ構成

- 対象Bounded Context名: `announcement`（②「3. Bounded Context」、ディレクトリ名は規約「5. Bounded Context構成」の命名規則に従い`internal/announcement`）
- ②で採用した設計パターン: Transaction Script（②「4. 設計パターン」）
- 採用パターンに対応する構造: 規約「アーキテクチャ規約.md」4章「Transaction Script」の構造をそのまま適用する。domain層・usecase層・Repository Interfaceは作成しない

## ディレクトリ一覧

```
internal/announcement/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    └── response/
```

## 作成するファイル一覧

```
internal/announcement/application/list_announcements.go
internal/announcement/application/show_announcement.go
internal/announcement/application/query_condition.go
internal/announcement/infrastructure/model.go
internal/announcement/infrastructure/announcement_query.go
internal/announcement/presentation/handler/announcement_handler.go
internal/announcement/presentation/response/announcement_response.go
internal/announcement/presentation/routes.go
```

**②からの補足**: 規約のTransaction Script構造例は`application/list_xxx.go`・`infrastructure/xxx_query.go`の1ファイルずつの例示のみだが、本機能は一覧・詳細の2操作があり、検索条件・GORMモデルを共有するため、`application/query_condition.go`（検索条件structの置き場所）・`infrastructure/model.go`（GORMモデル定義の置き場所）をファイル分割として追加した。`presentation/request/`は作成しない（理由は6節参照）。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層を設けない）。

**②からの補足**: ②「6. Entity設計」「7. Value Object設計」では、Announcement/Publisherを Entity、AnnouncementStatus/AnnouncementAudienceをValue Objectとして設計意図を説明しているが、これは規約「アーキテクチャ規約.md」11章にあるとおり「Domain Model以外のパターンでは概念的な設計意図として書かれており、実装構造への変換は③で行う」ものである。規約4章のTransaction Script構造では「domain層は原則作らない。入力検証が必要な場合も、Value Object等の型は作らず、関数内のガード節で行う」と定められているため、本書では以下のとおり読み替える。

- Announcement / Publisher（②のEntity） → 4節「DTO」のクエリ出力構造体、5節のGORMモデルとして表現する
- AnnouncementStatus（②のValue Object） → 型を作らず、`"published"`という文字列定数をinfrastructure層のクエリ条件内で直接使用する
- AnnouncementAudience（②のValue Object） → 4節・5節で述べる`AnnouncementQueryCondition`という検索条件構造体として表現する（Value Objectとしての検証メソッドは持たせない）

---

# 4. Application層設計

**実装上の位置づけ**: Transaction Script採用のため、usecase層（struct）は設けない。1つの業務操作を1つの関数として`application/`直下に置く（②「10. UseCase設計」の実装上の位置づけと同じ）。

## DTO（Command / Query）

いずれもQueryに属する（本機能は読み取りのみのため）。

- `AnnouncementQueryCondition`（infrastructure層で定義、5節参照）: application関数がcurrent userの属性から組み立て、infrastructure関数へ渡す入力値
- `AnnouncementListResult`（`application/list_announcements.go`）
  - `Announcements []AnnouncementSummary`
  - `CurrentPage int`
  - `TotalPages int`
  - `TotalCount int64`
  - `PerPage int`
- `AnnouncementSummary`（`application/query_condition.go`）
  - `ID uint`
  - `Title string`
  - `Content string`
  - `PublishedAt time.Time`
  - `Publisher PublisherSummary`
- `AnnouncementDetailResult`（`application/show_announcement.go`）
  - `ID uint`
  - `Title string`
  - `Content string`
  - `PublishedAt time.Time`
  - `Publisher PublisherSummary`
- `PublisherSummary`（`application/query_condition.go`）
  - `ID uint`
  - `Name string`
  - `NameKana string`（②「6. Entity設計」Publisher: 「id・氏名・氏名カナの表示用属性」）

## 関数: ListAnnouncements

- 配置: `internal/announcement/application/list_announcements.go`
- シグネチャ: `func ListAnnouncements(ctx context.Context, currentUser CurrentUser, page int) (AnnouncementListResult, error)`
- 処理ステップ（②「10. UseCase設計」ListAnnouncementsを関数単位に具体化。ロジックそのものは書かない）:
  1. `currentUser`の属性（所属高校ID・役割・学年・ユーザーID）から`AnnouncementQueryCondition`を組み立てる
  2. `page`が1未満または未指定の場合のガード節（既定値を用いるか、Presentation側で検証済みの値を受け取る前提とする。9節参照）
  3. `infrastructure.FindPublishedAnnouncements(ctx, db, condition, page, perPage)`を呼び出し、レコード一覧・総件数を取得する
  4. 取得したレコードを`AnnouncementSummary`へ変換し、ページ情報とあわせて`AnnouncementListResult`を組み立てて返す
- 呼び出すinfrastructure関数: `FindPublishedAnnouncements`
- トランザクション境界: なし（読み取りのみ。②「11. Transaction設計」）
- 発生しうるApplication Error: なし（一覧は0件でも正常応答。②に一覧取得固有のエラーの記載はない）

## 関数: ShowAnnouncement

- 配置: `internal/announcement/application/show_announcement.go`
- シグネチャ: `func ShowAnnouncement(ctx context.Context, currentUser CurrentUser, announcementID uint) (AnnouncementDetailResult, error)`
- 処理ステップ（②「10. UseCase設計」ShowAnnouncementを関数単位に具体化）:
  1. `currentUser`の属性から`AnnouncementQueryCondition`を組み立てる
  2. `infrastructure.FindPublishedAnnouncementByID(ctx, db, condition, announcementID)`を呼び出す
  3. レコードが存在しない場合（対象範囲外・`published`以外・ID不存在のいずれも含む）、`ErrAnnouncementNotFound`を返すガード節（②「10. UseCase設計」判断根拠: 「対象範囲外・存在しないお知らせへのアクセスを防ぐため、対象範囲条件を含めた検索が必要」）
  4. 取得したレコードを`AnnouncementDetailResult`へ変換して返す
- 呼び出すinfrastructure関数: `FindPublishedAnnouncementByID`
- トランザクション境界: なし（②「11. Transaction設計」）
- 発生しうるApplication Error: `ErrAnnouncementNotFound`（②「14. Error設計」Domain Error「対象範囲外のお知らせへのアクセス」およびApplication Error「指定お知らせが存在しない場合」に対応。規約「8. 横断的関心事の置き場所」のとおり、Transaction Script採用時はDomain Error層がないため、関数側で発生させるエラーをApplication Error相当として扱う）

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

### FindPublishedAnnouncements

- 配置: `internal/announcement/infrastructure/announcement_query.go`
- シグネチャ: `func FindPublishedAnnouncements(ctx context.Context, db *gorm.DB, cond AnnouncementQueryCondition, page, perPage int) (records []AnnouncementRecord, totalCount int64, err error)`
- 発行するクエリ内容（②「9. Repository設計」の保持する検索機能を列挙。SQL文そのものは書かない）:
  - `status = 'published'`による絞り込み
  - `cond`（所属高校ID・役割・学年・ユーザーID）による対象範囲絞り込み（`announcement_targets`との結合。具体的な結合条件は14節を参照）
  - `published_at`降順・`id`降順によるソート
  - `page`・`perPage`によるページネーション（件数取得は別クエリまたは`COUNT`相当で取得）
  - Publisher情報（id・氏名・氏名カナ）はJoinまたはPreloadで同時に取得する

### FindPublishedAnnouncementByID

- 配置: `internal/announcement/infrastructure/announcement_query.go`
- シグネチャ: `func FindPublishedAnnouncementByID(ctx context.Context, db *gorm.DB, cond AnnouncementQueryCondition, id uint) (AnnouncementRecord, error)`
- 発行するクエリ内容:
  - `id`による単一取得に加え、`status = 'published'`および`cond`による対象範囲条件を同時に適用する（②「10. UseCase設計」ShowAnnouncement判断根拠）
  - 該当レコードが存在しない場合は`gorm.ErrRecordNotFound`を返す（4節でapplication関数が`ErrAnnouncementNotFound`へ変換する）

## GORMモデル（`internal/announcement/infrastructure/model.go`）

- `Announcement`（テーブル: `announcements`）
  - `ID uint`
  - `Title string`
  - `Content string`
  - `Status string`
  - `PublishedAt time.Time`
  - `PublisherID uint`（発行者=Userへの参照。②「6. Entity設計」Publisher）
- `AnnouncementTarget`（テーブル: `announcement_targets`）
  - 所属高校・役割・学年・ユーザー単位の対象条件を保持するレコード（②「9. Repository設計」対象範囲絞り込みに使用する検索対象テーブル。詳細カラムは14節参照）
- `AnnouncementRecord`（infra関数の戻り値。GORMモデルとJoin結果を1レコードに集約した内部表現）
  - `ID uint`, `Title string`, `Content string`, `PublishedAt time.Time`
  - `PublisherID uint`, `PublisherName string`, `PublisherNameKana string`

## AnnouncementQueryCondition（検索条件。`internal/announcement/infrastructure/announcement_query.go`に定義）

- `HighSchoolID uint`
- `Role string`
- `Grade int`
- `UserID uint`

**②からの補足**: ②「7. Value Object設計」のAnnouncementAudienceに相当する検索条件をinfrastructure層に配置した。application層に置くと`application`→`infrastructure`（関数呼び出し）と`infrastructure`→`application`（型参照）の循環importが発生するため、infrastructure層が所有し、application関数が値を組み立てて渡す形とした（規約「アーキテクチャ規約.md」4章のTransaction Script構造には明記がないための実装判断）。

## 外部連携実装

対象外（②にMail・Cache・Queue等の連携は記載されていない）。

---

# 6. Presentation層設計

## Handler

- struct名: `AnnouncementHandler`
- 対応する呼び出し先: application関数（`ListAnnouncements` / `ShowAnnouncement`）
- メソッド一覧:
  - `List(c *gin.Context)`: `GET /api/v1/student/announcements`
  - `Show(c *gin.Context)`: `GET /api/v1/student/announcements/:id`

### List処理順序

1. Middlewareが設定した`current user`（生徒）を`c.Request.Context()`経由で取得する
2. クエリパラメータ`page`を取得し、数値であることを検証する（未指定時は既定ページを使用。9節）
3. `application.ListAnnouncements(ctx, currentUser, page)`を呼び出す
4. 戻り値`AnnouncementListResult`を`AnnouncementListResponse`へ変換する
5. 200 OKで返却する。エラー発生時は11節の変換方針に従いHTTPステータスへ変換する

### Show処理順序

1. `current user`を取得する
2. パスパラメータ`id`を取得し、整数形式であることを検証する（9節）
3. `application.ShowAnnouncement(ctx, currentUser, id)`を呼び出す
4. `ErrAnnouncementNotFound`の場合は404を返す
5. 戻り値`AnnouncementDetailResult`を`AnnouncementResponse`へ変換し200 OKで返却する

Transaction Script採用のためUseCase層を経由しない分、対象範囲判定はapplication関数内（4節）で行われることをHandlerの実装コメント等に明記し、Handler自体には業務権限判定を持たせない（②「13. Authorization設計」Handler節: 「対象範囲判定などの業務権限判定は持たせない」）。

## Request / Response DTO

**Request**: 専用のRequest DTO structは設けない。規約「アーキテクチャ規約.md」4章のTransaction Script構造例は`request/`ディレクトリを含まないため、`page`はクエリパラメータ、`id`はパスパラメータとしてHandler内で直接取得し、`strconv`等での変換とガード節による検証をHandler内で行う方針とする（②からの補足。判断理由は14節参照）。

**Response**（`internal/announcement/presentation/response/announcement_response.go`）:

- `AnnouncementListResponse`
  - `Announcements []AnnouncementResponseItem` (`json:"announcements"`)
  - `Meta PaginationMeta` (`json:"meta"`)
- `AnnouncementResponseItem`
  - `ID uint` (`json:"id"`)
  - `Title string` (`json:"title"`)
  - `Content string` (`json:"content"`)
  - `PublishedAt time.Time` (`json:"published_at"`)
  - `Publisher PublisherResponse` (`json:"publisher"`)
- `PaginationMeta`
  - `CurrentPage int` (`json:"current_page"`)
  - `TotalPages int` (`json:"total_pages"`)
  - `TotalCount int64` (`json:"total_count"`)
  - `PerPage int` (`json:"per_page"`)
- `AnnouncementResponse`（詳細用）
  - `ID uint` (`json:"id"`)
  - `Title string` (`json:"title"`)
  - `Content string` (`json:"content"`)
  - `PublishedAt time.Time` (`json:"published_at"`)
  - `Publisher PublisherResponse` (`json:"publisher"`)
- `PublisherResponse`
  - `ID uint` (`json:"id"`)
  - `Name string` (`json:"name"`)
  - `NameKana string` (`json:"name_kana"`)

（②「16. API互換方針」Response: 「announcements配列（id, title, content, published_at, publisher）+ meta」「id, title, content, published_at, publisher」をそのまま反映）

## Routing（`internal/announcement/presentation/routes.go`）

|Method|Path|Handler|
|-|-|-|
|GET|/api/v1/student/announcements|AnnouncementHandler.List|
|GET|/api/v1/student/announcements/:id|AnnouncementHandler.Show|

両ルートに認証Middleware・studentロールチェックMiddlewareを適用する（②「13. Authorization設計」Middleware節）。

---

# 7. API仕様

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|GET|/api/v1/student/announcements|AnnouncementHandler.List|page（query, 任意）|AnnouncementListResponse|200|
|GET|/api/v1/student/announcements/:id|AnnouncementHandler.Show|id（path, 必須）|AnnouncementResponse|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|指定お知らせが存在しない、または対象範囲外|404|②「16. API互換方針」Status Codeどおり|
|未認証|401|認証Middlewareでの検証失敗。②に明記なし（14節参照）|
|studentロールでない|403|Middlewareでのロールチェック失敗。②に明記なし（14節参照）|
|pageが数値でない／idが整数形式でない|400|②「12. Validation設計」の検証内容に基づくが、ステータスコードは②に明記なし（14節参照）|

---

# 8. Transaction実装方針

- Transaction開始箇所: なし（②「11. Transaction設計」: 「なし（参照のみのため使用しない）」。application関数内でトランザクションを開始しない）
- Transaction終了箇所: 該当なし
- 複数関数・複数クエリにまたがる場合の扱い: `ListAnnouncements`・`ShowAnnouncement`はいずれもinfrastructure関数を1回呼び出すのみで完結するため、複数クエリにまたがる整合性管理は不要である（②「11. Transaction設計」理由: 「読み取り一貫性が必要な場合でも、単一クエリの範囲で完結するため明示的なトランザクション管理は行わない」）

---

# 9. Validation実装方針

## Presentation

- `page`: 数値であることを検証する。数値変換に失敗した場合は400を返すガード節（未指定時は既定ページを使用）
- `id`: パスパラメータの整数形式チェック。変換失敗時は400を返すガード節
- `id`必須チェック: ルーティングにより担保される（パスパラメータのため）

（②「12. Validation設計」Presentation: 「型チェック」「必須チェック」「フォーマットチェック」を反映）

## 業務ルール検証（Transaction Script採用時: application関数内のガード節で検証）

- 対象のお知らせが自分に配信対象となっているか: `AnnouncementQueryCondition`をinfrastructure関数の検索条件に含めることで実現する（②「12. Validation設計」Domain: 「業務ルール: 対象のお知らせが自分に配信対象となっているか」を検索条件として実現）
- statusが`published`であるかどうか: infrastructure関数の検索条件に含める（②「12. Validation設計」Domain: 「状態チェック」）
- 指定IDのお知らせが存在し、かつ対象範囲内であるか: `ShowAnnouncement`関数がinfrastructure関数の結果が空の場合に`ErrAnnouncementNotFound`を返すガード節（②「12. Validation設計」Domain: 「整合性チェック」）

②「12. Validation設計」責務分離のとおり、Presentationは入力形式、application関数（②のDomain相当）は「表示してよいものか」という業務的妥当性を担当する。

---

# 10. Authorization実装方針

- Middleware: 認証済みユーザーを特定しcontextへ保持する。役割がstudentであることを確認する（②「13. Authorization設計」Middleware）
- Handler: APIの入口としてリクエストを受け取り、認証失敗時のHTTP応答を整える。対象範囲判定などの業務権限判定は持たせない（②「13. Authorization設計」Handler）
- application関数（Transaction Script採用のためUseCase相当）: `currentUser`の属性を`AnnouncementQueryCondition`に変換し、infrastructure関数への検索条件として利用する。対象範囲外のお知らせが取得結果に含まれないことを保証する（②「13. Authorization設計」UseCase節を関数単位に読み替え）
- Domain: 該当なし。Announcement自体は対象範囲を判定する振る舞いを持たない。対象範囲判定はapplication関数からinfrastructure関数への条件指定で完結する（②「13. Authorization設計」Domain節および判断根拠をそのまま踏襲）

---

# 11. Error実装方針

- Domain Error → Application Errorへの変換方針: Transaction Script採用のためDomain Error層は存在しない。②「14. Error設計」Domain Error（対象範囲外アクセス）の責務は、application関数が返す`ErrAnnouncementNotFound`がApplication Error相当として引き受ける（規約「アーキテクチャ規約.md」8章の注記に基づく）
- Application Error → HTTPレスポンスへの変換方針: Handlerで下表のとおりステータスコードへ変換する
- Infrastructure Errorのハンドリング方針: infrastructure関数が返す`gorm.ErrRecordNotFound`はapplication関数側で`ErrAnnouncementNotFound`へ変換する。DB接続断等の技術的エラーは`fmt.Errorf`でラップしてapplication・Handlerへ伝播させ、Handlerで500として扱う（②「14. Error設計」Infrastructure Error: 「DB接続失敗など、永続化層の技術的な障害を表現する」）

|Error種別|発生層|HTTP Status|
|-|-|-|
|ErrAnnouncementNotFound（対象範囲外・未存在・非公開）|application|404|
|バリデーションエラー（page/idフォーマット不正）|presentation|400|
|認証エラー|presentation（middleware）|401|
|権限エラー（studentロール以外）|presentation（middleware）|403|
|Infrastructure Error（DB接続失敗等）|infrastructure|500|

Errorレスポンスの形式自体は既存のエラーレスポンス形式を踏襲する（②「16. API互換方針」Error Response）。

---

# 12. GORM / DBクエリ設計

- 利用するGORMモデルとテーブルの対応: `Announcement`↔`announcements`、`AnnouncementTarget`↔`announcement_targets`（②「17. DB設計方針」: 既存Rails DBを継続利用し、Schema変更なし）
- 主要クエリの条件・ソート・ページネーション方針: 5節に記載のとおり（`status = 'published'`、`AnnouncementQueryCondition`による対象範囲絞り込み、`published_at`降順＋`id`降順、`page`／`perPage`によるページング）
- 既存Schemaに対する変更: なし（②「17. DB設計方針」Schema変更有無: 「変更なし」、理由: 「現行の`announcements`・`announcement_targets`テーブルで本機能の要件を満たしており、参照系機能の移行にスキーマ変更は不要であるため」）

SQL文そのものはここに記載しない。

---

# 13. テストケース設計

②「18. テスト戦略」を関数単位に具体化する。Transaction Script採用のため、指示書の読み替え規則に従い「Domain Test」「Repository Test」は対象外とし、「UseCase Test」は「Application関数 Test」と読み替える。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test（UseCase Testの読み替え）

|対象|テストケース|
|-|-|
|ListAnnouncements|対象範囲内かつ`published`のお知らせのみ一覧に含まれること|
|ListAnnouncements|対象範囲外のお知らせが一覧から除外されること|
|ListAnnouncements|`published_at`降順・`id`降順でソートされること|
|ListAnnouncements|ページネーションが正しく機能すること（件数・総ページ数）|
|ShowAnnouncement|対象範囲内かつ`published`のお知らせが取得できること|
|ShowAnnouncement|対象範囲外のお知らせを指定した場合に`ErrAnnouncementNotFound`を返すこと|
|ShowAnnouncement|存在しないIDを指定した場合に`ErrAnnouncementNotFound`を返すこと|
|ShowAnnouncement|`draft`／`scheduled`状態のお知らせを指定した場合に`ErrAnnouncementNotFound`を返すこと|

## Repository Test

対象外（Transaction Script採用のため、Repository Interfaceを設けない）。

**②からの補足**: ②「18. テスト戦略」Repository Testで言及される「検索条件（status・対象範囲・ページング・ソート順）の正確性」の検証観点は、Transaction Script構造ではinfrastructure関数がapplication関数から直接呼び出されるため、上記「Application関数 Test」のテストケース（対象範囲外の除外・ソート順・ページネーション）に統合して担保する。指示書の読み替え規則により本区分は「対象外」と明記するが、②のテスト意図自体は欠落させていない。

## Handler Test

|対象|テストケース|
|-|-|
|List|pageパラメータが不正な場合に400を返すこと|
|List|正常系で一覧が200で返ること|
|Show|idが整数形式でない場合に400を返すこと|
|Show|対象お知らせが存在しない、または対象範囲外の場合に404を返すこと|
|List / Show|未認証の場合に401を返すこと（Middleware経由）|
|List / Show|studentロール以外でアクセスした場合に403を返すこと（Middleware経由）|

## Integration Test

|対象|テストケース|
|-|-|
|GET /api/v1/student/announcements|エンドポイント経由で一覧取得が正常に動作し、対象外お知らせが返却されないこと|
|GET /api/v1/student/announcements/:id|エンドポイント経由で詳細取得が正常に動作し、対象範囲外の場合404が返ること|

（②「18. テスト戦略」Handler Test / Integration Testの目的をそのまま反映）

---

# 14. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容は以下のとおりである。

1. **AnnouncementQueryConditionの配置場所**: ②「7. Value Object設計」のAnnouncementAudienceに相当する検索条件を、application層ではなくinfrastructure層に配置した。application層に置くとinfrastructure⇔application間で循環importが生じるため。規約「アーキテクチャ規約.md」4章のTransaction Script構造には明記がない実装判断であり、推測ではなく規約の依存方向制約からの妥当な帰結である。
2. **CurrentUserの型定義**: ②では「current user」としか記載されておらず、Go実装上の型・所属パッケージが規定されていない。本書では、認証Middlewareが設定する共有の現在ユーザー型（UserID・SchoolID・Role・Grade等の属性を持つ）が`context.Context`経由で取得できるものと仮定した。実際の型定義・取得方法（`shared/`または他Context側の公開関数）は本書の対象外であり、実装時に既存の共通実装を確認する必要がある。**推測**。
3. **対象範囲条件の具体的な結合ロジック**: ②「9. Repository設計」には「AnnouncementAudienceによる対象範囲絞り込み」とのみ記載され、所属高校・役割・学年・ユーザー単位の各条件をAND/ORのどちらで結合するか、またNULL値をワイルドカード（全対象）として扱うか等の具体的なロジックは記載がない。①（Rails実装 `Announcement.for_user`）は本タスクにおいて未提供のため参照不可。本書では検索条件の入力項目と絞り込みが必要であることのみを記載し、具体的な結合ロジックは実装時にRailsの現行実装・スキーマを別途確認して確定する必要があることを明記した。**推測を含む。①未提供のため参照不可**。
4. **GORMモデルの詳細カラム定義**: ①（Railsのマイグレーション定義・スキーマ）が未提供のため参照不可。②「17. DB設計方針」では既存の`announcements`・`announcement_targets`テーブルを変更なしで利用するとのみ記載されているため、本書では②の記述から論理的に導出できるフィールド（title, content, status, published_at, publisher情報、対象範囲条件カラム）のみを示すに留めた。nullable設定・外部キー名等の物理定義は実装時にRailsスキーマを確認して確定する必要がある。**推測。①未提供のため参照不可**。
5. **Request DTOを設けない方針**: 規約「アーキテクチャ規約.md」4章のTransaction Script構造例には`presentation/request/`が含まれないため、`page`・`id`はHandler内で直接パラメータ取得・変換・検証を行う方針とした。②には明記がなく、規約の構造例からの実装判断である。
6. **バリデーション・認証・認可エラーの具体的HTTPステータスコード**: ②「16. API互換方針」のStatus Codeには200・404のみが記載されている。認証エラー（401）・権限エラー（403）・入力形式エラー（400）に対する具体的なステータスコードは②に明記がないため、一般的なREST APIの慣例を採用した。**推測**。
7. **Repository Testの扱い**: 本プロンプトのTransaction Script読み替え規則では「Repository Test」を「対象外」とするが、②「18. テスト戦略」のRepository Testが言及する検索条件（status・対象範囲・ページング・ソート順）の正確性という検証観点を欠落させないため、該当するテストケースを「Application関数 Test」に統合して記載した。判断理由は13節に記載のとおり。
8. **ディレクトリ構成への`query_condition.go`・`model.go`の追加**: 規約のTransaction Script構造例は`application/list_xxx.go`・`infrastructure/xxx_query.go`の1ファイルずつの例示のみだが、本機能は一覧・詳細の2操作間でDTO・検索条件・GORMモデルを共有するため、ファイルを分割した。②には明記がなく、規約の構造方針の範囲内での実装上の分割である。
