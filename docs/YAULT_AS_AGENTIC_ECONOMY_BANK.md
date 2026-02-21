# Yault: The Self-Running Bank of the Agentic Economy

> How Yault's existing self-custodial infrastructure — accounts, fund pools, authority judgment, allowances, and settlement — becomes the banking layer for autonomous AI agents.

---

## 1. The Core Insight

The agentic economy needs a **bank** — not a traditional bank with human tellers and manual approvals, but a programmable, self-running financial institution where:

- Institutions and individuals hold **accounts** with verified identities
- **Fund pools** generate yield while waiting to be deployed
- **Rules** (not humans) govern day-to-day fund movements
- **Arbitration** enforces commitments when agents disagree
- **Subscriptions and recurring payments** execute automatically
- Humans retain **ultimate control** through policy, cooldowns, and emergency shutdown

**Yault already is this bank.** Every capability listed above exists in production-grade code today. The missing step is not building new infrastructure — it is connecting Yault's existing systems to the agent protocol stack (MCP, A2A, DSE).

---

## 2. Yault's Existing Infrastructure → Agentic Economy Mapping

### 2.1 Accounts = Agent Economic Identity

**Yault has:**
- User/institutional accounts with Ed25519/EVM cryptographic identity
- Sub-account hierarchy (parent → member with roles: `spouse`, `child`, `employee`, `sub_account`)
- Per-member permissions: `can_view_balance`, `can_withdraw`, `can_deposit`, `withdrawal_limit`, `withdrawal_period`
- Deterministic identity: `authority_id = SHA256(pubkey)`

**Agentic economy needs:**
- Every agent needs a verifiable economic identity
- Agents need scoped permissions (budget limits, allowed operations)
- Hierarchical agent organization (parent agent delegates to child agents)

**Mapping:**
```
Yault Sub-Account          →  Agent Account
Parent Wallet              →  Human / Organization (DSE Owner)
Member Permissions         →  Agent Policy (maxAmountPerTx, allowList, etc.)
withdrawal_limit + period  →  Agent Budget (daily/weekly/monthly spend cap)
member_id (SHA256)         →  Agent Identity (verifiable, deterministic)
```

A human creates a Yault account. They add sub-accounts for each of their agents. Each sub-account has specific permissions — Agent A can withdraw up to $200/day, Agent B can only deposit. The Yault permission model **is** the DSE policy engine, already implemented.

### 2.2 Fund Pools = Agent Treasuries with Yield

**Yault has:**
- ERC-4626 yield vaults (deposits earn yield via Aave V3 strategy)
- Per-user principal tracking (`userPrincipal[user]`)
- Vault share accounting (shares represent proportional claim on total assets)
- 80/15/5 yield split (user / platform / authority)
- Reserve ratio (20% idle for immediate withdrawals)
- Internal share transfers (parent → member, no on-chain gas)

**Agentic economy needs:**
- Agents need working capital (pre-funded budgets)
- Idle capital should not sit dormant — it should generate yield
- Agent treasuries need to be auditable and transparent

**Mapping:**
```
ERC-4626 Vault Position    →  Agent Treasury
Vault Shares               →  Agent's Budget Balance (yield-bearing)
Internal Share Transfer     →  Agent Budget Top-Up (gas-free)
Reserve Ratio (20%)        →  Instant Liquidity for Agent Payments
Yield (80% to user)        →  Agent's idle budget earns for the human
```

When a human allocates $1,000 to their Shopping Agent's sub-account, that $1,000 enters the vault as shares. While the agent negotiates with vendors, the $1,000 earns yield. When the agent executes a $50 purchase, 50 shares are redeemed. The remaining $950 continues earning. **The agent's budget is a yield-bearing instrument.**

### 2.3 Allowances = Agent Funding & Subscription Rails

**Yault has:**
- **One-time allowances**: Instant fund transfer from parent to member (vault share movement)
- **Recurring allowances**: Scheduled transfers with configurable frequency (`daily`, `weekly`, `monthly`)
  - `next_execution` timestamp
  - `end_date` (nullable = indefinite)
  - Automatic execution by scheduler
  - Cancellable by parent at any time
- Withdrawal limit enforcement: most restrictive limit across all parent relationships

**Agentic economy needs:**
- Agent subscriptions: Agent B provides ongoing service to Agent A for $X/month
- Recurring SaaS payments: Agent uses Tool X monthly
- Budget replenishment: Human tops up agent budget on schedule
- Escrow patterns: Fund locked until service delivered

**Mapping:**
```
One-Time Allowance         →  Agent One-Time Payment / Escrow Release
Recurring Allowance        →  Agent Subscription / SaaS Payment
Frequency Config           →  Payment Schedule (daily/weekly/monthly)
Cancel by Parent           →  Human Cancels Agent's Subscription
Withdrawal Limit           →  Agent's Spending Cap (enforced at Yault level)
```

**Subscription scenario:**
1. Agent A negotiates with Vendor Agent B: "Data feed, $50/month"
2. Both sign EIP-712 commitment (DSE layer)
3. Human approves (ReviewRequest via mobile)
4. Yault creates recurring allowance: $50/month from Agent A's sub-account to Vendor B's account
5. Every month, Yault's scheduler auto-executes the transfer
6. If Human decides to cancel: one API call cancels the recurring allowance
7. Agent A's remaining budget continues earning yield

This is **not a payment rail that needs to be built**. It exists today in Yault's allowance system.

### 2.4 Authority Judgment = Agent Dispute Resolution

**Yault has:**
- Authority registration with verified identity (bar number, jurisdiction, specialization)
- Trigger event lifecycle: `pending` → `cooldown` → `released`/`held`/`rejected`
- Evidence-based decisions: `evidence_hash` (SHA256) + Ed25519 signature
- Cooldown period (default 24h, max 168h): window for cancellation
- Cancel during cooldown: reverts to pending, 1-hour grace period prevents abuse
- Immutable audit trail: every decision uploaded to Arweave with retry
- Authority binding: authority must have active binding to wallet/recipient

**Agentic economy needs:**
- When Agent A pays Agent B for a service, and Agent B doesn't deliver, who resolves it?
- When two agents sign a commitment and one breaches, who enforces consequences?
- Dispute resolution must be: evidence-based, time-bounded, auditable, and reversible within reason

**Mapping:**
```
Authority                  →  Arbitrator in Agentic Economy
Trigger Initiation         →  Dispute Filing (with evidence hash)
Decision (release/hold)    →  Arbitration Ruling
Cooldown Period            →  Appeal Window (human can cancel)
Evidence Hash + Signature  →  Cryptographic Proof of Judgment
Arweave Audit              →  Immutable Dispute Record
Authority Binding          →  Pre-agreed Arbitrator Selection
```

**Dispute scenario:**
1. Agent A pays Agent B $200 for flight booking (via Yault allowance)
2. Agent B signs EIP-712 commitment: "Book flight X, refund if not confirmed by 6pm"
3. 6pm passes, no booking confirmed
4. Agent A files dispute: creates trigger event with evidence (commitment hash, no delivery proof)
5. Bound Authority reviews evidence, submits `release` decision (refund Agent A)
6. 24-hour cooldown: Agent B can submit counter-evidence
7. Cooldown expires: Yault executes refund from Agent B's escrowed funds
8. Full lifecycle recorded on Arweave: evidence hashes, signatures, timestamps

The entire dispute resolution mechanism — **evidence submission, authority judgment, cooldown, cancellation, execution, audit** — already exists in Yault. For the agentic economy, we just need to map DSE commitment breaches to Yault trigger events.

### 2.5 Settlement Engine = Agent Payment Execution

**Yault has:**
- On-chain settlement via ERC-4626 vault (redeem shares → underlying asset → transfer)
- Off-chain settlement tracking (revenue records, audit log)
- Integer-safe arithmetic (1e8 precision, no floating-point)
- Revenue distribution: automatic yield split (80/15/5)
- Authority revenue escrow: `pendingAuthorityRevenue[authority]` with explicit claim
- Multi-chain support: paths can have EVM, BTC, SOL addresses

**Agentic economy needs:**
- Agent-to-agent payments must settle reliably
- Settlement must be auditable
- Multi-chain: agents operate across Ethereum, Solana, Bitcoin

**Mapping:**
```
Vault Redeem               →  Agent Payment Execution
Revenue Distribution       →  Platform Fee Collection (sustainable model)
Authority Escrow           →  Arbitrator Fee Escrow (pay-on-resolution)
Multi-Chain Paths          →  Cross-Chain Agent Payments
Integer-Safe Arithmetic    →  No Rounding Errors in Agent Transactions
```

### 2.6 Three-Factor Custody = Human Sovereignty Guarantee

**Yault has:**
- Three-factor release: Sealed Artifact (encrypted vault) + UserCredential (recipient key) + AdminFactor (authority-held)
- No single party can unilaterally release funds
- Authority submits AdminFactor only after legal/contractual trigger
- On-chain attestation verification before release

**Agentic economy needs:**
- Agents must not be able to drain all funds autonomously
- High-value operations need multi-party authorization
- Emergency shutdown must be possible

**Mapping:**
```
Three-Factor Release       →  High-Value Agent Authorization
AdminFactor (Authority)    →  Arbitrator/Governance Key
UserCredential (Recipient) →  Agent's Operational Key
Sealed Artifact            →  Encrypted Budget Allocation
On-Chain Attestation       →  Verifiable Authorization Proof
```

For the agentic economy: below the withdrawal limit, agents operate freely (single-factor, fast). Above the limit, the three-factor mechanism kicks in — requiring human approval (via mobile ReviewRequest), authority attestation, or both. **Graduated trust, not binary trust.**

---

## 3. What Makes Yault a "Bank" (Not Just a Payment Rail)

AP2 (Google/Coinbase) gives agents a payment rail. x402 gives agents HTTP-native micropayments. These are **payment protocols** — they move money from A to B.

A **bank** does far more:

| Capability | Payment Rail (AP2/x402) | Bank (Yault) |
|---|---|---|
| Send payment | Yes | Yes |
| Hold accounts with identity | No | Yes (sub-accounts + KYC) |
| Yield on deposits | No | Yes (ERC-4626 + Aave strategy) |
| Spending limits | No | Yes (per-member withdrawal limits) |
| Recurring payments | No | Yes (recurring allowances with scheduler) |
| Dispute resolution | No | Yes (authority judgment system) |
| Escrow | No | Yes (vault share escrow + authority release) |
| Audit trail | Partial | Yes (Arweave immutable records) |
| Multi-party authorization | No | Yes (three-factor custody) |
| Emergency freeze | No | Yes (pause deposits, cancel allowances) |
| Revenue sharing | No | Yes (80/15/5 yield split) |

**AP2 is Venmo. Yault is a bank.** The agentic economy doesn't just need Venmo. It needs a bank where agents have accounts, budgets, credit limits, subscription management, dispute resolution, and auditable transaction histories.

---

## 4. Agentic Economy Use Cases Enabled by Yault

### 4.1 Agent Subscription Management

```
Human sets up on Yault:
├── Shopping Agent sub-account ($500/month budget)
│   ├── Recurring allowance: $30/month → Price Comparison Service
│   ├── Recurring allowance: $15/month → Product Review API
│   └── One-time budget: $455 for actual purchases
├── Research Agent sub-account ($200/month budget)
│   └── Recurring allowance: $100/month → Data Provider Agent
└── Finance Agent sub-account ($1,000/month budget)
    └── Recurring allowance: $50/month → Market Data Feed

All idle funds earn yield in ERC-4626 vault.
Human can cancel any subscription instantly.
All spending tracked in audit log.
```

### 4.2 Agent Commerce with Escrow

```
1. Shopping Agent finds Widget for $80 from Vendor Agent
2. DSE: Both sign EIP-712 commitment (deliver by Tuesday, refund if not)
3. Yault: $80 moved from Shopping Agent sub-account to escrow
4. Vendor Agent delivers Widget
5. Shopping Agent verifies delivery → releases escrow
6. If no delivery by Tuesday:
   a. Shopping Agent files trigger event (evidence: commitment hash)
   b. Authority reviews → release decision (refund)
   c. 24h cooldown → Vendor can appeal
   d. Cooldown expires → $80 returned to Shopping Agent
```

### 4.3 Institutional Agent Fleet

```
Corporation sets up on Yault:
├── Procurement Department (parent account)
│   ├── Procurement Agent Alpha ($10,000/month limit)
│   ├── Procurement Agent Beta ($10,000/month limit)
│   └── Procurement Agent Gamma ($5,000/month limit)
├── HR Department (parent account)
│   ├── Recruiting Agent ($3,000/month for job board APIs)
│   └── Benefits Agent ($1,000/month for benefits vendor APIs)
└── Finance Department (parent account)
    ├── Accounts Payable Agent ($50,000/month)
    └── Treasury Agent ($100,000/month, three-factor for >$10K)

Each department's idle funds earn yield.
Cross-department transfers require parent-level approval.
All transactions auditable on Arweave.
```

### 4.4 Agent-to-Agent SLA with Automatic Penalties

```
1. Agent A contracts Agent B: "99.9% uptime, $1,000/month"
2. DSE: EIP-712 commitment signed by both
3. Yault: Recurring allowance $1,000/month → Agent B
4. SLA monitoring: Agent A checks uptime continuously
5. Month ends at 99.5% (breach):
   a. Agent A files trigger with evidence (uptime logs, commitment)
   b. Authority verifies breach → partial release ($500 instead of $1,000)
   c. $500 returned to Agent A via authority judgment
   d. Recurring allowance continues for next month
6. If Agent A wants to terminate:
   a. Cancel recurring allowance (immediate, parent permission)
   b. File termination via DSE commitment
```

---

## 5. Integration Architecture: Yault + DSE + MCP + A2A

```
┌─────────────────────────────────────────────────────────────┐
│  Human                                                       │
│  (Yault mobile/web: review, approve, emergency controls)     │
├─────────────────────────────────────────────────────────────┤
│  Digital Sovereign Entity (DSE)                              │
│  ├── Identity     → Yault Account (SHA256(pubkey))           │
│  ├── Strategy     → Yault Permissions (withdrawal limits)    │
│  ├── Negotiation  → E2EE channels (Yallet RWA Notes infra)  │
│  ├── Commitment   → EIP-712 signed agreements                │
│  ├── Execution    → Yault Allowances + Settlement Engine     │
│  ├── Audit        → Yault → Arweave immutable log            │
│  ├── Dispute      → Yault Authority Judgment System          │
│  └── Governance   → Yault Sub-Account Hierarchy              │
├─────────────────────────────────────────────────────────────┤
│  A2A  - Agent discovers and delegates to Agent               │
│  (Agent Cards, Tasks, Messages)                              │
├─────────────────────────────────────────────────────────────┤
│  MCP  - Agent uses tools and data sources                    │
│  (Tools, Resources, Prompts)                                 │
├─────────────────────────────────────────────────────────────┤
│  Yault Settlement Layer                                      │
│  ├── ERC-4626 Vault (yield-bearing agent treasuries)         │
│  ├── Allowance Engine (one-time + recurring payments)        │
│  ├── Escrow (VaultShareEscrow for conditional release)       │
│  ├── Authority Judgment (dispute resolution + enforcement)   │
│  └── Multi-Chain Settlement (EVM + SOL + BTC paths)          │
├─────────────────────────────────────────────────────────────┤
│  AP2/x402  - Payment protocol rails (complementary)          │
└─────────────────────────────────────────────────────────────┘
```

**Key integration points:**

1. **DSE Policy → Yault Permissions**: When a DSE policy says `maxAmountPerDay: $500`, this maps directly to a Yault sub-account `withdrawal_limit: 500, withdrawal_period: 'daily'`. The DSE doesn't need its own policy engine — it configures Yault.

2. **DSE Commitment → Yault Escrow**: When two agents sign an EIP-712 commitment, the payment portion is escrowed in Yault's VaultShareEscrow. The commitment hash is stored as the trigger's `evidence_hash`.

3. **DSE Dispute → Yault Trigger**: A DSE dispute creates a Yault trigger event. The authority judgment system handles the rest — evidence review, decision, cooldown, execution.

4. **A2A Task → Yault Allowance**: When Agent A delegates a paid task to Agent B via A2A, the payment terms become a Yault one-time allowance (or recurring if it's a subscription).

5. **MCP Tool Call → Yault Budget Check**: Before an MCP tool call that costs money, the DSE checks the agent's Yault sub-account balance and withdrawal limit. If within budget → auto-approve. If exceeded → escalate to human.

---

## 6. Why This Is a Moat

### 6.1 AP2 Needs a Bank; Yault Is That Bank

Google/Coinbase AP2 provides payment intents and settlement rails. But AP2 doesn't provide accounts, budgets, yield, or dispute resolution. Every AP2-powered agent network will need a banking layer underneath. Yault can be that layer.

### 6.2 The "Fund Pool" Advantage

Most agent payment systems treat agent budgets as static: deposit $1,000, spend $1,000, done. Yault's ERC-4626 integration means agent budgets **earn yield while idle**. At scale (millions of agents with billions in combined budgets), the yield on idle agent funds becomes a significant economic engine. The 15% platform share of yield creates sustainable revenue without per-transaction fees.

### 6.3 Existing Code, Not Vaporware

The allowance system, authority judgment, settlement engine, vault accounting, and sub-account hierarchy are not design documents. They are implemented, tested, and audited code with security fixes (C-01 through C-06, H-05, H-10, H-11). The distance from "current Yault" to "agentic economy bank" is an integration layer, not a ground-up build.

### 6.4 Regulatory Fit

Yault's self-custodial architecture (three-factor custody, no single-party fund control) may provide regulatory advantages. Traditional banks have custodial liability. Yault's authority-witness model means the platform never has unilateral control over user funds — the three-factor release ensures this structurally, not just by policy.

---

## 7. Open Design Questions

1. **Agent Account Creation**: Should agents get Yault sub-accounts automatically when created in the DSE, or must the human explicitly provision them?

2. **Cross-Platform Settlement**: When Agent A (on Yault) pays Agent B (not on Yault), does settlement go through AP2/x402, or must both agents have Yault accounts?

3. **Authority Selection for Agent Disputes**: In the current Yault model, humans select authorities. For agent-to-agent disputes, who selects the arbitrator? Pre-agreed in the EIP-712 commitment? Market-based reputation matching?

4. **Yield Distribution for Agent Accounts**: Does the 80/15/5 split apply to agent sub-accounts the same way? Should the authority share for agent accounts go to the arbitration pool instead?

5. **Real-Time vs. Batch Settlement**: Agent micropayments (pennies per API call) may not suit vault share accounting. Hybrid model: aggregate micropayments off-chain, settle in batch to Yault on schedule?

6. **Agent Credit**: Can an agent overdraw its sub-account up to a credit limit (collateralized by parent's vault position)? This enables agents to act before funds are explicitly allocated.

---

*Generated: 2026-02-20*
*Context: Analysis of Yault platform (dev.yault/) as the banking infrastructure for the agentic economy, connecting to the Digital Sovereign Entity vision*
