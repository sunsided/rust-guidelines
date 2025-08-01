# Rust Coding Guidelines

These are human-friendly and AI-compatible guidelines for building robust, maintainable, and idiomatic Rust applications, especially in async and service-oriented contexts.

---

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

## 3. Prefer `thiserror` and Specific Error Types

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

## 4. Use Tokio Runtime for Async Work

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

## 5. Organize Code Logically (Prefer Feature-Based Modules)

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

## 6. Handle Errors Explicitly, Avoid `unwrap()`

* Convert errors as early as possible into your domain types or propagate via `?`.
* Use `expect()` in production code only if you can guarantee that it will not fail.
* Never use `unwrap()` in production code.
* In unit tests, prefer `expect` over `unwrap`, or use the `assert!` macros.

---

## 7. Validate Input Early and Clearly

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

## 8. Avoid Global State, Prefer Explicit Dependency Injection

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

## 9. Write Tests: Unit, Integration, and Property-Based

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

## 10. Use Logging and Tracing

* Prefer `tracing` over `log`:

```rust
use tracing::{info, error};

info!(user_id = %uid, "User created successfully");
```

* Use `tracing_subscriber` for layered structured logging.
* Pass context (`trace_id`, `request_id`) through spans for better observability.

---

## 11. Secure by Default

* Always validate input and sanitize outputs
* Avoid leaking internal errors via HTTP responses—wrap in proper error responses
* If exposing public APIs, document and version them.

---

## 12. Configuration and Secrets

* Use the `config` crate (or `figment`, `envy`, etc.) to load layered config:

```rust
#[derive(Deserialize)]
struct Settings {
    port: u16,
    database_url: String,
}
```

* Load from environment variables, files, or CLI args. Never commit secrets.
* Use [`dotenvy`] in your application code to load `.env` files.
* Add `.env` files as a `.gitignore` rule.

[dotenvy]: https://github.com/allan2/dotenvy

---

## 13. Versioning, Linting, Formatting

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

## 14. Prefer Compile-Time Over Runtime Safety

* Use `Option` and `Result` for error signaling.
* Prefer `enum` variants for domain logic over stringly-typed logic.
* Minimize use of `unsafe`; encapsulate it behind safe APIs if needed.
* Consider using `#![forbid(unsafe_code)]` on the `lib.rs` or `main.rs`. See [Rust Safety Dance].

[Rust Safety Dance]: (https://github.com/rust-secure-code/safety-dance)

---

## 15. Avoid Overengineering Too Early

* Don't introduce traits just to “test easily”. Stick to concrete types until abstraction is justified.
* Don’t prematurely build plugin systems, service locators, or fancy macro DSLs.

---

## 16. Crate Recommendations

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

---

## 17. Feature Flags and CI

* Use conditional compilation for platform or feature-specific code (`#[cfg(...)]`)
* Use `cargo nextest` for faster test runs in CI
* Structure CI to include: `cargo check`, `cargo clippy`, `cargo fmt`, `cargo test`

---

## 18. Graceful Shutdown

* On servers, always support `SIGTERM` / `SIGINT` for graceful shutdown:

```rust
tokio::signal::ctrl_c().await?;
```

* Cancel long-running tasks via `tokio::select!` and `CancellationToken`

---

## 19. Workspace Management

* Use workspace inheritance (`[workspace.dependencies]`) to avoid version mismatches.
* Keep top-level `Cargo.lock` for apps (omit for libraries).

---

## 20. Documentation and Examples

* Write `//!` crate-level docs and `///` API docs.
* Include examples in `examples/` or inline doc tests with `/// ```rust`.

---

## 21. Naming Conventions

### General Naming Rules

| Entity              | Convention                                   | Example                            |
| ------------------- | -------------------------------------------- | ---------------------------------- |
| Modules             | `snake_case`                                 | `mod user_service;`                |
| Structs & Enums     | `PascalCase`                                 | `struct CreateUserInput`           |
| Traits              | `PascalCase`, usually a capability or action | `trait Serialize`, `trait UseCase` |
| Functions & Methods | `snake_case`                                 | `fn process_user()`                |
| Constants           | `SCREAMING_SNAKE_CASE`                       | `const MAX_RETRIES: u32 = 5;`      |
| Type Aliases        | `PascalCase`                                 | `type UserId = Uuid;`              |
| Generic Parameters  | `T`, `E`, `Item` or domain-meaningful        | `fn parse<T: Deserialize>()`       |
| Feature Flags       | `snake_case`                                 | `#[cfg(feature = "tls")]`          |

##### Explanation

* These follow standard Rust community idioms.
* Avoid acronyms in all caps unless conventional (e.g., `HttpClient`, not `HTTPClient`).
* Avoid Java-style naming (`get_`, `set_`) unless it improves clarity — prefer `fn name()` over `fn get_name()`.

---

### Test Function Naming

**Avoid test function names that redundantly contain `test_`.**
The attribute `#[test]` already marks the function as a test.

##### Not recommended

```rust
#[test]
fn test_create_user() { ... }

#[test]
fn test1() { ... }
```

##### Recommended

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

##### Explanation

* This naming pattern makes the test suite read like a specification.
* It's especially helpful for developers coming from Python (`unittest`, `pytest`) or Java/Kotlin (JUnit/Spock), where test method names often follow `testMethodName` or `given_when_then` naming.
* Avoid overly verbose names. Be specific, but concise.

#### Optional: Parametrized Test Names

* When using crates like [`test-case`](https://docs.rs/test-case), provide descriptive labels:

```rust
#[test_case("a@b.com", true; "valid email")]
#[test_case("invalid@", false; "invalid email")]
fn validates_email_format(input: &str, expected: bool) {
    assert_eq!(validate_email(input), expected);
}
```

### Public API Naming

When designing HTTP APIs or exposing serialized structs to external clients (e.g., via JSON or gRPC), follow consistent naming for compatibility and readability.

#### HTTP Route Naming

* Use **`kebab-case`** for URL path segments:

```
GET    /api/v1/users
POST   /api/v1/create-user
GET    /api/v1/users/{user_id}/reset-password
```

* Avoid `snake_case` in URLs, unless used for path placeholders.
* Avoid using `snake_case` for query arguments

##### Explanation

* `kebab-case` is easier to read and commonly adopted in HTTP APIs (e.g., GitHub, Stripe).
* This naming also avoids ambiguity in tooling or path parsing.


#### JSON Field Naming

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

##### Explanation

* This aligns with JSON naming conventions used in most web APIs.
* Avoid mixing naming conventions between backend field names and JSON.

---

#### gRPC Protobuf Definitions

When designing `.proto` files for use with gRPC and Rust (e.g. via `tonic` + `prost`), follow idiomatic naming conventions to ensure consistency and compatibility across languages and generated clients.

##### Message and Field Naming

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

##### Explanation

* `snake_case` for fields ensures consistent mapping to Rust and Python.
* `PascalCase` is used for all top-level identifiers: messages, services, enums, and RPCs.
* Enum values are conventionally written in uppercase with a type prefix (e.g., `USER_STATUS_ACTIVE`) to avoid naming conflicts.

##### Rust-Specific Notes

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

Here is a new section for your **Rust Coding Guidelines** document, continuing the established structure and voice:

---

## 23. Newtype Domain Wrappers

Use **newtypes** to model domain concepts explicitly — especially identifiers, secrets, tokens, or business rules. This improves type safety, clarifies intent, and avoids accidental mixing of unrelated data.

---

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

---

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

---

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

##### Explanation

* `const fn` allows your accessor to be used in `const` contexts (e.g., static routes, match arms).
* Implementing `AsRef<str>` or `AsRef<[u8]>` makes your type ergonomic in APIs expecting raw types.

This is especially useful when composing with `Path`, `Display`, `Serialize`, or custom error messages.

---

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
