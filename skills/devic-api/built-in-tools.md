# Built-in Tools

Devic provides a set of built-in tool groups that can be added to assistants and agents via the `availableToolsGroupsUids` array. Each tool group has a fixed UID.

## Available Built-in Tool Groups

| Name | UID | Description |
|------|-----|-------------|
| Code Execution Tools | `92e3a014-4493-4547-a00a-4e394193a210` | Execute Python code in a sandboxed environment |
| Search knowledge | `c0930948-219d-42a7-92d5-ad597acb77a7` | RAG-based knowledge search from uploaded documents |
| Advanced Knowledge Search | `f7a2c841-3b9e-4d5f-a1c8-92e7d4b06f13` | Advanced RAG search with enhanced retrieval strategies |
| Update Knowledge | `b902a8a8-ecbe-4b06-bc1d-0c079c1d71aa` | Write and update entries in the knowledge base |
| Web Search | `1d063a05-b8c3-42b9-89ca-4ed2e3b174bf` | Search the web and retrieve information from URLs |
| Read emails | `c4bf8a70-2ee7-40c2-8428-22fd6fc8f476` | Read emails from connected IMAP accounts |
| Send email | `2492c61d-38ef-4e24-b110-cba113d203cb` | Send emails via connected SMTP accounts |
| Suntropy Tasks Tools | `03d49bbf-b1bc-4c97-88fb-c44682ed1dcb` | Manage tasks within the Devic platform |
| Databases | `1ea39541-e1f2-482c-b268-c2c6811f3f0a` | Query and interact with connected databases |
| OCR | `0c3b1c25-b1d6-4cd2-a34a-e7f73ddd64dc` | Extract text from images using optical character recognition |
| Image Generation | `9b7f024a-2b99-4e99-85f5-2d06f4b9cad7` | Generate images from text prompts |
| Files and Directories | `8a667cd7-d9e1-4e95-ae6e-fdde2c1370e3` | Manage files and directories in the assistant's workspace |
| PDF from Markdown | `24217810-72f7-4f82-aadc-c660f4f9f1ab` | Generate PDF documents from Markdown content |
| PDF from HTML | `7df63d5c-f3e1-4030-9d79-502974781bbc` | Generate PDF documents from HTML content |
| Chart images | `d6619437-f962-4d86-9283-906381641eaa` | Generate chart and graph images from data |
| Text File Templates | `85407e07-71e1-4314-8902-c8381265f591` | Create and manage text file templates |
| Spreadsheet Tools | `8d100ced-0121-49a4-8a13-b9f2474ea844` | Read and manipulate spreadsheet files |
| Short and Medium Term Memory | `a2fa4767-d86e-4486-87b0-4680880b8868` | Agent memory for recalling previous thread context |
| Terminal Sandbox | `f8a3c1d0-5e2b-4a9f-8c7d-6b5e4f3a2c1d` | Execute terminal commands in an isolated sandbox |
| Browser Sandbox | `a7b2c3d4-e5f6-4a8b-9c0d-1e2f3a4b5c6d` | Automate browser interactions in a sandboxed environment |

## Usage Example

To assign built-in tools to an assistant or agent, include their UIDs in `availableToolsGroupsUids`:

### Create an assistant with Web Search and Code Execution

```bash
curl -X POST "https://api.devic.ai/api/v1/assistants" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Research Assistant",
    "description": "An assistant with web search and code execution",
    "model": "gpt-4.1-mini",
    "provider": "openai",
    "availableToolsGroupsUids": [
      "1d063a05-b8c3-42b9-89ca-4ed2e3b174bf",
      "92e3a014-4493-4547-a00a-4e394193a210"
    ]
  }'
```

### Update an agent to add RAG and Email tools

```bash
curl -X PUT "https://api.devic.ai/api/v1/agents/{agentId}" \
  -H "Authorization: Bearer devic-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "assistantSpecialization": {
      "availableToolsGroupsUids": [
        "c0930948-219d-42a7-92d5-ad597acb77a7",
        "c4bf8a70-2ee7-40c2-8428-22fd6fc8f476",
        "2492c61d-38ef-4e24-b110-cba113d203cb"
      ]
    }
  }'
```

> **Note:** Built-in tool UIDs can be combined with custom MCP tool server UIDs in the same `availableToolsGroupsUids` array.
