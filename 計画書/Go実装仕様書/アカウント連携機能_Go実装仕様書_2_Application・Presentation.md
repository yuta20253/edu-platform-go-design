# アカウント連携機能 Go実装仕様書（2. Application・Presentation）

---

# 1. 機能概要

本書はApplication層・Presentation層（9ファイル、テスト込みで約16）を扱う。

## 本書が担当するエンドポイント

|Method|Path|Handler|概要|
|-|-|-|-|
|POST|`/api/v1/student/account_link`|`AccountLinkHandler.LinkAccount`|生徒番号を指定してアカウント連携を実行する|
|POST|`/api/v1/student/account_link/preview`|`AccountLinkHandler.PreviewAccountLink`|統合を確定する前に、対象アカウントの学校・学年・学級名を取得する|

## 本書の位置づけ

本書は`アカウント連携機能_Go実装仕様書_1_Domain・Infrastructure.md`（以下「1_Domain・Infrastructure」）のスライス編である。

1_Domain・Infrastructureが定めるもの（本書では繰り返さない）:

- Domain層のすべて（Entity・Value Object・Repository Interface・Domain Service・Domain Error）
- Infrastructure層のすべて（Repository実装・GORMモデル）
- `context.go`、およびTransaction・Validation・Authorization・Error・GORMの実装方針

本書が定めるもの:

- 上記2エンドポイントのApplication層（`LinkAccountUseCase`・`PreviewAccountLinkUseCase`・DTO）
- 同Presentation層（Handler・Request / Response DTO・`routes.go`）
- 同API仕様・テストケース設計

## 採用設計パターン

Domain Model（②「4. 設計パターン」）。1_Domain・Infrastructure「1. 機能概要」を参照。

1_Domain・Infrastructureを実装・マージしたうえで本書の実装に着手する。

---

# 2. ディレクトリ構成

1_Domain・Infrastructure「2. ディレクトリ構成」の機能全体のファイル一覧のうち、本書が担当するものは以下の9である。

```
internal/accountlinking/application/command/link_account_command.go
internal/accountlinking/application/dto/link_account_result.go
internal/accountlinking/application/usecase/errors.go
internal/accountlinking/application/usecase/link_account_usecase.go
internal/accountlinking/application/usecase/preview_account_link.go
internal/accountlinking/presentation/handler/account_link_handler.go
internal/accountlinking/presentation/request/account_link_request.go
internal/accountlinking/presentation/response/account_link_response.go
internal/accountlinking/presentation/routes.go
```

---

# 3. Domain層設計

本書では定義しない。1_Domain・Infrastructure「3. Domain層設計」を参照。

---

# 4. クラス図

1_Domain・Infrastructure「4. クラス図」を参照。本書はUseCase・Handlerを追加するのみである。

---

# 5. 状態遷移図

1_Domain・Infrastructure「5. 状態遷移図」を参照。本書の`LinkAccountUseCase`が、`MergeCandidateAccount.MarkMerged`による遷移（未利用→論理削除済み）を発生させる唯一のUseCaseである。`PreviewAccountLinkUseCase`は読み取りのみで遷移を発生させない。

---

# 6. Application層設計

## DTO（Command / Query）

|struct名|フィールドと型|区分|
|-|-|-|
|`LinkAccountCommand`|`TargetUserID uint`（current userのID）、`StudentNumber string`（生の入力値）|Command|
|`LinkAccountResult`|`Message string`（成功メッセージ）|出力|

Query DTOは設けない（本機能はLinkAccountというCommand操作のみを提供するため）。

## UseCase

### LinkAccountUseCase（`application/usecase/link_account_usecase.go`）

②「12. UseCase設計」LinkAccountに対応する。

- struct名: `LinkAccountUseCase`
- コンストラクタが受け取る依存: `repository.AccountRepository`, `repository.AccountLinkAuditRepository`, `*service.AccountLinkEligibilityPolicy`, `service.AccountLinkRateLimiter`, `transaction.Manager`（アーキテクチャ規約「11. Transaction実装パターン」）
- 公開メソッド: `(u *LinkAccountUseCase) Execute(ctx context.Context, cmd command.LinkAccountCommand) (dto.LinkAccountResult, error)`
- 処理ステップ（②「13. シーケンス図・処理フロー図」に対応）:
  1. `valueobject.NewStudentNumber(cmd.StudentNumber)`で書式検証する。不正時は`domainerrors.ErrInvalidStudentNumberFormat`を返す（監査ログは記録しない。書式不正の時点では統合元候補が特定できないため）
  2. `AccountLinkAuditRepository.CountRecentAttempts(ctx, cmd.TargetUserID, 現在時刻-10分)`で直近の試行件数を取得する
  3. `AccountLinkRateLimiter.IsExceeded(件数)`で上限超過を判定する。超過時は`usecase.ErrRateLimitExceeded`を返す（監査ログは記録しない。トランザクションの外側で行う判定であるため）
  4. `AccountRepository.FindMergeCandidateByStudentNumber(ctx, studentNumber)`で統合元候補を検索する。該当なしは`domainerrors.ErrMergeCandidateNotFound`を返す（監査ログは記録しない）
  5. `AccountRepository.FindTargetAccountByID(ctx, cmd.TargetUserID)`で統合先（current user）のEntityを取得する
  6. `AccountRepository.ExistsUsageData(ctx, candidate.ID())`で統合元候補の利用データ有無を取得する
  7. `AccountLinkEligibilityPolicy.Judge(candidate, target, hasUsageData)`で適格性を判定する
  8. 適格性を満たさない場合: `AccountLinkEligibility`の各条件を`IsAlreadyLinked → IsAlreadyUsed → IsDifferentSchool → HasUsageData`の順に判定し、最初に該当した条件に対応するDomain Error（`ErrAlreadyLinked` / `ErrAlreadyUsed` / `ErrDifferentSchool` / `ErrHasUsageData`）を確定する。`entity.NewAccountLinkAudit`で失敗ログ（`valueobject.NewAccountLinkResultFailed()`）を生成し、`AccountLinkAuditRepository.Save`で記録したうえで、確定したDomain Errorを返す（トランザクションの外側で記録する。「11. Transaction実装方針」参照）
  9. 適格性を満たす場合: `transaction.Manager.WithinTransaction`内で以下を実行する
     - `target.ApplyMergedRegistration(candidate)`
     - `AccountRepository.UpdateRegistration(ctx, target)`
     - `candidate.MarkMerged(現在時刻)`
     - `AccountRepository.MarkMerged(ctx, candidate)`
     - `entity.NewAccountLinkAudit`で成功ログ（`valueobject.NewAccountLinkResultSuccess()`）を生成し、`AccountLinkAuditRepository.Save`で記録する
  10. `LinkAccountResult`（成功メッセージ）を返す
- トランザクション境界: ステップ9（学籍情報コピー〜統合元論理削除〜成功ログ記録）のみをトランザクション内で実行する。詳細は「11. Transaction実装方針」参照
- 発生しうるApplication Error: `usecase.ErrRateLimitExceeded`。Domain Error（`ErrInvalidStudentNumberFormat` / `ErrMergeCandidateNotFound` / `ErrAlreadyLinked` / `ErrAlreadyUsed` / `ErrDifferentSchool` / `ErrHasUsageData`）はUseCaseで変換せずそのまま上位（Presentation）へ伝播させる

`application/usecase/errors.go`に、Application層のセンチネルエラーとして`ErrRateLimitExceeded`を定義する。

### PreviewAccountLinkUseCase（`application/usecase/preview_account_link.go`）

- 目的: 統合を確定する前に、対象アカウントの所属情報（高校名・学年の表示名・学級名）を返す（②「12. UseCase設計」）
- 関数シグネチャ: `func (u *PreviewAccountLinkUseCase) Execute(ctx context.Context, cmd PreviewAccountLinkCommand) (PreviewAccountLinkResult, error)`
- `PreviewAccountLinkCommand`のフィールド: `CurrentUserID uint` / `StudentNumber string`
- `PreviewAccountLinkResult`のフィールド: `HighSchoolName *string` / `GradeDisplayName *string` / `SchoolClassName *string`（いずれも対象アカウントに該当データが紐づいていない場合は`nil`）
- コンストラクタが受け取る依存: `AccountRepository`, `*service.AccountLinkEligibilityPolicy`, `AccountLinkRateLimiter`（`LinkAccountUseCase`と同一）
- 処理ステップ:
  1. `StudentNumber` Value Objectを生成する（書式検証）
  2. `AccountRepository.FindTargetAccountByID`で統合先（操作者）を取得する
  3. `AccountLinkRateLimiter.IsExceeded`で試行回数を確認する
  4. `AccountRepository.FindMergeCandidateByStudentNumber`で統合元候補を取得する
  5. `AccountLinkEligibilityPolicy.Judge`で適格性を判定する。判定順序・返すDomain Errorは`LinkAccountUseCase`と同一
  6. 適格性を満たす場合、統合元候補の高校名・学年の表示名・学級名を返す
- トランザクション境界: 使用しない（読み取りのみ）
- 監査ログ: 記録しない（データを変更しないため）
- 返さない情報: 氏名・メールアドレスなどの個人情報は含めない（②「12. UseCase設計」）

**②からの補足**: 適格性判定を`LinkAccountUseCase`と共有するため、`AccountLinkEligibilityPolicy`と`AccountRepository`の検索メソッドを両UseCaseが同じ形で呼ぶ構成とした。Rails現行は`Student::AccountLinkFinder`として検証・検索を共通化しており、同じ分担である。

---

# 7. シーケンス図・処理フロー図

## シーケンス図（LinkAccount）

```mermaid
sequenceDiagram
    participant H as AccountLinkHandler
    participant UC as LinkAccountUseCase
    participant ALAR as AccountLinkAuditRepository
    participant RL as AccountLinkRateLimiter
    participant AR as AccountRepository
    participant EP as AccountLinkEligibilityPolicy
    participant TM as transaction.Manager

    H->>UC: Execute(ctx, LinkAccountCommand)
    UC->>ALAR: CountRecentAttempts(userID, since)
    ALAR-->>UC: count
    UC->>RL: IsExceeded(count)
    RL-->>UC: bool
    UC->>AR: FindMergeCandidateByStudentNumber(studentNumber)
    AR-->>UC: MergeCandidateAccount
    UC->>AR: FindTargetAccountByID(userID)
    AR-->>UC: TargetAccount
    UC->>AR: ExistsUsageData(candidateID)
    AR-->>UC: bool
    UC->>EP: Judge(candidate, target, hasUsageData)
    EP-->>UC: AccountLinkEligibility
    alt 適格性を満たす
        UC->>TM: WithinTransaction(fn)
        TM->>AR: UpdateRegistration(target)
        TM->>AR: MarkMerged(candidate)
        TM->>ALAR: Save(成功ログ)
        TM-->>UC: error
        UC-->>H: LinkAccountResult
    else 適格性を満たさない
        UC->>ALAR: Save(失敗ログ)
        UC-->>H: Domain Error
    end
```

## 処理フロー図（LinkAccount）

適格性判定の分岐が多いため、フローチャートで可視化する（②「13. シーケンス図・処理フロー図」の処理フロー図を実装単位に具体化）。

```mermaid
flowchart TD
    A[LinkAccountUseCase.Execute] --> B{StudentNumberの形式は正しいか}
    B -- No --> Z1[ErrInvalidStudentNumberFormat]
    B -- Yes --> C{直近10分の試行が5回以内か}
    C -- No --> Z2[ErrRateLimitExceeded]
    C -- Yes --> D{生徒番号に一致する候補が存在するか}
    D -- No --> Z3[ErrMergeCandidateNotFound]
    D -- Yes --> E[TargetAccount取得・利用データ有無確認]
    E --> F[AccountLinkEligibilityPolicy.Judge]
    F --> G{IsSelf}
    G -- true --> H1[監査ログ記録: 失敗] --> Z4[ErrAlreadyLinked]
    G -- false --> I{IsAlreadyUsed}
    I -- true --> H2[監査ログ記録: 失敗] --> Z5[ErrAlreadyUsed]
    I -- false --> J{IsDifferentSchool}
    J -- true --> H3[監査ログ記録: 失敗] --> Z6[ErrDifferentSchool]
    J -- false --> K{HasUsageData}
    K -- true --> H4[監査ログ記録: 失敗] --> Z7[ErrHasUsageData]
    K -- false --> L[トランザクション開始: 学籍情報コピー・論理削除]
    L --> M[監査ログ記録: 成功]
    M --> N[LinkAccountResultを返す]
```

---

# 8. Infrastructure層設計

本書では扱わない。1_Domain・Infrastructure「8. Infrastructure層設計」を参照。本書のUseCaseが使うInfrastructure層の要素: `AccountRepository`・`AccountLinkAuditRepository`の実装（いずれも`infrastructure/gormrepo/`）。

---

# 9. Presentation層設計

## Handler

### AccountLinkHandler（`presentation/handler/account_link_handler.go`）

- struct名: `AccountLinkHandler`
- 対応する呼び出し先: `*usecase.LinkAccountUseCase`, `*usecase.PreviewAccountLinkUseCase`
- メソッド一覧:

|メソッド|HTTPメソッド|パス|
|-|-|-|
|`(h *AccountLinkHandler) LinkAccount(c *gin.Context)`|POST|/api/v1/student/account_link|
|`(h *AccountLinkHandler) PreviewAccountLink(c *gin.Context)`|POST|/api/v1/student/account_link/preview|

- 処理順序（`LinkAccount`）:
  1. Middlewareで設定済みのcurrent user（student）をcontextから取得する
  2. `AccountLinkRequest`をJSONボディからバインドし、Presentation Validation（1_Domain・Infrastructure「12. Validation実装方針」）を行う
  3. `command.LinkAccountCommand{TargetUserID: current userのID, StudentNumber: request.StudentNumber}`を組み立てる（統合先を必ずcurrent userに固定し、クライアントからの指定を許容しない。②「16. Authorization設計」）
  4. `LinkAccountUseCase.Execute`を呼び出す
  5. 戻り値のエラーを「14. Error実装方針」に従いHTTPステータスへ変換する
  6. 成功時は`AccountLinkResponse`へ変換し200で返す
- 処理順序（`PreviewAccountLink`）: 同様にRequestをバインドし、`PreviewAccountLinkUseCase.Execute`を呼び出して`AccountLinkPreviewResponse`へ変換する

## Request / Response DTO

### Request（`presentation/request/account_link_request.go`）

|struct名|フィールドと型|バリデーションタグ／チェック内容|
|-|-|-|
|`AccountLinkRequest`|`StudentNumber string`|`json:"student_number" binding:"required"`（②「15. Validation設計」Presentation 型チェック・必須チェック。詳細な書式検証はDomain側の`StudentNumber` VOに委ねる）|

### Response（`presentation/response/account_link_response.go`）

|struct名|フィールドと型|
|-|-|
|`AccountLinkResponse`|`Message string`|
|`AccountLinkPreviewResponse`|`HighSchoolName *string`, `GradeDisplayName *string`, `SchoolClassName *string`|

## Routing（`presentation/routes.go`。本書の担当）

|Method|Path|Handler|
|-|-|-|
|POST|/api/v1/student/account_link|`AccountLinkHandler.LinkAccount`|
|POST|/api/v1/student/account_link/preview|`AccountLinkHandler.PreviewAccountLink`|

認証Middleware（本人確認）・認可Middleware（student roleチェック）を経由する（②「16. Authorization設計」Middleware、1_Domain・Infrastructure「13. Authorization実装方針」参照）。

---

# 10. API仕様

②「19. API仕様」に基づく実装対象Endpoint一覧。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|
|POST|/api/v1/student/account_link|AccountLinkHandler.LinkAccount|`AccountLinkRequest`|`AccountLinkResponse`|200|
|POST|/api/v1/student/account_link/preview|AccountLinkHandler.PreviewAccountLink|`AccountLinkRequest`|`AccountLinkPreviewResponse`|200|

## Errorケース

|条件|Status Code|Error内容|
|-|-|-|
|`student_number`未入力|422（②「19. API仕様」には`400`のみ明記されており、Presentationバインドエラーの扱いは他機能のパターンに合わせて422とした。**推測**。「17. ②からの補足事項」参照）|Presentation Validationエラー|
|`StudentNumber`の書式が不正（`ErrInvalidStudentNumberFormat`）|400|②「17. Error設計」生徒番号の形式が不正|
|生徒番号に一致するアカウントが存在しない（`ErrMergeCandidateNotFound`）|404|②「17. Error設計」対象アカウントなし。存在の有無を推測されないよう、利用開始済みと同じ文言を返す|
|操作者アカウントが既に生徒番号を持っている（`ErrAlreadyLinked`）|400|②「17. Error設計」|
|統合対象アカウントが既に利用開始済み（`ErrAlreadyUsed`）|400|②「17. Error設計」|
|統合対象アカウントの所属高校が異なる（`ErrDifferentSchool`）|400|②「17. Error設計」|
|統合対象アカウントに利用データが存在する（`ErrHasUsageData`）|400|②「17. Error設計」|
|直近10分の試行回数が上限を超過（`ErrRateLimitExceeded`）|429|②「17. Error設計」|
|未認証|401|認証エラー（Middleware）|
|studentロールでない|403|認可エラー（Middleware）|
|DB接続失敗等のInfrastructure Error|500|内部エラー|

Errorケースは2つのエンドポイントで共通である。`PreviewAccountLink`は適格性判定を`LinkAccount`と共有するため、確認画面で通った入力が確定時に落ちることがない。

---

# 11. Transaction実装方針

本書では扱わない。1_Domain・Infrastructure「11. Transaction実装方針」を参照。

---

# 12. Validation実装方針

本書では扱わない。1_Domain・Infrastructure「12. Validation実装方針」を参照。本書のRequest DTO（`AccountLinkRequest`）は同方針の`AccountLinkRequest`の行に対応する。

---

# 13. Authorization実装方針

本書では扱わない。1_Domain・Infrastructure「13. Authorization実装方針」を参照。

---

# 14. Error実装方針

本書では扱わない。1_Domain・Infrastructure「14. Error実装方針」を参照（上記「10. API仕様」のErrorケースは同方針の表の抜粋である）。

---

# 15. GORM / DBクエリ設計

本書では扱わない。1_Domain・Infrastructure「15. GORM / DBクエリ設計」を参照。本書固有のGORMモデル・クエリはない。

---

# 16. テストケース設計（本書の担当分）

1_Domain・Infrastructureが担当するDomain Test・Repository Testは同文書を参照。本書は本スライスのUseCase・Handlerのテストを扱う。

## UseCase Test

|対象|テストケース|
|-|-|
|`LinkAccountUseCase`|適格性をすべて満たす場合、学籍情報がコピーされ候補が論理削除され成功ログが記録されること|
|`LinkAccountUseCase`|連携済み・利用開始済み・所属高校不一致・利用データありのそれぞれで対応するDomain Errorが返り、失敗ログが記録され、統合先・統合元のデータが変更されないこと|
|`LinkAccountUseCase`|生徒番号に一致するアカウントが存在しない場合、`ErrMergeCandidateNotFound`が返り監査ログが記録されないこと|
|`LinkAccountUseCase`|直近10分の試行回数が上限を超えている場合、`ErrRateLimitExceeded`が返り統合処理が実行されないこと|
|`LinkAccountUseCase`|トランザクション内の処理が途中で失敗した場合、ロールバックされ統合先・統合元のデータが変更されないこと|
|`PreviewAccountLinkUseCase`|適格性を満たす場合に高校名・学年の表示名・学級名が返ること／氏名・メールアドレスを含まないこと|

## Handler Test

|対象|テストケース|
|-|-|
|`AccountLinkHandler.LinkAccount`|`student_number`未入力時に422が返ること|
|`AccountLinkHandler.LinkAccount`|正常時に200と成功メッセージが返ること|
|`AccountLinkHandler.LinkAccount`|未認証・非student roleでのアクセスが401/403となること|
|`AccountLinkHandler.PreviewAccountLink`|正常時に200と所属情報が返ること|

## Integration Test

|対象|テストケース|
|-|-|
|アカウント連携成功|エンドポイント経由で統合が成功し、統合先の学籍情報が更新され、統合元が論理削除されること|
|アカウント連携失敗（各適格性違反）|エンドポイント経由で失敗パターンごとに正しいエラーが返り、データが変更されず、失敗ログのみが記録されること|
|レート制限|直近10分で6回目の試行が429となること|

---

# 17. ②からの補足事項

②に明記がなく、実装のために本書で補った内容である。1_Domain・Infrastructure「17. ②からの補足事項」に記載した内容に加え、本書固有のものを以下に示す。

|判断した内容|判断理由|推測か|
|-|-|-|
|`student_number`未入力（Presentation Validation失敗）時のHTTP Statusを422とした|②「19. API仕様」のStatus Code一覧には`400`（形式不正・適格性違反）のみが明記され、Presentationレベルのバインドエラーの扱いは明記がない。他の学生向け機能がPresentation Validation失敗を422として扱っている前例に倣った|推測|
|`AccountLinkPreviewResponse`を`AccountLinkResponse`と同じファイルに定義した|②自動生成プロンプトの「文書の分割」方針上、新規ファイルを増やす必要がなく、Previewが`LinkAccount`と密接に関連するレスポンスであるため|推測ではない（実装判断）|

上記以外の設計判断は1_Domain・Infrastructureの記載をそのまま踏襲する。
