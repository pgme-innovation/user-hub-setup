# Signing in

The CLI acts as a hub account through an API key, `HUB_API_KEY`, saved in the folder's `.env.local` next to `HUB_URL` (mode 600, keep it gitignored). The hub's URL comes from the human, or from `HUB_URL` already in `.env.local` (then `--hub` can be left out).

## Check

```sh
bunx user-hub whoami          # "Signed in to <hub> as <username> (<role>)", or exit code 4
```

## Sign in

Pick one:

- **The human runs it** in a terminal: `bunx user-hub login --hub https://<hub url>`. It asks them to paste an API key from `<hub>/account`, or press Enter to sign in with the browser.
- **You run it** without a terminal: `bunx user-hub login --hub https://<hub url> --browser`. It prints a link and a code, then waits up to 10 minutes. Run it in the background or with a long timeout, give the link to the human to open and approve, and wait. Never open or approve the link yourself. When approved it makes its own API key, saves it and prints `Signed in to …`.
- **A script with a key** from a secret manager: pipe it in with `--api-key-stdin`.

If the request was denied or expired, run it again or ask the human. If the human keeps `HUB_URL` and `HUB_API_KEY` in a secret manager (Infisical, say), run every command through it: `infisical run -- bunx user-hub …`.

## Sign out

```sh
bunx user-hub logout          # removes HUB_API_KEY from .env.local, keeps HUB_URL
```

The key itself stays valid until it is deleted on `<hub>/account`; tell the human if it should stop working.
