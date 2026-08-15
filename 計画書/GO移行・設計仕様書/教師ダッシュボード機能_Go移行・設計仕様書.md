# 教師ダッシュボード機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

教師のホーム画面に、同校の学年別生徒数と、教師向けに公開中のお知らせ最新5件を表示する機能である。Rails現行仕様では、単一のエンドポイントで両者を集計・取得し、まとめて返却している。

## 利用者

- `teacher` ロールのユーザー（教師）

## 業務上の目的

- 教師が自校の生徒規模を学年別に把握できるようにする
- 教師に関係のある最新のお知らせを、日々のホーム画面で見逃さないようにする

---

# 2. 設計方針

- 責務分離: HTTP入出力と、集計・参照処理を分離する
- 保守性: 本機能は他Context（生徒数集計・お知らせ）の情報を組み合わせて表示するだけの「参照専用の合成処理」であるため、独自のドメインルールを持ち込まず、必要最小限の構造にとどめる
- テスト容易性: 集計ロジックと表示用の組み立てをUseCase単位で検証できるようにする
- API互換性: 既存フロントエンドとの接続を維持するため、エンドポイントとレスポンス構造を維持する
- 拡張性: 将来的に表示項目（統計・お知らせ以外の情報）が増えても、UseCase内で参照先を追加するだけで対応できる構造とする

---

# 3. Bounded Context

## Context名

- teacher-dashboard（教師ダッシュボードコンテキスト）

## Contextの責務

- 同校の学年別生徒数集計結果の組み立て
- 教師向け公開中お知らせの取得（最新順・件数制限）
- 上記2つを1つの画面表示用データとして合成する

## 他Contextとの依存関係

- Student/School Context: 同校の生徒を学年別に集計するための生徒・学年情報に依存する
- Announcement Context: 公開中かつ教師が閲覧可能なお知らせの取得に依存する（本機能はAnnouncement Contextが提供する検索機能を呼び出すのみで、お知らせの作成・状態管理は行わない）

## 依存する理由

本Contextは生徒数集計もお知らせ管理も自ら所有せず、既存Contextが管理する情報を読み取り専用で組み合わせて画面用データを構成する「合成専用」の性質を持つ。そのため、生徒・お知らせの真正な管理責務は元のContextに残し、本Contextは参照のみを行う。

---

# 4. 設計パターン

## 採用パターン

Transaction Script

## 判断根拠

本機能は、書き込みを一切伴わない「同校生徒数の学年別集計」と「公開中お知らせの上位5件取得」という2つの読み取り処理を1つのレスポンスにまとめるだけの機能である。状態を持つEntityを自ら管理することはなく、業務ルール（誰の生徒を数えるか、どのお知らせを表示するか）も他Context側の絞り込み条件をそのまま利用するにとどまる。

このように、本機能自体には保持すべき状態も、独自の複雑な業務ルールも存在しないため、以下の理由からTransaction Scriptを採用する。

- 業務ルール: 生徒数集計対象・お知らせ閲覧条件は他Contextの既存ルールを利用するだけであり、本機能固有のルールがない
- 状態管理: 本機能が所有する永続化対象のEntityが存在しない
- 将来拡張: 表示項目が増えても、application層の関数内に参照処理を追加するだけで対応可能である
- テスト容易性: 単一の関数に対する入出力テストで十分に検証できる

## 採用しなかったパターン

### Active Record

- 本機能が独自に永続化・更新するEntityが存在しないため、そもそも適用対象がない

### Domain Model

- 状態遷移や複数の業務ルールが絡むEntityが存在せず、Domain Modelを導入する動機がない
- 導入すると、単純な集計処理に対して過剰な抽象化を持ち込むことになる

### Event Sourcing

- 状態の変更や履歴の再構築という概念が本機能には存在しないため、適用の余地がない

---

# 5. Aggregate設計

本機能ではAggregateを設計しない。

理由: 本機能は他Contextが管理するEntity（生徒・お知らせ）を読み取り専用で参照し、集計結果を組み立てるだけであり、自らが整合性を保証すべき永続化対象のEntity群を持たないため、Aggregateという概念自体が該当しない。

---

# 6. Entity設計

本機能は独自のEntityを持たない。

理由: 生徒数集計はStudent/School Contextの生徒情報を、お知らせ取得はAnnouncement Contextのお知らせ情報を参照するのみであり、本機能が状態を保持・変更する対象が存在しない。本機能の応答内容（学年別生徒数、お知らせ一覧）は、他Contextの情報を組み合わせた表示専用の合成結果（Read Model）として扱う。

---

# 7. Value Object設計

## GradeStudentCounts（学年別生徒数の組）

- 採用理由: 「学年1/2/3の生徒数」という3つの数値を意味のあるまとまりとして表現するため
- 独自ルール:
  - 各学年の件数は0以上の整数である
- Entity属性ではなくValue Objectにする理由: 特定のEntityに従属する属性ではなく、集計結果として独立した意味を持つ値の組であるため

## Value Objectを採用しないもの

- お知らせ一覧: Announcement Contextが定義するお知らせ情報をそのまま参照表示するのみであり、本機能側で独自のルールを持つValue Objectとして再定義する必要はない

---

# 8. Domain Service

本機能ではDomain Serviceを設けない。

理由: 生徒数集計は単純な条件付き件数カウントであり、お知らせ取得はAnnouncement Context側の閲覧可能条件をそのまま利用するだけである。複数Entityにまたがる判断や、Entityに持たせるべきではない複雑な業務ルールが存在しないため、UseCase内で完結させることが最も単純で保守しやすい。

---

# 9. Repository設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、Repository Interfaceをdomain層に定義しない。以下は永続化・検索責務の設計意図であり、実装時はinfrastructure層の関数として直接実装する(規約: アーキテクチャ規約.md「4. 設計パターンごとの構造適用方針」)。

## StudentStatisticsRepository

- 管理対象: Student（参照専用）
- 責務:
  - 同校の生徒を学年別に集計する
- 保持する検索機能:
  - 高校IDによる絞り込み
  - 学年ごとの件数集計
- 保持しない責務:
  - 生徒情報の作成・更新（Student/School Contextの責務）
- 判断根拠: 生徒の真正な管理はStudent/School Contextに残し、本機能は集計のための参照のみを行うため

## AnnouncementRepository（Announcement Contextの提供機能を利用）

- 管理対象: Announcement（参照専用）
- 責務:
  - 教師が閲覧可能な公開中お知らせを最新順に上位N件取得する
- 保持する検索機能:
  - 閲覧可能条件（対象条件）による絞り込み
  - 公開日時降順・件数制限（上位5件）
- 保持しない責務:
  - お知らせの作成・状態更新（Announcement Contextの責務）
- 判断根拠: お知らせの管理責務をAnnouncement Contextに残し、本機能は表示用の取得のみを行うため

---

# 10. UseCase設計

**実装上の位置づけ**: 本機能はTransaction Script採用のため、UseCase層(struct)を設けない。以下は業務操作の設計意図であり、実装時はapplication層の関数として直接実装する。

## ShowTeacherDashboard

- 目的: 同校の学年別生徒数と、教師向け公開中お知らせ最新5件を取得し、ダッシュボード表示用データを組み立てる
- 入力: current user
- 出力: 学年別生徒数、お知らせ一覧
- トランザクション範囲: 読み取りのみ、トランザクションは不要
- 呼び出す関数:
  - 学年別生徒数集計関数
  - 公開中お知らせ取得関数
- 判断根拠: 2つの独立した読み取り処理を1つの表示用データに組み立てるだけであり、単一の関数で責務が完結するため

---

# 11. Transaction設計

## Transaction開始位置

- なし（書き込みを伴わないため、トランザクションを使用しない）

## Transaction終了位置

- 該当なし

## 理由

- 本機能は読み取り専用であり、複数の書き込みを1単位で保証する必要がないため、トランザクション管理の対象外とする

---

# 12. Validation設計

## Presentation

- 型チェック: 本機能はリクエストパラメータを取らないため、該当なし
- 必須チェック: 該当なし
- フォーマットチェック: 該当なし

## Domain

- 業務ルール: 該当なし（本機能固有の業務ルールを持たない）
- 状態チェック: 該当なし
- 整合性チェック: 該当なし

## 責務分離

本機能はパラメータを受け取らない参照専用APIであるため、Presentation/Domainいずれの層にも独自の検証ロジックを持たない。生徒集計対象・お知らせ閲覧条件のスコープ限定は、認可（current userの所属校情報の利用）として13章で扱う。

---

# 13. Authorization設計

## Middleware

- 認証済みユーザーを特定し、`teacher` ロールであることを確認する

## Handler

- ルーティングとHTTP出力のみを担当し、業務権限判定は持たせない

## UseCase

- current userの所属校IDを用いて、生徒集計とお知らせ取得の両方を同校・閲覧可能範囲にスコープする

## Domain

- 該当なし（本機能は独自のEntityを持たないため、Domain層での認可判定は発生しない）

## 判断理由

本機能は他Contextの情報を参照するだけであるため、認可の実体的な判定（誰の生徒を数えるか、どのお知らせを見せるか）は参照先Context（Student/School Context、Announcement Context）の検索条件に委ね、本機能のapplication関数はcurrent userの所属校情報を検索条件として引き渡す役割にとどめる。

---

# 14. Error設計

## Domain Error

責務: 本機能固有のDomain Errorは定義しない

## Application Error

責務: ユースケース実行時の失敗を表現する

- current userに所属校情報が存在しない等の前提条件不備

## Infrastructure Error

責務: DB接続失敗、参照先Contextへの問い合わせ失敗等の技術的障害を表現する

## 判断理由

本機能には業務ルール違反という概念が存在しないため、Domain Errorは定義しない。前提条件の不備（所属校未設定等）はUseCase実行時の異常としてApplication Errorで扱い、技術的な失敗と区別する。

---

# 15. Domain Event

本機能ではDomain Eventを採用しない。

理由: 本機能は表示専用の参照処理であり、状態変化や他処理への通知を伴わないため、イベントという概念自体が該当しない。

---

# 16. API互換方針

## URL

- Rails現行仕様と同じエンドポイントを維持する
  - GET /api/v1/teacher/dashboard

## HTTP Method

- 既存仕様どおりGETを維持する

## Request

- パラメータなし。現行仕様を維持する

## Response

- `stats`（学年別生徒数）、`announcements`（お知らせ最新5件）という構造をRails現行仕様に近い意味で維持する

## Status Code

- 200: 取得成功

## Error Response

- 認証前提のAPIであり、業務エラーは発生しない。技術的失敗時は500系で統一する

---

# 17. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 本機能は既存の生徒・学年・お知らせテーブルを参照するのみであり、追加のスキーマ変更は不要である

---

# 18. テスト戦略

## Domain Test

- 該当なし（本機能は独自のEntity/ドメインルールを持たないため）

## UseCase Test

- 目的: ShowTeacherDashboardUseCaseが、生徒集計結果とお知らせ一覧を正しく組み立てて返すことを検証する

## Repository Test

- 目的: StudentStatisticsRepositoryによる学年別集計の正確性、AnnouncementRepositoryによる上位5件取得の正確性を検証する

## Handler Test

- 目的: レスポンス構造とHTTPステータスの変換を検証する

## Integration Test

- 目的: エンドポイント経由でダッシュボード情報が正しく取得できることを確認する

---

# 19. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller（集計処理を直接記述） | Handler + application関数 | 集計処理をapplication関数へ切り出し、HandlerはHTTP変換のみを担当する(Transaction Script採用のためusecase層は設けない) |
| ActiveRecordのgroupクエリ | StudentStatisticsRepository | 集計クエリの責務をRepositoryに集約する |
| Announcementスコープ呼び出し | AnnouncementRepository（Announcement Context提供） | お知らせ取得はAnnouncement Contextの責務として維持する |
| Serializer | Presenter / Response DTO | レスポンス整形をPresentation層に分離する |

---

# 20. 採用しなかった設計

## Domain Model

- 採用しなかった理由: 本機能が管理すべき状態を持つEntityが存在せず、導入する動機がないため
- 将来的に採用する可能性: 低い。表示専用の性質が変わらない限り不要である

## 独自の統計Entity/永続化テーブルの新設

- 採用しなかった理由: 現行仕様は都度集計で要件を満たしており、集計結果を永続化する必要性が示されていないため
- 将来的に採用する可能性: 生徒数が非常に多くなり、都度集計のコストが問題になった場合は、集計結果のキャッシュ化を検討する余地がある（推測）

---

# 21. 設計判断サマリー

| 項目 | 採用 | 判断理由 |
|---|---|---|
| 設計パターン | Transaction Script | 状態を持たない読み取り専用の合成処理であるため |
| Aggregate | 不要 | 自ら整合性を保証すべきEntityを持たないため |
| Transaction境界 | なし | 書き込みを伴わないため |
| Domain Event | 未採用 | 状態変化・他処理への通知が存在しないため |
| Domain Service | 未採用 | 複数Entityにまたがる複雑な業務ルールがないため |
| Repository | 参照専用（StudentStatistics/Announcement） | 他Contextの管理責務を侵さず、集計・取得に限定するため |

---

# 設計差分管理

## Rails現行仕様

- Controllerのshowアクション内で、生徒数集計クエリとお知らせ取得クエリを直接記述している

## Go設計での変更内容

- 集計処理とお知らせ取得処理をUseCase（ShowTeacherDashboardUseCase）に切り出す
- 生徒数集計をStudentStatisticsRepository、お知らせ取得をAnnouncementRepositoryとして責務ごとに分離する
- Handlerは入出力の変換のみを担当する

## 変更理由

- ControllerにActiveRecordクエリを直接書く現行実装は、Goでは責務が曖昧になりHTTP層と参照処理が密結合するため、UseCase/Repositoryに分離することでテスト容易性を高める

## 影響範囲

- フロントエンドから見たAPI外部仕様は維持する
- 既存DBスキーマは維持するため、データ移行は不要
