# 教員招待通知機能 Go移行・設計仕様書

---

# 1. 機能概要

## 機能概要

新規に登録されたものの、まだ招待（パスワード設定）を完了していない教員に対して、招待メールを再送・一括送信できる機能である。Rails現行仕様では、同校の招待未完了教員一覧を取得し、選択した教員へ一括送信（非同期、202 Accepted）を行い、送信結果（成功・失敗）を履歴として日付で絞り込んで参照できる。

## 利用者

- `teacher` ロールのユーザー（教師）
- 招待未完了教員一覧の取得・送信操作には、他職員操作権限（`manage_other_teachers`）が必要。送信結果履歴の参照には特別な権限は不要（同校の教師であれば閲覧できる）

## 業務上の目的

- パスワード未設定のまま招待が完了していない教員に対し、招待メールを再送できるようにする
- 一括送信操作を、対象教員の絞り込み（同校・招待未完了）を適切に適用した上で安全に行えるようにする
- 送信結果を履歴として残し、送信済み・未送信の状況を後から確認できるようにする

---

# 2. 設計方針

- 責務分離: HTTP入出力、入力形式検証、送信対象の絞り込み、送信結果の記録、招待メールの送信依頼（`user` Contextの`RequestInvitationEmail`）を分離する。招待メールの組み立て・パスワード設定用トークンの発行・メール送信基盤の呼び出しは、本Contextが持たない
- 保守性: 「同校」「招待未完了」という送信対象の絞り込みルールを一箇所に集約する。招待メールの内容と送信手順は、アカウント作成時の送信と共通に`user` Contextの1箇所で定義され、本Contextは結果を受け取って記録するのみとする
- テスト容易性: 送信対象の絞り込みロジックと、送信結果記録の正確性を独立して検証できるようにする
- API互換性: 既存フロントエンドとの接続を維持するため、エンドポイント・リクエスト構造・レスポンス構造は概ね維持する
- 拡張性: 将来的に他の招待手段（SMS等）が追加された場合にも対応しやすい構造とする

---

# 3. Bounded Context

## Context名

- teacher-notification

## Contextの責務

- 同校の招待未完了教員一覧の取得
- 選択した教員への招待通知（招待メール）の一括送信の依頼（招待メールの組み立て・トークン発行・送信そのものは`user` Contextが担う）
- 送信結果（成功・失敗）の履歴管理・参照

## 他Contextとの依存関係

- User Context（Context名`user`）: 次の2点に依存する
  - 参照: 教員アカウントの識別情報、所属校情報、招待完了状態（初回パスワード未設定かどうか）
  - 招待メールの送信依頼: `user`②「12. UseCase設計」の`RequestInvitationEmail`を、実行方式「即時」で呼ぶ。結果（送信成功／送信失敗／送信対象外）は戻り値として受け取り、本Contextが`TeacherNotification`へ記録する
- メール送信基盤・パスワード設定用トークンの発行（`authentication`）: 本Contextは直接呼ばない。`user`の送信手順が呼ぶ（`user`②「3. Bounded Context」）

## 依存する理由

招待未完了かどうかの判定は、教員アカウント本体（User）が持つパスワード状態を前提とする。この情報の真正な管理はUser Contextに属するため、本Contextは参照のみ行い、教員アカウント自体の作成・更新は行わない。

招待メールの送信は、アカウント作成時の送信と同じ文面・同じトークンの有効期間・同じ手順で行う必要がある。Rails現行実装では、この送信手順がアカウント作成時（`Common::CreateUserService`）と教員招待通知の再送（`Teacher::TeacherNotificationJob`）に別々に実装されているため、`user`②は招待メールの送信依頼（`RequestInvitationEmail`）を1つに集約している。本Contextは、その実行方式「即時」を呼ぶ。「即時」は、呼び出しの中で送信して結果を返す方式であり、Rails現行の再送（`deliver_now`の直後にその場で成否を判定して記録する）と同じ結果の受け渡しを保てる。`jobs`を経由する「登録」方式は、結果を非同期に確定させるため本Contextでは使わない（`user`②「24. 採用しなかった設計」）。依存の向きは`teacher-notification`→`user`の一方向であり、`user`は本Contextを参照しない（アーキテクチャ規約5章）。

なお、教師教員管理機能（`teacher-management` Context）は、教員一覧表示のために本Context（`teacher-notification`）が管理する送信結果を参照する。本Contextから`teacher-management`への依存は存在しない（依存の向きは一方向であり、アーキテクチャ規約5章が禁止する循環依存を避けるため、本Contextは教員一覧の取得を`teacher-management`に依存せず、User Contextから直接行う）。

---

# 4. 設計パターン

## 採用パターン

Active Record

## 判断根拠

本機能が扱う操作は、招待未完了教員一覧の参照、選択教員への一括送信、送信結果履歴の参照という、状態遷移を伴わないCRUD中心の業務である。

- TeacherNotificationの状態（pending/sent/failed）は、送信処理の実行結果に応じて作成時点で一度だけ確定し、その後の遷移（ユーザー操作や別の業務イベントによる状態変更）は発生しない。複数の状態を跨いだ遷移ルールを持つEntityではなく、「結果を記録する1件のレコード」という性質が強い
- 送信対象の絞り込み（同校・招待未完了）は入力値と参照データの突き合わせであり、複雑なドメインロジックというよりは値の妥当性検証に近い
- 各教員への送信・記録は互いに独立しており（1人の送信失敗が他の対象者に影響しない）、複数Entityにまたがる整合性を保証する集約構造を必要としない

このため、以下の理由からActive Recordを採用する。

- 主要操作が一覧・一括送信・履歴参照というCRUD中心の操作であり、複雑な状態遷移が存在しない
- 送信対象の絞り込みロジック（同校・招待未完了）は、Entity（TeacherNotification）が保持する情報だけでは完結せず、User Context側の参照情報との突き合わせによる値の妥当性検証として十分に表現できる
- 各送信結果が独立したレコードとして記録されれば足り、複雑なドメイン振る舞いを必要としない

## 採用しなかったパターン

### Transaction Script

対象教員の絞り込み・他職員操作権限の確認という複数の判定を手続き型に寄せると、テスト単位が不明瞭になりやすい。送信結果の記録という永続化対象が明確に存在するため、structへ責務を持たせるActive Recordの方が適している。

### Domain Model

TeacherNotification自体に状態遷移や、複数Entityにまたがる複雑な業務ルールの蓄積は存在しない。送信対象の絞り込みは値検証に近く、Entityに振る舞いを集約するメリットが薄く、DDDを目的化した過剰設計になるため不採用とする。

### Event Sourcing

送信という単発の操作に対して、履歴の再構築や監査目的のイベント管理は現行仕様上求められていない。送信結果は`teacher_notifications`テーブルへの記録で十分に表現できるため、過剰な設計であり不採用とする。

---

# 5. Aggregate設計

不要と判断する。

理由: TeacherNotificationは1件ごとに独立したレコードであり、複数Entityにまたがる整合性を保証する必要がある集約構造を持たない。各教員への送信・記録が互いに独立している（1人の送信失敗が他の対象者の記録に影響しない）という業務要件そのものが、Aggregateとして境界を設ける必要性を排除している。

---

# 6. Entity設計

## TeacherNotification

- 役割: 教員への招待通知1件の送信結果を表す概念
- ライフサイクル: 一括送信処理の実行結果として1件ずつ作成される。作成後の更新・削除は行わない
- 状態変化: 招待メールの送信依頼（`user`の`RequestInvitationEmail`、実行方式「即時」）の結果に応じて、sent（送信成功）/failed（送信失敗・送信対象外）のいずれかが作成時点で確定する。pending（未送信）はレコード上のデフォルト値であり、Rails現行と同じく、本Contextが`pending`のまま作成することはない。作成後の状態遷移は発生しない
  - 結果の対応づけ: 送信成功→`sent`＋`sent_at`を記録、送信失敗→`failed`（`sent_at`なし）、送信対象外（対象の教員が存在しない、または無効化済み。送信していない）→`failed`（`sent_at`なし）。Rails現行の記録（成功は`sent`と`sent_at`、例外は`failed`で`sent_at`なし）と同じである
- 保持する責務:
  - 送信者（sender）・送信先教員（receiver）・送信先メールアドレス・送信日時（`sent`の場合のみ）・送信結果（status）を保持する
- 判断根拠: 送信結果を1件ごとに記録するという業務要件の中心的な対象であるため

## Teacher（外部参照、招待未完了教員の一覧表示対象）

- 役割: 招待未完了かどうかの判定対象となる教員アカウント
- 判断根拠: 教員アカウント自体の真正な管理はUser Contextに属し、本ContextはUser Contextの`password_reset_required`相当の状態を参照するのみであるため（推測: 教師生徒参照機能・教師教員管理機能のデータモデルに記載された`password_reset_required`フィールドと同一の仕組みを、招待完了判定に用いると仮定する）

---

# 7. Value Object設計

## InvitationStatus

- 採用理由: 送信結果を文字列のまま扱うと、許容されない値が実装のあちこちで再チェックされ、抜け漏れが起きやすいため
- 独自ルール: pending/sent/failedのいずれかのみを許容する。作成時に、招待メールの送信依頼の結果（送信成功／送信失敗／送信対象外。6章の対応づけ）から一意に決定され、以降変更されない
- Entity属性ではなくValue Objectにする理由: 許容値の妥当性チェックを型として表現し、送信処理・履歴参照双方のstructメソッドから再利用できるようにするため

## Value Objectを採用しないもの

- 送信先メールアドレス: 送信対象の教員アカウントに紐づく既存のメールアドレスをそのまま複写する値であり、本Context独自の検証ルールを持たないためValue Object化は不要とする

---

# 8. Domain Service

## InvitationEligibilityFilter

- 責務: 指定された教員ID一覧（`teacher_ids`）から、実際に送信対象とすべき教員（操作者と同校、かつ招待未完了）を絞り込む
- Entityへ持たせない理由: 判定にはTeacherNotification単体の情報ではなく、User Context側の教員アカウント情報（所属校・パスワード状態）が必要なため
- 判断根拠: 同校でない教員・既に招待完了済みの教員が指定されても、エラーとはせず単に送信対象から除外するという業務ルール（Rails現行仕様「業務ルール」節参照）を1箇所に集約し、一覧取得・一括送信の双方から再利用できるようにするため
- 無効化済み（`deleted_at`あり）の教員の扱い: 絞り込みの条件は「同校」「招待未完了」のみで、無効化済みの教員を除外しない（Rails現行の`base_teachers_scope`が有効なユーザーへの絞り込みを持たないことと同じ）。無効化済みの教員は、絞り込みの後、招待メールの送信依頼が「送信対象外」として扱う（設計差分管理の「Rails現行との差」）

---

# 9. クラス図

Active Record採用のため、Entity間の関係は単純である。TeacherNotificationは他Entityとの集約関係を持たず、送信対象の絞り込みに必要な外部参照（Teacher）のみを持つため、以下に簡略化して示す。

```mermaid
classDiagram
    class TeacherNotification {
      +id
      +senderUserId
      +receiverUserId
      +email string
      +sentAt time
      +status InvitationStatus
    }
    class InvitationStatus {
      <<ValueObject>>
      pending
      sent
      failed
    }
    class InvitationEligibilityFilter {
      <<DomainService相当（Active Record採用のためstruct非依存の関数として実装）>>
    }
    class Teacher {
      <<外部参照 User Context>>
    }
    class InvitationEmailResult {
      <<外部参照 User Context（RequestInvitationEmail・即時の結果）>>
      送信成功
      送信失敗
      送信対象外
    }

    TeacherNotification --> InvitationStatus : 保持
    TeacherNotification --> Teacher : 参照（sender/receiver）
    InvitationEligibilityFilter ..> Teacher : 送信対象を絞り込み
    InvitationEmailResult ..> InvitationStatus : 送信成功=sent／送信失敗・送信対象外=failed
```

---

# 10. 状態遷移図

省略する。

理由: TeacherNotificationのstatus（pending/sent/failed）は、送信処理の実行結果に応じて作成時点で一度だけ確定する値であり、作成後にユーザー操作や別の業務イベントによって遷移する多段階の状態遷移ルールを持たない。「6. Entity設計」に記載のとおり作成時点で結果が確定する単純な結果値であるため、状態遷移図としての可視化は行わない。

---

# 11. Repository設計

**実装上の位置づけ**: 本機能はActive Record採用のため、Repository Interfaceをdomain層に定義しない。以下は永続化・検索責務の設計意図であり、実装時はEntity相当のstructと同一packageに置くStore（例: `〇〇Store`）として直接実装する（規約: アーキテクチャ規約.md「3. 設計パターンごとの構造適用方針」）。

## TeacherNotificationStore

- 管理対象: TeacherNotification
- 責務:
  - 送信結果の一括作成
  - 同校の招待未完了教員一覧の取得（User Context参照を含む）
  - 送信結果履歴の取得（送信者が同校、日付絞り込み任意）
- 保持する検索機能:
  - high_school_idによる絞り込み（招待未完了教員一覧・履歴の双方）
  - 招待未完了（パスワード未設定）による絞り込み
  - sent_atの日付絞り込み（履歴取得時）
  - 送信日時降順のソート
- 保持しない責務:
  - 送信対象の絞り込み判定そのもの（InvitationEligibilityFilterが担う）
  - 教員アカウント自体の作成・更新
  - 招待メールの組み立て・パスワード設定用トークンの発行・メール送信（`user` Contextの`RequestInvitationEmail`が担う）
- 判断根拠: 送信結果の永続化・検索に責務を限定し、絞り込みロジックはInvitationEligibilityFilter、教員アカウントの真正な管理はUser Contextに委ねるため

---

# 12. UseCase設計

**実装上の位置づけ**: 本機能はActive Record採用のため、UseCase層（struct）を設けない。以下はHandlerが行う業務操作の設計意図であり、実装時はHandlerがStoreを直接呼び出す処理として実装する。

## ListUnsentTeachers（Handler処理）

- 目的: 同校の招待未完了教員一覧を取得する
- 入力: current teacher
- 出力: 招待未完了教員一覧
- トランザクション範囲: 読み取りのみ、トランザクション不要
- 呼び出すStore: TeacherNotificationStore（current teacherが「他職員操作権限」を持つことをHandler処理内で確認したうえで呼び出す）
- 判断根拠: current teacherの所属校・招待未完了という条件で絞り込むだけの単純な参照処理であるため

## SendInvitationNotifications（Handler処理）

- 目的: 選択した教員に対して招待通知を一括送信する
- 入力: current teacher, teacher_ids（任意、配列）
- 出力: 受付結果（送信処理を開始した旨）
- トランザクション範囲: 使用しない（各教員への送信・記録は独立した処理であり、全体を1トランザクションにまとめる必要がない）。招待メールの送信依頼は、トランザクションの外から呼ぶ（`user`②「12. UseCase設計」の`RequestInvitationEmail`（即時）の規定）
- 呼び出すStore: TeacherNotificationStore（current teacherの「他職員操作権限」確認、InvitationEligibilityFilterによる送信対象確定後、対象教員ごとに送信・記録）
- 呼び出す他Contextの操作: `user` Contextの`RequestInvitationEmail`（実行方式「即時」）。対象教員ごとのgoroutine内で、教員のIDを渡して呼び、結果（送信成功／送信失敗／送信対象外）を受け取る。結果は次のとおり`TeacherNotification`として記録する（6章）
  - 送信成功→`sent`＋`sent_at`
  - 送信失敗→`failed`（`sent_at`なし）
  - 送信対象外→`failed`（`sent_at`なし）
- 判断根拠: 1人の教員への送信が失敗しても他の対象教員への送信処理が継続されるという業務要件（Rails現行仕様「業務ルール」節）があるため、対象教員ごとに独立した処理として扱う。送信の手順（トークン発行→メール送信）と結果の判定は`user`に委ね、本Contextは結果を受け取って記録する。結果は呼び出しの戻り値として同期的に得られるため、`TeacherNotification`は作成時に一度だけ確定し、後から更新しない（4章・6章）

## ListNotificationResults（Handler処理）

- 目的: 同校の教員宛てに送信された招待通知の結果履歴を取得する
- 入力: current teacher, sent_at（任意、日付）
- 出力: 送信結果一覧
- トランザクション範囲: 読み取りのみ
- 呼び出すStore: TeacherNotificationStore
- 判断根拠: 権限確認不要（同校の教師であれば誰でも閲覧できる）で、日付絞り込み・同校スコープの単純な参照処理であるため

---

# 13. シーケンス図・処理フロー図

## シーケンス図

### SendInvitationNotifications（Handler処理）

```mermaid
sequenceDiagram
    participant H as Handler
    participant S as TeacherNotificationStore
    participant F as InvitationEligibilityFilter
    participant U as User Context（参照）
    participant R as User Context（RequestInvitationEmail・即時）

    H->>S: 操作者の他職員操作権限を確認
    alt 権限なし
        S-->>H: 403 Forbidden
    else 権限あり
        H->>S: SendInvitationNotifications(current teacher, teacher_ids)
        S->>U: 同校の招待未完了教員一覧を取得
        U-->>S: 教員一覧
        S->>F: teacher_idsを同校・招待未完了で絞り込み
        F-->>S: 送信対象の教員一覧
        S-->>H: 202 Accepted（受付完了）
        par 対象教員ごとに独立して実行（goroutine。トランザクションの外）
            S->>R: 招待メールの送信を依頼（教員ID・実行方式=即時）
            Note over R: トークン発行・メール組み立て・送信はuserが行う
            R-->>S: 送信成功 / 送信失敗 / 送信対象外
            S->>S: TeacherNotificationを作成（成功=sent+sent_at、失敗・送信対象外=failed。sent_atなし）
        end
    end
```

`RequestInvitationEmail`の内部（対象ユーザーの取得、無効化済みの判定、トークン発行、メール送信）は、`user`②「13. シーケンス図・処理フロー図」の「RequestInvitationEmail（教員招待通知の再送。即時方式）」に示されている。

## 処理フロー図

単純なCRUD処理（絞り込み・一括送信・履歴参照）であり、状態遷移を伴わないため、条件分岐が多い複雑な業務ロジックは存在しない。処理フロー図は省略する。

---

# 14. Transaction設計

## Transaction開始位置

- ListUnsentTeachers / ListNotificationResultsでは使用しない
- SendInvitationNotificationsでは、対象教員ごとの送信・記録を独立した処理として扱うため、全体を1つのトランザクションにまとめない。`user`の`RequestInvitationEmail`（即時）は、トランザクションの外から呼ぶ（`user`②が、コミット前に送信されてはならないこと、トークン発行が`authentication`の書き込みであることから、呼び出し側のトランザクションで包まない前提としている）。`TeacherNotification`の作成は、送信依頼の結果を受け取った後の単発の書き込みである

## Transaction終了位置

- 対象教員ごとに、送信結果（TeacherNotification）の作成が完了した時点でその教員分の処理が完了する

## 理由

Rails現行仕様書は「1人の教員へのメール送信が失敗しても、他の対象教員への送信処理は継続される（1件ごとに成功・失敗が記録される）」と明記している。これは複数の独立した処理を1トランザクションにまとめてはならないという業務要件そのものであり、対象教員ごとに個別の書き込みとして扱う。

---

# 15. Validation設計

## Presentation

- 型チェック: teacher_idsが整数の配列であることを検証する
- 必須チェック: なし（teacher_idsが空でもエラーとしない。Rails現行仕様どおり）
- フォーマットチェック: sent_at（履歴取得時）が日付形式であることを検証する

## Domain

- 業務ルール: 操作者が「他職員操作権限」を持つこと（一覧取得・一括送信時）、送信対象が同校かつ招待未完了であること（InvitationEligibilityFilter）。対象の教員が存在し、無効化されていないか（招待メールを実際に送ってよいか）の最終判定は`user`の`RequestInvitationEmail`が行い、該当しない場合は「送信対象外」の結果として返る（本Contextでは`failed`として記録する）
- 状態チェック: 該当なし（TeacherNotificationは作成時に状態が確定するのみで、状態に対する事前チェックは発生しない）
- 整合性チェック: 該当なし

## バリデーション仕様

|フィールド|検証層|ルール|エラーメッセージ方針|
|-|-|-|-|
|teacher_ids|Presentation|任意・整数の配列|なし（空でもエラーとしない）|
|teacher_idsの送信対象|Domain|同校かつ招待未完了であること|エラーとせず対象から除外する|
|操作者の権限（一覧取得・一括送信）|Domain|「他職員操作権限」を保持すること|「他教員を招待する権限がありません」|
|sent_at|Presentation|任意・日付形式|なし（不正な場合は絞り込み条件として無視、またはPresentation層で422）|

---

# 16. Authorization設計

## Middleware

- 認証済みユーザーを特定し、`teacher` ロールであることを確認する

## Handler

- 招待未完了教員一覧の取得・一括送信の実行前に、current teacherが「他職員操作権限（`manage_other_teachers`）」を保持しているかを確認し、保持していない場合は処理を中断する
- 送信結果履歴の参照には、この権限確認を行わない（同校の教師であれば誰でも閲覧できる）
- `user`の`RequestInvitationEmail`は認可を行わない（`user`②「16. Authorization設計」）。「その教員へ送ってよいか」（他職員操作権限・同校・招待未完了）は、本Contextが判定した上で呼ぶ

## UseCase

- （Active Record採用のためUseCase層は設けない。上記Handler処理内の確認がこれに相当する）

## Domain

- InvitationEligibilityFilterが、同校でない教員・招待完了済みの教員を送信対象から除外する

## 判断理由

「他職員操作権限」の要否は、ロール（`teacher`であるか）のような粗い認可ではなく、教員個人が持つ業務権限に基づく判定であるため、Middlewareのロールチェックとは別に、Handler処理内で確認する（アーキテクチャ規約7章「認可（所有権・業務権限）」の配置方針、Active Record採用時はHandler/Storeに従う）。送信結果履歴の参照にはこの権限を要求しないという非対称な権限設計は、Rails現行仕様「8. 権限制御」に明記された業務要件をそのまま踏襲したものである。

---

# 17. Error設計

## Domain Error

責務: 業務ルール違反を表現する

- 該当なし（送信対象の絞り込みはエラーとせず除外で処理するため、Domain Errorとして扱う業務ルール違反は本機能には存在しない）

## Application Error

責務: ユースケース実行時の失敗を表現する

- 操作者が「他職員操作権限」を持たないまま一覧取得・一括送信を試みた（Forbidden）

## Infrastructure Error

責務: DB接続・永続化失敗等の技術的障害を表現する。招待メールの送信失敗（トークン発行の失敗・メール送信基盤の失敗）は、本Contextではエラーとして扱わない。`user`の`RequestInvitationEmail`（即時）が、エラーではなく結果「送信失敗」として返すため（`user`②「17. Error設計」）、本Contextはこれを業務結果として受け取り、`TeacherNotification`の`status=failed`として記録する

## 判断理由

本機能は「対象を絞り込んで記録する」という性質が強く、入力値そのものの業務ルール違反（Domain Error）はほぼ発生しない。唯一の業務的なエラーケースは操作者の権限欠如であり、これはリソースの状態ではなく操作者の資格に起因する失敗であるため、Application Errorとして扱い403に変換する。メール送信の失敗と、送信対象外（対象の教員が存在しない、または無効化済み）は、TeacherNotificationのstatus=failedとして正常に記録される業務結果であり、エラーとしては扱わない。

`RequestInvitationEmail`の呼び出し自体が内部エラー（`user`②が定めるのは、実行方式・ユーザーIDの欠如という呼び出しの誤用。即時方式で対象ユーザーの取得に失敗した場合などは規定がない）を返した場合の扱いは、`user`②に規定がない。Rails現行の再送は、例外の種類を問わず`failed`として記録するため、本Contextも、他の教員の処理を継続した上で、その教員を`failed`（`sent_at`なし）として記録し、原因をログに残す（推測）。

## エラー仕様

|業務シナリオ|エラー種別|発生層|想定するHTTPステータス|
|-|-|-|-|
|操作者が他職員操作権限を持たずに一覧取得・一括送信を試みた|Forbidden|Application|403|
|個々の教員への送信が失敗した（`RequestInvitationEmail`が「送信失敗」を返した）|該当なし（正常な業務結果としてstatus=failedを記録。`sent_at`なし）|-|202（受付自体は成功）|
|個々の教員が送信対象外だった（`RequestInvitationEmail`が「送信対象外」を返した。対象が存在しない、または無効化済み）|該当なし（正常な業務結果としてstatus=failedを記録。`sent_at`なし）|-|202（受付自体は成功）|
|未認証|-|Middleware|401|

---

# 18. Domain Event

不要と判断する。

理由: 一括送信対象の教員ごとの送信・記録は互いに独立した処理であり、Rails現行仕様書も「対象教員ごとに、招待メールを送信し、送信結果を記録する」という単純なループ処理として記述している。生徒CSVインポート機能・教師面談機能・クラス編成機能のように「1つの状態変化が複数の異なるトリガー・複数の宛先へ波及する」構造を持たず、対象教員1人につき送信→記録という1対1の処理が繰り返されるのみである。そのため、規約5章のDomain Event採用条件（複数の処理へ波及する非同期通知が必要な場合）には該当しない。

実装上は、対象教員ごとの送信処理をアーキテクチャ規約13章（非同期ジョブ実行パターン）の「ベストエフォートで良い処理」として扱う。招待メールの送信は、失敗しても操作者が再度対象教員を選択して送信し直せる（Rails現行仕様「業務ルール」節: `teacher_ids`が空でもエラーにならない）ため、パスワードリセットメール送信と同様、確実な再試行（`jobs`テーブル）までは必須としない。

対象教員ごとのgoroutine内では、`user` Contextの`RequestInvitationEmail`を実行方式「即時」で呼ぶ。`jobs`へ登録する実行方式（「登録」）は使わない。本Contextは送信の結果を`TeacherNotification`として作成時に一度だけ確定させる構造であり、結果が呼び出しの戻り値として得られる「即時」であれば、記録の更新や、`jobs`の状態の参照・完了の通知（依存の向きの逆転や、`teacher_notifications`と`jobs`の対応付けを要する）が不要になるためである（`user`②「12. UseCase設計」「24. 採用しなかった設計」）。

---

# 19. API仕様

## エンドポイント一覧

|エンドポイント|HTTP Method|概要|
|-|-|-|
|/api/v1/teacher/teacher_notifications|GET|招待未完了の教員一覧を取得|
|/api/v1/teacher/teacher_notifications|POST|選択した教員へ招待通知を送信|
|/api/v1/teacher/teacher_notification_results|GET|招待通知の送信結果履歴を取得|

## 各エンドポイントの仕様

### GET /api/v1/teacher/teacher_notifications

- Request: なし
- Response: 招待未完了の教員一覧（id, name, name_kana, email）
- Status Code: 200 / 403（他職員操作権限なし）
- Error Response: なし（他職員操作権限を持つ教師であることが前提。権限がない場合は既存のerrors形式を踏襲する）

### POST /api/v1/teacher/teacher_notifications

- Request: teacher_ids[]（任意、送信対象教員IDの配列）
- Response: message（「送信処理を開始しました」相当）
- Status Code: 202 / 403（他職員操作権限なし）
- Error Response: 既存のerrors形式を踏襲する

### GET /api/v1/teacher/teacher_notification_results

- Request: sent_at（任意、日付。指定日の0時〜24時に絞り込む）
- Response: 送信結果一覧（id, email, status, formatted_sent_at, 送信者(id, name), 送信先教員(id, name)）
- Status Code: 200
- Error Response: なし（認証前提）

## Railsとの差分

現時点でRails仕様からの変更はない。エンドポイント・リクエスト構造・レスポンス構造・ステータスコードはRails現行仕様を維持する（無効化済みの教員が送信対象に含まれた場合に、履歴の`status`の値が変わりうる点のみ、設計差分管理の「Rails現行との差」を参照）。

---

# 20. DB設計方針

## 現行DBを利用するか

- 既存Rails DBを継続利用する

## Schema変更有無

- 変更なし

## 変更理由

- 現行の `teacher_notifications` テーブルは、送信結果の1件ごとの記録という業務要件を満たしており、追加のスキーマ変更は不要である（`failed`は`sent_at`をNULLとして記録する。Rails現行も同じ）
- 招待メールの送信は`user`の`RequestInvitationEmail`（即時）の呼び出しで行うため、本機能に固有の送信用テーブルや、`jobs`テーブルの利用は発生しない

---

# 21. DB操作仕様

## TeacherNotificationStore

- 対象テーブル: `teacher_notifications`
- 操作種別: 作成（対象教員ごとに1件。`failed`は`sent_at`をNULLで作成する）、参照（招待未完了教員一覧、送信結果履歴）
- 主な検索条件・絞り込み条件:
  - 招待未完了教員一覧: `high_school_id`、教員ロール、パスワード未設定（User Context参照）。無効化済み（`deleted_at`あり）は除外しない（Rails現行と同じ。8章）
  - 送信結果履歴: `sender_user_id`が同校、`sent_at`の日付範囲（任意）
- 関連テーブルとの結合: `users`との結合（送信者名・送信先教員名の取得、招待未完了判定のため）
- 招待メールの送信（トークン発行・メール送信）: 本Contextのテーブル操作ではなく、`user`の`RequestInvitationEmail`（即時）の呼び出しで行う。`jobs`は使わない
- ページネーション・ソート: 送信結果履歴は送信日時降順。招待未完了教員一覧・送信結果ともに、Rails現行仕様にページネーションの明記はないため、全件返却を前提とする（推測: 対象件数が少数であることを前提とした設計と推測される。データ量が増加した場合はページネーションの追加を検討する）

---

# 22. テスト戦略

## Domain Test

- 目的: InvitationStatusの許容値検証、InvitationEligibilityFilterによる送信対象の絞り込みロジック（同校・招待未完了の判定、指定外教員の除外。無効化済みの教員を除外しないこと）、招待メールの送信依頼の結果から`TeacherNotification`の状態への対応づけ（送信成功→`sent`＋`sent_at`、送信失敗→`failed`（`sent_at`なし）、送信対象外→`failed`（`sent_at`なし））を検証する

## UseCase Test

- 目的: ListUnsentTeachers / SendInvitationNotifications（他職員操作権限の確認、送信対象の絞り込み、個々の送信失敗が他に影響しないこと）/ ListNotificationResultsの業務振る舞いを検証する。SendInvitationNotificationsでは、`user`の`RequestInvitationEmail`をテスト用の代替（送信成功・送信失敗・送信対象外・呼び出しの内部エラーを返せるもの）に差し替え、次を確認する
  - 送信対象の教員ごとに、実行方式「即時」で呼ばれること
  - 結果ごとに、`TeacherNotification`が対応づけどおりに作成されること（送信成功のみ`sent_at`あり、送信失敗・送信対象外・呼び出しの内部エラーは`failed`で`sent_at`なし）
  - 1人の結果が送信失敗・送信対象外・内部エラーでも、他の教員の呼び出しと記録が継続されること

## Repository Test

- 目的: TeacherNotificationStoreによる招待未完了教員一覧の絞り込み（無効化済みの教員を除外しないこと）、送信結果の一括作成（`failed`が`sent_at`NULLで作成されること）、日付絞り込みを伴う履歴取得の正確性を検証する

## Handler Test

- 目的: teacher_idsの入力検証、他職員操作権限確認によるHTTPステータス（200/202/403）変換を検証する。招待メールの送信結果（送信失敗・送信対象外を含む）にかかわらず、受付は202であること

## Integration Test

- 目的: エンドポイント経由で招待未完了教員一覧の取得・一括送信・送信結果履歴の参照が、権限制御と絞り込みルールに従って正しく連携して動作することを確認する。`user`の`RequestInvitationEmail`（即時）を実物とし、メール送信基盤とトークン発行のみテスト用の代替にして、次を確認する
  - 送信成功・送信失敗・送信対象外（無効化済みの教員を含めて選択した場合）が、履歴の`status`（`sent` / `failed`）として記録され、`sent_at`が`sent`の場合のみ設定されること
  - 作成時の送信と再送で、同じ件名・本文・リンクのメールが送られること（`user`②「22. テスト戦略」の教員招待通知の再送の項と同じ観点）

---

# 23. Railsとの責務対応

| Rails | Go | 設計方針 |
|---|---|---|
| Controller（TeacherNotificationsController, TeacherNotificationResultsController） | Handler | HTTP入出力のみを担当する |
| Controller内の他職員操作権限チェック | Handler処理内の権限確認（TeacherNotificationStore経由） | ロール確認（Middleware）とは別に、業務権限の判定として明示的に配置する |
| Service（Teacher::TeacherNotificationSenderService） | SendInvitationNotifications（Handler処理）+ InvitationEligibilityFilter | 送信対象の絞り込みロジックを独立させ、一覧取得・一括送信の双方から再利用する |
| Job（Teacher::TeacherNotificationJob）の対象教員ごとのループ・記録 | ベストエフォート非同期処理（規約13章。対象教員ごとのgoroutine） | 対象教員ごとの独立した送信依頼・記録処理として実装する。結果は戻り値で受け取り、`TeacherNotification`を作成時に一度だけ確定させる |
| Job（Teacher::TeacherNotificationJob）の送信部分（トークン発行 + `AuthMailer#invite_user` + `deliver_now`） | `user` Contextの`RequestInvitationEmail`（実行方式「即時」） | 送信手順とメールの内容を、アカウント作成時の送信と共通の1箇所（`user`）に集約する。本Contextはメール送信基盤を直接呼ばず、トークン発行とメール組み立てを持たない |
| Model（TeacherNotification） | TeacherNotification（struct）+ TeacherNotificationStore（同一package） | 送信結果の保持と永続化をstruct/Storeに整理する |
| Serializer（Teacher::UnsentTeacherSerializer, Teacher::TeacherNotificationSerializer） | Presenter / Response DTO | レスポンス整形を分離する |

---

# 24. 採用しなかった設計

## Transaction Script

- 採用しなかった理由: 他職員操作権限の確認・送信対象の絞り込みという複数の判定を手続き型に寄せると、送信結果という明確な永続化対象を持つ本機能にはstructへの責務集約の方が適しているため
- 将来的に採用する可能性: 送信対象の絞り込みルールがさらに単純化された場合は検討できる

## Domain Model

- 採用しなかった理由: TeacherNotification自体に状態遷移や複雑な業務ルールの蓄積がなく、CRUD中心の構造で十分に保守可能であるため
- 将来的に採用する可能性: 招待フローが多段階（招待予約→送信中→送信済み等）の状態遷移を持つようになった場合は再検討の余地がある

## Event Sourcing

- 採用しなかった理由: 送信という単発操作に対する履歴再構築・監査要件が現時点で存在しないため
- 将来的に採用する可能性: 送信履歴の詳細な監査要件が生じた場合に有効な可能性がある

## 招待メールの送信を、本Contextがメール送信基盤を直接呼んで持つ

- 採用しなかった理由: アカウント作成時の送信（`user`）と、同じ文面・トークン・送信手順が2箇所に分かれる。文面やトークンの有効期間、無効化済みの扱いを変更するときに、2箇所を直す必要が生じる（`user`②「24. 採用しなかった設計」の「再送用と作成用で、招待メールの送信を別々に持つ」）
- 将来的に採用する可能性: 想定しない

## 再送も`jobs`へ登録し、結果を非同期に受け取って`TeacherNotification`へ反映する

- 採用しなかった理由: `TeacherNotification`を`pending`で作成して後から`sent` / `failed`へ更新することになり、「作成時に一度だけ確定し、以後遷移しない」構造（4章・6章）が崩れる。完了の通知は、`user`から本Contextへの依存（循環依存。アーキテクチャ規約5章で禁止）を招く。`jobs`の状態を本Contextが参照する方式は、`teacher_notifications`と`jobs`の対応付け（スキーマ変更）と完了を待つ仕組みを要し、`user`の実装の詳細に他Contextが依存することになる（`user`②「24. 採用しなかった設計」）
- 将来的に採用する可能性: 再送に、プロセス再起動後の再実行を含む確実な再試行が求められるようになった場合。その際は、`user`②が示すとおり、本Contextが記録の作成と依頼の登録を同一トランザクションで行い、完了の受け渡し方式を含めて再設計する

---

# 25. 設計判断サマリー

|項目|採用|判断理由|
|-|-|-|
|設計パターン|Active Record|状態遷移のないCRUD中心の業務であり、送信対象の絞り込みは値検証に近く十分に表現できるため|
|Aggregate|不要|TeacherNotificationが1件ごとに独立し、複数Entityにまたがる整合性単位が存在しないため|
|Transaction境界|対象教員ごとに独立（全体を1トランザクションにしない）|1人の送信失敗が他の対象者に影響しないという業務要件のため|
|Domain Event|未採用|対象教員1人につき送信→記録という1対1の処理であり、複数宛先への波及的な非同期通知に該当しないため|
|Value Object|採用（InvitationStatus）|許容される送信結果の値を型として明示するため|
|Domain Service|InvitationEligibilityFilter|送信対象の絞り込みがTeacherNotification単体の責務に収まらないため|
|招待メールの送信|`user` Contextの`RequestInvitationEmail`（実行方式「即時」）を、対象教員ごとのgoroutine内で、トランザクションの外から呼ぶ|メールの内容・トークン発行・送信手順をアカウント作成時の送信と1箇所に集約し、再送の結果を戻り値で受け取って作成時に一度だけ確定させる構造を保つため|
|送信結果の記録|送信成功→`sent`＋`sent_at`、送信失敗・送信対象外→`failed`（`sent_at`なし）|Rails現行の記録（成功は`sent`と`sent_at`、例外は`failed`で`sent_at`なし）を維持するため|
|Authorization|Middleware + Handler|ロール確認と、送信結果履歴参照には課さない非対称な業務権限確認を分離して管理しやすくするため|

---

# 設計差分管理

## Rails現行仕様

- Controllerのcreateアクション冒頭で「他職員操作権限」の有無をチェックしている
- 送信対象の絞り込み（同校・教員・招待未完了。`base_teachers_scope`と`target_teachers_ids`）はController（`Api::V1::Teacher::TeacherNotificationsController`）が行い、Teacher::TeacherNotificationSenderServiceは、絞り込み済みの教員IDでTeacher::TeacherNotificationJobを`perform_later`で登録するのみである
- Teacher::TeacherNotificationJobが、対象教員ごとに、パスワード設定用トークンの発行（`set_reset_password_token`）→`AuthMailer.invite_user`の`deliver_now`（同期送信）→`TeacherNotification`の作成（成功は`status = sent`と`sent_at`）をループで実行し、例外が起きた場合は`status = failed`（`sent_at`なし）を記録する。1人の失敗は他の教員の処理に影響せず、ジョブ内の再試行もない
- 同じ`AuthMailer.invite_user`の送信は、アカウント作成時（`Common::CreateUserService`。`deliver_later`）にも、上記のJobとは別の実装として存在する

## Go設計での変更内容

- 送信対象の絞り込みロジックをInvitationEligibilityFilter（Domain Service相当）として独立させ、一覧取得・一括送信の双方から再利用する
- 対象教員ごとの送信・記録を、規約13章のベストエフォート非同期処理（対象教員ごとのgoroutine）として明確化する
- 他職員操作権限の確認をHandler処理内の明示的なステップとして位置づける
- 招待メールの送信（トークン発行・メール組み立て・メール送信基盤の呼び出し）を本Contextから外し、`user` Contextの`RequestInvitationEmail`（実行方式「即時」）の呼び出しに置き換える。呼び出しは、対象教員ごとのgoroutine内で、トランザクションの外から行い、結果（送信成功／送信失敗／送信対象外）を戻り値として受け取る
- 結果を、送信成功→`sent`＋`sent_at`、送信失敗・送信対象外→`failed`（`sent_at`なし）として`TeacherNotification`へ記録する（Rails現行の記録と同じ。送信対象外は、Rails現行にない結果であり、`failed`に含める）

## 変更理由

- Rails実装では、絞り込みルールがControllerに、送信処理がJobに置かれ、さらに同じ招待メールの送信がアカウント作成時にも別に実装されている。Go設計では絞り込みロジックを独立させることで、影響範囲を限定し保守性・テスト容易性を高める
- 招待メールの送信を`user`の`RequestInvitationEmail`に集約するのは、アカウント作成時の送信と再送で、文面・トークンの有効期間・送信手順の定義が2箇所に分かれている重複を解消するため。実行方式を「即時」とするのは、Rails現行の再送が送信の成否をその場で判定して記録していることに合わせ、`jobs`を介した非同期の結果通知（依存の向きの逆転、`TeacherNotification`の状態遷移の導入、スキーマ変更のいずれかを伴う）を避けるため（`user`②「12. UseCase設計」「24. 採用しなかった設計」）

## Rails現行との差（無効化済みの教員の扱い）

- Rails現行: 送信対象の絞り込み（`base_teachers_scope`。`User.by_high_school(...).teachers.invitation_pending`）は、有効なユーザーへの絞り込み（`active`）を持たず、`deleted_at`が設定された（無効化済みの）教員を除外しない。招待未完了教員一覧（index）にも、送信対象（create）にも含まれうる。Jobは、渡された教員IDのユーザーを`User.where(id:)`で取得し、無効化済みかどうかを確認せずに、トークン発行と送信を行い、成功すれば`sent`を記録する
- Go設計: InvitationEligibilityFilterの対象（同校・招待未完了）は変えず、無効化済みの教員も除外しない。ただし、`user`の`RequestInvitationEmail`は、無効化済みのユーザーを「送信対象外」（メールを送らない）として返す（`user`②「12. UseCase設計」）。本Contextは、これを`failed`（`sent_at`なし）として記録する
- 挙動差の現れ方: 無効化済みで、同校かつ招待待ち（`password_reset_required`が真）の教員が招待未完了の一覧に含まれ、操作者が選択して送信した場合、Rails現行はメールを送って`sent`＋`sent_at`を記録するが、Go設計はメールを送らず`failed`（`sent_at`なし）を記録する。履歴の`status`と、教員一覧の招待状況（`teacher-management`が参照する最新の`status`）に、この違いが現れる。一覧の取得・送信の受付（202）・権限確認には差がない
- 根拠（Rails `main`の確認結果）:
  - 絞り込みに`active`がないこと: `app/controllers/api/v1/teacher/teacher_notifications_controller.rb`（`base_teachers_scope`）
  - Jobが無効化済みを確認しないこと: `app/jobs/teacher/teacher_notification_job.rb`
  - `users.deleted_at`を設定するコードは、`app/`と`lib/`の中では2箇所のみ。管理者の削除（`Api::V1::Admin::AdminsController#destroy`。対象は`User.admins.active`）と、生徒アカウントの統合（`Student::AccountLinkService`。対象は生徒番号で特定した生徒）である。教員を無効化する経路（教員の削除・無効化のAPI。`Api::V1::Teacher::TeachersController`・`Api::V1::Admin::TeachersController`に該当するアクションがない）は、`main`のコードでは確認できなかった
- 現実に差が顕在化するか: 上記のとおり、Rails現行のアプリケーションコードでは、無効化済みで招待待ちの教員が生じる経路を確認できなかったため、通常の操作では差は現れない（推測: `main`のコードを読んだ範囲での判断であり、DBの直接操作・コンソール・マイグレーション等の、コード外の経路による無効化の有無は確認できない）。教員の無効化が将来追加された場合に、差が現れる
- 扱い: 設計は変更しない。無効化済みの教員に招待を送る意味は薄く、`user`が作成時の送信と同じ扱い（送信しない）に統一している。無効化済みを一覧・絞り込みの対象から除外するかどうかは、Rails現行の絞り込みの挙動を変えない方針のため、本Contextでは決めない（`user`②「未解決の論点」の8「無効化ユーザーの扱いの不統一」と同じ論点）

## 影響範囲

- フロントエンドから見たAPI外部仕様は維持する（202・403、レスポンスの構造とも変わらない）
- 既存DBスキーマは維持するため、データ移行は不要。`teacher_notifications`テーブルのスキーマ変更もない
- 送信対象の絞り込みロジックの変更は、教員一覧の招待状況表示（教師教員管理機能、`teacher-management`）には影響しない（`teacher-management`は本Contextの送信結果を参照するのみであり、絞り込みロジック自体には依存しないため）
- 招待メールの内容（件名・本文・リンク・トークンの有効期間）は、`user`②が定める内容に従い、Rails現行の`AuthMailer#invite_user`と同じである。本Contextの変更によって、メールの内容は変わらない
- 送信の失敗・送信対象外の記録が`failed`（`sent_at`なし）となることは、Rails現行の失敗の記録と同じである。無効化済みの教員に対する挙動のみ、上記「Rails現行との差」のとおり異なる
