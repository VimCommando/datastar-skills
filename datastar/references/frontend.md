# Frontend Reference

Use this file when writing or reviewing Datastar frontend code: `data-*` attributes, signal syntax, event modifiers, backend action expressions, and core browser-side behavior.

## Installation

Datastar is available as a CDN-hosted ES module. Ensure it is loaded before using Datastar attributes.

```html
<script type="module" src="https://cdn.jsdelivr.net/gh/starfederation/datastar@1.0.0-RC.8/bundles/datastar.js"></script>
```

## State Management

Signals are reactive variables denoted with a `$` prefix that automatically track and propagate changes through expressions. Treat signals as a minimal frontend state layer while the backend remains the source of truth.

### Methods To Create Signals

1. **Explicit via `data-signals`**: Directly sets signal values. Signal names convert to camelCase.

```html
<div data-signals:count="0" data-signals:message="'Hello'"></div>
<div data-signals:form.name="'John'" data-signals:form.email="''"></div>
<div data-signals="{form: {name: 'John', email: ''}, count: 0}"></div>
<div data-signals:isActive="true" data-signals:price="99.99"></div>
```

2. **Implicit via `data-bind`**: Automatically creates signals when binding inputs. Preserves types of predefined signals.

```html
<input data-bind:foo />
<input type="checkbox" data-bind:isActive />
<input type="file" data-bind:avatar />
<input type="file" data-bind:documents multiple />
<input type="number" data-bind:quantity />
<select data-bind:category>
  <option value="a">Category A</option>
  <option value="b">Category B</option>
</select>
```

File signals contain `{name, size, type, lastModified, dataURL}`.

3. **Computed via `data-computed`**: Creates derived, read-only signals that automatically update when dependencies change.

```html
<div data-computed:doubled="$foo * 2"></div>
<div data-computed:fullName="$firstName + ' ' + $lastName"></div>
<div data-computed:total="$price * $quantity"></div>
<div data-computed:isEmpty="$items.length === 0"></div>
```

4. **Element reference via `data-ref`**: Creates a signal that is a reference to the DOM element for direct manipulation.

```html
<input data-ref:searchInput />
<button data-on:click="$searchInput.focus()">Focus Search</button>

<div data-ref:modal></div>
<button data-on:click="$modal.showModal()">Open Modal</button>
```

### Signal Rules

- Signals are globally accessible throughout the application.
- Signals accessed without explicit creation default to an empty string.
- Signals prefixed with an underscore like `$_private` are local-only and are not sent to the backend by default.
- Use dot-notation for organization: `$form.email`, `$user.profile.name`.
- Setting values to `null` or `undefined` removes the signal.
- All non-underscore signals are sent with backend requests by default.
- Signal names with hyphens convert to camelCase: `my-value` becomes `$myValue`.

## DOM Binding And Display

```html
<span data-text="$count"></span>
<p data-text="`Hello, ${$name}!`"></p>

<div data-attr:title="$message"></div>
<input data-attr:disabled="$foo === ''" />
<a data-attr:href="`/page/${$id}`"></a>
<img data-attr:src="$imageUrl" data-attr:alt="$imageAlt" />

<input data-bind:message />
<input type="checkbox" data-bind:isActive />
<textarea data-bind:description></textarea>

<div data-show="$count > 5"></div>
<div data-show="$isLoggedIn && !$isLoading"></div>

<div data-class:active="$isActive"></div>
<div data-class="{active: $isActive, error: $hasError, disabled: $isDisabled}"></div>

<div data-style:color="$color"></div>
<div data-style="{color: $color, fontSize: `${$size}px`, display: $visible ? 'block' : 'none'}"></div>

<div data-ignore></div>
<div data-ignore-morph></div>

<details open data-preserve-attr="open">...</details>
<input data-preserve-attr="value" />
```

## Event Handling

```html
<button data-on:click="@post('/increment')">Click Me</button>
<button data-on:click="$count = 0">Reset</button>
<button data-on:click="$count++; @post('/update')">Increment & Save</button>

<input data-on:input="$query = evt.target.value" />
<div data-on:mouseenter="$hover = true" data-on:mouseleave="$hover = false"></div>
<form data-on:submit__prevent="@post('/save')"></form>

<input data-on:input__debounce.500ms="@get('/search')" />
<input data-on:input__throttle.200ms="@get('/filter')" />
<button data-on:click__delay.1s="console.log('Delayed')">Delay</button>

<div data-on:click__stop="console.log('Propagation stopped')"></div>
<button data-on:click__once="@post('/init')">Initialize Once</button>
<div data-on:scroll__passive="console.log('Passive listener')"></div>

<div data-on:keydown__window="console.log('Global key press')"></div>
<div data-on:click__outside="$menuOpen = false"></div>
<button data-on:click__viewtransition="@get('/next-page')">Navigate</button>

<input data-on:input__debounce.300ms__leading="@get('/search')" />
<input data-on:input__debounce.300ms__noleading="@get('/search')" />
<div data-on:scroll__throttle.100ms__trailing="console.log('Scroll end')"></div>
<div data-on:scroll__throttle.100ms__notrailing="console.log('Scroll start')"></div>

<div data-on-intersect="@get('/load-more')"></div>
<div data-on-intersect__once__half="$visible = true"></div>
<div data-on-intersect__full="console.log('Fully visible')"></div>

<div data-on-interval="$count++"></div>
<div data-on-interval__2s="@get('/poll')"></div>
<div data-on-interval__500ms__leading="console.log('Tick')"></div>

<div data-on-signal-patch="console.log('Any signal changed:', patch)"></div>
<div data-on-signal-patch-filter="{include: /^counter$/}"></div>
<div data-on-signal-patch-filter="{exclude: /^_/}"></div>

<div data-on:datastar-fetch-started="$loading = true"></div>
<div data-on:datastar-fetch-finished="$loading = false"></div>
<div data-on:datastar-fetch-error="$error = evt.detail.error"></div>
<div data-on:datastar-fetch-retrying="console.log('Retry attempt:', evt.detail.attempt)"></div>
<div data-on:datastar-fetch-retries-failed="alert('Request failed after retries')"></div>
<div data-on:datastar-fetch="evt.detail.type === 'error' && alert('Failed')"></div>
```

### Event Modifiers

- `__once`: Event fires only once.
- `__passive`: Event listener is passive.
- `__capture`: Use capture phase instead of bubbling.
- `__prevent`: Calls `evt.preventDefault()`.
- `__stop`: Calls `evt.stopPropagation()`.
- `__delay.{time}`: Delay execution.
- `__debounce.{time}`: Wait for silence before executing.
- `__throttle.{time}`: Rate-limit execution.
- `__leading`: Execute on the leading edge.
- `__noleading`: Disable leading-edge execution.
- `__trailing`: Execute on the trailing edge.
- `__notrailing`: Disable trailing-edge execution.
- `__window`: Attach listener to `window`.
- `__outside`: Trigger when clicking outside the element.
- `__viewtransition`: Enable the View Transitions API.
- `__half`: Trigger `data-on-intersect` at 50 percent visibility.
- `__full`: Trigger `data-on-intersect` at 100 percent visibility.

## Backend Actions

Assume all non-underscore signals are sent with Datastar HTTP method actions unless the expression explicitly filters them out.

```html
<button data-on:click="@get('/endpoint')">GET</button>
<button data-on:click="@post('/endpoint')">POST</button>
<button data-on:click="@put('/endpoint')">PUT</button>
<button data-on:click="@patch('/endpoint')">PATCH</button>
<button data-on:click="@delete('/endpoint')">DELETE</button>

<form data-on:submit__prevent="@post('/save', {contentType: 'form'})">
  <input name="name" data-bind:name />
  <button>Submit</button>
</form>

<button data-on:click="@post('/partial', {filterSignals: {include: /^count/}})">
  Send Only Count Signals
</button>

<button data-on:click="@post('/safe', {filterSignals: {exclude: /^_/}})">
  Exclude Private Signals
</button>

<button data-on:click="@get('/update', {selector: '#custom-target'})">
  Update Custom Target
</button>

<button data-on:click="@post('/api', {headers: {'X-Custom-Header': 'value'}})">
  With Headers
</button>

<input data-on:input="@get('/search', {requestCancellation: 'auto'})" />

<button
  data-on:click="@get('/unstable', {
    retryInterval: 1000,
    retryScaler: 2,
    retryMaxWaitMs: 10000,
    retryMaxCount: 3
  })"
>
  Retry on Failure
</button>

<button data-on:click="@post('/special', {override: true})">
  Override Defaults
</button>

<div data-on:interval__10s="@get('/heartbeat', {openWhenHidden: true})"></div>
```

### Backend Action Options

- `contentType`: `'json'` by default or `'form'`.
- `filterSignals`: Object with `include` and or `exclude` regex patterns.
- `selector`: CSS selector for targeted DOM patching.
- `headers`: Custom request headers.
- `openWhenHidden`: Keep the SSE connection open when the tab is hidden.
- `retryInterval`: Initial retry delay in milliseconds.
- `retryScaler`: Exponential backoff multiplier.
- `retryMaxWaitMs`: Maximum wait time between retries.
- `retryMaxCount`: Maximum retry attempts.
- `requestCancellation`: `'auto'`, `'none'`, or `'abort'`.
- `override`: Override default backend action behavior.

### How Signals Are Sent

- `GET`: Sent as a `datastar` query parameter with URL-encoded JSON.
- `POST`, `PUT`, `PATCH`, and `DELETE`: Sent as a JSON request body unless `contentType: 'form'` is used.
- `contentType: 'form'`: Sends `application/x-www-form-urlencoded` and omits signals.

## Frontend-Only Actions

```html
<div data-effect="console.log(@peek(() => $foo))"></div>

<button data-on:click="@setAll(true, {include: /^menu\.isOpen/})">
  Open All Menus
</button>

<button data-on:click="@toggleAll({include: /^is/})">
  Toggle All Boolean Flags
</button>

<button data-on:click="@toggleAll({include: /^feature\./, exclude: /disabled/})">
  Toggle Features
</button>
```

## Initialization And Effects

```html
<div data-init="console.log('Element initialized')"></div>
<div data-init="$startTime = Date.now()"></div>

<div data-effect="console.log('Count changed:', $count)"></div>
<div data-effect="$total = $price * $quantity"></div>

<div data-effect="console.log('User:', $user)" data-init="console.log('Component mounted')"></div>
```

## Loading States

```html
<div data-indicator:saving>
  <button data-on:click="@post('/save')">Save</button>
  <span data-show="$saving">Saving...</span>
</div>

<div data-indicator:loading data-indicator:deleting>
  <button data-on:click="@get('/data')">Load</button>
  <button data-on:click="@delete('/item')">Delete</button>
  <span data-show="$loading">Loading...</span>
  <span data-show="$deleting">Deleting...</span>
</div>
```

## Debug And Introspection

```html
<pre data-json-signals></pre>
<pre data-json-signals="{include: /^user/}"></pre>
<pre data-json-signals="{exclude: /^_/}"></pre>
<pre data-json-signals="{include: /^form\./, exclude: /password/}"></pre>
```

## Pro Features

Datastar Pro includes:

- `data-persist` for local storage or session storage persistence.
- `data-query-string` for syncing signals with URL query parameters.
- `data-animate` for declarative animations.
- `@clipboard` for clipboard operations.

This reference focuses on the core open-source framework.

## Reference Links

- Official Docs: https://data-star.dev
- GitHub: https://github.com/starfederation/datastar
- CDN: https://cdn.jsdelivr.net/gh/starfederation/datastar@v1.0.0-RC.8/bundles/datastar.js

## Version Info

- Datastar: v1.0.0-RC.8
- License: MIT for the core framework
