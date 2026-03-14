# TypeScript Backend Reference

Use this file for Datastar + TypeScript backend implementation details.

## Table of Contents

- [TypeScript Backend Patterns](#typescript-backend-patterns)
- [Common Patterns & Best Practices](#common-patterns--best-practices)
- [Debugging](#debugging)
- [Common Gotchas](#common-gotchas)
- [Advanced Patterns](#advanced-patterns)
- [Reference Links](#reference-links)
- [Version Info](#version-info)

## TypeScript Backend Patterns

### SDK Installation

```bash
npm install @starfederation/datastar-sdk
```

Runtime imports:

- Node.js: `@starfederation/datastar-sdk/node`
- Deno: `npm:@starfederation/datastar-sdk/web`
- Bun: `@starfederation/datastar-sdk/web`

### Basic Handler Pattern (Node HTTP)

```ts
import http from "node:http";
import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/node";

type CounterSignals = {
  count?: number;
  name?: string;
};

const server = http.createServer(async (req, res) => {
  if (req.url !== "/counter/increment" || req.method !== "POST") {
    res.writeHead(404).end();
    return;
  }

  const reader = await ServerSentEventGenerator.readSignals(req);
  if (!reader.success) {
    res.writeHead(400, { "content-type": "text/plain" });
    res.end(reader.error ?? "Invalid signals");
    return;
  }

  const signals = (reader.signals ?? {}) as CounterSignals;
  const nextCount = (signals.count ?? 0) + 1;
  const isValid = (signals.name ?? "").trim().length > 0;

  ServerSentEventGenerator.stream(req, res, (stream) => {
    stream.patchSignals(JSON.stringify({
      count: nextCount,
      isValid,
      loading: false,
    }));

    stream.patchElements(`<p id="counter">Current count: ${nextCount}</p>`);
  });
});

server.listen(3000);
```

### Reading Client Signals

```ts
import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/node";

const reader = await ServerSentEventGenerator.readSignals(req);
if (!reader.success) {
  res.writeHead(400).end(reader.error ?? "Bad Request");
  return;
}

const { email = "", password = "", remember = false } =
  (reader.signals ?? {}) as {
    email?: string;
    password?: string;
    remember?: boolean;
  };
```

### SSE Event Methods

TypeScript stream instance methods map to Datastar SSE events.

```ts
import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/node";

ServerSentEventGenerator.stream(req, res, (stream) => {
  // DOM updates
  stream.patchElements('<div id="result">Updated content</div>');

  stream.patchElements('<li>New Item</li>', {
    selector: "#item-list",
    mode: "append",
  });

  // Remove elements
  stream.removeElements("#temporary-message");
  stream.removeElements(undefined, '<div id="stale-row"></div>');

  // Signal updates
  stream.patchSignals(
    JSON.stringify({
      count: 42,
      message: "Hello from TypeScript",
      user: { name: "John", email: "john@example.com" },
    }),
  );

  stream.patchSignals(JSON.stringify({ defaultTheme: "dark" }), {
    onlyIfMissing: true,
  });

  // Remove signals
  stream.removeSignals("errorMessage");
  stream.removeSignals(["draft", "tempSelection"]);

  // Execute client-side scripts
  stream.executeScript("console.log('Server says hello!')");
  stream.executeScript("window.scrollTo({ top: 0, behavior: 'smooth' })", {
    autoRemove: true,
    attributes: { type: "module" },
  });
});
```

`patchElements(..., { mode })` supports:
`outer`, `inner`, `replace`, `prepend`, `append`, `before`, `after`, `remove`.

### SSE Event Types (Low-level)

The stream methods above emit Datastar events:

- `datastar-patch-elements`
- `datastar-patch-signals`
- `datastar-remove-elements`
- `datastar-remove-signals`
- `datastar-execute-script`

### Integration with Node HTTP

```ts
import http from "node:http";
import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/node";

http
  .createServer(async (req, res) => {
    if (req.url === "/search" && req.method === "POST") {
      const reader = await ServerSentEventGenerator.readSignals(req);
      if (!reader.success) {
        res.writeHead(400).end(reader.error ?? "Bad Request");
        return;
      }

      const query = String((reader.signals as any)?.query ?? "").toLowerCase();
      const rows = ["apple", "apricot", "banana", "blueberry"]
        .filter((x) => x.includes(query))
        .map((x) => `<li>${x}</li>`)
        .join("");

      ServerSentEventGenerator.stream(req, res, (stream) => {
        stream.patchElements(`<ul id="search-results">${rows}</ul>`);
      });
      return;
    }

    res.writeHead(404).end();
  })
  .listen(3000);
```

### Integration with Deno/Bun (Web Runtime)

```ts
// Deno: import { ServerSentEventGenerator } from "npm:@starfederation/datastar-sdk/web";
// Bun:  import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/web";

import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/web";

export default {
  async fetch(request: Request): Promise<Response> {
    if (new URL(request.url).pathname !== "/counter/increment") {
      return new Response("Not Found", { status: 404 });
    }

    const reader = await ServerSentEventGenerator.readSignals(request);
    if (!reader.success) {
      return new Response(reader.error ?? "Invalid signals", { status: 400 });
    }

    const next = Number((reader.signals as any)?.count ?? 0) + 1;

    return ServerSentEventGenerator.stream(
      request,
      (stream) => {
        stream.patchSignals(JSON.stringify({ count: next }));
        stream.patchElements(`<p id="counter">Current count: ${next}</p>`);
      },
      { keepalive: false },
    );
  },
};
```

### Multi-Step SSE Response

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  stream.patchSignals(JSON.stringify({ loading: true }));

  // ...do work...

  stream.patchSignals(
    JSON.stringify({
      loading: false,
      saved: true,
      message: "Saved successfully",
    }),
  );

  stream.patchElements(
    '<div id="result" class="success">Saved successfully!</div>',
  );
});
```

### Error Handling

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  const email = String((reader.signals as any)?.email ?? "").trim();

  if (!email) {
    stream.patchSignals(
      JSON.stringify({
        loading: false,
        hasError: true,
        errorMessage: "Email is required",
      }),
    );

    stream.patchElements(
      '<div id="form-error" class="error">Email is required</div>',
    );
    return;
  }

  stream.patchSignals(JSON.stringify({ loading: false, hasError: false }));
  stream.patchElements('<div id="form-error"></div>');
});
```

### Server Setup

Use any server/framework that exposes request/response objects compatible with the selected runtime implementation.

Minimal Node page + action example:

```ts
import http from "node:http";
import { ServerSentEventGenerator } from "@starfederation/datastar-sdk/node";

const page = `<!doctype html>
<html>
  <head>
    <script type="module" src="https://cdn.jsdelivr.net/gh/starfederation/datastar@v1.0.0-RC.6/bundles/datastar.js"></script>
  </head>
  <body>
    <p id="counter">Current count: 0</p>
    <button data-on-click="@$count++" data-on-click__post="/counter/increment">Increment</button>
  </body>
</html>`;

http.createServer(async (req, res) => {
  if (req.url === "/" && req.method === "GET") {
    res.writeHead(200, { "content-type": "text/html" }).end(page);
    return;
  }

  if (req.url === "/counter/increment" && req.method === "POST") {
    const reader = await ServerSentEventGenerator.readSignals(req);
    if (!reader.success) {
      res.writeHead(400).end(reader.error ?? "Bad Request");
      return;
    }
    const next = Number((reader.signals as any)?.count ?? 0) + 1;
    ServerSentEventGenerator.stream(req, res, (stream) => {
      stream.patchElements(`<p id="counter">Current count: ${next}</p>`);
    });
    return;
  }

  res.writeHead(404).end();
}).listen(3000);
```

## Common Patterns & Best Practices

### Form Submission

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  const s = (reader.signals ?? {}) as {
    name?: string;
    email?: string;
    message?: string;
  };

  if (!s.name || !s.email || !s.message) {
    stream.patchSignals(JSON.stringify({ loading: false }));
    stream.patchElements(
      '<div id="contact-status" class="error">All fields are required.</div>',
    );
    return;
  }

  stream.patchSignals(
    JSON.stringify({
      loading: false,
      form: { name: "", email: "", message: "" },
    }),
  );

  stream.patchElements(
    '<div id="contact-status" class="success">Message sent.</div>',
  );
});
```

### Live Search with Debouncing

```ts
const query = String((reader.signals as any)?.query ?? "").trim().toLowerCase();

ServerSentEventGenerator.stream(req, res, (stream) => {
  if (query.length < 2) {
    stream.patchElements('<ul id="search-results"></ul>');
    return;
  }

  const items = ["apple", "apricot", "banana", "blueberry"]
    .filter((v) => v.includes(query))
    .map((v) => `<li>${v}</li>`)
    .join("");

  stream.patchElements(`<ul id="search-results">${items}</ul>`);
});
```

### Infinite Scroll / Load More

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  const page = Number((reader.signals as any)?.page ?? 0);
  const next = page + 1;

  stream.patchElements(
    `<li>Item ${next * 2 - 1}</li><li>Item ${next * 2}</li>`,
    { selector: "#feed", mode: "append" },
  );

  stream.patchSignals(JSON.stringify({ page: next }));
});
```

### Signal Naming & Organization

- Keep signal keys in `camelCase`.
- Parse and validate incoming signals before using them.
- Prefer typed local shapes (`type`/`interface`) over unchecked `any`.
- Use explicit defaults when reading optional signals.

### Loading Indicators

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  stream.patchSignals(JSON.stringify({ loading: true }));

  // ...work...

  stream.patchSignals(JSON.stringify({ loading: false }));
});
```

### Optimistic Updates

Pattern:

1. Update local UI signal immediately in frontend expression.
2. Post action to backend.
3. Reconcile with backend source of truth via `patchSignals` and `patchElements`.
4. Roll back on error.

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  const rollback = { post: { liked: false, likesCount: 10 } };
  stream.patchSignals(JSON.stringify(rollback));
  stream.patchElements('<span id="likes-count">10</span>');
});
```

### Error Handling Pattern

Use one predictable error region and one structured error signal set.

```ts
function sendError(stream: any, message: string) {
  stream.patchSignals(
    JSON.stringify({
      loading: false,
      hasError: true,
      errorMessage: message,
    }),
  );
  stream.patchElements(
    `<div id="global-error" role="alert" class="error">${message}</div>`,
  );
}
```

### Keep Expressions Simple

Keep `data-*` expressions small and deterministic.
Move validation, authorization, and business logic to backend handlers.

## Debugging

### Using `data-json-signals`

```html
<pre data-json-signals></pre>
```

### Datastar Inspector

Use the Datastar inspector/extension to inspect:

- active signals
- stream events
- patch operations

### Console Debugging

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  stream.executeScript("console.debug('Datastar TS stream event')");
});
```

## Common Gotchas

### 1. Signal Casing

Keep server field names aligned with client signal names (`camelCase`).
Normalize once at request boundaries.

### 2. Actions Require `@` Prefix

Datastar action expressions must use action syntax (for example `@post('/endpoint')`).

### 3. Multi-statement Expressions Need Semicolons

Separate statements in `data-*` expressions with semicolons.

### 4. Element IDs for Morphing/Replacement

When patching by HTML, keep stable element IDs so updates target the intended DOM nodes.

### 5. SSE Headers and Streaming Behavior

Do not let proxies buffer SSE endpoints.
Keep streaming semantics (`text/event-stream`) intact.

### 6. Signal Types and Binding

Coerce and validate types on the backend (`number`, `boolean`, etc.) to avoid drift.

### 7. File Upload Base64 Encoding

File signals may include base64 payloads.
Validate MIME type and size before processing.

### 8. Underscore-prefixed Signals

Underscore-prefixed signals are typically client-local by convention.
Do not rely on them server-side unless explicitly configured.

### 9. Request Cancellation

Rapid interactions can cancel in-flight requests.
Keep handlers idempotent and safe to retry.

### 10. Empty vs Missing Signals

Treat missing and empty values differently when business rules depend on that distinction.

## Advanced Patterns

### Polling with Auto-retry

- Trigger periodic backend actions from frontend.
- Patch only changed regions/signals.
- Send lightweight heartbeat/status updates if needed.

### View Transitions

Use `patchElements(..., { useViewTransition: true })` for supported transitions.
Keep patched regions narrow and IDs stable.

### Modal Dialog Pattern

- Store modal state in signals.
- Return both signal and modal-body updates in one stream.
- Keep validation errors inside the modal patch target.

### Tabs Pattern

- Keep `activeTab` as a signal.
- Patch only the tab-panel container on tab change.
- Avoid replacing the full page for small tab updates.

### Toast Notifications

```ts
ServerSentEventGenerator.stream(req, res, (stream) => {
  stream.patchElements(
    '<div id="toast-123" class="toast">Saved!</div>',
    { selector: "#toasts", mode: "append" },
  );

  stream.removeElements("#toast-123");
});
```

## Reference Links

- [Datastar TypeScript SDK repo](https://github.com/starfederation/datastar-typescript)
- [Datastar TypeScript package](https://www.npmjs.com/package/@starfederation/datastar-sdk)
- [Datastar docs/site](https://data-star.dev)

## Version Info

- This reference targets the TypeScript SDK API documented in the official `datastar-typescript` repository README and npm package.
