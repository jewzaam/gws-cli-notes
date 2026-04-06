# Docs

## Read a Google Doc

```bash
gws docs documents get --params '{"documentId":"DOC_ID","includeTabsContent":true}' 2>/dev/null > output.json
```

- `includeTabsContent:true` is required to get all tabs; without it only first tab is in `doc.body.content`
- With it, content is in `doc.tabs[N].documentTab.body.content`
- Document ID is the string between `/d/` and `/edit` in the Google Docs URL
- `-o` flag is for binary responses only; use shell redirect for JSON

## Suggestions

Add `suggestionsViewMode` query param:

- `SUGGESTIONS_INLINE` (default) - raw doc with `suggestedInsertionIds`/`suggestedDeletionIds` on textRun elements
- `PREVIEW_SUGGESTIONS_ACCEPTED` - doc as if all suggestions accepted
- `PREVIEW_WITHOUT_SUGGESTIONS` - doc with all suggestions rejected (original text)

Example:

```bash
gws docs documents get --params '{"documentId":"DOC_ID","includeTabsContent":true,"suggestionsViewMode":"SUGGESTIONS_INLINE"}' 2>/dev/null > output.json
```

Finding suggestions in the JSON: look for `suggestedInsertionIds` (new text) and `suggestedDeletionIds` (removed text) on `textRun` elements. Same suggestion ID on an insertion and deletion = a replacement.
