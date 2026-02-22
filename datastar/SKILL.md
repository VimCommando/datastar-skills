---
name: datastar
description: Datastar-first web app development skill focused on architecture, reactive signal design, attribute usage, and frontend interaction patterns. Use when building or reviewing Datastar applications, designing signal flow, authoring `data-*` attributes, implementing frontend behaviors, or composing backend-driven SSE interactions.
---

# Datastar Skill

Use this skill as the primary guide for Datastar design concepts, architectural patterns, and frontend attribute usage.

Select backend references by language from `references/`:
- For Go backends, load `references/go.md`.
- For Rust backends, load `references/rust.md`.
- For TypeScript backends, load `references/typescript.md`.
- For future languages (for example PHP), load the matching language reference file when available.

## Core Philosophy

**Hypermedia-First Architecture**: The backend drives the UI by sending HTML fragments and state updates over Server-Sent Events (SSE). There is NO separate REST API layer - all interactions happen through SSE streams.

**Backend Reactivity**: The server is responsible for rendering HTML and managing application state. The frontend is a thin reactive layer that responds to backend updates. As the docs state: "The backend determines what the user can do by controlling DOM patches, maintaining single source of truth."

**Progressive Enhancement**: Start with semantic HTML, enhance with `data-*` attributes for reactivity. No npm dependencies or JavaScript build step required - just include the Datastar script from CDN.

**Simplicity First**: Keep Datastar expressions simple - complex logic belongs in backend handlers or external scripts. The framework uses a "props down, events up" pattern: pass data into functions via arguments, return results or dispatch custom events.

**v1.0.0-RC.6 Enhancements**: This release includes refined attribute syntax, expanded event modifiers, improved SSE handling, and better TypeScript/Go SDK support.

---

## Architecture Pattern

```
┌─────────────┐                    ┌──────────────┐
│   Browser   │                    │  Go Backend  │
│             │                    │              │
│  Datastar   │ ◄─── SSE Stream ───┤  HTTP Handler│
│  (signals)  │      (HTML/State)  │              │
│             │                    │              │
│  DOM        │ ──── HTTP POST ───►│  Read State  │
│             │      (all signals) │  Process     │
│             │                    │  Send SSE    │
└─────────────┘                    └──────────────┘
```

**Key Points:**
- Client sends ALL signals (application state) with every request.
- Server reads signals, processes logic, and sends back SSE events.
- SSE events update DOM (HTML fragments) and/or signals (state).
- NO REST endpoints - only SSE-returning handlers.

---

## Datastar Attribute Reference (Latest Syntax)

### State Management (Reactive Signals)

Signals are reactive variables denoted with a `$` prefix that automatically track and propagate changes through expressions. They serve as the frontend state layer while the backend acts as the source of truth.

**Methods to Create Signals:**

1.  **Explicit via `data-signals`**: Directly sets signal values. Signal names convert to camelCase.
    ```html
    <!-- Basic signals -->
    <div data-signals:count="0" data-signals:message="'Hello'"></div>

    <!-- Nested signals with dot-notation -->
    <div data-signals:form.name="'John'" data-signals:form.email="''"></div>

    <!-- Object syntax for complex state -->
    <div data-signals="{form: {name: 'John', email: ''}, count: 0}"></div>

    <!-- Boolean and numeric types -->
    <div data-signals:isActive="true" data-signals:price="99.99"></div>
    ```

2.  **Implicit via `data-bind`**: Automatically creates signals when binding inputs. Preserves types of predefined signals.
    ```html
    <!-- Text input creates string signal -->
    <input data-bind:foo />

    <!-- Checkbox creates boolean signal -->
    <input type="checkbox" data-bind:isActive />

    <!-- File input with base64 encoding -->
    <input type="file" data-bind:avatar />
    <input type="file" data-bind:documents multiple />
    <!-- File signals contain: {name, size, type, lastModified, dataURL} -->

    <!-- Number input preserves numeric type -->
    <input type="number" data-bind:quantity />

    <!-- Select creates signal from selected value -->
    <select data-bind:category>
      <option value="a">Category A</option>
      <option value="b">Category B</option>
    </select>
    ```

3.  **Computed via `data-computed`**: Creates derived, read-only signals that automatically update when dependencies change.
    ```html
    <div data-computed:doubled="$foo * 2"></div>
    <div data-computed:fullName="$firstName + ' ' + $lastName"></div>
    <div data-computed:total="$price * $quantity"></div>
    <div data-computed:isEmpty="$items.length === 0"></div>
    ```

4. **Element Reference via `data-ref`**: Creates a signal that is a reference to the DOM element for direct manipulation.
    ```html
    <input data-ref:searchInput />
    <button data-on:click="$searchInput.focus()">Focus Search</button>

    <div data-ref:modal></div>
    <button data-on:click="$modal.showModal()">Open Modal</button>
    ```

**Important Signal Rules:**
- Signals are globally accessible throughout the application.
- Signals accessed without explicit creation default to an empty string.
- Signals prefixed with an underscore (`$_private`) are NOT sent to the backend by default (local-only signals).
- Use dot-notation for organization: `$form.email`, `$user.profile.name`.
- Setting values to `null` or `undefined` removes the signal.
- All non-underscore signals are sent with every backend request (GET as query param, POST/PUT/PATCH/DELETE in body).
- Signal names with hyphens convert to camelCase: `my-value` becomes `$myValue`.

### DOM Binding & Display

```html
<!-- Bind text content -->
<span data-text="$count"></span>
<p data-text="`Hello, ${$name}!`"></p>

<!-- Bind attributes -->
<div data-attr:title="$message"></div>
<input data-attr:disabled="$foo === ''" />
<a data-attr:href="`/page/${$id}`"></a>
<img data-attr:src="$imageUrl" data-attr:alt="$imageAlt" />

<!-- Two-way binding for inputs -->
<input data-bind:message />
<input type="checkbox" data-bind:isActive />
<textarea data-bind:description></textarea>

<!-- Conditional display -->
<div data-show="$count > 5"></div>
<div data-show="$isLoggedIn && !$isLoading"></div>

<!-- CSS classes (object or single) -->
<div data-class:active="$isActive"></div>
<div data-class="{active: $isActive, error: $hasError, disabled: $isDisabled}"></div>

<!-- Inline styles (object or single) -->
<div data-style:color="$color"></div>
<div data-style="{color: $color, fontSize: `${$size}px`, display: $visible ? 'block' : 'none'}"></div>

<!-- Ignore element from Datastar processing -->
<div data-ignore>
  <!-- This and child elements won't be processed by Datastar -->
</div>

<!-- Ignore morphing (element won't be updated by SSE patches) -->
<div data-ignore-morph>
  <!-- Content is static, won't be replaced by server updates -->
</div>

<!-- Preserve attributes during DOM morphing -->
<details open data-preserve-attr="open">...</details>
<input data-preserve-attr="value">
```

### Event Handling

```html
<!-- Basic event listeners (evt variable available for event object) -->
<button data-on:click="@post('/increment')">Click Me</button>
<button data-on:click="$count = 0">Reset</button>
<button data-on:click="$count++; @post('/update')">Increment & Save</button>

<!-- All standard DOM events supported -->
<input data-on:input="$query = evt.target.value" />
<div data-on:mouseenter="$hover = true" data-on:mouseleave="$hover = false"></div>
<form data-on:submit__prevent="@post('/save')"></form>

<!-- Event modifiers - Timing Control -->
<input data-on:input__debounce.500ms="@get('/search')" />
<input data-on:input__throttle.200ms="@get('/filter')" />
<button data-on:click__delay.1s="console.log('Delayed')">Delay</button>

<!-- Event modifiers - Behavior Control -->
<form data-on:submit__prevent="@post('/save')">Submit</form>
<div data-on:click__stop="console.log('Propagation stopped')"></div>
<button data-on:click__once="@post('/init')">Initialize Once</button>
<div data-on:scroll__passive="console.log('Passive listener')"></div>

<!-- Event modifiers - Scope Control -->
<div data-on:keydown__window="console.log('Global key press')"></div>
<div data-on:click__outside="$menuOpen = false"></div>

<!-- Event modifiers - View Transitions -->
<button data-on:click__viewtransition="@get('/next-page')">Navigate</button>

<!-- Timing edge modifiers (for debounce/throttle) -->
<input data-on:input__debounce.300ms__leading="@get('/search')" />
<input data-on:input__debounce.300ms__noleading="@get('/search')" />
<div data-on:scroll__throttle.100ms__trailing="console.log('Scroll end')"></div>
<div data-on:scroll__throttle.100ms__notrailing="console.log('Scroll start')"></div>

<!-- Special event attributes - Intersection Observer -->
<div data-on-intersect="@get('/load-more')"></div>
<div data-on-intersect__once__half="$visible = true"></div>
<div data-on-intersect__full="console.log('Fully visible')"></div>

<!-- Special event attributes - Intervals -->
<div data-on-interval="$count++"></div>
<div data-on-interval__2s="@get('/poll')"></div>
<div data-on-interval__500ms__leading="console.log('Tick')"></div>

<!-- Special event attributes - Signal Changes -->
<div data-on-signal-patch="console.log('Any signal changed:', patch)"></div>
<div data-on-signal-patch-filter="{include: /^counter$/}">
  <!-- Only triggers when 'counter' signal changes -->
</div>
<div data-on-signal-patch-filter="{exclude: /^_/}">
  <!-- Triggers for all non-private signal changes -->
</div>

<!-- Fetch lifecycle events -->
<div data-on:datastar-fetch-started="$loading = true"></div>
<div data-on:datastar-fetch-finished="$loading = false"></div>
<div data-on:datastar-fetch-error="$error = evt.detail.error"></div>
<div data-on:datastar-fetch-retrying="console.log('Retry attempt:', evt.detail.attempt)"></div>
<div data-on:datastar-fetch-retries-failed="alert('Request failed after retries')"></div>

<!-- Generic fetch event handler -->
<div data-on:datastar-fetch="evt.detail.type === 'error' && alert('Failed')"></div>
```

**Available Event Modifiers:**

**Standard Event Options:**
- `__once` - Event fires only once
- `__passive` - Event listener is passive (improves scrolling performance)
- `__capture` - Use capture phase instead of bubbling

**Event Behavior Control:**
- `__prevent` - Calls `evt.preventDefault()`
- `__stop` - Calls `evt.stopPropagation()`

**Timing Control (accepts ms or s suffix, e.g., 300ms, 1s):**
- `__delay.{time}` - Delays execution by specified time
- `__debounce.{time}` - Debounces the event (waits for silence)
- `__throttle.{time}` - Throttles the event (rate limits)

**Timing Edge Control (for debounce/throttle):**
- `__leading` - Execute on the leading edge
- `__noleading` - Don't execute on the leading edge
- `__trailing` - Execute on the trailing edge
- `__notrailing` - Don't execute on the trailing edge

**Scope Modifiers:**
- `__window` - Attach listener to window instead of element
- `__outside` - Trigger when clicking outside the element

**View Transitions:**
- `__viewtransition` - Enable View Transitions API for smooth animations

**Intersection Thresholds (for data-on-intersect):**
- `__half` - Trigger when 50% visible (0.5 threshold)
- `__full` - Trigger when 100% visible (1.0 threshold)

### Backend Actions (`@` Prefix)

Datastar provides five HTTP method actions. All non-underscore signals are sent with the request.

```html
<!-- HTTP methods -->
<button data-on:click="@get('/endpoint')">GET</button>
<button data-on:click="@post('/endpoint')">POST</button>
<button data-on:click="@put('/endpoint')">PUT</button>
<button data-on:click="@patch('/endpoint')">PATCH</button>
<button data-on:click="@delete('/endpoint')">DELETE</button>

<!-- With options object -->
<form data-on:submit__prevent="@post('/save', {contentType: 'form'})">
  <input name="name" data-bind:name />
  <button>Submit</button>
</form>

<!-- Filter which signals to send -->
<button data-on:click="@post('/partial', {filterSignals: {include: /^count/}})">
  Send Only Count Signals
</button>

<button data-on:click="@post('/safe', {filterSignals: {exclude: /^_/}})">
  Exclude Private Signals
</button>

<!-- Custom selector for DOM updates -->
<button data-on:click="@get('/update', {selector: '#custom-target'})">
  Update Custom Target
</button>

<!-- Custom headers -->
<button data-on:click="@post('/api', {headers: {'X-Custom-Header': 'value'}})">
  With Headers
</button>

<!-- Request cancellation strategies -->
<input data-on:input="@get('/search', {requestCancellation: 'auto'})" />
<!-- 'auto' (default): Cancel previous requests on same element -->
<!-- 'none': Don't cancel, allow concurrent requests -->
<!-- 'abort': Explicitly abort ongoing request -->

<!-- Retry configuration -->
<button data-on:click="@get('/unstable', {
  retryInterval: 1000,
  retryScaler: 2,
  retryMaxWaitMs: 10000,
  retryMaxCount: 3
})">
  Retry on Failure
</button>

<!-- Override default behavior -->
<button data-on:click="@post('/special', {override: true})">
  Override Defaults
</button>

<!-- Open SSE connection even when tab is hidden -->
<div data-on:interval__10s="@get('/heartbeat', {openWhenHidden: true})"></div>
```

**Backend Action Options:**

- `contentType`: `'json'` (default) or `'form'`. Form sends as `application/x-www-form-urlencoded` without signals.
- `filterSignals`: Object with `include` and/or `exclude` regex patterns to filter which signals are sent.
- `selector`: CSS selector for where to patch DOM updates (overrides default ID-based matching).
- `headers`: Object of custom HTTP headers to send with the request.
- `openWhenHidden`: Boolean, whether to keep SSE connection open when page is hidden (default: false).
- `retryInterval`: Initial retry delay in milliseconds (default: 1000).
- `retryScaler`: Multiplier for exponential backoff (default: 2).
- `retryMaxWaitMs`: Maximum wait time between retries in milliseconds (default: 10000).
- `retryMaxCount`: Maximum number of retry attempts (default: 3).
- `requestCancellation`: `'auto'` (default), `'none'`, or `'abort'`.
- `override`: Boolean to override default backend action behaviors.

**How Signals Are Sent:**
- **GET**: As a `datastar` query parameter (URL-encoded JSON).
- **POST/PUT/PATCH/DELETE**: As a JSON request body (unless `contentType: 'form'`).
- **`contentType: 'form'`**: Sends data as `application/x-www-form-urlencoded`; no signals are sent.

### Frontend-Only Actions (`@` Prefix)

```html
<!-- Access a signal without subscribing to its changes -->
<div data-effect="console.log(@peek(() => $foo))"></div>

<!-- Set multiple signals matching a regex -->
<button data-on:click="@setAll(true, {include: /^menu\.isOpen/})">
  Open All Menus
</button>

<!-- Toggle multiple boolean signals -->
<button data-on:click="@toggleAll({include: /^is/})">
  Toggle All Boolean Flags
</button>

<button data-on:click="@toggleAll({include: /^feature\./, exclude: /disabled/})">
  Toggle Features
</button>
```

### Initialization & Effects

```html
<!-- Run once on element mount/patch -->
<div data-init="console.log('Element initialized')"></div>
<div data-init="$startTime = Date.now()"></div>

<!-- React to signal changes (runs whenever dependencies change) -->
<div data-effect="console.log('Count changed:', $count)"></div>
<div data-effect="$total = $price * $quantity"></div>

<!-- Multiple effects on same element -->
<div data-effect="console.log('User:', $user)"
     data-init="console.log('Component mounted')"></div>
```

### Loading States

```html
<!-- Create a 'saving' signal that is true during the request -->
<div data-indicator:saving>
  <button data-on:click="@post('/save')">Save</button>
  <span data-show="$saving">Saving...</span>
</div>

<!-- Multiple indicators for different operations -->
<div data-indicator:loading data-indicator:deleting>
  <button data-on:click="@get('/data')">Load</button>
  <button data-on:click="@delete('/item')">Delete</button>
  <span data-show="$loading">Loading...</span>
  <span data-show="$deleting">Deleting...</span>
</div>
```

### Debug & Introspection

```html
<!-- Display all signals as JSON -->
<pre data-json-signals></pre>

<!-- Display filtered signals -->
<pre data-json-signals="{include: /^user/}"></pre>
<pre data-json-signals="{exclude: /^_/}"></pre>
<pre data-json-signals="{include: /^form\./, exclude: /password/}"></pre>
```

---
