# Events and apps

## Events

```sh
bunx user-hub events list [--active]
bunx user-hub events create --name "<name>" --time-zone <IANA zone>
```

Create an event only when the human asks, and ask for its time zone (such as `Europe/Berlin`): it defaults to this machine's, which in a sandbox is often UTC. The slug comes from the name and never changes; a taken slug is exit code 3.

## Apps

Every command works on one event (`--event <slug>` or `./user-hub.json`). `<app>` is an app key or its name.

```sh
bunx user-hub apps list
bunx user-hub apps register --name "<name>" --new --dev http://localhost:3000 [--prod <url>] [--listed|--unlisted]
bunx user-hub apps update <app> [--name "<name>"] [--listed|--unlisted]
bunx user-hub apps delete <app> --yes
```

An app has a list of development URLs (where the hub opens it for developers; any number) and at most one production URL (where badge scans send people). It always keeps at least one of them. `user-hub dev` and `user-hub prod` manage them.

### register

- The key is `<event slug>-<slugified name>`. The same event and name update the same app.
- `--new` refuses a taken name (exit code 3) so you never take over another app. Use it for a first registration; leave it out when updating this app's own registration.
- A URL left out keeps the registered one. A new app needs `--prod`, `--dev` or both. Leave out `--prod` until the app is deployed.
- It never overwrites a URL the app already has: the hub refuses (exit code 2) and says to use `user-hub dev` or `user-hub prod`. Do that instead; don't retry.
- New apps are listed in the badge app list (shown when the event's badges go to the app list, for apps with a production URL). `--unlisted` keeps one out; neither flag keeps the current setting.
- It prints the key, `HUB_ISSUER`, `HUB_JWKS` and `VITE_HUB_URL` for wiring the app (user-hub-auth skill).
- Any account can update any app: register only under this app's own name.

### update

Changes the name or listing; the **key never changes**. URLs are `user-hub dev` and `user-hub prod`.

### delete

Removes the app and its schedule slots, and clears it as the live app. Apps using its key stop accepting sign-ins. Only when the human asks.

## Development and production URLs

They work on this folder's app (its event and `name` in `./user-hub.json`); `--app <key or name>` targets another app, `--event <slug>` another event. With another event or hub than the saved ones, `--app` is required (exit code 2 without it).

```sh
bunx user-hub dev list                       # the development URLs
bunx user-hub dev add http://localhost:3001  # https://, or http:// on a local address; one already there changes nothing
bunx user-hub dev remove <url>               # exit code 5 when it is not there
bunx user-hub prod                           # the production URL
bunx user-hub prod set https://<deployed>    # set or replace it
bunx user-hub prod remove                    # badge scans stop reaching the app
```

- `prod set` on this folder's own app also sets `HUB_ISSUER` and `HUB_JWKS` on the production Convex deployment, as setup does; then the human (or you, if asked) runs `bunx convex deploy` and deploys the frontend. Ask the human for the deployed URL; never guess it.
- The hub refuses to remove the production URL of an app that is live or scheduled, and to remove an app's last URL (the last development URL without a production URL, or the production URL without development URLs).
