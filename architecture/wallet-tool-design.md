# Wallet Tool Design: Why Wallet Alone is Not Enough

> **Question**: If we add a wallet tool (for key management and signing), can it complete crypto tasks like "Buy $XYZ token for $100" or "Bet $50 on Polymarket"? What's missing compared to the browser tool?

This document analyzes the architectural requirements for adding crypto/wallet capabilities to OpenClaw, comparing it to the browser tool to understand what makes a tool "complete."

## Table of Contents

1. [The Browser Tool is Self-Sufficient](#the-browser-tool-is-self-sufficient)
2. [The Wallet Tool is Not Self-Sufficient](#the-wallet-tool-is-not-self-sufficient)
3. [What's Missing?](#whats-missing-three-architectural-options)
4. [Option 1: Use Browser (Primary Approach)](#option-1-use-browser-like-users-already-do)
5. [Option 2: Add DeFi Skills](#option-2-add-defi-skills)
6. [Option 3: Add Web3/Blockchain Tool](#option-3-add-web3blockchain-tool)
7. [Comparison Table](#comparison-whats-the-browser-equivalent-for-crypto)
8. [Recommended Hybrid Approach](#recommended-approach-hybrid-browserskills)
9. [Summary](#summary-the-missing-layer)

---

## The Browser Tool is Self-Sufficient

### **Browser = Complete Web Automation Platform**

The browser tool can complete entire workflows without needing additional tools or skills:

```
User Task: "Buy groceries from Tesco"

Browser tool execution:
├── navigate → Open https://www.tesco.com
├── snapshot → See page structure with clickable refs
├── act (click) → Click "Sign In"
├── act (type) → Enter email/password
├── act (click) → Submit login
├── snapshot → View product search
├── act (type) → Search "milk"
├── act (click) → Add to basket
├── act (click) → Checkout
└── Result: Task complete ✅

Everything needed is provided by the browser tool.
No additional skills required.
```

**Why browser is self-sufficient:**
- Provides **navigation** (open URLs, go back/forward)
- Provides **inspection** (snapshot to see page state)
- Provides **interaction** (click, type, fill forms)
- Provides **verification** (screenshot, console logs)

**This mirrors real life:**
- Users can complete almost any web task using just a browser
- No need for API knowledge or programming
- The browser UI provides everything needed

---

## The Wallet Tool is Not Self-Sufficient

### **Wallet = Only Signing Layer**

A wallet tool (key persistence + signing) **cannot** complete crypto tasks alone:

```
User Task: "Buy $100 of $XYZ token"

Wallet tool provides:
├── sign → Sign transaction
├── getBalance → Check wallet balance
└── sendTransaction → Broadcast signed transaction

But MISSING:
├── ❌ How to find $XYZ token contract address?
├── ❌ How to interact with Uniswap DEX contract?
├── ❌ How to encode swap() function call?
├── ❌ How to calculate slippage tolerance?
├── ❌ How to approve token spending first?
├── ❌ How to estimate gas costs?
└── Result: Task INCOMPLETE ❌
```

**Why wallet alone is insufficient:**
- Only handles the **final step** (signing)
- Doesn't know about **smart contracts** (DeFi protocols)
- Doesn't know how to **encode function calls** (swap, bet, lend)
- Doesn't know **protocol-specific logic** (Uniswap router, Polymarket CLOB)

**This is unlike real life:**
- Real users don't just use a wallet alone
- They use: **Wallet + DeFi Web UI** (Uniswap, Polymarket, etc.)
- The web UI handles contract interaction, wallet only signs

---

## What's Missing? Three Architectural Options

To make crypto tasks completable, you need to add the **smart contract interaction layer**.

---

### **Option 1: Use Browser (Like Users Already Do)**

**The "XXX" is Browser!**

Instead of building crypto-specific tools, use the existing browser tool to interact with DeFi web UIs:

```
User: "Buy $100 of $XYZ token"

System uses BROWSER tool:
1. browser action=open targetUrl=https://app.uniswap.org
2. browser action=snapshot (see UI with refs)
   → [1] Connect Wallet button
   → [2] Token selector
   → ...
3. browser action=act request={kind:"click", ref:"1"} (Connect Wallet)
4. browser action=snapshot (see token input)
5. browser action=act request={kind:"type", ref:"2", text:"XYZ"} (Select token)
6. browser action=act request={kind:"type", ref:"3", text:"100"} (Enter amount)
7. browser action=act request={kind:"click", ref:"4"} (Click Swap)
8. browser action=snapshot (see MetaMask confirmation popup)
9. browser action=act request={kind:"click", ref:"5"} (Confirm in MetaMask)

Result: Swap complete ✅
```

**How this works:**
- **Uniswap web UI** handles all the complexity:
  - Finding token addresses
  - Encoding swap function calls
  - Calculating slippage
  - Estimating gas
- **Browser tool** interacts with the UI (just like a human user)
- **MetaMask browser extension** handles wallet signing
- **No custom wallet tool needed** - browser + MetaMask = complete solution

**Why this mirrors real life:**
- This is exactly what users do:
  1. Open browser
  2. Go to Uniswap.org
  3. Connect wallet (MetaMask)
  4. Enter swap details
  5. Confirm transaction
- Browser is the primary interface, wallet is just the signing layer

**Pros:**
- ✅ Works immediately (no new tools to build)
- ✅ Handles all DeFi protocols (Uniswap, Aave, Polymarket, etc.)
- ✅ UIs are constantly updated by protocol teams
- ✅ Handles edge cases automatically (low liquidity, failed txs, etc.)
- ✅ Users can verify visually (screenshots)
- ✅ Mimics real user behavior (less likely to be blocked)

**Cons:**
- ⚠️ Slower than programmatic execution
- ⚠️ Requires browser to be running
- ⚠️ Can't run fully headless (needs MetaMask extension)
- ⚠️ UI changes can break automation

**Use cases:**
- Interactive DeFi tasks (swaps, lending, betting)
- One-off operations
- Tasks requiring visual verification
- Complex multi-protocol workflows

---

### **Option 2: Add DeFi Skills**

Add protocol-specific skills that encapsulate smart contract interaction logic:

**Skills Architecture:**
```
extensions/defi-skills/
├── uniswap-swap/
│   ├── skill.md
│   └── index.ts
├── polymarket-bet/
│   ├── skill.md
│   └── index.ts
├── aave-lend/
│   ├── skill.md
│   └── index.ts
└── token-transfer/
    ├── skill.md
    └── index.ts
```

**Each skill knows:**
- Contract addresses (mainnet, testnet, L2s)
- Contract ABIs (function signatures)
- How to encode function calls
- Protocol-specific logic (e.g., Uniswap routing, Polymarket order book)

**Example: Uniswap Swap Skill**

```markdown
---
name: uniswap-swap
description: "Swap tokens on Uniswap"
requires:
  tools: [wallet, web3]
---

Swaps tokens using Uniswap V2/V3 protocol.
```

**Skill Implementation:**
```typescript
// extensions/defi-skills/uniswap-swap/index.ts

export async function executeSwap(params: {
  fromToken: string;
  toToken: string;
  amount: string;
  slippage: number;
  wallet: WalletService;
}) {
  // 1. Resolve token addresses
  const fromAddress = await resolveToken(params.fromToken);
  const toAddress = await resolveToken(params.toToken);

  // 2. Find best route (V2 vs V3, multi-hop)
  const route = await findBestRoute(fromAddress, toAddress, params.amount);

  // 3. Encode swap function call
  const swapData = encodeSwapCall({
    router: UNISWAP_V3_ROUTER,
    path: route.path,
    amountIn: params.amount,
    amountOutMin: calculateMinOut(route.quote, params.slippage),
    deadline: Date.now() + 300000, // 5 min
  });

  // 4. Build transaction
  const tx = {
    to: UNISWAP_V3_ROUTER,
    data: swapData,
    value: isETH(fromAddress) ? params.amount : 0,
  };

  // 5. Use wallet tool to sign and send
  const signedTx = await params.wallet.sign(tx);
  const txHash = await params.wallet.sendTransaction(signedTx);

  return { txHash, route };
}
```

**User Task Flow:**
```
User: "Buy $100 of $XYZ token"

System:
1. Claude sees available skills in system prompt
2. Matches task to "uniswap-swap" skill
3. Invokes skill with params: {fromToken:"ETH", toToken:"XYZ", amount:"100"}
4. Skill executes:
   - Resolves XYZ token address (via Coingecko API or on-chain registry)
   - Calculates swap parameters (best route, slippage)
   - Encodes Uniswap Router.swapExactETHForTokens() function call
   - Calls wallet tool to sign transaction
   - Broadcasts transaction to network
5. Returns transaction hash
6. Result: Swap complete ✅
```

**Architecture Diagram:**
```
┌─────────────────────────────────────────┐
│ DeFi Skills Layer                       │
│ (Protocol-specific knowledge)           │
│                                         │
│ ├── uniswap-swap skill                 │
│ │   └── Knows: router address, ABI,    │
│ │       encoding, routing logic        │
│ ├── polymarket-bet skill               │
│ │   └── Knows: CLOB API, market IDs,   │
│ │       order encoding                 │
│ └── aave-lend skill                    │
│     └── Knows: lending pool, deposit,  │
│         withdraw encoding              │
└──────────────┬──────────────────────────┘
               │ Uses ↓
┌──────────────▼──────────────────────────┐
│ Wallet Tool                             │
│ (Signing & broadcast only)              │
│                                         │
│ ├── sign(rawTransaction)                │
│ ├── sendTransaction(signedTx)           │
│ └── getBalance(address)                 │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ Clean separation of concerns (skills = logic, wallet = signing)
- ✅ Can optimize for best routes, lowest gas, minimal slippage
- ✅ Works headless (no browser needed)
- ✅ Fast execution (direct contract calls)
- ✅ Can be scheduled (cron jobs)

**Cons:**
- ⚠️ Need to build and maintain a skill for every protocol
- ⚠️ Skills break when contracts upgrade
- ⚠️ Need to handle all edge cases manually
- ⚠️ Less flexible than browser (can't adapt to new UIs)
- ⚠️ Requires keeping ABIs and addresses up to date

**Use cases:**
- Automated trading/DCA strategies
- Scheduled operations (daily buy $10 of ETH)
- High-frequency interactions
- Headless server deployments
- Gas-optimized operations

---

### **Option 3: Add Web3/Blockchain Tool**

Add a general-purpose smart contract interaction tool that provides primitives for any contract:

**Web3 Tool Schema:**
```typescript
{
  name: "web3",
  description: "Interact with Ethereum and EVM-compatible blockchains",
  actions: [
    "call_contract",      // Read contract state (view functions)
    "encode_function",    // Encode function call data
    "decode_return",      // Decode function return values
    "simulate_tx",        // Simulate transaction before sending
    "get_token_info",     // Get token metadata (symbol, decimals, etc.)
    "find_dex_route",     // Find best DEX route for token swap
    "estimate_gas",       // Estimate gas for transaction
    "get_logs",           // Query event logs
  ]
}
```

**User Task Flow:**
```
User: "Buy $100 of $XYZ token"

System uses WEB3 + WALLET tools:
1. web3 action=get_token_info token=$XYZ
   → Returns: {address:"0x123...", symbol:"XYZ", decimals:18}

2. web3 action=find_dex_route from=ETH to=$XYZ amount=$100
   → Returns: {protocol:"Uniswap V3", route:[WETH,USDC,XYZ], quote:"1250 XYZ"}

3. web3 action=encode_function
   contract=UniswapRouter
   function=swapExactETHForTokens
   params=[amountOutMin, path, to, deadline]
   → Returns: {data:"0x7ff36ab5..."}

4. web3 action=simulate_tx
   to=UniswapRouter
   data=0x7ff36ab5...
   value=$100
   → Returns: {success:true, gasEstimate:150000, outputAmount:"1250 XYZ"}

5. wallet action=sign
   transaction={to, data, value, gasLimit}
   → Returns: {signedTx:"0xf86c..."}

6. web3 action=broadcast_tx signedTx=0xf86c...
   → Returns: {txHash:"0xabc..."}

Result: Swap complete ✅
```

**Architecture Diagram:**
```
┌─────────────────────────────────────────┐
│ Web3 Tool                               │
│ (General contract interaction)          │
│                                         │
│ ├── encode_function                     │
│ ├── simulate_tx                         │
│ ├── get_token_info                      │
│ ├── find_dex_route                      │
│ └── call_contract                       │
└──────────────┬──────────────────────────┘
               │ Uses ↓
┌──────────────▼──────────────────────────┐
│ Wallet Tool                             │
│ (Signing only)                          │
│                                         │
│ ├── sign(rawTransaction)                │
│ └── getBalance(address)                 │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ General-purpose (works for any EVM contract)
- ✅ Claude can reason about contract interactions
- ✅ More flexible than fixed protocol skills
- ✅ Can discover new protocols without new skills
- ✅ Composable primitives (encode, simulate, sign, broadcast)

**Cons:**
- ⚠️ Complex tool with many actions (harder for Claude to use correctly)
- ⚠️ Still needs knowledge of contract ABIs
- ⚠️ Claude must understand DeFi protocols to use effectively
- ⚠️ Error handling is harder (Claude must debug failed simulations)
- ⚠️ Requires external data sources (token registries, DEX APIs)

**Use cases:**
- Exploring new DeFi protocols
- Custom contract interactions
- Advanced users who understand smart contracts
- Multi-chain operations (same tool works on all EVMs)

---

## Comparison: What's the Browser Equivalent for Crypto?

| Approach | Web World | Crypto World | Completeness | Best For |
|----------|-----------|--------------|--------------|----------|
| **UI-based** | Browser tool | Browser + MetaMask | ✅ Complete | Interactive tasks, one-offs |
| **Minimal** | Browser tool | Wallet tool alone | ❌ Incomplete | Nothing (can't interact with contracts) |
| **Skill-based** | Browser tool | Wallet + DeFi Skills | ✅ Complete | Automation, scheduled ops |
| **SDK-based** | Browser tool | Wallet + Web3 Tool | ✅ Complete | Flexibility, multi-protocol |

**Key Insight:**

- **Browser** = complete because it provides UI interaction layer
- **Wallet alone** = incomplete because it only provides signing
- **To complete crypto tasks**, you need: **Wallet + (Browser OR Skills OR Web3 Tool)**

The "XXX" you're looking for is:
1. **Browser** (for UI-based DeFi like Uniswap web)
2. **DeFi Skills** (for programmatic automation)
3. **Web3 Tool** (for flexible contract interaction)

---

## Recommended Approach: Hybrid (Browser+Skills)

**Use Browser as Primary (90%), Add Skills for Automation (10%)**

### **Tier 1: Browser for Interactive Tasks**

For most DeFi operations, use the browser tool:

```
User: "Buy $100 of $XYZ on Uniswap"
→ browser open app.uniswap.org
→ browser snapshot (see UI)
→ browser act (connect wallet)
→ browser act (select token, enter amount)
→ browser act (confirm swap)
→ MetaMask extension signs transaction
```

**Why browser first:**
- Web UIs are constantly updated by protocol teams
- Handles edge cases automatically (low liquidity, slippage alerts, etc.)
- Users can verify visually via screenshots
- Works like real user behavior (less likely to trigger anti-bot measures)
- No maintenance needed (UIs update themselves)

### **Tier 2: Skills for Headless Automation**

For programmatic/scheduled tasks, use DeFi skills:

```
User: "Every day at 9am, buy $10 of ETH"
→ Schedule cron job
→ Run uniswap-swap skill (headless)
→ Wallet signs and broadcasts
→ Log transaction hash
```

**Why skills for automation:**
- Automation requires headless execution (no browser UI)
- Skills can optimize for gas costs and slippage
- Faster execution than browser automation
- Can run on servers without display

### **Architecture Diagram:**

```
┌──────────────────────────────────────────────────────┐
│ User Task: "Buy $100 of $XYZ token"                 │
└────────────────────┬─────────────────────────────────┘
                     │
          ┌──────────▼───────────┐
          │ Claude Reasoning:    │
          │ "Is this interactive │
          │  or automated?"      │
          └──────────┬───────────┘
                     │
         ┌───────────┴────────────┐
         │                        │
    [Interactive]           [Automated/Scheduled]
         │                        │
         ▼                        ▼
┌─────────────────┐      ┌────────────────────┐
│ Browser Tool    │      │ DeFi Skill         │
│ + MetaMask Ext  │      │ + Wallet Tool      │
│                 │      │                    │
│ Opens Uniswap   │      │ Direct contract    │
│ web UI, interacts│     │ interaction        │
└─────────────────┘      └────────────────────┘
```

**Decision Criteria:**

| Criteria | Use Browser | Use Skills |
|----------|-------------|------------|
| One-off task | ✅ | |
| Visual verification needed | ✅ | |
| New/unknown protocol | ✅ | |
| Scheduled/recurring | | ✅ |
| Headless server | | ✅ |
| Gas optimization critical | | ✅ |
| Speed critical | | ✅ |

---

## Summary: The Missing Layer

**The key insight:** Wallet alone is like having a "sign" tool without a browser.

### **What's Missing from Wallet-Only Approach:**

| Layer | Browser World | Crypto World | Solution |
|-------|---------------|--------------|----------|
| **UI Layer** | Browser (navigate, view) | Browser (Uniswap UI, Polymarket UI) | Use browser tool |
| **Interaction Layer** | Browser (click, type) | ❌ **MISSING** (encode contract calls) | Add Skills or Web3 Tool |
| **Signing Layer** | N/A (no signing) | Wallet (sign transactions) | Wallet tool |

**The missing piece is the "smart contract interaction layer":**

- **Option A**: Use **browser** for everything (easiest, most robust)
- **Option B**: Add **DeFi skills** (protocol-specific, optimized)
- **Option C**: Add **web3 tool** (general-purpose, flexible)

### **Recommended Implementation:**

1. **Phase 1**: Start with **browser + MetaMask extension**
   - Covers 90% of DeFi use cases immediately
   - No custom tools needed
   - Works with all protocols

2. **Phase 2**: Add **popular DeFi skills**
   - Uniswap swap
   - Token transfers
   - Polymarket bets
   - For automation and optimization

3. **Phase 3** (optional): Add **web3 tool**
   - For advanced users
   - Multi-chain operations
   - Custom contract interactions

### **Why This Mirrors Real Life:**

Real users don't use wallets alone either:
- **Browser + Wallet** (MetaMask) is how 99% of users interact with DeFi
- Web UIs handle the complexity
- Wallet just signs the final transaction

Your OpenClaw should follow the same pattern:
- **Browser tool** = the DeFi interface
- **Wallet tool** = the signing layer
- **Skills** (optional) = automation for power users

---

**Final Answer:**

The "XXX" you're looking for is:
- **Browser** (primary, for UI-based DeFi)
- **+ Skills** (optional, for automation)

Just like real users don't interact with smart contracts directly - they use **Uniswap's web UI + MetaMask** - your agent should do the same.

---

*Documentation generated from wallet architecture discussion on 2026-02-02*
