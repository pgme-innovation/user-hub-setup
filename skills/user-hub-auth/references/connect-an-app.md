# Connect an existing app

Five steps, in order. The `user-hub …` commands are explained in the **user-hub-cli** skill.

## 1. Install

```sh
bun add @user-hub/auth
bun add -d user-hub
```

## 2. Register the app

From the app's folder, once it is signed in (`bunx user-hub whoami`; if it exits with code 4, sign in as the user-hub-cli skill describes):

```sh
bunx user-hub apps register --event <slug> --name "<app name>" --new --dev http://localhost:3000 [--prod https://app.example.com]
```

- `--new` never takes over another app: a taken name exits with code 3 and changes nothing. Ask the human for another name. Drop `--new` only when updating this app's own registration later.
- Leave out `--prod` until the app is deployed; then set it with `bunx user-hub prod set <url> --event <slug> --app "<app name>"`. An app has a list of development URLs: add others with `bunx user-hub dev add <url> --event <slug> --app "<app name>"`.
- It prints the app **key** (`<event slug>-<name>`), `HUB_ISSUER`, `HUB_JWKS` and `VITE_HUB_URL`. With `--json`: `key`, `env`, `hubUrl`.

## 3. Set the values

- Convex, on every deployment the app uses: `bunx convex env set HUB_ISSUER '…'` and `bunx convex env set HUB_JWKS '…'` (add `--prod` for production).
- Other servers: the same two names in their environment.
- Frontend env: `VITE_HUB_URL` (the hub's URL), for production builds too.

## 4. Wire it up

`convex/auth.config.ts`:

```ts
import type { AuthConfig } from "convex/server";
import { hubAuth } from "@user-hub/auth/convex";

export default { providers: [hubAuth({ app: "<key>" })] } satisfies AuthConfig;
```

The provider replaces `ConvexProvider` (same `ConvexReactClient`), **inside the root document's `<body>`**: in TanStack Start the root route's shell component (`src/routes/__root.tsx`), in a Vite SPA around `<App />`, in Next.js inside `<body>` of the root layout.

```tsx
import { ConvexHubAuthProvider, HubSignedIn } from "@user-hub/auth/convex-react";

<body>
  <ConvexHubAuthProvider client={convex} app="<key>" hubUrl={import.meta.env.VITE_HUB_URL}>
    <HubSignedIn>{children}</HubSignedIn>
  </ConvexHubAuthProvider>
  <Scripts />
</body>
```

`<HubSignedIn>` shows its children only to a signed-in person and "Scan your badge to sign in" otherwise; style that with `signedOut={…}` and the first render with `loading={…}`. Anything outside it (a public landing page) shows to everyone.

**Without Convex:** `HubAuthProvider` and `HubSignedIn` from `@user-hub/auth/react`, same place, same props minus `client`.

**Render signed in on the server (TanStack Start).** Without this, every page load or refresh shows `loading` until the page hydrates. Read the token in the root route and pass it as `initialToken`:

```tsx
import { createServerFn } from "@tanstack/react-start";
import { getHubToken } from "@user-hub/auth/start";

const fetchHubToken = createServerFn({ method: "GET" }).handler(() =>
  getHubToken({ app: "<key>", hubUrl: import.meta.env.VITE_HUB_URL }),
);

// In createRootRoute / createRootRouteWithContext (server only: the browser already has the token):
beforeLoad: async () => ({ hubToken: import.meta.env.SSR ? await fetchHubToken() : null }),

// In the shell component:
const { hubToken } = Route.useRouteContext();
<ConvexHubAuthProvider client={convex} app="<key>" hubUrl={import.meta.env.VITE_HUB_URL} initialToken={hubToken}>
```

With `ConvexQueryClient` (Convex through TanStack Query), also call `context.convexQueryClient.serverHttpClient?.setAuth(hubToken)` in `beforeLoad` when there is a token, so server-side queries run as the person.

With Convex, create the client so queries wait for the person's token and keep it, instead of running signed out first and then signing in twice: `new ConvexReactClient(url, { expectAuth: true, initialAuthTokenReuse: true })`, or the same options as the second argument of `new ConvexQueryClient(url, …)`. On Vite, also pre-bundle Convex and this package together, so the browser holds one `convex/react` ("Could not find Convex client" otherwise): `optimizeDeps: { include: ["convex/react", "@user-hub/auth/convex-react"] }`, plus `"@convex-dev/react-query"` when you use it. `ConvexHubAuthProvider` reloads the page when the person signs out (here, in another tab, or by the hub), as Convex recommends with `expectAuth`: the new page starts signed out, so no cached query runs again without the token. Never read or set the `user-hub-token.<key>` cookie yourself: the provider writes it, and `getHubToken` and the server-function middleware read it.

## 5. Use the person

| Where | How |
|---|---|
| Components | `const me = useHubUser()` → `{ id, name, team, role }` or `null` |
| Convex functions | `const me = await requireHubUser(ctx)` (throws when signed out); `getHubUser(ctx)` when signing in is optional |
| TanStack Start server functions and routes | `hubConvexMiddleware` (or `hubMiddleware`): [server-functions.md](server-functions.md) |
| Other servers | Send `useHubToken()` as `Authorization: Bearer <token>`; check it with `await verifyHubToken(request, { app: "<key>", hubUrl })` from `@user-hub/auth/server` (`hubUrl` is the issuer when the server has no `HUB_ISSUER`) |
| Sign out | `const signOut = useHubSignOut()` from the provider's entry point. It forgets this app's token only |

If the app sets a Content Security Policy, allow the hub in `connect-src`: the provider calls `<hub>/api/token/refresh`.
