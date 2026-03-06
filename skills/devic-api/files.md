# Files API

The Files API allows you to upload files and receive shareable download URLs. This is typically used to attach files to assistant messages.

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/files/upload` | Upload a file and get a download URL |

---

## Upload File

Uploads a file using multipart/form-data and returns a download URL.

```
POST /api/v1/files/upload
```

### Request

- **Content-Type:** `multipart/form-data`
- **Body:** Form field `file` containing the binary file data

### Constraints

| Constraint | Value |
|------------|-------|
| Max file size | 25 MB |
| Field name | `file` |

### Response (200)

```json
{
  "success": true,
  "data": {
    "name": "report.pdf",
    "downloadUrl": "https://storage.devic.ai/shared/abc123",
    "fileType": "document"
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Original file name |
| `downloadUrl` | string | Public download URL for the uploaded file |
| `fileType` | string | Detected file type category: `image`, `document`, `audio`, `video`, or `other` |

### File Type Detection

| MIME type prefix | fileType |
|------------------|----------|
| `image/*` | `image` |
| `audio/*` | `audio` |
| `video/*` | `video` |
| `application/pdf`, `application/msword`, `application/vnd.openxmlformats-officedocument.*`, `text/*` | `document` |
| Everything else | `other` |

### Error Responses

| Status | Description |
|--------|-------------|
| 400 | No file provided or file exceeds 25 MB |
| 401 | Unauthorized |

### Example

```bash
curl -X POST "https://api.devic.ai/api/v1/files/upload" \
  -H "Authorization: Bearer devic-your-api-key" \
  -F "file=@/path/to/document.pdf"
```

### Usage with Assistant Messages

Upload a file first, then include the download URL in the message:

```bash
# 1. Upload file
UPLOAD_RESPONSE=$(curl -s -X POST "https://api.devic.ai/api/v1/files/upload" \
  -H "Authorization: Bearer devic-your-api-key" \
  -F "file=@report.pdf")

DOWNLOAD_URL=$(echo $UPLOAD_RESPONSE | jq -r '.data.downloadUrl')

# 2. Send message with file reference
curl -X POST "https://api.devic.ai/api/v1/assistants/default/messages?async=true" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"message\": \"Please analyze this report\",
    \"files\": [{
      \"name\": \"report.pdf\",
      \"downloadUrl\": \"$DOWNLOAD_URL\",
      \"fileType\": \"document\"
    }]
  }"
```
