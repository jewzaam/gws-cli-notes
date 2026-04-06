# Gmail

## Search Messages

```bash
gws gmail users messages list --params '{"userId": "me", "q": "is:unread newer_than:7d (ANSTRAT OR nexus)", "maxResults": 20}'
```

- `q` uses Gmail search syntax — same operators as the Gmail search bar (`is:unread`, `from:`, `subject:`, `newer_than:`, `OR`, parentheses for grouping)
- Returns message IDs only — fetch headers or full content separately
- `maxResults` max is 500; use `nextPageToken` for pagination

## Get Message Headers (Metadata Only)

```bash
gws gmail users messages get --params '{"userId": "me", "id": "<MESSAGE_ID>", "format": "metadata", "metadataHeaders": ["Subject", "Date", "From", "To", "Cc"]}'
```

- `format: "metadata"` returns only headers specified in `metadataHeaders` — lightweight
- Headers are in `payload.headers[]` as `{name, value}` pairs

## Get Full Message Content

```bash
gws gmail users messages get --params '{"userId": "me", "id": "<MESSAGE_ID>", "format": "full"}'
```

- Returns full MIME structure including body content
- Body is base64url-encoded in `payload.body.data` or nested in `payload.parts[].body.data`
- `format` options: `minimal` (IDs only), `metadata` (headers), `full` (everything), `raw` (RFC 2822)

## Read Helper (Shortcut)

```bash
gws gmail +read --id <MESSAGE_ID> --headers --format json
```

- Convenience wrapper — `--headers` returns just headers without the full MIME payload
