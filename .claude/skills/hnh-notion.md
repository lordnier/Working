---
name: hnh-notion
description: >
  Interact with Notion — read pages, create pages, update properties, append content,
  query databases, manage records, get database schemas, and search across the workspace.
  Use this skill whenever the user mentions Notion, says "in Notion", "with Notion", "on Notion",
  shares a Notion URL (notion.so/..., notion.site/...), asks to create/read/update a Notion page
  or database, or wants to search for something in their Notion workspace. Also trigger when the
  user mentions "database" or "records" in a context that implies Notion (e.g., project tracker,
  task list, CRM in Notion). This skill is for the ZenLabs workspace by default.
  Do NOT trigger for the /hnh-plan-notion skill which creates implementation plans from Notion
  pages — that's a different workflow.
---

# Notion Skill

Interact with Notion through a Python CLI tool wrapping the Notion REST API.

## Prerequisites

- **Python `requests` library** (already installed)
- **NOTION_API_TOKEN** in `~/.zshrc`

The integration must be connected to pages/databases in Notion before it can access them. If a 404 or "object not found" error occurs, the user likely needs to share the page with "Claude Integration" in Notion (page menu → Connections → Claude Integration).

## Auth Pattern

Following the credential rules, read the token from `~/.zshrc` and inline it:

```bash
# Read token, then use it inline
NOTION_TOKEN="<value from ~/.zshrc>"
python3 ~/.claude/skills/hnh-notion/scripts/notion.py --token "$NOTION_TOKEN" <command> [args]
```

The script also accepts `NOTION_API_TOKEN` env var, but inlining via `--token` is more reliable in subshells.

## The CLI Tool

```
python3 ~/.claude/skills/hnh-notion/scripts/notion.py --token TOKEN <command> [args]
```

All commands output JSON to stdout, errors to stderr. The script accepts both Notion page/database IDs and full URLs (auto-extracts the ID).

## Commands Reference

### read — Get page content and properties

```bash
python3 <script> --token $TOKEN read PAGE_ID_OR_URL
```

Returns: title, properties (as key-value pairs), and all content rendered as markdown-like text. Add `--raw` to also get the raw Notion block objects.

### create — Create a new page

```bash
# Under a parent page
python3 <script> --token $TOKEN create PARENT_PAGE_ID \
  --title "Meeting Notes"

# In a database
python3 <script> --token $TOKEN create DATABASE_ID \
  --database \
  --title "New Task" \
  --title-property "Task Name" \
  --properties '{"Status": {"status": {"name": "Not started"}}}'

# With content
python3 <script> --token $TOKEN create PARENT_ID \
  --title "My Page" \
  --content "# Overview\n\nThis is the first paragraph.\n\n- Item 1\n- Item 2"
```

**Content format**: The `--content` flag accepts text with basic markdown:
- `# / ## / ###` for headings
- `- ` for bullets
- `1. ` for numbered lists
- `[ ] / [x]` for to-dos
- `> ` for quotes
- `` ``` `` for code blocks
- `---` for dividers
- Plain text for paragraphs

Use `--content-file path/to/file.md` to read content from a file instead.

### update — Update page properties

```bash
python3 <script> --token $TOKEN update PAGE_ID \
  --properties '{"Status": {"status": {"name": "Done"}}}'
```

Properties must match the Notion property format. Use `schema` first to understand what properties exist and their types. `--properties` is optional when using `--archive` or `--unarchive`.

Add `--archive` to archive a page, `--unarchive` to restore it.

### append — Add content to a page

```bash
python3 <script> --token $TOKEN append PAGE_ID \
  --content "## New Section\n\nMore content here.\n\n- Point A\n- Point B"
```

Appends blocks at the end of the page. Same markdown format as `create --content`.

### query — Query a database

```bash
# All records
python3 <script> --token $TOKEN query DATABASE_ID

# With filter
python3 <script> --token $TOKEN query DATABASE_ID \
  --filter '{"property": "Status", "status": {"equals": "In progress"}}'

# With sort
python3 <script> --token $TOKEN query DATABASE_ID \
  --sort '[{"property": "Due Date", "direction": "ascending"}]'

# Combined
python3 <script> --token $TOKEN query DATABASE_ID \
  --filter '{"and": [{"property": "Status", "status": {"does_not_equal": "Done"}}, {"property": "Assignee", "people": {"contains": "USER_ID"}}]}' \
  --sort '[{"property": "Priority", "direction": "descending"}]' \
  --limit 20
```

Returns records with all properties extracted as readable values. Use `--cursor` for pagination when `has_more` is true.

### create-record — Insert a database record

```bash
python3 <script> --token $TOKEN create-record DATABASE_ID \
  --title "New Task" \
  --title-property "Task Name" \
  --properties '{"Status": {"status": {"name": "Not started"}}, "Priority": {"select": {"name": "High"}}}'
```

### update-record — Update a database record

```bash
python3 <script> --token $TOKEN update-record RECORD_ID \
  --properties '{"Status": {"status": {"name": "Done"}}}'
```

### schema — Get database structure

```bash
python3 <script> --token $TOKEN schema DATABASE_ID
```

Returns all property names, types, and options (for select/multi_select/status). **Run this before querying or updating a database** — it tells you the exact property names and valid values to use in filters and updates.

### search — Find pages and databases

```bash
# Search everything
python3 <script> --token $TOKEN search "project tracker"

# Only databases
python3 <script> --token $TOKEN search "tasks" --type database

# Only pages
python3 <script> --token $TOKEN search "meeting notes" --type page --limit 5
```

## Property Format Reference

When setting properties in `create`, `update`, `create-record`, or `update-record`, use Notion's property format. Here are the common types:

```json
{
  "Title Prop":     {"title": [{"text": {"content": "My Title"}}]},
  "Text Prop":      {"rich_text": [{"text": {"content": "Some text"}}]},
  "Number Prop":    {"number": 42},
  "Checkbox Prop":  {"checkbox": true},
  "Select Prop":    {"select": {"name": "Option A"}},
  "Multi Prop":     {"multi_select": [{"name": "Tag1"}, {"name": "Tag2"}]},
  "Status Prop":    {"status": {"name": "In progress"}},
  "Date Prop":      {"date": {"start": "2026-03-15", "end": "2026-03-20"}},
  "URL Prop":       {"url": "https://example.com"},
  "Email Prop":     {"email": "<email>"},
  "Phone Prop":     {"phone_number": "+1234567890"},
  "People Prop":    {"people": [{"id": "USER_ID"}]},
  "Relation Prop":  {"relation": [{"id": "PAGE_ID"}]}
}
```

## Workflow Patterns

### Reading a page the user links

1. Extract the page ID from the URL
2. Run `read` to get content and properties
3. Present a summary to the user

### Creating content in a database

1. Run `schema` to discover property names, types, and options
2. Build the properties JSON matching the schema
3. Run `create-record` with the correct properties

### Updating database records

1. Run `schema` to understand the database structure
2. Run `query` to find the record(s) to update
3. Run `update-record` with the new property values

### Searching and navigating

1. Run `search` to find the page/database
2. Use the returned ID for subsequent operations

## Error Handling

- **401 Unauthorized**: Token is invalid or expired — check `~/.zshrc`
- **404 Object not found**: The integration doesn't have access — user needs to connect "Claude Integration" to the page in Notion
- **400 Validation error**: Property format is wrong — run `schema` first to check types
- **409 Conflict**: Concurrent edit — retry the operation
- **429 Rate limited**: Wait and retry (Notion allows ~3 requests/second)
