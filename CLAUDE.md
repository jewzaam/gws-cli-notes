# gws-cli-notes

Reference notes for using the `gws` CLI with Google Workspace APIs.

## Structure

- `NOTES.md` — index and general topics (auth, scopes, pagination). Read this first.
- `docs/` — per-service usage notes. Read the relevant service doc when working with a specific API.

## Key patterns

- `--params` carries URL/query parameters, `--json` carries request bodies
- `--format` is available on all commands: json (default), table, yaml, csv
- `-o` / `--output` is for binary responses only; use shell redirect (`> file.json`) for JSON
- Always paginate `list` calls — check for `nextPageToken` in responses
