# Whisper API (Speech-to-Text)

The Whisper API transcribes audio to text using OpenAI Whisper. Transcription runs with Devic's own OpenAI key — you do **not** need to configure an OpenAI provider for your client. Each transcription is stored in the `transcripts` collection and the endpoint returns a `transcriptId` you can attach to the resulting assistant message.

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/whisper` | Transcribe an audio file (binary or URL) to text |

---

## Transcribe Audio

Transcribes speech to text. The audio can be provided in **two ways**:

1. **Binary upload** — send the audio file as `multipart/form-data` (field `audio`). The file is stored in Devic before transcription, so the transcript keeps a durable download URL.
2. **Download URL** — send an `audioUrl` pointing to an already hosted audio file. Devic downloads it and transcribes it.

```
POST /api/v1/whisper
```

### Request

Either `multipart/form-data` (binary) or `application/json` / form fields (URL):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `audio` | file (binary) | one of `audio`/`audioUrl` | Audio file to transcribe (multipart field). Max 25 MB |
| `audioUrl` | string | one of `audio`/`audioUrl` | Download URL of an already hosted audio file |
| `language` | string | no | ISO-639-1 language hint (e.g. `es`, `en`). Improves accuracy when known |
| `messageUid` | string | no | uid of the ChatMessage (within a chat's `chatContent`) this audio refers to |
| `chatUid` | string | no | Chat uid this transcript belongs to |

At least one of `audio` or `audioUrl` must be provided.

### Constraints

| Constraint | Value |
|------------|-------|
| Max audio size (binary) | 25 MB |
| Binary field name | `audio` |
| Model | `whisper-1` (configurable server-side) |

### Response (200)

```json
{
  "success": true,
  "data": {
    "transcriptId": "550e8400-e29b-41d4-a716-446655440000",
    "text": "Hola, ¿me puedes ayudar con mi pedido?",
    "language": "es",
    "audioUrl": "https://storage.devic.ai/shared/abc123.webm",
    "model": "whisper-1"
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `transcriptId` | string | Public id of the stored transcript. Send it back as `transcriptId` when posting the message |
| `text` | string | Transcribed text |
| `language` | string | Language hint used, if any |
| `audioUrl` | string | Download URL of the source audio (stored copy for binary uploads, or the URL you provided) |
| `model` | string | Transcription model used |

### Error Responses

| Status | Description |
|--------|-------------|
| 400 | No audio provided (`audio`/`audioUrl` missing), audio exceeds 25 MB, or the `audioUrl` could not be downloaded |
| 401 | Unauthorized |
| 500 | Transcription failed or storage not configured |

### Examples

**Binary upload:**

```bash
curl -X POST "https://api.devic.ai/api/v1/whisper" \
  -H "Authorization: Bearer devic-your-api-key" \
  -F "audio=@/path/to/recording.webm" \
  -F "language=es"
```

**From a download URL:**

```bash
curl -X POST "https://api.devic.ai/api/v1/whisper" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "audioUrl": "https://example.com/recording.mp3",
    "language": "en"
  }'
```

## Linking a transcript to a message

The transcript is meant to be referenced from the message it produced. Send the
returned `transcriptId` in the `transcriptId` field when posting the message to an
assistant. It is stored on the user's `ChatMessage` inside `chatContent`, so the
conversation keeps a link to the original audio. Because the message text and the
transcript text are stored separately, you can tell whether the user edited the
transcription before sending.

```bash
# 1. Transcribe the recording
RESPONSE=$(curl -s -X POST "https://api.devic.ai/api/v1/whisper" \
  -H "Authorization: Bearer devic-your-api-key" \
  -F "audio=@recording.webm" \
  -F "language=es")

TRANSCRIPT_ID=$(echo $RESPONSE | jq -r '.data.transcriptId')
TEXT=$(echo $RESPONSE | jq -r '.data.text')

# 2. (Optionally let the user review/edit TEXT here)

# 3. Send the message linking the transcript
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages?async=true" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"message\": \"$TEXT\",
    \"transcriptId\": \"$TRANSCRIPT_ID\"
  }"
```

### `transcriptId` on the message DTO

The assistant message endpoint (`POST /api/v1/assistants/:identifier/messages`)
accepts an optional `transcriptId` field:

| Field | Type | Description |
|-------|------|-------------|
| `transcriptId` | string | Id of a transcript from `/api/v1/whisper` that seeded this message. Persisted on the stored `ChatMessage` |

## Permissions (OAuth scopes)

When using OAuth-scoped tokens, transcription requires the `whisper:write` scope
(also covered by the catch-all `platform:write`). Standard API keys without an
endpoint restriction can call it directly; keys restricted by endpoint must allow
`/api/v1/whisper/*`.
