# Langfuse MCP Quickstart

Langfuse provides a built-in Model Context Protocol (MCP) server that allows AI assistants (like Claude Code, Cursor, and other agents) to interact with Langfuse programmatically.

This guide explains how to start a local, out-of-the-box instance of Langfuse via Docker Compose, and how to connect other tools to its MCP server.

## 1. Start Langfuse Locally

Langfuse can be started in minutes using the provided Docker Compose configuration.

1. Ensure you have Docker and Docker Compose installed.
2. Clone this repository if you haven't already:
   ```bash
   git clone --depth=1 https://github.com/langfuse/langfuse.git
   cd langfuse
   ```
3. Start the Langfuse services:
   ```bash
   docker compose up -d
   ```
   This will start the necessary services (Web UI, Worker, PostgreSQL, ClickHouse, MinIO, Redis) and bind the Langfuse Web UI to port `3000`.
   *(Note: The first startup may take a few minutes as images are downloaded and databases are initialized.)*

4. Verify it's running by opening `http://localhost:3000` in your browser. You can sign up with any email/password (local instance only).

## 2. Get API Keys

To connect to the MCP server, you need a project-scoped API key.

1. Log in to your local Langfuse instance at `http://localhost:3000`.
2. Create a new project (or use the default one).
3. Navigate to **Settings > API Keys**.
4. Create new API credentials. Note down the **Public Key** (`pk-lf-...`) and **Secret Key** (`sk-lf-...`).

## 3. Connect your MCP Client

The Langfuse MCP server uses **BasicAuth** with your API keys.

First, encode your keys to base64:

```bash
echo -n "pk-lf-your-public-key:sk-lf-your-secret-key" | base64
```
*(You will get a string like `cGstbGYteHh4OnNrLWxmLXh4eA==`)*

### Example: Claude Code

Add the Langfuse MCP server to Claude Code using the HTTP transport:

```bash
claude mcp add --transport http langfuse http://localhost:3000/api/public/mcp \
    --header "Authorization: Basic {your-base64-token}"
```

### Example: Cursor

Add the following to your Cursor MCP settings (found in Cursor Settings > Features > MCP):

```json
{
  "mcp": {
    "servers": {
      "langfuse": {
        "url": "http://localhost:3000/api/public/mcp",
        "headers": {
          "Authorization": "Basic {your-base64-token}"
        }
      }
    }
  }
}
```

## 4. Test the Connection

Once configured, your agent/tool can dynamically discover the available Langfuse tools.
For example, in Claude Code, you can ask:
> "List all prompts in the Langfuse project"
or
> "List recent Langfuse observations"

The MCP server handles routing the requests to your local Langfuse instance dynamically!

---
*For more details on the MCP integration architecture, refer to `web/src/features/mcp/README.md`.*
