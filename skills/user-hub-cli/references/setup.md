# Setting up a starter app

`user-hub setup` (run as `bun run setup`) wires an app made with `bun create user-hub@latest`: it signs the folder in when needed, picks the event, creates the Convex project, registers the app, sets `HUB_ISSUER` and `HUB_JWKS` on Convex (and on production once the app has a production URL), writes `VITE_HUB_URL` to `.env.local`, and pushes the functions. It is safe to run again.

It saves the app's `hub`, `event` and `name` to `./user-hub.json`. Commit that file; it holds nothing secret. The app's URLs live in the hub: a list of development URLs (`user-hub dev`) and a production URL (`user-hub prod`).

## Commands

```sh
bun run setup                              # in a terminal it asks for the event
bun run setup --event <slug>               # without a terminal: ask the human which event, then pass it
bun run setup --unlisted                   # keep it out of the badge app list (--listed puts it back)
bun run setup --name "<name>"              # the app's name in the hub (default: the package name)
bun run setup --hub-moved                  # the hub's URL changed: same app on the new hub
bunx user-hub prod set https://<deployed>  # once deployed, so badge scans can reach the app
bunx user-hub dev add http://localhost:<port> # another place the hub may open the app for developers
```

## What to know

- **Own registration vs a new one.** The app's key comes from its event and name. With the hub, event and name in `user-hub.json`, setup updates that registration. On a different hub the same event and name are someone else's app: setup refuses (exit code 3) unless the human confirms the hub moved, then pass `--hub-moved` (in a terminal it asks). Another `--event` or `--name` registers a **new** app with a new key and leaves the old one as it was: do that only when the human asks to move or rename the app.
- **A taken name** (exit code 3) needs another `--name` from the human. **An unknown event** (exit code 5) needs the right slug from the human.
- **The development URLs** start with `http://localhost:` plus the port in `package.json`'s `dev` script. When that port changes, add the new URL with `user-hub dev add` (and remove the old one). See [events-and-apps.md](events-and-apps.md).
- **Not a starter app?** Setup refuses (exit code 2). Register with `user-hub apps register` instead ([events-and-apps.md](events-and-apps.md)) and wire the code with the user-hub-auth skill.
- Setup does not deploy: after `user-hub prod set`, the human (or you, if asked) runs `bunx convex deploy` and deploys the frontend.
