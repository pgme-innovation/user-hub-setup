---
name: user-hub-auth
description: Sign people into an app from their User Hub badge, and run a hub event's people and apps from the terminal. Use when an app needs User Hub auth (workshop badges, QR sign-in, hubAuth, ConvexHubAuthProvider, HubAuthProvider, HubSignedIn, useHubUser, verifyHubToken), when asked to connect an app to the hub, or for the user-hub CLI (user-hub login, setup, events, apps, people). Never hand-roll login, passwords or token checks for these apps.
---

# User Hub auth

People at a workshop scan a printed QR badge. One hub runs many workshops, called **events**; each person and each app belongs to one. The hub sends them, signed in, to whichever app is live in their event (or shows that event's app list so they pick one), with a token in the URL fragment (`#token=…`) that only that app accepts. The app gets the same token either way and needs nothing extra for the app list. `@user-hub/auth` turns that token into a signed-in person. The app keeps its own data, keyed on the person's hub id.

Two packages:

- `@user-hub/auth` (a dependency): the sign-in code that runs in the app.
- `user-hub` (a devDependency, or `bunx user-hub …` anywhere): the CLI. It signs a folder in to the hub, registers apps, and manages an event's people and apps. Run `bunx user-hub --help` (or `<command> --help`) for every option.

## Starting a new app

If there is no app yet, don't build one by hand: scaffold one with sign-in already wired (TanStack Start + Convex, sign-in and sign-out, a facilitator page, `CLAUDE.md` and this skill):

```sh
bun create user-hub@latest my-app
```

In a terminal it asks for the hub's URL, then signs the new app in (`user-hub login`: paste an API key, or approve a sign-in in the browser). As an agent you never handle an API key, so either ask the human to run it, or run it without a terminal with the hub's URL (`bun create user-hub@latest my-app --hub-url https://hub.example.com`) and then sign the new app in from its folder as described in **Signing in** below.

Then:

```sh
cd my-app
bun run setup    # user-hub setup: event, Convex project, hub registration, HUB_ISSUER and HUB_JWKS
bun run dev      # http://localhost:3000
```

In a terminal, `bun run setup` asks which event the app is for (or creates one). Without a terminal (as an agent), ask the human which event (**Before you start**) and pass it: `bun run setup --event <slug>`. Setup saves the app's event, name and URLs to `user-hub.json`: commit it.

That covers every step below; read on only to add User Hub to an existing app or to change the wiring.

## The hub's URL

The hub has one address, for example `https://hub.example.com`. The human gives it to you; you need it in three places, all with the same value:

- `HUB_URL` in the app's `.env.local`, next to the API key: `user-hub login` puts it there (**Signing in**).
- `HUB_ISSUER` on the app's Convex deployment: `user-hub apps register` prints it for you (**Set the two values**).
- `VITE_HUB_URL` in the app's frontend env, for production builds too: lets the dev server send you to the hub to sign in, and limits token refresh to that hub (**Wire it up**).

## Before you start: ask the human

You need five things. Ask for them; never guess.

1. **Which event** the app is for. Run `bunx user-hub events list` once the folder is signed in (question 4) to list the events and their slugs, and let the human pick one, or create one only when they ask for it (`bunx user-hub events create --name "<event name>" --time-zone <IANA zone>`; ask the human for the time zone, such as `Europe/Berlin`: it defaults to this machine's, which in a sandbox is often UTC). If the app has a `user-hub.json` with an `event`, that is the app's event: check it with the human only when asked to move the app.
2. **The app's name** in the hub, unique to this app within the event (the same event and name update the same app, and apps copied from a template often share a package name).
3. **Its local URL** (usually `http://localhost:3000`) and **its production URL** if it is deployed. Without a production URL the app can be developed but not scheduled or shown in the app list, so badge scans never reach it; add it later.
4. **Whether the app's folder is signed in to the hub.** Check it yourself with `bunx user-hub whoami` (it prints the account, or exits with code 4 when not signed in). If it is not, see **Signing in**. If the human keeps `HUB_URL` and `HUB_API_KEY` in a secret manager (Infisical, say), run every `user-hub` command through it instead (`infisical run -- bunx user-hub …`).
5. **Which Convex deployments** the app uses (dev, prod), so the values go on each.

## Signing in

The CLI acts as a hub account through an API key, `HUB_API_KEY`, saved in the app folder's `.env.local` next to `HUB_URL`. You never ask for an API key or password in chat, never write one into a file yourself, never commit one and never print `.env.local`. The key is never a command-line flag.

Sign the folder in one of two ways, from the app's folder:

- **The human runs** `bunx user-hub login --hub https://<hub url>` in a terminal. It asks them to paste an API key from `<hub>/account` or press Enter to sign in with the browser.
- **You run** `bunx user-hub login --hub https://<hub url> --browser`. It prints a link and a code, then waits (up to 10 minutes) until the request is approved: run it in the background or with a long timeout, read the link from its output and give it to the human to open, sign in to the hub and approve. Never open or approve the link yourself. When it is approved, the command creates an API key for itself, saves it and prints `Signed in to <hub> as <username> (<role>)`.

The hub's URL comes from the human (or `HUB_URL` already in `.env.local`, which `--hub` can then be left out for). If login fails because the request was denied or expired, run it again or ask the human. `bunx user-hub logout` removes the key from the folder.

## How the CLI behaves

- **The event:** `apps` and `people` commands work on one event: `--event <slug>`, or `event` in `./user-hub.json`.
- **Output:** pass `--json` and parse stdout: one JSON value, the result or `{ "error": { "code", "message", "hint" } }`. Progress goes to stderr.
- **No terminal:** it never asks; it fails with exit code 2 and names the flag to pass. Destructive commands (`apps delete`, `people delete`, `people badge --regenerate`, `people revoke`) need `--yes`: run them only when the human asked for exactly that.
- **Exit codes:** 0 done, 1 error (the message says why; nothing to work around), 2 usage, 3 name or slug taken, 4 not signed in or the key was refused (sign in again: **Signing in**), 5 unknown event, person or app, 130 cancelled.
- Always quote a value with spaces (`--name "Summer Camp"`).

## Steps

1. **Install:**

   ```sh
   bun add @user-hub/auth
   bun add -d user-hub
   ```

2. **Register the app** in its event from the app's repo root, once the folder is signed in (question 4 above):

   ```sh
   bunx user-hub apps register --event <slug> --name "<app name>" --new --prod https://app.example.com --dev http://localhost:3000
   ```

   If it exits with code 5, no event has that slug: run `bunx user-hub events list`, check the slug with the human and run it again; never create an event unless the human asks.

   `--new` makes sure you never take over another app: if the name is already taken, nothing changes and it exits with code 3. The same code 3 comes without `--new` when an app of another event holds the key. Then ask the human for another name and run it again. Drop `--new` only when updating this app's own registration later.

   Leave out `--prod` if the app is not deployed yet. Running it again with the same event and `--name` updates the URLs; a URL left out keeps the registered one, and with no URL flags it keeps the registered URLs and just prints the values. Use the exact name the app has in the hub: the hub finds the app by the key made from the event's slug and the name. Any hub account can update any app, so register only under this app's own name, never another app's.

   New apps are **listed** in the badge app list by default. When whoever runs the hub sets **Badges go to** to the app list, a scan shows the listed apps that have a production URL and the person picks one. Pass `--unlisted` to keep one out of it (it can still be scheduled or forced live), or `--listed` to show it again (in a starter app, pass them to `bun run setup`). Pass neither when updating and the hub's current setting stays; the output shows it.

   It prints the app **key** (`<event slug>-<name>`), `HUB_ISSUER` and `HUB_JWKS` (server side) and `VITE_HUB_URL` (frontend); with `--json`, as `key`, `env` and `hubUrl`.

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
- If the dev server runs on another port, register the new `--dev` URL (starter apps: `bun run setup --dev <url>`; others: `bunx user-hub apps register` with the same event and `--name`): the hub only sends tokens back to registered URLs.
- Need test people? `bunx user-hub people create --name "Test Person" --team Test` (see **The event's people**).
- This only happens on local addresses (localhost, 127.x, 10.x, 172.16-31.x, 192.168.x, *.local). In production a person without a token sees "Scan your badge to sign in".

## The event's people

The same CLI does what the hub's People page does. Run these only when the human asks.

```sh
bunx user-hub people list [--team <team>] [--role member|facilitator]
bunx user-hub people create --name "Ada" --team Blue --role facilitator --expiry 2h
bunx user-hub people create --file people.csv            # or a JSON array; "-" reads stdin
bunx user-hub people update <person> --team Red          # --move-to <event slug> moves them
bunx user-hub people delete <person>... --yes
bunx user-hub people badge <person> [--regenerate --yes]
bunx user-hub people revoke <person> --yes               # signs them out of every app
```

- `<person>` is an id from `people list` or their exact name; an ambiguous name fails with the ids to choose from.
- `--expiry` is how long a scan keeps them signed in: `1h`, `2h`, `4h` (default) or `never`. A file has a header row with any of `name`, `team`, `role` and `expiry`; empty values take the defaults, and at most 500 people go in at once, all or none.
- **Badge links are lasting sign-in links.** `people create` and `people badge` print them, and `people list --badges` adds them. Give them only to the human who asked, never put them in code, commits or logs, and never use one to sign in yourself.
- The event's apps: `bunx user-hub apps list`, `apps update <app> [--name] [--prod <url>] [--dev <url>] [--listed|--unlisted]` (`""` removes a URL; the key never changes) and `apps delete <app> --yes`.

## Production URL

Badge scans only reach an app once the hub knows its production URL. When the human asks to set up the production URL (or says the app is deployed), ask for the deployed URL (never guess), then:

- **Starter apps** (their `setup` script is `user-hub setup`): `bun run setup --prod https://<deployed url>`. It updates the app's own registration and sets `HUB_ISSUER` and `HUB_JWKS` on the production Convex deployment too.
- **Other apps:** `bunx user-hub apps register --event <slug> --name "<the app's name in the hub>" --prod https://<deployed url>`, without `--new` (this updates the app's own registration; ask the human for the event and name if you do not know them). Then set the two printed values on production: `bunx convex env set --prod HUB_ISSUER '…'` and `bunx convex env set --prod HUB_JWKS '…'`.

Then:

1. `bunx convex deploy`.
2. On the host, set `VITE_CONVEX_URL` (the production deployment's URL) and `VITE_HUB_URL` (the hub's URL).
3. Tell the human the app is ready to be scheduled (or forced live) under **Schedule** of its event in the hub; until then badge scans go to whichever app is live in that event. While **Badges go to** shows the app list, a listed app appears there as soon as it has its production URL.

## Hub moved to a new URL

When the human says the hub's URL changed, hub ids stay the same, so data keyed on the id carries over. Until the app is updated it rejects every sign-in.

- **Starter apps:** sign the folder in to the new hub with `bunx user-hub login --hub <new url>` (**Signing in**), then run `bun run setup` (with `--event <slug>` when the new hub has no event with the app's saved slug). It updates the app's own registration and sets `HUB_ISSUER`, `HUB_JWKS` and `VITE_HUB_URL` again (on the production Convex deployment too, once the app has a production URL). Then redeploy with `just push`: Vite bakes `VITE_HUB_URL` in at build time.
- **Other apps:**
  1. Sign the app's folder in to the new hub with `bunx user-hub login --hub <new url>` (**Signing in**); it replaces `HUB_URL` and `HUB_API_KEY` in `.env.local`. If the human keeps them in a secret manager, ask them to change them there.
  2. Run `bunx user-hub apps register --event <slug> --name "<the app's name in the hub>"` without `--new` or any URL flags: it keeps the registered URLs and prints the new values. Use the event's slug and the exact name from the event's **Apps** page in the hub. The printed key must equal the `app` key already in the code; if it differs, or the hub says to give a URL (no app has that name), stop and ask the human: you found another app, or none.
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

- Key all data on the hub id (`identity.subject` / `useHubUser().id`). Never on the name, never on the badge token, and never on Convex's `identity.tokenIdentifier`, which includes the hub's URL and changes if the hub moves.
- Check access on the server: `requireFacilitator(ctx)` / `requireHubUser(ctx)` in Convex, or `verifyHubToken` elsewhere. `useHubUser()` is for display only.
- The provider refreshes tokens with the hub (`<hub>/api/token/refresh`) while the app is open; the hub decides how long a session lasts and when it ends, and the provider then shows the signed-out screen. Do not refresh, store or extend tokens yourself. Read `useHubToken()` for each request, since it changes. If the app sets a Content Security Policy, allow the hub in `connect-src`.
- Do not add other sign-in methods, sessions or token handling. If something is missing, change `@user-hub/auth` instead.
