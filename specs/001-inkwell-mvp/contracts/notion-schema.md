# Contract: Notion Database Schema & Write Operations

**Consumer**: PWA frontend (`src/services/notion.ts`)
**Provider**: Notion API (`api.notion.com/v1`) — called directly from the browser using the user's OAuth access token

---

## Database Bootstrap

On first publish, Inkwell verifies or creates a Notion database with this schema. The database is created in the user's default workspace.

**Database title**: `Inkwell Journal`

### Required Properties

| Property Name | Notion Type | Options / Config |
|---|---|---|
| `Title` | `title` | Auto-generated: `{YYYY-MM-DD} — {first 50 chars of entry text}` |
| `Captured At` | `date` | ISO 8601 datetime; includes time and timezone |
| `Tags` | `multi_select` | Options created dynamically as tags are applied; no predefined option list |
| `Related Entries` | `relation` | Self-referential relation on this database |
| `Has Doodles` | `checkbox` | `true` if `doodles.length > 0` after OCR |
| `Word Count` | `number` | Integer; approximate word count of `OCROutput.text` |
| `Calibration Version` | `number` | Integer; value of `CalibrationProfile.schemaVersion` at capture time |

**Bootstrap contract**: If the `Inkwell Journal` database already exists (detected by querying the user's workspace for databases with this title), Inkwell verifies that all required properties are present. Missing properties are added; existing properties of the wrong type surface an error to the user.

---

## Page Creation Contract

### Input → Notion API call mapping

```typescript
interface NotionPageWriteInput {
  databaseId: string;
  entry: PendingEntry;
  tags: string[];                  // AI-extracted themes and entities
  relatedPageIds: string[];        // from entry linking step
}
```

**API call**: `POST /v1/pages`

```json
{
  "parent": { "database_id": "<databaseId>" },
  "properties": {
    "Title":             { "title": [{ "text": { "content": "<auto-title>" } }] },
    "Captured At":       { "date": { "start": "<ISO 8601>" } },
    "Tags":              { "multi_select": [{ "name": "tag1" }, { "name": "tag2" }] },
    "Related Entries":   { "relation": [{ "id": "<pageId>" }, ...] },
    "Has Doodles":       { "checkbox": true },
    "Word Count":        { "number": 142 },
    "Calibration Version": { "number": 1 }
  },
  "children": "<block sequence — see Block Structure below>"
}
```

---

## Block Structure Contract

Blocks are assembled in reading order, interleaving text paragraphs, highlight callouts, doodle images, and strikethrough text.

### Block Types Used

**Plain text paragraph**:
```json
{ "type": "paragraph", "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "..." } }] } }
```

**Highlighted passage** (one per `FormattingAnnotation` with `type: 'highlight'`):
```json
{
  "type": "callout",
  "callout": {
    "rich_text": [{ "type": "text", "text": { "content": "<highlighted text>" } }],
    "icon": { "type": "emoji", "emoji": "🖊️" },
    "color": "yellow_background"
  }
}
```

**Strikethrough text** (inline rich text, not a block type):
```json
{
  "type": "paragraph",
  "paragraph": {
    "rich_text": [{
      "type": "text",
      "text": { "content": "<struck text>" },
      "annotations": { "strikethrough": true }
    }]
  }
}
```

**Doodle image** (one per `DoodleRegion`, inserted at nearest paragraph boundary):
```json
{
  "type": "image",
  "image": {
    "type": "external",
    "external": { "url": "<DoodleRegion.uploadedUrl>" },
    "caption": [{ "type": "text", "text": { "content": "Doodle — <pageZone>" } }]
  }
}
```

### Block Assembly Algorithm

```
1. Split OCROutput.text into segments at paragraph_break annotation positions
2. For each segment:
   a. Scan for highlight annotations overlapping this segment
      → Extract highlighted sub-strings, replace with callout blocks
      → Emit remaining text as paragraph block(s)
   b. Scan for strikethrough annotations overlapping this segment
      → Apply strikethrough annotation to rich_text spans inline
   c. After emitting all paragraph blocks for this segment,
      check if any DoodleRegion.pageZone is 'inline' and falls within this segment's character range
      → Emit image block immediately after
3. After all paragraph blocks, emit remaining DoodleRegion image blocks
   grouped by pageZone (top → before first paragraph, bottom → after last, margins → after last)
```

---

## Tagging Contract

**Endpoint called**: `POST /api/tag` (serverless proxy to Claude)

```typescript
// Request
interface TagRequest {
  text: string;                    // OCROutput.text (first 2000 chars for MVP)
}

// Response 200
interface TagResponse {
  tags: string[];                  // 3–8 tags; each ≤ 30 chars; Title Case
  processingMs: number;
}
```

**Tag extraction prompt** (stable contract):
```
Extract 3–8 topic tags and named entities from this journal entry excerpt.
Return ONLY a JSON array of strings. Each tag must be ≤30 characters.
Use Title Case. Include both thematic tags (e.g., "Anxiety", "Goal Setting")
and named entities where present (e.g., "Therapy Session", "Morning Run").
Do not include generic tags like "Journal Entry" or "Writing".

Text:
<entry text here>
```

---

## Entry Linking Contract

**Endpoint called**: `POST /api/link-entries` (serverless proxy to Claude)

```typescript
// Request
interface LinkEntriesRequest {
  newEntryText: string;            // first 1000 chars of new entry
  newEntryTags: string[];
  existingEntries: Array<{
    notionPageId: string;
    title: string;
    tags: string[];
    capturedAt: string;            // ISO date
  }>;                              // max 50 most recent
}

// Response 200
interface LinkEntriesResponse {
  relatedPageIds: string[];        // Notion page IDs to link; may be empty
  processingMs: number;
}
```

**Linking prompt** (stable contract):
```
Given a new journal entry, identify which of the existing entries are topically related.
Return ONLY a JSON array of Notion page IDs for related entries.
Return an empty array if none are related. Maximum 5 links.

New entry text: <text>
New entry tags: <tags>

Existing entries:
<id>: <title> [<tags>] (<date>)
...

Related page IDs:
```

---

## Atomicity Contract

Per FR-024, all three Notion operations (page creation, tags, relations) are part of a single logical publish. Implementation:

1. `POST /v1/pages` with `properties` including tags, relations, and all block children in a single call
2. If the API call fails for any reason, no partial page is left in Notion (the page was never created)
3. If the call succeeds but returns a page without all properties applied (Notion API inconsistency), surface an error and store the Notion page ID in `PendingEntry.notionPageId` so the user can retry property application without creating a duplicate page

**Retry safety**: The presence of `PendingEntry.notionPageId` signals that the page was created but properties may be incomplete. The retry path PATCHes the existing page rather than creating a new one.
