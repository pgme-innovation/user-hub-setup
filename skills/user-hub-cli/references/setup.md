# Setting up a starter app

`user-hub setup` (run as `bun run setup`) wires an app made with `bun create user-hub@latest`: it signs the folder in when needed, picks the event, creates the Convex project, registers the app, sets `HUB_ISSUER` and `HUB_JWKS` on Convex (and on production once there is a production URL), writes `VITE_HUB_URL` to `.env.local`, and pushes the functions. It is safe to run again.

It saves the app's `event`, `name`, `devUrl` and `prodUrl` to `./user-hub.json`. Commit that file; it holds nothing secret.

## Commands

```sh
bun run setup                              # in a terminal it asks for the event and the production URL
bun run setup --event <slug>               # without a terminal: ask the human which event, then pass it
bun run setup --prod https://<deployed>    # once deployed, so badge scans can reach the app
bun run setup --dev http://localhost:<port> # where the hub opens the app for developers
bun run setup --unlisted                   # keep it out of the badge app list (--listed puts it back)
bun run setup --name "<name>"              # the app's name in the hub (default: the package name)
```

## What to know

- **Own registration vs a new one.** The app's key comes from its event and name. With the event and name in `user-hub.json`, setup updates that registration. Another `--event` or `--name` registers a **new** app with a new key and leaves the old one as it was: do that only when the human asks to move or rename the app.
- **A taken name** (exit code 3) needs another `--name` from the human. **An unknown event** (exit code 5) needs the right slug from the human.
- **The development URL** starts as `http://localhost:` plus the port in `package.json`'s `dev` script. Change both together.
- **Not a starter app?** Setup refuses (exit code 2). Register with `user-hub apps register` instead ([events-and-apps.md](events-and-apps.md)) and wire the code with the user-hub-auth skill.
- Setup does not deploy: after `--prod`, the human (or you, if asked) runs `bunx convex deploy` and deploys the frontend.
