---
name: datastar
description: Datastar-first web app development skill for backend-driven HTML over SSE, signal design, `data-*` attribute usage, and Datastar interaction patterns. Use when building, reviewing, debugging, refactoring, or planning Datastar apps; when choosing what belongs in backend state vs signals; when authoring `data-*` attributes or SSE patches; or when implementing backend-driven frontend behavior.
---

# Datastar Skill

Use this skill as the primary guide for Datastar architecture and backend-driven UI decisions.

Agent default: after selecting this skill, load only the references needed for the task.

- Frontend attribute usage, signal syntax, event modifiers, and action examples: load `references/frontend.md`.
- Go backend: load `references/go.md`.
- Rust backend: load `references/rust.md`.
- TypeScript backend: load `references/typescript.md`.
- Another backend language: load the matching reference file when available.

## Datastar Tao

Apply these as agent defaults. If a design choice conflicts with one of these points, treat that as a review trigger and justify the exception explicitly.

**State in the right place**: Agent default: keep most state in the backend. Since the frontend is exposed to the user, the backend should be the source of truth for your application state.

**Start with the Defaults**: Agent default: start with default configuration options. The default configuration options are the recommended settings for the majority of applications. Before changing them, stop and identify the concrete requirement forcing the change.

**Patch Elements & Signals**: Agent default: have the backend drive the frontend by patching (adding, updating and removing) HTML elements and signals.

**Use Signals Sparingly**: Agent trigger: if you are adding many signals, stop and re-check whether state belongs in the backend. Favor fetching current state from the backend rather than pre-loading and assuming frontend state is current. A good rule of thumb is to only use signals for user interactions and for sending new state to the backend.

**Prefer Morphing**: Agent default: prefer morphing over fine-grained frontend state management. Morphing ensures that only modified parts of the DOM are updated, preserving state and improving performance. This allows you to send down large chunks of the DOM tree, including "fat morph" updates, rather than trying to manage fine-grained updates yourself. If you want to explicitly ignore morphing an element, place the `data-ignore-morph` attribute on it.

**SSE Responses**: Agent default: use SSE responses that send 0 to n events, in which you can patch elements, patch signals, and execute scripts. Since event streams are just HTTP responses with some special formatting that SDKs can handle for you, there is usually no benefit to using a content type other than `text/event-stream`.

**Compression**: Agent default: enable compression for SSE responses when available. Since SSE responses stream events from the backend and morphing allows sending large chunks of DOM, compressing the response is a natural choice.

**Backend Templating**: Agent default: use your backend templating language to keep generated HTML DRY.

**Page Navigation**: Agent default: use the anchor element (`<a>`) to navigate to a new page, or a redirect if redirecting from the backend. For smooth page transitions, use the View Transition API.

**Browser History**: Agent trigger: if you are about to manage browser history directly, stop and justify it. Browsers automatically keep a history of pages visited. Each page is a resource. Use anchor tags and let the browser handle history by default.

**CQRS**: Agent default: separate commands (writes) from requests (reads). CQRS makes it possible to have a single long-lived request to receive updates from the backend while making multiple short-lived requests to the backend for commands.

```html
<div id="main" data-init="@get('/cqrs_endpoint')">
  <button data-on:click="@post('/do_something')">Do something</button>
</div>
```

When using CQRS, agent default: manually show a loading indicator when backend requests are made, and allow it to be hidden when the DOM is updated from the backend.

```html
<div>
  <button data-on:click="el.classList.add('loading'); @post('/do_something')">
    Do something
    <span>Loading...</span>
  </button>
</div>
```

**Optimistic Updates**: Agent default: do not use optimistic updates unless the task explicitly requires them. Rather than showing success before confirmation, use loading indicators to show the user that the action is in progress, and only confirm success from the backend.

**Accessibility**: Agent default: preserve accessibility in every Datastar change. Use semantic HTML, apply ARIA where it makes sense, and ensure your app works well with keyboards and screen readers.

```html
<button
  data-on:click="$_menuOpen = !$_menuOpen"
  data-attr:aria-expanded="$_menuOpen ? 'true' : 'false'"
>
  Open/Close Menu
</button>
<div data-attr:aria-hidden="$_menuOpen ? 'false' : 'true'"></div>
```

## Architecture Pattern

```text
┌─────────────┐                    ┌──────────────┐
│   Browser   │                    │   Backend    │
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

- Client sends all non-underscore signals with requests by default.
- Server reads signals, processes logic, and sends back SSE events.
- SSE events update DOM fragments and or signals.
- Prefer Datastar handlers over layering a separate REST-style frontend state protocol.

## Workflow

1. Decide which state must stay authoritative on the backend.
2. Design the HTML fragments the backend will patch back over SSE.
3. Keep frontend signals minimal and scoped to interaction, form binding, and transient UI state.
4. Load `references/frontend.md` when writing or reviewing `data-*` attributes, signal syntax, event modifiers, or frontend-only Datastar behavior.
5. Load the matching backend reference before writing handlers or SSE patch code.
