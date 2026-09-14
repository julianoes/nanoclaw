---
name: status
description: Quick read-only health check — session context, workspace mounts, tool availability, and task snapshot. Use when the user asks for system status or runs /status.
---

# /status — System Status Check

Generate a quick read-only status report of the current agent environment.

## How to gather the information

Run the checks below and compile results into the report format.

### 1. Session context

```bash
echo "Timestamp: $(date)"
echo "Working dir: $(pwd)"
```

### 2. Workspace and mount visibility

```bash
echo "=== Workspace ==="
ls /workspace/ 2>/dev/null
echo "=== Agent folder ==="
ls /workspace/agent/ 2>/dev/null | head -20
echo "=== Extra mounts ==="
ls /workspace/extra/ 2>/dev/null || echo "none"
```

### 3. Tool availability

Confirm which tool families are available to you:

- **Core:** Bash, Read, Write, Edit, Glob, Grep
- **Web:** WebSearch, WebFetch
- **Orchestration:** Task/Agent subagents (the built-in SendMessage is disabled — use mcp__nanoclaw__send_message)
- **MCP:** mcp__nanoclaw__* (send_message, send_file, send_card, edit_message, add_reaction, ask_user_question, create_agent, install_packages, add_mcp_server)

### 4. Container utilities

```bash
which agent-browser 2>/dev/null && echo "agent-browser: available" || echo "agent-browser: not installed"
node --version 2>/dev/null
claude --version 2>/dev/null
```

### 5. Task snapshot

Scheduled tasks are managed with `ncl tasks`:

```bash
ncl tasks list
```

If no tasks exist, report "No scheduled tasks." If `ncl` is unavailable (CLI access disabled for this group), report "Tasks: unavailable (ncl disabled)".

## Report format

Present as a clean, readable message:

```
🔍 *NanoClaw Status*

*Session:*
• Time: 2026-03-14 09:30 UTC
• Working dir: /workspace/agent

*Workspace:*
• Agent folder: ✓ (N files)
• Extra mounts: none / N directories

*Tools:*
• Core: ✓  Web: ✓  Orchestration: ✓  MCP: ✓

*Container:*
• agent-browser: ✓ / not installed
• Node: vXX.X.X
• Claude Code: vX.X.X

*Scheduled Tasks:*
• N active tasks / No scheduled tasks
```

Adapt based on what you actually find. Keep it concise — this is a quick health check, not a deep diagnostic.

**See also:** `/capabilities` for a full list of installed skills and tools.
