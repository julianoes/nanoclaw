---
name: capabilities
description: Show what this NanoClaw instance can do — installed skills, available tools, and system info. Read-only. Use when the user asks what the bot can do, what's installed, or runs /capabilities.
---

# /capabilities — System Capabilities Report

Generate a structured read-only report of what this NanoClaw instance can do.

## How to gather the information

Run these commands and compile the results into the report format below.

### 1. Installed skills

List skill directories available to you:

```bash
ls -1 /home/node/.claude/skills/ 2>/dev/null || echo "No skills found"
```

Each directory is an installed skill. The directory name is the skill name (e.g., `agent-browser` → `/agent-browser`).

### 2. Available tools

Read the allowed tools from your SDK configuration. You always have access to:
- **Core:** Bash, Read, Write, Edit, Glob, Grep
- **Web:** WebSearch, WebFetch
- **Orchestration:** Task/Agent subagents (the built-in SendMessage is disabled — use mcp__nanoclaw__send_message)
- **Other:** TodoWrite, ToolSearch, Skill, NotebookEdit
- **MCP:** mcp__nanoclaw__* (messaging, questions, agents, self-modification)
- **CLI:** `ncl` for tasks and group administration (unless disabled for this group)

### 3. MCP server tools

The NanoClaw MCP server exposes these tools (via `mcp__nanoclaw__*` prefix):
- `send_message` — send a message to a named destination
- `send_file` — send a file to a named destination
- `send_card` — send a structured card
- `edit_message` — edit a previously sent message
- `add_reaction` — react to a message
- `ask_user_question` — ask the user a question with options
- `create_agent` — create a new agent group
- `install_packages` — request apt/npm packages for this group's image (admin approval)
- `add_mcp_server` — request a new MCP server for this group (admin approval)

Scheduled tasks are not MCP tools — they are managed with `ncl tasks list/create/update/pause/resume/cancel/delete/run`.

### 4. Container skills (Bash tools)

Check for executable tools in the container:

```bash
which agent-browser 2>/dev/null && echo "agent-browser: available" || echo "agent-browser: not found"
```

### 5. Group info

```bash
ls /workspace/agent/memory/ >/dev/null 2>&1 && echo "Group memory: $(find /workspace/agent/memory -type f | wc -l | tr -d ' ') files" || echo "Group memory: no"
ls /workspace/extra/ 2>/dev/null && echo "Extra mounts: $(ls /workspace/extra/ 2>/dev/null | wc -l | tr -d ' ')" || echo "Extra mounts: none"
```

## Report format

Present the report as a clean, readable message. Example:

```
📋 *NanoClaw Capabilities*

*Installed Skills:*
• /agent-browser — Browse the web, fill forms, extract data
• /capabilities — This report
(list all found skills)

*Tools:*
• Core: Bash, Read, Write, Edit, Glob, Grep
• Web: WebSearch, WebFetch
• Orchestration: Task/Agent subagents
• MCP: send_message, send_file, send_card, edit_message, add_reaction, ask_user_question, create_agent, install_packages, add_mcp_server
• CLI: ncl (tasks, groups)

*Container Tools:*
• agent-browser: ✓

*System:*
• Group memory: N files / no
• Extra mounts: N directories
```

Adapt the output based on what you actually find — don't list things that aren't installed.

**See also:** `/status` for a quick health check of session, workspace, and tasks.
