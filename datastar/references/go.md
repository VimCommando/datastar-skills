# Go Backend Reference

Use this file for all Datastar + Go backend implementation details.

## Table of Contents

- [Go Backend Patterns](#go-backend-patterns)
- [Common Patterns & Best Practices](#common-patterns--best-practices)
- [Debugging](#debugging)
- [Common Gotchas](#common-gotchas)
- [Advanced Patterns](#advanced-patterns)
- [Pro Features](#pro-features)
- [Reference Links](#reference-links)
- [Version Info](#version-info)

## Go Backend Patterns

### SDK Installation

```bash
go get github.com/starfederation/datastar-go
```

**Requires Go 1.24+**

### Basic Handler Pattern

```go
package handlers

import (
    "net/http"
    "github.com/starfederation/datastar-go/datastar"
)

// Define your application state struct
type MyStore struct {
    Count   int    `json:"count"`
    Name    string `json:"name"`
    IsValid bool   `json:"isValid"`
}

func UpdateHandler(w http.ResponseWriter, r *http.Request) {
    // 1. Read client signals into a struct
    store := &MyStore{}
    if err := datastar.ReadSignals(r, store); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // 2. Process logic
    store.Count++
    store.IsValid = store.Name != ""

    // 3. Create SSE writer (handles headers automatically)
    sse := datastar.NewSSE(w, r)

    // 4. Send updates back to the client

    // Option A: Update signals (state only)
    datastar.MarshalAndPatchSignals(sse, store)

    // Option B: Update DOM (HTML fragment only)
    html := fmt.Sprintf(`<div id="counter">Count: %d</div>`, store.Count)
    datastar.PatchElements(sse, html)

    // Option C: Both signals and DOM
    datastar.MarshalAndPatchSignals(sse, map[string]any{"loading": false})
    datastar.PatchElements(sse, `<div id="result">Done!</div>`)
}
```

### Reading Client Signals

```go
// ReadSignals extracts signals from the request and unmarshals into a struct
func MyHandler(w http.ResponseWriter, r *http.Request) {
    type FormData struct {
        Email    string `json:"email"`
        Password string `json:"password"`
        Remember bool   `json:"remember"`
    }

    form := &FormData{}
    if err := datastar.ReadSignals(r, form); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    // form now contains the client's signal values
    // GET requests: signals come from ?datastar=... query param
    // POST/PUT/PATCH/DELETE: signals come from JSON request body
}
```

### SSE Event Methods

The backend responds by streaming Server-Sent Events (SSE). Multiple events can be sent in a single response.

```go
sse := datastar.NewSSE(w, r)

// ===== DOM Updates =====

// 1. PatchElements - Update/replace DOM elements
// By default, uses morphing to update element with matching ID
datastar.PatchElements(sse, `<div id="result">Updated content</div>`)

// With custom selector (CSS selector for target)
datastar.PatchElements(sse, `<li>New Item</li>`,
    datastar.WithSelector("#item-list"))

// With merge mode (how to insert the HTML)
datastar.PatchElements(sse, `<li>New Item</li>`,
    datastar.WithSelector("#list"),
    datastar.WithMode("append"))

// Available modes:
// - "morph" (default): Intelligently updates the element
// - "inner": Replaces innerHTML (like .innerHTML = ...)
// - "outer": Replaces entire element (like .outerHTML = ...)
// - "prepend": Inserts at beginning of children
// - "append": Inserts at end of children
// - "before": Inserts before the element
// - "after": Inserts after the element
// - "replace": Replaces the element entirely
// - "remove": Removes the element

// Combining selector and mode
datastar.PatchElements(sse, `<div class="alert">Warning!</div>`,
    datastar.WithSelector("body"),
    datastar.WithMode("prepend"))

// 2. RemoveElement - Remove an element by selector
datastar.RemoveElement(sse, "#temporary-message")
datastar.RemoveElement(sse, ".toast-notification")

// ===== Signal Updates =====

// 3. MarshalAndPatchSignals - Update signals from struct/map
datastar.MarshalAndPatchSignals(sse, map[string]any{
    "count": 42,
    "message": "Hello from Go!",
    "user": map[string]any{
        "name": "John",
        "email": "john@example.com",
    },
})

// Update only if signals don't exist on client
datastar.MarshalAndPatchSignals(sse,
    map[string]any{"defaultTheme": "dark"},
    datastar.WithOnlyIfMissing(true))

// 4. PatchSignals - Send raw JSON bytes for signals
jsonBytes := []byte(`{"count": 42, "message": "Hello"}`)
datastar.PatchSignals(sse, jsonBytes)

// ===== Script Execution =====

// 5. ExecuteScript - Run JavaScript on the client
datastar.ExecuteScript(sse, `console.log('Server says hello!')`)
datastar.ExecuteScript(sse, `window.scrollTo({top: 0, behavior: 'smooth'})`)

// ===== Navigation =====

// 6. Redirect - Navigate to a new page
datastar.Redirect(sse, "/dashboard")
datastar.Redirect(sse, "/login?redirect=/dashboard")
```

### SSE Event Types (Low-level)

The Go SDK provides high-level methods above, but under the hood these SSE event types are sent:

```go
// Event: datastar-patch-elements
// Patches DOM elements with HTML content
// Options: selector (CSS selector), mode (merge strategy)

// Event: datastar-patch-signals
// Updates signal values
// Options: onlyIfMissing (only set if signal doesn't exist)

// Event: datastar-remove-element
// Removes an element by selector

// Event: datastar-execute-script
// Executes JavaScript code

// Event: datastar-redirect
// Navigates to a new URL
```

### Integration with Templ

```go
import (
    "github.com/a-h/templ"
    "github.com/starfederation/datastar-go/datastar"
)

func TemplHandler(w http.ResponseWriter, r *http.Request) {
    sse := datastar.NewSSE(w, r)

    // Render a Templ component
    component := myTemplComponent("data")

    // PatchElementTempl renders the Templ component and patches DOM
    datastar.PatchElementTempl(sse, component,
        datastar.WithSelector("#target"))
}
```

### Integration with GoStar

```go
import (
    "github.com/delaneyj/gostar"
    "github.com/starfederation/datastar-go/datastar"
)

func GoStarHandler(w http.ResponseWriter, r *http.Request) {
    sse := datastar.NewSSE(w, r)

    // Create GoStar element
    element := gostar.Div(
        gostar.ID("result"),
        gostar.Text("Hello from GoStar"),
    )

    // PatchElementGostar renders GoStar element and patches DOM
    datastar.PatchElementGostar(sse, element)
}
```

### Compression Support

```go
import "github.com/starfederation/datastar-go/datastar"

func CompressedHandler(w http.ResponseWriter, r *http.Request) {
    // NewSSE automatically handles compression based on Accept-Encoding header
    sse := datastar.NewSSE(w, r)

    // Supports:
    // - gzip (WithGzip option)
    // - brotli (WithBrotli option)
    // - zstd (WithZstd option)

    // Compression is negotiated automatically
    datastar.PatchElements(sse, `<div id="large-content">...</div>`)
}
```

### Multi-Step SSE Response

Stream multiple UI updates in a single request to create responsive, multi-step flows.

```go
func ComplexHandler(w http.ResponseWriter, r *http.Request) {
    sse := datastar.NewSSE(w, r)

    // Step 1: Show loading state
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "loading": true,
        "status": "Processing...",
    })

    // Step 2: Simulate work
    time.Sleep(1 * time.Second)
    result := performOperation()

    // Step 3: Update UI with intermediate result
    datastar.PatchElements(sse,
        fmt.Sprintf(`<div id="progress">50%% complete</div>`))

    // Step 4: More work
    time.Sleep(1 * time.Second)
    finalResult := finalize(result)

    // Step 5: Update with final result
    datastar.PatchElements(sse,
        fmt.Sprintf(`<div id="result-area">%s</div>`, finalResult))

    // Step 6: Hide loading state
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "loading": false,
        "status": "Complete!",
    })
}
```

### Error Handling

```go
func SafeHandler(w http.ResponseWriter, r *http.Request) {
    store := &MyStore{}
    if err := datastar.ReadSignals(r, store); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    sse := datastar.NewSSE(w, r)

    // Validate input
    if store.Email == "" {
        datastar.MarshalAndPatchSignals(sse, map[string]any{
            "error": "Email is required",
            "errorField": "email",
        })
        return
    }

    // Process...
    if err := processData(store); err != nil {
        // Send error back to client
        datastar.MarshalAndPatchSignals(sse, map[string]any{
            "error": err.Error(),
        })
        return
    }

    // Success
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "error": "",
        "success": "Data saved successfully",
    })
}
```

### Server Setup

```go
package main

import (
    "log"
    "net/http"
    "path/to/your/handlers"
)

func main() {
    mux := http.NewServeMux()

    // Serve static files (Datastar JS, CSS, images)
    mux.Handle("/static/", http.StripPrefix("/static/",
        http.FileServer(http.Dir("./static"))))

    // Page handlers (return full HTML documents)
    mux.HandleFunc("/", handlers.HomePage)
    mux.HandleFunc("/about", handlers.AboutPage)

    // SSE handlers (return event streams)
    // Use HTTP method routing (Go 1.22+)
    mux.HandleFunc("POST /api/submit", handlers.SubmitForm)
    mux.HandleFunc("GET /api/search", handlers.LiveSearch)
    mux.HandleFunc("PUT /api/update", handlers.UpdateItem)
    mux.HandleFunc("DELETE /api/delete", handlers.DeleteItem)
    mux.HandleFunc("PATCH /api/partial", handlers.PartialUpdate)

    log.Println("Server starting on :8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatalf("Server failed: %v", err)
    }
}
```

---

## Common Patterns & Best Practices

### Form Submission

Use `data-on:submit__prevent` to intercept form submission. Use `contentType: 'form'` for standard form encoding, or omit it to send signals as JSON.

```html
<!-- Standard form submission (no signals, uses FormData) -->
<form data-on:submit__prevent="@post('/save', {contentType: 'form'})"
      data-signals:error="''"
      data-signals:success="''">
    <input name="name" required>
    <input name="email" type="email" required>
    <button type="submit">Save</button>

    <div data-show="$error" data-text="$error" class="error"></div>
    <div data-show="$success" data-text="$success" class="success"></div>
</form>

<!-- Signal-based form (sends signals as JSON) -->
<form data-on:submit__prevent="@post('/save')"
      data-signals:form.name="''"
      data-signals:form.email="''">
    <input data-bind:form.name required>
    <input data-bind:form.email type="email" required>
    <button type="submit">Save</button>
</form>
```

Backend handler for standard form:

```go
func SaveFormHandler(w http.ResponseWriter, r *http.Request) {
    if err := r.ParseForm(); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    name := r.FormValue("name")
    email := r.FormValue("email")

    sse := datastar.NewSSE(w, r)

    // Validate
    if name == "" {
        datastar.MarshalAndPatchSignals(sse, map[string]any{
            "error": "Name is required",
        })
        return
    }

    // Save...

    // Success
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "error": "",
        "success": "Saved successfully",
    })
}
```

Backend handler for signal-based form:

```go
func SaveSignalsHandler(w http.ResponseWriter, r *http.Request) {
    type FormData struct {
        Form struct {
            Name  string `json:"name"`
            Email string `json:"email"`
        } `json:"form"`
    }

    data := &FormData{}
    if err := datastar.ReadSignals(r, data); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    sse := datastar.NewSSE(w, r)

    // Validate and save...

    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "success": "Saved successfully",
    })
}
```

### Live Search with Debouncing

```html
<div data-signals:query="''" data-signals:results="[]">
    <input type="search"
           data-bind:query
           data-on:input__debounce.300ms="@get('/api/search')"
           placeholder="Search...">

    <div id="results-container">
        <!-- Server-rendered results will be patched here -->
        <template data-show="$results.length === 0">
            <p>No results found</p>
        </template>
    </div>
</div>
```

Backend:

```go
func SearchHandler(w http.ResponseWriter, r *http.Request) {
    type Store struct {
        Query string `json:"query"`
    }

    store := &Store{}
    datastar.ReadSignals(r, store)

    results := performSearch(store.Query)

    sse := datastar.NewSSE(w, r)

    // Update results signal
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "results": results,
    })

    // Render HTML results
    html := renderSearchResults(results)
    datastar.PatchElements(sse, html,
        datastar.WithSelector("#results-container"),
        datastar.WithMode("inner"))
}
```

### Infinite Scroll / Load More

```html
<div data-signals:page="1" data-signals:hasMore="true">
    <div id="items-container">
        <!-- Items rendered here -->
    </div>

    <div data-on-intersect__once="@get('/api/load-more')"
         data-show="$hasMore">
        <div class="loading">Loading more...</div>
    </div>
</div>
```

Backend:

```go
func LoadMoreHandler(w http.ResponseWriter, r *http.Request) {
    type Store struct {
        Page int `json:"page"`
    }

    store := &Store{}
    datastar.ReadSignals(r, store)

    nextPage := store.Page + 1
    items := fetchItemsForPage(nextPage)
    hasMore := len(items) > 0

    sse := datastar.NewSSE(w, r)

    // Update signals
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "page": nextPage,
        "hasMore": hasMore,
    })

    // Append new items to container
    html := renderItems(items)
    datastar.PatchElements(sse, html,
        datastar.WithSelector("#items-container"),
        datastar.WithMode("append"))
}
```

### Signal Naming & Organization

```html
<!-- Use camelCase for signals -->
<div data-signals:userName="'John'"
     data-signals:userEmail="'john@example.com'">

<!-- Use dot-notation for nested state -->
<div data-signals:user.name="'John'"
     data-signals:user.email="'john@example.com'"
     data-signals:user.preferences.theme="'dark'">

<!-- Use underscore prefix for private/local signals -->
<div data-signals:_internalCounter="0"
     data-signals:_tempValue="''">
<!-- These won't be sent to the backend -->

<!-- Initialize signals at the highest appropriate level -->
<div data-signals="{
    form: {name: '', email: ''},
    validation: {errors: []},
    ui: {loading: false, step: 1}
}">
    <!-- Child elements can access and modify these signals -->
</div>
```

### Loading Indicators

```html
<!-- Simple indicator -->
<div data-indicator:loading>
    <button data-on:click="@get('/data')">Load Data</button>
    <span data-show="$loading">Loading...</span>
</div>

<!-- Multiple operations -->
<div data-indicator:saving data-indicator:deleting>
    <button data-on:click="@post('/save')">Save</button>
    <button data-on:click="@delete('/item')">Delete</button>

    <div data-show="$saving" class="spinner">Saving...</div>
    <div data-show="$deleting" class="spinner">Deleting...</div>
</div>

<!-- Disable button during operation -->
<div data-indicator:submitting>
    <button data-on:click="@post('/submit')"
            data-attr:disabled="$submitting">
        <span data-show="!$submitting">Submit</span>
        <span data-show="$submitting">Submitting...</span>
    </button>
</div>
```

### Optimistic Updates

Update the UI immediately, then send the request. Backend can correct if validation fails.

```html
<div data-signals:count="0">
    <button data-on:click="$count++; @post('/increment')">
        Increment (Optimistic)
    </button>
    <span data-text="$count"></span>
</div>
```

Backend corrects if needed:

```go
func IncrementHandler(w http.ResponseWriter, r *http.Request) {
    type Store struct {
        Count int `json:"count"`
    }

    store := &Store{}
    datastar.ReadSignals(r, store)

    // Validate
    if store.Count > 100 {
        sse := datastar.NewSSE(w, r)
        // Reset to valid value
        datastar.MarshalAndPatchSignals(sse, map[string]any{
            "count": 100,
            "error": "Count cannot exceed 100",
        })
        return
    }

    // Save to database...

    sse := datastar.NewSSE(w, r)
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "error": "",
    })
}
```

### Error Handling Pattern

```html
<form data-on:submit__prevent="@post('/save')"
      data-signals:form.name="''"
      data-signals:error="''"
      data-signals:fieldErrors="{}">

    <div>
        <input data-bind:form.name>
        <span data-show="$fieldErrors.name"
              data-text="$fieldErrors.name"
              class="field-error"></span>
    </div>

    <!-- Global error -->
    <div data-show="$error"
         data-text="$error"
         class="error"></div>

    <button type="submit">Submit</button>
</form>
```

Backend:

```go
func SaveHandler(w http.ResponseWriter, r *http.Request) {
    type FormData struct {
        Form struct {
            Name string `json:"name"`
        } `json:"form"`
    }

    data := &FormData{}
    datastar.ReadSignals(r, data)

    sse := datastar.NewSSE(w, r)

    // Field-level validation
    fieldErrors := make(map[string]string)
    if data.Form.Name == "" {
        fieldErrors["name"] = "Name is required"
    }

    if len(fieldErrors) > 0 {
        datastar.MarshalAndPatchSignals(sse, map[string]any{
            "fieldErrors": fieldErrors,
            "error": "Please fix the errors below",
        })
        return
    }

    // Save...
    if err := save(data); err != nil {
        datastar.MarshalAndPatchSignals(sse, map[string]any{
            "error": "Failed to save: " + err.Error(),
        })
        return
    }

    // Success
    datastar.MarshalAndPatchSignals(sse, map[string]any{
        "error": "",
        "fieldErrors": map[string]string{},
        "success": "Saved successfully",
    })
}
```

### Keep Expressions Simple

Complex logic belongs in the backend. Keep frontend expressions simple and declarative.

```html
<!-- GOOD: Simple expressions -->
<div data-show="$count > 5"></div>
<button data-on:click="$count++; @post('/update')"></button>
<span data-text="`Total: ${$price * $quantity}`"></span>

<!-- AVOID: Complex logic in frontend -->
<div data-show="$items.filter(i => i.active).reduce((sum, i) => sum + i.price, 0) > 100"></div>

<!-- BETTER: Use computed signals for derived values -->
<div data-computed:activeTotal="$items.filter(i => i.active).reduce((sum, i) => sum + i.price, 0)">
<div data-show="$activeTotal > 100"></div>

<!-- BEST: Calculate on backend, send as signal -->
<div data-show="$activeTotal > 100"></div>
<!-- Backend computes activeTotal and sends it -->
```

---

## Debugging

### Using data-json-signals

Display all or filtered signals in the DOM for inspection:

```html
<!-- Show all signals -->
<pre data-json-signals></pre>

<!-- Show only user-related signals -->
<pre data-json-signals="{include: /^user/}"></pre>

<!-- Show all except private signals -->
<pre data-json-signals="{exclude: /^_/}"></pre>

<!-- Combine filters -->
<pre data-json-signals="{include: /^form\./, exclude: /password/}"></pre>
```

### Datastar Inspector

Install the **Datastar Inspector** browser extension (available for Chrome/Firefox) to:
- View all signals and their values in real-time
- Monitor SSE events as they arrive
- See DOM morphing changes
- Debug event handlers and actions
- Inspect timing and performance

### Console Debugging

```html
<!-- Log signal changes -->
<div data-on-signal-patch="console.log('Signal changed:', patch)"></div>

<!-- Log specific signal changes -->
<div data-effect="console.log('Count is now:', $count)"></div>

<!-- Log fetch events -->
<div data-on:datastar-fetch-started="console.log('Request started')"></div>
<div data-on:datastar-fetch-finished="console.log('Request finished')"></div>
<div data-on:datastar-fetch-error="console.error('Request failed:', evt.detail)"></div>
```

---

## Common Gotchas

### 1. Signal Casing

Signal names with hyphens convert to camelCase:

```html
<!-- This attribute: -->
<div data-signals:my-user-name="'John'"></div>

<!-- Creates this signal: -->
<script>
  console.log($myUserName); // 'John'
</script>
```

### 2. Actions Require `@` Prefix

```html
<!-- WRONG: Will not work -->
<button data-on:click="post('/save')">Save</button>

<!-- CORRECT: Use @ prefix -->
<button data-on:click="@post('/save')">Save</button>
```

### 3. Multi-statement Expressions Need Semicolons

```html
<!-- WRONG: Statements need semicolons -->
<button data-on:click="$count = 0
                       @post('/reset')">

<!-- CORRECT: Use semicolons -->
<button data-on:click="$count = 0; @post('/reset')">Reset</button>

<!-- Also correct: -->
<button data-on:click="
  $count = 0;
  $message = 'Reset complete';
  @post('/reset')
">Reset</button>
```

### 4. Element IDs for Morphing

`PatchElements` works best with stable, unique IDs:

```html
<!-- GOOD: Elements have unique IDs -->
<div id="user-profile">...</div>
<div id="user-settings">...</div>

<!-- AVOID: No IDs, morphing relies on position -->
<div>...</div>
<div>...</div>

<!-- Backend -->
datastar.PatchElements(sse, `<div id="user-profile">Updated!</div>`)
```

Without IDs, use selectors:

```go
datastar.PatchElements(sse, `<div>Updated!</div>`,
    datastar.WithSelector(".user-profile"))
```

### 5. SSE Response Headers

Always use `datastar.NewSSE(w, r)` to ensure correct headers:

```go
// WRONG: Manual SSE without helper
func BadHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    // Missing other required headers...
}

// CORRECT: Use NewSSE
func GoodHandler(w http.ResponseWriter, r *http.Request) {
    sse := datastar.NewSSE(w, r)
    // Headers are set correctly automatically
    datastar.PatchElements(sse, `<div>Content</div>`)
}
```

### 6. Signal Types and Binding

Signals preserve types if pre-initialized:

```html
<!-- Pre-initialize with correct type -->
<div data-signals:count="0">
  <!-- This preserves numeric type -->
  <input type="number" data-bind:count />
</div>

<!-- Without pre-initialization, becomes string -->
<div>
  <input type="number" data-bind:count />
  <!-- $count will be a string "42" not number 42 -->
</div>
```

### 7. File Upload Base64 Encoding

File inputs create signals with base64-encoded data:

```html
<input type="file" data-bind:avatar />

<!-- Signal structure: -->
{
  name: "photo.jpg",
  size: 12345,
  type: "image/jpeg",
  lastModified: 1234567890,
  dataURL: "data:image/jpeg;base64,/9j/4AAQ..."
}
```

Backend decoding:

```go
type Store struct {
    Avatar struct {
        Name    string `json:"name"`
        Size    int    `json:"size"`
        Type    string `json:"type"`
        DataURL string `json:"dataURL"`
    } `json:"avatar"`
}

store := &Store{}
datastar.ReadSignals(r, store)

// Decode base64 data
data := strings.TrimPrefix(store.Avatar.DataURL, "data:"+store.Avatar.Type+";base64,")
decoded, err := base64.StdEncoding.DecodeString(data)
```

### 8. Underscore-prefixed Signals

Signals starting with `_` are local-only and NOT sent to backend:

```html
<div data-signals:_localCounter="0"
     data-signals:syncedCounter="0">

    <button data-on:click="$_localCounter++">
        Local (not sent to server)
    </button>

    <button data-on:click="$syncedCounter++; @post('/update')">
        Synced (sent to server)
    </button>
</div>
```

### 9. Request Cancellation

By default, Datastar cancels previous requests on the same element:

```html
<!-- Each keystroke cancels the previous request -->
<input data-on:input="@get('/search')" />

<!-- Disable cancellation to allow concurrent requests -->
<input data-on:input="@get('/search', {requestCancellation: 'none'})" />

<!-- Explicit abort -->
<button data-on:click="@get('/data', {requestCancellation: 'abort'})">
```

### 10. Empty vs Missing Signals

```html
<!-- Signal exists with empty string value -->
<div data-signals:message="''"></div>

<!-- Signal doesn't exist until accessed -->
<div data-text="$undefinedSignal"></div>
<!-- Shows empty string, creates $undefinedSignal with '' -->

<!-- Remove a signal -->
<button data-on:click="$message = null">Clear Message</button>
```

---

## Advanced Patterns

### Polling with Auto-retry

```html
<div data-on-interval__5s="@get('/api/status', {
  retryInterval: 1000,
  retryMaxCount: 3,
  retryScaler: 2
})">
  <div id="status">Checking...</div>
</div>
```

### View Transitions

```html
<nav>
  <a href="/page1"
     data-on:click__prevent__viewtransition="@get('/page1')">
    Page 1
  </a>
  <a href="/page2"
     data-on:click__prevent__viewtransition="@get('/page2')">
    Page 2
  </a>
</nav>

<style>
  /* Define view transition animations */
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation-duration: 0.3s;
  }
</style>
```

### Modal Dialog Pattern

```html
<div data-signals:modalOpen="false">
    <button data-on:click="$modalOpen = true">Open Modal</button>

    <dialog data-ref:modal
            data-attr:open="$modalOpen"
            data-on:click__outside="$modalOpen = false">
        <h2>Modal Title</h2>
        <p>Modal content</p>
        <button data-on:click="$modalOpen = false">Close</button>
    </dialog>
</div>
```

### Tabs Pattern

```html
<div data-signals:activeTab="'tab1'">
    <div class="tabs">
        <button data-on:click="$activeTab = 'tab1'"
                data-class:active="$activeTab === 'tab1'">
            Tab 1
        </button>
        <button data-on:click="$activeTab = 'tab2'"
                data-class:active="$activeTab === 'tab2'">
            Tab 2
        </button>
    </div>

    <div data-show="$activeTab === 'tab1'">
        <h2>Tab 1 Content</h2>
    </div>

    <div data-show="$activeTab === 'tab2'">
        <h2>Tab 2 Content</h2>
    </div>
</div>
```

### Toast Notifications

```html
<div data-signals:toasts="[]">
    <!-- Toast container -->
    <div class="toast-container">
        <template data-for="toast in $toasts">
            <div class="toast" data-class:error="toast.type === 'error'">
                <span data-text="toast.message"></span>
                <button data-on:click="$toasts = $toasts.filter(t => t.id !== toast.id)">
                    ×
                </button>
            </div>
        </template>
    </div>
</div>
```

Backend adding toast:

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    sse := datastar.NewSSE(w, r)

    // Execute script to add toast
    datastar.ExecuteScript(sse, `
        $toasts = [...$toasts, {
            id: Date.now(),
            type: 'success',
            message: 'Operation completed!'
        }];
        setTimeout(() => {
            $toasts = $toasts.filter(t => t.id !== ${Date.now()});
        }, 3000);
    `)
}
```

---

## Pro Features

Datastar offers **Pro features** under a commercial license, which include:

- `data-persist` - Persist signals to localStorage/sessionStorage
- `data-query-string` - Sync signals with URL query parameters
- `data-animate` - Declarative animations
- `@clipboard` - Clipboard operations
- Additional tooling and support

This prompt focuses on the core, open-source framework available under the MIT license.

---

## Reference Links

- **Official Docs**: https://data-star.dev
- **Go SDK**: https://pkg.go.dev/github.com/starfederation/datastar-go
- **GitHub**: https://github.com/starfederation/datastar
- **CDN**: https://cdn.jsdelivr.net/npm/@sudodevnull/datastar

---

## Version Info

- **Datastar**: v1.0.0-RC.6
- **Go SDK**: Requires Go 1.24+
- **License**: MIT (core framework)
