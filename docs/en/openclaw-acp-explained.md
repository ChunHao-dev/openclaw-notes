# Understanding ACP (Agent Client Protocol) in OpenClaw

> **Version Info**  
> - OpenClaw: 2026.3.8  
> - ACPX: 0.1.15  
> - Last Updated: 2026-03-09

## TL;DR

- **ACP** is an open protocol that connects IDEs and AI coding agents, and OpenClaw uses it to support IDE integration (like Zed Editor) and chat interfaces
- **ACPX** is OpenClaw's ACP implementation engine, supporting coding agents like Codex/Claude/Gemini/Pi
- **Three-layer architecture**: IDE/Chat → Gateway → ACPX Runtime → Coding Agents
- **Main uses**: IDE integration, chat interface, programmatic calls

## What is ACP?

**What can you do with ACP?**
- Call Codex/Claude directly from Zed/VS Code
- Use professional coding agents in Discord/Telegram
- Integrate once, support multiple AI tools

**ACP (Agent Client Protocol)** is the open protocol that makes this possible. It standardizes communication between "editors/IDEs" and "AI coding agents."

- Official website: https://agentclientprotocol.com/
- GitHub: https://github.com/agentclientprotocol/agent-client-protocol
- Similar concept: MCP (Model Context Protocol), but ACP focuses on coding scenarios

## Why Do We Need ACP?

### Problem: Every AI coding tool has its own interface

```
Codex CLI     → Own command format
Claude Code   → Own command format  
Gemini CLI    → Own command format
Pi Agent      → Own command format
```

If you want to integrate these AIs into your tool, you need to:
- Write different integration code for each tool
- Handle different input/output formats
- Maintain multiple PTY (terminal) capture logic

### Solution: ACP unified interface

```
Your Tool → ACP Protocol → Any ACP-compatible AI agent
```

Implement ACP once, and you can support all ACP-compatible AI agents!

## OpenClaw's ACP Architecture

### Three-layer architecture

```
External Interface (IDE/Chat)
         ↓
OpenClaw Gateway (Routing Hub)
         ↓
ACPX Runtime (ACP Implementation)
         ↓
Coding Agents (Codex/Claude/...)
```

**Simply put:**
- **Gateway**: Unified message routing hub
- **ACPX**: ACP protocol implementation engine
- **Agents**: AI tools that actually execute coding tasks

## What is ACPX?

**ACPX** = ACP eXecution/eXtension

- npm package: `acpx@0.1.15`
- Purpose: CLI client implementation of ACP protocol
- Exists as a plugin in OpenClaw (`extensions/acpx/`)

### Agents Supported by ACPX

| Agent Name | Underlying Tool | Command |
|-----------|---------|------|
| `pi` | Pi Coding Agent | `npx pi-acp` |
| `claude` | Claude Code (Anthropic) | `npx -y @zed-industries/claude-agent-acp` |
| `codex` | Codex CLI (OpenAI) | `npx @zed-industries/codex-acp` |
| `opencode` | OpenCode | `npx -y opencode-ai acp` |
| `gemini` | Gemini CLI (Google) | `gemini` |

## Use Cases

### Case 1: Chat Interface (Discord/Telegram)

```bash
# Use in Discord
/acp spawn codex --mode persistent --thread auto

# Use in Telegram
/acp spawn claude --mode persistent
```

**Flow:**
```
Discord/Telegram Message
  ↓
OpenClaw Gateway
  ↓
ACPX Runtime
  ↓
Codex CLI / Claude Code
```

### Case 2: Programmatic Calls

```typescript
// Via sessions_spawn tool
sessions_spawn({
  runtime: "acp",
  agentId: "codex",
  message: "Fix the bug in auth.ts"
})
```

## Configuration Examples

### Enable ACP Runtime

```json
{
  "acp": {
    "enabled": true,
    "backend": "acpx",
    "defaultAgent": "codex"
  }
}
```

### Configure ACPX Plugin

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "config": {
          "permissionMode": "approve-reads",
          "timeoutSeconds": 300,
          "mcpServers": {
            "playwright": {
              "command": "npx",
              "args": ["-y", "@playwright/mcp"]
            }
          }
        }
      }
    }
  }
}
```

### Permission Modes

- `approve-all`: Auto-approve all operations
- `approve-reads`: Auto-approve reads, ask for writes (default)
- `deny-all`: Deny all operations

## Common Commands

### Create ACP Session

```bash
# Create persistent session
/acp spawn codex --mode persistent --thread auto

# One-time execution
/acp spawn claude --mode oneshot
```

### Manage Session

```bash
# Check status
/acp status

# Set model
/acp model anthropic/claude-3-5-sonnet-20241022

# Set permissions
/acp permissions approve-all

# Set timeout
/acp timeout 600

# Cancel current task
/acp cancel

# Close session
/acp close
```

### Steer Session

```bash
# Adjust direction without resetting context
/acp steer "focus on performance optimization"
```

## Thread Binding

ACP sessions can bind to chat threads:

```bash
# In Discord thread
/acp spawn codex --mode persistent --thread auto
```

**Effect:**
- All subsequent messages in that thread route to the same ACP session
- Output returns to the same thread
- Until session closes or thread expires

**Supported platforms:**
- Discord threads/channels
- Telegram forum topics

## Security Notes

### Permission Mode Selection

- **Development environment**: Can use `approve-all` for efficiency
- **Production environment**: Recommend `approve-reads` or stricter settings
- **Shared use**: Must set `deny-all` and manually review each operation

### Sandbox Limitations

⚠️ **Important**: ACP sessions currently run in **host runtime**, not inside OpenClaw sandbox

- Cannot spawn ACP sessions from sandboxed sessions
- ACP agents can access host file system
- If sandbox execution needed, use `runtime="subagent"`

### Token Security

```bash
# ❌ Unsafe: token appears in process list
openclaw acp --token <token>

# ✅ Safe: use file
openclaw acp --token-file ~/.openclaw/gateway.token

# ✅ Safe: use environment variable
export OPENCLAW_GATEWAY_TOKEN=<token>
openclaw acp
```

## Troubleshooting

### Symptom: `ACP runtime backend is not configured`
- **Possible cause**: Backend plugin not installed or not enabled
- **Solution**:
  ```bash
  openclaw plugins install acpx
  openclaw config set plugins.entries.acpx.enabled true
  /acp doctor
  ```

### Symptom: `ACP is disabled by policy (acp.enabled=false)`
- **Possible cause**: ACP feature globally disabled
- **Solution**:
  ```bash
  openclaw config set acp.enabled true
  # Restart Gateway
  openclaw gateway restart
  ```

### Symptom: IDE connection fails or no response
- **Possible causes**:
  1. Gateway not started
  2. URL/token settings incorrect
  3. Network connection issues
- **Solution**:
  ```bash
  # 1. Check Gateway status
  openclaw gateway status
  
  # 2. Test ACP bridge
  openclaw acp client
  
  # 3. Check connection settings
  openclaw config get gateway.remote.url
  ```

### Symptom: `Sandboxed sessions cannot spawn ACP sessions`
- **Possible cause**: Trying to start ACP from sandboxed session
- **Solution**: ACP currently only runs in host runtime, use `runtime="subagent"` or start from non-sandbox session

### Symptom: `Permission prompt unavailable in non-interactive mode`
- **Possible cause**: `permissionMode` setting too strict
- **Solution**:
  ```bash
  # Allow all operations (development environment)
  openclaw config set plugins.entries.acpx.config.permissionMode approve-all
  
  # Or set graceful degradation
  openclaw config set plugins.entries.acpx.config.nonInteractivePermissions deny
  ```

## Summary

### Value of ACP

1. **Unified interface**: One codebase supports multiple AI agents
2. **Standard protocol**: No need to handle PTY capture yourself
3. **Flexible integration**: Can use from IDE, chat, CLI
4. **Permission control**: Fine-grained permission management
5. **Session management**: Persistent conversations, parallel workflows

### Key Concepts

- **ACP** = Protocol standard (like HTTP)
- **ACPX** = OpenClaw's ACP implementation engine
- **Agents** = AI tools that actually execute coding tasks (Codex/Claude/Gemini/Pi)

## References

- ACP Official Website: https://agentclientprotocol.com/
- ACPX npm: https://www.npmjs.com/package/acpx
- OpenClaw ACP Documentation: https://docs.openclaw.ai/tools/acp-agents
- OpenClaw Configuration Reference: https://docs.openclaw.ai/gateway/configuration-reference
