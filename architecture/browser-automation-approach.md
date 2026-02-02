# OpenClaw Browser Automation Architecture

**Compared to Simular.ai**

## Overview

OpenClaw takes a different architectural approach from Simular.ai, leveraging standard web automation protocols rather than a custom JavaScript DSL for browser automation.

## Architecture

### Multi-Layer Browser Control Stack

OpenClaw uses:
- **Chrome DevTools Protocol (CDP)** as the low-level control layer
- **Playwright** on top of CDP for high-level interactions (clicking, typing, navigation)
- A **local control server** that exposes HTTP APIs for the agent

### Grounding/Element Selection

Instead of a custom DSL like Simular's, OpenClaw uses two grounding strategies:

#### 1. AI Snapshots (Playwright's `_snapshotForAI`)

- Uses Playwright's built-in experimental AI snapshot API
- Returns a text representation of the page with **numeric refs** (e.g., `12`, `23`)
- These are internally Playwright's `aria-ref` IDs that map to accessibility tree nodes
- Actions: `browser click 12`, `browser type 23 "text"`

**Code location:** `src/browser/pw-tools-core.snapshot.ts:40-82`

```typescript
// Example usage:
const result = await page._snapshotForAI({
  timeout: 5000,
  track: "response"
});
// Returns: text snapshot with aria-ref numeric IDs
```

#### 2. Role-Based Snapshots

- Extracts the accessibility tree via CDP's `Accessibility.getFullAXTree`
- Builds a structured tree with semantic refs like `e12`, `e23`
- Each ref is a tuple of `{role, name, nth}`

**Code location:** `src/browser/pw-role-snapshot.ts`

```typescript
type RoleRef = {
  role: string;      // "button", "link", "textbox", etc
  name?: string;     // accessible name
  nth?: number;      // index when duplicates exist
};

// Example refs:
// e12 -> {role: "button", name: "Submit", nth: 0}
// e23 -> {role: "textbox", name: "Email", nth: 0}
```

Resolves refs at action time via Playwright's `getByRole(role, {name}).nth(nth)`

## Key Differences from Simular.ai

| Aspect | OpenClaw | Simular.ai |
|--------|----------|-----------|
| **Control Protocol** | CDP + Playwright | Likely CDP + custom runtime |
| **Element Grounding** | Accessibility tree + role/aria refs | Custom JavaScript DSL (Simulang) |
| **Execution** | Standard Playwright locators (`getByRole`, `aria-ref`) | Custom DSL interpreter |
| **Complexity** | Leverages existing tools (Playwright) | Custom grounding system |
| **Stability** | Uses semantic roles/aria (more stable) | Unknown (DSL-dependent) |
| **Agent Learning** | Natural language → semantic actions | Needs to learn custom DSL |

## End-to-End Flow

For a task like **"book a flight from SF to NYC tomorrow"**:

### 1. Agent receives the `browser` tool

With actions like `snapshot`, `act`, `navigate`, `screenshot`

**Code location:** `src/agents/tools/browser-tool.ts:220-724`

### 2. Takes a snapshot

```typescript
browser(action: "snapshot", format: "ai")

// Returns text representation with refs:
// "From: [12]
//  To: [23]
//  Search: [34]"
```

### 3. LLM (Claude) plans actions

```typescript
browser(action: "act", request: {
  kind: "type",
  ref: "12",
  text: "San Francisco"
})

browser(action: "act", request: {
  kind: "type",
  ref: "23",
  text: "New York City"
})

browser(action: "act", request: {
  kind: "click",
  ref: "34"
})
```

### 4. OpenClaw resolves refs

- For AI snapshot refs (`12`): uses Playwright's `aria-ref` locator
- For role refs (`e12`): calls `page.getByRole("button", {name: "Search"}).nth(0)`

**Code location:** `src/browser/pw-session.ts` (role refs cache and resolution)

### 5. Playwright executes

- Locates the element via accessibility tree
- Performs the action (click, type, etc.)

**Code location:** `src/browser/pw-tools-core.interactions.ts`

```typescript
// Example: clicking via role ref
export async function clickViaPlaywright(opts: {
  cdpUrl: string;
  targetId?: string;
  ref: string;
  doubleClick?: boolean;
  button?: "left" | "right" | "middle";
  modifiers?: Array<"Alt" | "Control" | "ControlOrMeta" | "Meta" | "Shift">;
  timeoutMs?: number;
}): Promise<void> {
  const page = await getPageForTargetId(opts);
  const locator = refLocator(page, opts.ref);
  await locator.click({timeout, button, modifiers});
}
```

## Why This Approach Works

### Accessibility-First

- Uses the same semantic layer screen readers use
- More stable than CSS selectors or XPath
- Resilient to visual/DOM changes if semantics stay consistent

### No Custom DSL Required

- Playwright already provides robust element location
- Claude's reasoning maps naturally to "click button named X" vs learning a custom language
- Reduces cognitive load on the LLM

### Deterministic & Debuggable

CLI debugging tools:
- Inspect refs: `openclaw browser highlight e12`
- Trace actions: `openclaw browser trace start/stop`
- Screenshots with labeled refs: `openclaw browser snapshot --labels`

## Technical Implementation Details

### Ref Stability Across Calls

**Problem:** Playwright may return different Page objects for the same CDP target across requests.

**Solution:** Role refs cache by target ID

```typescript
// src/browser/pw-session.ts
const roleRefsByTarget = new Map<string, RoleRefsCacheEntry>();

export function rememberRoleRefsForTarget(opts: {
  cdpUrl: string;
  targetId: string;
  refs: RoleRefs;
  frameSelector?: string;
  mode?: "role" | "aria";
}): void {
  roleRefsByTarget.set(roleRefsKey(opts.cdpUrl, targetId), {
    refs: opts.refs,
    frameSelector: opts.frameSelector,
    mode: opts.mode
  });
}
```

This ensures refs remain valid across multiple tool calls within the same page session.

### Interaction Modes

1. **Interactive snapshot mode** (`--interactive`):
   - Filters to only interactive elements (buttons, links, inputs)
   - Flat list format optimized for action selection
   - Best for "drive the UI" tasks

2. **Compact mode** (`--compact`):
   - Removes unnamed structural elements
   - Prunes empty branches
   - Reduces token usage

3. **Efficient mode** (`--efficient`):
   - Preset combining interactive + compact + depth limits
   - Lowest token usage for action-oriented tasks

## Advantages Over Custom DSL

### 1. Leverage Existing Infrastructure

OpenClaw doesn't need to maintain:
- Custom element selector language
- Custom execution runtime
- Custom debugging tools

### 2. Natural Language Mapping

Claude understands:
- "Click the Submit button" → `click ref=<button role="button" name="Submit">`
- "Type email@example.com into the email field" → `type ref=<textbox role="textbox" name="Email"> text="email@example.com"`

No need to teach the model a new syntax.

### 3. Accessibility Benefits

Using aria roles means:
- Better support for accessible websites
- More semantic understanding of UI structure
- Natural compatibility with screen reader workflows

### 4. Playwright's Maturity

Inherits Playwright's:
- Auto-waiting for elements
- Retry logic
- Actionability checks (visible, stable, enabled, not covered)
- Cross-browser support

## Code References

| Component | File Path |
|-----------|-----------|
| Tool definition | `src/agents/tools/browser-tool.ts:220-724` |
| Playwright interactions | `src/browser/pw-tools-core.interactions.ts` |
| Role snapshot building | `src/browser/pw-role-snapshot.ts` |
| AI snapshot | `src/browser/pw-tools-core.snapshot.ts:40-82` |
| Ref resolution & caching | `src/browser/pw-session.ts` |
| Browser client actions | `src/browser/client-actions-core.ts` |
| Control server | `src/browser/server.ts` |

## Why No Custom Grounding Needed

**OpenClaw's insight:**

1. Playwright's accessibility tree already provides semantic structure
2. Claude is excellent at reasoning about natural semantic actions ("click button named Submit")
3. The ref system bridges Claude's output to Playwright's locators
4. No need to invent a new language when the problem is already solved

This is **simpler and more maintainable** than Simular's DSL approach while still being robust for complex GUI automation.

## Future Considerations

Potential enhancements that maintain the accessibility-first approach:

- Vision-based grounding fallback for non-accessible sites
- Hybrid ref system (semantic + visual coordinates)
- Learning from action success/failure to optimize ref selection
- Multi-modal snapshot (text + screenshot) for ambiguous UIs

All while keeping Playwright as the stable foundation.
