# How OpenClaw Decides Which Tools to Use

> **Question**: When a user says "Order my weekly groceries from Tesco", how does OpenClaw technically decide to use the browser tool? What's the decision-making process?

This document provides a technical analysis of how Claude selects tools to fulfill user requests, using the Tesco grocery shopping example.

## Table of Contents

1. [Overview](#overview)
2. [Phase 1: System Prompt Construction](#phase-1-system-prompt-construction)
3. [Phase 2: User Message Processing](#phase-2-user-message-processing)
4. [Phase 3: Claude's Reasoning](#phase-3-claudes-reasoning-inside-claudes-model)
5. [Phase 4: Tool Invocation](#phase-4-tool-invocation)
6. [Phase 5: Browser Tool Execution](#phase-5-browser-tool-execution)
7. [Phase 6: Tool Result Back to Claude](#phase-6-tool-result-back-to-claude)
8. [Phase 7: Snapshot → Act Loop](#phase-7-snapshot--act-loop)
9. [Why Browser vs API?](#why-browser-vs-api-the-technical-reason)
10. [Decision Logic Summary](#decision-logic-summary)
11. [Key Insights](#key-insight-for-wallet-or-other-tools)

---

## Overview

**Key Finding**: There is **NO explicit decision code** in OpenClaw that chooses which tool to use. The decision happens entirely within Claude's neural network based on:

1. System prompt tool descriptions
2. Available tools in the registry
3. User's task requirements
4. Claude's reasoning about which tool fits best

The decision is **emergent behavior**, not programmed logic.

---

## Phase 1: System Prompt Construction

When the agent starts, OpenClaw builds a system prompt that includes available tools.

**File**: `src/agents/system-prompt.ts` (lines 217-270)

**System Prompt Section**:
```
## Tooling

You have access to these tools:
- read: Read files from the workspace
- write: Write files to the workspace
- exec: Execute shell commands
- browser: Control the browser via OpenClaw's browser control server
  (status/start/stop/profiles/tabs/open/snapshot/screenshot/actions).
  Profiles: use profile="chrome" for Chrome extension relay...
  Use snapshot+act for UI automation.
- web_search: Search the web
- web_fetch: Fetch web page content
- message: Send messages to channels
...
```

**Key Insight**: The browser tool description **explicitly tells Claude**:
- What it does: "Control the browser"
- What actions it supports: `open`, `snapshot`, `screenshot`, `act`
- How to use it: "Use snapshot+act for UI automation"

This is **guidance text**, not executable logic.

---

## Phase 2: User Message Processing

**User Message**: "Order my weekly groceries from Tesco"

**File**: `src/auto-reply/reply/get-reply.ts`

The message goes through the processing pipeline:
1. **Routing** → determines agent `main`
2. **Session loading** → loads conversation history
3. **System prompt + tools** → sent to Claude API
4. **User message** appended to conversation

**What Claude Receives** (API Request):
```json
{
  "system": [
    {
      "type": "text",
      "text": "## Tooling\n\n- browser: Control the browser..."
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "Order my weekly groceries from Tesco"
    }
  ],
  "tools": [
    {
      "name": "browser",
      "description": "Control the browser via OpenClaw's browser control server...",
      "input_schema": {
        "type": "object",
        "properties": {
          "action": {
            "enum": ["open", "snapshot", "act", "screenshot", ...]
          },
          "targetUrl": { "type": "string" },
          "profile": { "type": "string" },
          ...
        },
        "required": ["action"]
      }
    },
    {
      "name": "read",
      "description": "Read files from the workspace",
      ...
    },
    // ... other tools
  ]
}
```

---

## Phase 3: Claude's Reasoning (Inside Claude's Model)

Claude (the LLM) receives the request and reasons internally:

**Claude's Internal Thought Process**:
> "User wants to order groceries from Tesco. Let me analyze this:
>
> **Task Requirements:**
> - Open a website (Tesco.com)
> - Navigate their UI
> - Search for products
> - Add items to cart
> - Complete checkout
>
> **Available Tools:**
> - `read/write` - for files, not websites ❌
> - `exec` - for shell commands, not web UIs ❌
> - `browser` - description says 'Control the browser' and supports `open/snapshot/act` ✅
> - `web_search` - for searching Google, not purchasing ❌
> - `web_fetch` - for reading HTML content, not interacting ❌
> - `message` - for sending messages, not web automation ❌
>
> **Browser Tool Capabilities:**
> - `action=open` with `targetUrl` → I can open Tesco.com
> - `action=snapshot` → I can see the page state
> - `action=act` → I can click buttons and type in forms
>
> **Conclusion:**
> No API tool exists for Tesco (I don't see one in the tool list).
> Browser is the only tool that supports web UI automation.
>
> **Decision: Use browser tool.**"

**Important**: This reasoning happens **entirely in Claude's neural network**, not in OpenClaw code. OpenClaw just provides the tools and their descriptions.

---

## Phase 4: Tool Invocation

Claude generates a response with a tool call:

**Claude's API Response**:
```json
{
  "content": [
    {
      "type": "text",
      "text": "I'll help you order groceries from Tesco using the browser."
    },
    {
      "type": "tool_use",
      "id": "toolu_01ABC123",
      "name": "browser",
      "input": {
        "action": "open",
        "targetUrl": "https://www.tesco.com"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

**File**: `src/agents/pi-embedded-runner/run/attempt.ts` (lines 800-805)

OpenClaw receives this response and:
1. Parses the `tool_use` block
2. Extracts: `name="browser"`, `input={action:"open", targetUrl:"..."}`
3. Finds the `browser` tool in the registered tool list
4. Calls: `browserTool.execute("toolu_01ABC123", { action: "open", targetUrl: "https://www.tesco.com" })`

---

## Phase 5: Browser Tool Execution

**File**: `src/agents/tools/browser-tool.ts` (lines 270-320)

The browser tool's `execute()` function runs:

```typescript
execute: async (toolCallId, args) => {
  const action = readStringParam(args, "action", { required: true });

  if (action === "open") {
    const targetUrl = readStringParam(args, "targetUrl", { required: true });
    const profile = readStringParam(args, "profile") ?? "openclaw";

    // Call browser control service
    const result = await browserControlService.openTab({
      url: targetUrl,
      profile: profile
    });

    return {
      content: [
        { type: "text", text: `Opened ${targetUrl} in ${profile} profile` }
      ]
    };
  }

  // ... handle other actions (snapshot, act, screenshot, etc.)
}
```

This delegates to the **browser control service**:

**File**: `src/browser/server.ts`

The service:
1. Checks if Chrome is running (starts if needed)
2. Creates a new tab in the specified profile
3. Navigates to `https://www.tesco.com`
4. Waits for page load
5. Returns success confirmation

---

## Phase 6: Tool Result Back to Claude

The tool execution result is sent back to Claude as a new API request:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01ABC123",
      "content": "Opened https://www.tesco.com in openclaw profile"
    }
  ]
}
```

Claude receives this and continues reasoning:

> "Good! The page is now open. To interact with it, I need to see what's on the page. I'll take a snapshot to get the page structure with clickable references."

**Claude's Next Tool Call**:
```json
{
  "type": "tool_use",
  "id": "toolu_02DEF456",
  "name": "browser",
  "input": {
    "action": "snapshot",
    "format": "ai"
  }
}
```

---

## Phase 7: Snapshot → Act Loop

The browser tool returns a snapshot with **clickable references**:

**Snapshot Result**:
```
Page: Tesco.com - Groceries, Clubcard & Offers

Interactive elements:
[1] Search for products (textbox)
[2] Sign in (link)
[3] View basket (button)
[4] Clubcard (link)
[5] Offers (link)
[6] Groceries (link)
...
```

Claude analyzes the snapshot:

> "I can see a search box at reference [1]. I'll type 'milk' there to search for groceries."

**Tool Call**:
```json
{
  "type": "tool_use",
  "name": "browser",
  "input": {
    "action": "act",
    "request": {
      "kind": "type",
      "ref": "1",
      "text": "milk",
      "submit": true
    }
  }
}
```

**The Loop Continues**:
```
1. snapshot → get current page state
2. analyze → find relevant elements
3. act → click/type/interact
4. snapshot → verify action succeeded
5. act → next interaction
6. repeat until task complete
```

**Example Multi-Step Flow**:
```
User: "Order my weekly groceries from Tesco"

Step 1: browser open https://www.tesco.com
Step 2: browser snapshot (see page)
Step 3: browser act (click Sign In)
Step 4: browser snapshot (see login form)
Step 5: browser act (type email)
Step 6: browser act (type password)
Step 7: browser act (click Submit)
Step 8: browser snapshot (see homepage)
Step 9: browser act (type "milk" in search)
Step 10: browser snapshot (see results)
Step 11: browser act (click "Add to basket" for first item)
Step 12: browser snapshot (verify added)
... continue until checkout complete
```

---

## Why Browser vs API? The Technical Reason

**OpenClaw has NO Tesco API tool.**

**File**: `src/agents/openclaw-tools.ts` (lines 22-50)

```typescript
export function createOpenClawTools(options?: CreateToolsOptions): AnyAgentTool[] {
  const tools: AnyAgentTool[] = [
    createBrowserTool(options),
    createCanvasTool(options),
    createNodesTool(options),
    createCronTool(options),
    createMessageTool(options),
    createTtsTool(options),
    createGatewayTool(options),
    createSessionsSpawnTool(options),
    createWebFetchTool(options),
    createWebSearchTool(options),
    createImageTool(options),
  ];

  return filterToolsByPolicy(tools, options?.policy);
}
```

**Analysis**:
- No `createTescoApiTool` exists
- No `createEcommerceApiTool` exists
- No generic shopping API integration

**Claude's Decision is By Elimination**:
1. Task requires web interaction (open site, click buttons, fill forms)
2. Only `browser` tool supports interactive web automation
3. `web_fetch` can read HTML but not interact
4. `web_search` can search Google but not purchase items
5. Therefore: **use `browser`**

---

## Decision Logic Summary

```
┌──────────────────────────────────────────────────────────────┐
│ NO EXPLICIT DECISION CODE IN OPENCLAW                       │
│                                                              │
│ The decision happens in Claude's neural network based on:   │
│                                                              │
│  1. System prompt tool descriptions                         │
│     → Browser: "Control the browser...snapshot+act"         │
│                                                              │
│  2. Available tools in the registry                         │
│     → [browser, read, write, exec, web_search, ...]         │
│                                                              │
│  3. User's task requirements                                │
│     → "Order groceries" = web automation needed             │
│                                                              │
│  4. Claude's reasoning about which tool fits best           │
│     → Browser matches web automation requirements           │
│                                                              │
│  5. No alternative API tools exist                          │
│     → No Tesco API tool in registry                         │
│                                                              │
│  Result: Claude chooses browser tool                        │
└──────────────────────────────────────────────────────────────┘
```

**The "decision" is emergent behavior from:**
- Tool descriptions that guide Claude's understanding
- No alternative API tools being available
- Claude's training on web automation patterns
- The tool schema explicitly listing `open/snapshot/act` actions

**No code in OpenClaw says**: "If task contains 'order' and 'groceries', use browser."

Instead, Claude **reasons** based on available capabilities.

---

## Key Insight for Wallet or Other Tools

The browser is chosen because:
1. **Tool description tells Claude what it does** ("Control the browser")
2. **No API alternative exists** (no Tesco API tool)
3. **Claude reasons that browser = web automation**

### Applying This to New Tools (e.g., Wallet)

**If you add a wallet tool:**

```typescript
// In createOpenClawTools()
const tools = [
  createBrowserTool(options),
  createWalletTool(options),  // ← New tool
  // ...
];
```

**With description:**
```typescript
{
  name: "wallet",
  description: "Send cryptocurrency transactions. Supports signing, balance checks, and multi-chain operations (Ethereum, Polygon, etc.). Use for any crypto/blockchain tasks."
}
```

**Then Claude will reason:**
```
User: "Send 0.1 ETH to alice.eth"

Claude thinks:
- Task requires sending cryptocurrency
- I see a 'wallet' tool with "Send cryptocurrency transactions"
- This matches the requirement
- Decision: Use wallet tool

Tool call: { name: "wallet", input: { action: "send", to: "alice.eth", amount: "0.1", token: "ETH" } }
```

**If NO wallet tool exists:**
```
User: "Send 0.1 ETH to alice.eth"

Claude thinks:
- Task requires sending cryptocurrency
- I see: browser, read, write, exec, message...
- None of these are designed for crypto transactions
- Browser could theoretically open MetaMask, but that's not what it's for
- Decision: Tell user I can't do this

Response: "I don't have the capability to send cryptocurrency transactions. You would need to use a wallet app or command-line tool directly."
```

---

## Making a Tool Core vs Plugin

The technical mechanism for tool selection is **identical** whether a tool is core or plugin.

**Core Tool** (like browser):
- Always registered in `createOpenClawTools()`
- Always available to Claude (unless filtered by policy)
- Claude sees it in every session

**Plugin Tool**:
- Registered via `api.registerTool()` in plugin
- Only available if plugin is enabled
- Claude only sees it if plugin is loaded

**Decision Process is the Same**:
1. Claude sees available tools in system prompt
2. Claude reads tool descriptions
3. Claude matches tool capabilities to task requirements
4. Claude chooses the best-fit tool

**The difference**: Core tools are **always in the list**, plugins are **conditionally in the list**.

---

## Summary

| Phase | What Happens | Where |
|-------|-------------|-------|
| **1. System Prompt** | Tools listed with descriptions | `system-prompt.ts` |
| **2. Message Processing** | User message + tools sent to Claude API | `get-reply.ts` |
| **3. Claude Reasoning** | Claude picks tool based on descriptions | Claude's neural network |
| **4. Tool Invocation** | Claude returns tool_use block | API response |
| **5. Tool Execution** | OpenClaw calls tool's execute() | `browser-tool.ts` |
| **6. Result Return** | Tool result sent back to Claude | API request |
| **7. Continue Loop** | Claude continues with next action | Repeat 3-6 |

**Key Takeaway**: OpenClaw doesn't "decide" which tool to use. It **presents tools to Claude**, and Claude's neural network makes the decision based on reasoning about the task requirements and available capabilities.

This is why good **tool descriptions** are critical - they're Claude's only guide for understanding what each tool does.

---

*Documentation generated from analysis of OpenClaw codebase on 2026-02-02*
