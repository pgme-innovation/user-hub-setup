# User Hub setup

> **Private beta.** User Hub is not public yet. This is for teams already running one.

An agent skill that connects an app to [User Hub](https://www.npmjs.com/package/@user-hub/auth): people scan their printed badge and land in your app already signed in. Your coding agent installs [`@user-hub/auth`](https://www.npmjs.com/package/@user-hub/auth), registers the app with the hub and wires up sign-in, following the skill.

## Start a new app

The quickest way: a TanStack Start + Convex app with sign-in, sign-out, a facilitator page, `CLAUDE.md` and this skill already wired.

```sh
bun create @user-hub my-app
```

It asks for your hub's URL and admin password (the password shows as stars) and saves them to the app's `.env.local`. Then:

```sh
cd my-app
bun run setup    # connects Convex and the hub, once
bun run dev      # http://localhost:3000
```

Scripts and agents skip the questions with `--hub-url` and `--admin-password`, or with `HUB_URL` and `HUB_ADMIN_PASSWORD` already in the environment.

## Add it to an existing app

### Install the skill

```sh
bunx skills add pgme-innovation/user-hub-setup
```

Then ask your agent to connect your app to User Hub.

### What you need

Ask the hub's admin for its URL and the admin password, and put both in your app's `.env` or `.env.local` (keep it gitignored):

```sh
HUB_URL=https://hub.example.com
HUB_ADMIN_PASSWORD=…
```

The agent never asks for the password in chat and never writes it anywhere itself.

### What the agent does

1. `bun add @user-hub/auth`
2. `bunx hub-user register --name "<app name>" --dev http://localhost:3000` (and `--prod <url>` once deployed)
3. Sets `HUB_ISSUER` and `HUB_JWKS` on your Convex deployments and `VITE_HUB_URL` in your frontend env
4. Wires `hubAuth`, `ConvexHubAuthProvider` and `HubSignedIn`, with facilitator-only pages where needed

Open your app on its dev server and it sends you to the hub to pick who to be, then back signed in.
