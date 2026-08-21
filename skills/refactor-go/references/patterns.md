# refactor-go: Patterns reference

Specific, well-known designs to reach for when a package's shape calls for one. These are refinements layered on top of the Principles in SKILL.md, not a replacement for them — a pattern only gets applied where it actually fits the package's conceptual model.

Read this file only once a pattern's trigger condition (listed in SKILL.md) actually fires for the target package. Each entry below: **When to use**, then an **Example**. Where an example's identifiers are exported, the example itself follows Principle 11 (doc comments) so it models the exact standard being taught, not just generic Go.

## Functional Options

**When to use:** a struct is highly configurable — roughly 4+ independent *optional* settings (e.g. an HTTP server: host, port, timeout, max connections). Keeps the constructor's API clean, readable, and extensible without a telescoping list of positional/boolean params. A builder pattern, but built from a private options struct and closures instead of chained setter calls. Scope it to optional configuration only — required dependencies (a DB client, an API client, another business object) go through the config struct from Principle 13, not through a `With...` option.

```go
package server

// Server is an HTTP server with configurable host, port, timeout, and max connections.
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

// serverOptions holds the values Options mutate before New builds a Server.
type serverOptions struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

// Option configures a Server at construction time.
type Option func(*serverOptions)

// New creates a Server, applying any given Options over the defaults.
func New(options ...Option) *Server {
    opts := serverOptions{
        // defaults go here
    }
    for _, o := range options {
        o(&opts)
    }

    return &Server{
        host:    opts.host,
        port:    opts.port,
        timeout: opts.timeout,
        maxConn: opts.maxConn,
    }
}

// WithHost sets the Server's listen host.
func WithHost(host string) Option {
    return func(o *serverOptions) { o.host = host }
}

// WithPort sets the Server's listen port.
func WithPort(port int) Option {
    return func(o *serverOptions) { o.port = port }
}

// WithTimeout sets the Server's request timeout.
func WithTimeout(timeout time.Duration) Option {
    return func(o *serverOptions) { o.timeout = timeout }
}

// WithMaxConn sets the Server's maximum concurrent connections.
func WithMaxConn(maxConn int) Option {
    return func(o *serverOptions) { o.maxConn = maxConn }
}
```

## Worker Pool

**When to use:** concurrent execution over a stream of jobs where resource usage needs a cap — a fixed pool of goroutines pulls from a shared jobs channel instead of spawning one goroutine per job (avoids goroutine explosion).

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    for w := 1; w <= 5; w++ {
        go worker(w, jobs, results)
    }

    for j := 1; j <= 10; j++ {
        jobs <- j
    }
    close(jobs)

    for a := 1; a <= 10; a++ {
        fmt.Println(<-results)
    }
}
```

## Pipeline

**When to use:** processing needs multiple sequential stages (e.g. transform → filter → enrich → persist). Each stage is a function taking an input channel and returning an output channel, so stages stay modular and independently testable.

```go
func stage1(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for v := range in {
            out <- v * 2
        }
        close(out)
    }()
    return out
}

func stage2(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for v := range in {
            out <- v * v
        }
        close(out)
    }()
    return out
}

func pipeline() {
    in := make(chan int)
    s1 := stage1(in)
    s2 := stage2(s1)
    // ...
}
```

## Strategy

**When to use:** many if/switch branches picking behavior, each branch growing separately, or new behavior needs to plug in without touching the caller. Swaps an algorithm at runtime behind a common interface.

```go
// Strategy computes a result from two operands using a swappable algorithm.
type Strategy interface {
    // Execute applies the strategy's algorithm to a and b.
    Execute(a, b int) int
}

// Add is a Strategy that sums its operands.
type Add struct{}

// Execute returns a + b. Conforms to the Strategy interface.
func (Add) Execute(a, b int) int { return a + b }

// Multiply is a Strategy that multiplies its operands.
type Multiply struct{}

// Execute returns a * b. Conforms to the Strategy interface.
func (Multiply) Execute(a, b int) int { return a * b }

func main() {
    var strategy Strategy = Add{}
    fmt.Println("Add:", strategy.Execute(2, 3))

    strategy = Multiply{}
    fmt.Println("Multiply:", strategy.Execute(2, 3))
}
```

## Command

**When to use:** undo/redo is needed, actions must be queued/scheduled or logged for replay, or the invoker needs to be decoupled from the receiver (e.g. a menu button that doesn't know what it triggers). Wraps a request/action as an object with `Execute`/`Undo`.

```go
// Command is a reversible action that can be run and later undone.
type Command interface {
    // Execute runs the action.
    Execute()
    // Undo reverses the action. Only valid after Execute has run.
    Undo()
}

// AddItemCommand is a Command that adds an Item to a Cart, and can remove it again.
type AddItemCommand struct {
    cart *Cart // cart is the Cart the item is added to.
    item Item  // item is the Item to add.
}

// Execute adds item to cart. Conforms to the Command interface.
func (c AddItemCommand) Execute() { c.cart.Add(c.item) }

// Undo removes item from cart. Conforms to the Command interface.
func (c AddItemCommand) Undo() { c.cart.Remove(c.item) }

// Invoker runs Commands and keeps a history so the most recent one can be undone.
type Invoker struct {
    history []Command // history holds executed Commands, most recent last.
}

// Run executes cmd and records it in the history for later undo.
func (i *Invoker) Run(cmd Command) {
    cmd.Execute()
    i.history = append(i.history, cmd)
}

// UndoLast undoes the most recently run Command, if any. No-op if history is empty.
func (i *Invoker) UndoLast() {
    if len(i.history) == 0 {
        return
    }
    last := i.history[len(i.history)-1]
    i.history = i.history[:len(i.history)-1]
    last.Undo()
}
```

## Fan-out/Fan-in

**When to use:** work needs to be distributed across multiple goroutines to process concurrently (fan-out), then their results merged back into a single channel (fan-in).

```go
func producer(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i
        time.Sleep(time.Millisecond * 100)
    }
    close(ch)
}

func worker(id int, ch <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range ch {
        fmt.Printf("Worker %d processing %d\n", id, job)
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 5)
    results := make(chan int, 5)

    // Fan-out: start workers
    var wg sync.WaitGroup
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Fan-in: collect results
    go producer(jobs)
    go func() {
        wg.Wait()
        close(results)
    }()

    for res := range results {
        fmt.Println("Result:", res)
    }
}
```

## Repository

**When to use:** abstracting a database (or similar external store) layer behind a focused interface. This is a concrete instance of Principle 4 (purpose-built clients) — apply it whenever the conceptual model is specifically "operations between a service and a data store."

```go
// User is a person record stored by a UserRepository.
type User struct {
    ID   int    // ID uniquely identifies the user.
    Name string // Name is the user's display name.
}

// UserRepository is the set of operations for reading users from storage.
type UserRepository interface {
    // GetByID returns the User with the given id, or an error if none exists.
    GetByID(id int) (*User, error)
}

// userRepo is the concrete UserRepository backed by the database.
type userRepo struct{}

// GetByID returns the User with the given id. Conforms to the UserRepository interface.
func (u userRepo) GetByID(id int) (*User, error) {
    // DB logic here (e.g. SELECT query)
    return &User{ID: id, Name: "Alice"}, nil
}

// NewUserRepository creates a UserRepository backed by the database.
func NewUserRepository() UserRepository {
    return userRepo{}
}
```
