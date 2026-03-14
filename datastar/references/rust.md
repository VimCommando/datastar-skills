# Rust Backend Reference

Use this file for Datastar + Rust backend implementation details.

## Table of Contents

- [Rust Backend Patterns](#rust-backend-patterns)
- [Common Patterns & Best Practices](#common-patterns--best-practices)
- [Debugging](#debugging)
- [Common Gotchas](#common-gotchas)
- [Advanced Patterns](#advanced-patterns)
- [Reference Links](#reference-links)
- [Version Info](#version-info)

## Rust Backend Patterns

### SDK Installation

```bash
cargo add datastar
cargo add axum tokio --features tokio/full
cargo add serde serde_json --features serde/derive
```

### Basic Handler Pattern

```rust
use axum::{extract::State, response::IntoResponse};
use datastar::{
    patch_elements::PatchElements,
    patch_signals::PatchSignals,
    request::ReadSignals,
};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug, Default)]
struct AppState {
    count: usize,
}

type Shared = Arc<Mutex<AppState>>;

#[derive(Debug, Deserialize)]
struct CounterSignals {
    #[serde(default)]
    count: usize,
    #[serde(default)]
    name: String,
}

#[derive(Debug, Serialize)]
struct CounterResponse {
    count: usize,
    is_valid: bool,
}

async fn update_handler(
    State(state): State<Shared>,
    ReadSignals(signals): ReadSignals<CounterSignals>,
) -> impl IntoResponse {
    // 1. Read client signals
    let mut store = state.lock().await;

    // 2. Process logic
    store.count = signals.count + 1;
    let payload = CounterResponse {
        count: store.count,
        is_valid: !signals.name.trim().is_empty(),
    };

    // 3. Return one or more Datastar events
    (
        PatchSignals::new(json!(payload)),
        PatchElements::new(format!(
            r#"<div id=\"counter\">Count: {}</div>"#,
            store.count
        )),
    )
}
```

### Reading Client Signals

`ReadSignals<T>` is the extractor for Datastar signal payloads.

```rust
use axum::response::IntoResponse;
use datastar::request::ReadSignals;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct FormSignals {
    #[serde(default)]
    email: String,
    #[serde(default)]
    password: String,
    #[serde(default)]
    remember: bool,
}

async fn login(ReadSignals(form): ReadSignals<FormSignals>) -> impl IntoResponse {
    if form.email.is_empty() || form.password.is_empty() {
        return datastar::patch_elements::PatchElements::new(
            r#"<p id=\"error\">Email and password are required.</p>"#,
        );
    }

    datastar::redirect::Redirect::new("/dashboard")
}
```

### SSE Event Methods

Rust SDK event types implement `DatastarEvent` and can be returned directly from handlers.

```rust
use datastar::{
    execute_script::ExecuteScript,
    patch_elements::{MergeMode, PatchElements},
    patch_signals::PatchSignals,
    redirect::Redirect,
    remove_element::RemoveElement,
};
use serde_json::json;

// DOM updates
let patch = PatchElements::new(r#"<div id=\"result\">Updated</div>"#);
let append = PatchElements::new(r#"<li>New Item</li>"#)
    .with_selector("#item-list")
    .with_mode(MergeMode::Append);

// Remove element
let remove = RemoveElement::new("#temporary-message");

// Signal updates
let signals = PatchSignals::new(json!({
    "count": 42,
    "message": "Hello from Rust",
    "user": {
        "name": "John",
        "email": "john@example.com"
    }
}));
let defaults = PatchSignals::new(json!({"defaultTheme": "dark"}))
    .with_only_if_missing(true);

// Script execution
let script = ExecuteScript::new("console.log('Server says hello!')");
let once = ExecuteScript::new("window.initChart?.()")
    .with_auto_remove(true)
    .with_attributes(r#"{ "type": "module" }"#);

// Navigation
let redirect = Redirect::new("/dashboard");
```

Supported `MergeMode` values:
`Morph` (default), `Inner`, `Outer`, `Prepend`, `Append`, `Before`, `After`, `Remove`.

### SSE Event Types (Low-level)

The SDK maps high-level Rust event structs to these Datastar events:

- `datastar-patch-elements`
- `datastar-patch-signals`
- `datastar-remove-element`
- `datastar-execute-script`
- `datastar-redirect`

### Integration with Axum

```rust
use axum::{extract::State, routing::post, Router};
use datastar::{patch_elements::PatchElements, request::ReadSignals};
use serde::Deserialize;
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug, Default)]
struct Store {
    count: usize,
}

type Shared = Arc<Mutex<Store>>;

#[derive(Debug, Deserialize)]
struct CounterSignals {
    #[serde(default)]
    count: usize,
}

async fn increment(
    State(state): State<Shared>,
    ReadSignals(signals): ReadSignals<CounterSignals>,
) -> PatchElements {
    let mut s = state.lock().await;
    s.count = signals.count + 1;
    PatchElements::new(format!(
        r#"<p id=\"counter\">Current count: {}</p>"#,
        s.count
    ))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/counter/increment", post(increment))
        .with_state(Arc::new(Mutex::new(Store::default())));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Integration with Actix-web

Use `DatastarEvent::write_as_actix_web_sse_event` when manually writing SSE frames in custom Actix responses.

```rust
use datastar::{patch_elements::PatchElements, DatastarEvent};

let event = PatchElements::new(r#"<div id=\"status\">OK</div>"#);
let frame = event.write_as_actix_web_sse_event().expect("sse frame");
```

### Integration with Rocket

Use `DatastarEvent::write_as_rocket_sse_event` when composing Rocket SSE streams.

```rust
use datastar::{patch_signals::PatchSignals, DatastarEvent};
use serde_json::json;

let event = PatchSignals::new(json!({"ready": true}));
let frame = event.write_as_rocket_sse_event().expect("sse frame");
```

### Compression Support

SSE can be compressed by your HTTP stack or reverse proxy, but keep these constraints:

- Do not buffer the stream (disable proxy buffering for SSE endpoints).
- Keep `text/event-stream` semantics intact.
- Send heartbeat events for long-lived streams where infra times out idle connections.

### Multi-Step SSE Response

Return multiple events in one handler to update state and UI together.

```rust
use datastar::{patch_elements::PatchElements, patch_signals::PatchSignals};
use serde_json::json;

async fn submit_ok() -> impl axum::response::IntoResponse {
    (
        PatchSignals::new(json!({
            "loading": false,
            "saved": true,
            "message": "Saved successfully"
        })),
        PatchElements::new(
            r#"<div id=\"result\" class=\"success\">Saved successfully!</div>"#,
        ),
    )
}
```

### Error Handling

```rust
use axum::response::IntoResponse;
use datastar::{patch_elements::PatchElements, patch_signals::PatchSignals};
use serde_json::json;

async fn submit_with_validation(email: &str) -> impl IntoResponse {
    if email.trim().is_empty() {
        return (
            PatchSignals::new(json!({
                "loading": false,
                "hasError": true,
                "errorMessage": "Email is required"
            })),
            PatchElements::new(
                r#"<div id=\"form-error\" class=\"error\">Email is required</div>"#,
            ),
        );
    }

    (
        PatchSignals::new(json!({"loading": false, "hasError": false})),
        PatchElements::new(r#"<div id=\"form-error\"></div>"#),
    )
}
```

### Server Setup

Minimal Axum setup for Datastar actions:

```rust
use axum::{routing::{get, post}, Router};

async fn page() -> &'static str {
    r#"<!doctype html>
<html>
  <head>
    <script type=\"module\" src=\"https://cdn.jsdelivr.net/gh/starfederation/datastar@v1.0.0-RC.6/bundles/datastar.js\"></script>
  </head>
  <body>
    <p id=\"counter\">Current count: 0</p>
    <button data-on-click=\"@$count++\" data-on-click__post=\"/counter/increment\">Increment</button>
  </body>
</html>"#
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(page))
        .route("/counter/increment", post(|| async {
            datastar::patch_elements::PatchElements::new(
                r#"<p id=\"counter\">Current count: 1</p>"#,
            )
        }));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## Common Patterns & Best Practices

### Form Submission

```rust
use axum::response::IntoResponse;
use datastar::{patch_elements::PatchElements, patch_signals::PatchSignals, request::ReadSignals};
use serde::Deserialize;
use serde_json::json;

#[derive(Debug, Deserialize)]
struct ContactSignals {
    #[serde(default)]
    name: String,
    #[serde(default)]
    email: String,
    #[serde(default)]
    message: String,
}

async fn submit_contact(ReadSignals(form): ReadSignals<ContactSignals>) -> impl IntoResponse {
    if form.name.is_empty() || form.email.is_empty() || form.message.is_empty() {
        return (
            PatchSignals::new(json!({"loading": false})),
            PatchElements::new(
                r#"<div id=\"contact-status\" class=\"error\">All fields are required.</div>"#,
            ),
        );
    }

    (
        PatchSignals::new(json!({"loading": false, "form": {"name": "", "email": "", "message": ""}})),
        PatchElements::new(
            r#"<div id=\"contact-status\" class=\"success\">Message sent.</div>"#,
        ),
    )
}
```

### Live Search with Debouncing

```rust
use datastar::{patch_elements::PatchElements, request::ReadSignals};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct SearchSignals {
    #[serde(default)]
    query: String,
}

async fn search(ReadSignals(signals): ReadSignals<SearchSignals>) -> PatchElements {
    if signals.query.trim().len() < 2 {
        return PatchElements::new(r#"<ul id=\"search-results\"></ul>"#);
    }

    let items = ["apple", "apricot", "banana", "blueberry"]
        .into_iter()
        .filter(|v| v.contains(&signals.query.to_lowercase()))
        .map(|v| format!("<li>{v}</li>"))
        .collect::<String>();

    PatchElements::new(format!(r#"<ul id=\"search-results\">{items}</ul>"#))
}
```

### Infinite Scroll / Load More

```rust
use datastar::{patch_elements::{MergeMode, PatchElements}, request::ReadSignals};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct FeedSignals {
    #[serde(default)]
    page: usize,
}

async fn load_more(ReadSignals(signals): ReadSignals<FeedSignals>) -> PatchElements {
    let next_page = signals.page + 1;
    let html = format!(
        r#"<li>Item {}</li><li>Item {}</li><button id=\"load-more\" data-signals:page=\"{}\" data-on-click__post=\"/feed/load-more\">Load More</button>"#,
        next_page * 2 - 1,
        next_page * 2,
        next_page
    );

    PatchElements::new(html)
        .with_selector("#feed")
        .with_mode(MergeMode::Append)
}
```

### Signal Naming & Organization

- Keep client signal keys in `camelCase`.
- Mirror client structure with nested Rust structs where practical.
- Use `#[serde(default)]` to tolerate missing optional signals.
- Use `Option<T>` when you need to distinguish missing vs empty.

### Loading Indicators

```rust
use datastar::patch_signals::PatchSignals;
use serde_json::json;

async fn start_loading() -> PatchSignals {
    PatchSignals::new(json!({"loading": true}))
}

async fn stop_loading() -> PatchSignals {
    PatchSignals::new(json!({"loading": false}))
}
```

### Optimistic Updates

For optimistic UI:

1. Update local client signal immediately (frontend action).
2. Post to backend.
3. Reconcile with backend truth using `PatchSignals` + `PatchElements`.
4. Return rollback events on failure.

```rust
use datastar::{patch_elements::PatchElements, patch_signals::PatchSignals};
use serde_json::json;

async fn rollback_like() -> impl axum::response::IntoResponse {
    (
        PatchSignals::new(json!({"post": {"liked": false, "likesCount": 10}})),
        PatchElements::new(r#"<span id=\"likes-count\">10</span>"#),
    )
}
```

### Error Handling Pattern

Use one dedicated error area and structured error signals:

```rust
use datastar::{patch_elements::PatchElements, patch_signals::PatchSignals};
use serde_json::json;

fn error_response(msg: &str) -> (PatchSignals, PatchElements) {
    (
        PatchSignals::new(json!({
            "loading": false,
            "hasError": true,
            "errorMessage": msg
        })),
        PatchElements::new(format!(
            r#"<div id=\"global-error\" role=\"alert\" class=\"error\">{}</div>"#,
            msg
        )),
    )
}
```

### Keep Expressions Simple

Keep `data-*` expressions small and move logic to Rust handlers. Prefer:

- Frontend: trigger events, set temporary UI-only state.
- Backend: validation, authorization, business rules, and final state.

## Debugging

### Using `data-json-signals`

Add temporary debug output in templates:

```html
<pre data-text="$signals | JSON.stringify(_, null, 2)"></pre>
```

Or use Datastar's built-in helper where available:

```html
<pre data-json-signals></pre>
```

### Datastar Inspector

Use the Datastar browser inspector/extension to inspect:

- current signal values
- event stream activity
- DOM patch operations

### Console Debugging

Use `ExecuteScript` for temporary diagnostics:

```rust
use datastar::execute_script::ExecuteScript;

let debug = ExecuteScript::new("console.debug('signals changed')")
    .with_auto_remove(true);
```

## Common Gotchas

### 1. Signal Casing

Rust struct fields are usually `snake_case`, while Datastar signals are often `camelCase`.
Use serde rename attributes when needed.

```rust
#[derive(serde::Deserialize)]
struct Example {
    #[serde(rename = "isActive", default)]
    is_active: bool,
}
```

### 2. Actions Require `@` Prefix

Datastar action attributes must use `@` where required by syntax; plain expressions without action prefix do not trigger backend actions.

### 3. Multi-statement Expressions Need Semicolons

When using multiple statements in a Datastar expression, separate them with semicolons.

### 4. Element IDs for Morphing

`PatchElements` default mode is `Morph`, which relies on stable IDs for predictable replacement.

### 5. SSE Response Headers

Do not proxy/cache SSE as normal HTTP responses. Ensure SSE endpoints preserve streaming behavior.

### 6. Signal Types and Binding

Type drift happens when frontend bindings and backend structs disagree. Keep a single canonical Rust signal struct per feature.

### 7. File Upload Base64 Encoding

Datastar file signals may include Data URLs/base64 payloads. Validate size and content type server-side before decoding.

### 8. Underscore-prefixed Signals

Underscore-prefixed client signals are local-only by convention and may not be sent to the backend.

### 9. Request Cancellation

Rapid interactions can cancel in-flight requests. Handlers should be idempotent and tolerant of stale submissions.

### 10. Empty vs Missing Signals

Use `Option<T>` in `ReadSignals` structs when missing vs empty matters.

## Advanced Patterns

### Polling with Auto-retry

- Frontend schedules polling action.
- Backend returns partial DOM/signal updates.
- Add heartbeat or status signals for stale-connection detection.

### View Transitions

Coordinate transitions by sending container-level `PatchElements` updates with stable IDs and minimal subtree churn.

### Modal Dialog Pattern

- Keep modal open/closed in signal state.
- Return both signal updates and modal body patches in one response.
- Use server validation to preserve error state in modal flows.

### Tabs Pattern

- Keep active tab as signal (for example `activeTab`).
- Patch only the tab panel region on tab changes.
- Keep tab button markup stable to reduce morph churn.

### Toast Notifications

Use append patches into a toast container and remove later by selector:

```rust
use datastar::{patch_elements::{MergeMode, PatchElements}, remove_element::RemoveElement};

let show = PatchElements::new(
    r#"<div id=\"toast-123\" class=\"toast\">Saved!</div>"#,
)
.with_selector("#toasts")
.with_mode(MergeMode::Append);

let hide = RemoveElement::new("#toast-123");
```

## Reference Links

- [Datastar Rust SDK repo](https://github.com/starfederation/datastar-rust)
- [Datastar Rust docs](https://docs.rs/datastar/latest/datastar/)
- [Datastar website/docs](https://data-star.dev)

## Version Info

- This reference targets the `datastar` Rust crate API documented at docs.rs.
