---
name: golang-unit-testing-adaptation
description: Use when a Go codebase has few or no tests because its logic is welded to a database, a message broker, or an HTTP client, and the request is to make it unit-testable. Describes how to find or create dependency seams, extract a repository layer, declare consumer-side ports, replace package-level functions with a service holding injected collaborators, drive HTTP handlers in process, and build a test suite that runs with no database, no broker and no network. Trigger on "add unit tests", "make this testable", "no test coverage", "refactor for testability", "extract a repository layer", "mock the database", "dependency injection in Go", "test the handlers", or any request to introduce interfaces so fakes can replace real infrastructure.
license: MIT
metadata:
  audience: maintainers
  workflow: refactoring
---

# Adapting a Go Codebase to Unit Testing

This skill takes a Go codebase whose tests cannot be written — because the domain logic
opens database connections, dials a message broker, and calls the network inline — and
restructures it so a real unit-test suite exists, runs fast, and passes with every
external service stopped.

The work is not "add tests". Tests are the *output*. The work is to install the seams the
tests need, and the test suite is what proves the seams are real.

## 0. Scope and Precedence

**What this covers.** Restructuring production code to make it testable, and building the
resulting test suite: a repository/persistence layer, consumer-side interfaces for
infrastructure, dependency injection into the domain layer, a port between the HTTP
control layer and the domain, in-memory test doubles, domain-layer tests, and
handler tests driven in process.

**What this does not cover.** Integration, end-to-end, contract and BDD testing. Those
live in a separate suite and deliberately exercise the real collaborators this skill
removes from the unit path. A unit suite is evidence about *logic*, never about *SQL
correctness* or *wire compatibility*; do not let it be cited as such.

**Precedence.** This skill sets a floor for how a codebase is made testable, not the
whole of the change. If the project ships its own agent instructions (`AGENTS.md`,
`CLAUDE.md` or equivalent), those govern, and this skill is applied inside them. The
`spec-driven-development` skill applies too: a specification and a plan must exist on
disk before the first production file is edited. See section 11.

**A worked reference implementation exists.** `psf-file-management`, spec
`specs/000002-unit-testing-capability/`, is a complete instance of this procedure applied
to a five-module gin/GORM service. Read its `spec.md` and `plan.md` when a detail here is
too compressed. Its business rules are not reusable; its *moves* are.

## 1. Why the Codebase Resists Tests

Diagnose before touching anything. A Go service without tests almost always has the same
four walls, and each has a distinct fix:

| Wall | Symptom | Fix |
|---|---|---|
| **Queries inline in the domain** | Domain functions import the ORM and open a connection | Extract a repository layer; the domain depends on an interface |
| **Collaborators reached through package-level globals and functions** | `business.DoThing()` called directly from a handler; `db` is a package variable | Turn functions into methods on a service that holds injected collaborators |
| **No seam on the outbound client** | The domain calls the broker/RPC peer directly | Find the existing injection point, or declare a consumer-side port for it |
| **Handlers construct their own dependencies** | The router function reaches for the domain package itself | Declare a port in the transport layer, pass the service in as a parameter |

Two corollaries that are not walls but decide the project's fate:

- **Fakes in non-test files are a trap.** A `fakes.go` compiled into the production binary
  puts test doubles in production, invites accidental use, and is a supply-chain smell. All
  doubles go in `_test.go` files. See section 7.
- **A seam that is only half-wired is worse than no seam.** If the domain still has a
  package-level function that opens a connection, the tests are green and prove nothing.
  Section 10 has the mechanical checks that catch this.

## 2. Target Architecture

Four layers, dependencies pointing inward only. The arrows mean "imports":

```text
  transport        handler.go, routes.go        declares the port the handlers use
     |                                           receives its collaborator as a parameter
     v
  domain           service.go, ports.go         declares FileStore/ParameterStore/Client ports
     |                                           holds them as fields; no ORM, no client
     v
  repository       *_store.go                   the ONLY place a query is written
     |
     v
  models           model structs, connection    the schema in Go; no query
```

Rules that make the layers enforceable rather than aspirational:

- **Ports are declared by the consumer, in the consumer's package.** `domain` declares
  `FileStore`; `repository` implements it. Not the other way round. A repository that
  declares its own interface forces the domain to import the repository, which couples the
  domain to infrastructure and makes the fake a second implementation of the repository
  rather than a replacement for it.
- **The repository must not import the domain.** Go's structural typing makes the
  `var _ domain.FileStore = (*Store)(nil)` assertion unnecessary — and writing it in the
  repository creates exactly the forbidden import. Satisfaction is proven by the
  compiler at the wiring site in `main`. This is a real trap; see section 8.
- **Models and the connection wrapper live below the repository, not in it.** The
  repository is the only writer of queries; the model package is the definition of the
  schema in Go. Both may be the same package. What matters is that no query and no domain
  logic share a package.
- **The repository stores no connection.** It resolves the connection per call, so
  connection timing and lazy-initialisation behaviour are preserved exactly (section 9).
- **Error *policy* stays in the domain; error *classification* stays in the repository.**
  The repository reports "which constraint rejected this" as data. The domain decides
  what that means and what the client is told. Mixing them is how repositories grow into
  domain layers.

### Multi-module layouts

In a Go workspace the layer split is often also a module split, and it usually already
partially exists — this project had `data`, `business`, `integration`, `service` and
`main` before any of this work. Prefer **a new module for the repository** when the
existing modules already separate concerns; do not add a sixth module if the codebase is a
single module and a package suffices. Either way:

- A module the workspace does not list does not resolve, and every dependent build fails.
  Add the `use` directive as its own first step, before writing a line of query code.
- Run `go mod tidy` in **every** module directory afterwards. Deploy and CI scripts in
  these ecosystems frequently loop tidy over each subdirectory; an untidy module breaks
  the build in a way that looks unrelated to your change.
- Pin the new module's version of the shared library to whatever the existing modules
  already use. Adding a module that requires a *newer* version silently raises the
  workspace's version, which is drift nobody decided on and cannot easily be undone.
  Record the resulting version set in the spec's verification notes.

## 3. Method: Seam First, Then Tests

The order matters and is not negotiable. Writing tests first against an un-injected
codebase produces tests that must be rewritten, and a rewrite is where coverage quietly
disappears.

1. **Inventory** every outbound dependency and, for each, whether a seam already exists.
2. **Install** the missing seams: repository layer, ports, injection, transport port.
3. **Build the doubles** in `_test.go` files.
4. **Write the tests**, domain layer first, transport layer second.
5. **Verify** with the gates in section 10, including the external-services-down run.

### Step 1 — Inventory the outbound edges

Enumerate what leaves the process. For each edge record: what calls it, from which package,
whether it is reached through a variable/field/parameter (a seam) or a hard-wired package
reference (no seam), and what the domain actually needs from it.

```bash
# what leaves the process, and from where
grep -rn --include='*.go' -E '\b(sql\.Open|gorm\.Open|redis\.NewClient|http\.NewRequest|http\.Get|grpc\.Dial|zeromq|sock_|DialContext)\b' .
# where the ORM/query API is actually used
grep -rn --include='*.go' -E 'gorm\.G\[|\.Where\(|\.First\(|\.Find\(|db\.(Query|Exec|Get|Select)' .
# existing injection points: parameters, interface-typed fields, initialisers
grep -rn --include='*.go' -E 'func New[A-Z]|InitializeAlternative|SetUserManager|interface \{' .
```

**Always look for an existing seam before adding one.** Testability holes are frequently
already plugged and simply not used: a constructor that takes an interface, a package-level
`var x Client = &realClient{}`, an `InitializeAlternative...` function. Using the existing
seam means no new production code, no new abstraction to learn, and no risk of a
second, competing path. Record "already solved, no change needed" as a *finding* in the
spec — a finding is a legitimate deliverable and it saves work.

Only declare a new port when no seam exists. Then declare it consumer-side, with one
method per operation the consumer actually performs — no more. Every method takes an
explicit `context.Context` if the real implementation does; that is not decoration, it is
what lets the test cancel and timeout like production.

### Step 2 — Extract the repository layer

The move must be **provably verbatim**. A query retyped by hand can silently change a
`Where` clause, a selected column, or an error path and still compile.

- **Copy, then delete, in two separate steps.** Between them, both versions exist and can
  be diffed. Delete within the same feature; a duplicated query left behind means two
  sources of truth, and the second one will be edited alone.
- Cut and paste; do not retype. Preserve the ORM call shape, the explicit context, the
  `Where` clauses, the column selection, the terminal call (`Count`/`First`/`Create`/
  `Save`/`Update`), and the existing error handling.
- **Diff each query against its original** and confirm the only changed tokens are the
  enclosing function name and receiver. Record that diff in the plan's verification notes.
- Keep the connection resolution per call, in the repository, exactly as the original did
  it. Do not hoist it into a constructor.
- Where a schema fact is duplicated — a constraint name in a struct tag and the same name
  in a constant — see the reflection guard in section 6.

The one intended change to a port signature: if the domain currently misreports a database
failure, the port must carry the information the domain needs to report it correctly.
Widening a port to surface a fact the domain requires is part of the work. Reshaping
queries to suit a port is not.

### Step 3 — Turn package-level functions into methods

Package-level functions reaching for a global connection cannot be injected. Group them
onto a service that holds the collaborators as fields.

- The constructor **requires every collaborator**. A service built partially wired fails
  to compile at the call site instead of nil-dereferencing on the first request.
- **Delete the package-level originals.** Leaving them as wrappers around the methods means
  two implementations and a way for callers to keep bypassing the injection. Verify exactly
  one implementation of each operation survives.
- Functions that are pure — or that only need a collaborator passed in — stay
  package-level. Do not make everything a method; that is a style change, not testability.
- A function that reaches a collaborator *by calling another function that does* must
  become a method too. Tracing the call graph for the connection accessor, not just
  grepping for the ORM, is how this is found. Functions found this way should be
  enumerated explicitly in the spec so the number is not guessed.
- Unreferenced exported functions are **relocated faithfully, not deleted.** Deleting
  public API is a separate decision for a separate spec. Record the dead code as a
  pendency with a recommendation.

### Step 4 — Add the transport port and rewire

- Declare in the transport package an interface exposing **exactly the operations the
  handlers call, and nothing else**, plus a small production adapter that forwards each
  call to the domain service. The adapter exists so the handlers depend on the interface
  and the domain keeps its own dependencies private.
- The route-assignment function takes the port as a **parameter** instead of importing the
  domain package. This is the single change that makes handlers testable.
- Change **nothing else in any handler body**: not the routes, not the methods, not the
  decorators, not a status code, not an envelope key, not a header. Diff the handlers and
  confirm the only changed token per line is the receiver. This is a mechanical
  acceptance criterion and it is the difference between a refactor and a rewrite.
- The production seam (auth, permissions, tracing) keeps running in the handler tests.
  Swap its *collaborator* for a fake; do not add a test-only bypass. A bypass is a branch
  in production code that only tests exercise, and it is never covered by the tests it
  exists for.

### Step 5 — Build the doubles, then the tests

Sections 6 and 7.

## 4. Assertions That Carry Weight

A test that asserts "no panic" is a smoke test. A test that asserts returned values
propagated correctly can fail for the right reason. Prefer, in order:

- **Assert the value**, not the absence of an error. `got == want`, not `err == nil`.
- **Assert error identity and cause together.** `errors.Is(err, ErrTarget)` for the
  classification, plus a check that the wrapped cause survived.
- **Assert the exact wire text** when a client reads the text, but reach for a locally
  built `errors.New(want)` rather than the production sentinel when the point of the
  assertion is the *format* rather than the identity. Depending on the sentinel makes the
  test pass for the wrong reason if the sentinel's text changes.
- **Assert the negative space.** "The service was never called" is a distinct and often
  more valuable assertion than the status code beside it: it proves the guard ran in the
  right *order*.
- **Assert multiplicity when a retry or loop is involved.** "Exactly one insert attempt"
  is what proves a retry was removed. A test that only checks the final error would pass
  with the retry still in place.
- **Assert absence of leakage** when a response must not disclose existence — a private
  resource reported as not-found, a body that must not contain the payload.

## 5. Test Suite Layout

One test file per layer, in the package under test, so the suite sits next to the code it
covers and can reach unexported behaviour.

```text
<repo>/model_test.go          pure guards: duplicated literals, encoders, schema facts
<repo>/ports_test.go          all doubles for the domain package
<repo>/service_test.go        domain logic, against the doubles
<repo>/repository_test.go     only if the repository has pure logic (classification, SQL build)
<transport>/handler_test.go   handlers, in process, against a fake service
<transport>/transport_test.go doubles for the transport package: fake service, request builders
```

One file holding all the doubles for a package, named for the ports, is worth the
inconsistency: it is the single place a reader looks to answer "what can these tests
substitute?".

**Duplicating doubles across packages is correct, not lazy.** A `_test.go` file cannot be
imported by another package's tests, so each package that needs a fake writes one. This
only becomes a problem if the fakes are large; keep them thin, driven by configuration
fields rather than by behaviour, and the duplication stays small.

**No `t.Parallel` in the first suite.** Global collaborators, memoised configuration, and
connection singletons are near-universal in this kind of codebase, and parallel tests
against them produce intermittent failures that cost more to diagnose than the wall-clock
saves. Sequence first; parallelise only after the suite is green and the globals are gone.
Instead of `t.Parallel`, use `t.Cleanup` to restore any global a test changed — the pattern
in section 6.

## 6. The Double Patterns Worth Reusing

These are the reusable pieces. Adapt the field names; the shape is what matters.

### A stateful in-memory store, driven by configuration

Back the double with a slice or map so reads are real, and expose **one field per failure
mode** the test needs to reach. Do not write behaviour-switching logic inside methods; make
the state describe the scenario.

```go
type fakeFileStore struct {
    files []Model                              // real rows, so reads are real

    createAttempts []string                    // one entry per attempt, in order
    rejectWith     string                      // "" = succeed; otherwise fail naming this index
    createErr      error                       // explicit error, else a realistic one
    findErr        error
    payloadMissing bool                        // hand back a row whose payload column is nil
    unavailable    bool                        // every method reports ErrDatabaseUnavailable
}
```

Points that matter:

- **Model the real driver's error, not a bare `errors.New`.** A duplicate-key failure looks
  like `Error 1062 (23000): Duplicate entry ...` in production. A test that feeds the
  domain a generic error is not exercising the branch it thinks it is.
- **Record every attempt.** This is the only way to prove a loop was removed.
- **One shared private helper for the failure modes** when several methods fail the same
  way (`find(matches func(Model) bool)`), so two methods cannot drift apart in how they
  fail.
- **Expose unreachable-looking states as flags.** A nullable column, a row written by
  another service, a negative result — these are only reachable through the double, and a
  flag is honest about it.

### A fake for an outbound client

Configurable user/result, plus separate fields per distinct failure
(`authorizeErr`, `refreshErr`, `returnNilResult`). Methods the tests do not exercise get a
comment saying so, and succeed. A double is a control surface for the test, not a
simulation of the real client.

### Install and restore a global, safely

```go
func installFakeUserManager(t *testing.T, m integration.UserManager) {
    t.Helper()
    previous := integration.LoadUserManager()   // read, if a getter exists
    integration.InitializeAlternativeUserManager(m)
    t.Cleanup(func() { integration.InitializeAlternativeUserManager(previous) })
}
```

`t.Cleanup`, not `defer` in each test: it runs after subtests, it cannot be forgotten by
copy-paste, and it makes the restore invisible in the test body. Add a convenience wrapper
for the common "everything is authorised" baseline so individual tests stay short.

### The engine builder

Build the real router in the test, in the framework's test mode, so routing, middleware and
handlers are all the production ones:

```go
func newTestEngine(files FileService) *gin.Engine {
    gin.SetMode(gin.TestMode)
    router := gin.New()                  // gin.New, not gin.Default: no logger noise
    assignRoutes(router, files)          // the production route-assignment function
    return router
}
```

And one request helper that takes method, target, body and auth header, sets content type
only when there is a body, and returns the recorder. Every handler test then reads as
three lines of intent plus assertions. Equivalent for other frameworks: build the real
mux/router, call it with `httptest.NewRequest` / `httptest.NewRecorder`, or use the
framework's own test client. **No listening socket, no outbound request, no subprocess.**

### Table-driven cases with a configure hook

```go
cases := []struct {
    name       string
    authHeader string
    configure  func(*fakeUserManager)
}{ /* ... */ }

for _, c := range cases {
    t.Run(c.name, func(t *testing.T) { /* install, request, assert */ })
}
```

A `configure func` rather than a struct of every possible field keeps the table readable
and lets a case override one thing. Use `t.Run` subtests so a failure names the case. Prefer
`errors.New` over an unexported `errorString` type for test sentinels.

### Pure round-trip guards — the highest-yield tests in the suite

Encode then decode, over a **range of sizes that includes the boundaries**, and assert the
content came back identical. No collaborators, no fixtures, runs in microseconds.

This is where silent corruption is found, and it is found by *exhaustive boundary
coverage*, not by a representative sample. Documented instance: a decoder required four
bytes of remaining room and returned early **without an error** when fewer were available,
so a buffer sized to the input truncated payloads of 1, 2 and 5 bytes out of every 201
lengths tried. A sample-based fixture at some other length would never have met it, and no
error anywhere would have revealed it. Two consequences for how you write this guard:

- Sweep **every** short length (0 through some bound), not a sample. The bug lives at the
  boundary by definition.
- Assert the **decoded content**, not the absence of an error. The absence of an error is
  exactly what the defect reported.

If the guard fails, the defect is fixed rather than the threshold adjusted, and the fix is
recorded as a behaviour change in the spec, with the reason the guard exists noted so a
later reader does not delete it as redundant.

### A pure guard for duplicated literals

Go struct tags cannot reference constants, so a name in a tag and the same name in a
constant cannot be linked. Parse the tags by reflection in a pure test and assert they
agree. The test needs no database, and it catches the drift that would otherwise silently
disable error classification. Also assert the *absent* cases deliberately — e.g. that a
third constraint is unnamed, so if it ever gains a name the mapping is expected to grow
with it, and that omission is a recorded decision rather than an oversight.

### With GORM: capturing SQL without a database

For write-shaped queries, open a `DryRun` session and inspect the generated statement. This
verifies the assembled SQL against the real schema without sending anything. Note that
`gorm.G[T].Create(ctx, *T)` takes a context but returns only an `error`, so statement
capture needs the non-generic form; and `*gorm.DB.Create` takes no context at all. For
other ORMs, the equivalent is a query builder or an `EXPLAIN` against a live read-only
connection.

## 7. What This Method Finds

Building the test suite for a codebase with no tests reliably surfaces real defects, because
the suite is the first thing that forces every path to be reachable. Treat these as
expected output, fix them in the same feature, and record each in the spec with the
mechanism. The recurring classes:

1. **A retry loop that misreports the cause.** A loop retries an operation on any failure
   and then blames the input it regenerates each iteration, while the actual failing error
   was never inspected — often discarded by a bare `!= nil` comparison whose result is
   dropped. Fix: capture and classify the error, return a distinct named error per
   identifiable cause, and for a failure that cannot be attributed, report **one honest
   generic sentence with nothing appended** — no driver text, no attempt count, no
   identifier. A guess about the cause is worse than admitting there was none. Keep the
   retry only for the unattributable case, and only when the regenerated input can actually
   resolve it.
2. **A discarded error followed by a dereference.** An upstream lookup's error is assigned
   and ignored, and the result is dereferenced. Reachable whenever the peer is unreachable
   or the record is gone; the caller sees a dropped connection instead of an error. Fix:
   one named error, a pure helper that validates the result and wraps the cause so
   `errors.Is` succeeds, applied at every call site. If the handler returns the error text
   as the body, the cause reaching the client is a consequence to state explicitly, not
   something to quietly suppress later.
3. **Silent truncation at a size boundary.** See the round-trip guard above.
4. **"Not found" and "could not be read" are indistinguishable.** A read's error is
   assigned and never inspected, so an empty list and a nil error are returned on failure.
   **Do not fix this here** unless a requirement authorises it: it changes what every
   caller sees. Record it as a pendency with the trade-off spelled out. A refactor that
   silently corrects such behaviour has changed the contract, and the change is now
   invisible in the diff.
5. **A self-inflicted regression during the work.** Widening an error path to a new
   sentinel can break the very behaviour you were preserving. When a change like that lands,
   re-run the pre-existing behavioural tests immediately, and if the plan claims behaviour is
   preserved, prove it with a test that would have failed before.

## 8. Traps

- **A compile-time interface assertion in the wrong package.** `var _ domain.Store = (*Store)(nil)`
  inside the repository imports the domain, inverting the dependency. Go's structural typing
  makes it unnecessary; the wiring in `main` proves satisfaction. Forbid it explicitly in the
  spec — it is the single most common way this architecture gets broken while adding a
  convenience.
- **Half-wired seams.** Removing the ORM from the domain's imports is not enough if a
  package-level function still resolves the connection, or if a handler still calls the
  domain package directly. Green tests and a broken architecture look identical. Section 10.
- **Fakes in a non-test file.** Compiled into the binary, and the fakes become a second
  production implementation. All doubles in `_test.go`. Verify with `go tool nm` on the
  built binary.
- **Normalising inconsistencies you happened to touch.** Error envelopes that disagree
  between handlers, misspelled user-facing strings, inconsistent route registration. Preserve
  them; assert the current behaviour in a test that names it as pre-existing. Fixing them is
  a separate feature with its own spec. Do add a test that pins the inconsistency, so a
  later change cannot alter a client's parsing unnoticed.
- **A test-only bypass in production code.** Anything added purely so tests can skip a path
  is uncovered by the tests that need it and is reachable in production. Substitute the
  collaborator instead.
- **Blanket formatting.** Repositories in this ecosystem are frequently not `gofmt`-clean.
  Running `gofmt -w` across the tree buries a real change in noise and makes the diff
  unreviewable. Check `gofmt -l` before starting, leave the pre-existing offenders alone,
  match the surrounding import-grouping style in files you edit, and keep new files clean.
- **Tidying the wrong thing.** `go mod tidy` in one module of a workspace can remove a
  requirement another module relied on. Expect a `go.mod` to *lose* a dependency as a
  correct result when an import is removed, and record the resulting version set.
- **A stale build artefact polluting greps.** A compiled binary in the tree matches every
  text search. Restrict searches with `--include='*.go'`.
- **Trusting the count.** "Ten call sites" estimated rather than counted will be wrong.
  Count occurrences at their exact locations, and correct the spec when the count differs.

## 9. Preserving Behaviour That Is Not Testability

The refactor must not change observable behaviour that no requirement authorises a change
to. Specifically:

- **Connection timing.** If the connection is established lazily on first use rather than at
  start-up, keep it that way, and keep the resolution per query inside the repository.
  Hoisting it into a constructor turns a database outage at boot — where it is a clear
  failure — into a request-time error, which is a behaviour change.
- **Existing quirks in configuration**, including hardcoded values that other suites depend
  on. Note them as accepted, explain what depends on them, and leave them.
- **The lazy/default semantics of absent configuration.** A missing configuration row
  returning an empty value and no error is what makes defaults reachable. Document it in the
  port's Godoc so it survives the move, and treat a change to it as a behaviour change.
- **Error text a client reads**, unless the text was wrong. Where it changes, it changes
  because a named defect is being fixed, and the spec says which defect.

Record each preserved quirk in the spec as an accepted edge case with its reason. That is
what stops a later reader from "tidying" it away.

## 10. Verification Gates

Run these. Each one has caught something in practice; they are not ceremony.

**A. Compiles and vets, per module.**

```bash
go build ./... && go vet ./...        # in each module directory, or per package path
go build -o <name> ./<main dir>       # the deployable binary
```

In a multi-module workspace `go build ./...` from the root does not work. Iterate the
module directories or target package paths.

**B. The suite passes in one invocation per module.**

```bash
go test -count=1 -timeout 60s ./...            # in each module directory
```

This is the single most informative check in the whole procedure. A codebase whose tests
only pass one-per-process has a global-state problem; getting this to pass in one block is
proof the injection actually took effect.

**C. Order independence, proven rather than sampled.**

```bash
go test -count=1 -shuffle=on ./...             # repeat 5x
```

`go test` builds a cache; `-count=1` defeats it. Run in **both** a green state and, where
the change is large, with the external services stopped (gate E) — passing with the
database up may be a coincidence of ordering.

**D. The layering invariants, checked mechanically.** These are greps, and they are
acceptance criteria:

```bash
# the only package writing queries
grep -rn --include='*.go' 'gorm\.G\[' . | grep -v '/repository/'
# the domain imports no ORM
grep -rn --include='*.go' -E '"gorm\.io|"database/sql' domain/
# the model package writes no query
grep -rn --include='*.go' -E '\.(Where|First|Find|Create|Save)\(' model/
# the repository does not import the domain
grep -rn --include='*.go' '"<module>/domain"' repository/
# no connection resolution outside the repository and the model package
grep -rn --include='*.go' 'GetDatabaseWrapper' .
```

**E. The suite passes with every external service stopped.** This is the gate that
distinguishes a real seam from a decorated one, and it is the one most often argued rather
than demonstrated. **Ask the user to stop the database and any broker/registry, confirm they
are down, and only then run.** Do not claim this check from reasoning about the code: a
test that quietly reaches a real connection passes on the machine where it happens to be
available and fails on CI.

```bash
ss -ltn | grep -E ':<db-port>|<broker-port>' || echo "confirmed down"
go test -count=1 ./...                        # in each module
```

**F. No test double in the production binary.**

```bash
go build -o /tmp/app ./<main dir>
go tool nm /tmp/app | grep -iE 'fake|stub|mock'   # expect only runtime internals
```

**G. Handler bodies unchanged apart from the receiver.** Read `git diff` on the transport
file. Every changed line should differ only by the collaborator. A status code, an envelope
key or a route path appearing in the diff is a finding, not a refactor.

**H. Exactly one implementation of each moved operation.** Count each query at its exact
call site. Residue from the copy-then-delete means two sources of truth.

**I. `go mod tidy` is idempotent across every module.** Capture `git status` of the
`go.mod`/`go.sum` files, run tidy everywhere, capture again, diff. Unchanged means the
change introduced no dependency and no version drift.

**J. The relocated queries still work against the real schema.** Read-only, outside the
unit suite, and deliberately not part of it: connect, run every read-shaped query, assemble
writes through the dry-run facility of section 6, and confirm the table is untouched
afterwards. No DDL, no writes. Delete the temporary harness afterwards — it is not part of
the delivered change. This is the one check that gives evidence about the SQL, and it is
what makes it legitimate to say the unit suite is *not* that evidence.

**K. The tests actually fail when the code is wrong.** Take three meaningful behaviours and
break them: invert a condition, drop an error check, change a status code. Confirm a
specific test fails, then revert. A test suite that survives three deliberate mutations is
decorative, and this is the only way to know before the reviewer does. The fixes in section
7 should each be reproducible this way.

## 11. Planning the Work

The `spec-driven-development` skill governs the artifacts and the no-code gate. For this
kind of change specifically:

- The spec's requirements are the **seams and the invariants**, not the test names. R-items
  like "the domain declares a `FileStore` port covering every persistence operation it
  performs" and "the repository is the only package where a query is written" are testable
  and reviewable. "There are tests for the main flow" is neither.
- Every **layering invariant is an acceptance criterion**, and each has a mechanical check
  in section 10 attached to it. Pair them in the plan so a reviewer can run the check.
- Put the **behaviour-preservation requirements** in the spec as requirements, not as
  assumptions: connection timing stays lazy, the handler diff is receiver-only, no response
  changes except where a named defect is fixed.
- **Order the steps so the tree compiles after each one**, and isolate the two steps that
  need the tree broken. Turning package-level functions into methods breaks the transport
  package until the port exists; say so in the plan rather than leaving a reviewer to find
  the broken intermediate state alarming.
- **The plan must include the external-services-down run, and must state who performs it.**
  It requires a human to stop the database. Do not tick it until the result is reported
  back; record the instruction in the plan's notes instead of claiming the check.
- Add a **pendencies section** for the defects deliberately left unfixed. Each entry states
  the gap, why it was not fixed here, what the consequence is, and which requirement it
  relates to.
- Keep a **verification notes** section recording what each gate actually returned,
  including the version set and the fact that a `go.mod` legitimately *lost* a requirement.
  The next reader should not have to repeat the runs or guess what was checked.
- Tick the plan's checkboxes on disk as work completes, not only in the conversation.

## 12. Completion Report

Report to the user:

- **The gates, with their results.** Which passed, which was performed by them, and
  explicitly that nothing is claimed that was only argued.
- **The seams installed**, as a list of files, and the size of the handler diff — a
  receiver-only diff is the headline evidence that this was a refactor.
- **The defects fixed**, each with the mechanism that found it, because that is the
  argument for the whole exercise.
- **The mutation check** (gate K): what was broken and which test caught it.
- **Anything that looks wrong but was left alone**, and why. Dead code, discarded errors,
  version drift, hardcoded values, a `go.mod` that lost a dependency.
- **What is uncommitted**, and the fact that it awaits their review. Do not commit or push
  unless asked.

If the work is large, say plainly that it is large and unreviewed, and point at the order
the reviewer should read it in: the ports, then the service, then the transport port, then
the doubles, then the tests.
