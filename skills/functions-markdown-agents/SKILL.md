---
name: functions-markdown-agents
description: "Converts a declarative markdown-based agent project (AGENTS.md, skills, MCP servers, Python tools) into a deployable Azure Functions app using azd up."
metadata:
  author: anthonychu
  version: "1.0"
---

# Deploy Markdown-Based Agents to Azure Functions

This skill converts a local declarative markdown-based agent project into a deployable Azure Functions app. The converted app follows the structure of the [functions-markdown-agent sample](https://github.com/Azure-Samples/functions-markdown-agent). The user runs `azd up` to deploy.

## When to Use

- User has an `AGENTS.md` file and wants to run their agent in the cloud
- User wants to share their local Copilot agent with their team
- User wants to deploy an agent as an HTTP API or MCP server
- User asks about hosting a markdown agent on Azure

## Reference: Target Project Structure

Two layouts are supported. Choose based on the user's existing project.

### Layout A: `src/` Subfolder (Default — matches sample repo)

Use when the agent project can live in a dedicated `src/` subfolder. This is the cleanest separation between agent code and infrastructure.

```
<project-root>/
├── azure.yaml                    # azd project definition
├── infra/
│   ├── main.bicep                # Main Bicep template
│   ├── main.parameters.json      # Bicep parameters
│   ├── abbreviations.json        # Resource naming abbreviations
│   ├── app/
│   │   ├── api.bicep             # Function app definition
│   │   ├── rbac.bicep            # Role assignments
│   │   ├── foundry.bicep         # AI Foundry model deployment
│   │   ├── vnet.bicep            # Optional VNet
│   │   └── storage-PrivateEndpoint.bicep
│   ├── hooks/
│   │   └── prepackage.sh         # Copies src → infra/tmp for packaging
│   └── assets/                   # Runtime assets merged during packaging (copied from sample repo, do not modify)
│       └── (host.json, function_app.py, etc. — maintained separately from the user's agent project)
└── src/                          # The agent project — pure Copilot project, no cloud code
    ├── AGENTS.md                 # Agent instructions + optional YAML frontmatter
    ├── .github/
    │   └── skills/               # Agent skills (each skill is a folder with SKILL.md)
    │       └── <skill-name>/
    │           └── SKILL.md
    ├── .vscode/
    │   └── mcp.json              # MCP server configurations (remote/HTTP only for cloud)
    ├── tools/                    # Custom Python tools (plain functions, no framework needed)
    │   └── <tool_name>.py
    └── requirements.txt          # Python dependencies for custom tools
```

### Layout B: Root Layout

Use when the agent project already lives at the repo root (e.g., the user's `AGENTS.md`, `.github/skills/`, `.vscode/mcp.json` are at the top level). The `infra/` and `azure.yaml` are added alongside the existing agent files.

```
<project-root>/
├── AGENTS.md                     # Agent instructions + optional YAML frontmatter
├── .github/
│   └── skills/                   # Agent skills
│       └── <skill-name>/
│           └── SKILL.md
├── .vscode/
│   └── mcp.json                  # MCP server configurations (remote/HTTP only for cloud)
├── tools/                        # Custom Python tools
│   └── <tool_name>.py
├── requirements.txt              # Python dependencies for custom tools
├── azure.yaml                    # azd project definition
└── infra/                        # Infrastructure (added for deployment)
    ├── main.bicep
    ├── main.parameters.json
    ├── abbreviations.json
    ├── app/
    │   ├── api.bicep
    │   ├── rbac.bicep
    │   ├── foundry.bicep
    │   ├── vnet.bicep
    │   └── storage-PrivateEndpoint.bicep
    ├── hooks/
    │   └── prepackage.sh         # Copies root agent files → infra/tmp for packaging
    └── assets/                   # Runtime assets (copied from sample repo, do not modify)
        └── (host.json, function_app.py, etc.)
```

## Step-by-Step Conversion Instructions

### Step 1: Analyze the User's Existing Project

Examine the user's workspace for:

1. **`AGENTS.md`** — the agent's instructions. This is required.
2. **Skills** — may be in `.github/skills/`, `skills/`, or another location.
3. **`mcp.json`** — may be in `.vscode/mcp.json` or elsewhere. Look for MCP server definitions.
4. **Tools or scripts** — any custom code the agent relies on.
5. **Any existing Azure configuration** — Bicep, ARM templates, terraform, etc. If found, warn the user that the `infra/` directory from the sample repo will replace any existing infrastructure. Back up or rename their existing `infra/` directory before proceeding.

If there is no `AGENTS.md`, ask the user to provide one or help them create it.

**Choose the layout**:
- If the agent files (`AGENTS.md`, `.github/skills/`, `.vscode/mcp.json`, `tools/`) are already at the repo root and the user wants to keep them there → use **Layout B (root)**.
- If starting fresh or the user prefers a clean separation → use **Layout A (`src/` subfolder)**.
- If in doubt, prefer Layout B (root) when the user already has agent files at the root to minimize disruption.

### Step 2: Create the Project Structure

Scaffold the deployment infrastructure from the [sample repo](https://github.com/Azure-Samples/functions-markdown-agent).

**How to get the sample repo files**: Clone the repo with `git clone https://github.com/Azure-Samples/functions-markdown-agent.git` into a temporary directory or use GitHub tools (e.g., `mcp_github-mcp_get_file_contents`) to fetch individual files.

1. Copy `azure.yaml` from the sample repo — adjust the `name` field to match the user's project name. For **root layout**, you'll also need a modified `prepackage.sh` (see Step 8).
2. Copy the entire `infra/` directory from the sample repo, including:
   - `infra/main.bicep`, `infra/main.parameters.json`, `infra/abbreviations.json`
   - `infra/app/` (all `.bicep` files: `api.bicep`, `rbac.bicep`, `foundry.bicep`, `vnet.bicep`, `storage-PrivateEndpoint.bicep`)
   - `infra/hooks/prepackage.sh` (will be replaced for root layout — see Step 8)
   - `infra/assets/` — these are runtime files (e.g., `function_app.py`, `host.json`, `package.json`, `copilot_shim/`, `public/`, `.funcignore`, `extra-requirements.txt`). They are required for deployment and must be copied from the sample repo. The user should NOT modify these files.
   - Do **not** copy `infra/main.json` — it's a compiled ARM template that gets regenerated.
3. For **Layout A (`src/`)**: create the `src/` directory and place agent files there.
4. For **Layout B (root)**: agent files stay at the repo root — no `src/` directory needed.
5. The infra does not typically need modification unless the agent needs access to additional Azure resources (see Step 6 for adding role assignments).

### Step 3: Place AGENTS.md

- **Layout A**: Copy or move the user's `AGENTS.md` into `src/AGENTS.md`.
- **Layout B**: Keep `AGENTS.md` at the project root.

#### YAML Frontmatter

Add YAML frontmatter if not already present. The frontmatter is optional but recommended:

```yaml
---
name: <agent-name>
description: <brief description of what the agent does>
---
```

#### Timer Triggers (Optional)

If the user wants scheduled/autonomous agent runs, add a `functions` array to the frontmatter:

```yaml
---
name: my-agent
description: My agent description
functions:
  - name: morningBriefing
    trigger: timer
    schedule: "0 0 8 * * *"
    prompt: "Check the latest build status and summarize any new critical issues."
---
```

Frontmatter fields for timer triggers:
- `name` — optional (auto-generated if omitted)
- `trigger` — must be `timer` (only supported trigger type currently)
- `schedule` — required, NCRONTAB expression (`{second} {minute} {hour} {day} {month} {day-of-week}`)
- `prompt` — required, the prompt to send to the agent on each trigger
- `logger` — optional, defaults to `true`. When enabled, logs full agent output including session_id, response, tool_calls

**Note**: Only `trigger: timer` is supported. Other trigger types will be rejected at startup. This is useful for agents that perform periodic tasks like checking build status, sending reports, monitoring systems, etc.

### Step 4: Move Skills

Skills must be in `.github/skills/<skill-name>/SKILL.md` (relative to the agent project root — that's `src/.github/skills/` for Layout A, or `.github/skills/` for Layout B).

- If the user's skills are in a different location, move them.
- Each skill is a folder containing at minimum a `SKILL.md` file.
- The folder name must match the `name` field in the SKILL.md frontmatter.
- Skills can also contain `references/` and `assets/` subdirectories.

#### Review and Sanitize Skills

When deployed to Azure Functions, skills run in a cloud environment without shell access. Review each skill and apply these rules:

1. **Remove instructions that execute shell commands or arbitrary code.** Skills must NOT tell the agent to run shell commands (e.g., `bash`, `curl`, `python`, `npm`, `git`), execute scripts, or invoke command-line tools. Any such functionality must be converted to a Python tool in `tools/` (see Step 6).

2. **Anonymous GET requests are OK.** Skills CAN instruct the agent to fetch data from publicly accessible URLs using HTTP GET (e.g., "Fetch the JSON from `https://prices.azure.com/api/retail/prices?...`"). This works because the agent runtime supports fetching URLs directly.

3. **Non-GET HTTP methods → Python tool.** If a skill instructs the agent to make POST, PUT, PATCH, or DELETE requests, convert that logic into a Python tool in `tools/`. The tool should use `httpx` or `requests` to make the call, and the skill should reference the tool by name instead.

4. **Remove `scripts/` directories.** If a skill contains a `scripts/` folder with executable code (bash, Python, etc.), convert those scripts into Python tools in `tools/`. Remove the `scripts/` directory from the skill — it will not be executed in the cloud.

5. **`allowed-tools` in skill frontmatter.** If a skill's frontmatter has `allowed-tools` referencing `Bash(*)`, shell commands, or other local execution tools, remove those entries. Replace with references to the corresponding Python tools created in `tools/`.

**Example conversion**: If a skill says "Run `curl -X POST https://api.example.com/data -d '{...}'`", create a Python tool:

```python
# tools/example_api.py
import httpx
from pydantic import BaseModel, Field

class PostDataParams(BaseModel):
    payload: str = Field(description="JSON payload to send")

async def post_to_example_api(params: PostDataParams) -> str:
    """Post data to the example API."""
    async with httpx.AsyncClient() as client:
        response = await client.post("https://api.example.com/data", content=params.payload,
                                     headers={"Content-Type": "application/json"})
        return response.text
```

Then update the skill instructions to say: "Use the **post_to_example_api** tool to send data to the API."

### Step 5: Handle MCP Servers

Place MCP configuration in `.vscode/mcp.json` (relative to the agent project root — that's `src/.vscode/mcp.json` for Layout A, or `.vscode/mcp.json` for Layout B).

**IMPORTANT: Only remote (HTTP-based) MCP servers are supported in the cloud deployment.**

The format for a remote MCP server:

```json
{
  "servers": {
    "server-name": {
      "url": "https://example.com/api/mcp",
      "type": "http"
    }
  }
}
```

#### Handling Local MCP Servers

If the user's `mcp.json` contains local MCP servers (e.g., `stdio` type, `command`-based, `npx`-based), warn them:

> **Warning**: Local MCP servers (stdio/command-based) will not work when deployed to Azure Functions. Only remote HTTP-based MCP servers are supported in the cloud. Consider one of these options:
> 1. Find and use a remote/hosted version of the MCP server
> 2. Convert the MCP server's functionality into a Python tool in `tools/` (see Step 6)
> 3. Keep the local server in `mcp.json` for local development, but it will be ignored in the cloud

When converting a local MCP server to a Python tool:

1. Identify the tools/functions the MCP server provides
2. Create equivalent Python functions in `tools/`
3. The agent runtime will auto-discover and register these as tools

### Step 6: Create Python Tools

For any custom functionality the agent needs (especially when migrating from local MCP servers), create Python tool files in `tools/` (that's `src/tools/` for Layout A, or `tools/` at the root for Layout B).

#### Tool File Structure

Each `.py` file in `tools/` should contain:

1. A Pydantic `BaseModel` for typed parameters
2. An async function that takes the params model and returns a string
3. A clear docstring (becomes the tool description)

```python
from pydantic import BaseModel, Field


class MyToolParams(BaseModel):
    param1: str = Field(description="Description of param1")
    param2: int = Field(description="Description of param2")


async def my_tool(params: MyToolParams) -> str:
    """Description of what this tool does. This becomes the tool description."""
    # Tool implementation
    result = f"Processed {params.param1} with {params.param2}"
    return result
```

#### Tool Discovery Rules

- The runtime scans `tools/*.py` at startup
- Functions starting with `_` are excluded
- Only ONE public function per file is registered — put each tool in its own `.py` file
- If a tool fails to import, the runtime logs the error and continues
- The function docstring becomes the tool description (fallback: `Tool: <function_name>`)
- Custom tools only run in the cloud — they don't work in local Copilot Chat

#### Accessing Azure Resources from Tools

If a tool needs to access Azure resources (Cosmos DB, Storage, Key Vault, etc.), use `DefaultAzureCredential` with the managed identity — never hardcode credentials. You'll also need to update the Bicep infrastructure to deploy the resource, add role assignments, and pass endpoints as app settings.

See [the Azure resources reference](references/AZURE-RESOURCES.md) for the full pattern with code examples and Bicep templates.

### Step 7: Add requirements.txt

Create `requirements.txt` (in `src/` for Layout A, or at the project root for Layout B) with dependencies needed by custom tools:

```
# Only needed if tools require extra dependencies
pydantic>=2.0.0
```

Add any additional packages the tools import (e.g., `azure-identity`, `azure-cosmos`, `requests`, `httpx`, etc.).

### Step 8: Set Up azure.yaml and prepackage.sh

Copy `azure.yaml` from the sample repo and adjust the `name` field. The `azure.yaml` is the same for both layouts — see [the template](references/TEMPLATES.md) for the exact content. Both layouts use `./infra/tmp` as the deployment source.

The `prepackage.sh` hook assembles agent files + runtime assets into `infra/tmp/`:

- **Layout A**: Use the default `prepackage.sh` from the sample repo. It copies `src/` → `infra/tmp/` and merges `infra/assets/`.
- **Layout B**: Replace `infra/hooks/prepackage.sh` with a version that copies from the project root, skipping `infra/`, `azure.yaml`, `.git/`, `.azure/`, etc.

See [the prepackage scripts reference](references/PREPACKAGE-SCRIPTS.md) for both full scripts.

### Step 9: Set Up .gitignore

Add entries to `.gitignore` (create it if it doesn't exist, or merge into the existing one) to exclude Python bytecode, venvs, Azure Functions local artifacts, secrets, IDE files, `.azure/`, `tmp/`, `infra/main.json`, and specific `infra/assets/` sub-files.

See [the templates reference](references/TEMPLATES.md) for the full `.gitignore` template.

### Step 10: Verify and Summarize

After conversion, verify (paths are relative to agent project root — `src/` for Layout A, repo root for Layout B):

- [ ] `AGENTS.md` exists with agent instructions
- [ ] Skills are in `.github/skills/<name>/SKILL.md`
- [ ] `.vscode/mcp.json` contains only remote HTTP MCP servers (warn about any local ones)
- [ ] Custom tools are in `tools/` with proper Pydantic params + async functions
- [ ] `requirements.txt` lists all tool dependencies
- [ ] `azure.yaml` is at the project root with correct hooks
- [ ] `infra/hooks/prepackage.sh` matches the chosen layout (copies from `src/` or from root)
- [ ] `infra/` directory has all Bicep files and `infra/assets/` from the sample repo
- [ ] If tools access Azure resources: DefaultAzureCredential is used, role assignments are in rbac.bicep

Tell the user:

1. **Deploy**: Run `azd auth login` (if not already authenticated), then `azd up` from the project root
2. **During deployment** they'll be prompted for:
   - Azure location
   - GitHub PAT with Copilot Requests permission ([create one here](https://github.com/settings/personal-access-tokens/new))
   - Model selection (GitHub models like `github:<model-name>` require no extra infra; Foundry models deploy an AI Foundry account)
   - Whether to enable VNet integration
3. **After deployment**:
   - Chat UI is at the root URL: `https://<app>.azurewebsites.net/`
   - Chat API: `POST /agent/chat` (JSON) and `POST /agent/chatstream` (SSE streaming). Use `x-ms-session-id` header for multi-turn conversations.
   - MCP server: `/runtime/webhooks/mcp` (requires a system key in `x-functions-key` header — find it in Azure portal > Function App > App keys > System keys, or via `az functionapp keys list --name <app> --resource-group <rg> --query systemKeys -o json`)
4. **Local development**: Open the agent project folder in VS Code (`src/` for Layout A, or the repo root for Layout B), enable `chat.useAgentSkills`, and chat with the agent in Copilot Chat (note: Python tools in `tools/` only work after cloud deployment)

## Troubleshooting

- **Use `azd up`, not `azd provision` + `azd deploy` separately** — the hooks don't run correctly when split
- **Windows is not supported** — the packaging hooks are `.sh` scripts; use macOS, Linux, or WSL
- **Python tools don't work locally** — they only execute in the cloud runtime after deployment
- **Python version** — the Azure Functions runtime uses Python 3.11; ensure tools are compatible
- **Model change after deployment**: Run `azd env set MODEL_SELECTION "github:gpt-5.2"` then `azd up`