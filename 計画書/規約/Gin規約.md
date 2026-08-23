# Gin規約

## 0. 本プロジェクトでの採用方針

以下は、Gin標準の規約（1章以降）を前提として、本プロジェクトで実際に採用する仕様を確定したものである。

### ルーティング構成

ルートグループはBounded Context単位（アーキテクチャ規約「5. Bounded Context構成」）で切る。1つの`routes.go`は1つのContextのルートのみを登録する。

### Bindingの方針

入力バインドには`ShouldBind`系（`ShouldBindJSON`等）のみを使用し、`Bind`系（`BindJSON`等の自動400応答）は使用しない。エラーハンドリングをアプリケーション側で一元的に制御するため（詳細は「5. エラーハンドリングミドルウェア」）。

### エラーハンドリングの方針

Handlerは、UseCase（またはTransaction Script/Active Recordの場合は該当関数・Store）から返されたエラーを`c.Error(err)`でgin.Contextへ登録するだけに留める。実際のエラー種別判定・ログ出力・レスポンス整形は、集中エラーハンドリングミドルウェア1箇所で行う。これは、アーキテクチャ規約「13. Error変換パターン（AppError）」の「Presentation層で`errors.As`を1回だけ呼び出す」という方針を、Ginのミドルウェア機構で具体的に実現したものである。

### ミドルウェアの配置

アーキテクチャ規約「8. 横断的関心事の置き場所」に従い、以下をGinのMiddlewareとして実装する。

|関心事|Middleware|登録範囲|
|-|-|-|
|認証（本人確認）|`AuthMiddleware`|認証が必要なルートグループにのみ`Use`|
|認可（役割チェック）|`RequireRole(role)`|ロールごとに異なるルートグループに`Use`|
|エラーハンドリング|`ErrorHandler`|ルーター全体に最初に`Use`|
|アクセスログ|`AccessLogger`|ルーター全体に`Use`|

---

## 1. ルーティング

パスパラメータは`:name`（1セグメント）または`*action`（それ以降すべて、先頭スラッシュを含む）で定義し、`c.Param("name")`で取得する。

```go
// Good
router.GET("/tasks/:id", func(c *gin.Context) {
    id := c.Param("id")
    // ...
})
```

クエリパラメータは`c.Query("key")`（未指定時は空文字列）または`c.DefaultQuery("key", "default")`（未指定時のデフォルト値）で取得する。`c.Query`はURLクエリ文字列のみを読み、`c.PostForm`はリクエストボディのみを読む。両者は混同しない。

```go
// Good
page := c.DefaultQuery("page", "1")
```

パス・クエリの単純な値取得を超えて、複数フィールドをまとめて扱う場合は「4. Model BindingとValidation」の`ShouldBindQuery`を使用する。

---

## 2. ルートグルーピング

`router.Group()`でURLプレフィックスとミドルウェアを共有するルート集合を作る。グループはネストでき、内側のグループは外側のプレフィックス・ミドルウェアを継承する。

```go
// Good
api := router.Group("/api")
{
    v1 := api.Group("/v1")
    {
        tasks := v1.Group("/tasks")
        tasks.Use(AuthMiddleware())
        {
            tasks.GET("", handler.ListTasks)
            tasks.POST("", handler.CreateTask)
        }
    }
}
```

Context単位でルート登録関数を分割し、各Contextの`presentation/routes.go`が自分のグループのみを構築する（アーキテクチャ規約「2. ディレクトリ構成」）。

```go
// Good
// internal/task/presentation/routes.go
func RegisterRoutes(rg *gin.RouterGroup, h *Handler) {
    tasks := rg.Group("/tasks")
    tasks.GET("", h.ListTasks)
    tasks.POST("", h.CreateTask)
}
```

```go
// Bad
// main.go にすべてのContextのルートを直接書き並べる
router.GET("/tasks", taskHandler.List)
router.GET("/goals", goalHandler.List)
router.GET("/announcements", announcementHandler.List)
// ...Contextをまたいだ大量のルートが1ファイルに集中する
```

---

## 3. ミドルウェア

ミドルウェアは`gin.HandlerFunc`を返す関数として定義する。`c.Next()`の前が「リクエスト処理前」、後が「レスポンス後処理」にあたる。

```go
// Good
func AccessLogger(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        c.Next() // ハンドラを実行

        logger.InfoContext(c.Request.Context(), "request",
            "method", c.Request.Method,
            "path", c.Request.URL.Path,
            "status", c.Writer.Status(),
            "duration_ms", time.Since(start).Milliseconds(),
        )
    }
}
```

認証・認可のように処理を中断する必要がある場合は`c.Next()`を呼ばず`c.Abort()`（または`c.AbortWithStatus()`）を使用する。

```go
// Good
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatus(http.StatusUnauthorized)
            return
        }

        c.Next()
    }
}
```

グローバルに適用するミドルウェア（エラーハンドリング・アクセスログ等）は`router.Use()`で登録し、特定のルートグループにのみ必要なミドルウェア（認証・認可）はそのグループの`Use()`で登録する。すべてのミドルウェアをグローバルに登録しない。

---

## 4. Model BindingとValidation

リクエストのバインドには`ShouldBind`系メソッド（`ShouldBindJSON` / `ShouldBindQuery` / `ShouldBindUri`等）を使用する。バインドに失敗した場合の応答はアプリケーション側（エラーハンドリングミドルウェア）で制御する。

```go
// Good
func (h *Handler) CreateTask(c *gin.Context) {
    var req CreateTaskRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.Error(toValidationError(err))
        return
    }

    // ...
}
```

```go
// Bad
func (h *Handler) CreateTask(c *gin.Context) {
    var req CreateTaskRequest
    c.BindJSON(&req) // 失敗時にGinが直接400を返し、エラーハンドリングを一元化できない
    // ...
}
```

検証には`go-playground/validator`ベースの`binding`タグを使用する。型・必須・フォーマットの検証に留め、業務ルールの検証は行わない（コーディング規約「8. 関数・メソッド設計」の「入力検証とドメイン検証の責務分離」に従う）。

```go
// Good
type CreateTaskRequest struct {
    Name    string    `json:"name" binding:"required"`
    DueDate time.Time `json:"due_date" binding:"required"`
}
```

バインドエラーは、アーキテクチャ規約「13. Error変換パターン（AppError）」の「複数フィールドのバリデーションエラー」で定義した`apperror.ValidationError`へ変換する。`validator.ValidationErrors`（フィールド単位のエラー）と、それ以外（JSON構文エラー等）を区別する。

```go
// Good
func toValidationError(err error) *apperror.ValidationError {
    var ve validator.ValidationErrors
    if errors.As(err, &ve) {
        fields := make([]apperror.FieldError, 0, len(ve))
        for _, fe := range ve {
            fields = append(fields, apperror.FieldError{
                Field:   fe.Field(),
                Message: fe.Tag(), // 必要に応じてタグごとの日本語メッセージへ変換する
            })
        }

        return &apperror.ValidationError{Fields: fields}
    }

    // フィールド単位に分解できないエラー（JSON構文エラー等）
    return &apperror.ValidationError{Fields: []apperror.FieldError{{Message: err.Error()}}}
}
```

---

## 5. エラーハンドリングミドルウェア

Handlerは、業務処理の呼び出しで得たエラーを`c.Error(err)`でgin.Contextへ登録し、以降の処理をエラーハンドリングミドルウェアに委ねる。

```go
// Good
func (h *Handler) CreateTask(c *gin.Context) {
    var req CreateTaskRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.Error(toValidationError(err))
        return
    }

    task, err := h.usecase.Execute(c.Request.Context(), req.ToCommand())
    if err != nil {
        c.Error(err) // ログ出力・レスポンス整形はErrorHandlerミドルウェアが行う
        return
    }

    c.JSON(http.StatusCreated, toResponse(task))
}
```

エラーハンドリングミドルウェアは`c.Next()`の後で`c.Errors`を確認し、アーキテクチャ規約「13. Error変換パターン（AppError）」の`AppError`インターフェースを`errors.As`で判定して、ログ出力とレスポンスを1箇所にまとめる。`AppError`が同時に`FieldErrorer`（同13章「複数フィールドのバリデーションエラー」）を満たす場合は、レスポンスに`fields`を含める。ルーターの先頭（他のどのミドルウェアよりも先）に登録する。

```go
// Good
type ErrorResponse struct {
    Message string                `json:"message"`
    Fields  []apperror.FieldError `json:"fields,omitempty"`
}

func ErrorHandler(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        if len(c.Errors) == 0 {
            return
        }

        err := c.Errors.Last().Err
        ctx := c.Request.Context()

        var appErr apperror.AppError
        if errors.As(err, &appErr) {
            resp := ErrorResponse{Message: appErr.Error()}

            var fieldErr apperror.FieldErrorer
            if errors.As(err, &fieldErr) {
                resp.Fields = fieldErr.Fields()
            }

            logger.Log(ctx, appErr.LogLevel(), "request failed", "error", appErr)
            c.JSON(appErr.StatusCode(), resp)
            return
        }

        logger.ErrorContext(ctx, "unexpected error", "error", err)
        c.JSON(http.StatusInternalServerError, ErrorResponse{Message: "internal server error"})
    }
}

router.Use(ErrorHandler(logger)) // 最初に登録する
```

```go
// Bad
func (h *Handler) CreateTask(c *gin.Context) {
    task, err := h.usecase.Execute(c.Request.Context(), cmd)
    if err != nil {
        // Handlerごとにログ出力とステータスコード判定を書く（多重ログ・判定ロジックの重複）
        log.Printf("failed to create task: %v", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, toResponse(task))
}
```

---

## 6. テスト

HTTPハンドラのテストには`net/http/httptest`を使用する。実際にサーバーを起動せず、`router.ServeHTTP(w, req)`でルーターへ直接リクエストを投げる。

```go
// Good
func TestCreateTask(t *testing.T) {
    gin.SetMode(gin.TestMode)

    router := setupRouter()

    body := strings.NewReader(`{"name":"test"}`)
    req := httptest.NewRequest(http.MethodPost, "/api/v1/tasks", body)
    req.Header.Set("Content-Type", "application/json")

    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusCreated {
        t.Errorf("status = %d, want %d", w.Code, http.StatusCreated)
    }
}
```

`gin.SetMode(gin.TestMode)`でデバッグ出力を抑制する。ミドルウェア単体のテストは、対象のミドルウェアと簡易なハンドラのみを持つ最小構成のルーターで行い、他のミドルウェアの影響を受けないようにする。複数ケースを検証する場合は、コーディング規約「14. テスト」のテーブル駆動テストに従う。
