# Slides

## Read a Presentation

```bash
gws slides presentations get --params '{"presentationId": "<PRESENTATION_ID>"}' --format json
```

- Presentation ID is the string between `/d/` and `/edit` in the Google Slides URL
- Response includes: title, presentationId, locale, pageSize, slides (with all pageElements, shapes, text, images, tables), layouts, masters
- Each slide has an `objectId` and array of `pageElements`
- Text content is nested: `pageElement.shape.text.textElements[].textRun.content`

## Export Presentation as PDF

```bash
gws drive files export --params '{"fileId": "<PRESENTATION_ID>", "mimeType": "application/pdf"}' -o deck.pdf
```

- Google Workspace files (Slides, Docs, Sheets) cannot be downloaded with `files get --alt media` — use `files export` instead
- `files get` on a Workspace file returns metadata JSON, not binary content
- The `-o` flag saves the exported bytes to a local file
- Response JSON includes `bytes`, `mimeType`, `saved_file`, `status`
- Claude Code's Read tool can then read the PDF directly (with `pages` param for >10 pages) — useful for extracting text, tables, and visual content from slides without JSON parsing
- For programmatic text extraction, the Slides API JSON (`presentations get`) gives structured access to text elements, but requires traversing nested `pageElements > shape > text > textElements > textRun > content` and `table > tableRows > tableCells > text` paths — the PDF export + Read is simpler for one-off content extraction

## Get a Single Slide Page

```bash
gws slides presentations pages get \
  --params '{"presentationId": "<ID>", "pageObjectId": "<SLIDE_OBJECT_ID>"}'
```
