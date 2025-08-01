# Rust Coding Guidelines

These are human-friendly and AI-compatible guidelines for building robust, maintainable, and idiomatic Rust applications, especially in async and service-oriented contexts.

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [A. Project Foundation (Setup & Structure)](#a-project-foundation-setup--structure)
  - [1. Use the Latest Rust Edition](#1-use-the-latest-rust-edition)
  - [2. Choose the Right Project Layout](#2-choose-the-right-project-layout)
    - [Flat Layout (Simple Crates)](#flat-layout-simple-crates)
    - [Workspace Layout (Complex Apps / Services)](#workspace-layout-complex-apps--services)
  - [3. Workspace Management](#3-workspace-management)
  - [4. Configuration and Secrets](#4-configuration-and-secrets)
      - [Explanation](#explanation)
  - [5. Versioning, Linting, Formatting](#5-versioning-linting-formatting)
  - [6. Feature Flags and CI](#6-feature-flags-and-ci)
- [B. Code Organization & Architecture](#b-code-organization--architecture)
  - [1. Organize Code Logically (Prefer Feature-Based Modules)](#1-organize-code-logically-prefer-feature-based-modules)
  - [2. Avoid Global State, Prefer Explicit Dependency Injection](#2-avoid-global-state-prefer-explicit-dependency-injection)
  - [3. Avoid Overengineering Too Early](#3-avoid-overengineering-too-early)
  - [4. Documentation and Examples](#4-documentation-and-examples)
- [C. Error Handling & Safety](#c-error-handling--safety)
  - [1. Prefer `thiserror` and Specific Error Types](#1-prefer-thiserror-and-specific-error-types)
  - [2. Handle Errors Explicitly, Avoid `unwrap()`](#2-handle-errors-explicitly-avoid-unwrap)
  - [3. Validate Input Early and Clearly](#3-validate-input-early-and-clearly)
  - [4. Prefer Compile-Time Over Runtime Safety](#4-prefer-compile-time-over-runtime-safety)
  - [5. Secure by Default](#5-secure-by-default)
- [D. Async & Runtime](#d-async--runtime)
  - [1. Use Tokio Runtime for Async Work](#1-use-tokio-runtime-for-async-work)
  - [2. Graceful Shutdown](#2-graceful-shutdown)
- [E. Testing & Quality](#e-testing--quality)
  - [1. Write Tests: Unit, Integration, and Property-Based](#1-write-tests-unit-integration-and-property-based)
  - [2. Use Logging and Tracing](#2-use-logging-and-tracing)
- [F. Naming Conventions & API Design](#f-naming-conventions--api-design)
  - [1. General Naming Rules](#1-general-naming-rules)
    - [Explanation](#explanation-1)
  - [2. Test Function Naming](#2-test-function-naming)
    - [Not recommended](#not-recommended)
    - [Recommended](#recommended)
    - [Explanation](#explanation-2)
    - [Optional: Parametrized Test Names](#optional-parametrized-test-names)
  - [3. Public API Naming](#3-public-api-naming)
    - [HTTP Route Naming](#http-route-naming)
    - [Explanation](#explanation-3)
    - [JSON Field Naming](#json-field-naming)
    - [Explanation](#explanation-4)
    - [gRPC Protobuf Definitions](#grpc-protobuf-definitions)
      - [Message and Field Naming](#message-and-field-naming)
      - [Explanation](#explanation-5)
      - [Rust-Specific Notes](#rust-specific-notes)
      - [Compatibility with Other Languages](#compatibility-with-other-languages)
  - [4. Newtype Domain Wrappers](#4-newtype-domain-wrappers)
    - [Prefer Newtypes Over Aliases for Domain Values](#prefer-newtypes-over-aliases-for-domain-values)
    - [Implement `Default` for Empty/Sentinel States](#implement-default-for-emptysentinel-states)
    - [Implement `pub const fn as_ref()` Alongside `AsRef`](#implement-pub-const-fn-as_ref-alongside-asref)
      - [Explanation](#explanation-6)
    - [Optional Trait Derives](#optional-trait-derives)
  - [5. Ownership and Borrowing in API Design](#5-ownership-and-borrowing-in-api-design)
    - [Prefer Owned Values When in Doubt](#prefer-owned-values-when-in-doubt)
    - [Avoid Allocations When Possible](#avoid-allocations-when-possible)
    - [Implement APIs in Terms of Borrows Where Possible](#implement-apis-in-terms-of-borrows-where-possible)
    - [Use Generic Bounds for Flexible APIs](#use-generic-bounds-for-flexible-apis)
    - [Prefer `where` Clause Syntax for Generic Code](#prefer-where-clause-syntax-for-generic-code)
    - [Guidelines Summary](#guidelines-summary)
- [G. Recommended Tools & Libraries](#g-recommended-tools--libraries)
  - [1. Crate Recommendations](#1-crate-recommendations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

---

# A. Project Foundation (Setup & Structure)

## 1. Use the Latest Rust Edition

* Always use the latest Rust edition (`2024` as of now) in all crates:

```toml
[package]
edition = "2024"
```

* This ensures access to the latest language features and compiler improvements.

---

## 2. Choose the Right Project Layout

### Flat Layout (Simple Crates)

* For libraries, CLI tools, or small services, use a flat layout:

```
my_crate/
├── README.md
├── .gitignore
├── Cargo.toml
├── src/
│   ├── main.rs
│   └── lib.rs
```

* For libraries, do not commit the `Cargo.lock` file. Add it to `.gitignore`.
* For binaries, commit the `Cargo.lock` file.

### Workspace Layout (Complex Apps / Services)

* For larger applications, use a Cargo workspace:

```
my_project/
├── README.md
├── .gitignore
├── Cargo.toml  # Workspace definition
├── crates/
│   ├── api/
│   ├── service/
│   └── model/
├── bins/
│   └── http-api/
```

* For libraries, do not commit the `Cargo.lock` file. Add it to `.gitignore`.
* For binaries, commit the `Cargo.lock` file.

Use the latest resolver and workspace dependencies:

```toml
[workspace]
resolver = "3"
members = ["crates/*", "bins/*"]
default-members = ["bins/*"]

[workspace.dependencies]
tokio = { version = "1.38", features = ["full"] }
thiserror = "1.0"
eyre = "0.6"
serde = { version = "1.0", features = ["derive"] }
```

---

## 3. Workspace Management

* Use workspace inheritance (`[workspace.dependencies]`) to avoid version mismatches.
* Keep top-level `Cargo.lock` for apps (omit for libraries).

---

## 4. Configuration and Secrets

* Use the `config` crate (or `figment`, `envy`, etc.) to load layered config:

```rust
#[derive(Deserialize)]
struct Settings {
    port: u16,
    database_url: String,
}
```

* Load from environment variables, files, or CLI args. Never commit secrets.
* Use [`dotenvy`] in your application code to load `.env` files; store environment variables and secrets there.
* Add `.env` files as a `.gitignore` rule.
* Document any default configuration assumptions in `README.md`.

#### Explanation

* In Docker and Kubernetes deployments, configuration is trivially injected through files or environment variables
* Using `.env` files allows to keep private configuration while being deployment-compatible

[`dotenvy`]: https://github.com/allan2/dotenvy

---

## 5. Versioning, Linting, Formatting

* Use `rustfmt` and `clippy` on CI.
* Add `.cargo/config.toml` to enable strict defaults:

```toml
[alias]
check-all = "check --all-targets --all-features"
fix = "clippy --fix --allow-dirty --allow-staged"

[target.'cfg(all())']
rustflags = ["-Dwarnings"]
```

---

## 6. Feature Flags and CI

* Use conditional compilation for platform or feature-specific code (`#[cfg(...)]`)
* Use `cargo nextest` for faster test runs in CI
* Structure CI to include: `cargo check`, `cargo clippy`, `cargo fmt`, `cargo test`

---

# B. Code Organization & Architecture

## 1. Organize Code Logically (Prefer Feature-Based Modules)

* Group related code (routes, services, models) in self-contained modules or crates:

```
src/
├── lib.rs
├── user.rs
├── user/
│   ├── model.rs
│   ├── handler.rs
│   └── service.rs
```

* Avoid splitting by "controllers/services/models" unless you're working in a monolith with shared concerns.

---

## 2. Avoid Global State, Prefer Explicit Dependency Injection

* Use struct-based injection for service composition:

```rust
pub struct AppState {
    db: PgPool,
    email: EmailClient,
}

impl AppState {
    pub fn new(db: PgPool, email: EmailClient) -> Self {
        Self { db, email }
    }
}
```

* Share state safely with `Arc<AppState>`.

---

## 3. Avoid Overengineering Too Early

* Don't introduce traits just to "test easily". Stick to concrete types until abstraction is justified.
* Don't prematurely build plugin systems, service locators, or fancy macro DSLs.

---

## 4. Documentation and Examples

* Write `//!` crate-level docs and `///` API docs.
* Include examples in `examples/` or inline doc tests with `/// ```rust`.

---

# C. Error Handling & Safety

## 1. Prefer `thiserror` and Specific Error Types

* Define domain-specific error types early using `thiserror`.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum UserServiceError {
    #[error("User not found")]
    NotFound,
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
}
```

* Avoid `eyre::Error` in core business logic; reserve it for top-level catch-alls (e.g., CLI main or HTTP error mappers).
* Using `eyre::Error` in unit tests is allowed.
* Avoid `Result<_, Box<dyn std::error::Error + Send + Sync + 'static>>` return values.
* Prefer `thiserror`; if you must implement error types manually, always implement the `std::error::Error` trait for them.

---

## 2. Handle Errors Explicitly, Avoid `unwrap()`

* Convert errors as early as possible into your domain types or propagate via `?`.
* Use `expect()` in production code only if you can guarantee that it will not fail.
* Never use `unwrap()` in production code.
* In unit tests, prefer `expect` over `unwrap`, or use the `assert!` macros.

---

## 3. Validate Input Early and Clearly

* Use typed structs with validation in deserialization layers:

```rust
#[derive(Deserialize)]
pub struct CreateUser {
    #[serde(rename = "username")]
    pub name: String,
    #[serde(rename = "email")]
    pub email: String,
}
```

* If applicable, use crates like `validator` or `serde_valid` to validate structured input.

---

## 4. Prefer Compile-Time Over Runtime Safety

* Use `Option` and `Result` for error signaling.
* Prefer `enum` variants for domain logic over stringly-typed logic.
* Minimize use of `unsafe`; encapsulate it behind safe APIs if needed.
* Consider using `#![forbid(unsafe_code)]` on the `lib.rs` or `main.rs`. See [Rust Safety Dance].

[Rust Safety Dance]: https://github.com/rust-secure-code/safety-dance

---

## 5. Secure by Default

* Always validate input and sanitize outputs
* Avoid leaking internal errors via HTTP responses—wrap in proper error responses
* If exposing public APIs, document and version them.

---

# D. Async & Runtime

## 1. Use Tokio Runtime for Async Work

* For async code, use `tokio` as the default runtime.

```rust
#[tokio::main]
async fn main() -> eyre::Result<()> {
    // startup
    Ok(())
}
```

* Prefer Rust's native `async` support in trait objects where possible.
* If unergonomic, prefer `async-trait` for trait objects involving async.

---

## 2. Graceful Shutdown

* On servers, always support `SIGTERM` / `SIGINT` for graceful shutdown:

```rust
tokio::signal::ctrl_c().await?;
```

* Cancel long-running tasks via `tokio::select!` and `CancellationToken`

---

# E. Testing & Quality

## 1. Write Tests: Unit, Integration, and Property-Based

* Unit test all business logic in isolation.
* Integration tests should be in `/tests`, optionally using `tokio::test`.

```rust
#[tokio::test]
async fn test_create_user() {
    let client = setup_test_client().await;
    let response = client.post("/users").json(&payload).send().await.unwrap();
    assert_eq!(response.status(), 201);
}
```

* Use `proptest` or `quickcheck` for data variation when needed.

---

## 2. Use Logging and Tracing

* Prefer `tracing` over `log`:

```rust
use tracing::{info, error};

info!(user_id = %uid, "User created successfully");
```

* Use `tracing_subscriber` for layered structured logging.
* Pass context (`trace_id`, `request_id`) through spans for better observability.

---

# F. Naming Conventions & API Design

## 1. General Naming Rules

| Entity              | Convention                                   | Example                            |
| ------------------- | -------------------------------------------- | ---------------------------------- |
| Modules             | `snake_case`                                 | `mod user_service;`                |
| Structs & Enums     | `PascalCase`                                 | `struct CreateUserInput`           |
| Traits              | `PascalCase`, usually a capability or action | `trait Serialize`, `trait UseCase` |
| Functions & Methods | `snake_case`                                 | `fn process_user()`                |
| Constants           | `SCREAMING_SNAKE_CASE`                       | `const MAX_RETRIES: u32 = 5;`      |
| Type Aliases        | `PascalCase`                                 | `type UserId = Uuid;`              |
| Generic Parameters  | `T`, `E`, `Item` or domain-meaningful        | `fn parse<T>() where T: Deserialize` |
| Feature Flags       | `snake_case`                                 | `#[cfg(feature = "tls")]`          |

### Explanation

* These follow standard Rust community idioms.
* Avoid acronyms in all caps unless conventional (e.g., `HttpClient`, not `HTTPClient`).
* Avoid Java-style naming (`get_`, `set_`) unless it improves clarity — prefer `fn name()` over `fn get_name()`.

---

## 2. Test Function Naming

**Avoid test function names that redundantly contain `test_`.**
The attribute `#[test]` already marks the function as a test.

### Not recommended

```rust
#[test]
fn test_create_user() { ... }

#[test]
fn test1() { ... }
```

### Recommended

* **Concise behavior-focused names** for simple units:

```rust
#[test]
fn creates_user_with_valid_input() { ... }
```

* **`when_condition_then_result` style** for behavioral clarity:

```rust
#[test]
fn when_email_is_invalid_then_returns_error() { ... }

#[test]
fn when_user_does_not_exist_then_returns_404() { ... }

#[test]
fn when_updating_profile_then_database_is_updated() { ... }
```

### Explanation

* This naming pattern makes the test suite read like a specification.
* It's especially helpful for developers coming from Python (`unittest`, `pytest`) or Java/Kotlin (JUnit/Spock), where test method names often follow `testMethodName` or `given_when_then` naming.
* Avoid overly verbose names. Be specific, but concise.

### Optional: Parametrized Test Names

* When using crates like [`test-case`](https://docs.rs/test-case), provide descriptive labels:

```rust
#[test_case("a@b.com", true; "valid email")]
#[test_case("invalid@", false; "invalid email")]
fn validates_email_format(input: &str, expected: bool) {
    assert_eq!(validate_email(input), expected);
}
```

---

## 3. Public API Naming

When designing HTTP APIs or exposing serialized structs to external clients (e.g., via JSON or gRPC), follow consistent naming for compatibility and readability.

### HTTP Route Naming

* Use **`kebab-case`** for URL path segments:

```
GET    /api/v1/users
POST   /api/v1/create-user
GET    /api/v1/users/{user_id}/reset-password
```

* Avoid `snake_case` in URLs, unless used for path placeholders.
* Avoid using `snake_case` for query arguments

### Explanation

* `kebab-case` is easier to read and commonly adopted in HTTP APIs (e.g., GitHub, Stripe).
* This naming also avoids ambiguity in tooling or path parsing.

### JSON Field Naming

* Use **`camelCase`** for all serialized field names in public APIs.
* Apply `#[serde(rename_all = "camelCase")]` on the struct level:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateUserRequest {
    pub email: String,
    pub full_name: String,
    pub phone_number: Option<String>,
}
```

### Explanation

* This aligns with JSON naming conventions used in most web APIs.
* Avoid mixing naming conventions between backend field names and JSON.

### gRPC Protobuf Definitions

When designing `.proto` files for use with gRPC and Rust (e.g. via `tonic` + `prost`), follow idiomatic naming conventions to ensure consistency and compatibility across languages and generated clients.

#### Message and Field Naming

| Element          | Convention             | Example                   |
|------------------|------------------------|---------------------------|
| Message names    | `PascalCase`           | `CreateUserRequest`       |
| Field names      | `snake_case`           | `string full_name = 1;`   |
| Enum types       | `PascalCase`           | `UserStatus`              |
| Enum values      | `SCREAMING_SNAKE_CASE` | `USER_STATUS_ACTIVE = 1;` |
| Service names    | `PascalCase`           | `UserService`             |
| RPC method names | `PascalCase`           | `GetUser`, `CreateUser`   |

```proto
message CreateUserRequest {
  string email = 1;
  string full_name = 2;
  string phone_number = 3;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_DISABLED = 2;
}
```

#### Explanation

* `snake_case` for fields ensures consistent mapping to Rust and Python.
* `PascalCase` is used for all top-level identifiers: messages, services, enums, and RPCs.
* Enum values are conventionally written in uppercase with a type prefix (e.g., `USER_STATUS_ACTIVE`) to avoid naming conflicts.

#### Rust-Specific Notes

* The `prost` code generator will translate `snake_case` Protobuf fields to `snake_case` Rust struct fields.
* When exposing gRPC-generated structs as JSON via REST, use `#[serde(rename_all = "camelCase")]` to match external API conventions:

```rust
#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateUserResponse {
    pub user_id: String,
    pub full_name: String,
}
```

#### Compatibility with Other Languages

* Java/Kotlin gRPC clients will receive `camelCase` field accessors automatically.
* Go clients will use `PascalCase` for struct fields.
* The `.proto` definition remains the canonical schema and should use `snake_case` for field names regardless of language.

---

## 4. Newtype Domain Wrappers

Use **newtypes** to model domain concepts explicitly — especially identifiers, secrets, tokens, or business rules. This improves type safety, clarifies intent, and avoids accidental mixing of unrelated data.

### Prefer Newtypes Over Aliases for Domain Values

Avoid type aliases for values with distinct semantic meaning:

```rust
type UserId = String; // ❌ Can be confused with Email or OrderId
```

Prefer `struct` newtypes for domain clarity and compiler enforcement:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct UserId(String);
```

### Implement `Default` for Empty/Sentinel States

For newtypes over `String`, `Uuid`, or `Vec`, implement `Default` when a neutral or placeholder value makes sense:

```rust
impl Default for UserId {
    fn default() -> Self {
        Self(String::new())
    }
}
```

This allows structs using your type to derive `Default` cleanly and simplifies tests and construction in generic contexts.

### Implement `pub const fn as_ref()` Alongside `AsRef`

Define an idiomatic `.as_ref()` method with `const fn` when possible:

```rust
impl UserId {
    #[inline]
    pub const fn as_ref(&self) -> &str {
        &self.0
    }
}

impl AsRef<str> for UserId {
    fn as_ref(&self) -> &str {
        &self.0
    }
}
```

#### Explanation

* `const fn` allows your accessor to be used in `const` contexts (e.g., static routes, match arms).
* Implementing `AsRef<str>` or `AsRef<[u8]>` makes your type ergonomic in APIs expecting raw types.

This is especially useful when composing with `Path`, `Display`, `Serialize`, or custom error messages.

### Optional Trait Derives

Derive the following traits where applicable:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(transparent)]
pub struct SessionToken(String);
```

* Use `#[serde(transparent)]` for ergonomic (de)serialization.
* Implement `Display` for logging and diagnostics:

```rust
impl fmt::Display for UserId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.0)
    }
}
```

---

## 5. Ownership and Borrowing in API Design

When designing APIs, balance memory efficiency with code clarity and maintainability. Prefer working code over premature optimization, but avoid unnecessary allocations when possible.

### Prefer Owned Values When in Doubt

* **Prioritize working code over memory-efficient code** — use owned values (`String`, `Vec<T>`) when the ownership semantics are unclear or when it simplifies the API.
* Owned values eliminate lifetime complexity and make APIs easier to use correctly.

```rust
// ✅ Clear and simple - prefer this when in doubt
pub fn process_user(name: String, email: String) -> Result<User, UserError> {
    // Implementation
}

// ❌ Only use borrows if you're certain about the lifetime requirements
pub fn process_user<'a, 'b, 'c: 'a + 'b>(name: &'a str, email: &'b str) -> Result<&'c User, UserError> {
    // Implementation - now callers must manage lifetimes
}
```

### Avoid Allocations When Possible

* **Return `&'static str` for constant values** instead of `String`:

```rust
// ✅ No allocation needed
pub fn get_default_theme() -> &'static str {
    "dark"
}

// ❌ Unnecessary allocation
pub fn get_default_theme() -> String {
    "dark".to_string()
}
```

* **Use `Cow<str>` when you might return either owned or borrowed data**:

```rust
use std::borrow::Cow;

// ✅ Flexible - can return either owned or borrowed
pub fn get_display_name(user: &User) -> Cow<str> {
    if let Some(nickname) = &user.nickname {
        Cow::Borrowed(nickname)
    } else {
        Cow::Owned(format!("{} {}", user.first_name, user.last_name))
    }
}
```

### Implement APIs in Terms of Borrows Where Possible

* **Design function signatures to accept borrowed values** when the function doesn't need to own the data:

```rust
// ✅ Accepts both &str and String via deref coercion
pub fn validate_email(email: &str) -> bool {
    email.contains('@')
}

// ✅ Can be called with either:
validate_email("user@example.com");  // &str
validate_email(&user.email);         // &String -> &str
```

* **Be explicit about ownership requirements** — if your function must own the value internally (e.g., for storage, spawning tasks), make this clear in the signature:

```rust
// ✅ Clear that we need ownership - no surprise internal clones
pub fn store_user_async(name: String, email: String) -> JoinHandle<Result<(), Error>> {
    tokio::spawn(async move {
        // We need owned values for the async task
        save_to_database(name, email).await
    })
}
```

### Use Generic Bounds for Flexible APIs

* **Use `AsRef<T>` or `Borrow<T>` to accept both owned and borrowed values**:

```rust
use std::borrow::Borrow;

// ✅ Accepts both String and &str
pub fn process_message<S>(message: S) -> String
where
    S: AsRef<str>,
{
    let msg = message.as_ref();
    format!("Processed: {}", msg)
}

// ✅ Can be called with either:
process_message("hello");           // &str
process_message(String::from("hello")); // String

// ✅ For custom types, use Borrow<T>
pub fn find_user<E>(email: E) -> Option<User>
where
    E: Borrow<Email>,
{
    let email_ref = email.borrow();
    // Implementation
}
```

* **This pattern allows callers maximum flexibility** while keeping your implementation simple.

### Prefer `where` Clause Syntax for Generic Code

* **Use `where` clause syntax instead of inline trait bounds** for better readability and consistency:

```rust
// ✅ Preferred - use where clause
pub fn process_message<S>(message: S) -> String
where
    S: AsRef<str>,
{
    let msg = message.as_ref();
    format!("Processed: {}", msg)
}

// ❌ Avoid inline bounds for generic code
pub fn process_message<S: AsRef<str>>(message: S) -> String {
    let msg = message.as_ref();
    format!("Processed: {}", msg)
}
```

* **`where` clauses are especially beneficial with multiple or complex bounds**:

```rust
// ✅ Clear and readable with where clause
pub fn serialize_and_validate<T, E>(data: T) -> Result<String, E>
where
    T: Serialize + Clone + Debug,
    E: From<serde_json::Error> + From<ValidationError>,
{
    // Implementation
}

// ❌ Hard to read with inline bounds
pub fn serialize_and_validate<T: Serialize + Clone + Debug, E: From<serde_json::Error> + From<ValidationError>>(data: T) -> Result<String, E> {
    // Implementation
}
```

* **For simple single bounds, `where` clause is still preferred for consistency**:

```rust
// ✅ Consistent style
pub fn find_user<E>(email: E) -> Option<User>
where
    E: Borrow<Email>,
{
    let email_ref = email.borrow();
    // Implementation
}
```

### Guidelines Summary

1. **Default to owned values** (`String`, `Vec<T>`) when designing new APIs
2. **Avoid allocations** when you can return `&'static str` or use `Cow<T>`
3. **Accept borrows in function parameters** unless you need ownership
4. **Be explicit about ownership needs** — no surprise internal clones
5. **Use generic bounds** (`AsRef<T>`, `Borrow<T>`) for maximum API flexibility

---

# G. Recommended Tools & Libraries

## 1. Crate Recommendations

| Purpose            | Crate                                          |
|--------------------|------------------------------------------------|
| Async runtime      | `tokio`                                        |
| Errors             | `thiserror`, `eyre`                            |
| Serialization      | `serde`                                        |
| HTTP server/client | `axum`, `reqwest`                              |
| Validation         | `validator`                                    |
| Logging/Tracing    | `tracing`, `tracing-subscriber`                |
| Database           | `sqlx`, `diesel`                               |
| Tests + Fixtures   | `insta`, `assert_cmd`, `test-case`, `proptest` |
