---
name: user-hub-cli
description: Run a User Hub from the terminal with the user-hub CLI (bunx user-hub). Use for signing an app folder in to a hub (user-hub login, whoami), setting up a create-user-hub starter app (user-hub setup, bun run setup), listing or creating events, registering, updating or deleting apps, and listing, adding (one or from a CSV/JSON file), editing, moving or deleting an event's people, showing or regenerating badges, drawing a badge's QR code in the terminal, saving badge QR codes as SVG files to print (one, or every badge of an event with a badges.csv index) and signing people out. Use it whenever the user mentions the hub's events, people, attendees, badges or app registration, even without naming the CLI.
---

# user-hub CLI

`user-hub` does from the terminal what an account can do on the hub's pages: sign a folder in, wire a starter app, and run an event's apps and people. It is a devDependency of User Hub apps (`bunx user-hub …` there) and runs anywhere with `bunx user-hub …`. Wiring sign-in into app code is the **user-hub-auth** skill.

`bunx user-hub --help` and `bunx user-hub <command> --help` list every option.

## Before any command

1. **Is the folder signed in?** `bunx user-hub whoami`. Exit code 4 means no: see [references/signing-in.md](references/signing-in.md). Never ask for an API key in chat.
2. **Which event?** `apps` and `people` commands work on one event: `--event <slug>`, else `event` in `./user-hub.json` (written by `user-hub setup`). `bunx user-hub events list` shows the slugs. If neither is known, ask the human.

## What to read

| Task | Read |
|---|---|
| Sign in, sign out, check the account | [references/signing-in.md](references/signing-in.md) |
| Set up or update a starter app (`bun run setup`) | [references/setup.md](references/setup.md) |
| Events; register, list, update or delete apps | [references/events-and-apps.md](references/events-and-apps.md) |
| People: list, add, import a file, edit, move, delete, badges and their QR codes (terminal, SVG files to print), sign out | [references/people.md](references/people.md) |

## How it behaves

- **`--json`** prints one JSON value on stdout: the result, or `{ "error": { "code", "message", "hint" } }`. Progress goes to stderr. Parse the JSON instead of the text.
- **No terminal, no questions.** It fails with exit code 2 and names the flag to pass.
- **Destructive commands need `--yes`** without a terminal: `apps delete`, `people delete`, `people badge --regenerate`, `people revoke`, and saving badge files over existing ones (`people badge --save`/`-o`, `people badges`).
- **Exit codes:** 0 done · 1 error (read the message) · 2 usage · 3 name or slug taken · 4 not signed in or key refused · 5 unknown event, person or app · 130 cancelled.
- Quote values with spaces: `--name "Summer Camp"`.

## Rules

- **Act only on what the human asked.** Changing or deleting people, apps or events affects a live workshop. Never create an event, delete anything or regenerate a badge unless asked for exactly that.
- **Badge links are lasting sign-in links.** `people create`, `people badge`, `people list --badges` and `people badges` print them, and saved QR files (`badge-*.svg`, `badges.csv`) hold them. Give them only to the human who asked; never put them in code, commits or logs, never commit the files (save them outside the repo or gitignore them), and never use one yourself.
- **The API key never leaves `.env.local`.** Don't print `.env.local`, copy the key, or pass it as a flag (the CLI has none for it).
- **Don't work around a refusal.** Exit code 4 means sign in again; 3 means pick another name; 5 means check the slug, id or name with the human.
