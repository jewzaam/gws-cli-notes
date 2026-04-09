# gws-cli-notes

Reference notes for using the `gws` CLI with Google Workspace APIs. Per-service details are in [docs/](docs/).

## Services

- [Calendar](docs/calendar.md) — agenda, events, RSVP, meeting artifacts, working location
- [Docs](docs/docs.md) — read documents, suggestions, write (batchUpdate, tabs, text)
- [Docs API Content Structure](docs/docs-api-content-structure.md) — JSON structure, element types, markdown conversion mapping
- [Drive](docs/drive.md) — comments, file listing, export, downloads, org-policy restrictions
- [Gmail](docs/gmail.md) — search, message headers, full content, triage (bulk metadata), format comparison
- [Meet](docs/meet.md) — conference records, recordings, transcripts, participants
- [People API](docs/people.md) — resolve user IDs, get authenticated user
- [Sheets](docs/sheets.md) — read spreadsheets, list tabs, cell formatting/colors
- [Slides](docs/slides.md) — read presentations, PDF export

## Key patterns

- `--params` carries URL/query parameters, `--json` carries request bodies
- `--format` is available on all commands: json (default), table, yaml, csv
- `-o` / `--output` is for binary responses only; use shell redirect (`> file.json`) for JSON
- Always paginate `list` calls — check for `nextPageToken` in responses

## GCP Project Setup

Create a new GCP project (e.g., under "Default Projects" in the org). Do not reuse an existing project. Then run `gws auth setup` (requires `gcloud` CLI) which walks through OAuth consent screen and client credential creation. APIs are enabled on-demand (first call to a service prompts with an enablement URL).

## Authentication

```bash
gws auth logout
gws auth login --readonly -s docs,drive,calendar,sheets,slides,meet
```

- Scopes were added incrementally: `docs,drive,calendar` -> `+sheets` -> `+slides` -> `+meet`
- After adding a new scope, must logout and re-login for the token to include it
- API enablement errors include URLs to enable the API on the GCP project (one-time action per API)

## Schema Lookup

Inspect the Google API discovery schema for any service, resource, or method — read-only, no auth required, no API calls to user data.

```bash
gws schema <service.resource.method> [--resolve-refs]
```

Examples:
```bash
gws schema drive.files.list
gws schema calendar.events.list
gws schema meet.conferenceRecords.participants
```

- Path must be at least `service.Message` or `service.resource.method`
- Returns parameter names, types, descriptions, required/optional, enum values
- `--resolve-refs` inlines referenced schema types instead of showing `$ref` pointers
- Useful for discovering available query parameters, field names, and filter syntax without reading Google's web docs

## Auth Status

### Check Token and Scopes

```bash
gws auth status
```

- Returns JSON with `user` (email), `scopes`, `token_valid` (boolean), `scope_count`, `storage`, `enabled_apis`, and more
- Useful for programmatic checks — meet-enrich uses this to resolve the authenticated user's email
- No `--params` or `--format` needed

## Scope Requirements

| Service | Operation | Scope URL | Login flag |
|---------|-----------|-----------|------------|
| Calendar | Read events, settings | `https://www.googleapis.com/auth/calendar.readonly` | `-s calendar` |
| Calendar | RSVP / patch events (PA dashboard) | `https://www.googleapis.com/auth/calendar.events` | requires `--scopes` (see below) |
| Docs | Read content, suggestions | `https://www.googleapis.com/auth/documents.readonly` | `-s docs` |
| Docs | Write (batchUpdate: insert text, add/delete tabs, style) | `https://www.googleapis.com/auth/documents` | requires `--scopes` |
| Drive | Comments, file metadata, downloads | `https://www.googleapis.com/auth/drive.readonly` | `-s drive` |
| Slides | Read presentations | `https://www.googleapis.com/auth/presentations.readonly` | `-s slides` |
| Sheets | Read spreadsheets | `https://www.googleapis.com/auth/spreadsheets.readonly` | `-s sheets` |
| Meet | Conference records, recordings, transcripts, participants | `https://www.googleapis.com/auth/meetings.space.readonly` | `-s meet` |
| Gmail | Read messages | `https://www.googleapis.com/auth/gmail.readonly` | `-s gmail` |
| People | Get self (`people/me`) | `https://www.googleapis.com/auth/userinfo.profile` | auto-included |
| People | Resolve other users (`people/<id>`) | `https://www.googleapis.com/auth/directory.readonly` | requires `--scopes` |
| Meet | Smart notes | unavailable via OAuth | n/a |

### Login with all scopes (PA dashboard + meet-enrich)

The PA dashboard requires `calendar.events` (write) for RSVP. The `-s calendar` flag only grants `calendar.readonly`. To get both read and write, use explicit `--scopes`:

```bash
gws auth logout
gws auth login --scopes https://www.googleapis.com/auth/calendar.events,https://www.googleapis.com/auth/calendar.readonly,https://www.googleapis.com/auth/documents.readonly,https://www.googleapis.com/auth/drive.readonly,https://www.googleapis.com/auth/gmail.readonly,https://www.googleapis.com/auth/meetings.space.readonly,https://www.googleapis.com/auth/presentations.readonly,https://www.googleapis.com/auth/spreadsheets.readonly,https://www.googleapis.com/auth/directory.readonly
```

The `--readonly` + `-s` shorthand flags don't support write scopes. Use `--scopes` with full URLs when any write scope is needed.

Must logout first — cached token won't pick up new scopes without a fresh login.

## Pagination

Most Google API `list` endpoints return a maximum number of results per page (e.g., Calendar events default to 250, Drive files default to 100). When more results exist, the response includes a `nextPageToken` field.

### Manual Pagination

Pass `pageToken` in subsequent requests:

```python
all_items = []
page_token = None
while True:
    params = {<base params>}
    if page_token:
        params["pageToken"] = page_token
    response = run_gws(..., params=params)
    all_items.extend(response.get("items", []))
    page_token = response.get("nextPageToken")
    if not page_token:
        break
```

### `--page-all` Flag

Some `gws` commands support `--page-all` to automatically fetch all pages. Check if available before relying on it — not all endpoints support it.

### Known Page Size Defaults

| API | Endpoint | Default `maxResults` / `pageSize` |
|-----|----------|----------------------------------|
| Calendar | `events list` | 250 |
| Drive | `files list` | 100 (max 1000) |
| Gmail | `messages list` | 100 (max 500) |
| Meet | `conferenceRecords list` | 25 |
| Meet | `transcripts entries list` | 100 |

**Gotcha**: Getting exactly the default page size back (e.g., 250 calendar events) is a strong signal that results were truncated. Always paginate `list` calls in production code.
