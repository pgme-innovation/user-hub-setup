---
name: user-hub-auth
description: Sign people into an app from their printed User Hub badge with @user-hub/auth. Use when connecting an app to a User Hub, adding badge or QR sign-in, wiring hubAuth, ConvexHubAuthProvider, HubAuthProvider, HubSignedIn, useHubUser or verifyHubToken, building facilitator-only pages, or moving an app to a new hub URL or to production. Use it even when the user only says "connect this to the hub" or "add sign-in for the workshop". Never hand-roll login, passwords or token checks for these apps.
---

# User Hub auth

People at a workshop scan a printed QR badge. One hub runs many workshops, called **events**; every person and every app belongs to one. The scan sends the person, signed in, to the app that is live in their event (or to the event's app list), with a token in the URL fragment (`#token=…`) that only that app accepts. `@user-hub/auth` turns that token into a signed-in person: `{ id, name, team, role }`. The app keeps its own data, keyed on the person's hub id.

Registering apps, signing a folder in to the hub and managing people is done with the `user-hub` CLI: follow the **user-hub-cli** skill for any `user-hub …` command.

## Pick the path

| Situation | Do this |
|---|---|
| No app yet | Scaffold one: `bun create user-hub@latest my-app`, then `bun run setup` and `bun run dev` in it. It comes wired; read on only to change the wiring |
| Existing app, not connected | Follow [references/connect-an-app.md](references/connect-an-app.md): install, register, set the values, wire the provider, use the person |
| Facilitator-only pages or actions | [references/roles.md](references/roles.md) |
| Running it locally, going to production, or the hub moved | [references/dev-and-production.md](references/dev-and-production.md) |

A starter app is one whose `package.json` has `"setup": "user-hub setup"`: for it, prefer `bun run setup` (with `--event`, `--prod` or `--dev`) over doing the steps by hand.

## Ask the human first

Never guess these; ask:

1. **Which event** the app is for (the CLI lists them). Create an event only when the human asks.
2. **The app's name** in the hub, unique within the event.
3. **Its local URL** (usually `http://localhost:3000`) and its **production URL** once deployed.
4. **Which Convex deployments** it uses (dev, prod), so the values go on each.

The hub's URL also comes from the human. It goes in three places with the same value: `HUB_URL` (saved by `user-hub login`), `HUB_ISSUER` on Convex, and `VITE_HUB_URL` in the frontend env.

## Rules

- **Key data on the hub id** (`useHubUser().id`, `identity.subject` in Convex). Never on the name, the badge, or Convex's `tokenIdentifier`, which contains the hub's URL and changes if the hub moves.
- **Check access on the server**: `requireHubUser(ctx)` / `requireFacilitator(ctx)` in Convex, `verifyHubToken` elsewhere. `useHubUser()` is for display; hiding a page protects nothing.
- **Mount the provider inside `<body>`**, around the page content, never outside `<html>` or in the router's `Wrap`. It always renders its children; `<HubSignedIn>` does the gating.
- **Server rendering (TanStack Start):** pass `getHubToken()` from `@user-hub/auth/start` to the provider as `initialToken` from the root route's `beforeLoad` ([references/connect-an-app.md](references/connect-an-app.md)), so a page load renders the person instead of `loading`. Everything inside `<HubSignedIn>` then renders on the server: keep `window` and `document` inside effects. With Convex, create the client with `expectAuth: true` and `initialAuthTokenReuse: true`.
- **Let the provider own the token.** It refreshes it with the hub while the app is open and signs the person out when the hub ends the session. Do not store, refresh or extend tokens yourself; read `useHubToken()` for each request.
- **No other sign-in.** No login screens, passwords or sessions. If something is missing, it belongs in `@user-hub/auth`.
- **Secrets stay out of chat and files you write.** The API key lives only in `.env.local`, saved by `user-hub login`.
