# OpenClaw Message Processing Flow

> **Question**: After I send a message, what is the process of handling this message? How does OpenClaw decide which skill to use and how can skills use built-in tools?

This document provides a comprehensive explanation of how messages flow through OpenClaw, from reception through routing, agent processing, skill selection, and tool invocation.

## Quick Overview

```
Web Chat Message → Inbound Monitor → Routing → Access Control →
Reply Generation → Skill Selection → Tool Registration →
Claude API Call → Tool Invocation Loop → Response Delivery
```

## Table of Contents

1. [Message Reception](#1-message-reception-webchat-example)
2. [Message Routing](#2-message-routing)
3. [Inbound Message Processing](#3-inbound-message-processing)
4. [Reply Generation](#4-reply-generation-agent-entry)
5. [Skill Selection & Invocation](#5-skill-selection--invocation)
6. [Tool Registration & Integration](#6-tool-registration--pi-agent-integration)
7. [Agent Execution](#7-agent-execution-pi-agent-embedded)
8. [Tool Invocation & Execution](#8-tool-invocation--execution)
9. [Skills System](#9-skills-system)
10. [Complete Flow Diagram](#complete-message-flow-diagram)
11. [Examples](#tool-use-examples)

---

## 1. Message Reception (WebChat Example)

**Entry Point**: `src/web/inbound/monitor.ts`

The WhatsApp/web channel listener receives incoming messages:
- `monitorWebInbox()` function connects to WhatsApp via Baileys library
- Listens to `messages.upsert` events from the WhatsApp socket
- Extracts message text, media, metadata, group info
- Creates `WebInboundMessage` object with all context
- Debounces rapid messages from same sender (configurable)
- **Key flow lines 154-341**: Builds complete inbound envelope with sender, receiver, media paths, reply context

**Output**: `WebInboundMessage` object containing:
- `body`, `from`, `to`, `mediaType`, `mediaPath`, `chatType` (group/direct)
- `sendComposing()`, `reply()`, `sendMedia()` callbacks for async responses

---

## 2. Message Routing

**File**: `src/routing/resolve-route.ts`

The gateway routes messages to agents based on config bindings:

```typescript
resolveAgentRoute(input: ResolveAgentRouteInput) → ResolvedAgentRoute
```

**Routing Priority** (lines 211-259):
1. **Peer binding** (direct DM to specific user) → `binding.peer`
2. **Parent peer** (thread inheritance from parent conversation) → `binding.peer.parent`
3. **Guild binding** (Discord guild/team server) → `binding.guild`
4. **Team binding** (Teams/workspace) → `binding.team`
5. **Account binding** (specific account within channel) → `binding.account`
6. **Wildcard account binding** (any account in channel) → `binding.channel`
7. **Default agent** (fallback) → `default`

**Output**: `ResolvedAgentRoute`:
```typescript
{
  agentId: string;           // Which agent receives the message
  channel: string;           // Normalized channel (e.g., "whatsapp")
  accountId: string;         // Account identifier
  sessionKey: string;        // Persistence key for agent's session
  mainSessionKey: string;    // For DM collapse
  matchedBy: string;         // Which rule matched
}
```

---

## 3. Inbound Message Processing

**File**: `src/web/auto-reply/monitor/process-message.ts`

Once routed, the message is processed through `processMessage()` (line 106):

**Key steps**:
1. **Access control** (line 215-217): Check if sender is allowed (pairing, allowlist)
2. **Echo detection** (line 184-193): Avoid responding to own messages
3. **Build context envelope** (line 143-305): Create message context with:
   - Combined body (including group history if group chat)
   - Inbound metadata (`From`, `To`, `SessionKey`, `CommandAuthorized`)
   - Media information
   - Group members and subject
4. **Command authorization** (line 254-256): Check if sender can execute commands
5. **Session metadata** (line 321-335): Record in-band session updates

**Output**: Context payload passed to reply system:
```typescript
ctxPayload: {
  Body: string;              // Processed message text
  From: string;              // Sender identifier
  SessionKey: string;        // Agent's session key
  CommandAuthorized?: boolean;
  MediaPath?: string;        // Path to downloaded media
  ChatType: "group" | "direct";
  // ...additional metadata
}
```

---

## 4. Reply Generation (Agent Entry)

**File**: `src/auto-reply/reply/get-reply.ts`

Entry point: `getReplyFromConfig(ctx, opts)` (line 27)

This orchestrates the entire reply pipeline:

1. **Config loading** (line 33): Load agent config, workspace
2. **Model resolution** (line 43-61): Determine which Claude model to use
3. **Session initialization** (line 105-127): Load/create session state
4. **Media & link understanding** (line 87-96): Process attachments and URLs
5. **Directive resolution** (line 144-169): Parse `/think`, `/verbose`, model directives
6. **Inline actions** (line 205-245): Handle simple commands early (`/help`, `/new`, etc.)
7. **Media staging** (line 249-255): Prepare sandbox media for tools
8. **Prepared reply execution** (line 257-305): Call `runPreparedReply()`

**File**: `src/auto-reply/reply/get-reply-run.ts` (line 107)

Function `runPreparedReply()` coordinates skill selection and agent execution.

---

## 5. Skill Selection & Invocation

**File**: `src/auto-reply/reply/get-reply-inline-actions.ts` (line 56)

Function `handleInlineActions()` processes early skill commands:

**Skill command detection** (line 14):
```typescript
import { listSkillCommandsForWorkspace, resolveSkillCommandInvocation }
  from "../skill-commands.js";
```

**Skill Lookup Flow**:
1. Call `listSkillCommandsForWorkspace()` → `src/auto-reply/skill-commands.ts` (line 25)
   - Loads all skills from workspace via `buildWorkspaceSkillCommandSpecs()`
   - Returns array of `SkillCommandSpec[]`

2. Resolve command invocation via `resolveSkillCommandInvocation()` (line 94)
   - Normalizes skill command name (e.g., `/python` → lookup table)
   - Returns matched skill + args

**Skill Command Spec** (from `src/agents/skills/workspace.ts`, line ~180):
```typescript
type SkillCommandSpec = {
  name: string;           // e.g., "python"
  skillName: string;      // e.g., "python-repl"
  description: string;
  // ... invocation details
}
```

---

## 6. Tool Registration & Pi-Agent Integration

**File**: `src/agents/pi-tools.ts` (line 113)

Function `createOpenClawCodingTools()` creates all tools available to Claude:

**Tool Categories**:

1. **Coding Tools** (base from pi-coding-agent library):
   - `read` → sandboxed or workspace read
   - `write` → sandboxed or workspace write
   - `edit` → applied patching (OpenAI models only)
   - List/directory operations

2. **Built-in OpenClaw Tools** (from `openclaw-tools.ts`, line 22):
   ```typescript
   createOpenClawTools(): AnyAgentTool[] → [
     createBrowserTool,
     createCanvasTool,
     createNodesTool,
     createCronTool,
     createMessageTool,      // For sending to channels
     createTtsTool,
     createGatewayTool,      // For arbitrary gateway calls
     createSessionsSpawnTool, // Subagents
     createWebFetchTool,
     createWebSearchTool,
     createImageTool,        // Vision understanding
   ]
   ```

3. **Bash/Exec Tool** (conditional):
   - `createExecTool()` / `createProcessTool()`
   - Gated by policy (tool allowlist/blocklist)

4. **Plugin Tools** (dynamic):
   - Loaded from `resolvePluginTools()` (line 142)
   - Tools registered via plugin skill system

**Tool Policy Filtering** (lines 172-222):

Tools are filtered by:
- **Global policy** (`config.tools.policies.global`)
- **Agent policy** (`config.agents[agentId].tools.policies`)
- **Provider policy** (e.g., Anthropic-specific blocks for OAuth)
- **Group policy** (channel/workspace-level restrictions)
- **Sandbox policy** (if running in restricted environment)

```typescript
filterToolsByPolicy(
  allTools,
  resolveEffectiveToolPolicy({ config, sessionKey, modelProvider })
)
```

---

## 7. Agent Execution (Pi-Agent Embedded)

**File**: `src/agents/pi-embedded-runner/run.ts` (line 71)

Function `runEmbeddedPiAgent()` executes the Claude API call with:

**Setup Phase**:
- Load model credentials and auth profiles
- Build system prompt with skills description
- Prepare session history (past messages)
- Validate context window

**File**: `src/agents/pi-embedded-runner/run/attempt.ts` (line ~180+)

Core agent execution in `runEmbeddedAttempt()`:
1. Create `AgentSession` from pi-coding-agent library
2. Register tools via `toClientToolDefinitions(tools)`
3. Stream Claude API response with tool use
4. Handle tool calls in a loop until final response

**Tool Invocation Loop** (within `streamSimple()`):
```
1. Claude responds with tool call: { name: "read", arguments: { path: "file.txt" } }
2. Agent invokes tool directly (synchronous)
3. Tool result returned to Claude
4. Claude continues streaming (or uses result to refine)
5. Repeat until Claude produces final text response
```

---

## 8. Tool Invocation & Execution

**File**: `src/gateway/tools-invoke-http.ts` (line 102)

Function `handleToolsInvokeHttpRequest()` is the HTTP endpoint for external tool calls:

**Tool Invocation via Gateway**:
- Tools can also be invoked via `/tools/invoke` HTTP POST
- Body contains `{ tool, args, sessionKey }`
- Resolves policies and invokes tool
- Returns result to caller

**Example: Message/Send Tool** (`openclaw-tools.ts` line 86):
```typescript
createMessageTool({
  agentSessionKey: options?.agentSessionKey,
  // Tool that sends messages to channels
  // Invokes: gateway.send(channel, text)
})
```

When Claude calls `message.send()`:
1. Tool validates session authorization
2. Calls gateway `send()` method
3. Routes to correct channel provider (Telegram, Discord, etc.)
4. Message delivered to user

---

## 9. Skills System

**File**: `src/agents/skills/workspace.ts` (line 99)

Function `loadWorkspaceSkillEntries()` discovers and loads skills:

**Skill Loading**:
- Bundled skills: `resolveBundledSkillsDir()` (built-in)
- Workspace skills: `{workspace}/skills/` directory
- Managed skills: `~/.openclaw/skills/` (installed via `npm install`)
- Plugin skills: From `config.plugins.skills`
- Extra skills: From `config.skills.load.extraDirs`

**Skill Structure** (parsed from `skill.md`):
```markdown
---
name: python
description: "Execute Python code"
type: coding
---

/python code here
```

**Skill Invocation in Agent**:
1. Skills converted to system prompt hints
2. Claude can invoke skills via special syntax: `/python code`
3. Each skill is a "slash command" Claude can call
4. Skill runner handles execution

---

## Complete Message Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    USER SENDS MESSAGE (WebChat)                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  1. INBOUND MONITOR (monitor.ts)                                    │
│  • Socket listens for messages.upsert                               │
│  • Extracts message, media, metadata                                │
│  • Creates WebInboundMessage                                        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. MESSAGE ROUTING (resolve-route.ts)                              │
│  • resolveAgentRoute({ channel, accountId, peer, ... })            │
│  • Apply binding rules (peer → guild → team → account → default)   │
│  • Return: agentId, sessionKey                                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  3. INBOUND PROCESSING (process-message.ts)                         │
│  • Access control check (pairing, allowlist)                        │
│  • Echo detection                                                    │
│  • Build message context envelope                                   │
│  • Finalize inbound context (ctx)                                   │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. REPLY GENERATION (get-reply.ts)                                 │
│  • getReplyFromConfig(ctx)                                          │
│  • Load config, model selection, workspace                          │
│  • Session state initialization                                     │
│  • Media/link understanding                                         │
│  • Directive parsing (/think, /verbose)                             │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  5. INLINE ACTIONS (get-reply-inline-actions.ts)                    │
│  • handleInlineActions()                                            │
│  • Check for early commands (/help, /new, /reset)                   │
│  • Skill command lookup & invocation                                │
│  • Return early if handled                                          │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ (if not early handled)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  6. SKILL SELECTION (skill-commands.ts)                             │
│  • listSkillCommandsForWorkspace()                                  │
│    → buildWorkspaceSkillCommandSpecs()                              │
│    → loadWorkspaceSkillEntries()                                    │
│  • Match skill commands: /python, /bash, custom skills              │
│  • Available to Claude in system prompt                             │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  7. PREPARED REPLY (get-reply-run.ts)                               │
│  • runPreparedReply()                                               │
│  • Build system prompt with skills, tools, directives               │
│  • Create tool list from createOpenClawCodingTools()                │
│  • Invoke runEmbeddedPiAgent()                                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  8. TOOL CREATION (pi-tools.ts)                                     │
│  • createOpenClawCodingTools(options)                               │
│    - Base coding tools (read, write, edit, list)                    │
│    - OpenClaw tools (message, browser, gateway, etc.)               │
│    - Bash/exec tool (gated by policy)                               │
│    - Plugin tools                                                    │
│  • Apply tool policy filtering                                      │
│  • Filter by session/agent/model/provider policies                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  9. AGENT EXECUTION (pi-embedded-runner/run.ts)                     │
│  • runEmbeddedPiAgent(params)                                       │
│    - Load model credentials                                         │
│    - Build system prompt                                            │
│    - Load session history                                           │
│    - Register tools                                                 │
│    - Call Claude API with streamSimple()                            │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  10. TOOL INVOCATION LOOP (pi-agent-core library)                   │
│  ┌─ Loop While Claude Outputs Tool Calls ─┐                        │
│  │ 1. Claude responses: { type: "tool_use"                          │
│  │                        name: "read"                              │
│  │                        arguments: {...} }                        │
│  │ 2. Invoke tool directly (sync): tool.invoke(args)                │
│  │ 3. Tool executes:                                                │
│  │    - read: load file                                             │
│  │    - write: save file                                            │
│  │    - exec: run bash command                                      │
│  │    - message.send: route to channel provider                     │
│  │    - gateway: call gateway API                                   │
│  │    - skill: invoke skill script/runner                           │
│  │ 4. Return result to Claude                                       │
│  │ 5. Claude processes result, decides next action                  │
│  └────────────────────────────────────────┘                        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  11. RESPONSE DELIVERY (deliver-reply.ts)                           │
│  • Stream reply blocks to user as Claude generates                  │
│  • Handle tool update notifications                                 │
│  • Format for channel (markdown, plain text, etc.)                  │
│  • Route via provider dispatcher (Web, Telegram, Discord, etc.)     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  12. CHANNEL DELIVERY                                               │
│  • deliverWebReply() / route to channel-specific sender             │
│  • Format media (compress if needed)                                │
│  • Send to user                                                      │
│  • Record session state                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key File Reference Summary

| Function | File | Purpose |
|----------|------|---------|
| `monitorWebInbox()` | `src/web/inbound/monitor.ts` | Listen for incoming messages |
| `resolveAgentRoute()` | `src/routing/resolve-route.ts` | Route message to agent |
| `processMessage()` | `src/web/auto-reply/monitor/process-message.ts` | Build message context |
| `getReplyFromConfig()` | `src/auto-reply/reply/get-reply.ts` | Main reply pipeline |
| `handleInlineActions()` | `src/auto-reply/reply/get-reply-inline-actions.ts` | Early command handling |
| `listSkillCommandsForWorkspace()` | `src/auto-reply/skill-commands.ts` | Skill discovery |
| `buildWorkspaceSkillCommandSpecs()` | `src/agents/skills/workspace.ts` | Skill command specs |
| `runPreparedReply()` | `src/auto-reply/reply/get-reply-run.ts` | Execute reply with tools |
| `runEmbeddedPiAgent()` | `src/agents/pi-embedded-runner/run.ts` | Call Claude API |
| `createOpenClawCodingTools()` | `src/agents/pi-tools.ts` | Register all tools |
| `createOpenClawTools()` | `src/agents/openclaw-tools.ts` | Built-in OpenClaw tools |
| `handleToolsInvokeHttpRequest()` | `src/gateway/tools-invoke-http.ts` | Tool HTTP endpoint |

---

## Tool Use Examples

### Example 1: Claude Reads a File

```
User: "What's in myfile.txt?"

1. Message routed to agent "main"
2. Claude decides: "I'll read the file"
3. Tool Call: { name: "read", arguments: { path: "myfile.txt" } }
4. Tool Execution: createReadTool → fs.readFileSync()
5. Tool Result: file contents returned to Claude
6. Claude: Processes result and responds with summary
```

### Example 2: Claude Executes a Skill

```
User: "Calculate 2+2 using Python"

1. Claude: "I'll use the Python skill to calculate"
2. Tool Call: { name: "skill", arguments: { skillName: "python", code: "2+2" } }
3. Tool Execution: loadSkill(python) → run in sandbox/exec context
4. Tool Result: "4" returned to Claude
5. Claude: "The result is 4"
```

### Example 3: Claude Sends a Message

```
User: "Send 'Hello!' to the team channel"

1. Claude: "I'll send a message to the channel"
2. Tool Call: { name: "message.send", arguments: { text: "Hello!", channel: "team" } }
3. Tool Execution: createMessageTool → gateway.send() → Telegram/Discord API
4. Tool Result: { success: true, messageId: "..." }
5. Claude: "Message sent successfully"
```

---

## Summary

This comprehensive flow shows how OpenClaw integrates:
- **Message routing**: Flexible binding rules for multi-channel support
- **Agent orchestration**: Session management and context handling
- **Skills**: Extensible slash commands for specialized tasks
- **Tools**: Rich set of capabilities (file ops, browser, messaging, etc.)
- **Policy system**: Fine-grained control over tool availability

All orchestrated into a unified conversational system where Claude intelligently selects the right tools and skills to fulfill user requests.

---

*Documentation generated from analysis of OpenClaw codebase on 2026-02-01*
