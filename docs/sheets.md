# Sheets

Reading spreadsheets requires `spreadsheets.readonly` scope (added via `-s sheets`).

## Read a Spreadsheet

Use the `+read` helper for simple reads:

```bash
gws sheets +read --spreadsheet <SPREADSHEET_ID> --range "Sheet Name" --format table
```

## List Tabs

To list all tabs in a spreadsheet (find tab names for a given gid):

```bash
gws sheets spreadsheets get --params '{"spreadsheetId": "<ID>", "fields": "sheets.properties"}'
```

## Cell Formatting (Colors)

The `+read` helper and `values` endpoint return cell values only — no formatting or colors. To get cell background colors (e.g., for color-coded legends):

```bash
gws sheets spreadsheets get --params '{"spreadsheetId": "<ID>", "fields": "sheets(data(rowData(values(effectiveValue,effectiveFormat.backgroundColor))))", "ranges": "Sheet Name"}'
```

- Returns both `effectiveValue` (cell content) and `effectiveFormat.backgroundColor` (RGB floats 0-1)
- Colors are `{red, green, blue}` — missing keys default to 0 (e.g., `{green: 1}` = pure green, `{}` = black)
- Useful for spreadsheets where color coding carries meaning (e.g., event type legends)
- Response can be large — use `ranges` to limit to the relevant sheet/area
