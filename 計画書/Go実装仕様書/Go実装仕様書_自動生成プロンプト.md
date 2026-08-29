# Go実装仕様書 自動生成プロンプト

## 目的

添付された「〇〇機能_Go移行・設計仕様書」をもとに、
Goでどのように実装するかを具体化し、
「〇〇機能_Go実装仕様書」を作成してください。

この資料の目的は、

> 「②で決めた設計を、Goでどのようなコード構成として実装するか」

を明確にすることです。

本資料は、②で確定した設計方針を前提とした実装レベルの仕様書です。

設計方針そのものの再検討は目的ではありません。

---

# 基本方針

以下を厳守してください。

- ②Go移行・設計仕様書で決定した内容（Bounded Context・設計パターン・Aggregate・Entity・Value Object・Repository・UseCase・Transaction境界・Validation方針・Authorization方針・Error設計・Domain Event・API互換方針・DB方針・テスト戦略）を変更しない
- ②の設計判断に矛盾する実装案を提示しない
- ②に記載のない実装詳細を補う場合は、必ず「②からの補足」であることを明示する
- Railsの実装詳細（①）を実装仕様の根拠にする場合は、参照している旨を明示する
- `規約/`ディレクトリ配下の規約ドキュメント（アーキテクチャ規約.md・コーディング規約.md・Gorm規約.md）を厳守する。実装仕様の記載内容がこれらの規約と矛盾してはならず、規約に定めのない事項について独自のルールを新設しない
- 完全に動作するアプリケーションコード（関数本体のロジック）は書かない
- package構成・struct定義・interfaceのメソッドシグネチャ・method一覧・クエリ内容など、実装者が迷わず着手できる粒度の設計情報を記載する
- **②が採用した設計パターンに応じて、実装構造そのものを変える。** Domain Modelと同じフルレイヤー構成（domain/application/infrastructure/presentationのフルセット）を、Transaction ScriptやActive Recordを採用した機能にまで一律で適用しない。パターンごとの構造は`規約/アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」に従う

---

# ②仕様書の分析方針

実装仕様を作成する前に、
②Go移行・設計仕様書から以下を抽出してください。

- 採用した設計パターン（Transaction Script / Active Record / Domain Model / Event Sourcing）
- Bounded Context名と責務
- Aggregate構成
- Entity一覧とその責務
- Value Object一覧
- Domain Service一覧
- Repository一覧と責務
- UseCase一覧と入出力
- Transaction境界
- Validation方針（Presentation / Domain）
- Authorization方針（Middleware / Handler / UseCase / Domain）
- Error設計（Domain / Application / Infrastructure）
- Domain Event（採用している場合）
- API互換方針
- DB方針（既存Schema利用有無）
- テスト戦略

②に記載のない項目は実装仕様書側で判断せず、②に立ち返って確認してください。
どうしても②に情報がなく実装のために判断が必要な場合は、「推測」と明記してください。

---

# 前提アーキテクチャ

以下を前提としてください。

- Clean Architecture
- Bounded Context
- Gin
- GORM
- MySQL

ディレクトリ構成は、②が採用した設計パターンによって異なります。詳細は`規約/アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」を参照してください。以下に各パターンの構造を示します。

## Domain Model / Event Sourcing採用時

```
internal
└── {context}
    ├── domain
    │   ├── entity
    │   ├── valueobject
    │   ├── repository
    │   ├── service
    │   ├── specification
    │   ├── event
    │   └── errors
    ├── application
    │   ├── dto
    │   ├── command
    │   ├── query
    │   └── usecase
    ├── infrastructure
    │   ├── persistence
    │   │   └── gorm
    │   ├── repository
    │   ├── mail
    │   ├── cache
    │   └── queue
    └── presentation
        ├── handler
        ├── request
        ├── response
        └── routes.go
```

②で当該機能に不要と判断されたディレクトリ（例：Domain Eventを採用しない機能のevent、非同期処理のないqueue等）は無理に埋めず、「対象外」と明記してください。

## Active Record採用時

domain/infrastructureのレイヤー分離、usecase層を設けません。Entity相当のstructと永続化操作（Store）を同一packageに置きます。

```
internal
└── {context}
    ├── model.go
    ├── store.go
    └── presentation
        ├── handler
        ├── request
        ├── response
        └── routes.go
```

## Transaction Script採用時

domain層・usecase層・Repository Interfaceを設けません。1つの業務操作を1つの関数として実装します。

```
internal
└── {context}
    ├── application
    │   └── list_xxx.go
    ├── infrastructure
    │   └── xxx_query.go
    └── presentation
        ├── handler
        ├── response
        └── routes.go
```

②の「4. 設計パターン」で採用パターンを確認し、該当する構造のみを本書「2. ディレクトリ構成」以降に反映してください。Domain Modelを前提とした節（Domain層設計のEntity/Value Object/Repository Interface等）は、Transaction Script/Active Record採用時にはそのまま使わず、後述の各節の指示に従って読み替えてください。

---

# 出力ファイル

〇〇機能_Go実装仕様書.md

---

# 出力フォーマット

# 〇〇機能 Go実装仕様書

---

# 1. 機能概要

②Go移行・設計仕様書の要点を要約してください。

記載内容:

- 機能概要
- 採用設計パターンとその理由（②からの要約）
- 本書が対象とする実装範囲

---

# 2. ディレクトリ構成

本機能で実際に作成するファイル・ディレクトリを整理してください。

記載内容:

- 対象Bounded Context名
- ②で採用した設計パターン（Transaction Script / Active Record / Domain Model / Event Sourcing）
- 採用パターンに対応する構造（前提アーキテクチャ節参照）に基づく、作成するディレクトリ一覧
- 作成するファイル一覧（パス単位）

例（Domain Model採用時）:

```
internal/task/domain/entity/task.go
internal/task/domain/valueobject/task_status.go
internal/task/domain/repository/task_repository.go
internal/task/application/usecase/create_task.go
internal/task/infrastructure/repository/task_repository.go
internal/task/presentation/handler/task_handler.go
```

例（Active Record採用時）:

```
internal/teacher_directory/model.go
internal/teacher_directory/store.go
internal/teacher_directory/presentation/handler/teacher_handler.go
```

例（Transaction Script採用時）:

```
internal/dashboard/application/list_recent_goals.go
internal/dashboard/infrastructure/goal_query.go
internal/dashboard/presentation/handler/dashboard_handler.go
```

---

# 3. Domain層設計

本節はDomain Model / Event Sourcing採用時に適用します。

- **Transaction Script採用時**: 本節は「対象外（Transaction Script採用のため、Domain層を設けない）」と記載し、代わりに「6. Application層設計」で関数の入出力・処理ステップを記載してください
- **Active Record採用時**: 「Entity」の代わりに「Model（Entity相当）」として、struct定義・フィールド・Validate()等の検証メソッドを記載してください。「Value Object」「Repository Interface」「Domain Service」は原則「対象外」とし、検証ルールはModelのメソッドとして記載してください。「Domain Error」は「struct/Storeが返すエラー」として記載してください

## Entity

Entityごとに以下を記載してください。

- struct名
- 保持するフィールドと型
- 各フィールドの意味
- 公開するmethod一覧（メソッド名・引数・戻り値・責務。実装ロジックは書かない）
- 不変条件（コンストラクタ／ファクトリで保証する内容）

## Value Object

Value Objectごとに以下を記載してください。

- struct名
- 保持するフィールドと型
- 生成時に検証するルール
- 公開するmethod一覧

## Repository Interface

Repositoryごとに以下を記載してください。

- interface名
- メソッドシグネチャ一覧（引数・戻り値）
- 各メソッドの責務

## Domain Service

②でDomain Serviceが採用されている場合、以下を記載してください。

- struct/interface名
- メソッドシグネチャ
- 責務

## Domain Event

②でDomain Eventが採用されている場合、以下を記載してください。

- イベントstruct名
- 保持するフィールド
- 発火元（どのEntity／UseCaseから発火するか）

## Domain Error

- エラー種別ごとの型／変数定義方針
- 発生条件

---

# 4. クラス図

「3. Domain層設計」で記載したEntity・Value Object・Repository Interface・Domain Serviceの関係を、Mermaidのクラス図（`classDiagram`）でGoのstruct/interfaceレベルの粒度で可視化してください。コードブロックの言語指定は`mermaid`としてください。②のクラス図（②文書に記載がある場合）より具体的に、実際に定義するstruct名・フィールド・メソッドシグネチャを反映してください。

Active Record採用時はModel（Entity相当のstruct）とStoreの関係を、Transaction Script採用時は主要なstruct（あれば）と関数群の呼び出し関係を示してください。単純すぎて図示する意味がない場合は省略し、理由を記載してください。

---

# 5. 状態遷移図

②の状態遷移図（記載がある場合）をもとに、実際のstructのフィールド値（例: `TaskStatus`という独自型の各定数）を用いてMermaidの状態遷移図（`stateDiagram-v2`）を記載してください。コードブロックの言語指定は`mermaid`としてください。状態を持たない機能では省略し、理由を記載してください。

---

# 6. Application層設計

本節の「UseCase」は、②が採用した設計パターンに応じて以下のように読み替えてください。

- **Domain Model / Event Sourcing採用時**: 以下の記載どおり、UseCase struct + Repository Interfaceとして記載する
- **Active Record採用時**: usecase層を設けないため、「UseCase」の代わりに「Handlerが直接呼び出すStoreのメソッド」として、Handler側の処理ステップの中で記載する（本節では「対象外（Active Record採用のため、usecase層を設けない）」と明記し、「9. Presentation層設計」のHandler処理順序に統合して記載する）
- **Transaction Script採用時**: 「UseCase」の代わりに、`application/`直下に置く関数として記載する（struct化しない）。関数ごとに以下と同等の項目（関数名・引数・戻り値・処理ステップ・呼び出すinfrastructure関数）を記載する

## DTO（Command / Query）

UseCaseへの入力・出力として使うDTOを整理してください。

- struct名
- フィールドと型
- Command / Queryどちらに属するか

## UseCase

UseCaseごとに以下を記載してください。

- struct名
- コンストラクタが受け取る依存（Repository・Domain Service等）
- 公開メソッドのシグネチャ（引数・戻り値）
- 処理ステップ（呼び出し順序。ロジックそのものは書かない）
- トランザクション境界（②の記載を実装単位に落とし込む）
- 発生しうるApplication Error

---

# 7. シーケンス図・処理フロー図

## シーケンス図

主要なUseCase（Active Record採用時はHandler処理、Transaction Script採用時は関数）について、Handler → UseCase/Store/関数 → Repository/Storeという実際の呼び出し関係をMermaidのシーケンス図（`sequenceDiagram`）で記載してください。コードブロックの言語指定は`mermaid`としてください。②のシーケンス図（記載がある場合）より、実際に定義するstruct/関数名を用いて具体化してください。

## 処理フロー図

分岐の多い処理（複数の検証・条件判定を経るUseCase等）について、Mermaidのフローチャート（`flowchart TD`）で処理の分岐を記載してください。コードブロックの言語指定は`mermaid`としてください。単純なCRUDで分岐がほとんどない場合は省略し、理由を記載してください。

---

# 8. Infrastructure層設計

## Repository実装（Domain Model / Event Sourcing採用時）

Repositoryごとに以下を記載してください。

- 実装struct名
- 対応するGORMモデル
- 各メソッドで発行するクエリ内容（条件・ソート・ページネーション等。SQL文そのものは書かない）
- Entity ⇔ GORMモデルの変換方針

## Store実装（Active Record採用時）

「3. Domain層設計」で定義したModelと同一packageに置くStoreについて、以下を記載してください。

- struct名（`〇〇Store`）
- 対応するGORMモデル（Modelと同一structをGORMタグ付きで扱うか、別途変換するかを明記する）
- 各メソッド（Save/FindByID/FindAll/Delete等）で発行するクエリ内容

## Infrastructure関数（Transaction Script採用時）

`infrastructure/`直下に置くDBアクセス関数について、以下を記載してください。

- 関数名・引数・戻り値
- 発行するクエリ内容

## 外部連携実装

Mail・Cache・Queue等が②で必要とされている場合、以下を記載してください。

- 実装対象
- 呼び出し元（どのUseCaseから利用するか）
- 実装方針

不要な場合は「対象外」と明記してください。

---

# 9. Presentation層設計

## Handler

Handlerごとに以下を記載してください。

- struct名
- 対応する呼び出し先（Domain Model/Event Sourcing採用時はUseCase、Active Record採用時はStore、Transaction Script採用時はapplication関数）
- メソッド一覧（HTTPメソッド・パスとの対応）
- 処理順序（入力バインド → Validation → 呼び出し先の実行 → レスポンス変換、等）。Active Record/Transaction Script採用時は、UseCase層を経由しない分、この処理順序の中に本来UseCaseが担う手順（権限チェック・呼び出し順序等）を明記する

## Request / Response DTO

- struct名
- フィールドと型
- バリデーションタグ／チェック内容

## Routing

- 登録するルート一覧（Method・Path・Handler対応）

---

# 10. API仕様

②のAPI仕様（②文書「19. API仕様」）をもとに、実装対象のEndpointを一覧化してください。

|Method|Path|Handler|Request|Response|Status Code|
|-|-|-|-|-|-|

各Endpointについて、Errorケースを整理してください。

|条件|Status Code|Error内容|
|-|-|-|

---

# 11. Transaction実装方針

②で定義したTransaction境界を、具体的にどのレイヤー・どのコードで開始／終了するか記載してください。

記載内容:

- Transaction開始箇所（Domain Model/Event Sourcing採用時はUseCase内のどの時点か、Active Record採用時はStoreのどのメソッド内か、Transaction Script採用時はapplication関数内のどの時点か）
- Transaction終了箇所（Commit / Rollback条件）
- 複数Repository（またはStore・関数）にまたがる場合の扱い

---

# 12. Validation実装方針

②のValidation設計（②文書「15. Validation設計」の「バリデーション仕様」表）を実装レベルに落とし込んでください。

## Presentation

- Request DTOでのチェック内容（型・必須・フォーマット）

|フィールド|struct名|バリデーションタグ|エラーメッセージ|
|-|-|-|-|

## 業務ルール検証

- Domain Model/Event Sourcing採用時: Entity／Value Object生成時に検証する内容、UseCase内で判定する業務ルール
- Active Record採用時: Modelのメソッド（Validate()等）で検証する内容
- Transaction Script採用時: application関数内のガード節で検証する内容

---

# 13. Authorization実装方針

②のAuthorization設計を実装レベルに落とし込んでください。

記載内容:

- Middlewareで行う処理
- Handlerで行う処理
- Domain Model/Event Sourcing採用時: UseCaseで行う処理、Domainで行う処理
- Active Record採用時: Store／Modelで行う処理
- Transaction Script採用時: application関数で行う処理

---

# 14. Error実装方針

②のError設計（②文書「17. Error設計」の「エラー仕様」表）を実装レベルに落とし込んでください。

記載内容:

- Domain Error → Application Errorへの変換方針（アーキテクチャ規約「12. Error変換パターン（AppError）」に従う）
- Application Error → HTTPレスポンスへの変換方針（Status Code対応表）
- Infrastructure Errorのハンドリング方針

|業務シナリオ|Error変数名／型|発生層|HTTP Status|
|-|-|-|-|

---

# 15. GORM / DBクエリ設計

②のDB操作仕様（②文書「21. DB操作仕様」）をもとに、実装で必要なクエリ・モデル定義方針を整理してください。

記載内容:

- 利用するGORMモデルとテーブルの対応
- 主要クエリの条件・ソート・ページネーション方針
- 既存Schemaに対する変更が②で提案されている場合、その反映方針

|Repository/Store|メソッド|対象テーブル|条件|結合|
|-|-|-|-|-|

SQL文そのものは記載しないでください。

---

# 16. テストケース設計

②のテスト戦略を、具体的なテストケース単位に落とし込んでください。テスト区分の名称は、②が採用した設計パターンに応じて読み替えてください。

- Domain Model/Event Sourcing採用時: 下記の区分をそのまま使用する
- Active Record採用時: 「Domain Test」→「Model Test（Validate()等の検証ロジック）」、「UseCase Test」は「対象外」、「Repository Test」→「Store Test」
- Transaction Script採用時: 「Domain Test」「Repository Test」は「対象外」、「UseCase Test」→「Application関数 Test」

## Domain Test

|対象|テストケース|

## UseCase Test

|対象|テストケース|

## Repository Test

|対象|テストケース|

## Handler Test

|対象|テストケース|

## Integration Test

|対象|テストケース|

---

# 17. ②からの補足事項

②に明記がなく、実装のために追加で判断した内容があれば記載してください。

記載内容:

- 判断した内容
- 判断理由
- 「推測」であるかどうかの明記

補足事項がない場合は「なし」と記載してください。

---

# 禁止事項

以下は禁止してください。

- ②で決定した設計方針（設計パターン・Bounded Context・Aggregate等）を変更する
- `規約/`ディレクトリ配下の規約ドキュメント（アーキテクチャ規約.md・コーディング規約.md・Gorm規約.md）に反する実装方針を記載する
- ①Railsの実装をそのまま実装仕様として転記する
- 完全に動作する関数本体のロジックを記述する
- SQL文をそのまま記述する
- ②に根拠のない新しい業務ルールを追加する
- Transaction Script／Active Record採用機能に対して、Domain Model相当のusecase層・Repository Interface・依存性逆転を不要に追加する（過剰設計）

---

# 完成条件

生成された実装仕様書だけを読むことで、

- どのpackage・ディレクトリに何を作成するか
- 各struct／interfaceがどのようなフィールド・メソッドを持つか
- UseCaseがどの順序で何を呼び出すか
- APIのEndpoint仕様とErrorレスポンス
- Transaction・Validation・Authorization・Errorの実装方針
- どのようなテストケースを用意すべきか

をGoエンジニアが理解し、実装に着手できる状態にしてください。

本資料は、

「設計思想を説明する資料（②）」

ではなく、

「実装者がそのままコーディングに移せる詳細仕様書」

として作成してください。

②に記載のない詳細を補った場合は、必ずその旨と判断理由を記載してください。
