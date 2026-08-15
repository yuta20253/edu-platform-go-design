# 教師ダッシュボード機能 Go実装仕様書

---

# 1. 機能概要

## 機能概要

教師のホーム画面に、同校の学年別生徒数と、教師向けに公開中のお知らせ最新5件を表示する機能である。単一のエンドポイント（`GET /api/v1/teacher/dashboard`）で、学年別生徒数集計とお知らせ取得という2つの読み取り処理の結果をまとめて返却する。

## 採用設計パターンとその理由（②からの要約）

- 採用パターン: **Transaction Script**（②「4. 設計パターン」）
- 理由:
  - 本機能は書き込みを一切伴わない「同校生徒数の学年別集計」と「公開中お知らせの上位5件取得」という2つの読み取り処理を1レスポンスに合成するだけであり、自ら保持・管理する状態（Entity）を持たない
  - 生徒数集計対象・お知らせ閲覧条件は、いずれも参照先Context（Student/School Context、Announcement Context）が持つ既存ルールをそのまま利用するのみであり、本機能固有の業務ルールを持たない
  - 将来の表示項目追加も、application層の関数内に参照処理を追加するだけで対応できる
  - 単一の関数に対する入出力テストで十分に検証できる

②はActive Record（永続化対象Entityが存在しない）、Domain Model（状態遷移・複数業務ルールを持つEntityが存在しない）、Event Sourcing（状態変更・履歴再構築の概念が存在しない）をいずれも採用しなかった。

## 本書が対象とする実装範囲

- `internal/teacher_dashboard` 配下の application 関数・infrastructure 関数・presentation層（Handler・Response DTO・Routing）の実装単位
- Student/School Context、Announcement Context側の実装（生徒データ・お知らせデータの真正な管理）は本書の対象外。本書ではteacher_dashboard側から見た呼び出し方針のみを記載する
- ①Rails実装の詳細は本セッションでは提供されていないため、「①未提供のため参照不可」として扱い、Railsの実際のコードやAPIレスポンスのJSONキー名等は本書では推測に留める

---

# 2. ディレクトリ構成

## 対象Bounded Context名

- `teacher-dashboard`（②「3. Bounded Context」）
- `internal/`配下のディレクトリ名: `teacher_dashboard`（アーキテクチャ規約「5. Bounded Context構成」命名規則に基づき、kebab-caseのContext名をスネークケースのディレクトリ名に変換）

## ②で採用した設計パターン

Transaction Script

アーキテクチャ規約「4. 設計パターンごとの構造適用方針」に従い、domain層・usecase層・Repository Interfaceは設けない。

## 作成するディレクトリ一覧

```
internal/teacher_dashboard/
├── application/
├── infrastructure/
└── presentation/
    ├── handler/
    ├── response/
    └── (routes.go はpresentation直下)
```

## 作成するファイル一覧

```
internal/teacher_dashboard/application/show_teacher_dashboard.go
internal/teacher_dashboard/infrastructure/student_grade_statistics_query.go
internal/teacher_dashboard/infrastructure/announcement_query.go
internal/teacher_dashboard/presentation/handler/teacher_dashboard_handler.go
internal/teacher_dashboard/presentation/response/teacher_dashboard_response.go
internal/teacher_dashboard/presentation/routes.go
```

本機能はRequestパラメータを取らない（②「16. API互換方針」）ため、`presentation/request/`は作成しない。

---

# 3. Domain層設計

対象外（Transaction Script採用のため、Domain層を設けない。アーキテクチャ規約「4. 設計パターンごとの構造適用方針」）。

Entity・Repository Interface・Domain Service・Domain Event・Domain Errorのいずれも設けない。②で定義された内容の実装上の扱いは以下のとおり読み替える。

- ②「6. Entity設計」（独自Entityを持たない） → 該当構造なし。生徒数集計・お知らせ一覧は表示専用の合成結果として、application層のDTOで表現する（4章参照）
- ②「7. Value Object設計」の`GradeStudentCounts`（学年別生徒数の組） → Transaction Script採用のためValue Objectという型は作らず、application層のDTO（struct）として実装する（4章参照。②からの補足）
- ②「8. Domain Service」（設けない） → 該当構造なし。集計・取得処理はapplication関数内で完結させる
- ②「9. Repository設計」の`StudentStatisticsRepository`・`AnnouncementRepository` → domain層のInterfaceとしては定義せず、infrastructure層の関数として直接実装する（②「9. Repository設計」の「実装上の位置づけ」に明記済み。5章参照）

業務処理の入出力・処理ステップは「4. Application層設計」に記載する。

---

# 4. Application層設計

②「10. UseCase設計」の「実装上の位置づけ」に基づき、UseCase struct/interfaceは設けず、`application/`直下に置く関数として実装する。

## DTO

| struct名 | フィールドと型 | 属性 |
|-|-|-|
| `GradeStudentCounts` | `Grade1 int`, `Grade2 int`, `Grade3 int` | Query（出力）。②「7. Value Object設計」のGradeStudentCountsを、Transaction Script実装ではdomain層のVOではなくapplication層のDTOとして配置する（②からの補足）。各フィールドは0以上の整数（②の独自ルールを維持） |
| `ShowTeacherDashboardInput` | `SchoolID uint`（current userの所属校ID） | Query（入力）。current userそのものをapplication層まで引き回さず、Handlerで所属校IDを取り出して渡す方針とする（②に明記がないため実装判断。②からの補足・推測） |
| `ShowTeacherDashboardOutput` | `Stats GradeStudentCounts`, `Announcements []AnnouncementSummary` | Query（出力） |
| `AnnouncementSummary` | Announcement Context側の一覧表示用の型をそのまま利用する（②「7. Value Object設計」：お知らせ一覧は独自VOとして再定義しない） | Query（出力）。具体的なフィールド構成はAnnouncement Context側の②③文書の定義に従う。本書では対象外とし、フィールド詳細を規定しない（②からの補足） |

## UseCase（application関数として実装）

### ShowTeacherDashboard

- 関数シグネチャ: `func ShowTeacherDashboard(ctx context.Context, input ShowTeacherDashboardInput) (ShowTeacherDashboardOutput, error)`
- 依存: 生徒数集計を行うinfrastructure関数、お知らせ取得を行うinfrastructure関数（②「10. UseCase設計」の「呼び出す関数」に対応）
- 処理ステップ（②「10. UseCase設計」を実装単位に展開）:
  1. `input.SchoolID`が未設定（ゼロ値）の場合、Application Error（前提条件不備）を返す（②「14. Error設計」）
  2. `infrastructure.CountStudentsByGrade(ctx, input.SchoolID)`を呼び出し、学年別生徒数を取得する
  3. `infrastructure.ListVisibleAnnouncementsForTeacher(ctx, input.SchoolID, 5)`を呼び出し、公開中お知らせを最新順で上位5件取得する
  4. 2で得た集計結果と3で得たお知らせ一覧を`ShowTeacherDashboardOutput`に組み立てて返す
- トランザクション境界: なし（読み取りのみ。②「11. Transaction設計」）
- 発生しうるApplication Error:
  - 所属校ID未設定（前提条件不備）
  - infrastructure関数からのエラー伝播（DB接続失敗等のInfrastructure Errorをラップして返す）

呼び出し先infrastructure関数の具体的な関数名・シグネチャは②に明記がないため、本書で命名した（14章参照。②からの補足・推測）。

---

# 5. Infrastructure層設計

## Infrastructure関数（Transaction Script採用時）

### CountStudentsByGrade

- 関数シグネチャ: `func CountStudentsByGrade(ctx context.Context, schoolID uint) (application.GradeStudentCounts, error)`
- 発行するクエリ内容: 同校（`schoolID`一致）の生徒を対象に、学年ごとにグループ化して件数を集計する（②「9. Repository設計」StudentStatisticsRepositoryの「保持する検索機能」：高校IDによる絞り込み、学年ごとの件数集計）
- 参照するテーブル: 既存の生徒・学年関連テーブル（Student/School Contextが所有）。本機能は参照のみを行い、作成・更新は行わない
- 参照先Contextの具体的なpackage（Student/School Context側でどのContext名・ディレクトリに実装されているか）は②に記載がなく、アーキテクチャ規約「5. Bounded Context構成」の一覧上の候補（`school-directory`等）から実装時に確定する必要がある（14章参照。②からの補足・推測）

### ListVisibleAnnouncementsForTeacher

- 関数シグネチャ: `func ListVisibleAnnouncementsForTeacher(ctx context.Context, schoolID uint, limit int) ([]application.AnnouncementSummary, error)`
- 発行するクエリ内容: 教師が閲覧可能な公開中お知らせを、対象条件（②「9. Repository設計」AnnouncementRepositoryの「閲覧可能条件（対象条件）」）で絞り込み、公開日時降順で`limit`件（本機能では5件固定）取得する
- 実装方針: ②「9. Repository設計」に「AnnouncementRepository（Announcement Contextの提供機能を利用）」と明記されているため、本関数はAnnouncement Context側が公開する参照用関数（アーキテクチャ規約「6. Context間連携ルール」：Transaction Script採用時は参照用の関数を呼び出す）を呼び出すラッパーとして実装する。Announcement Context側の具体的な関数名・シグネチャは②に記載がなく、実装時にAnnouncement Context側の②③文書に合わせて確定する（14章参照。②からの補足・推測）

いずれの関数も、GORM経由でのSELECTのみを行い、書き込みは行わない。SQL文そのものはここでは記載しない（12章「GORM / DBクエリ設計」参照）。

## 外部連携実装

対象外。②に、Mail・Cache・Queue等の外部連携は挙げられていない。

---

# 6. Presentation層設計

## Handler

### TeacherDashboardHandler

- struct名: `TeacherDashboardHandler`
- 対応する呼び出し先: `application.ShowTeacherDashboard`関数
- メソッド一覧:

| メソッド | HTTPメソッド・パス |
|-|-|
| `Show(c *gin.Context)` | `GET /api/v1/teacher/dashboard` |

- 処理順序:
  1. Middlewareで認証・`teacher`ロール確認済みのcurrent userをcontextから取得する
  2. current userから所属校ID（`SchoolID`）を取り出し、`application.ShowTeacherDashboardInput`を組み立てる（入力バインドは不要。本機能はリクエストパラメータを取らない）
  3. `application.ShowTeacherDashboard(ctx, input)`を呼び出す
  4. 取得した`ShowTeacherDashboardOutput`を`TeacherDashboardResponse`へ変換する
  5. `200 OK`でJSONレスポンスを返す
  6. エラー発生時は11章「Error実装方針」に従いHTTP Statusへ変換して返す

Transaction Script採用のためUseCase層を経由しない。本来UseCaseが担う「所属校IDによるスコープ限定」の準備（current userからSchoolIDを取り出す処理）はHandler側で行い、実際のスコープ適用（検索条件としての利用）はapplication関数・infrastructure関数側で行う（②「13. Authorization設計」）。

## Request / Response DTO

Requestは対象外（パラメータなしのAPI）。

### TeacherDashboardResponse

| フィールド | 型 | 説明 |
|-|-|-|
| `Stats` | `GradeStudentCountsResponse` | 学年別生徒数 |
| `Announcements` | `[]AnnouncementSummaryResponse` | 公開中お知らせ最新5件 |

### GradeStudentCountsResponse

| フィールド | 型 | 説明 |
|-|-|-|
| `Grade1` | `int` | 学年1の生徒数 |
| `Grade2` | `int` | 学年2の生徒数 |
| `Grade3` | `int` | 学年3の生徒数 |

JSONキー名（`grade1`等）は②に明記がなく、①未提供のため既存フロントエンド・Rails現行仕様との厳密な一致を本書では確認できない。実装時に実際のフロントエンド側の期待キー名と突き合わせる必要がある（14章参照。②からの補足・推測）。

### AnnouncementSummaryResponse

Announcement Context側が提供するお知らせ一覧表示用のResponse構造をそのまま利用、または最小限のフィールドに変換する。②「7. Value Object設計」の方針（お知らせ一覧を独自に再定義しない）に従い、本書ではフィールド詳細を規定しない。

バリデーションタグ: 該当なし（Requestを持たないAPIのため）。

## Routing

| Method | Path | Handler |
|-|-|-|
| GET | `/api/v1/teacher/dashboard` | `TeacherDashboardHandler.Show` |

Middleware: 認証Middleware（current user特定）、`teacher`ロール確認Middleware（②「13. Authorization設計」）をルート登録時に適用する。

---

# 7. API仕様

| Method | Path | Handler | Request | Response | Status Code |
|-|-|-|-|-|-|
| GET | `/api/v1/teacher/dashboard` | `TeacherDashboardHandler.Show` | なし | `TeacherDashboardResponse` | 200 |

## Errorケース

| 条件 | Status Code | Error内容 |
|-|-|-|
| 未認証 | 401 | 認証Middlewareで検知（②に明記なし。認証必須APIとしての一般的な実装方針からの補足・推測） |
| `teacher`ロールでない | 403 | ロール確認Middlewareで検知（②に明記なし。補足・推測） |
| current userに所属校情報が存在しない（前提条件不備） | 500 | Application Error（②「14. Error設計」・「16. API互換方針」：業務エラーは発生せず、技術的失敗時は500系で統一する方針に従う） |
| DB接続失敗・参照先Contextへの問い合わせ失敗 | 500 | Infrastructure Error |

②「16. API互換方針」に「認証前提のAPIであり、業務エラーは発生しない。技術的失敗時は500系で統一する」と明記されているため、前提条件不備も含め500系に統一する。

---

# 8. Transaction実装方針

## Transaction開始箇所

なし。②「11. Transaction設計」のとおり、本機能は書き込みを一切伴わないためトランザクションを使用しない。

## Transaction終了箇所

該当なし。

## 複数関数にまたがる場合の扱い

`CountStudentsByGrade`と`ListVisibleAnnouncementsForTeacher`は、それぞれ独立したSELECTのみの読み取り処理であり、1トランザクションでまとめて実行する必要はない。`application.ShowTeacherDashboard`関数内で順に呼び出し、結果を組み立てるのみとする。

---

# 9. Validation実装方針

## Presentation

該当なし。②「12. Validation設計」のとおり、本機能はリクエストパラメータを取らないため、型・必須・フォーマットいずれのチェックも発生しない。

## 業務ルール検証

application関数内のガード節で検証する（Transaction Script採用時の読み替え）。

- `ShowTeacherDashboard`関数内で、`input.SchoolID`が未設定（ゼロ値）の場合はApplication Errorを返すガード節を設ける（②「14. Error設計」の「current userに所属校情報が存在しない等の前提条件不備」に対応）

②「12. Validation設計」に明記のとおり、本機能固有の業務ルール（状態チェック・整合性チェック）は存在しない。

---

# 10. Authorization実装方針

## Middleware

- 認証済みユーザーを特定し、`teacher`ロールであることを確認する（②「13. Authorization設計」）

## Handler

- ルーティングとHTTP出力のみを担当し、業務権限判定は持たせない。current userから所属校IDを取り出し、application関数への入力として引き渡す

## application関数（Transaction Script採用時のUseCase相当）

- current userの所属校ID（`SchoolID`）を、生徒数集計・お知らせ取得の両infrastructure関数へ検索条件として引き渡し、同校・閲覧可能範囲にスコープする（②「13. Authorization設計」）

## Domain

該当なし。本機能は独自のEntityを持たないため、Domain層での認可判定は発生しない（②「13. Authorization設計」）。

認可の実体的な判定（誰の生徒を数えるか、どのお知らせを見せるか）は参照先Context（Student/School Context、Announcement Context）の検索条件に委ね、teacher_dashboard側は所属校IDを検索条件として引き渡す役割にとどめる（②の判断理由をそのまま維持）。

---

# 11. Error実装方針

## Domain Error → Application Errorへの変換方針

Domain Errorは定義しない（②「14. Error設計」）。前提条件の不備（所属校未設定）は、application関数のガード節でApplication Error相当のエラーとして直接生成する。

## Application Error → HTTPレスポンスへの変換方針

- Handlerで受け取ったエラーをHTTP Statusへ変換する
- ②「16. API互換方針」に従い、業務エラーという概念を持たず、Application Error・Infrastructure Errorのいずれも500系に統一する

## Infrastructure Errorのハンドリング方針

- infrastructure関数内で発生したDBエラー・参照先Context問い合わせエラーは、コーディング規約「6. エラーハンドリング」に従い`fmt.Errorf`＋`%w`でラップしてapplication関数に伝播させる
- application関数はさらに必要な文脈を付けてHandlerへ伝播させ、Handlerで500に変換する

| Error種別 | 発生層 | HTTP Status |
|-|-|-|
| 前提条件不備（所属校未設定） | Application（application関数） | 500 |
| DB接続失敗 | Infrastructure（infrastructure関数） | 500 |
| 参照先Context問い合わせ失敗 | Infrastructure（infrastructure関数） | 500 |
| 未認証 | Middleware | 401（②に明記なし。補足・推測） |
| ロール不一致 | Middleware | 403（②に明記なし。補足・推測） |

---

# 12. GORM / DBクエリ設計

## DB方針（②「17. DB設計方針」）

- 既存Rails DBを継続利用する。Schema変更なし
- 本機能は既存の生徒・学年・お知らせテーブルを参照するのみであり、追加のテーブル・カラムは不要

## 利用するGORMモデルとテーブルの対応

- teacher_dashboard自体は永続化対象のEntity・GORMモデルを持たない（②「6. Entity設計」「9. Repository設計」）
- 生徒数集計: Student/School Context側が既に保有する生徒・学年テーブルに対応するGORMモデルを参照する。当該GORMモデルの具体的なstruct定義はStudent/School Context側の②③文書に従う（本書の対象外）
- お知らせ取得: Announcement Context側が既に保有するお知らせテーブルに対応するGORMモデルを参照する。当該GORMモデルの具体的なstruct定義はAnnouncement Context側の②③文書に従う（本書の対象外）
- Gorm規約に従い、テーブル名・カラム名は各所有Context側のGORMモデル定義（構造体名の複数形snake_case、フィールド名のsnake_case）に準拠する。本機能は新規GORMモデルを定義しない

## 主要クエリの条件・ソート・ページネーション方針

- 学年別生徒数集計: 生徒テーブルを`school_id`で絞り込み、学年カラムでグループ化して件数を集計する（GROUP BY相当）。ページネーションは行わない（集計結果は学年1〜3の3件で固定）
- お知らせ取得: お知らせテーブルを、教師向け公開中の対象条件で絞り込み、公開日時の降順でソートし、上位5件に限定する（LIMIT相当）。オフセットによるページネーションは行わない（②「16. API互換方針」：最新5件固定）

## Schemaに対する変更

②「17. DB設計方針」のとおり変更なし。本書でも追加のマイグレーションは不要と判断する。

SQL文そのものはここでは記載しない。

---

# 13. テストケース設計

②「18. テスト戦略」を、Transaction Script採用時の区分読み替え（「Domain Test」「Repository Test」は対象外、「UseCase Test」→「Application関数 Test」）に従って具体化する。

## Domain Test

対象外（Transaction Script採用のため、Domain層を設けない）。

## Application関数 Test

| 対象 | テストケース |
|-|-|
| `ShowTeacherDashboard` | 正常系: 所属校の学年別生徒数集計結果とお知らせ上位5件が、`ShowTeacherDashboardOutput`に正しく組み立てられることを検証する |
| `ShowTeacherDashboard` | 異常系: `SchoolID`が未設定（ゼロ値）のとき、Application Errorを返すことを検証する |
| `ShowTeacherDashboard` | 異常系: 呼び出し先infrastructure関数がエラーを返したとき、エラーがラップされて呼び出し元に伝播することを検証する |

## Repository Test

対象外（Transaction Script採用のため、Repository Interfaceを設けない）。

## Infrastructure関数 Test（②からの補足）

②「18. テスト戦略」のRepository Testに記載された検証観点（学年別集計の正確性、上位5件取得の正確性）を失わないよう、infrastructure層の関数テストとして以下のとおり実施する。指示書の読み替えルールでは本区分は「対象外」に該当するが、②の検証意図を保持するために追加した（14章参照）。

| 対象 | テストケース |
|-|-|
| `CountStudentsByGrade` | 同校の生徒が学年ごとに正しく集計されることを検証する |
| `CountStudentsByGrade` | 他校の生徒が集計に含まれないことを検証する |
| `ListVisibleAnnouncementsForTeacher` | 公開中かつ教師閲覧可能なお知らせが、公開日時降順で上位5件取得されることを検証する |
| `ListVisibleAnnouncementsForTeacher` | 非公開・対象外お知らせが結果に含まれないことを検証する |
| `ListVisibleAnnouncementsForTeacher` | 該当お知らせが5件未満の場合に、存在する件数分のみ返ることを検証する |

## Handler Test

| 対象 | テストケース |
|-|-|
| `TeacherDashboardHandler.Show` | 認証済み`teacher`ユーザーのリクエストで200と、レスポンス構造（`stats`・`announcements`）が正しいことを検証する |
| `TeacherDashboardHandler.Show` | application関数がエラーを返した場合に500を返すことを検証する |

## Integration Test

| 対象 | テストケース |
|-|-|
| `GET /api/v1/teacher/dashboard` | エンドポイント経由で、学年別生徒数とお知らせ最新5件が正しく取得できることを確認する |
| `GET /api/v1/teacher/dashboard` | 未認証でアクセスした場合に401となることを確認する（②に明記なし。補足・推測） |
| `GET /api/v1/teacher/dashboard` | `teacher`以外のロールでアクセスした場合に403となることを確認する（②に明記なし。補足・推測） |

---

# 14. ②からの補足事項

②に明記がなく、実装のために本書で追加判断した内容は以下のとおり。いずれも②の設計判断（採用パターン・責務分離・Authorization方針・Error方針等）そのものを変更するものではない。

| 判断した内容 | 判断理由 | 推測かどうか |
|-|-|-|
| `GradeStudentCounts`をdomain層のValue Objectではなく、application層のDTO（struct）として実装する | ②はValue Objectとして設計しているが、Transaction Script採用時はアーキテクチャ規約「4. 設計パターンごとの構造適用方針」によりdomain層を設けないため、同等の構造をapplication層のDTOとして実装する必要がある | 推測ではない（規約の機械的な読み替え） |
| `application.ShowTeacherDashboard`、`infrastructure.CountStudentsByGrade`、`infrastructure.ListVisibleAnnouncementsForTeacher`という具体的な関数名・シグネチャ | ②「10. UseCase設計」「9. Repository設計」は業務操作の設計意図のみを記載しており、実装レベルの関数名・引数構成までは規定していない。アーキテクチャ規約「9. 命名規約」（Transaction Scriptの関数は「動詞+対象」）に基づき本書で命名した | 推測 |
| 生徒数集計の参照先が、Student/School Context側のどの`internal/`配下Context（例: `school-directory`等）に実装されるか | ②「3. Bounded Context」は「Student/School Context」という抽象名のみを記載し、アーキテクチャ規約「5. Bounded Context構成」の一覧上のどのContextに対応するかは明記がない | 推測 |
| お知らせ取得の呼び出し方式（同一プロセス内のGo関数呼び出しを前提とする） | ②「9. Repository設計」に「AnnouncementRepository（Announcement Contextの提供機能を利用）」とあるが、呼び出し方式（同一プロセス内関数呼び出しか、別プロセス経由か）は明記がない。モノリシックなGo実装を前提とし、同一プロセス内の関数呼び出しと仮定した | 推測 |
| `TeacherDashboardResponse`のJSONフィールド名（`stats`、`announcements`、`grade1`〜`grade3`等） | ②「16. API互換方針」は`stats`・`announcements`という構造をRails現行仕様に近い意味で維持するとのみ記載し、学年別内訳のキー名までは規定していない。①Rails実装（実際のJSONキー）は本セッションでは未提供のため参照不可であり、実装時に既存フロントエンドとの整合を別途確認する必要がある | 推測 |
| 未認証時401、ロール不一致時403というHTTP Status | ②「16. API互換方針」の「Error Response」は業務エラー・技術的失敗（500系）についてのみ記載しており、認証・認可Middlewareレベルのエラーコードには言及していない。一般的なAPI設計慣行から補った | 推測 |
| Repository Testを「対象外」としつつ、別区分「Infrastructure関数 Test」を追加した | ③自動生成プロンプトの読み替えルールでは「Repository Test」は対象外となるが、②「18. テスト戦略」のRepository Testに記載された検証観点（集計・取得の正確性）を本書から失わせないため、区分を追加した | 推測ではない（②の検証意図を保持するための補足） |

上記以外に、②の設計判断（Bounded Context・採用パターン・Aggregate不要・Domain Event不採用・Transaction境界なし等）を変更した箇所はない。
