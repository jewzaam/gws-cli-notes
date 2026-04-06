# People API

## Get Person Info

```bash
gws people people get --params '{"resourceName": "people/<NUMERIC_ID>", "personFields": "names,emailAddresses"}'
```

- `resourceName`: `people/me` for authenticated user, `people/<numeric_id>` for a specific person
- `personFields`: comma-separated list of fields to return (e.g., `names`, `emailAddresses`, `photos`)
- Response: `names[0].displayName`, `emailAddresses[0].value`, `resourceName`
- The Meet API uses `users/<id>` prefix while People API uses `people/<id>` — the numeric ID is the same
- meet-enrich uses this to resolve Meet participant IDs to email addresses and display names

## Get Authenticated User

```bash
gws people people get --params '{"resourceName": "people/me", "personFields": "names,emailAddresses"}'
```

- Returns the authenticated user's `resourceName` (e.g., `people/12345`)
- Used to identify the current user in Meet participant lists (convert `people/` → `users/` prefix)
