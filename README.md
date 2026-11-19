# User Hub setup

> **Private beta.** User Hub is not public yet. This is for teams already running one.

Agent skills for [User Hub](https://www.npmjs.com/package/@user-hub/auth): people scan their printed badge and land in your app already signed in.

| Skill | What your coding agent does with it |
|---|---|
| [`user-hub-auth`](./skills/user-hub-auth/SKILL.md) | Connects an app: installs [`@user-hub/auth`](https://www.npmjs.com/package/@user-hub/auth), registers the app, wires sign-in and facilitator pages, takes it to production |
| [`user-hub-cli`](./skills/user-hub-cli/SKILL.md) | Runs the hub from the terminal with the [`user-hub`](https://www.npmjs.com/package/user-hub) CLI: signs a folder in, sets up a starter app, manages an event's apps and people (one at a time or from a CSV), badges and sessions |

Each skill is a short `SKILL.md` with `references/` the agent reads only when it needs them.

## Start a new app

The quickest way: a TanStack Start + Convex app with sign-in, sign-out, a facilitator page, `CLAUDE.md` and both skills already wired.

```sh
bun create user-hub@latest my-app
```

It asks for your hub's URL, then signs the new app in to the hub: paste an API key from your hub's **Account** page (it shows as stars), or press Enter to sign in with your browser. It saves `HUB_URL` and `HUB_API_KEY` to the app's `.env.local`. Then:

```sh
cd my-app
bun run setup    # user-hub setup: event, Convex and the hub, once
bun run dev      # http://localhost:3000
```

Scripts and agents skip the questions when `HUB_URL` and `HUB_API_KEY` are already in the environment. If another app already has your app's name in the hub, `bun run setup` asks you for a new one.

## Add it to an existing app

### Install the skills

```sh
bunx skills add pgme-innovation/user-hub-setup
```

It finds both skills (add `--skill user-hub-auth` or `--skill user-hub-cli` for just one). Then ask your agent to connect your app to User Hub.

### What you need

Ask the hub's admin for its URL and an account. Then sign your app's folder in to the hub with the `user-hub` CLI:

```sh
bunx user-hub login --hub https://hub.example.com
```

Paste an API key from your hub's **Account** page, or press Enter to sign in with your browser. It saves `HUB_URL` and `HUB_API_KEY` to the folder's `.env.local` (keep it gitignored). Your agent can also run `bunx user-hub login --browser` and show you a link: open it, sign in to the hub and approve. The agent never asks for an API key or password in chat, never writes one anywhere itself and never prints `.env.local`.

### What the agent does

1. `bun add @user-hub/auth` and `bun add -d user-hub`
2. `bunx user-hub apps register --event <slug> --name "<app name>" --new --dev http://localhost:3000` (and `--prod <url>` once deployed). New apps show in the hub's badge app list once they have a production URL; `--unlisted` keeps one out of it
3. Sets `HUB_ISSUER` and `HUB_JWKS` on your Convex deployments and `VITE_HUB_URL` in your frontend env
4. Wires `hubAuth`, `ConvexHubAuthProvider` and `HubSignedIn`, with facilitator-only pages where needed

Open your app on its dev server and it sends you to the hub (sign in with your account) to pick who to be, then back signed in.

## Manage an event from the terminal

With the folder signed in, the same CLI (or your agent, following `user-hub-cli`) does what the hub's People and Apps pages do:

```sh
bunx user-hub events list
bunx user-hub people create --file people.csv --event <slug>   # name,team,role,expiry
bunx user-hub people list --badges
bunx user-hub apps list
bunx user-hub --help
```
