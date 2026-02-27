---
description: Generate output schema (dataset_schema.json, output_schema.json) for an Apify Actor by analyzing its source code
argument-hint: Optional path to Actor directory or description of Actor output
---

# Generate Actor Output Schema

You are generating output schema files for an Apify Actor. The output schema tells Apify Console how to display run results. You will analyze the Actor's source code, create `dataset_schema.json` and `output_schema.json`, and update `actor.json`.

## Core Principles

- **Analyze code first**: Read the Actor's source to understand what data it actually pushes to the dataset — never guess
- **Every field is nullable**: APIs and websites are unpredictable — always set `"nullable": true`
- **Anonymize examples**: Never use real user IDs, usernames, or personal data in examples
- **Verify against code**: If TypeScript types exist, cross-check the schema against both the type definition AND the code that produces the values

---

## Phase 1: Discover Actor Structure

**Goal**: Locate the Actor and understand its output

Initial request: $ARGUMENTS

**Actions**:
1. Create todo list with all phases
2. Find the `.actor/` directory containing `actor.json`
3. Read `actor.json` to understand the Actor's configuration
4. Check if `dataset_schema.json` and `output_schema.json` already exist
5. Find all places where data is pushed to the dataset:
   - **JavaScript/TypeScript**: Search for `Actor.pushData(`, `dataset.pushData(`, `Dataset.pushData(`
   - **Python**: Search for `Actor.push_data(`, `dataset.push_data(`, `Dataset.push_data(`
6. Find output type definitions:
   - **TypeScript**: Look for output type interfaces/types (e.g., in `src/types/`, `src/types/output.ts`)
   - **Python**: Look for TypedDict, dataclass, or Pydantic model definitions
7. If inline `storages.dataset` config exists in `actor.json`, note it for migration

Present findings to user: list all discovered output fields, their types, and where they come from.

---

## Phase 2: Generate `dataset_schema.json`

**Goal**: Create a complete dataset schema with field definitions and display views

### File structure

```json
{
    "actorSpecification": 1,
    "fields": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "properties": {
            // All output fields here
        },
        "required": [],
        "additionalProperties": true
    },
    "views": {
        "overview": {
            "title": "Overview",
            "description": "Most important fields at a glance",
            "transformation": {
                "fields": [
                    // 8-12 most important field names
                ]
            },
            "display": {
                "component": "table",
                "properties": {
                    // Display config for each overview field
                }
            }
        }
    }
}
```

### Hard rules (no exceptions)

| Rule | Detail |
|------|--------|
| `"nullable": true` | On **every** field — APIs are unpredictable |
| `"additionalProperties": true` | On **every** object (top-level and nested) |
| `"required": []` | Always empty array |
| Anonymized examples | No real user IDs, usernames, or content |
| `"type"` required with `"nullable"` | AJV rejects `nullable` without a `type` on the same field |

> **Note**: `nullable` is an Apify-specific extension to JSON Schema draft-07. It is intentional and correct.

### Field type patterns

**String field:**
```json
"title": {
    "type": "string",
    "description": "Title of the scraped item",
    "nullable": true,
    "example": "Example Item Title"
}
```

**Number field:**
```json
"viewCount": {
    "type": "number",
    "description": "Number of views",
    "nullable": true,
    "example": 15000
}
```

**Boolean field:**
```json
"isVerified": {
    "type": "boolean",
    "description": "Whether the account is verified",
    "nullable": true,
    "example": true
}
```

**Array field:**
```json
"hashtags": {
    "type": "array",
    "description": "Hashtags associated with the item",
    "items": { "type": "string" },
    "nullable": true,
    "example": ["#example", "#demo"]
}
```

**Nested object field:**
```json
"authorInfo": {
    "type": "object",
    "description": "Information about the author",
    "properties": {
        "name": { "type": "string", "nullable": true },
        "url": { "type": "string", "nullable": true }
    },
    "required": [],
    "additionalProperties": true,
    "nullable": true,
    "example": { "name": "Example Author", "url": "https://example.com/author" }
}
```

**Enum field:**
```json
"contentType": {
    "type": "string",
    "description": "Type of content",
    "enum": ["article", "video", "image"],
    "nullable": true,
    "example": "article"
}
```

**Union type (e.g., TypeScript `ObjectType | string`):**
```json
"metadata": {
    "type": ["object", "string"],
    "description": "Structured metadata object, or error string if unavailable",
    "nullable": true,
    "example": { "key": "value" }
}
```

### Anonymized example values

Use realistic but generic values. Follow platform ID format conventions:

| Field type | Example approach |
|---|---|
| IDs | Match platform format and length (e.g., 11 chars for YouTube video IDs) |
| Usernames | `"exampleuser"`, `"sampleuser123"` |
| Display names | `"Example Channel"`, `"Sample Author"` |
| URLs | Use platform's standard URL format with fake IDs |
| Dates | `"2025-01-15T12:00:00.000Z"` (ISO 8601) |
| Text content | Generic descriptive text, e.g., `"This is an example description."` |

### Views section

- `transformation.fields`: List 8–12 most important field names (order = column order in UI)
- `display.properties`: One entry per overview field with `label` and `format`
- Available formats: `"text"`, `"number"`, `"date"`, `"link"`, `"boolean"`, `"image"`, `"array"`, `"object"`

Pick fields that give users the most useful at-a-glance summary of the data.

---

## Phase 3: Generate `output_schema.json`

**Goal**: Create the output schema that tells Apify Console where to find results

For most Actors that push data to a dataset, this is a minimal file:

```json
{
    "actorOutputSchemaVersion": 1,
    "title": "<Descriptive title — what the Actor returns>",
    "description": "<One sentence describing the output data>",
    "properties": {
        "dataset": {
            "type": "string",
            "title": "Results",
            "description": "Dataset containing all scraped data",
            "template": "{{links.apiDefaultDatasetUrl}}/items"
        }
    }
}
```

> **Critical**: Each property entry **must** include `"type": "string"` — this is an Apify-specific convention. The Apify meta-validator rejects properties without it (and rejects `"type": "object"` — only `"string"` is valid here).

If the Actor also stores files in key-value store, add a second property:
```json
"files": {
    "type": "string",
    "title": "Files",
    "description": "Key-value store containing downloaded files",
    "template": "{{links.apiDefaultKeyValueStoreUrl}}/keys"
}
```

### Available template variables

- `{{links.apiDefaultDatasetUrl}}` — API URL of default dataset
- `{{links.apiDefaultKeyValueStoreUrl}}` — API URL of default key-value store
- `{{links.publicRunUrl}}` — Public run URL
- `{{links.consoleRunUrl}}` — Console run URL
- `{{links.apiRunUrl}}` — API run URL
- `{{links.containerRunUrl}}` — URL of webserver running inside the run
- `{{run.defaultDatasetId}}` — ID of the default dataset
- `{{run.defaultKeyValueStoreId}}` — ID of the default key-value store

---

## Phase 4: Update `actor.json`

**Goal**: Wire the schema files into the Actor configuration

**Actions**:
1. Read the current `actor.json`
2. Add or update the `storages.dataset` reference:
   ```json
   "storages": {
       "dataset": "./dataset_schema.json"
   }
   ```
3. Add or update the `output` reference:
   ```json
   "output": "./output_schema.json"
   ```
4. If `actor.json` had an inline `storages.dataset` object (not a string path), migrate its content into `dataset_schema.json` and replace the inline object with the file path string

---

## Phase 5: Review and Validate

**Goal**: Ensure correctness and completeness

**Checklist**:
- [ ] Every output field from the source code is in `dataset_schema.json`
- [ ] Every field has `"nullable": true`
- [ ] Every object has `"additionalProperties": true` and `"required": []`
- [ ] Every field has a `"description"` and an `"example"`
- [ ] All example values are anonymized
- [ ] `"type"` is present on every field that has `"nullable"`
- [ ] Views list 8–12 most useful fields with correct display formats
- [ ] `output_schema.json` has `"type": "string"` on every property
- [ ] `actor.json` references both schema files

Present the generated schemas to the user for review before writing them.

---

## Phase 6: Summary

**Goal**: Document what was created

Report:
- Files created or updated
- Number of fields in the dataset schema
- Fields selected for the overview view
- Any fields that need user clarification (ambiguous types, unclear nullability)
- Suggested next steps (test locally with `apify run`, verify output tab in Console)
