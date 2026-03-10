# How to Add New Tools to the FMC MCP Server

This guide walks through adding a new tool to the FMC MCP server, using the `list_ftd_devices` tool as a step-by-step example. It also documents the challenges we faced and how to resolve them.

---

## Overview

The FMC MCP server exposes Cisco Firepower Management Center (FMC) capabilities as MCP tools. To add a new tool, you typically need to:

1. **Identify the FMC REST API endpoint** — What does the FMC API support?
2. **Add or use the client method** — Does `FMCClient` already have what you need?
3. **Register the tool in the server** — Expose it via FastMCP's `@mcp.tool()` decorator
4. **Update the server instructions** — So clients know the tool exists
5. **Add the tool schema** (optional) — For Cursor MCP discovery
6. **Rebuild and restart** — See [Docker & Deployment](#docker--deployment) below

---

## Lessons Learned: Why Doesn't My New Tool Appear?

When adding `list_ftd_devices`, we hit several issues. Here's what we learned.

### 1. Reloading Cursor Is Not Enough (Docker Users)

**Symptom:** You add a new tool, reload Cursor, but still see only the original tools (e.g., 4 instead of 5).

**Cause:** The FMC MCP server runs in **Docker**. Cursor uses a pre-built image (`fmc-mcp-server`). The image was built *before* your code changes. Reloading Cursor restarts the container, but it still uses the same old image.

**Fix:** Rebuild the Docker image after every code change:

```bash
cd /path/to/fmc-mcp
docker build -t fmc-mcp-server -f docker/Dockerfile .
```

Then reload Cursor. The new tool will appear.

---

### 2. Container Name Conflict

**Symptom:** Cursor fails to start the FMC MCP server with:

```
docker: Error response from daemon: Conflict. The container name "/fmc-mcp-container" is already in use by container "..."
```

**Cause:** A previous container with that name is still present (e.g., from a crash or unclean exit). The `--rm` flag removes the container when it exits normally, but a stuck or orphaned container can remain.

**Fix:** Force-remove the existing container:

```bash
docker rm -f fmc-mcp-container
```

Then reload Cursor. It will start a fresh container.

---

### 3. Development Workflow: Avoid Rebuilding Every Time

If you're iterating on tools frequently, rebuilding the Docker image after each change is slow. You can run the server **directly with Python** instead of Docker. Update `~/.cursor/mcp.json`:

```json
"fmc-mcp": {
  "command": "python",
  "args": ["/path/to/fmc-mcp/fmc_mcp_server.py"],
  "envFile": "/path/to/fmc-mcp/.env"
}
```

Or with explicit env vars:

```json
"fmc-mcp": {
  "command": "python",
  "args": ["/path/to/fmc-mcp/fmc_mcp_server.py"],
  "env": {
    "FMC_HOST": "your-fmc-host",
    "FMC_USER": "your-username",
    "FMC_PASSWORD": "your-password",
    "SSL_VERIFY": "false"
  }
}
```

With this setup, code changes take effect after reloading Cursor—no Docker rebuild needed.

---

## Step 1: Identify the FMC REST API Endpoint

The FMC REST API uses paths like:

```
/api/fmc_config/v1/domain/{domainUUID}/devices/devicerecords
```

For device listing, the endpoint is:

- **Path:** `/devices/devicerecords`
- **Method:** GET
- **Response:** JSON with `items` array of device records

Check the [Cisco FMC REST API documentation](https://developer.cisco.com/docs/cisco-security-cloud-control-firewall-manager/) for other endpoints.

---

## Step 2: Add or Use the Client Method

The client lives in `sfw_mcp_fmc/fmc/client.py`. It already had `list_device_records()`:

```python
async def list_device_records(
    self,
    *,
    limit: int = 1000,
    hard_page_limit: int = 5,
    expanded: bool = True,
    offset: int = 0,
) -> List[Dict[str, Any]]:
    return await self._list_paginated(
        "/devices/devicerecords",
        limit=limit,
        hard_page_limit=hard_page_limit,
        expanded=expanded,
        start_offset=offset,
    )
```

If your endpoint doesn't exist yet:

1. Add a new method to `FMCClient`
2. Use `_request_json()` for single requests or `_list_paginated()` for list endpoints
3. Follow the same pattern as `list_device_records()` or `list_access_policies()`

---

## Step 3: Register the Tool in the Server

In `sfw_mcp_fmc/server.py`, add a new tool decorated with `@mcp.tool()`:

```python
@mcp.tool()
async def list_ftd_devices(
    domain_uuid: Optional[str] = None,
    fmc_profile: Optional[str] = None,
) -> Dict[str, Any]:
    """List all FTD (Firepower Threat Defense) devices managed by this FMC."""
    try:
        client = create_client(fmc_profile, domain_uuid_override=domain_uuid)
        await client.ensure_domain_uuid()

        devices = await client.list_device_records(limit=1000, hard_page_limit=10, expanded=True)

        # Build a concise summary for each device
        result: List[Dict[str, Any]] = []
        for d in devices:
            entry: Dict[str, Any] = {
                "id": d.get("id"),
                "name": d.get("name"),
                "hostName": d.get("hostName"),
                "type": d.get("type"),
                "model": d.get("model"),
                "healthStatus": d.get("healthStatus"),
                "managementState": d.get("managementState"),
                "deploymentStatus": d.get("deploymentStatus"),
                "isConnected": d.get("isConnected"),
            }
            if d.get("accessPolicy"):
                ap = d["accessPolicy"]
                entry["accessPolicy"] = {"id": ap.get("id"), "name": ap.get("name")}
            if d.get("ftdMode"):
                entry["ftdMode"] = d["ftdMode"]
            result.append(entry)

        return {"devices": result, "count": len(result)}
    except Exception as exc:
        logger.exception("Unexpected error in list_ftd_devices")
        return {"error": {"category": "UNEXPECTED", "message": str(exc)}}
```

Notes:

- Use `create_client(fmc_profile, domain_uuid_override=domain_uuid)` for consistent auth and profile handling
- Call `await client.ensure_domain_uuid()` before using domain-scoped endpoints
- Return `Dict[str, Any]` or `str` (JSON) for tool output
- Catch exceptions and return structured errors instead of raising

---

## Step 4: Update the Server Instructions

Update `SERVER_INSTRUCTIONS` in `server.py` so clients know about the new tool:

```python
SERVER_INSTRUCTIONS = (
    "Use list_fmc_profiles first to discover available FMC instances (env mode exposes a single "
    "default profile). Pass the chosen profile id/alias as fmc_profile or omit it to let the server "
    "use its default. All tools are read-only:\n"
    "• list_ftd_devices lists all FTD devices managed by the FMC.\n"   # <-- Add this line
    "• find_rules_by_ip_or_fqdn searches one access policy by id.\n"
    # ... rest of instructions
)
```

---

## Step 5: Add the Tool Schema (Optional)

For Cursor MCP discovery, add a JSON schema in `mcps/user-fmc-mcp/tools/`. Create `list_ftd_devices.json`:

```json
{
  "name": "list_ftd_devices",
  "description": "List all FTD (Firepower Threat Defense) devices managed by this FMC.",
  "arguments": {
    "type": "object",
    "properties": {
      "domain_uuid": {
        "anyOf": [
          { "type": "string" },
          { "type": "null" }
        ],
        "default": null,
        "description": "Optional domain UUID override"
      },
      "fmc_profile": {
        "anyOf": [
          { "type": "string" },
          { "type": "null" }
        ],
        "default": null,
        "description": "Optional FMC profile id/alias"
      }
    },
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "devices": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "id": { "type": "string" },
            "name": { "type": "string" },
            "hostName": { "type": "string" },
            "type": { "type": "string" },
            "model": { "type": "string" },
            "healthStatus": { "type": "string" },
            "managementState": { "type": "string" },
            "deploymentStatus": { "type": "string" },
            "isConnected": { "type": "boolean" },
            "accessPolicy": {
              "type": "object",
              "properties": {
                "id": { "type": "string" },
                "name": { "type": "string" }
              }
            },
            "ftdMode": { "type": "string" }
          }
        }
      },
      "count": { "type": "integer" }
    }
  }
}
```

The schema describes the tool name, description, arguments, and optional output structure.

---

## Docker & Deployment

### If You Use Docker (Default Cursor Config)

After adding or changing a tool:

1. **Rebuild the image:**
   ```bash
   cd /path/to/fmc-mcp
   docker build -t fmc-mcp-server -f docker/Dockerfile .
   ```

2. **If you see a container name conflict:**
   ```bash
   docker rm -f fmc-mcp-container
   ```

3. **Reload Cursor** (or restart the MCP server).

The new tool will appear in the tool list (e.g., 5 tools instead of 4).

### If You Run Without Docker

Code changes take effect after reloading Cursor. No rebuild needed.

---

## Quick Reference: File Locations

| What | Where |
|------|-------|
| Server & tools | `sfw_mcp_fmc/server.py` |
| FMC API client | `sfw_mcp_fmc/fmc/client.py` |
| Config | `sfw_mcp_fmc/config.py` |
| Tool schemas (Cursor) | `mcps/user-fmc-mcp/tools/*.json` |

---

## Common Patterns

### Client method for a new list endpoint

```python
async def list_my_resource(
    self,
    *,
    limit: int = 1000,
    hard_page_limit: int = 5,
    expanded: bool = True,
    offset: int = 0,
) -> List[Dict[str, Any]]:
    return await self._list_paginated(
        "/your/resource/path",
        limit=limit,
        hard_page_limit=hard_page_limit,
        expanded=expanded,
        start_offset=offset,
    )
```

### Tool with optional profile/domain

```python
@mcp.tool()
async def my_new_tool(
    domain_uuid: Optional[str] = None,
    fmc_profile: Optional[str] = None,
) -> Dict[str, Any]:
    """Description of what the tool does."""
    try:
        client = create_client(fmc_profile, domain_uuid_override=domain_uuid)
        await client.ensure_domain_uuid()
        # ... call client methods and return result
        return {"result": data}
    except Exception as exc:
        logger.exception("Unexpected error in my_new_tool")
        return {"error": {"category": "UNEXPECTED", "message": str(exc)}}
```

---

## Testing

Run the test suite:

```bash
cd /path/to/fmc-mcp
python -m pytest tests/ -v
```

Then restart the FMC MCP server (and rebuild the Docker image if applicable) and invoke the new tool from Cursor.

---

## Summary Checklist

- [ ] Identify FMC REST API endpoint
- [ ] Add or use client method in `fmc/client.py`
- [ ] Add `@mcp.tool()` function in `server.py`
- [ ] Update `SERVER_INSTRUCTIONS`
- [ ] Add tool schema JSON (optional, for Cursor)
- [ ] Run tests
- [ ] **Rebuild Docker image** (if using Docker)
- [ ] **Remove stuck container** if you see a name conflict (`docker rm -f fmc-mcp-container`)
- [ ] Reload Cursor and verify the new tool appears
