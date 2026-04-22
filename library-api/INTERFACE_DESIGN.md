# Public Library Interface Design Guideline

An actionable, LLM-friendly playbook for designing stable, ergonomic public library (SDK / crate / module) interfaces. Examples use **Rust** and **Go**; the principles apply to any statically-typed language used to expose a public contract to external consumers.

> In an application you can refactor anything — you own all callers. In a library every decision is frozen into every downstream codebase the moment you publish. Optimize for **future-proofing**, **discoverability**, and **safety by default**, not for the shortest time-to-first-commit. The junior-dev instinct is "ship the minimum viable version." The library-author instinct is "ship the minimum version I can commit to forever." Those are genuinely different bars, and most of the rules below exist to close the gap between them.

For Rust specifically, this document assumes familiarity with the official [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) and their [checklist](https://rust-lang.github.io/api-guidelines/checklist.html); it restates the items most often skipped in real codebases and adds cross-cutting rationale and Go analogues. §23 consolidates the Rust-specific `C-*` items for conformance review.

## Table of Contents

### Core Framework
- [1. Core Principles](#1-core-principles)
- [2. Data-Flow Direction & Evolvability](#2-data-flow-direction--evolvability)
- [3. Design for Misuse Resistance](#3-design-for-misuse-resistance)
- [4. Types over Strings](#4-types-over-strings)
- [5. Typed Identifiers (Newtypes)](#5-typed-identifiers-newtypes)
- [6. Error Design](#6-error-design)

### Async & I/O
- [7. Streaming vs Buffered Responses](#7-streaming-vs-buffered-responses)
- [8. Cancellation & Deadlines](#8-cancellation--deadlines)
- [9. Observability Hooks](#9-observability-hooks)

### Ergonomics
- [10. Trait / Interface Shape](#10-trait--interface-shape)
- [11. Configuration](#11-configuration)
- [12. Numbers, Bools, Enums](#12-numbers-bools-enums)
- [13. Ownership & Allocation](#13-ownership--allocation)
- [14. Performance Contracts Are API Contracts](#14-performance-contracts-are-api-contracts)
- [15. Dependency Leakage in Public APIs](#15-dependency-leakage-in-public-apis)
- [16. Naming & Serialization Consistency](#16-naming--serialization-consistency)
- [17. Optimize for Reader Time](#17-optimize-for-reader-time)

### Quality & Release
- [18. Documentation Style](#18-documentation-style)
- [19. Test Utilities & Examples](#19-test-utilities--examples)
- [20. Lints & Warnings](#20-lints--warnings)
- [21. Versioning & Stability](#21-versioning--stability)
- [22. FFI & Cross-Language Contracts](#22-ffi--cross-language-contracts)

### Reference
- [23. Rust API Guidelines Conformance](#23-rust-api-guidelines-conformance)
- [24. Review Checklist](#24-review-checklist)
- [References](#references)

---

## 1. Core Principles

- **Minimum version I can commit to forever** — not minimum viable. Every public item (type, field, method, trait, error variant, feature flag, serialized key) is a promise. Assume someone will depend on it one hour after you publish and never upgrade again.
- **Docs live next to code** — rustdoc / godoc comments on every public item. The README is not a substitute: consumers read the IDE hover tooltip and [docs.rs](https://docs.rs) / [pkg.go.dev](https://pkg.go.dev), not the README.
- **Types narrate intent** — prefer enums, newtypes, and small structs over `String`, `i32`, and raw `bool` flags. If the compiler can express the constraint, it should. Arguments should convey meaning through types, not through `bool` / `Option<T>` toggles (Rust API Guidelines: **C-CUSTOM-TYPE**).
- **Safe defaults, opt-in power** — one obvious way to do the common thing; advanced knobs are explicit and named. Defaults that are merely correct in the simple case are not enough — they must also *fail loudly* on the weird case instead of silently degrading.
- **Additive evolution** — new fields and variants should not break downstream builds unless they represent a real semantic change the consumer *must* reconsider. Most additions are purely informative and should ship as minor releases.
- **Preserve information** — never flatten rich upstream errors, IDs, or structures into plain strings. A string is a sink: once a `reqwest::Error` becomes `format!("{e}")`, the URL, status code, and IO cause are gone, and no amount of downstream log-parsing gets them back.

## 2. Data-Flow Direction & Evolvability

Before applying any "library best practice," ask one question first: **who produces values of this type, and who matches on them?** Many rules — especially around `#[non_exhaustive]` — *flip* depending on that direction. Using the pattern without checking where the `match` lives is one of the easiest ways to get public API design wrong.

Two cases:

**Case 1 — Consumer produces, library matches.** The library defines an enum, and consumers construct values and pass them back. Typical examples are a configuration enum supplied by the caller, or a value returned by a caller-provided callback that the library acts on. Consumers only construct the variants they need; the important exhaustive `match` lives **inside your library**. That means *your* code should break when a new variant appears, so you are forced to handle it deliberately. → Do **not** mark this type `#[non_exhaustive]`.

**Case 2 — Library produces, consumer matches.** Examples: a parser error (`serde_json::Error`), an HTTP result (`reqwest::Error`), a status code the library returns, or an event it emits. Here the exhaustive `match` lives in **consumer code**. When you add a variant, what should happen on their side?

There are **two valid strategies**. Pick one per type, and **document that choice in the type's rustdoc** so consumers know what contract they are relying on:

- **Strategy A — Mark the type `#[non_exhaustive]`.** Downstream exhaustive matches no longer compile without a `_ =>` arm. Consumers keep building across library upgrades, and new variants fall into their catch-all (ideally a logged, fail-safe default):
  ```rust
  match err {
      MyError::Transient(_) => retry(),
      MyError::Permanent(_) => give_up(),
      _ => { tracing::error!(?err, "unknown variant — library ahead of consumer"); give_up() }
  }
  ```
  Good for large external ecosystems, third-party consumers, and types where new variants are expected to be routine and additive.

  **Cost:** silent behavioral drift. A new variant takes effect with whatever fallback behavior the consumer put in `_`, which may be wrong, and nobody notices until production misbehaves.

- **Strategy B — Leave the type exhaustive (no `#[non_exhaustive]`).** Every new variant becomes a hard compile error for consumers when they upgrade:
  ```rust
  match err {
      MyError::Transient(_) => retry(),
      MyError::Permanent(_) => give_up(),
      // no `_ =>` arm — compile error the day a new variant ships
  }
  ```
  Forces each consumer to make a deliberate decision for the new case before their code ships.

  **Cost:** every additive release blocks consumer builds until they write code. Under deadline pressure, this often turns into `_ => unreachable!()` or `_ => panic!()` just to unblock the upgrade — same catch-all, plus a crash.

**How to choose between A and B.** The question is simple: which failure mode is cheaper for this type — a loud build break during upgrade, or silent behavioral drift in production?

| Ecosystem                                                 | Recommended strategy         |
|-----------------------------------------------------------|------------------------------|
| Small, internal, one team, disciplined upgrades           | **B** — leave exhaustive     |
| Security / financial / auth where every variant must be handled exactly right | **B** — leave exhaustive     |
| Many teams / external / third-party consumers / SLA-driven | **A** — `#[non_exhaustive]`  |
| Broad event streams, observability types, informative metadata enums | **A** — `#[non_exhaustive]`  |
| Mixed surface — some enums closed by semantics, some growing | Per-type choice, documented  |

The library's choice directly constrains consumers. Under Strategy A, consumers **cannot** write an exhaustive match; the catch-all is mandatory. Under Strategy B, consumers get a compile break on upgrade, but they can still choose to add a `_` arm themselves if they want defensive behavior. Make this choice per type, not as a blanket rule for the whole library.

When you later add `Starting`, this match fails to compile — in *your* code. That is exactly where the decision ("should traffic be routed to a warming-up service?") must be made. Consumers that never produce `Starting` are unaffected; they keep returning the variants they already used.

If you had marked `HealthStatus` `#[non_exhaustive]`, your own match would have needed a `_ => false` arm. Then `Starting` would silently fall into that arm, and the compiler could no longer remind you that a real routing decision was missing. That is strictly worse when *you own the matcher*.

**Rule of thumb:** point the compile-time break at the party who must make the decision. `#[non_exhaustive]` protects *other people's* builds; leave a type exhaustive when *you* want the compiler to force the decision back onto your own code.

In **Go**, the trade-off is the same, but the mechanism is weaker: there is no `#[non_exhaustive]`, and the compiler does not check `switch` exhaustiveness for enum-like constants by default. That means library-produced values behave like Rust's forward-compatible mode unless consumers opt into tooling such as `exhaustive` via `golangci-lint`. When the library itself owns the `switch`, enable that tooling in your own CI so newly added constants cannot go unhandled silently.

### 2.1 Struct evolvability

The same direction analysis applies to **adding fields** to public structs. A new required field breaks external struct literals and destructuring patterns. Standard defenses (Rust API Guidelines: **C-STRUCT-PRIVATE**, **C-SEALED**, **C-BUILDER**, **C-CTOR**):

- Mark structs intended for future growth `#[non_exhaustive]` — prevents external struct-literal construction entirely; callers must go through a constructor or builder.
- Keep fields private by default; expose via methods or a `Builder`.
- Provide `Foo::new(…)` as a static inherent constructor; use a `FooBuilder` for complex construction.
- **Sealed traits** (trait with a private supertrait) prevent downstream implementations, which would otherwise constrain your ability to add methods with default implementations later.

In Go: keep exported struct fields stable and provide `NewX(…)` constructors. Unexport any field whose invariants are enforced by methods. If consumers rely on struct literals, adding a required field will break them. Either document that construction must go through `NewX(…)`, or accept the break as the signal that callers must migrate.

## 3. Design for Misuse Resistance

A public API should make correct usage easy and incorrect usage hard. This is not extra safety layered on later; it is part of the interface contract. If the type system can prevent a misuse, do that instead of relying on prose like "must be called after connect" or "do not use this client for admin operations."

- **Make impossible states unrepresentable.** If two fields are mutually exclusive, model them as enum variants, not `Option<A>` + `Option<B>` plus documentation saying "exactly one must be set."
- **Use builders to enforce required fields while keeping growth room.** Builders are not just for ergonomics; they validate combinations, preserve forward compatibility, and avoid long parameter lists with positional mistakes.
- **Model phases explicitly.** If an object transitions from disconnected to connected, unauthenticated to authenticated, or uninitialized to ready, reflect that in the types so methods only exist in the valid phase.
- **Expose capability handles.** If some callers may read while others may administer, prefer separate types such as `ReadClient` and `AdminClient` over one large client with runtime permission errors on half the methods.
- **Prevent call-order bugs through types.** The more a library depends on "first call A, then B, then maybe C," the more it should encode that sequencing structurally.

**Rust:**
```rust
pub struct ConnectParams {
    endpoint: url::Url,
    api_key: String,
}

pub struct ConnectParamsBuilder {
    endpoint: Option<url::Url>,
    api_key: Option<String>,
}

impl ConnectParamsBuilder {
    pub fn endpoint(mut self, endpoint: url::Url) -> Self { self.endpoint = Some(endpoint); self }
    pub fn api_key(mut self, api_key: impl Into<String>) -> Self { self.api_key = Some(api_key.into()); self }
    pub fn build(self) -> Result<ConnectParams, ConfigError> {
        Ok(ConnectParams {
            endpoint: self.endpoint.ok_or(ConfigError::MissingField("endpoint"))?,
            api_key: self.api_key.ok_or(ConfigError::MissingField("api_key"))?,
        })
    }
}

pub struct DisconnectedClient { params: ConnectParams }
pub struct ConnectedClient { session: Session }

impl DisconnectedClient {
    pub async fn connect(self) -> Result<ConnectedClient, ClientError> { /* ... */ }
}

impl ConnectedClient {
    pub async fn send(&self, req: Request) -> Result<Response, ClientError> { /* ... */ }
}
```
A caller cannot accidentally `send()` before `connect()`, because the method does not exist on `DisconnectedClient`.

**Go:**
```go
type ReadClient struct { inner *client }
type AdminClient struct { inner *client }

func NewReadClient(cfg Config) (*ReadClient, error) { /* validate cfg */ }
func NewAdminClient(cfg Config) (*AdminClient, error) { /* validate cfg */ }

func (c *ReadClient) Get(ctx context.Context, id ResourceID) (Resource, error) { /* ... */ }
func (c *AdminClient) Delete(ctx context.Context, id ResourceID) error { /* ... */ }
```
Go cannot express typestate as strongly as Rust, but it can still make misuse rarer by not exposing invalid operations on the wrong handle.

## 4. Types over Strings

If a field has a known, finite set of valid values, prefer an **enum** (or the closest equivalent), not a raw `String`. Strings are appropriate for open-ended human input, opaque passthrough data, or deliberately open vocabularies; they are a poor default for closed sets.

**Bad (Rust):**
```rust
pub struct Session { pub lifecycle_state: String } // "active"? "Active"? "ACTIVE"? "alive"?
```
What happens in practice: Monday, Alice writes `state == "active"`. Tuesday, Bob writes `"Active"`. Wednesday, a producer refactor changes the value to `"ACTIVE"`. Thursday, two consumers silently break in production because the compiler accepted all three spellings.

**Good (Rust):**
```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum SessionLifecycleState { Active, Archived, Deleted }
```
The compiler enforces the valid set, serde canonicalizes the wire encoding, IDE autocompletion makes discovery free, and refactors are grep-able.

**Good (Go):**
```go
type SessionLifecycleState string

const (
    SessionActive   SessionLifecycleState = "active"
    SessionArchived SessionLifecycleState = "archived"
    SessionDeleted  SessionLifecycleState = "deleted"
)

func (s SessionLifecycleState) Valid() bool {
    switch s {
    case SessionActive, SessionArchived, SessionDeleted:
        return true
    }
    return false
}
```
Go's type system is weaker here (any string literal is assignable to a named string type), so always pair the type with a `Valid()` method and validate at the deserialization boundary.

*Rationale:* strings force every consumer to parse and validate independently. They also allow silent drift when casing changes, provide no useful autocomplete, and make refactors harder to find reliably.

## 4. Typed Identifiers (Newtypes)

Never expose raw `String` / `uuid::Uuid` for domain IDs in public signatures. Wrap each in a distinct type so the compiler refuses to swap them (Rust API Guidelines: **C-NEWTYPE**).

**Rust:**
```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct TenantId(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct UserId(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct OrderId(pub String);
```
Now the classic argument-swap bug stops compiling:
```rust
fn lookup(user: UserId, tenant: TenantId) -> Permissions { /* … */ }

lookup(ctx.tenant_id, ctx.user_id);
//     ^^^^^^^^^^^^^ expected `UserId`, found `TenantId`
```
In a multi-tenant system this swap is a silent not-found or data-leak incident. The newtype is three lines of boilerplate to prevent a whole class of bug that would otherwise reach production.

**Go** — use named types, not type aliases; `type TenantId = string` (alias) does not create a distinct type, `type TenantId string` (definition) does:
```go
type TenantId string
type UserId   string
type OrderId  string
// Public signatures MUST use these named types, never bare `string`.
```
Go still allows untyped string-literal assignment to a named `string` type, but catches cross-type mix-ups (`TenantId` cannot be passed where `UserId` is expected).

*Rationale:* zero runtime cost; the one real downside in Rust is needing `Deref<Target = str>` or helper accessors to interoperate with string-taking APIs. That boilerplate is worth the safety.

## 5. Error Design

An error type is part of the public contract, so design it as deliberately as the happy path. Rust API Guidelines call this out under **C-GOOD-ERR**: good error types are meaningful, implement `std::error::Error`, are `Send + Sync + 'static`, and carry useful information instead of opaque strings.

**Rules:**

- **Preserve the cause.** Never `format!("{e}")` an upstream error into a string variant. That discards the status code, URL, IO cause, and any structured metadata, and it breaks `.source()` walking in logs and traces. Use `#[source]` (Rust) / `Unwrap()` (Go) to preserve the chain.
- **One variant per handling decision.** If consumers would branch differently (retry, surface to user, page on-call, silently swallow), it is a different variant. Conflating "caller sent bad input" and "the library is misconfigured" under a single `Permanent(String)` is a design bug: the caller cannot tell them apart, yet the correct handling is opposite (`400` vs `500`, show vs hide, user vs on-call).
- **Include actionable metadata on the variant itself.** `RateLimited { retry_after: Option<Duration>, source: … }` not `RateLimited(String)`. The retry delay is structured data, not prose.
- **Stable error codes** for cross-language / wire consumers: a `code: &'static str` alongside the human message lets clients branch without string-matching `Display` output.
- **Implement standard traits.** `Debug`, `Display` (via `thiserror`), `Error`, and — where sensible — `PartialEq` for tests.

**Rust (with `thiserror`):**
```rust
#[derive(Debug, thiserror::Error)]
pub enum ClientError {
    #[error("transient: {message}")]
    Transient {
        message: String,
        #[source] source: Option<Box<dyn std::error::Error + Send + Sync>>,
    },
    #[error("rate limited")]
    RateLimited {
        retry_after: Option<std::time::Duration>,
        #[source] source: Option<Box<dyn std::error::Error + Send + Sync>>,
    },
    #[error("invalid input: {message}")]
    InvalidInput { message: String },
    Unauthorized,
    NotFound,
    Timeout { message: String },
    #[error("internal: {message}")]
    Internal { message: String, #[source] source: Option<Box<dyn std::error::Error + Send + Sync>> },
}
```

Note: this type is **not** `#[non_exhaustive]`. The consumer that matches on it is your own library (see §2). When you add a variant, you *want* your match to break at compile time.

**Go** — sentinel + typed errors, checked via `errors.Is` / `errors.As`, never by string-comparing `err.Error()`:
```go
var (
    ErrUnauthorized = errors.New("unauthorized")
    ErrNotFound     = errors.New("not found")
)

type RateLimitedError struct {
    RetryAfter time.Duration
    Err        error
}
func (e *RateLimitedError) Error() string { return "rate limited" }
func (e *RateLimitedError) Unwrap() error { return e.Err }

type InvalidInputError struct{ Message string; Err error }
func (e *InvalidInputError) Error() string { return "invalid input: " + e.Message }
func (e *InvalidInputError) Unwrap() error { return e.Err }
```
Consumers then write:
```go
var rl *RateLimitedError
if errors.As(err, &rl) { time.Sleep(rl.RetryAfter); /* retry */ }
if errors.Is(err, ErrUnauthorized) { /* surface 401 */ }
```

## 6. Streaming vs Buffered Responses

If any meaningful subset of implementations produces output incrementally (LLM tokens, log tails, large query results, paginated database reads), the **trait method returns a stream**, not a collected `Vec<T>`. A buffered shape forces all implementations to the worst-case latency profile (time-to-first-byte equals time-to-last-byte), prevents mid-stream cancellation, and inflates memory proportional to output size.

The buffered-first mistake is especially costly for generative / streaming workloads: an implementation that has to `collect()` its whole response before returning cannot stream to the end user, and every downstream layer that synthesizes streaming events from strings is paying for work that never needed to exist.

**Rust:**
```rust
use futures::stream::BoxStream;

async fn generate_response(
    &self,
    ctx: CallContext,
) -> Result<BoxStream<'static, Result<Event, ClientError>>, ClientError>;
```

**Go:**
```go
// Prefer a channel-returning method or an io.Reader / iter.Seq2 over []string.
func (s *Service) GenerateResponse(ctx context.Context, in Input) (<-chan Event, error)
```

Provide a **buffered helper** so non-streaming implementations stay ergonomic:
```rust
pub fn buffered(id: Uuid, text: String) -> BoxStream<'static, Result<Event, ClientError>> {
    Box::pin(futures::stream::iter([
        Ok(Event::Start { id }),
        Ok(Event::Chunk { id, text }),
        Ok(Event::Complete { id }),
    ]))
}
```

*Rationale:* one trait method, one consumer code path, both styles ergonomic. Avoid dual `on_x` / `on_x_stream` methods — they double the surface without doubling the value, confuse implementers about which to override, and force the host to pick one to call (usually the streaming one, which makes the buffered method pure ceremony).

**Also:** name methods after what they return (`generate_response`), not after the triggering event (`on_message`). A verb-based name makes the streaming shape obvious and separates event notification ("a message arrived — FYI") from response generation ("produce the assistant's reply"). Those are different responsibilities and often want different hooks; gluing them together forces observer-only implementations to pretend to generate output.

**General principle.** When tempted to ship two versions of an API ("simple" and "powerful"), ask: can the simple version be expressed as a trivial wrapper over the powerful one? If yes → ship only the powerful one plus a helper. If no → you probably have two genuinely different use cases, but be suspicious.

## 7. Cancellation & Deadlines

Any long-running call must propagate cancellation and a deadline from the caller. Otherwise disconnected clients burn compute and money indefinitely (an LLM keeps generating tokens nobody will read), and slow implementations cause cascading timeouts because nothing upstream can signal "give up."

**Rust:**
```rust
pub struct CallContext {
    pub deadline: Option<time::OffsetDateTime>,
    pub cancel:   tokio_util::sync::CancellationToken,
    // ...
}
```
Implementations use `tokio::select!` on `ctx.cancel.cancelled()` and abort upstream calls.

**Go:** standard `context.Context` as the **first parameter** of every I/O-bound public function, always. Respect `ctx.Done()` and `ctx.Deadline()`.

## 8. Observability Hooks

Every call context carries correlation IDs. Retrofitting observability is one of the most thankless tasks in engineering — adding `request_id` to a context in version 0.1 is trivial, adding it in version 3.0 after a hundred consumers exist is a coordinated cross-team migration.

- **`request_id` / `trace_id`** minimum. Ideally a W3C trace context or `opentelemetry::Context` so spans propagate across service boundaries.
- Structured fields (`tenant_id`, `user_id`, `request_id`, etc.) available on the context as typed IDs (§5).
- Provide — or at least document — the tracing/logging crate integration (`tracing` in Rust, `log/slog` in Go). Do not invent a custom logging interface; integrate with the ecosystem standard.

Debugging flow when this is missing: customer reports "request was slow at 14:32," support pings engineering, engineering greps logs for "tenant Foo at 14:32," finds 400 log lines, cannot tell which are the same request, two days of digging. With a single `request_id` you grep once and get every log line across gateway, service, library, and upstream dependencies.

## 10. Trait / Interface Shape

- **Small, orthogonal methods.** Each method has one responsibility. Splitting "notify of event X" from "produce reply" lets observer-only implementations exist without pretending to generate output.
- **Default implementations for optional hooks.** Implementers override only what they need. Keep the README example consistent with this — do not override every method in the sample code, or you contradict the "override only what matters" guidance immediately below it.
- **Prefer composition over deep inheritance-like trait hierarchies.** In Go especially, define small interfaces *at the consumer* (the function that needs them), not at the producer — the producer simply returns concrete types or broad interfaces, and each caller narrows.
- **Document concurrency semantics explicitly.** State whether a client / handle may be used from multiple threads or goroutines concurrently, whether calls may overlap, and whether internal mutation is synchronized or caller-confined.
- **Say whether the API is re-entrant.** If callbacks may call back into the library, or if one method may be invoked again before a previous invocation finishes, document whether that is supported, serialized internally, or forbidden. Re-entrancy bugs are notoriously hard to debug because they appear only under specific timing and callback patterns.

## 11. Configuration

Untyped `serde_json::Value` / `map[string]any` config forces every caller to reinvent schema validation and produces runtime errors for what should be compile-time or load-time checks. A consumer discovering a config bug on first real traffic — instead of at initialization — is a design failure.

**Rust:**
```rust
pub trait Service: Send + Sync {
    type Config: serde::de::DeserializeOwned + schemars::JsonSchema + Send + Sync;
    fn configure(&mut self, cfg: Self::Config) -> Result<(), ConfigError>;
}
```
The host can then validate config against the implementation's schema at initialization time.

**Go:** define a `Config` struct per implementation; accept it in a constructor (`func New(cfg Config) (*Service, error)`) and validate in the constructor, never lazily. Functions should validate their arguments on entry (Rust API Guidelines: **C-VALIDATE**); a library that accepts bad input and fails three layers deep is worse than one that rejects it at the door.

## 12. Numbers, Bools, Enums

- **Prefer non-negative types for counts / indices** that cannot be negative (`u32`, `usize`, `uint32`, etc.), but do not force unsigned integers blindly. If the value participates in subtraction, sentinel values, or FFI boundaries where signed math is simpler, a signed type with validation may be clearer. The goal is semantic clarity, not unsignedness for its own sake.
- **Avoid ≥3 bools on a public struct.** Three bools encode `2^3 = 8` states, most of which are nonsense, and they cannot express "failed mid-stream" — a `Complete=false` message that also failed is neither "complete" nor "incomplete" in any useful sense. Replace with an enum for status plus a small struct for orthogonal flags:
  ```rust
  pub enum MessageStatus { Streaming, Complete, Failed }
  pub struct MessageVisibility { pub user: bool, pub backend: bool }
  ```
  The enum gives you `Failed` for free — a state the three-bool shape could not express.
- **Enums for closed sets** (roles, states, kinds). For open but constrained vocabularies, use a validated string newtype or document the forward-compatibility policy explicitly.
- **Enums for sets of independent flags** (Rust API Guidelines: **C-BITFLAG**). If values can combine (`READ | WRITE`), use the `bitflags` crate, not an enum with `ReadWrite` variants.

## 13. Ownership & Allocation

- **Avoid deep `Clone` on hot paths.** Passing `Vec<Message>` by value copies history on every call; prefer `Arc<[Message]>` (cheap clone, identical read ergonomics) or a slice reference. For long-lived sessions, the difference is the gap between O(1) and O(n) per invocation.
- **Let the caller decide where to copy and place data** (Rust API Guidelines: **C-CALLER-CONTROL**). Accept `&str` rather than `String` when you only read; accept `impl Into<String>` when you store; do not force copies the caller did not ask for.
- **Document ownership.** Say whether a callback receives a borrowed or owned value, whether it may outlive the call, and whether `Send + Sync` bounds apply.
- **Types are `Send + Sync` where possible** (Rust API Guidelines: **C-SEND-SYNC**). Async runtimes (`tokio`, `async-std`) require `Send`; not implementing it gratuitously locks consumers out of multi-threaded executors.
- **Go:** avoid passing large structs by value in hot paths; document whether returned slices/maps may be retained or must be copied (defensive copy on mutation).

## 14. Performance Contracts Are API Contracts

Performance-relevant behavior is part of the API contract. Callers do not just need to know *what* a function returns; they also need to know the cost model well enough to use it safely in hot paths, background jobs, and request handlers.

- **Document clone and copy cost.** `Clone` may be O(1) (`Arc`, handle types) or O(n) (`Vec`, deep trees). If callers will reasonably assume the wrong one, say so explicitly.
- **Document allocation behavior.** Say whether an API allocates, reuses caller-provided buffers, or returns borrowed / zero-copy views.
- **Document execution model.** Say whether a function blocks, may perform I/O, or is CPU-heavy.
- **Document ordering and laziness.** Say whether iteration preserves input order, whether results are lazy or eager, and whether work starts immediately or only when polled / consumed.
- **Document memory growth.** If buffering scales with result size, number of callbacks, or time in flight, say so.

A caller choosing between `items()` and `into_items()`, or between `list_all()` and `stream()`, is making a performance decision as much as a semantic one. Good library APIs make that cost visible instead of hiding it behind innocent-looking names.

## 15. Dependency Leakage in Public APIs

Every third-party type that appears in a public signature leaks part of your implementation strategy into your users' code. Sometimes that is the right trade; often it is an accidental commitment.

- **SemVer coupling.** If your public API exposes `reqwest::Error`, `sqlx::Pool`, or `hyper::StatusCode`, your stability is now coupled to theirs.
- **Forced transitive upgrades.** Consumers may need to upgrade a dependency they did not choose, just to take your new version.
- **Ecosystem lock-in.** A library that exposes one framework's types in every method becomes expensive to adopt in a different stack.
- **Harder internal refactors.** Replacing one HTTP client, async runtime, or serialization crate becomes a breaking change if those types leaked into public signatures.

Wrap external types by default unless they are a deliberate ecosystem standard or the wrapper would be pure boilerplate with no semantic value. `std::time::Duration`, `bytes::Bytes`, `http::StatusCode`, and `url::Url` are often reasonable to expose because they are broadly understood and semantically stable. A niche client, ORM, or runtime-specific handle usually is not.

**Rust:** prefer your own error, request, and configuration types at the boundary:
```rust
pub struct ResponseMeta {
    pub status: u16,
    pub request_id: RequestId,
}

pub enum ClientError {
    Transport { message: String, #[source] source: Option<Box<dyn std::error::Error + Send + Sync>> },
    InvalidResponse { status: u16 },
}
```
This leaves room to swap the HTTP client later without changing the public contract.

**Go:** the same rule applies. Returning `*http.Response` from a low-level transport package may be fine; returning framework-specific types from your domain package usually is not. Prefer your own structs and interfaces at package boundaries.

## 16. Naming & Serialization Consistency

- **Casing conforms to the language convention** (Rust API Guidelines: **C-CASE**, RFC 430): `UpperCamelCase` for types and traits, `snake_case` for functions and fields, `SCREAMING_SNAKE_CASE` for constants in Rust; Go equivalents enforced by `gofmt` / `go vet`.
- **One casing for wire formats.** Pick `snake_case` or `camelCase` across *every* serialized type in the public API. Mixed casing (`messageId` in one event, `message_role` in another) leaks into client code and is embarrassing at integration time.

## 17. Optimize for Reader Time

Library authors write the API once; consumers read it repeatedly for years. Optimize for **reader time**, not author cleverness.

- **Prefer explicit names over clever names.** `retry_after` is better than `backoff_hint`; `ConnectedClient` is better than `LiveHandle` unless the distinction is materially different.
- **Prefer extra types over ambiguous primitives.** A named `TenantId` or `RetryPolicy` costs a few lines once and saves explanation in every call site.
- **Prefer consistency over novelty.** Reusing the platform's naming and shape conventions makes the API feel learnable immediately.
- **Prefer boring APIs over surprising APIs.** A predictable method that behaves like the rest of the ecosystem is usually better than a "smart" abstraction that saves one line when used correctly and costs an hour when misunderstood.

When in doubt, choose the version that is easiest to skim in an IDE, easiest to grep for in a large codebase, and hardest to misread at 2am during an incident.

## 18. Documentation Style

Every public item should have a doc comment. `docs.rs` / `pkg.go.dev` **is** part of the product for external consumers; an uncommented crate looks unmaintained regardless of the code quality.

### Required Documentation Elements

1. **Summary & Purpose**
   - One sentence for what the item is.
   - When to use it vs alternatives.

2. **Invariants & Lifecycle**
   - State guarantees and preconditions.
   - Call-order or ownership assumptions, if any.

3. **Inputs / Outputs**
   - Meaning of parameters and return values.
   - Units, defaults, and constraints where relevant.

4. **Failure Modes**
   - `# Errors`, `# Panics`, and `# Safety` for fallible or `unsafe` APIs (Rust API Guidelines: **C-FAILURE**).
   - Distinguish caller error from library/internal failure.

5. **Code Example**
   - A runnable doctest in Rust (Rust API Guidelines: **C-EXAMPLE**) or an `ExampleFoo` function in Go.
   - Examples use realistic names and values.
   - Rust examples use `?` for error propagation, never `try!` or `.unwrap()` (Rust API Guidelines: **C-QUESTION-MARK**).

6. **Related Items**
   - Link to adjacent types and helper APIs (Rust API Guidelines: **C-LINK**).

### Documentation Template

**Item**: `Client::send`
**Purpose**: Send one request and return a typed response.
**When to use**: Use for single request / response operations; use `stream` for incremental output.
**Errors**: Returns `ClientError::Transient` for retryable failures and `ClientError::InvalidInput` for caller mistakes.

**Example**:
```rust
let response = client.send(request).await?;
println!("{}", response.id());
# Ok::<(), ClientError>(())
```

## 19. Test Utilities & Examples

- **Ship a `test-support` feature / subpackage** with builders and fakes (`MockHost`, `CallContextBuilder::default()`). Without this, every consumer repo copy-pastes the same fixtures and they drift within six months; with it, testing a downstream is three lines.
- **`examples/` directory** at the crate root with a minimal, runnable example per major use case. Cargo builds these as part of `cargo test`, and `docs.rs` surfaces them.
- **Example coverage should mirror the main consumer paths**: one minimal setup example, one error-handling example, and one advanced / streaming example if the surface supports it.
- **Doctests** (Rust) / **Example functions** (Go) for every non-trivial public function. Doctests double as compile-checked documentation: if the signature changes, the doc breaks visibly.

## 20. Lints & Warnings

- **Do not blanket `#![allow(...)]`** at the crate root. Lints like `clippy::struct_excessive_bools`, `clippy::module_name_repetitions`, and `clippy::struct_field_names` point at real design smells; disabling them crate-wide is like taping over the check-engine light. Fix the code, or scope the `#[allow(…)]` to the specific item that legitimately needs the exception and leave a comment explaining why.
- **Deny warnings in CI** for the library crate itself (`#![deny(warnings)]` on CI builds, not on published crates — the latter breaks downstream on new compiler versions).
- **Run `clippy::pedantic` selectively** and opt out per-item rather than disabling groups wholesale.
- **Go:** run `staticcheck`, `govet`, `errcheck`, and `golangci-lint` in CI; treat their output as errors for public-API code.

## 21. Versioning & Stability

- **SemVer is the contract.** `0.x` is explicitly unstable — every `0.y` bump may break. Do not claim "stable API" on `0.1.0`; either soften README language ("pre-1.0, expect breaking changes") or commit to 1.0.0 once blocking design issues are resolved.
- **Grow additively after 1.0** using `#[non_exhaustive]` where §2 says to, and a disciplined deprecation policy where it doesn't.
- **Changelog** with explicit **Breaking**, **Added**, **Deprecated**, **Fixed** sections for every release (Rust API Guidelines: **C-RELNOTES**). Consumers read this to plan upgrades.
- **Deprecation policy:** mark with `#[deprecated(since = "…", note = "use X instead")]` (Rust) or a doc comment + `// Deprecated:` line (Go) for at least one minor version before removal in the next major.
- **MSRV / Go version** declared in the manifest and tested in CI — a silent MSRV bump breaks users on older toolchains.
- **Public dependencies of a stable crate are stable** (Rust API Guidelines: **C-STABLE**): if you re-export `reqwest::Error` in your public API, your crate cannot be more stable than `reqwest`. Either hide third-party types behind your own wrappers or accept the coupling.
- **License permissively** where possible (Rust API Guidelines: **C-PERMISSIVE**): MIT / Apache-2.0 dual license is the Rust-ecosystem norm and removes a friction point for adoption.

## 22. FFI & Cross-Language Contracts

If the library is consumed by non-host languages (wire protocol, webhooks, WASM, cgo, gRPC), the README states this explicitly and links to the wire-protocol document. Otherwise external authors find the crate and are confused about whether they can use it directly or must go through the wire contract.

- **Wire types avoid language-specific niceties.** No Rust `Duration` on the wire — use ISO-8601 or integer milliseconds with units in the field name (`timeout_ms`). No Go `time.Time` with `time.Local` — always UTC with explicit timezone.
- **Enum values serialize as stable strings**, never numeric discriminants. A numeric discriminant becomes a wire break the first time you reorder the enum.
- **Document the stability tier separately** for the in-process API and the wire API — they evolve at different rates. In-process may break at 2.0; wire contracts usually cannot.

## 23. Rust API Guidelines Conformance

This section consolidates the items from the [official Rust API Guidelines checklist](https://rust-lang.github.io/api-guidelines/checklist.html) most relevant to public library design. Use it as a pre-release gate alongside §24. References below link back to the authoritative text.

### Naming
- **C-CASE** — casing conforms to RFC 430 (`UpperCamelCase` types, `snake_case` values).
- **C-CONV** — ad-hoc conversions follow `as_` / `to_` / `into_` conventions (cheap ref / expensive owned / consuming).
- **C-GETTER** — getters omit the `get_` prefix.
- **C-ITER** / **C-ITER-TY** — collection iterators use `iter` / `iter_mut` / `into_iter`; iterator types match method names.
- **C-FEATURE** — Cargo feature names avoid placeholder words (`use-std`, not `with-std-feature`).
- **C-WORD-ORDER** — consistent word order across related items.

### Interoperability
- **C-COMMON-TRAITS** — eagerly derive/implement `Copy`, `Clone`, `Eq`, `PartialEq`, `Ord`, `PartialOrd`, `Hash`, `Debug`, `Display`, `Default` where applicable.
- **C-CONV-TRAITS** — conversions go through `From` / `Into` / `AsRef` / `AsMut`.
- **C-COLLECT** — collections implement `FromIterator` and `Extend`.
- **C-SERDE** — wire/data structures implement `Serialize` / `Deserialize` (behind a feature flag if `serde` is optional).
- **C-SEND-SYNC** — types are `Send + Sync` where possible; document when they are not and why.
- **C-GOOD-ERR** — error types implement `Error`, are `Send + Sync + 'static`, and carry structured information (see §6).

### Documentation
- **C-CRATE-DOC** — crate-level `//!` docs are thorough and include a motivating example.
- **C-EXAMPLE** — every public item has a rustdoc example.
- **C-QUESTION-MARK** — examples use `?`, never `try!` or `.unwrap()`.
- **C-FAILURE** — `# Errors`, `# Panics`, `# Safety` sections on every fallible / panicking / `unsafe` function.
- **C-LINK** — prose hyperlinks to related items.
- **C-METADATA** — `Cargo.toml` includes `authors`, `description`, `license`, `homepage`, `documentation`, `repository`, `keywords`, `categories`.
- **C-HIDDEN** — `#[doc(hidden)]` on items that exist only for macro/internal use.
- **C-RELNOTES** — release notes document every significant change.

### Predictability
- **C-CTOR** — constructors are static inherent methods (`Foo::new`), not trait methods.
- **C-METHOD** — functions with a clear receiver are methods on that type.
- **C-NO-OUT** — no out-parameters; return tuples or structs.
- **C-OVERLOAD** — operator overloads behave unsurprisingly (`+` is commutative and pure, etc.).
- **C-DEREF** — only smart pointers implement `Deref` / `DerefMut`.
- **C-SMART-PTR** — smart pointers do not add inherent methods (to avoid shadowing methods on the pointee).

### Flexibility
- **C-INTERMEDIATE** — expose intermediate results so callers can avoid duplicate work.
- **C-CALLER-CONTROL** — the caller decides where to copy and place data.
- **C-GENERIC** — minimize parameter assumptions via generics (`impl AsRef<Path>` over `&Path`).
- **C-OBJECT** — traits intended for `dyn Trait` use are object-safe.

### Type safety
- **C-NEWTYPE** — newtypes provide static distinctions (see §5).
- **C-CUSTOM-TYPE** — arguments convey meaning through types, not `bool` / `Option<T>`.
- **C-BITFLAG** — sets of independent flags use `bitflags`, not enums.
- **C-BUILDER** — builders for complex construction.

### Dependability
- **C-VALIDATE** — functions validate their arguments on entry.
- **C-DTOR-FAIL** — destructors never fail.
- **C-DTOR-BLOCK** — blocking destructors have non-blocking alternatives.

### Debuggability
- **C-DEBUG** — all public types implement `Debug`.
- **C-DEBUG-NONEMPTY** — `Debug` output is non-empty and useful.

### Future-proofing
- **C-SEALED** — sealed traits protect against downstream implementations that would constrain future evolution.
- **C-STRUCT-PRIVATE** — struct fields are private by default; expose through methods or builders.
- **C-NEWTYPE-HIDE** — newtypes encapsulate implementation details.
- **C-STRUCT-BOUNDS** — data structures do not duplicate derived trait bounds.

### Necessities
- **C-STABLE** — public dependencies of a stable crate are themselves stable.
- **C-PERMISSIVE** — crate and transitive dependencies use permissive licenses (MIT / Apache-2.0 dual is the Rust norm).

## 24. Review Checklist

Use before publishing any new public item or a new library version. This is intentionally shorter than §23 — it catches the items most frequently missed in review.

**Types & signatures**
- [ ] Every public item has a doc comment explaining *what*, *why*, and *when*.
- [ ] No raw `String` where an enum or validated newtype fits; no ambiguous numeric type where units or sign matter; no `bool` where an enum fits.
- [ ] Domain IDs are newtypes, not raw strings or UUIDs.
- [ ] No ≥3 `bool` fields on a public struct; state is an enum.
- [ ] Misuse-resistant shapes considered: builders, phase types, capability handles, or narrower interfaces where sequencing/permissions matter.
- [ ] Wire casing is uniform across all serialized types.

**Errors & evolution**
- [ ] Errors preserve source via `#[source]` / `Unwrap()`; one variant per handling decision; `RateLimited` carries `retry_after`.
- [ ] `#[non_exhaustive]` applied only to types whose matcher lives outside this crate (§2).
- [ ] Each enum is classified deliberately as a closed semantic enum or a growing taxonomy, and the evolution strategy is documented (§2).

**Async & observability**
- [ ] Long-running methods return streams and accept cancellation + deadline.
- [ ] Context type carries `request_id` / trace context and typed IDs.
- [ ] Performance-relevant behavior (allocation, ordering, laziness, blocking, memory growth) is documented where a caller could reasonably depend on it.

**Quality & release**
- [ ] No crate-level `#![allow(clippy::…)]`; all allows are item-scoped with a comment.
- [ ] `examples/` contains a runnable minimal example; a `test-support` feature ships builders/fakes.
- [ ] Public signatures wrap incidental third-party types unless coupling to an ecosystem standard is deliberate.
- [ ] Cargo.toml metadata complete (**C-METADATA**); license is permissive (**C-PERMISSIVE**).
- [ ] Common traits derived (**C-COMMON-TRAITS**); all public types implement `Debug` (**C-DEBUG**).
- [ ] SemVer claim in README matches reality (no "stable" on `0.x`).
- [ ] Changelog updated; deprecations carry `since` and a migration note.

## References

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — official style & design recommendations.
- [Rust API Guidelines Checklist](https://rust-lang.github.io/api-guidelines/checklist.html) — enumerated `C-*` items.
- [SemVer specification](https://semver.org/) — versioning rules for public APIs.
- [Effective Go](https://go.dev/doc/effective_go) — idiomatic Go, including interface design.
- [`thiserror`](https://docs.rs/thiserror) — ergonomic error-type derivation in Rust.
- [`tokio_util::sync::CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html) — structured cancellation for async Rust.