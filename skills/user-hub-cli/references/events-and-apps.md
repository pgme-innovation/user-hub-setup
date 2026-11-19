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
bunx user-hub apps update <app> [--name "<name>"] [--prod <url>] [--dev <url>] [--listed|--unlisted]
bunx user-hub apps delete <app> --yes
```

### register

- The key is `<event slug>-<slugified name>`. The same event and name update the same app.
- `--new` refuses a taken name (exit code 3) so you never take over another app. Use it for a first registration; leave it out when updating this app's own registration.
- A URL left out keeps the registered one. A new app needs `--prod`, `--dev` or both. Leave out `--prod` until the app is deployed.
- New apps are listed in the badge app list (shown when the event's badges go to the app list, for apps with a production URL). `--unlisted` keeps one out; neither flag keeps the current setting.
- It prints the key, `HUB_ISSUER`, `HUB_JWKS` and `VITE_HUB_URL` for wiring the app (user-hub-auth skill).
- Any account can update any app: register only under this app's own name.

### update

Changes the name, URLs or listing; the **key never changes**. `--prod ""` or `--dev ""` removes that URL. The hub refuses to remove the production URL of an app that is live or scheduled.

### delete

Removes the app and its schedule slots, and clears it as the live app. Apps using its key stop accepting sign-ins. Only when the human asks.
