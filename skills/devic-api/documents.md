# Documents & Skills API

Knowledge documents are the markdown your assistants and agents read at runtime.
A **skill** is one of those documents (or a folder of them) flagged as such: its
name and description are injected into the system prompt, and the model loads the
full instructions on demand.

**Skills are documents.** There is no separate skills resource and no
`/api/v1/skills` endpoint. A skill is created and edited with the document and
folder endpoints (`isSkill: true`), or scaffolded in one call with
`POST /api/v1/documents/skills/scaffold`; the `/api/v1/documents/skills/*`
routes only list, scaffold and download them.

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

### Response envelope

Responses are wrapped as `{ "success": true, "data": …, "meta": { "timestamp", "duration" } }`.
The deletes, attaches and detaches answer `{ "success": true }` **without**
`data`; `DELETE /documents/skills/:id/install` answers `204` with no body.

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

`name` and `markdownContent` missing answer `400 name is required` /
`400 markdownContent is required`. A `projectId` the caller cannot reach answers
`403`. Returns `201` with the document.

Only **markdown** can be created here. To upload a PDF or DOCX, use the
[Files API](files.md) or the Devic dashboard.

On create the platform generates a `summary` with an LLM and indexes the content
for semantic search, so the call is not instantaneous. Both are best-effort: a
document whose indexing failed is still readable by id, just not findable by
semantic search. The response carries the indexing outcome in `ragStatus` (and
`ragError` when it did not index).

## Update a Document

```
PATCH /api/v1/documents/:id
```

Accepts `name`, `summary`, `markdownContent`, `projectId`, `parentDocumentId`,
`folderId`, `isSkill`, `tags`. Send `null` on `projectId`, `parentDocumentId` or
`folderId` to unset them.

**Changing `markdownContent` creates a new version** and regenerates the summary
(a `summary` sent in the same call is overwritten when that regeneration
succeeds). Sending the same content creates no version. The response carries
`versionCreated` so you can tell a real edit from a no-op — it is computed for
that response only and never appears on a `GET`. `tags` replaces the whole list.

## List Documents

```
GET /api/v1/documents?projectId=&status=&fileType=&search=&folderId=&parentOnly=&isSkill=&tags=&offset=&limit=
```

| Parameter | Description |
|-----------|-------------|
| `projectId` | Only documents of this project |
| `status` | `pending`, `ready` or `failed` |
| `fileType` | `md`, `pdf`, `txt` or `docx` |
| `search` | Case-insensitive match on the document **name** (not the content) |
| `folderId` | Documents directly in this folder. `none` returns unfiled documents |
| `parentOnly` | `true`: only root documents (no parent), each with a `childDocumentCount` |
| `isSkill` | `true` / `false` filters on the flag. Folder-skills are not documents: use the catalog |
| `tags` | Comma-separated; matches any of them |
| `offset` / `limit` | `limit` defaults to 20 and is capped at 100 |

Returns `{ "documents": [ … ], "total": <n> }`, newest first.

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

`targetId` is always the entity's `_id` — for an assistant too, not its
identifier. Any other `targetType` is a `400`; an unknown target is a `404`.

`/detach` takes the same body. Folders have the same pair under
`/api/v1/document-folders/:folderId/attach|detach`.

---

## Folders

```
POST /api/v1/document-folders
```

```json
{
  "name": "Playbooks",
  "projectId": "6874f21c9a3b4e5d2f108abc",
  "parentFolderId": null,
  "color": "#4f46e5",
  "isSkill": false,
  "tags": ["sales"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | `400 name is required` otherwise |
| `projectId` | string | no | Scopes the folder to a project |
| `parentFolderId` | string | no | Nests it inside another folder |
| `color` | string | no | Color tag |
| `isSkill` | boolean | no | Makes it a folder-skill (prefer the scaffold, below) |
| `tags` | string[] | no | Category tags |

`PATCH /api/v1/document-folders/:id` accepts the same fields, all optional.
`parentFolderId` and `projectId` take `null` to unset; changing `projectId`
cascades to every descendant folder and document. `tags` replaces the list.

`GET /api/v1/document-folders?projectId=` lists folders sorted by name, each
with a recursive document count.

`DELETE /api/v1/document-folders/:id?deleteDocuments=true` deletes the folder
**and every folder below it**. Without `deleteDocuments=true` their documents
are kept and left unfiled; with it they are deleted too.

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
  generated summary (for a folder-skill: the folder name and the `SKILL.md`
  summary).

The description is what the model sees before deciding whether to open the skill,
so write it as a trigger — *"How to … when …"* — rather than as a title.

**Two shapes:**

| Shape | What it is |
|---|---|
| Document-skill | one markdown document with `isSkill: true` |
| Folder-skill | a folder with `isSkill: true` containing a `SKILL.md`, plus any supporting documents |

For a folder-skill, the metadata comes from the `SKILL.md` **directly inside**
that folder.

**Creating and editing a skill** uses the document and folder endpoints above:

| To | Call |
|----|------|
| Create a document-skill | `POST /api/v1/documents` with `isSkill: true` and the frontmatter in `markdownContent` |
| Create a folder-skill | `POST /api/v1/documents/skills/scaffold` (below) |
| Add a file to a folder-skill | `POST /api/v1/documents` with `folderId` set to the skill folder |
| Edit a skill's text | `PATCH /api/v1/documents/:id` (the `SKILL.md`, or the document-skill itself) |
| Flag / unflag an existing document or folder | `PATCH` it with `isSkill` |
| Delete a skill | `DELETE` the document, or the folder with `deleteDocuments=true` |

### Scaffold a Skill

```
POST /api/v1/documents/skills/scaffold
```

```json
{
  "name": "Incident triage",
  "description": "How to triage a production incident.",
  "tags": ["ops"],
  "projectId": "6874f21c9a3b4e5d2f108abc",
  "parentFolderId": "6874f21c9a3b4e5d2f108def"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Skill name; also names the folder. Trimmed. `400 name is required` when empty |
| `description` | string | no | One line, phrased as a trigger. Newlines are collapsed to spaces |
| `projectId` | string | no | Scopes the folder and its `SKILL.md`. `403` when the caller cannot reach it |
| `parentFolderId` | string | no | Creates the skill folder inside another folder |
| `tags` | string[] | no | Set on the folder |

Creates the folder (flagged `isSkill`) **and** a `SKILL.md` inside it whose body
is the frontmatter plus a starter outline:

```markdown
---
name: Incident triage
description: How to triage a production incident.
---

# Incident triage

How to triage a production incident.

## Instructions

Describe here, step by step, how to perform this skill.
```

Returns `201` with `{ folder, skillDoc }`. Replace the outline by editing
`skillDoc` with `PATCH /api/v1/documents/:id`. Use `folder._id` when attaching
the skill, with `type: "folder"`.

Doing it by hand also works — create a folder with `isSkill: true`, then a
document named `SKILL` (any case) inside it — but the scaffold is one call and
cannot get the naming convention wrong.

### The Catalog

```
GET /api/v1/documents/skills?search=&tags=&projectId=&page=&limit=
```

| Parameter | Description |
|-----------|-------------|
| `search` | Case-insensitive substring of the skill's `name` or `description` |
| `tags` | Comma-separated; matches any of them |
| `projectId` | Only skills of this project |
| `page` | 1-based, default 1. Only used with `limit` |
| `limit` | Page size, max 200 |

Always returns, sorted by name:

```json
{
  "items": [
    {
      "id": "6874f21c9a3b4e5d2f108abc",
      "type": "folder",
      "name": "Incident triage",
      "description": "How to triage a production incident.",
      "tags": ["ops"],
      "projectId": "6874f21c9a3b4e5d2f108aaa",
      "readCount": 12,
      "linkedAgentsCount": 2,
      "linkedAssistantsCount": 1,
      "installCount": 3,
      "installedByUsersCount": 2
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 1
}
```

`id` is the document `_id` (document-skill) or folder `_id` (folder-skill), and
`type` says which. A folder-skill connected to a GitHub repository also carries
`github: { owner, repo, path, ref }`.

Omitting `limit` returns the **whole catalog in a single page** but without the
usage counters (`linkedAgentsCount`, `linkedAssistantsCount`, `installCount`,
`installedByUsersCount`, `lastInstallAt`), which are computed per page. Pass
`limit` (max 200) when you want them.

`GET /api/v1/documents/skills/tags?projectId=` returns the distinct tags used by
documents and folders, as a sorted string array.

### Skill Tree

```
GET /api/v1/documents/skills/:id/tree?type=document|folder
```

Returns `{ skill, files: [{ path, content, fileType }], version }` — everything
an install would download. `skill` is the catalog item; `path` is relative to the
skill root (subfolder names, then the file name). `type` is auto-detected when
omitted: a folder with that id wins, otherwise it is read as a document. `version`
is a timestamp token (ms) that changes whenever the skill changes, which is how
installed copies know they are stale.

`POST /api/v1/documents/skills/:id/install` returns the same tree and records who
downloaded it. Its optional body:

| Field | Type | Description |
|-------|------|-------------|
| `agents` | string[] | The local coding agents it was installed into |
| `scope` | string | `project` (default) or `global` |
| `cliVersion` | string | Version of the installing CLI |

`DELETE /api/v1/documents/skills/:id/install?scope=project|global` records the
uninstall (every scope when `scope` is omitted) and answers `204`; an install
that is not recorded is not an error. Both are what
`devic skills install|uninstall` use.

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
| Deleting a document | also deletes its subdocuments, and removes it from the `knowledgeDocumentIds` of every agent, assistant and environment. `knowledgeSkills` entries that point at it are **not** removed |
| Deleting a folder | keeps its documents unless `deleteDocuments=true` |
| Moving a folder between projects | cascades to every descendant folder and document |
| `[[wikilinks]]` in the body | produce edges in `/documents/graph` only; they attach nothing |
| Subdocuments | a real hierarchy, set with `parentDocumentId` |
| Every content update | regenerates the summary with an LLM and re-indexes |
