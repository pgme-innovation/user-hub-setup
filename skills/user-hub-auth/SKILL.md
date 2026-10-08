---
name: user-hub-auth
description: Sign people into an app from their User Hub badge. Use when an app needs User Hub auth (workshop badges, QR sign-in, hubAuth, ConvexHubAuthProvider, HubAuthProvider, HubSignedIn, useHubUser, verifyHubToken, hub-user register), or when asked to connect an app to the hub. Never hand-roll login, passwords or token checks for these apps.
---

# User Hub auth

People at a workshop scan a printed QR badge. The hub sends them to whichever app is live, signed in, with a token in the URL fragment (`#token=…`) that only that app accepts. This package turns that token into a signed-in person. The app keeps its own data, keyed on the person's hub id.

## Starting a new app

If there is no app yet, don't build one by hand: scaffold one with sign-in already wired (TanStack Start + Convex, sign-in and sign-out, a facilitator page, `CLAUDE.md` and this skill):

```sh
bun create @user-hub my-app
```

In a terminal it asks for the hub's URL and admin password and saves them to the new app's `.env.local`. As an agent you never handle the password, so either ask the human to run it, or run it only when `HUB_URL` and `HUB_ADMIN_PASSWORD` are already in the environment (for example `infisical run -- bun create @user-hub my-app`).

Then:

```sh
cd my-app
bun run setup    # Convex project, hub registration, HUB_ISSUER and HUB_JWKS, once
bun run dev      # http://localhost:3000
```

That covers every step below; read on only to add User Hub to an existing app or to change the wiring.

## The hub's URL

The hub has one address, for example `https://hub.example.com`. The human gives it to you; you need it in three places, all with the same value:

- `HUB_URL` in the app's `.env` or `.env.local`: read by `hub-user register` (**Register the app**). The human puts it there with the admin password.
- `HUB_ISSUER` on the app's Convex deployment: `hub-user register` prints it for you (**Set the two values**).
- `VITE_HUB_URL` in the app's frontend env: lets the dev server send you to the hub to sign in (**Wire it up**).

## Before you start: ask the human

You need four things. Ask for them; never guess.

1. **The app's name** in the hub, unique to this app (the same name updates the same app, and apps copied from a template often share a package name).
2. **Its local URL** (usually `http://localhost:3000`) and **its production URL** if it is deployed. Without a production URL the app can be developed but not scheduled, so badge scans never reach it; add it later by running register again with `--prod`.
3. **Where `HUB_URL` and `HUB_ADMIN_PASSWORD` are.** You do not have the password, and you must never ask for it in chat, write it into a file yourself, commit it or print it. Ask the human which of these applies:
   - **`.env` or `.env.local`** in the app's repo (must be gitignored): the human adds `HUB_URL=…` and `HUB_ADMIN_PASSWORD=…` themselves. `bunx hub-user` reads `.env` and `.env.local` automatically.
   - **Infisical** (or another secret manager): run the command through it, for example `infisical run -- bunx hub-user register …`.
   - **The human runs the command** and pastes you only the output, which contains no secrets.
4. **Which Convex deployments** the app uses (dev, prod), so the values go on each.

## Steps

1. **Install:**

   ```sh
   bun add @user-hub/auth
   ```

2. **Register the app** from the app's repo root, with `HUB_URL` and the password supplied the way the human chose in question 3 above:

   ```sh
   bunx hub-user register --name "<app name>" --new --prod https://app.example.com --dev http://localhost:3000
   ```

   `--new` makes sure you never take over another app: if the name is already taken, nothing changes and it exits with code 3 and "already exists". Then ask the human for another name and run it again. Drop `--new` only when updating this app's own registration later.

   Leave out `--prod` if the app is not deployed yet. Running it again with the same `--name` updates the URLs; a URL left out keeps the registered one. If it fails with "set HUB_URL", "set HUB_ADMIN_PASSWORD" or a wrong password, stop and ask the human; do not work around it.

   It prints the app **key** (made from the name), `HUB_ISSUER` and `HUB_JWKS` (server side) and `VITE_HUB_URL` (frontend). Add `--json` to parse them.

3. **Set the two values** where tokens are checked:
   - Convex: `bunx convex env set HUB_ISSUER '…'` and `bunx convex env set HUB_JWKS '…'` (on every deployment the app uses).
   - Other servers: their environment, same names.

4. **Wire it up** with the key from **Register the app**.

   Convex, `convex/auth.config.ts`:

   ```ts
   import type { AuthConfig } from "convex/server";
   import { hubAuth } from "@user-hub/auth/convex";

   export default { providers: [hubAuth({ app: "<key>" })] } satisfies AuthConfig;
   ```

   The provider replaces `ConvexProvider` (keep the same `ConvexReactClient`). Set `VITE_HUB_URL` (the hub's URL) in the app's frontend env.

   **Where it goes:** inside the root document's `<body>`, around the page content. In TanStack Start that is the root route's shell component (`src/routes/__root.tsx`); in a Vite SPA, around `<App />`; in Next.js, inside `<body>` in the root layout. **Never** in the router's `Wrap`, or anywhere outside `<html>`.

   ```tsx
   import { ConvexHubAuthProvider, HubSignedIn } from "@user-hub/auth/convex-react";

   <body>
     <ConvexHubAuthProvider
       client={convex}
       app="<key>"
       hubUrl={import.meta.env.VITE_HUB_URL}
     >
       <HubSignedIn>{children}</HubSignedIn>
     </ConvexHubAuthProvider>
     <Scripts />
   </body>
   ```

   The provider always renders its children; `<HubSignedIn>` shows them only to a signed-in person. Without a valid token it shows "Scan your badge to sign in": pass `signedOut={…}` to style that screen and `loading={…}` for the first render. Anything outside `<HubSignedIn>` (a public landing page, say) shows to everyone. Apps without Convex use `HubAuthProvider` and `HubSignedIn` from `@user-hub/auth/react`, with the same props minus `client`.

5. **Use the person.**
   - In components: `const me = useHubUser()` gives `{ id, name, team, role }`.
   - In Convex functions: `const me = await requireHubUser(ctx)` gives the same `{ id, name, team, role }` (see Facilitator pages).
   - Not Convex: wrap the app in `HubAuthProvider` and `HubSignedIn` (`@user-hub/auth/react`), send `useHubToken()` as `Authorization: Bearer <token>`, and check it on the server with `await verifyHubToken(request, { app: "<key>" })` from `@user-hub/auth/server`.

## Your dev server

Run the app as usual (`bun run dev`) on the `--dev` URL you registered. Opening it with no token sends the browser to the hub (admin password once per browser), where you pick who to be; the hub sends you back signed in as them. The token is kept for 4 hours across reloads and restarts, then the same thing happens again.

- To switch person: open `<hub>/open/<key>` and pick someone else.
- If the dev server runs on another port, run `hub-user register` again with the same `--name` and the new `--dev` URL: the hub only sends tokens back to registered URLs.
- This only happens on local addresses (localhost, 127.x, 10.x, 172.16-31.x, 192.168.x, *.local). In production a person without a token sees "Scan your badge to sign in".

## Sign out

Use `useHubSignOut()` from the same entry point as the provider; never clear storage or tokens yourself.

```tsx
const signOut = useHubSignOut();
<button onClick={signOut}>Sign out</button>
```

It forgets this app's token only. In production the person then sees the signed-out screen and scans their badge again; on a local dev server the hub opens to pick another person.

## Facilitator pages

People are either `"member"` or `"facilitator"`, set in the hub. To give facilitators their own page or actions:

- **Protect the data on the server, always.** In Convex functions use the helpers from `@user-hub/auth/convex`:

  ```ts
  import { requireFacilitator, requireHubUser } from "@user-hub/auth/convex";

  export const closeVoting = mutation({
    args: {},
    handler: async (ctx) => {
      const facilitator = await requireFacilitator(ctx); // throws for members
      // …
    },
  });

  export const myVotes = query({
    args: {},
    handler: async (ctx) => {
      const me = await requireHubUser(ctx); // throws when signed out
      // query by me.id
    },
  });
  ```

  `getHubUser(ctx)` returns the person or null when signing in is optional. Not Convex: check `(await verifyHubToken(request, { app })).role`.

- **Then hide what members cannot use**, for a clean UI only:

  ```tsx
  const me = useHubUser();
  if (me?.role !== "facilitator") return <p>This page is for facilitators.</p>;
  ```

  A hidden page is not protection: members can still call the functions, which is why the server check comes first.

## Rules

- Key all data on the hub id (`identity.subject` / `useHubUser().id`). Never on the name, and never on the badge token, which the app never sees.
- Check access on the server: `requireFacilitator(ctx)` / `requireHubUser(ctx)` in Convex, or `verifyHubToken` elsewhere. `useHubUser()` is for display only.
- Tokens last 4 hours; the provider then shows the signed-out screen. Do not try to refresh them: people scan their badge again.
- Do not add other sign-in methods, sessions or token handling. If something is missing, change this package instead.
