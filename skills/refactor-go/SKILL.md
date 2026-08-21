---
name: refactor-go
description: Refactor a Go package into a UNIX-style, black-box package built around a clear conceptual model. Invoke as /tbright:refactor-go <package path or description>. Interrogates the user about the package's conceptual model before proposing a plan.
---

# tbright-refactor-go

Refactor one target Go package so it conforms to the principles below, and to any Patterns (see that section) whose trigger fires. One package per invocation — if the arguments name multiple packages, pick or confirm a single one to work on and tell the user to re-invoke for the others.

Background: `references/effective-go.md` condenses idiomatic Go conventions (naming, errors, concurrency, formatting, ...) that this skill's Principles build on. Consult it for idiomatic style questions the Principles don't explicitly cover.

## Principles

Every package this skill touches must conform to all of these.

### General

1. **UNIX package philosophy.** A package is a small, self-contained program that does one small set of related tasks. Larger programs are built by orchestrating smaller packages, which may themselves orchestrate smaller packages. What defines a package's task set is its **conceptual model** — e.g. a single pipeline transform stage, or a client defining the operations between a service and a database.

2. **The conceptual model must be obvious from the public API.** A stranger should understand the package's job from its exported names alone. E.g. a pipeline stage exposes `func Transform(entity A) (B, error)` or `func Run(entity A) (B, error))` — nothing more exotic.

3. **Black-box encapsulation.** Everything not part of the public API is unexported by default. Exported methods and fields are the _only_ way to read or mutate a package's internal state. No back doors.

4. **Purpose-built clients over general-purpose ones.** A package can exist purely to define a focused set of operations (e.g. `Save`, `Load`) around a larger general-purpose library or external system (SQL database, Redis, Temporal, ...). This contains all the technology-specific plumbing to one place instead of spreading it through the codebase.

5. **Business logic lives in public methods/functions; private helpers are focused, single-purpose steps.**
   - Applies broadly, not just to packages that orchestrate other packages: any exported function whose work naturally splits into distinct phases gets each phase extracted into its own named private helper, instead of staying inline.
   - A comment marking off a phase inline (`// Pass 1`, `// Step 2`, `// first, ...`) is itself a signal that phase should be a named function — if you felt the need to label it, name it.
   - For composing packages specifically: the exported method carries the sequence of business logic, and each private method it calls does one specific, narrow task.

   Example (`OrderService.AddItem`):

   ```go
   // AddItem adds qty units of the sku product to order, reserving stock and repricing the order before saving it.
   func (s *OrderService) AddItem(ctx context.Context, orderID, sku string, qty int) (line *OrderLine, err error) {
       order, customer, product, err := s.loadForEdit(ctx, orderID, sku)
       if err != nil {
           return nil, err
       }

       if err = s.checkEligibility(order, customer, product, qty); err != nil {
           return nil, err
       }

       line, release, err := s.reserveStock(ctx, order, product, qty)
       if err != nil {
           return nil, err
       }

       defer func() {
           if err != nil {
               release()
           }
       }()

       if err = s.reprice(ctx, order, customer); err != nil {
           return nil, err
       }

       if err = s.orders.Save(ctx, order); err != nil {
           return nil, err
       }

       return line, s.events.Publish(ctx, OrderUpdated(order))
   }
   ```

   Reading `AddItem` top to bottom tells the whole story; each private helper (`loadForEdit`, `checkEligibility`, `reserveStock`, `reprice`) does one job and is easy to verify in isolation.

6. **Data structures in, data structures out.** Public API functions should be as pure as possible: transforms from one data structure to another via business logic. Even clients wrapping external services should be shaped this way. This is what makes units independently testable.

7. **Accept interfaces for collaborators, structs for data. This is a hard requirement, not a style preference — treat any violation as a bug to fix, not a note to leave for later.**
   - Governs a function's _collaborators/dependencies_ — anything the caller might swap, mock, or that reaches across a service boundary (a store, a client, a publisher, a cache, another service). Those are interface parameters, not concrete types, so a caller never has to convert its own data into a special input type just to call in. Go interfaces are satisfied implicitly, so any struct that already implements the interface works as-is.
   - Does **not** govern a function's primary data entity/payload — the thing being transformed. Per Principle 2's canonical shape (`func Transform(entity A) (B, error)`) and Principle 6 ("data structures in, data structures out"), the entity being operated on stays a concrete struct.
   - Interfaces are for swappable behavior; concrete structs are for data. Return values are always concrete structs, never interfaces — that keeps the output fully inspectable and composable for the next caller in the chain.
   - **Test to apply to every parameter and return value, individually:** would a caller ever want to fake, mock, or swap out the implementation behind this value — in a test, or across environments? If yes, it's a collaborator → must be an interface. Is it instead the thing being read, transformed, or written — data with no behavior of its own? If yes, it's a payload → must be a concrete struct. When genuinely unsure, default to interface for anything that talks to the outside world (DB, network, filesystem, clock, queue, ...).
   - A concrete collaborator type (`*sql.DB`, `*redis.Client`, `*http.Client`, or a hand-rolled concrete store/publisher/service type) appearing directly in an exported function/method signature is always a violation — wrap it behind a purpose-built interface (Principle 4), even if there is currently only one implementation. An interface return type is equally a violation, with no exception.

   ```go
   // WRONG — OrderStore is a concrete struct; callers/tests can't swap or mock it.
   func (s *OrderService) AddItem(ctx context.Context, store *sql.OrderStore, orderID string) (*OrderLine, error)

   // WRONG — OrderLineReader is an interface return type; callers can't inspect fields,
   // compare, or compose the result without an extra type assertion.
   func (s *OrderService) AddItem(ctx context.Context, orderID string) (OrderLineReader, error)

   // RIGHT — collaborator is an interface this package defines for exactly the operations
   // it needs; the payload in and out is a concrete struct.
   type OrderStore interface {
       Save(ctx context.Context, order Order) error
   }

   func (s *OrderService) AddItem(ctx context.Context, orders OrderStore, orderID string) (*OrderLine, error)
   ```

8. **One root data structure per package, in a file named after the package.** Most packages revolve around a single root type (`OrderService`, `Transformer`, ...). That type, its full public API, and any package-level constants or primitive-aliased types (`type Foo string`) live together in one file named after the package — e.g. package `order_service` → file `order_service.go`.

9. **Every public API function is tested, exhaustively.** Private functions are tested too, but more lightly. See the Testing section below for what "exhaustive" means and where the tests live.

10. **Files are organized by purpose, not scattered.** Aside from the package-named file reserved for the root struct/public API/constants (Principle 8), the rest of the package's files should each hold a coherent group of functions that serve one general purpose (e.g. validation, parsing, mapping) — named for that purpose. Avoid AI-generated sprawl: many small, unclearly-named files. Any file under 200 lines should be considered for merging into a related file, unless merging would make the result less readable.

11. **Every exported identifier is documented.** Public functions/methods, types, interfaces, fields, and constants all carry doc comments. See the Documentation section below for the exact format.

12. **One conceptual model, usually one root object — let root count drive package boundaries.** A package's root object (Principle 8) is the concrete home of its conceptual model (Principle 1). When a package genuinely has two such roots, that's a signal it's carrying two conceptual models, and it should usually be split into two packages, each with its own root and public API. Don't split when doing so would break the model's consistency — some things are only coherent held together (e.g. a value and the type-safe constructor that's the only way to produce a valid one). Not splitting along the root count is the exception to justify explicitly, not the default.

13. **Dependencies are declared at construction, not looked up.**
    - A root object's external dependencies — a database client, an API client, another business object — are injected at construction time, never resolved via a global, a service locator, or a DI framework/container.
    - Package them into a config struct named for the type it configures (e.g. a `Service` is built from a `ServiceConfig`, not a generic `Dependencies` bag), with each dependency as its own named field, so a reader can see exactly where every dependency comes from and which ones are injected just by reading the struct.
    - Distinct concern from Functional Options (see Patterns below): the config struct carries required collaborators, Functional Options carries optional, non-dependency configuration — never smuggle a required dependency in behind a `With...` option.

### Tactical

14. **Switch cases stay short.** A `switch` or type-switch case body is capped at 8 lines. Past that, extract the body into a helper function and call it from the case. (`select` statement cases are a different construct and are exempt.)

15. **Never use a `goto` statement.** No exceptions.

16. **Functions shouldn't be longer than 50 lines.** Functions should be generally be around 20-30 lines long, but capped at 50. Any longer and they probably contain code that can be refactored.

## Documentation

- **Public functions/methods:** a comment directly above the signature, starting with the identifier's own name — e.g. for `func Foo(...)`, the comment opens `// Foo is ...` (or `// Foo does ...` for a method). If the function/method exists specifically to satisfy an interface, say so at the end: `... Conforms to the io.Writer interface.`

- **Private functions/methods:** a one-line comment briefly explaining what it does. Most are narrow helpers (Principle 5), so one line is the ceiling, not a minimum to pad out.

- **Exported types (structs, interfaces, constants, primitive-aliased types like `type Foo string`):** a comment directly above the `type X struct`/`type X interface`/`const`/`type X string` declaration explaining what it's for, following the same name-first convention as functions.

- **Exported struct/interface fields:** each exported field gets its own comment explaining what it's for and, where the field's valid values are constrained (an enum-like string, a required range, a format), what those acceptable values are.

- **Scope:** applies to the package's entire resulting public surface after the refactor — including pre-existing exported code the refactor didn't otherwise touch, if it's undocumented or falls short of this format.

## Testing

- **Every exported function/method must have tests.** This applies to the package's entire resulting public API after the refactor — including pre-existing exported methods the refactor didn't otherwise touch, if they fall short of the bar below.

- **Exhaustive means every logic branch in the function is exercised**, plus targeted input-shape coverage:
  - Pointer and function-typed arguments: a case where the argument is `nil`.
  - String arguments: a case where the string is empty.
  - Struct arguments: a case at the zero value, a case fully populated, and one case per field where just that field is zeroed with the rest populated (linear in field count, not the full combinatorial product).
  - Any other branch-relevant input (error returns from collaborators, boundary values, etc.) needed to hit every branch in the function.

- **Every source file with at least one function gets a companion test file.** `decode.go` gets `decode_test.go`, containing the tests for the functions defined in `decode.go`. This applies package-wide — not just the package-named file from Principle 8 (`order_service.go` → `order_service_test.go`), but every purpose-grouped file from Principle 10 too. A file with no functions (e.g. a pure types file) needs no companion — nothing to exercise until it gains one.

- **No orphan test files.** Every `_test.go` file must have a companion non-test file of the same base name. A `testdata_test.go` needs a sibling `testdata.go`; if there isn't one, the orphan file needs refactoring — its tests move into whichever `_test.go` file actually corresponds to the source file whose functions they exercise (creating that companion file if it doesn't exist).

- **Exception: integration tests.** These may live without a companion source file, but must use the `_integration_test.go` suffix, and the filename must describe what the tests cover (e.g. `postgres_integration_test.go`, not something as vague as `testdata_test.go`).

- **Private function tests are lighter-weight.** Most private functions are narrow helpers — cover the primary path and any error path, but don't chase the same exhaustive permutation bar required for public functions.

- **Individual test cases are an implementation detail, not a separate plan-approval item.** The refactor plan (Process step 5) covers file _structure_ — including the resulting test-file layout required by the companion-file rules above — not individual test cases; write the actual test cases during implementation (step 7) without listing every case up front for approval.

## Patterns

Specific, well-known designs to reach for when a package's shape calls for one — refinements layered on top of the Principles above, applied only where a trigger condition actually fits the package's conceptual model. Full write-ups and ready-to-adapt Go examples live in `references/patterns.md`; read that file once a trigger below actually fires for the target package, not before.

**Every pattern in the table below must be actively checked against the target package, every run — not just the ones that happen to come to mind.** A pattern that isn't applicable still gets a row (see Process step 5); silently never mentioning a pattern is how it gets missed.

| Pattern            | Trigger                                                                                                                                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Functional Options | A struct has roughly 4+ independent _optional_ settings — keeps the constructor clean without a telescoping param list. Optional config only; required dependencies still go through Principle 13's config struct. |
| Worker Pool        | Concurrent execution over a stream of jobs where goroutine count needs a cap.                                                                                                                                      |
| Pipeline           | Processing needs multiple sequential stages (e.g. transform → filter → enrich → persist).                                                                                                                          |
| Strategy           | Many if/switch branches picking behavior, each growing separately, or new behavior must plug in without touching the caller.                                                                                       |
| Command            | Undo/redo is needed, actions must be queued/logged for replay, or the invoker must be decoupled from the receiver.                                                                                                 |
| Fan-out/Fan-in     | Work distributed across goroutines to process concurrently (fan-out), results merged back into one channel (fan-in).                                                                                               |
| Repository         | Abstracting a database/external store behind a focused interface — a concrete instance of Principle 4.                                                                                                             |

## Process

1. **Identify the target package** from the arguments (path or description). If multiple are named, confirm with the user which single package to work on this run.

2. **Enter plan mode** (EnterPlanMode) and read the target package's current code to ground the conversation in reality.

3. **Infer the conceptual model from the code first.** Before asking anything, form your own read of what the package actually does from its current exported API, types, and call sites — what specific, narrow job it looks like it's performing. As part of this, count the root objects (Principle 8/12): if there appear to be two or more, name them and state whether they look like two separate conceptual models. State the inference plainly and ask the user to confirm or correct it.

4. **Interrogate on any gaps.** If the user rejects or only partly confirms your inference, or the inferred model leaves the target public API ambiguous, use AskUserQuestion in small rounds until you are 95%+ confident you know the real conceptual model. If a multi-root/split-candidate signal came up in step 3, resolve it explicitly here — confirm whether the package should split (Principle 12) or whether the roots are inseparable and why, before treating the model as confirmed. Then restate your (now revised) understanding explicitly and get the user's confirmation before moving on. Do not proceed to a plan on an unconfirmed model.

5. **Produce the refactor plan** and present it via ExitPlanMode. The plan must:
   - List every public API method/field the refactored package will expose.
   - For each, state the specific task it performs and how that task serves the confirmed conceptual model.
   - **Interface/struct audit (Principle 7):** for every exported function/method, list each parameter and the return type, and classify each one Collaborator→interface or Data→concrete struct per Principle 7's test. Flag and fix any concrete collaborator type or interface return type before the plan is presented — Principle 7's conformance-table row (below) may only read "Yes" if every function in this list passes.
   - Call out which existing exported names become unexported, are removed, or are merged.
   - Note the file layout change (root struct + public API + constants/aliases consolidated into the package-named file, per Principle 8).
   - Name the non-root files the rest of the package's code will live in, grouped and named by purpose, including which existing small/scattered files get merged into which (Principle 10).
   - Call out any code colocation changes — relocating specific struct, interface, or function definitions to a different file for readability — separately from the file-merge list above.
   - Call out the resulting test-file layout: new companion test files to create, existing test files to rename or move, and any orphaned test files that get consolidated or renamed to conform to the companion-file and integration-test rules (Testing section).
   - **Patterns table**, one row per pattern in the Patterns section above (all 7, every run) with columns `Pattern`, `Trigger met?` (Yes / No), `Why`. "Why" must reference a concrete fact about the target package (a signature, a config-knob count, a branch count) for every row, Yes or No alike — a No still needs a one-line reason the trigger doesn't fire, not a blank or a skip. For every Yes, read `references/patterns.md` for that pattern's write-up before folding it into the plan.
   - **Include a conformance table**, one row per numbered Principle (1–15, General and Tactical both — do not skip file-naming/layout principles just because they're also covered narratively above), with columns: `#`, `Principle`, `Conforms after this plan?` (Yes / No / N/A), `Why`. "Why" must state the concrete fact making it true — a file name, a signature, a line count — not a restatement of the principle. A "No" is only acceptable here if the plan explicitly says why full conformance is out of scope for this run; otherwise revise the plan until every row is Yes or N/A. Naming principles (8, 9, 10) get the same row-level scrutiny as every other principle — do not fold them into prose elsewhere and skip the row.

6. **If the user modifies the API contract or plan** (whether during review or after rejecting the plan): check whether the change still conforms to the confirmed conceptual model.
   - If it doesn't conform, say so and propose alternatives that do — don't silently accept a model-breaking change.
   - If it does conform, update the plan and go back to step 5 for re-approval.

7. **Once the plan is approved, implement it.** Keep behavior identical unless the user asked otherwise — this is a structural refactor. Write or backfill tests per the Testing section, and doc comments per the Documentation section, for the package's full resulting public surface (and lighter tests/comments for private helpers). Run `go build ./...` and `go test ./...` on affected packages after changes.

8. **Audit the implemented code against every Principle — General and Tactical — before reporting, and print the updated conformance table.** Re-run the same table from step 5 against the actual diff, not the plan's intent — re-check every row from scratch rather than copying the plan's predictions forward, since the plan's "Yes" was a prediction and this is a measurement. Pay particular attention to checks that are easy to satisfy structurally while still violating in the details: file-naming/layout principles (8, 9, 10) against the actual filenames on disk, line-counting Tactical rules (switch/type-switch cases over 8 lines, any `goto`), every switch/if branch in every function touched, not just the ones the plan called out, and Principle 7 — re-run the interface/struct audit against the actual signatures in the diff (not the plan's list), one parameter and return type at a time; a single concrete collaborator type or interface return type anywhere in the public API fails this row even if the rest of the package conforms. Fix anything found non-conforming and re-check that row before moving on — do not report a refactor as done with a "No" row still in the table.

   **Also re-run the Patterns table from step 5 against the actual implementation, not the plan's predictions.** Implementation sometimes surfaces a trigger the plan missed (e.g. a constructor that grew a 5th optional param during implementation) or shows a planned pattern didn't end up fitting — update the table and the code to match reality, and print the re-checked Patterns table alongside the conformance table in the final report.

9. **Report** which principles/patterns were applied, the resulting public API, the tests and doc comments added/backfilled per function, the final conformance table, and any spots intentionally left as-is with a reason.
