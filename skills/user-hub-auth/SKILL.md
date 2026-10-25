---
name: user-hub-auth
description: Sign people into an app from their User Hub badge. Use when an app needs User Hub auth (workshop badges, QR sign-in, hubAuth, ConvexHubAuthProvider, HubAuthProvider, HubSignedIn, useHubUser, verifyHubToken, hub-user login, hub-user register), or when asked to connect an app to the hub. Never hand-roll login, passwords or token checks for these apps.
---

# User Hub auth

People at a workshop scan a printed QR badge. The hub sends them, signed in, to whichever app is live (or shows its app list so they pick one), with a token in the URL fragment (`#token=…`) that only that app accepts. The app gets the same token either way and needs nothing extra for the app list. This package turns that token into a signed-in person. The app keeps its own data, keyed on the person's hub id.

## Starting a new app

If there is no app yet, don't build one by hand: scaffold one with sign-in already wired (TanStack Start + Convex, sign-in and sign-out, a facilitator page, `CLAUDE.md` and this skill):

```sh
bun create user-hub@latest my-app
```

In a terminal it asks for the hub's URL, then signs the new app in (`hub-user login`: paste an API key, or approve a sign-in in the browser) and saves `HUB_URL` and `HUB_API_KEY` to the new app's `.env.local`. As an agent you never handle an API key, so either ask the human to run it, or run it without a terminal with the hub's URL (`bun create user-hub@latest my-app --hub-url https://hub.example.com`) and then sign the new app in from its folder as described in **Signing in** below.

Then:

```sh
cd my-app
bun run setup    # Convex project, hub registration, HUB_ISSUER and HUB_JWKS, once
bun run dev      # http://localhost:3000
```

That covers every step below; read on only to add User Hub to an existing app or to change the wiring.

## The hub's URL

The hub has one address, for example `https://hub.example.com`. The human gives it to you; you need it in three places, all with the same value:

- `HUB_URL` in the app's `.env.local`: read by `hub-user register` (**Register the app**). `hub-user login` puts it there, next to the API key (**Signing in**).
- `HUB_ISSUER` on the app's Convex deployment: `hub-user register` prints it for you (**Set the two values**).
- `VITE_HUB_URL` in the app's frontend env, for production builds too: lets the dev server send you to the hub to sign in, and limits token refresh to that hub (**Wire it up**).

## Before you start: ask the human

You need four things. Ask for them; never guess.

1. **The app's name** in the hub, unique to this app (the same name updates the same app, and apps copied from a template often share a package name).
2. **Its local URL** (usually `http://localhost:3000`) and **its production URL** if it is deployed. Without a production URL the app can be developed but not scheduled or shown in the app list, so badge scans never reach it; add it later by running register again with `--prod`.
3. **Whether the app's folder is signed in to the hub.** Check it yourself with `bunx hub-user whoami` (it prints the account, or exits with code 4 when not signed in). If it is not, see **Signing in**. If the human keeps `HUB_URL` and `HUB_API_KEY` in a secret manager (Infisical, say), run every `hub-user` command through it instead (`infisical run -- bunx hub-user register …`).
4. **Which Convex deployments** the app uses (dev, prod), so the values go on each.

## Signing in

`hub-user` acts as a hub account through an API key, `HUB_API_KEY`, saved in the app folder's `.env.local` next to `HUB_URL`. You never ask for an API key or password in chat, never write one into a file yourself, never commit one and never print `.env.local`. Sign the folder in one of two ways, from the app's folder:

- **The human runs** `bunx hub-user login --hub https://<hub url>` in a terminal. It asks them to paste an API key from `<hub>/account` (it shows as stars) or press Enter to sign in with the browser.
- **You run** `bunx hub-user login --hub https://<hub url> --browser`. It prints a link and a code, then waits (up to 10 minutes) until the request is approved: run it in the background or with a long timeout, read the link from its output and give it to the human to open, sign in to the hub and approve. Never open or approve the link yourself. When it is approved, the command creates an API key for itself, saves it and prints `Signed in to <hub> as <username> (<role>)`.

The hub's URL comes from the human (or `HUB_URL` already in `.env.local`, which `--hub` can then be left out for). If login fails because the request was denied or expired, run it again or ask the human. An older `.env.local` with `HUB_ADMIN_PASSWORD` is no longer used: ask the human to delete that line.

## Steps

1. **Install:**

   ```sh
   bun add @user-hub/auth
   ```

2. **Register the app** from the app's repo root, once the folder is signed in (question 3 above):

   ```sh
   bunx hub-user register --name "<app name>" --new --prod https://app.example.com --dev http://localhost:3000
   ```

   `--new` makes sure you never take over another app: if the name is already taken, nothing changes and it exits with code 3 and "already exists". Then ask the human for another name and run it again. Drop `--new` only when updating this app's own registration later.

   Leave out `--prod` if the app is not deployed yet. Running it again with the same `--name` updates the URLs; a URL left out keeps the registered one, and with no URL flags it keeps the registered URLs and just prints the values. Use the exact name the app has in the hub: the hub finds the app by the key made from the name and saves the name as you typed it. Updating with no URL flags needs `@user-hub/auth` 0.2.1 or newer (`bun add @user-hub/auth@latest`); older versions ask for `--prod` or `--dev`. If it exits with code 4, read the message: "not signed in" or "did not accept HUB_API_KEY" means sign in again (**Signing in**); "Ask a hub admin, or pick another name" means the app belongs to another account: ask the human for another name, or to have a hub admin change it. Do not work around either.

   New apps are **listed** in the badge app list by default. When the hub admin sets **Badges go to** to the app list, a scan shows the listed apps that have a production URL and the person picks one; a listed app without a production URL does not appear. Pass `--unlisted` to keep one out of it (it can still be scheduled or forced live), or `--listed` to show it again; in a starter app whose `scripts/setup.ts` mentions `--unlisted`, pass it to `bun run setup`. Both flags need `@user-hub/auth` 0.2.1 or newer (`bun add @user-hub/auth@latest`); older versions reject them. Pass neither when updating and the hub's current setting stays; the output shows it ("Listed in the badge app list: yes"). If it says the hub "did not apply" the flag, the hub is too old for the app list: tell the human; the rest of the registration went through.

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

Run the app as usual (`bun run dev`) on the `--dev` URL you registered. Opening it with no token sends the browser to the hub (the human signs in with their hub account once per browser), where you pick who to be; the hub sends you back signed in as them. You stay signed in across reloads and restarts until that person's expiration in the hub (4 hours unless set otherwise), then the same thing happens again.

- To switch person: open `<hub>/open/<key>` and pick someone else.
- If the dev server runs on another port, run `hub-user register` again with the same `--name` and the new `--dev` URL: the hub only sends tokens back to registered URLs.
- This only happens on local addresses (localhost, 127.x, 10.x, 172.16-31.x, 192.168.x, *.local). In production a person without a token sees "Scan your badge to sign in".

## Production URL

Badge scans only reach an app once the hub knows its production URL. When the human asks to set up the production URL (or says the app is deployed), ask for the deployed URL (never guess), then:

- **Starter apps** (they have `scripts/setup.ts`): `bun run setup --prod https://<deployed url>`. It updates the app's own registration and sets `HUB_ISSUER` and `HUB_JWKS` on the production Convex deployment too.
- **Other apps:** `bunx hub-user register --name "<the app's name in the hub>" --prod https://<deployed url>`, without `--new` (this updates the app's own registration; the name is `HUB_APP_NAME` in `.env.local`, or ask the human). Then set the two printed values on production: `bunx convex env set --prod HUB_ISSUER '…'` and `bunx convex env set --prod HUB_JWKS '…'`.

Then:

1. `bunx convex deploy`.
2. On the host, set `VITE_CONVEX_URL` (the production deployment's URL) and `VITE_HUB_URL` (the hub's URL).
3. Tell the human the app is ready for the hub admin to schedule (or force live) under **Schedule**; until then badge scans go to whichever app is live. While **Badges go to** shows the app list, a listed app appears there as soon as it has its production URL.

## Hub moved to a new URL

When the human says the hub's URL changed, hub ids stay the same, so data keyed on the id carries over. Until the app is updated it rejects every sign-in.

- **Starter apps** (they have `scripts/setup.ts`): sign the folder in to the new hub with `hub-user login --hub <new url>` (**Signing in**), then run `bun run setup`. It updates the app's own registration and sets `HUB_ISSUER`, `HUB_JWKS` and `VITE_HUB_URL` again (on the production Convex deployment too, once the app has a production URL). Then redeploy with `just push`: Vite bakes `VITE_HUB_URL` in at build time.
- **Other apps:**
  1. Sign the app's folder in to the new hub with `hub-user login --hub <new url>` (**Signing in**); it replaces `HUB_URL` and `HUB_API_KEY` in `.env.local`. If the human keeps them in a secret manager, ask them to change them there.
  2. Run `bunx hub-user register --name "<the app's name in the hub>"` without `--new` or any URL flags: it keeps the registered URLs and prints the new values. This needs `@user-hub/auth` 0.2.1 or newer: if it asks for `--prod` or `--dev`, run `bun add @user-hub/auth@latest` first. Use the exact name from the hub's **Apps** page: the CLI finds the app by the key made from the name. The printed key must equal the `app` key already in the code; if it differs, or the hub says to give a URL (no app has that name), stop and ask the human: you found another app, or none.
  3. Set `HUB_ISSUER` and `HUB_JWKS` on every Convex deployment the app uses (`bunx convex env set …`, and again with `--prod`). Convex re-reads `auth.config.ts` with the new values straight away; no deploy is needed.
  4. Set `VITE_HUB_URL` to the new URL in the frontend env and on the host, then rebuild and redeploy the frontend: Vite bakes it in at build time.

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

- Key all data on the hub id (`identity.subject` / `useHubUser().id`). Never on the name, never on the badge token, which the app never sees, and never on Convex's `identity.tokenIdentifier`, which includes the hub's URL and changes if the hub moves.
- Check access on the server: `requireFacilitator(ctx)` / `requireHubUser(ctx)` in Convex, or `verifyHubToken` elsewhere. `useHubUser()` is for display only.
- The provider refreshes tokens with the hub (`<hub>/api/token/refresh`) while the app is open; the hub decides how long a session lasts and when it ends, and the provider then shows the signed-out screen. Do not refresh, store or extend tokens yourself. Read `useHubToken()` for each request, since it changes. If the app sets a Content Security Policy, allow the hub in `connect-src`.
- Do not add other sign-in methods, sessions or token handling. If something is missing, change this package instead.
- `@user-hub/auth` 0.3 replaced the hub's admin password with accounts: `hub-user` no longer reads `HUB_ADMIN_PASSWORD` and needs `hub-user login` once per app folder. A `^0.2` range never installs 0.3: `bun add @user-hub/auth@latest`; no app code changes.
- An existing app on `@user-hub/auth` 0.1 does not refresh tokens, and a `^0.1` range never installs 0.2: update it with `bun add @user-hub/auth@latest`. No code changes are needed. The same command updates 0.2.0, whose `hub-user` has no `--listed`/`--unlisted` and needs a URL flag on every run (both need 0.2.1 or newer).
