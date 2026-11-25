# People and badges

Every command works on one event (`--event <slug>` or `./user-hub.json`). `<person>` is an id from `people list` or the person's exact name; a name shared by several people fails (exit code 2) and lists their ids to choose from.

| Field | Values | Default |
|---|---|---|
| `--name` | text, at most 200 characters, unique in the event | a generated name |
| `--team` | text, at most 200 characters | none |
| `--role` | `member`, `facilitator` | `member` |
| `--expiry` | how long a badge scan keeps them signed in: `1h`, `2h`, `4h`, `never` | `4h` |

## List

```sh
bunx user-hub people list [--team <team>] [--role member|facilitator] [--badges] [--json]
```

Newest first. `--badges` adds each person's badge link: only when the human asks for the links.

## Add

```sh
bunx user-hub people create --name "Ada Lovelace" --team Blue --role facilitator --expiry 2h
bunx user-hub people create --file people.csv [--team Blue] [--role member] [--expiry 4h]
cat people.json | bunx user-hub people create --file -
```

- Prints each new person's badge link.
- **A file** is a CSV with a header row, or a JSON array of objects, with any of `name`, `team`, `role`, `expiry`. Empty values take the defaults; with `--file`, `--team`, `--role` and `--expiry` fill in what a row leaves out (`--name` is not allowed with a file).
- At most 500 people at a time, all added or none, and at most 1,000 in an event. Every row is checked before anything is sent; an error names the row. A name someone in the event already has (or that appears twice in the file) is refused by the hub, and nothing is added.

```csv
name,team,role,expiry
Ada Lovelace,Blue,facilitator,
Grace Hopper,Blue,,
,Red,,2h
```

## Edit or move

```sh
bunx user-hub people update <person> [--name "<name>"] [--team "<team>"] [--role <role>] [--expiry <expiry>]
bunx user-hub people update <person> --move-to <event slug>
```

Fields left out keep their value. `--name ""` gives a generated name, `--team ""` removes the team. Moving keeps the badge working in the new event and ends their app sessions in the old one.

## Delete

```sh
bunx user-hub people delete <person>... --yes
```

Their badges stop working. Only when the human asks.

## Badges and sessions

```sh
bunx user-hub people badge <person>                       # show the badge link
bunx user-hub people badge <person> --regenerate --yes    # new badge: the printed one stops working
bunx user-hub people revoke <person> --yes                # sign out of every app; the badge keeps working
```

`--regenerate` also signs them out of every app. Use it when a badge is lost; use `revoke` when only the sessions should end. Both take effect within about 10 minutes.

## QR codes and printing

```sh
bunx user-hub people badge <person> --qr                  # draw the QR code in the terminal, under the link
bunx user-hub people badge <person> --save                # save ./badge-<name>.svg
bunx user-hub people badge <person> -o print/ada.svg      # a path ending in .svg is the file
bunx user-hub people badge <person> -o print              # anything else is a folder (made if needed)
bunx user-hub people badges [-o <folder>] [--team <team>] [--role <role>] [--yes]
```

- The SVGs are the same as the hub's People page downloads (margin 1, error correction M, `badge-<name>.svg`).
- `people badges` saves one SVG per person and `badges.csv` (`id,name,team,role,expiry,file,badgeUrl`) in `./badges-<event>` unless `-o` says where. People who would share a file name get their id added (`badge-sam-<id>.svg`). Nobody matching saves nothing. A CSV value starting with `=`, `+`, `-` or `@` gets a leading `'` so spreadsheets keep it as text.
- Replacing an existing file needs `--yes` without a terminal (exit code 2 names it). Ask the human before replacing.
- `--json`: `badge` never draws the QR code and adds `"file"` (absolute path) when it saved one; `badges` returns `{ event, folder, index, people: [{ ...person, badgeUrl, file }] }`.
- `--regenerate --save` saves the new badge.
- Save only when the human asks, outside the repo or in a gitignored folder: the files are sign-in links. Tell the human where they are; don't print the links back unless asked.
