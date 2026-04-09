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
- JSON output includes: `thread_id`, `message_id`, `references`, `from` (name+email), `to`, `cc`, `subject`, `date`, `body_text`, `body_html`
- Handles multipart/alternative and base64 decoding automatically
- Converts HTML-only messages to plain text

## Triage Helper (Bulk Metadata)

```bash
gws gmail +triage --max 500 --query 'newer_than:7d' --labels --format json
```

- One call returns metadata for up to N messages matching the query
- Without `--labels`: returns `id`, `from`, `subject`, `date` per message
- With `--labels`: adds `labels` array (label IDs like `UNREAD`, `Label_2`, `STARRED`, etc.)
- Does NOT return `threadId` — use `messages.list` for thread grouping
- Does NOT support pagination beyond `--max` (max observed: 500)
- Output shape: `{"messages": [...], "query": "...", "resultSizeEstimate": N}`

### Optimal Bulk Email Fetch Strategy

For fetching email with bodies + labels + thread grouping:

1. **`messages.list`** (paginated) — gets `id` + `threadId` for all messages in the window
2. **`+triage --labels`** (one call) — gets current labels for all messages in the window, replaces per-message metadata refresh
3. **`+read`** (parallel, new messages only) — gets full body + headers for messages not yet cached

This avoids per-message `format=metadata` calls for label refresh. `+triage` does it in bulk.

### Format Comparison

| Format | Fields | Use case |
|--------|--------|----------|
| `minimal` | id, threadId, labelIds, snippet, internalDate | Cheapest per-message call for label refresh |
| `metadata` | id, threadId, labelIds, snippet, headers (specified) | Per-message call with specific headers |
| `full` | Everything including MIME body | Full message fetch, body is base64 |
| `+triage` | id, from, subject, date, labels (optional) | Bulk metadata, one call for many messages |
| `+read` | thread_id, from, to, cc, subject, date, body_text, body_html | Full body with parsed text, one message at a time |
