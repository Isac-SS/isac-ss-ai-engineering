---
titulo: "Framework Reference — Gin (Go)"
fonte_primaria: "Go Documentation (go.dev/doc) + Gin Documentation (gin-gonic.com)"
fontes_complementares:
  - "Effective Go"
  - "Let's Go / Let's Go Further (Alex Edwards)"
  - "100 Go Mistakes (Teiva Harsanyi)"
versao_framework: "Gin 1.x / Go 1.22+"
---

# Gin (Go) — Reference para Arquiteto Bob

## Deteccao

O projeto usa Gin quando:

- `go.mod` contém `require github.com/gin-gonic/gin v1.x.x`
- `main.go` (ou equivalente em `cmd/`) importa `"github.com/gin-gonic/gin"` e chama `gin.Default()` ou `gin.New()`
- Handlers usam `*gin.Context` como parametro
- Rotas sao definidas com `router.GET()`, `router.POST()`, etc.

Comando rapido para confirmar:

```bash
grep -r "gin-gonic/gin" go.mod
grep -rn "gin.Default\|gin.New" cmd/ main.go internal/ 2>/dev/null
```

---

## Filosofia e Principios Core

Go foi projetado com principios claros que impactam diretamente decisoes arquiteturais. Todas as citacoes abaixo vem de **Effective Go** (go.dev/doc/effective_go) e do **Go Blog**.

### Simplicidade e clareza

Go prioriza codigo explicito sobre magico. Nao ha heranca, generics excessivos ou metaprogramacao pesada. O codigo deve ser legivel sem documentacao externa.

### Composicao sobre heranca

Go nao tem classes nem heranca. Comportamento e compartilhado via **embedding** e **interfaces implicitas**:

```go
// Fonte: Effective Go — Embedding
type Job struct {
    Command string
    *log.Logger // Campo embarcado — Job ganha todos os metodos de Logger
}

job := &Job{Command: "deploy", Logger: log.New(os.Stderr, "Job: ", log.Ldate)}
job.Println("starting now...") // Funciona via Logger embarcado
```

### Interfaces implicitas

Um tipo satisfaz uma interface simplesmente implementando seus metodos — sem declaracao explicita. Isso habilita acoplamento fraco naturalmente:

```go
// Fonte: Effective Go — Interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composicao de interfaces via embedding
type ReadWriter interface {
    Reader
    Writer
}
```

Principio citado em Effective Go: **"The bigger the interface, the weaker the abstraction."** Interfaces pequenas e focadas sao preferidas.

### Concurrency como cidada de primeira classe

Do Go Blog (go.dev/blog/pipelines): **"Do not communicate by sharing memory; instead, share memory by communicating."** Goroutines e channels sao primitivas da linguagem, nao bibliotecas externas.

---

## Estrutura Canonica

Estrutura recomendada pela documentacao oficial (go.dev/doc/modules/layout) para um servidor Go:

```
project-root/
  go.mod
  go.sum
  internal/           # pacotes internos — nao exportaveis para modulos externos
    auth/
      auth.go
      auth_test.go
    model/
      user.go
      user_test.go
    repository/
      user_repo.go
      user_repo_test.go
    service/
      user_service.go
      user_service_test.go
    handler/
      user_handler.go
      user_handler_test.go
    middleware/
      auth.go
      logging.go
    config/
      config.go
  cmd/
    api-server/       # ponto de entrada do servidor
      main.go
    migrate/          # ferramenta CLI auxiliar
      main.go
```

**Convencoes-chave (fonte: go.dev/doc/modules/layout):**

- **`internal/`** — pacotes que nao podem ser importados por modulos externos. Permite refatoracao livre sem quebrar consumidores
- **`cmd/`** — diretorios de executaveis. Cada subdiretorio contem um `main.go`
- **Servidores sao tipicamente self-contained** — mantenha logica em `internal/`, comandos em `cmd/`, e nao exporte pacotes

---

## Patterns Idiomaticos

### Middleware Chains

Gin usa o conceito de `gin.HandlerFunc` encadeadas. `gin.Default()` ja inclui Logger e Recovery. Middleware personalizado usa `c.Next()` para continuar a chain e `c.Abort()` para interrompe-la.

```go
// Fonte: Gin docs (github.com/gin-gonic/gin/docs/doc.md) — Custom Middleware
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()

        // Define valor acessivel por handlers subsequentes
        c.Set("example", "12345")

        // Processa request
        c.Next()

        // Apos o handler executar
        latency := time.Since(t)
        log.Print(latency)

        status := c.Writer.Status()
        log.Println(status)
    }
}

func main() {
    r := gin.New()
    r.Use(Logger())

    r.GET("/test", func(c *gin.Context) {
        example := c.MustGet("example").(string)
        log.Println(example)
    })

    r.Run(":8080")
}
```

Middleware de autorizacao com `c.Abort()`:

```go
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "authorization header required",
            })
            return
        }
        // Validar token e setar usuario no contexto
        c.Set("userID", "extracted-user-id")
        c.Next()
    }
}
```

Composicao de middleware em grupos (fonte: Gin docs — Using Middleware):

```go
func main() {
    r := gin.New()

    // Middleware global
    r.Use(gin.Logger())
    r.Use(gin.Recovery())

    // Grupo com middleware adicional
    authorized := r.Group("/")
    authorized.Use(AuthRequired())
    {
        authorized.POST("/login", loginEndpoint)
        authorized.POST("/submit", submitEndpoint)

        testing := authorized.Group("testing")
        testing.GET("/analytics", analyticsEndpoint)
    }

    r.Run(":8080")
}
```

### Error Handling (vs Exceptions)

Go **nao tem exceptions**. Erros sao valores retornados explicitamente. Isso e uma decisao de design fundamental documentada em go.dev/blog/error-handling-and-go.

**A interface `error`:**

```go
// Fonte: Go Blog — Error Handling and Go
type error interface {
    Error() string
}
```

**Custom error types para contexto rico:**

```go
// Fonte: Go Blog — Error Handling and Go (baseado em json.SyntaxError)
type AppError struct {
    Err     error
    Message string
    Code    int
}

func (e *AppError) Error() string {
    return fmt.Sprintf("%s: %v", e.Message, e.Err)
}

func (e *AppError) Unwrap() error {
    return e.Err
}
```

**Reconciliacao com Gin — mapeando erros para respostas HTTP:**

```go
// Pattern inspirado no Go Blog — appError com Gin
func handleGetUser(svc *UserService) gin.HandlerFunc {
    return func(c *gin.Context) {
        id := c.Param("id")

        user, err := svc.FindByID(c.Request.Context(), id)
        if err != nil {
            var appErr *AppError
            if errors.As(err, &appErr) {
                c.JSON(appErr.Code, gin.H{"error": appErr.Message})
                return
            }
            c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
            return
        }

        c.JSON(http.StatusOK, user)
    }
}
```

**Sentinel errors para casos conhecidos:**

```go
var (
    ErrNotFound     = &AppError{Message: "resource not found", Code: http.StatusNotFound}
    ErrUnauthorized = &AppError{Message: "unauthorized", Code: http.StatusUnauthorized}
)

// Uso no service
func (s *UserService) FindByID(ctx context.Context, id string) (*User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("finding user %s: %w", id, err)
    }
    return user, nil
}
```

### Dependency Injection sem Framework

Go idiomatico usa **interfaces + constructor injection manual**. Nao ha necessidade de containers DI na maioria dos casos. Fonte: Effective Go (interfaces implicitas habilitam DI naturalmente).

```go
// 1. Defina a interface no pacote que CONSOME (nao no que implementa)
// Fonte: Effective Go — "the bigger the interface, the weaker the abstraction"
type UserRepository interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
}

// 2. Service recebe a interface via constructor
type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) FindByID(ctx context.Context, id string) (*User, error) {
    return s.repo.GetByID(ctx, id)
}

// 3. Implementacao concreta em outro pacote
type PostgresUserRepo struct {
    db *sql.DB
}

func NewPostgresUserRepo(db *sql.DB) *PostgresUserRepo {
    return &PostgresUserRepo{db: db}
}

func (r *PostgresUserRepo) GetByID(ctx context.Context, id string) (*User, error) {
    var user User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *PostgresUserRepo) Create(ctx context.Context, user *User) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
        user.ID, user.Name, user.Email,
    )
    return err
}

// 4. Composicao no main.go
func main() {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    repo := NewPostgresUserRepo(db)
    svc := NewUserService(repo)

    router := gin.Default()
    router.GET("/users/:id", handleGetUser(svc))
    router.Run(":8080")
}
```

### Context Propagation

`context.Context` e o mecanismo padrao de Go para propagacao de timeouts, cancellation e valores request-scoped. Fonte: go.dev/blog/context.

**Convencao fundamental (citada no Go Blog):** Context e passado como primeiro parametro de toda funcao no call path entre requests de entrada e saida.

```go
// Fonte: Go Blog — Go Concurrency Patterns: Context
// Gin fornece o context HTTP via c.Request.Context()

func handleSearch(svc *SearchService) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Gin ja carrega o context do request HTTP
        ctx := c.Request.Context()

        query := c.Query("q")
        if query == "" {
            c.JSON(http.StatusBadRequest, gin.H{"error": "query required"})
            return
        }

        // Propagar context para camadas inferiores
        results, err := svc.Search(ctx, query)
        if err != nil {
            if errors.Is(err, context.DeadlineExceeded) {
                c.JSON(http.StatusGatewayTimeout, gin.H{"error": "search timed out"})
                return
            }
            c.JSON(http.StatusInternalServerError, gin.H{"error": "search failed"})
            return
        }

        c.JSON(http.StatusOK, results)
    }
}

// Service com timeout explicito
func (s *SearchService) Search(ctx context.Context, query string) ([]Result, error) {
    // Criar sub-context com timeout (fonte: Go Blog — WithTimeout)
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel() // Sempre chamar cancel para liberar recursos

    req, err := http.NewRequestWithContext(ctx, "GET", s.endpoint, nil)
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }

    q := req.URL.Query()
    q.Set("q", query)
    req.URL.RawQuery = q.Encode()

    resp, err := s.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("executing search: %w", err)
    }
    defer resp.Body.Close()

    var results []Result
    if err := json.NewDecoder(resp.Body).Decode(&results); err != nil {
        return nil, fmt.Errorf("decoding response: %w", err)
    }
    return results, nil
}
```

**Context keys type-safe (fonte: Go Blog — Context):**

```go
// Tipo nao-exportado evita colisoes entre pacotes
type contextKey int

const userIDKey contextKey = 0

func WithUserID(ctx context.Context, userID string) context.Context {
    return context.WithValue(ctx, userIDKey, userID)
}

func UserIDFromContext(ctx context.Context) (string, bool) {
    userID, ok := ctx.Value(userIDKey).(string)
    return userID, ok
}
```

### Repository Pattern em Go

Interface definida no pacote consumidor + implementacao concreta separada. Fonte: Effective Go (interfaces implicitas).

```go
// internal/model/user.go
type User struct {
    ID    string `json:"id" db:"id"`
    Name  string `json:"name" db:"name"`
    Email string `json:"email" db:"email"`
}

// internal/repository/user.go — interface no pacote consumidor
type UserRepository interface {
    GetByID(ctx context.Context, id string) (*model.User, error)
    List(ctx context.Context, offset, limit int) ([]model.User, error)
    Create(ctx context.Context, user *model.User) error
    Update(ctx context.Context, user *model.User) error
    Delete(ctx context.Context, id string) error
}

// internal/repository/postgres_user.go — implementacao com database/sql
type postgresUserRepo struct {
    db *sql.DB
}

func NewPostgresUserRepo(db *sql.DB) UserRepository {
    return &postgresUserRepo{db: db}
}

func (r *postgresUserRepo) GetByID(ctx context.Context, id string) (*model.User, error) {
    var user model.User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return &user, nil
}

func (r *postgresUserRepo) List(ctx context.Context, offset, limit int) ([]model.User, error) {
    rows, err := r.db.QueryContext(ctx,
        "SELECT id, name, email FROM users ORDER BY name LIMIT $1 OFFSET $2",
        limit, offset,
    )
    if err != nil {
        return nil, fmt.Errorf("listing users: %w", err)
    }
    defer rows.Close()

    var users []model.User
    for rows.Next() {
        var u model.User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, fmt.Errorf("scanning user: %w", err)
        }
        users = append(users, u)
    }
    return users, rows.Err()
}

func (r *postgresUserRepo) Create(ctx context.Context, user *model.User) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
        user.ID, user.Name, user.Email,
    )
    if err != nil {
        return fmt.Errorf("creating user: %w", err)
    }
    return nil
}

func (r *postgresUserRepo) Update(ctx context.Context, user *model.User) error {
    _, err := r.db.ExecContext(ctx,
        "UPDATE users SET name = $1, email = $2 WHERE id = $3",
        user.Name, user.Email, user.ID,
    )
    if err != nil {
        return fmt.Errorf("updating user %s: %w", user.ID, err)
    }
    return nil
}

func (r *postgresUserRepo) Delete(ctx context.Context, id string) error {
    _, err := r.db.ExecContext(ctx,
        "DELETE FROM users WHERE id = $1", id,
    )
    if err != nil {
        return fmt.Errorf("deleting user %s: %w", id, err)
    }
    return nil
}
```

### Concurrency Patterns

Padroes diretamente do Go Blog (go.dev/blog/pipelines).

**Pipeline basico — producer/consumer com channels:**

```go
// Fonte: Go Blog — Go Concurrency Patterns: Pipelines and cancellation
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16 then 81
    }
}
```

**Fan-out/fan-in com sync.WaitGroup:**

```go
// Fonte: Go Blog — Pipelines (merge function)
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

**Cancelamento explicito com done channel:**

```go
// Fonte: Go Blog — Pipelines (Explicit cancellation)
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}

func main() {
    done := make(chan struct{})
    defer close(done) // Ao fechar, todas as goroutines terminam

    in := gen(done, 2, 3)
    c1 := sq(done, in)
    c2 := sq(done, in)

    out := merge(done, c1, c2)
    fmt.Println(<-out) // Consome apenas um resultado; done libera o resto
}
```

**Bounded parallelism — pool fixo de workers:**

```go
// Fonte: Go Blog — Pipelines (Bounded parallelism)
func processItems(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    itemCh := make(chan Item)

    // Producer
    g.Go(func() error {
        defer close(itemCh)
        for _, item := range items {
            select {
            case itemCh <- item:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    // Pool fixo de workers
    const numWorkers = 5
    for i := 0; i < numWorkers; i++ {
        g.Go(func() error {
            for item := range itemCh {
                if err := process(ctx, item); err != nil {
                    return err
                }
            }
            return nil
        })
    }

    return g.Wait()
}
```

---

## API Design com Gin

### Route Groups e versionamento

```go
// Fonte: Gin docs — Grouping Routes
func setupRouter(svc *UserService) *gin.Engine {
    router := gin.Default()

    // Health check publico
    router.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    v1 := router.Group("/api/v1")
    {
        users := v1.Group("/users")
        users.GET("", handleListUsers(svc))
        users.POST("", handleCreateUser(svc))
        users.GET("/:id", handleGetUser(svc))
        users.PUT("/:id", handleUpdateUser(svc))
        users.DELETE("/:id", handleDeleteUser(svc))
    }

    return router
}
```

### Binding e Validation com struct tags

```go
// Fonte: Gin docs — Model binding and validation
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=2,max=100"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,gte=18,lte=130"`
}

func handleCreateUser(svc *UserService) gin.HandlerFunc {
    return func(c *gin.Context) {
        var req CreateUserRequest
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        user, err := svc.Create(c.Request.Context(), &req)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to create user"})
            return
        }

        c.JSON(http.StatusCreated, user)
    }
}
```

### Custom Validator

```go
// Fonte: Gin docs — Custom Validators
type Booking struct {
    CheckIn  time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
    CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn" time_format:"2006-01-02"`
}

var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
    date, ok := fl.Field().Interface().(time.Time)
    if ok {
        today := time.Now()
        if today.After(date) {
            return false
        }
    }
    return true
}

func main() {
    route := gin.Default()

    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        v.RegisterValidation("bookabledate", bookableDate)
    }

    route.GET("/bookable", getBookable)
    route.Run(":8085")
}

func getBookable(c *gin.Context) {
    var b Booking
    if err := c.ShouldBindWith(&b, binding.Query); err == nil {
        c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
    } else {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    }
}
```

### Paginacao

```go
type PaginatedResponse struct {
    Data       interface{} `json:"data"`
    Page       int         `json:"page"`
    PerPage    int         `json:"per_page"`
    TotalItems int64       `json:"total_items"`
    TotalPages int         `json:"total_pages"`
}

type PaginationQuery struct {
    Page    int `form:"page,default=1" binding:"min=1"`
    PerPage int `form:"per_page,default=20" binding:"min=1,max=100"`
}

func handleListUsers(svc *UserService) gin.HandlerFunc {
    return func(c *gin.Context) {
        var pq PaginationQuery
        if err := c.ShouldBindQuery(&pq); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        offset := (pq.Page - 1) * pq.PerPage
        users, err := svc.List(c.Request.Context(), offset, pq.PerPage)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to list users"})
            return
        }

        total, _ := svc.Count(c.Request.Context())
        totalPages := int(total) / pq.PerPage
        if int(total)%pq.PerPage > 0 {
            totalPages++
        }

        c.JSON(http.StatusOK, PaginatedResponse{
            Data:       users,
            Page:       pq.Page,
            PerPage:    pq.PerPage,
            TotalItems: total,
            TotalPages: totalPages,
        })
    }
}
```

### Graceful Shutdown

```go
// Fonte: Gin docs — Graceful restart or stop
package main

import (
    "context"
    "errors"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()
    router.GET("/", func(c *gin.Context) {
        time.Sleep(5 * time.Second)
        c.String(http.StatusOK, "Welcome Gin Server")
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Printf("listen: %s\n", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exiting")
}
```

---

## Anti-Patterns Go/Gin

### 1. Goroutine leak — falta de cancelamento

```go
// RUIM: goroutine fica bloqueada para sempre se ninguem consome ch
func fetchData() <-chan Result {
    ch := make(chan Result)
    go func() {
        result := expensiveCall()
        ch <- result // Bloqueia eternamente se o consumidor desistiu
    }()
    return ch
}
```

```go
// BOM: usar context para cancelamento (fonte: Go Blog — Pipelines)
func fetchData(ctx context.Context) <-chan Result {
    ch := make(chan Result, 1) // Buffer de 1 evita leak se consumidor desistir
    go func() {
        defer close(ch)
        result := expensiveCall(ctx)
        select {
        case ch <- result:
        case <-ctx.Done():
            return // Goroutine termina limpa
        }
    }()
    return ch
}
```

### 2. Shared mutable state sem protecao

```go
// RUIM: race condition — multiplas goroutines acessam o mapa
var cache = make(map[string]string)

func handleRequest(c *gin.Context) {
    key := c.Query("key")
    cache[key] = "value" // DATA RACE
    c.JSON(200, gin.H{"cached": cache[key]})
}
```

```go
// BOM: proteger com sync.RWMutex
type SafeCache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}
```

### 3. Interface pollution — interfaces desnecessarias

```go
// RUIM: interface criada "por precaucao" sem segundo consumidor
// Fonte: "100 Go Mistakes" (Teiva Harsanyi) — Mistake #5
type UserServiceInterface interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context) ([]User, error)
    Count(ctx context.Context) (int64, error)
    // ... 15 metodos que so tem uma implementacao
}
```

```go
// BOM: struct concreta direta; interfaces pequenas apenas onde necessario
// Crie a interface no pacote CONSUMIDOR quando precisar de polimorfismo
type UserService struct {
    repo *PostgresUserRepo
}

// Interface apenas no teste ou onde ha multiplas implementacoes
type UserFinder interface {
    FindByID(ctx context.Context, id string) (*User, error)
}
```

### 4. Ignorar erros

```go
// RUIM: erro silenciosamente ignorado
result, _ := svc.FindByID(ctx, id) // Se falhar, result e nil → panic
c.JSON(200, result)
```

```go
// BOM: sempre tratar erros (fonte: Effective Go — Errors)
result, err := svc.FindByID(ctx, id)
if err != nil {
    c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to find user"})
    return
}
c.JSON(http.StatusOK, result)
```

### 5. Abuso de init()

```go
// RUIM: init() com efeitos colaterais dificulta testes e startup
// Fonte: "100 Go Mistakes" (Teiva Harsanyi) — Mistake #3
func init() {
    db, _ = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    db.Ping() // Pode falhar silenciosamente
}
```

```go
// BOM: inicializacao explicita em main() ou construtores
func main() {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatalf("failed to open db: %v", err)
    }
    if err := db.Ping(); err != nil {
        log.Fatalf("failed to ping db: %v", err)
    }
    defer db.Close()

    // ... setup do servidor
}
```

### 6. Usar gin.Context fora do handler lifecycle

```go
// RUIM: usar c *gin.Context em goroutine apos o handler retornar
func handleAsync(c *gin.Context) {
    go func() {
        time.Sleep(time.Second)
        c.JSON(200, gin.H{"ok": true}) // c ja foi reciclado pelo pool!
    }()
}
```

```go
// BOM: copiar o contexto antes de usar em goroutine
func handleAsync(c *gin.Context) {
    cp := c.Copy() // Copia segura para uso em goroutine
    go func() {
        time.Sleep(time.Second)
        log.Println("processing for path:", cp.Request.URL.Path)
        // Nao escrever resposta HTTP de dentro da goroutine
    }()
    c.JSON(http.StatusAccepted, gin.H{"status": "processing"})
}
```

---

## Reconciliacao com Livros de Arquitetura

| Conceito (livro) | Equivalente Go/Gin | Nota |
|---|---|---|
| Repository (Evans, DDD) | Interface + struct com `database/sql` | Interface definida no pacote consumidor; implementacao em pacote separado. Satisfacao implicita elimina boilerplate |
| Service Layer (Fowler, PoEAA) | Service struct com metodos | Injecao de dependencias via constructor. Sem annotations nem container |
| Use Case (Martin, Clean Arch) | Service method ou func dedicada | Cada metodo do service encapsula um caso de uso. Sem classes abstratas |
| Circuit Breaker (Nygard, Release It!) | `sony/gobreaker` ou implementacao manual | Go nao tem equivalente built-in. gobreaker implementa o pattern com callback. Alternativa: retry com backoff via `cenkalti/backoff` |
| Bulkhead (Nygard) | Bounded goroutine pools / semaphore | `errgroup` com `SetLimit()` ou channel como semaforo: `sem := make(chan struct{}, N)` |
| Middleware / Decorator (GoF) | `gin.HandlerFunc` chain | `c.Next()` para continuar, `c.Abort()` para interromper. Composicao natural |
| Ports & Adapters (Cockburn) | Interface = Port, struct = Adapter | `internal/` garante que adapters nao vazam para fora do modulo |
| Value Object (Evans) | Struct sem ponteiro, comparavel | Go structs sao comparaveis por valor se todos os campos forem comparaveis |
| Aggregate Root (Evans) | Struct que encapsula transacao | Transacoes via `sql.Tx` passada ao repository |
| CQRS (Young) | Interfaces separadas Read/Write | `type UserReader interface { ... }` vs `type UserWriter interface { ... }` |

---

## Checklist Arquitetural Go/Gin

- [ ] **go.mod** limpo com versao Go minima declarada e dependencias atualizadas
- [ ] **Estrutura cmd/internal/** separando entrypoints de logica
- [ ] **Interfaces pequenas** definidas no pacote consumidor (nao no implementador)
- [ ] **context.Context** propagado como primeiro parametro em toda a call chain
- [ ] **Erros sempre tratados** — nenhum `_` ignorando erro de funcoes com efeito colateral
- [ ] **Error wrapping** com `%w` para preservar a cadeia de erros via `errors.Is`/`errors.As`
- [ ] **Graceful shutdown** implementado com signal handling e `srv.Shutdown(ctx)`
- [ ] **Middleware chain** configurada: Recovery + Logger + Auth + CORS conforme necessario
- [ ] **Sem goroutine leaks** — toda goroutine tem mecanismo de cancelamento (context ou done channel)
- [ ] **Race conditions** verificadas com `go test -race ./...`
- [ ] **Timeouts configurados** no `http.Server` (ReadTimeout, WriteTimeout) e nos clients HTTP
- [ ] **Dependency injection** via constructors — sem `init()` com efeitos colaterais
- [ ] **Binding e validation** via struct tags (`binding:"required"`) nos request DTOs
- [ ] **Testes** usando interfaces para mocks — sem necessidade de framework de mocking complexo
- [ ] **`internal/`** protegendo pacotes que nao devem ser importados externamente

---

## Quando Gin Nao Basta

Gin e um framework HTTP/REST leve e performatico, mas ha cenarios onde ele nao e a ferramenta ideal:

| Sinal | Alternativa | Motivo |
|---|---|---|
| API precisa de gRPC ou streaming bidirecional | **gRPC-Go** (`google.golang.org/grpc`) ou **Connect** (`connectrpc.com/connect`) | Gin e HTTP/REST-only; gRPC usa HTTP/2 com protobuf para comunicacao eficiente entre servicos |
| Event-driven / messaging assincrono | **NATS** (`github.com/nats-io/nats.go`), **Kafka** (Sarama/confluent-kafka-go) | Gin trata request-response sincrono; eventos requerem brokers dedicados |
| GraphQL como interface primaria | **gqlgen** (`github.com/99designs/gqlgen`) | Embora Gin possa servir um endpoint GraphQL, gqlgen gera resolvers type-safe |
| DI complexa com muitos servicos | **Wire** (`github.com/google/wire`) ou **Uber fx** (`go.uber.org/fx`) | Projetos grandes (50+ servicos) se beneficiam de geracao de codigo para DI |
| Full-stack com SSR e templates complexos | **Templ** (`github.com/a-h/templ`) + htmx, ou framework full-stack | Gin suporta templates, mas nao e otimizado para aplicacoes com SSR pesado |
| Necessidade de WebSockets nativos | **Gorilla WebSocket** (`github.com/gorilla/websocket`) como complemento | Gin nao tem suporte built-in a WebSockets, mas integra facilmente com Gorilla |

**Regra pratica:** se mais de 50% dos endpoints nao sao REST/JSON, considere se Gin e realmente o melhor ponto de partida ou se deveria ser um adapter secundario.