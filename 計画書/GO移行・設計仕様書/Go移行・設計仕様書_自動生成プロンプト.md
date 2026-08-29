# Go移行・設計仕様書 自動生成プロンプト

## 目的

添付された「〇〇機能_Rails現行仕様書」をもとに、
Goへ移行するための設計方針を決定し、
「〇〇機能_Go移行・設計仕様書」を作成してください。

この資料の目的は、

> 「Railsで実現されている業務仕様を、Goではどのような設計で実現するか」

を明確にすることです。

本資料はGoコードを書くための資料ではありません。

Go実装仕様書（③）を作成するための設計判断資料として作成してください。

---

# 基本方針

以下を厳守してください。

- Railsの実装をそのままGoへ置き換えない
- Rails固有の構造（Controller・Service・Form Objectなど）に引っ張られない
- 業務仕様を中心に設計する
- Goに適した設計を検討する
- ドメインの複雑さに応じて設計パターンを選択する
- 保守性・テスト容易性・拡張性を重視する
- 実装コードやサンプルコードは出力しない

---

# Rails現行仕様書の分析方針

設計を行う前に、
Rails現行仕様書から以下を抽出してください。

- 業務目的
- 利用者
- 業務フロー
- 業務ルール
- 状態管理
- 状態遷移
- データ関係
- 権限制御
- 外部連携
- 将来的な拡張ポイント

分析ではRailsの構造ではなく、
業務仕様を中心に判断してください。

以下のような単純変換は禁止します。

例:

Rails Controller
↓
Go Handler

Rails Service
↓
Go UseCase

Rails Model
↓
Go Entity


Railsの実装構造ではなく、
業務責務を基準としてGo設計を決定してください。

---

# 前提アーキテクチャ

以下を前提としてください。

- Clean Architecture
- Bounded Context
- Gin
- GORM
- MySQL

レイヤー構成（Domain/Application/Infrastructure/Presentation）の考え方は`規約/アーキテクチャ規約.md`「2. レイヤー責務と依存方向」を前提とします。

具体的なディレクトリツリー（`internal/`配下の物理配置等）は、①Rails現行仕様書の整理が完了していないため未定です（同「1. 全体方針」の注記を参照）。本書では特定のディレクトリパスを断定的に記載しないでください。

---

# 設計パターン判断

対象機能について、
以下のいずれを採用するか判断してください。

- Transaction Script
- Active Record
- Domain Model
- Event Sourcing

必ず理由を記載してください。

判断基準:

- ドメインロジックの複雑さ
- 状態管理の有無
- 業務ルールの複雑さ
- 将来の拡張性
- テスト容易性

---

# Domain Model採用基準

DDDを目的として採用してはいけません。

以下の場合のみDomain Modelを採用してください。

- Entityが状態を持つ
- 状態遷移ルールが存在する
- 複数の業務ルールがEntityに関連する
- 将来的な機能追加が想定される
- Entityへ振る舞いを集約することで保守性が向上する

単純CRUDの場合はTransaction Scriptを優先してください。

Domain Modelを採用する場合は、
必ず採用理由を説明してください。

---

# 設計パターンと実装構造の対応

採用する設計パターンによって、実装レイヤーの構造（`規約/アーキテクチャ規約.md`「3. 設計パターンごとの構造適用方針」）が以下のように変わります。本文書の「11. Repository設計」「12. UseCase設計」「14. Transaction設計」「16. Authorization設計」「23. Railsとの責務対応」を記載する際は、この対応を踏まえてください。

- **Transaction Script採用時**: usecase層・Repository Interfaceを設けません。「UseCase」は`application/`直下の関数として、「Repository」は`infrastructure/`直下のデータアクセス関数として実装されます
- **Active Record採用時**: usecase層・Repository Interfaceを設けません。「UseCase」はHandlerがStoreを直接呼び出す処理として、「Repository」はEntity相当のstructと同一packageに置くStoreとして実装されます
- **Domain Model / Event Sourcing採用時**: UseCase struct・Repository Interfaceによる依存性逆転を、記載どおりのフルレイヤー構成で実装します

「9. Repository設計」「10. UseCase設計」は、採用パターンに関わらず業務上必要な責務・操作の単位（何のためのデータアクセスか、何を行う業務操作か）を整理するセクションです。ここで整理した内容が、実際にどのGoの構造（interface/struct/関数）として実装されるかは、採用パターンに応じて③Go実装仕様書で決まります。Transaction Script/Active Record採用時にこれらのセクションを記載する場合は、冒頭で「本機能は〇〇採用のため、UseCase層／Repository Interfaceは設けない」旨を明記した上で、業務操作・データアクセスの整理を続けてください。

---

# 出力ファイル

〇〇機能_Go移行・設計仕様書.md


---

# 出力フォーマット

# 〇〇機能 Go移行・設計仕様書

---

# 1. 機能概要

Rails現行仕様書を要約してください。

記載内容:

- 機能概要
- 利用者
- 業務上の目的

---

# 2. 設計方針

この機能をGoでどのような思想で設計するか記載してください。

例:

- 責務分離
- 保守性
- テスト容易性
- API互換性
- 拡張性

---

# 3. Bounded Context

この機能が属するBounded Contextを整理してください。

記載内容:

- Context名
- Contextの責務
- 他Contextとの依存関係
- 依存する理由

---

# 4. 設計パターン

以下を記載してください。

## 採用パターン

Transaction Script
Active Record
Domain Model
Event Sourcing


## 判断根拠

なぜそのパターンを採用するのか。

必ず以下を考慮してください。

- 業務ルール
- 状態管理
- 将来拡張
- テスト容易性

## 採用しなかったパターン

各パターンについて、
採用しなかった理由を記載してください。

---

# 5. Aggregate設計

必要な場合のみ記載してください。

記載内容:

- Aggregate Root
- Aggregateに含めるEntity
- Aggregate境界
- 整合性を保証する単位

不要な場合は理由を記載してください。

---

# 6. Entity設計

この機能で管理するEntityを整理してください。

各Entityについて:

- 役割
- ライフサイクル
- 状態変化
- 保持する責務
- 判断根拠

を記載してください。

---

# 7. Value Object設計

Value Objectとして扱うものを整理してください。

例:

- Email
- Priority
- Status
- DueDate

各項目について:

- 採用理由
- 独自ルール
- Entity属性ではなくValue Objectにする理由

を記載してください。

不要な場合は理由を記載してください。

---

# 8. Domain Service

必要なDomain Serviceを整理してください。

各Serviceについて:

- 責務
- Entityへ持たせない理由
- 判断根拠

を記載してください。

不要な場合は理由を記載してください。

---

# 9. クラス図

「5. Aggregate設計」「6. Entity設計」「7. Value Object設計」「8. Domain Service」で整理した内容を、Mermaidのクラス図（`classDiagram`）で可視化してください。コードブロックの言語指定は`mermaid`としてください。

記載内容:

- Entity・Value Object・Domain Serviceの関係（保有・参照・依存）
- Aggregate境界（Aggregate Rootとその配下）
- 主要な属性名・型（概要レベル）

Goのstruct定義（フィールドの可視性・タグ・実装詳細）は書かないでください。それは③Go実装仕様書で扱います。

Domain Model・Event Sourcing以外のパターンを採用し、整理すべきEntity関係が単純な場合は、簡略化するか省略して理由を記載してください。

---

# 10. 状態遷移図

「6. Entity設計」で状態遷移を持つと整理したEntityについて、Mermaidの状態遷移図（`stateDiagram-v2`）で状態と遷移条件を可視化してください。コードブロックの言語指定は`mermaid`としてください。

記載内容:

- 状態の一覧
- 状態遷移の条件（何をトリガーに遷移するか）
- 遷移が禁止される組み合わせ（あれば）

状態を持たない機能では省略し、理由を記載してください。

---

# 11. Repository設計

Repositoryを整理してください。

各Repositoryについて:

- 管理対象
- 責務
- 保持する検索機能
- 保持しない責務
- 判断根拠

を記載してください。

Repositoryには業務ロジックを持たせない方針としてください。

---

# 12. UseCase設計

UseCaseを整理してください。

各UseCaseについて:

- 目的
- 入力
- 出力
- トランザクション範囲
- 呼び出すRepository
- 判断根拠

を記載してください。

---

# 13. シーケンス図・処理フロー図

## シーケンス図

「12. UseCase設計」で整理した主要なUseCaseについて、呼び出し関係をMermaidのシーケンス図（`sequenceDiagram`）で可視化してください。コードブロックの言語指定は`mermaid`としてください。

記載内容:

- 登場するレイヤー（Handler / UseCase / Repository / 外部Context 等）
- 呼び出し順序
- Context間連携がある場合はその呼び出し（「3. Bounded Context」「規約/アーキテクチャ規約.md「5. Context間連携ルール」」を踏まえる）

## 処理フロー図

業務ルールに条件分岐が多い、または状態遷移を伴うUseCaseについて、判断の流れをMermaidのフローチャート（`flowchart TD`）で可視化してください。コードブロックの言語指定は`mermaid`としてください。

単純なCRUD処理で分岐がほとんどない場合は省略し、理由を記載してください。

---

# 14. Transaction設計

以下を整理してください。

- Transaction開始位置
- Transaction終了位置
- 理由

基本方針:

UseCase単位


としてください。

---

# 15. Validation設計

以下を整理してください。

## Presentation

- 型チェック
- 必須チェック
- フォーマットチェック

## Domain

- 業務ルール
- 状態チェック
- 整合性チェック

それぞれ責務を分離してください。

## バリデーション仕様

対象フィールドごとの検証ルールを表形式で整理してください。

|フィールド|検証層|ルール|エラーメッセージ方針|
|-|-|-|-|
|例：email|Presentation|必須・メール形式|「メールアドレスの形式が不正です」|
|例：due_date|Domain|未来日であること|「締切日は現在より後の日付にしてください」|

Goのバリデーションタグ（`binding:"..."`等）は書かないでください。それは③Go実装仕様書で扱います。

---

# 16. Authorization設計

以下を整理してください。

- Middleware
- Handler
- UseCase
- Domain

どこで何を判定するか。

判断理由も記載してください。

---

# 17. Error設計

整理してください。

## Domain Error

責務:

---

## Application Error

責務:

---

## Infrastructure Error

責務:

---

判断理由を記載してください。

## エラー仕様

業務上発生しうるエラーケースを表形式で整理してください。

|業務シナリオ|エラー種別|発生層|想定するHTTPステータス|
|-|-|-|-|
|例：対象が存在しない|NotFound|Domain|404|
|例：入力形式が不正|Validation|Presentation|422|
|例：権限がない|Forbidden|UseCase/Domain|403|

具体的なGoのエラー型名・実装は書かないでください。それは③Go実装仕様書（`規約/アーキテクチャ規約.md`「12. Error変換パターン（AppError）」）で扱います。

---

# 18. Domain Event

必要な場合のみ記載してください。

記載内容:

- イベント名
- 発火タイミング
- 利用目的
- 採用理由

不要な場合は理由を記載してください。

---

# 19. API仕様

この機能のAPIエンドポイント一覧と、Rails APIとの互換性を整理してください。

## エンドポイント一覧

|エンドポイント|HTTP Method|概要|
|-|-|-|
|例：/tasks|GET|タスク一覧取得|
|例：/tasks|POST|タスク作成|

## 各エンドポイントの仕様

エンドポイントごとに以下を整理してください。

- Request（パスパラメータ・クエリパラメータ・ボディの概要）
- Response（概要）
- Status Code（正常系・異常系）
- Error Response方針

## Railsとの差分

Rails仕様から変更する場合:

- Rails仕様
- Go設計での変更
- 変更理由
- 影響範囲

を記載してください。

JSONスキーマの厳密な型定義・Goの構造体は書かないでください。それは③Go実装仕様書で扱います。

---

# 20. DB設計方針

原則として既存Rails DBを継続利用する前提で設計してください。

整理項目:

- 現行DBを利用するか
- Schema変更有無
- 変更理由

Schema変更を提案する場合は必ず:

- 現状問題
- 変更内容
- 変更によるメリット
- 影響範囲

を記載してください。

---

# 21. DB操作仕様

「11. Repository設計」で整理したRepositoryが、どのテーブルに対してどのような操作を行うかを整理してください。

記載内容（Repositoryごと）:

- 対象テーブル
- 操作種別（作成・参照・更新・論理削除等）
- 主な検索条件・絞り込み条件（概要レベル）
- 関連テーブルとの結合が必要か（必要な場合、目的）
- ページネーション・ソートが必要か

具体的なSQL・GORMのクエリコードは書かないでください。それは③Go実装仕様書（`規約/Gorm規約.md`）で扱います。

---

# 22. テスト戦略

以下を整理してください。

- Domain Test
- UseCase Test
- Repository Test
- Handler Test
- Integration Test

各テストの目的も記載してください。

---

# 23. Railsとの責務対応

以下の観点で整理してください。

| Rails | Go | 設計方針 |
|--------|----|----------|

例:

|Controller|Handler|HTTP責務のみ担当|
|Service|UseCase|業務処理を担当|
|Form Object|Request DTO + Validation|入力検証を分離|

ただし単純な置換ではなく、
責務単位で判断してください。

---

# 24. 採用しなかった設計

検討した設計案について記載してください。

対象:

- 採用しなかった案
- 採用しなかった理由
- 将来的に採用する可能性

---

# 25. 設計判断サマリー

最後に主要な設計判断を一覧化してください。

形式:

|項目|採用|判断理由|
|-|-|-|
|設計パターン|Domain Model|状態管理と業務ルールが複雑なため|
|Aggregate|Task|整合性単位となるため|
|Transaction境界|UseCase単位|業務処理単位と一致するため|
|Domain Event|未採用|現時点で複数処理への通知が不要なため|

---

# 設計差分管理

Rails仕様からGo設計で変更する場合は、
必ず以下を記載してください。

- Rails現行仕様
- Go設計での変更内容
- 変更理由
- 影響範囲

例:

Rails:

Model内で権限判定

Go:

Authorization Layerで管理

理由:

認可責務を分離し、
Entityの責務を限定するため


---

# 禁止事項

以下は禁止してください。

- Goコードを出力する
- package構成を詳細設計する
- interface設計を書く
- struct設計を書く
- GORMコードを書く
- SQLを書く
- API実装を書く
- Handler実装を書く
- UseCase実装を書く

それらは③ Go実装仕様書で扱います。

---

# 完成条件

生成された設計仕様書だけを読むことで、

- なぜこの設計を採用したのか
- なぜ他の設計を採用しなかったのか
- Contextをどう分割するのか
- Entity・Repository・UseCaseをどう設計するのか
- Transaction・Validation・Authorization・Errorをどう扱うのか

を理解できる状態にしてください。

本資料は、

「Goコードを書くための資料」

ではなく、

「Goでどのように設計するかを決定する資料」

として作成してください。

すべての設計判断について、
必ず判断根拠を記載してください。

設計結果だけではなく、

- なぜ採用したのか
- なぜ他案を採用しなかったのか
- 保守性
- 拡張性
- テスト容易性

を説明してください。

推測が含まれる場合は、
「推測」と明記してください。


