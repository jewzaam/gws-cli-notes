# Docs API Content Structure (for Markdown Conversion)

Analyzed a Gemini-generated meeting notes doc (two tabs: Notes, Transcript).

## Tab Structure

- With `includeTabsContent:true`, tabs are in `doc.tabs[]`
- Each tab: `tabProperties.title`, `tabProperties.tabId`, `documentTab.body.content[]`
- Without `includeTabsContent`, only the first tab appears in `doc.body.content`

## Paragraph Styles (namedStyleType)

Only three styles observed in Gemini meeting docs:

- `HEADING_2` — tab title, section headers
- `HEADING_3` — subsection headers (e.g., "Summary", "Details", timestamps in transcript)
- `NORMAL_TEXT` — everything else (body content, attendee lists, attachments)
- No `HEADING_1` — the document title is metadata, not a structural element
- No list styles via `namedStyleType`; bullets appear as Unicode characters (e.g., `\ue907`) in text content

## Element Types (within paragraph.elements[])

- `textRun` — primary content. Has `content` (string) and `textStyle` (bold, strikethrough, link, etc.)
- `person` — inline @mention. Has `personProperties.name` and `personProperties.email`. Can have `textStyle.strikethrough`
- `richLink` — smart chip / embedded link. Has `richLinkProperties.title` and `richLinkProperties.uri`
- `dateElement` — inline date chip (present but content structure not fully explored)
- `sectionBreak` — top-level structural element (not inside paragraphs), marks section boundaries
- `horizontalRule` — not observed in this doc but documented in the API
- `table` — not observed in this doc but documented in the API
- `inlineObjectElement` — images, not observed here

## Special Characters in Text Content

- `\x0b` (vertical tab, `U+000B`) — soft line break within a paragraph. Used heavily in transcript tab where multiple speaker turns are in one paragraph. Map to `\n` in markdown.
- `\xa0` (non-breaking space, `U+00A0`) — appears at start of transcript paragraphs. Strip or normalize to regular space.
- `\ue907` (private use area) — Gemini's custom bullet icon in summary/detail paragraphs. Map to a markdown list bullet (`-`) or strip.
- `\n` — paragraph-ending newline (always the last char in `textRun.content`)

## Structural Patterns in Gemini Meeting Docs

**Notes tab:**

1. H2: meeting title
1. NORMAL_TEXT: "Invited" + person elements (attendee list). Strikethrough persons = removed from invite.
1. NORMAL_TEXT: "Attachments" + richLink elements (links to slides, backlog, etc.)
1. NORMAL_TEXT: "Meeting records" + richLink elements (transcript doc, recording link)
1. H3: "Summary"
1. NORMAL_TEXT: `\ue907` prefix + bold topic headers + description text (Gemini-generated summary bullets)
1. H3: "Details"
1. NORMAL_TEXT: bold topic header + `:` + detailed text with inline timestamp references (e.g., `00:03:52`)

**Transcript tab:**

1. H2: meeting title + "- Transcript"
1. Repeating pattern: H3 timestamp (e.g., `00:05:15`) followed by NORMAL_TEXT with `\x0b`-separated speaker turns
1. Each speaker turn: bold speaker name + `:` + spoken text
1. Multiple speakers in a single paragraph, separated by `\x0b`

## Text Style Properties (textRun.textStyle)

- `bold` — used for topic headers (Notes) and speaker names (Transcript)
- `strikethrough` — used on removed attendees (person elements)
- `link.url` — hyperlinks (check `textStyle.link` for linked text)
- Other possible styles (not observed here): `italic`, `underline`, `fontSize`, `foregroundColor`, `weightedFontFamily`

## Markdown Conversion Mapping

| Docs API | Markdown |
|----------|----------|
| `HEADING_1` | `#` |
| `HEADING_2` | `##` |
| `HEADING_3` | `###` |
| `NORMAL_TEXT` | plain paragraph |
| `textRun` with bold | `**text**` |
| `textRun` with strikethrough | `~~text~~` |
| `textRun` with link | `[text](url)` |
| `person` | `@Name` or `~~@Name~~` if strikethrough |
| `richLink` | `[title](uri)` |
| `\x0b` in content | line break (`\n`) |
| `\ue907` prefix | `-` (list item) |
| `sectionBreak` | blank line or `---` |
| `horizontalRule` | `---` |
