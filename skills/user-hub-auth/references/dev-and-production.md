# Local development, production and a moved hub

## Your dev server

Run the app on the development URL registered with the hub. With no token, it sends the browser to the hub, where the human (signed in to the hub once per browser) picks who to be, and comes back signed in. That lasts across reloads until the person's session in the hub ends (4 hours unless set otherwise).

- **Switch person:** open `<hub>/open/<key>`.
- **Another port:** the hub only sends tokens to registered URLs. Starter apps: `bun run setup --dev http://localhost:<port>` and change the port in the `dev` script too. Others: `bunx user-hub apps register` again with the same event, `--name` and the new `--dev`.
- **Test people:** `bunx user-hub people create --name "Test Person" --team Test` (user-hub-cli skill).
- This redirect only happens on local addresses (localhost, 127.x, 10.x, 172.16-31.x, 192.168.x, *.local). In production a person without a token sees "Scan your badge to sign in".

## Production URL

Badge scans reach the app only once the hub knows its production URL. Ask the human for the deployed URL (never guess), then:

- **Starter apps:** `bun run setup --prod https://<deployed url>`. It also sets `HUB_ISSUER` and `HUB_JWKS` on the production Convex deployment.
- **Other apps:** `bunx user-hub apps register --event <slug> --name "<the app's name>" --prod https://<deployed url>` without `--new`, then `bunx convex env set --prod HUB_ISSUER '…'` and `… HUB_JWKS '…'`.

Then `bunx convex deploy`, set `VITE_CONVEX_URL` and `VITE_HUB_URL` on the host, and tell the human the app can be scheduled or forced live under **Schedule** of its event. While the event's badges go to the app list, a listed app with a production URL appears there straight away.

## The hub moved to a new URL

Hub ids stay the same, so data keyed on the id carries over, but the app rejects every sign-in until it is updated.

1. Sign the folder in to the new hub: `bunx user-hub login --hub <new url>`.
2. **Starter apps:** `bun run setup --hub-moved` (it updates the same app on the new hub; add `--event <slug>` if the new hub has no event with the saved slug), then redeploy with `just push`: `VITE_HUB_URL` is built into the frontend.
3. **Other apps:**
   - `bunx user-hub apps register --event <slug> --name "<the app's name>"` with no `--new` and no URLs: it keeps the registered URLs and prints the new values. The printed key must equal the `app` key in the code; if it differs, or the hub asks for a URL, stop and ask the human.
   - Set `HUB_ISSUER` and `HUB_JWKS` on every Convex deployment (with and without `--prod`). No deploy needed for that.
   - Set `VITE_HUB_URL` to the new URL, then rebuild and redeploy the frontend.
