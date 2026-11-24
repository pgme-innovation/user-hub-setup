# Server functions and server routes (TanStack Start)

For server code that acts as the signed-in person: a server function that calls another service and saves to Convex, a streaming chat route, an upload. Never hand-roll this (reading the header, `verifyHubToken`, a `ConvexHttpClient`): the middleware below does it, including the production container that has no runtime environment.

## 1. Create the middleware once

`src/lib/hub.ts`:

```ts
import { hubConvexMiddleware } from "@user-hub/auth/start-convex";

import { APP_KEY } from "../../convex/hub";

export const hub = hubConvexMiddleware({
  app: APP_KEY,
  hubUrl: import.meta.env.VITE_HUB_URL,
  convexUrl: import.meta.env.VITE_CONVEX_URL,
});
```

Without Convex: `hubMiddleware({ app, hubUrl })` from `@user-hub/auth/start-middleware`, the same minus `convexUrl` and `context.convex`.

Pass `hubUrl` and `convexUrl` from `import.meta.env` as above. Never `process.env.VITE_*`: Vite only builds `import.meta.env` values in, and a production container may have no runtime environment at all. The middleware checks tokens against `HUB_ISSUER` and `HUB_JWKS` when the server has them, else against `hubUrl` and its published key.

## 2. Server functions

```ts
import { createServerFn } from "@tanstack/react-start";

import { hub } from "@/lib/hub";
import { api } from "../../convex/_generated/api";

export const saveSummary = createServerFn({ method: "POST" })
  .middleware([hub.serverFn])
  .validator((data: { text: string }) => data)
  .handler(async ({ context, data }) => {
    // context.hub: { id, name, team, role }; context.hubToken: their token
    // context.convex: a ConvexHttpClient acting as them
    await context.convex.mutation(api.summaries.save, { text: data.text });
  });
```

Call it like any server function (`await saveSummary({ data })`), from anywhere inside the provider: event handlers, effects, route loaders and queries (`useSuspenseQuery` included), on the first page load too. The browser half sends the person's current token, refreshed first when it is about to expire (a call made before the provider is ready, during the page's first render, waits for it): no wrapper hook, no headers at the call site. While the server renders a page (a loader or query run there) there is no browser to send a token, so it uses the person's cookie, as `getHubToken` does. Without a valid token the handler never runs and the call throws "Scan your badge to sign in.".

## 3. Server routes

Server routes are plain HTTP, so the caller sends the token itself (`useHubToken()`, read for each request):

```ts
export const Route = createFileRoute("/api/chat")({
  server: {
    middleware: [hub.request],
    handlers: {
      POST: async ({ context, request }) => {
        // context.hub, context.hubToken, context.convex as above
      },
    },
  },
});
```

```ts
fetch("/api/chat", { method: "POST", headers: { Authorization: `Bearer ${token}` }, body });
```

Without a valid token it answers 401 "Scan your badge to sign in.".

## Rules

- **Convex still checks.** Keep `requireHubUser(ctx)` / `requireFacilitator(ctx)` in every Convex function `context.convex` calls: the middleware proves who is asking, Convex decides what they may do. Check roles on `context.hub.role` for anything outside Convex.
- **Every server function or route that touches private data** gets `hub.serverFn` or `hub.request`. A route guard (`beforeLoad`) is not protection: server functions are reachable on their own.
- **CSRF:** Start protects server functions from cross-site calls by default. An app that defines `src/start.ts` must add `createCsrfMiddleware` itself (see TanStack Start's server functions guide).
- **Server routes take only the header**, never the cookie: a server route is an API anyone can call. Server functions take the cookie only while the server renders a page; a call from the browser always needs the token the middleware sends.
- **Types:** `HubConvexContext` (`context.hub`, `context.hubToken`, `context.convex`) from `@user-hub/auth/start-convex`, `HubAuthContext` from `@user-hub/auth/start-middleware`, for helpers that take the handler's context.
- **Create the middleware once** and import it; do not call `hubConvexMiddleware` per function.
