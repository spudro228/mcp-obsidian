# TASK: Add obsidian_move_file tool to mcp-obsidian

## CONTEXT

The Obsidian Local REST API now supports a new endpoint `POST /vault/move` for moving/renaming files and folders.
This task adds support for this functionality in mcp-obsidian.

## API DOCUMENTATION

See full API specification in `openapi.yaml` in this repository.
Look for the `/vault/move` endpoint under `paths`.

## NEW API ENDPOINT

```
POST /vault/move
Content-Type: application/json

Request Body:
{
  "source": "old-path/file.md",        # Required: current path (relative to vault root)
  "destination": "new-path/file.md",   # Required: new path (relative to vault root)
  "updateLinks": true                   # Optional: update all links to this file (default: false)
}

Response 200:
{
  "source": "old-path/file.md",
  "destination": "new-path/file.md",
  "message": "Successfully moved"
}

Error Codes:
- 400 (40023): Missing source field
- 400 (40024): Missing destination field
- 400 (40022): Cannot move folder into its own subdirectory
- 404 (40420): Source file or folder does not exist
- 409 (40921): Destination path already exists
```

---

## IMPLEMENTATION STEPS

### STEP 1: Add method to obsidian.py

File: `src/mcp_obsidian/obsidian.py`

Add after `delete_file` method:

```python
def move_file(self, source: str, destination: str, update_links: bool = False) -> dict:
    """Move or rename a file or folder in the vault.

    Args:
        source: Current path of the file/folder (relative to vault root)
        destination: New path for the file/folder (relative to vault root)
        update_links: If True, update all links pointing to this file (only for files, not folders)

    Returns:
        Dict with source, destination, and message
    """
    url = f"{self.get_base_url()}/vault/move"

    payload = {
        "source": source,
        "destination": destination,
        "updateLinks": update_links
    }

    def call_fn():
        response = requests.post(
            url,
            headers=self._get_headers() | {'Content-Type': 'application/json'},
            json=payload,
            verify=self.verify_ssl,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()

    return self._safe_call(call_fn)
```

---

### STEP 2: Add tool handler to tools.py

File: `src/mcp_obsidian/tools.py`

Add new class after `DeleteFileToolHandler`:

```python
class MoveFileToolHandler(ToolHandler):
    def __init__(self):
        super().__init__("obsidian_move_file")

    def get_tool_description(self):
        return Tool(
            name=self.name,
            description="Move or rename a file or folder in the vault. Can also update all links pointing to the moved file.",
            inputSchema={
                "type": "object",
                "properties": {
                    "source": {
                        "type": "string",
                        "description": "Current path of the file or folder to move (relative to vault root)",
                        "format": "path"
                    },
                    "destination": {
                        "type": "string",
                        "description": "New path for the file or folder (relative to vault root)",
                        "format": "path"
                    },
                    "update_links": {
                        "type": "boolean",
                        "description": "If true, automatically update all links pointing to the moved file. Only works for files, not folders. (default: false)",
                        "default": False
                    }
                },
                "required": ["source", "destination"]
            }
        )

    def run_tool(self, args: dict) -> Sequence[TextContent | ImageContent | EmbeddedResource]:
        if "source" not in args:
            raise RuntimeError("source argument missing in arguments")
        if "destination" not in args:
            raise RuntimeError("destination argument missing in arguments")

        source = args["source"]
        destination = args["destination"]
        update_links = args.get("update_links", False)

        api = obsidian.Obsidian(api_key=api_key, host=obsidian_host)
        result = api.move_file(source, destination, update_links)

        return [
            TextContent(
                type="text",
                text=f"Successfully moved '{source}' to '{destination}'"
            )
        ]
```

---

### STEP 3: Register handler in server.py

File: `src/mcp_obsidian/server.py`

Add after the line `add_tool_handler(tools.DeleteFileToolHandler())`:

```python
add_tool_handler(tools.MoveFileToolHandler())
```

---

## USAGE EXAMPLES

After implementation, the tool can be used as follows:

### Rename a file
```json
{
  "source": "notes/old-name.md",
  "destination": "notes/new-name.md"
}
```

### Move file to different folder
```json
{
  "source": "inbox/note.md",
  "destination": "archive/2024/note.md"
}
```

### Move and update all links
```json
{
  "source": "projects/old-project.md",
  "destination": "archive/old-project.md",
  "update_links": true
}
```

### Rename a folder
```json
{
  "source": "old-folder",
  "destination": "new-folder"
}
```

---

## VERIFICATION

After implementation:

1. Restart the MCP server
2. Test moving a file: `obsidian_move_file(source="test.md", destination="moved/test.md")`
3. Test renaming: `obsidian_move_file(source="old.md", destination="new.md")`
4. Test with update_links: `obsidian_move_file(source="a.md", destination="b.md", update_links=true)`
5. Test error cases (missing source, existing destination)
