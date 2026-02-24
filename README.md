# Azure Functions Markdown Agents Skill

Deploy declarative markdown-based agent projects (AGENTS.md, skills, MCP servers, Python tools) to Azure Functions with `azd up`.

## Install

```
npx skills add https://github.com/anthonychu/functions-skills --skill functions-markdown-agents
```

## What It Does

Converts a local Copilot agent project into a deployable Azure Functions app based on the [functions-markdown-agent](https://github.com/Azure-Samples/functions-markdown-agent) sample. Handles project restructuring, Bicep infrastructure, MCP server migration, Python tool scaffolding, and DefaultAzureCredential setup.

## Learn More

See the [SKILL.md](skills/functions-markdown-agents/SKILL.md) for the full conversion instructions.
