# User Hub setup

> **Private beta.** User Hub is not public yet. This is for teams already running one.

An agent skill that connects an app to [User Hub](https://www.npmjs.com/package/@user-hub/auth): people scan their printed badge and land in your app already signed in. Your coding agent installs [`@user-hub/auth`](https://www.npmjs.com/package/@user-hub/auth), registers the app with the hub and wires up sign-in, following the skill.

## Install the skill

```sh
bunx skills add pgme-innovation/user-hub-setup
```

Then ask your agent to connect the app to User Hub.

## What you need

Ask the hub's admin for its URL and the admin password, and put both in your app's `.env` or `.env.local` (keep it gitignored):

```sh
HUB_URL=https://hub.example.com
HUB_ADMIN_PASSWORD=…
```

The agent never asks for the password in chat and never writes it anywhere itself.

## What the agent does

1. `bun add @user-hub/auth`
2. `bunx hub-user register --name "<app name>" --dev http://localhost:3000` (and `--prod <url>` once deployed)
3. Sets `HUB_ISSUER` and `HUB_JWKS` on your Convex deployments and `VITE_HUB_URL` in your frontend env
4. Wires `hubAuth`, `ConvexHubAuthProvider` and `HubSignedIn`, with facilitator-only pages where needed

Open your app on its dev server and it sends you to the hub to pick who to be, then back signed in.
