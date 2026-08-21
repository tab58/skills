# Effective Go

Distilled from [Effective Go](https://go.dev/doc/effective_go) (go.dev). Written for Go's 2009 release — still good for core-language idiom, but predates generics and modules.

## Formatting

- Run `gofmt`/`go fmt` on everything; it settles indentation, spacing, and comment alignment so no one argues about style.
- Use tabs for indentation; there's no line-length limit — wrap a long line with an extra indent instead.
- Control structures (`if`, `for`, `switch`) never parenthesize the condition, and the opening brace must stay on the same line as the condition — semicolon insertion breaks otherwise.

## Commentary

- Line comments (`//`) are the norm; block comments (`/* */`) are for package doc comments or disabling code.
- A comment directly above a top-level declaration, with no blank line between, becomes that declaration's doc comment.

## Names

- Package names: short, lowercase, single word, no underscores/mixedCaps — named after the source directory (`encoding/base64` → `base64`).
- Exported names lean on the package qualifier for context — write `bufio.Reader`, not `bufio.BufReader`; avoid dot-imports.
- Getters drop `Get`: an unexported field `owner` gets an exported method `Owner()`, not `GetOwner()`; a setter is `SetOwner`.
- One-method interfaces are named `<Method>er` (`Reader`, `Writer`, `Formatter`); reuse canonical method names/signatures (`Read`, `String`, ...) instead of inventing new ones for the same meaning.
- Multiword names use MixedCaps/mixedCaps, never underscores.

## Semicolons

- The lexer auto-inserts a semicolon at line end after anything that could end a statement — so an opening brace can never start its own line.

## Control structures

- No `do`/`while` — only `for`, in three forms: `for init; cond; post {}`, `for cond {}`, `for {}`.
- `if`/`switch`/`for` accept an init statement (`if err := f(); err != nil {}`); use it to scope a local.
- Skip the `else` when the `if` body ends in `return`/`break`/`continue`/`goto` — let error checks fall straight down the page.
- `range` iterates arrays/slices/strings/maps/channels; drop the index or value side with `_` when unneeded; ranging over a string decodes UTF-8 runes, not raw bytes.
- `switch` needs no expression (defaults to `switch true`), cases don't fall through automatically, and comma-separated case lists replace many `if`/`else if` chains.
- Label a loop to `break`/`continue` it from inside a nested `switch`.
- A type switch (`switch v := x.(type) { case T: ... }`) discovers an interface value's dynamic type; reusing the same variable name per case is idiomatic.

## Functions

- Return multiple values instead of smuggling an error code through a return value or an out-parameter — e.g. `func (f *File) Write(b []byte) (n int, err error)`.
- Named result parameters (`func nextInt(b []byte, pos int) (value, nextPos int)`) document intent and let a bare `return` send back their current values.
- `defer` schedules a call for just before the enclosing function returns — pair it with the resource's acquisition (`defer f.Close()` right after `Open`) so cleanup can't be forgotten.
- Deferred call arguments are evaluated when `defer` runs, not when the call fires; deferred calls stack up and run LIFO.

## Data

- `new(T)` zero-allocates and returns `*T`; design types so their zero value is already usable (`sync.Mutex`, `bytes.Buffer`) so callers need no explicit constructor.
- Prefer a composite literal over a hand-built constructor (`return &File{fd, name, nil, 0}`, or keyed as `&File{fd: fd, name: name}`) — keyed fields can appear in any order and omitted fields default to zero.
- `make(T, args)` — not `new` — allocates and initializes slices, maps, and channels, since those are reference types needing internal setup before use.
- Arrays are values (assignment copies, size is part of the type); pass a pointer for C-like sharing — but idiomatic Go reaches for a slice instead.
- Slices reference an underlying array — mutating one through a slice is visible everywhere sharing that array; reslicing changes length within capacity, `cap()` reports the ceiling, `append` grows/reallocates as needed.
- Map keys need equality defined on them (no slices as keys); a missing key returns the zero value, so use the "comma ok" form (`v, ok := m[k]`) to tell absent from zero.
- Use `%v` for a default representation of anything (including structs/maps), `%+v` to label struct fields, `%#v` for a Go-syntax literal, and `%T` for a value's type; give a type a `String() string` method to control its own `%v`/`Print` output.
- Avoid infinite recursion in a `String()` method — convert the receiver to its underlying type before formatting it with a verb that would otherwise call `String()` again.

## Initialization

- Constants are compile-time only (numbers, runes, strings, bools) and can be built with `iota` for enumerations.
- Package-level `var` initializers can run arbitrary expressions at program start; Go orders cross-package initialization correctly on its own.
- Each file may define `init()` (even several) for setup that can't be expressed as a declaration, or to validate/repair program state before `main` runs; all `init`s run after their package's vars initialize, and after imported packages have already initialized.

## Methods

- Value-receiver methods can be called on both values and pointers; pointer-receiver methods can only be called on (addressable) pointers — reach for a pointer receiver whenever the method must mutate the receiver.
- A method set — not just field access — is what makes a type satisfy an interface: e.g. a pointer-receiver `Write` method is what makes `*T` (not `T`) implement `io.Writer`.

## Interfaces and other types

- Interfaces describe behavior, not implementation; keep them small (often one or two methods) and name them for the method they capture.
- Convert between two types with identical underlying structure (e.g. a named slice type and its underlying `[]int`) to borrow the other type's methods cheaply — no copy is made.
- A type assertion (`v, ok := x.(T)`) safely extracts a concrete or interface type from an interface value; without the `ok` form, a mismatch panics.
- If a type exists only to implement an interface, don't export the type — export the interface and have the constructor return the interface value, so callers depend on behavior, not a concrete implementation.
- Nearly any type can grow methods (structs, ints, funcs, channels), which is how a bare function type can itself satisfy an interface like `http.Handler` via a `HandlerFunc` adapter.

## The blank identifier

- Use `_` to discard a value you don't need — a range key/value, or an unwanted return. Never silently discard an _error_ just to make code compile.
- `_` can absorb an otherwise-unused import or variable during active development, and `import _ "pkg"` imports purely for side effects (e.g. a registered `init()`).
- A package-level `var _ Interface = (*Type)(nil)` forces a compile-time check that a type satisfies an interface, when no other code already performs that conversion.

## Embedding

- Embedding an interface in an interface unions their method sets (`ReadWriter` from `Reader` + `Writer`); embedding a type in a struct promotes its methods onto the outer type for free, avoiding hand-written forwarding methods.
- A promoted method still runs with the embedded value as its receiver, not the outer struct — this is delegation, not subclassing.
- Name conflicts resolve by depth (an outer field/method hides a same-named one nested deeper); same-name conflicts at the same depth are only an error if the ambiguous name is actually used.

## Concurrency

- Share memory by communicating over channels rather than by synchronizing shared-variable access — a value owned by one goroutine at a time avoids data races by construction.
- `go f()` runs `f` concurrently in a lightweight goroutine (cheap, growable stack); use a closure when the call needs to capture local state.
- `make(chan T, n)` allocates a channel; unbuffered (`n` omitted or 0) sends and receives rendezvous, so the channel operation itself can serve as a synchronization point.
- Cap concurrent work with a buffered channel used as a semaphore, or with a fixed pool of goroutines reading off one shared channel — don't spawn an unbounded goroutine per unit of work.
- A channel is a first-class value — pass one inside a request struct to give each caller its own private reply path.
- `select` with a `default` case makes a channel operation non-blocking, useful for opportunistic free-lists and similar leaky-bucket patterns.
- Concurrency (structuring independent components) and parallelism (running work simultaneously across cores) are different concerns — Go's model targets the former.

## Errors

- By convention, errors satisfy the built-in `error` interface (`Error() string`); a package can wrap a richer type underneath to carry structured context (see `os.PathError`).
- Error strings should identify their origin (a package/operation prefix) so they're informative even printed far from the call site.
- A caller that needs error detail uses a type assertion/type switch on the returned `error` to recover the concrete type.

## Panic and recover

- `panic` is for truly unrecoverable conditions (or, arguably, failed initialization) — library code should generally return an `error` instead of panicking.
- `recover`, called directly inside a deferred function, stops an in-flight panic and returns its argument; it's a no-op anywhere else.
- A package may use `panic`/`recover` internally to unwind a deep call stack on error (e.g. a parser), but should convert every panic back into a normal `error` before it's visible to callers — don't let panics leak across a package boundary.
