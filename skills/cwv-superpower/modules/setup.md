# CoreDash MCP Setup

Do not skip steps. Walk the user through each step and wait for confirmation before moving to the next.

---

## Step 1: Detect the MCP client

Ask the user which tool they are working in:

> "Which AI coding tool are you using?"
> - Claude Desktop
> - Claude Code (terminal)
> - Cursor
> - Windsurf
> - Gemini CLI
> - Other

Store the answer for Step 4.

---

## Step 2: Check for a CoreDash account

Ask the user:

> "Do you have a CoreDash account at app.coredash.app?"

- If yes: proceed to Step 3.
- If no: say the following and wait:

> "You'll need a free CoreDash account first. Go to https://coredash.app/signup and create one. It takes about two minutes. Come back here when you're in."

---

## Step 3: Generate an API key

Tell the user:

> "Now let's generate an API key. Inside the CoreDash app:
> 1. Open the project you want to connect
> 2. Go to **Project Settings**
> 3. Click the **API Keys (MCP)** tab
> 4. Give the key a name (e.g. 'Claude Code' or 'Cursor')
> 5. Click **Generate API Key**
> 6. Copy the key now — it starts with `cdk_` and is shown exactly once"

Ask: "Do you have your API key copied?"

Wait for confirmation before continuing.

---

## Step 4: Configure MCP client

Provide the exact config for the tool the user chose in Step 1.

### Claude Code (terminal)

Run this command, replacing the key:

```bash
claude mcp add --transport http coredash https://app.coredash.app/api/mcp \
  --header "Authorization: Bearer cdk_YOUR_API_KEY"
```

### Claude Desktop

Open this file:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add this inside the `mcpServers` object (create the file if it does not exist):

```json
{
  "mcpServers": {
    "coredash": {
      "url": "https://app.coredash.app/api/mcp",
      "headers": {
        "Authorization": "Bearer cdk_YOUR_API_KEY"
      }
    }
  }
}
```

Replace `cdk_YOUR_API_KEY` with the key they copied. Then restart Claude Desktop.

### Cursor

Open Settings → MCP → Add Server. Set:
- Name: `coredash`
- Type: HTTP
- URL: `https://app.coredash.app/api/mcp`
- Header: `Authorization: Bearer cdk_YOUR_API_KEY`

### Windsurf

Open Settings → AI → MCP Servers → Add. Set:
- Name: `coredash`
- Type: HTTP
- URL: `https://app.coredash.app/api/mcp`
- Header: `Authorization: Bearer cdk_YOUR_API_KEY`

### Gemini CLI

Add to `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "coredash": {
      "url": "https://app.coredash.app/api/mcp",
      "headers": {
        "Authorization": "Bearer cdk_YOUR_API_KEY"
      }
    }
  }
}
```

### Other MCP clients

Tell the user:

> "Any MCP-compatible client needs two things:
> - Endpoint URL: `https://app.coredash.app/api/mcp`
> - Header: `Authorization: Bearer cdk_YOUR_API_KEY`
>
> Check your client's documentation for how to add an HTTP MCP server with a custom header."

---

## Step 5: Verify the connection

Once the user confirms they've saved the config and restarted their client, silently call `get_metrics` with no arguments. Do not announce the test beforehand.

- **If it returns data** (LCP, INP, CLS, FCP, TTFB values): say

> "Connected. CoreDash MCP is live — I can now see your real Core Web Vitals data."

Then tell the user which skill they were trying to run originally and offer to continue with it.

- **If it returns `-32001` Authentication Error**: say

> "The API key looks wrong. Check that it starts with `cdk_`, there are no extra spaces, and the `Bearer` prefix is included. If the key was lost, generate a new one from Project Settings."

- **If the tool is not found or unavailable**: say

> "The MCP server isn't loading yet. Try fully quitting and restarting your AI client (not just reloading). If you're on Claude Code, run `claude mcp list` to confirm the server appears."

---

## Notes (share only if the user asks)

- Each API key is scoped to one project. If you have multiple sites, generate a separate key per project.
- The MCP server is read-only. It cannot modify your site or your CoreDash data.
- To revoke a key, go to Project Settings → API Keys (MCP) and click Revoke.
- Rate limits per plan (reset at midnight UTC):

| Plan | Daily MCP requests |
|---|---|
| Trial | 30 |
| Starter | 100 |
| Standard | 500 |
| Pro | 1,000 |
| Enterprise | 50,000 |
