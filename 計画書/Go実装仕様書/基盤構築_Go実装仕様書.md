# 基盤構築 Go実装仕様書

---

## 0. 本書の位置づけ

- 本書は機能(Bounded Context)の実装仕様書ではない。**Rails→Go移行作業に着手する初日に、最初に1回だけ実行する**、全機能共通の基盤構築の手順書である。どの機能から着手するかを問わず、本書から始める。
- `Go実装仕様書_自動生成プロンプト.md`の17節フォーマットの対象外(対応する①②が存在しないため)。
- **本書だけを読めば実行できるように書いてある。** 他の文書を読む必要があるのは、各Stepで明示した規約のみである。
- 本書の完了後、`計画書/Go実装仕様書/`の各機能ファイルを`.claude/commands/impl/`の①spec-check以降で1機能ずつ進める。
- バージョン等の事実は2026-11-23に公式ソース(Goモジュールプロキシ・go.dev・GitHub・Docker Hub・golangci-lint公式リファレンス)で確認したものである。規約に定めのない事項は、本書が決定として書いている。

## 1. 実行対象・前提

- 実装先: 既存の`edu-platform`リポジトリ(`https://github.com/yuta20253/edu-platform`、ローカルは`../edu-platform`)。`rails/`・`frontend/`と並列に`go/`ディレクトリを作る。新規リポジトリは作らない。
- 変更してよいのは、`go/`配下、`.github/workflows/go-ci.yml`(新規)、ルート`docker-compose.yml`への`go`サービス・`go_mod_cache`ボリュームの追記のみ。`rails/`・`frontend/`配下、既存のCIワークフロー、既存の`docker-compose.yml`の記述は変更しない。
- 読み込む規約: `計画書/規約/アーキテクチャ規約.md`・`コーディング規約.md`・`Gorm規約.md`・`Gin規約.md`
- Go 1.26を使う。主要ライブラリは Gin v1.12系・GORM v1.3x系・go-sql-driver/mysql v1.10系。DBはRailsの既存MySQL(8.0.32、composeの`db`サービス)を使い続け、Go側にマイグレーションは持たない。
- コーディング作業(Step 7・8)は、メインエージェントが直接書かず、Agentツール(サブエージェント)に委譲する(`.claude/commands/impl/code.md`「運用ルール」と同じ方針)。

## 2. 実行ステップ

### Step 1: 作業ブランチ作成

本書は`/impl:plan`を経由しないため、`.claude/commands/impl/plan.md`「作業内容」1-2と同じ手順をここで行う。

1. `edu-platform`の現在のブランチと未コミットの変更を確認する(`git -C ../edu-platform branch --show-current`・`git -C ../edu-platform status`)。他の作業中ブランチ・変更がある場合は、それに触れずユーザーに確認する。
2. `main`から、Go基盤構築用の作業ブランチを作成する。ブランチ名はユーザーに確認する。
3. 以降のStepはすべて、このブランチ上で`edu-platform`内に対して行う。

### Step 2: `go/go.mod`作成

`go/`で次を実行する。`module`パスは`github.com/yuta20253/edu-platform/go`とする(リポジトリ内のサブディレクトリをモジュールにするため、リポジトリパスに`/go`を付ける)。

```bash
go mod init github.com/yuta20253/edu-platform/go
go get github.com/gin-gonic/gin@v1.12.0 gorm.io/gorm@v1.31.2 gorm.io/driver/mysql@v1.6.0 github.com/go-sql-driver/mysql@v1.10.2
```

`go.mod`の`go`ディレクティブは`go 1.26`にする(`go mod init`が書く値が異なる場合は書き換える)。`go.sum`の生成と依存の整理は、Step 7でコードが入った後に`go mod tidy`で行う。

- `gorm.io/driver/mysql`は、GORMからMySQLへ接続するために必要なドライバ。`github.com/go-sql-driver/mysql`は`コーディング規約.md`のDB接続の例で直接importされる。
- 版は2026-11-23時点の各最新(Goモジュールプロキシで確認)。GORMのMySQLドライバの最新は`v1.6.0`(2025-06公開)。

### Step 3: ディレクトリ骨格作成

次の構成を作る(`アーキテクチャ規約.md`「4. Bounded Context構成」の「ディレクトリ構成」の機能に依存しない部分)。`go/internal/{context}/`は対象外(各機能③の実装着手時に作る)。

```text
edu-platform/
├── .github/workflows/
│   └── go-ci.yml                 新規(Step 5)
├── docker-compose.yml            既存。goサービスを追記(Step 4)
└── go/
    ├── go.mod
    ├── go.sum
    ├── .golangci.yml             Step 6
    ├── Dockerfile                Step 4
    ├── cmd/
    │   └── server/
    │       └── main.go           Step 7
    └── internal/
        └── shared/
            ├── apperror/        AppError（アーキテクチャ規約「12. Error変換パターン（AppError）」）
            ├── transaction/     TransactionManagerインターフェース（同「11. Transaction実装パターン（TransactionManager）」）
            │   └── gormtx/      TransactionManagerのGORM実装
            ├── jobqueue/        JobPublisherインターフェース（同「13. 非同期ジョブ実行パターン（JobQueue）」）
            │   └── worker/      jobsテーブルのGORMモデル・JobPublisherの実装・ワーカー
            ├── scheduler/       定期実行の共通基盤（同「13」の「定期実行」）
            ├── pagination/      ページネーションのメタ情報（同「15. API規約」の「ページネーション」）
            ├── middleware/      アクセスログ・エラーハンドリング（本書ではこの2つのみ。下記Step 7参照）
            └── config/          設定の読み込み
```

`internal/shared/`配下のディレクトリは、Step 7で実装するまでの間は空になる。gitは空ディレクトリを追跡しないため、ディレクトリだけ先に作る場合は`.gitkeep`を置く(Step 7で実装が入るディレクトリは不要になる)。

### Step 4: `go/Dockerfile`作成、ルート`docker-compose.yml`へ`go`サービス追加

`rails/Dockerfile`(開発用、`CMD ["bash"]`で終わり、起動コマンドはcompose側の`command:`で上書きする構成)と同じ考え方の、開発用Dockerfileを作る。

```dockerfile
FROM golang:1.26-bookworm

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["bash"]
```

ベースイメージタグ`golang:1.26-bookworm`はDocker Hubに実在することを確認済み(Go 1.26系の最新パッチ1.26.11を含む)。

ルートの`docker-compose.yml`は既存の記述を変えず、`services:`配下に次の`go`サービスを追記し、`volumes:`に`go_mod_cache:`を追記する。

```yaml
  go:
    build: ./go
    command: go run ./cmd/server
    ports:
      - "8080:8080"
    volumes:
      - ./go:/app
      - go_mod_cache:/go/pkg/mod
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_PORT: 3306
      DB_USER: root
      DB_PASSWORD: password
      DB_NAME: myapp_development
      PORT: 8080
      LOG_LEVEL: debug
      APP_ENV: development
      GIN_MODE: debug
      TEST_DATABASE_DSN: root:password@tcp(db:3306)/myapp_test?charset=utf8mb4&parseTime=True&loc=Local
    tty: true
    stdin_open: true
```

- 環境変数名`DB_HOST`・`DB_PORT`・`PORT`・`LOG_LEVEL`は、`コーディング規約.md`「設定値(Config)の読み込み」の例と同じ。規約の例にない`DB_USER`・`DB_PASSWORD`・`DB_NAME`・`APP_ENV`は、本書の決定として`config`に追加する(Step 7)。値は既存`db`サービスの設定(`root`/`password`/`myapp_development`)と一致させている。`APP_ENV`は、`コーディング規約.md`「22. ロギング」の「本番は`slog.JSONHandler`、開発は`slog.TextHandler`を設定値に応じて切り替える」の判定に使う(`production`ならJSON、それ以外はText)。
- `ports`は`8080`とする(ホスト側の`5001`(Rails)・`3307`(MySQL)・`6379`(Redis)と重ならない)。
- テストは開発用DB(`myapp_development`)ではなく、同じMySQL上の別DB`myapp_test`を使う(`コーディング規約.md`「テストによるデータ破壊の禁止」が、テスト専用DBへの接続と、`TEST_DATABASE_DSN`に`test`を含めることを求めている)。DB名は本書の決定として`myapp_test`とする。
- `go run`は変更ごとに再コンパイルされるため、保存時にコンテナを再起動する運用とする(ホットリロードは行わない)。

### Step 5: `.github/workflows/go-ci.yml`作成

既存の`rails-ci.yml`・`frontend-ci.yml`と同じ形(`paths`フィルタ、`working-directory`指定)で作る。既存のワークフローは変更しない。

```yaml
name: Go CI

on:
  pull_request:
    paths:
      - 'go/**'
      - '.github/workflows/go-ci.yml'

defaults:
  run:
    working-directory: go

jobs:
  lint:
    name: golangci-lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go/go.mod
      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.15.0
          working-directory: go

  test:
    name: go test
    runs-on: ubuntu-latest
    services:
      db:
        image: mysql:8.0.32
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: myapp_test
        ports:
          - '3306:3306'
        options: >-
          --health-cmd "mysqladmin ping -h 127.0.0.1 -u root -ppassword"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 10
          --health-start-period 30s
    env:
      TEST_DATABASE_DSN: root:password@tcp(127.0.0.1:3306)/myapp_test?charset=utf8mb4&parseTime=True&loc=Local
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go/go.mod
      - run: go test -race ./...

  build:
    name: go build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go/go.mod
      - run: go build ./...
```

- `go-version-file: go/go.mod`により、CIのGoバージョンは`go.mod`の値と常に一致する(`コーディング規約.md`「Goバージョンの指定」が求める「CIのGoバージョンを`go.mod`と一致させる」への対応)。
- CIのMySQLは、開発・本番とは別インスタンスとして毎回起動し使い捨てる(`コーディング規約.md`「CIで使用するDB接続先」)。イメージは`edu-platform`の`docker-compose.yml`の`db`サービスと同じ`mysql:8.0.32`。
- 各Actionと`golangci-lint`の版は2026-11-23時点のGitHub最新リリース(`actions/checkout` v7.0.1、`actions/setup-go` v7.1.0、`golangci-lint-action` v9.3.0、`golangci-lint` v2.15.0)。`version`・`working-directory`・`go-version-file`の入力名は各Actionの`action.yml`で確認済み。`golangci-lint`は再現性のため`latest`ではなく版を固定している。
- `MYSQL_DATABASE: myapp_test`は、Step 4で決めたテスト用DB名と一致させている。
- `go test`のジョブは`go/Dockerfile`を使わず、Runner上に直接Goを入れて実行する。

### Step 6: `go/.golangci.yml`作成

`コーディング規約.md`が機械的な強制を求めている項目は次のとおり。

| 項目 | 設定 | 出典(`コーディング規約.md`) |
|-|-|-|
| `gofmt`・`goimports` | 有効 | 「フォーマット」(gofmt必須) |
| `funlen` | デフォルト値(60行) | 関数の長さ(960行目付近) |
| `lll` | `line-length: 100` | 1行の長さ(1004行目付近) |
| `govet` | `shadow`チェックを有効 | シャドーイング(1847行目付近) |
| `staticcheck` | 有効(`Deprecated:`コメントを認識する) | 非推奨の記法(3746行目付近) |

golangci-lint v2.15.0の公式リファレンス(`.golangci.reference.yml`)で確認した書式で、次のとおりとする。`default: standard`の標準セットに`staticcheck`・`govet`が含まれる。

```yaml
version: "2"

linters:
  default: standard
  enable:
    - funlen
    - lll
    - staticcheck
  settings:
    lll:
      line-length: 100
    govet:
      enable:
        - shadow

formatters:
  enable:
    - gofmt
    - goimports
```

### Step 7: `go/internal/shared/`・`go/cmd/server/main.go`の実装(サブエージェントに委譲)

メインエージェントは実装コードを直接書かず、Agentツールでサブエージェントに委譲する。

**実装順序(依存関係順。後続が常にコンパイルできる状態を保つ):**
`config` → `apperror` → `transaction` → `transaction/gormtx` → `jobqueue` → `jobqueue/worker` → `scheduler` → `pagination` → `middleware`(アクセスログ・エラーハンドリングのみ) → `cmd/server/main.go`

**実装仕様の出所:** 各パッケージは個別機能の①②に対応するものがないため、③は作らず、規約の該当節を実装仕様として直接用いる。

| パッケージ | 実装仕様 |
|-|-|
| `config` | `コーディング規約.md`「設定値(Config)の読み込み」(`Load()`、`Config`構造体)。Step 4で決めた`DB_USER`・`DB_PASSWORD`・`DB_NAME`・`APP_ENV`のフィールドを追加する |
| `apperror` | `アーキテクチャ規約.md`「12. Error変換パターン(AppError)」 |
| `transaction` / `transaction/gormtx` | 同「11. Transaction実装パターン(TransactionManager)」。インターフェースは`transaction`、GORM実装は`gormtx`に置き、`transaction`はGORMをimportしない |
| `jobqueue` / `jobqueue/worker` | 同「13. 非同期ジョブ実行パターン(JobQueue)」。インターフェースは`jobqueue`、GORMモデル・実装・ワーカーは`worker`に置く |
| `scheduler` | 同「13」の「定期実行」。GORMに依存しない |
| `pagination` | 同「15. API規約」の「ページネーション」(`Meta`型)。GORMに依存しない |
| `middleware` | **`AccessLogger`・`ErrorHandler`のみ。** 仕様は`Gin規約.md`「3. ミドルウェア」(`AccessLogger`の実装例、`ErrorHandler`の実装例と挙動、登録順は`AccessLogger`が最も外側で`ErrorHandler`がその次)、エラーレスポンスの形式は`アーキテクチャ規約.md`「15. API規約」の「エラーレスポンス」、ログ出力は`コーディング規約.md`「22. ロギング」。`AuthMiddleware`・`RequireRole`・`CSRFMiddleware`は、`currentuser`(User Context)・JWT・Cookie仕様に依存し、認証機能③・ユーザー基盤機能③が前提となるため、**本書では実装しない**(各機能③の実装時に行う) |
| `cmd/server/main.go` | 下記の最小版 |

**`cmd/server/main.go`の最小版**(Contextが0件の状態。`アーキテクチャ規約.md`「14. 依存関係の組み立て(DI配線)」の起動イメージから、Context登録を除いたもの。エンドポイントは1つも登録しない):

1. `config.Load()`で設定を読み込む
2. ロガーを作る。`APP_ENV`が`production`なら`slog.JSONHandler`、それ以外は`slog.TextHandler`(`コーディング規約.md`「22. ロギング」)。ログレベルは`LOG_LEVEL`
3. GORMでMySQLに接続し、起動時に`Ping`して失敗したら終了する(`config`のDB設定を使う)
4. `gin.New()`で`router`を作り、`middleware.AccessLogger(logger)`・`middleware.ErrorHandler(logger)`をこの順に`Use`する。CORSミドルウェアは登録しない(`アーキテクチャ規約.md`「15. API規約」の「CORS」)
5. `router.Run(":" + cfg.Port)`で起動する。`/api/v1`のルートグループ・各Contextの登録・グレースフルシャットダウン・`scheduler`の起動は、Contextが増える段階で追加する

業務用のエンドポイントもヘルスチェック用のエンドポイントも作らない。起動確認はStep 8で、起動時の`Ping`成功と未登録パスへの404応答で行う。

**テスト方針:**

- `コーディング規約.md`のテスト規約に従う(テーブル駆動テスト、goroutine/channelを使う`jobqueue/worker`・`scheduler`は`-race`必須、など)
- テスト用DBへのスキーマ作成は、Railsのスキーマダンプを持たず**GORMの`AutoMigrate`を使う**。`Gorm規約.md`「7. マイグレーション」が「`AutoMigrate`はローカル環境でのテスト用DBセットアップ等、開発補助用途に限定して使用してよい」としているため。`setupTestDB(t)`(`コーディング規約.md`「テストによるデータ破壊の禁止」の例。`TEST_DATABASE_DSN`に`test`が含まれることを確認してから接続する)の中で、そのテストが使うGORMモデルに対し`db.AutoMigrate(...)`を呼ぶ。`AutoMigrate`は冪等なので繰り返し呼んでよい。呼ぶ粒度(テストごと／`TestMain`)は実装時に決める
- 本番・開発のスキーマはRailsのマイグレーションが管理する。`AutoMigrate`は本番・開発では使わない
- 進め方はテスト→実装の順(TDD)。`/impl:plan`の「TDDステップ一覧」と同じ考え方で、1ファイルずつ、テストを書いて失敗を確認(Red)→実装して成功を確認(Green)を繰り返す

**メインエージェントによる完了確認:** サブエージェントの報告を鵜呑みにせず、`/impl:code-check`相当の確認(ビルド・静的解析・全テスト通過、スタブ・TODO・未実装が残っていないか)を自分で行う。

### Step 8: 動作確認

`go/`ディレクトリで実行する。

1. ルートで`docker-compose up -d db`によりDBを起動する。
2. テスト用DBを作る(composeの`db`は`myapp_development`しか作らないため): `docker-compose exec db mysql -uroot -ppassword -e "CREATE DATABASE IF NOT EXISTS myapp_test"`
3. `go build ./...`
4. `go vet ./...`
5. `golangci-lint run`
6. `go test -race ./...`(ホストから実行する場合、`TEST_DATABASE_DSN`は`localhost:3307`向けに`root:password@tcp(127.0.0.1:3307)/myapp_test?charset=utf8mb4&parseTime=True&loc=Local`を設定する。composeの`go`コンテナ内で実行する場合はcomposeの設定値を使う)
7. ルートで`docker-compose up go`を実行し、`go`サービスが終了せずに起動し続けること(起動時のDB `Ping`が成功している)、`curl -i http://localhost:8080/`が404を返すことを確認する。
8. pull requestを作成し(作成は**ユーザーの承認を得てから**行う)、`go-ci.yml`の`lint`・`test`・`build`の3ジョブが通ることを確認する。

## 3. 完了条件

- Step 1〜8がすべて完了し、Step 8の確認項目がすべて通る
- スタブ・TODO・未実装が残っていない(認証系ミドルウェアを実装しないことは、未実装ではなく範囲外である)
- `計画書/規約/`配下の規約に反する実装・設定がない
- 本書にない方針を無断で新設していない。本書に書かれていない判断が必要になった場合は、自分で決めずにユーザーに確認する

## 4. 禁止事項

- 本書にない方針を無断で新設する
- `go/internal/shared/`・`main.go`の実装をメインエージェントが直接書く(サブエージェント委譲を経ない)
- 規約に反する実装・設定をする
- `rails/`・`frontend/`配下、既存のCIワークフロー、既存の`docker-compose.yml`の記述を変更する
- ユーザーの承認なしにpull requestを作成する・pushする
- Step 8の確認を行わずに完了とする
