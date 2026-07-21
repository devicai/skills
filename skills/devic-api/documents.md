# Documents & Skills API

Knowledge documents are the markdown your assistants and agents read at runtime.
A **skill** is one of those documents (or a folder of them) flagged as such: its
name and description are injected into the system prompt, and the model loads the
full instructions on demand.

## Endpoints Overview

### Documents

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/documents` | List documents |
| POST | `/api/v1/documents` | Create a markdown document |
| GET | `/api/v1/documents/:id` | Get a document |
| PATCH | `/api/v1/documents/:id` | Update a document |
| DELETE | `/api/v1/documents/:id` | Delete a document (and its subdocuments) |
| GET | `/api/v1/documents/:id/versions` | List versions |
| GET | `/api/v1/documents/:id/versions/:version` | Get one version |
| POST | `/api/v1/documents/:id/revert/:version` | Revert to a version |
| GET | `/api/v1/documents/:id/subdocuments` | List child documents |
| GET | `/api/v1/documents/:id/usage` | Which entities can reach this document |
| GET | `/api/v1/documents/graph` | Documents and the links between them |
| POST | `/api/v1/documents/:id/attach` | Attach to an agent, assistant or environment |
| POST | `/api/v1/documents/:id/detach` | Detach |

### Folders

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/document-folders` | List folders |
| POST | `/api/v1/document-folders` | Create a folder |
| PATCH | `/api/v1/document-folders/:id` | Update a folder |
| DELETE | `/api/v1/document-folders/:id` | Delete a folder |
| POST | `/api/v1/document-folders/:folderId/attach` | Attach a whole folder |
| POST | `/api/v1/document-folders/:folderId/detach` | Detach a folder |

### Skills

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/documents/skills` | The skills catalog (documents + folders) |
| POST | `/api/v1/documents/skills/scaffold` | Create a folder-skill with its `SKILL.md` |
| GET | `/api/v1/documents/skills/tags` | Distinct tags |
| GET | `/api/v1/documents/skills/:id/tree` | Every file of a skill, with contents |
| POST | `/api/v1/documents/skills/:id/install` | Download the tree and record an install |
| DELETE | `/api/v1/documents/skills/:id/install` | Record an uninstall |

---

## Create a Document

```
POST /api/v1/documents
```

```json
{
  "name": "Sales playbook",
  "markdownContent": "# Sales playbook\n\n...",
  "projectId": "6874f21c9a3b4e5d2f108abc",
  "folderId": "6874f21c9a3b4e5d2f108def",
  "parentDocumentId": null,
  "isSkill": false,
  "tags": ["sales"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Without extension. The stored `fileName` is `<name>.md` |
| `markdownContent` | string | yes | The body |
| `projectId` | string | no | Scopes the document to a project |
| `folderId` | string | no | Files it into a folder |
| `parentDocumentId` | string | no | Makes it a subdocument |
| `isSkill` | boolean | no | Publishes it in the skills catalog |
| `tags` | string[] | no | Category tags, used to filter the catalog |

Only **markdown** can be created here. To upload a PDF or DOCX, use the
[Files API](files.md) or the Devic dashboard.

On create the platform generates a `summary` with an LLM and indexes the content
for semantic search, so the call is not instantaneous. Both are best-effort: a
document whose indexing failed is still readable by id, just not findable by
semantic search.

## Update a Document

```
PATCH /api/v1/documents/:id
```

Accepts `name`, `summary`, `markdownContent`, `projectId`, `parentDocumentId`,
`folderId`, `isSkill`, `tags`. Send `null` on `projectId`, `parentDocumentId` or
`folderId` to unset them.

**Changing `markdownContent` creates a new version** and regenerates the summary.
The response carries `versionCreated` so you can tell a real edit from a no-op —
it is computed for that response only and never appears on a `GET`.

## List Documents

```
GET /api/v1/documents?projectId=&status=&fileType=&search=&folderId=&parentOnly=&isSkill=&tags=&offset=&limit=
```

`limit` defaults to 20 and is capped at 100. `folderId=none` returns unfiled
documents. `tags` is comma-separated and matches any of them.

The response includes each document's full version history, which grows over
time — project the fields you need instead of holding the whole payload.

## Attach a Document

```
POST /api/v1/documents/:id/attach
```

```json
{ "targetType": "agent", "targetId": "6874f21c9a3b4e5d2f108abc" }
```

| `targetType` | Effect |
|---|---|
| `agent` | adds it to the agent's knowledge |
| `assistant` | adds it to the assistant's knowledge |
| `environment` | adds it to the environment, which merges into every connected agent and assistant |

`/detach` takes the same body. Folders have the same pair under
`/api/v1/document-folders/:folderId/attach|detach`.

> **Attaching is only half of it.** The entity also needs a knowledge tool group
> — *Search knowledge* or *Advanced knowledge search* (see
> [built-in-tools.md](built-in-tools.md)). Without one the model has no way to
> read what you attached, and the documents are silently invisible.

## Document Usage

```
GET /api/v1/documents/:id/usage
```

Returns the assistants, agents and environments that can reach the document, and
`via` explains how: `document` (attached directly), `folder` (through the folder
it lives in, or any folder above it) or `skill` (pulled in as part of a skill).
Check it before deleting anything.

---

## Skills

### What a skill is

A skill is a document or a folder flagged `isSkill: true`. Its identity comes
from YAML frontmatter at the very top of the markdown:

```markdown
---
name: Incident triage
description: How to triage a production incident, from first alert to postmortem.
---

# Incident triage
...
```

- Only `name` and `description` are read, and both must be **single-line**
  values. No lists, no multi-line block scalars.
- `description` is truncated to 1024 characters.
- Without frontmatter the catalog falls back to the document name and its
  generated summary.

The description is what the model sees before deciding whether to open the skill,
so write it as a trigger — *"How to … when …"* — rather than as a title.

**Two shapes:**

| Shape | What it is |
|---|---|
| Document-skill | one markdown document with `isSkill: true` |
| Folder-skill | a folder with `isSkill: true` containing a `SKILL.md`, plus any supporting documents |

For a folder-skill, the metadata comes from the `SKILL.md` **directly inside**
that folder.

### Scaffold a Skill

```
POST /api/v1/documents/skills/scaffold
```

```json
{
  "name": "Incident triage",
  "description": "How to triage a production incident.",
  "tags": ["ops"],
  "projectId": null,
  "parentFolderId": null
}
```

Creates the folder **and** its `SKILL.md` with the frontmatter already written,
and returns `{ folder, skillDoc }`. Use `folder._id` when attaching the skill,
with `type: "folder"`.

Doing it by hand also works — create a folder with `isSkill: true`, then a
document named exactly `SKILL` inside it — but the scaffold is one call and
cannot get the naming convention wrong.

### The Catalog

```
GET /api/v1/documents/skills?search=&tags=&projectId=&page=&limit=
```

Always returns:

```json
{
  "items": [
    {
      "id": "6874f21c9a3b4e5d2f108abc",
      "type": "folder",
      "name": "Incident triage",
      "description": "How to triage a production incident.",
      "tags": ["ops"],
      "readCount": 12,
      "linkedAgentsCount": 2,
      "linkedAssistantsCount": 1,
      "installCount": 3
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 1
}
```

Omitting `limit` returns the **whole catalog in a single page** but without the
usage counters (`linkedAgentsCount`, `linkedAssistantsCount`, `installCount`),
which are computed per page. Pass `limit` (max 200) when you want them.

### Skill Tree

```
GET /api/v1/documents/skills/:id/tree?type=document|folder
```

Returns `{ skill, files: [{ path, content }], version }` — everything an install
would download. `type` is auto-detected when omitted. `version` is a timestamp
token that changes whenever the skill changes, which is how installed copies know
they are stale.

`POST /api/v1/documents/skills/:id/install` returns the same tree and records who
downloaded it (`agents`, `scope`, `cliVersion` in the body). `DELETE` on the same
path records the uninstall. Both are what `devic skills install|uninstall` use.

---

## Attaching Skills to Agents and Assistants

Skills are not attached with `/attach`. They are a field on the entity:

```jsonc
// assistant — at the root
{ "knowledgeSkills": [ { "id": "<folderId>", "type": "folder", "enabled": true, "preloadContent": false } ] }

// agent — inside assistantSpecialization
{ "assistantSpecialization": { "knowledgeSkills": [ { "id": "<documentId>", "type": "document" } ] } }
```

| Field | Description |
|-------|-------------|
| `id` | the document id (document-skill) or folder id (folder-skill) |
| `type` | `document` or `folder`. Must match the skill's actual shape |
| `enabled` | defaults to `true` |
| `preloadContent` | defaults to `false`. Inlines the whole skill into the system prompt instead of loading it on demand |

Bad entries are rejected with `400 INVALID_SKILLS`, which distinguishes:

| Reason | Meaning |
|--------|---------|
| `WRONG_TYPE` | the id is a skill of the *other* shape — the message names the `type` to use |
| `NOT_FOUND` | unknown id, or the document/folder is not flagged `isSkill` |
| `INVALID_TYPE` | `type` missing or not one of the two values |

> **`availableSkillIds` is a different, legacy field.** It belongs to an older
> feature and points at another collection entirely. Putting a catalog skill id
> there is accepted and does nothing. Use `knowledgeSkills`.

### Make the skill loadable

Attach the **Advanced knowledge search** group
(`f7a2c841-3b9e-4d5f-a1c8-92e7d4b06f13`) to the entity. It is what lets the model
open a skill on demand. Without it the platform falls back to inlining every
attached skill's full text into the system prompt — not an error, but it spends
tokens on every run and defeats the purpose of progressive disclosure.

---

## Notes and Limits

| Behaviour | Detail |
|-----------|--------|
| Deleting a document | also deletes its subdocuments |
| Deleting a folder | keeps its documents unless `deleteDocuments=true` |
| Moving a folder between projects | cascades to every descendant folder and document |
| `[[wikilinks]]` in the body | produce edges in `/documents/graph` only; they attach nothing |
| Subdocuments | a real hierarchy, set with `parentDocumentId` |
| Every content update | regenerates the summary with an LLM and re-indexes |
