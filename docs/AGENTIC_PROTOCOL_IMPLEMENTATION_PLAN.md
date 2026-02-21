#Agentic Economy Protocol Implementation Plan

> Based on the existing code base of Yallet (Chrome Extension + Flutter Mobile) + Yault (backend API + smart contract), Claude Code vibe coding is used to implement a complete Agent economic protocol stack.

---

## Overview

| Dimensions | Data |
|------|------|
| **Involved projects** | dev.yallet.chrome-extension, dev.yallet.mobile-app, dev.yault, dev.yallet.backend-api |
| **Add/modify API endpoints** | ~25 |
| **New TypeScript (Extension)** | ~3,000 lines |
| **New Dart (Mobile)** | ~2,000 rows |
| **New JavaScript (Yault backend)** | ~1,500 lines |
| **New/Modify Solidity** | ~200 lines |
| **MCP/A2A Protocol Definition (JSON Schema)** | ~500 lines |
| **Total code size** | ~7,000-8,000 lines |
| **Claude Code estimated construction period** | 11-16 working days (2-3 weeks) |

---

## Phase 1: Agent account + policy engine (2-3 days)

> **Goal**: Agent can have Yault sub-account, and Yallet can automatically approve transactions based on strategy

### 1.1 Yault backend: Agent sub-account

**Modify files**: `dev.yault/server/models/schemas.js`, `dev.yault/server/api/accounts/`

| Mission | Interface/Changes | Description |
|------|----------|------|
| Add agent role to SubAccount | Modify `SubAccount` schema | Add `"agent"` to `role` enumeration and add `agent_metadata` fields (agent_type, model_provider, capabilities) |
| Agent exclusive permission field | Modify `permissions` object | Add `allowed_operations: string[]`, `allowed_chains: string[]`, `max_concurrent_txs: number`, `time_window: {start, end}` |
| Create Agent accounts in batches | `POST /api/accounts/agents/batch` | Organizations can create multiple Agent sub-accounts at one time, each with independent permission configuration |
| Agent budget aggregation query | `GET /api/accounts/agents/summary` | Returns the budget usage, balance, income, and active status of all Agents |
| Agent budget details query | `GET /api/accounts/agents/:id/budget` | Daily/weekly/monthly consumption details + remaining budget of a single Agent |

**Estimated change**: ~300 lines of JavaScript

### 1.2 Yallet Extension: Policy Engine

**New files**: `public/lib/agent-policy.ts`, `public/lib/agent-budget.ts`
**Modify file**: `public/background.ts`

| Task | Description |
|------|------|
| **AgentPolicy data structure** | Define policy rule interface (see below) |
| **PolicyEngine.checkPolicy()** | Insert a policy check layer before the background.ts approval queue |
| **BudgetTracker** | Sliding window counter to track the daily/weekly/monthly expenditure of each Agent |
| **Policy Storage** | chrome.storage.local storage policy JSON, Ed25519 signature tamper-proof |
| **Automatic approval path** | Matching strategy → Automatically resolve, no pop-up window |
| **Out-of-bounds processing** | Out of policy → go to the original approve.html pop-up window / push ReviewRequest |

```typescript
// public/lib/agent-policy.ts
interface AgentPolicy {
  agentId: string;
  agentLabel: string;
  scope: 'auto_payment' | 'negotiation' | 'commitment' | 'full';
  conditions: {
    maxAmountPerTx: number;
    maxAmountPerDay: number;
    maxAmountPerWeek: number;
    maxAmountPerMonth: number;
    allowListAddresses: string[];
    allowListChains: string[];
allowListMethods: string[]; // Allowed contract method signatures
    minBalanceAfter: number;
    requireReviewBeforeFirstPay: boolean;
timeWindow?: { start: string; end: string }; // HH:MM format
  };
  escalation: 'block' | 'ask_parent_agent' | 'ask_human';
  parentAgentId?: string;
  createdAt: string;
  signature: string;                  // Ed25519(JSON.stringify(conditions))
}

// public/lib/agent-budget.ts
interface AgentBudgetTracker {
  agentId: string;
  dailySpent: number;
  weeklySpent: number;
  monthlySpent: number;
  transactions: Array<{
    amount: number;
    timestamp: string;
    txHash: string;
    chain: string;
    method: string;
  }>;
}
```

**Estimated changes**: ~700 lines of TypeScript

### 1.3 Yallet Extension: Policy Management UI

**Modify files**: `public/popup.html`, add `public/wallets/agent-policies.ts`

| Task | Description |
|------|------|
| Agent Policies configuration panel | New "Agent Policies" subpage in Settings area |
| Policy creation/edit form | Agent selection, quota configuration, whitelist management, time window |
| Budget dashboard | Real-time display of each Agent’s consumption/balance/income |
| One-click freeze | Emergency Freeze button (stops all Agent activity) |

**Estimated changes**: ~400 lines of TypeScript + HTML

### Phase 1 demonstrable results

> An Agent that automatically executes DCA (regular fixed-amount purchase): Automatically sign within the budget without pop-up windows, and pop-up windows for approval when the daily limit is exceeded.

---

## Phase 2: Agent identity + negotiation channel (2-3 days)

> **Goal**: Agents have verifiable on-chain identities, and agents can negotiate through E2EE channels

### 2.1 Agent identity derivation

**Modify file**: `public/lib/acegf.ts` (call the existing WASM), add `public/lib/agent-identity.ts`

| Task | Description |
|------|------|
| **BIP44 Agent subpath** | `m/44'/501'/0'/0'/agent_index'` Derive Agent-specific Ed25519 key pair |
| **Agent identity certificate** | `{ agentId, pubkey, ownerXidentity, capabilities, policyHash, ownerSignature, createdAt }` |
| **Certificate Minting** | Option 1: Mint as cNFT to Solana (reuse existing Metaplex Bubblegum process); Option 2: Store to Arweave (reuse existing upload channel) |
| **DID wrapper** | `did:yallet:<agentId>` format, resolve returns certificate JSON |

```typescript
// public/lib/agent-identity.ts
interface AgentIdentityCertificate {
  version: '1.0';
  agentId: string;                    // SHA256(agent_pubkey)
pubkey: string; // Agent’s Ed25519 public key (hex)
ownerXidentity: string; // xidentity of human owner
  capabilities: string[];            // ['payment', 'negotiation', 'data_query']
  policyHash: string;                // SHA256(AgentPolicy JSON)
maxAutonomousAmount: number; // Maximum autonomous operation amount
  chains: string[];                  // ['solana', 'ethereum', 'polygon']
  createdAt: string;
  expiresAt: string;
ownerSignature: string; // Human signed with master key
registrationTx?: string; // cNFT cast tx hash or Arweave tx id
}

// Derived function (call existing WASM)
async function deriveAgentKeypair(
  mnemonic: string,
  agentIndex: number
): Promise<{ pubkey: string; secretKey: Uint8Array }> {
// Use the derive_path function of acegf.wasm
  // path: m/44'/501'/0'/0'/{agentIndex}'
}
```

**Estimated changes**: ~300 lines of TypeScript

### 2.2 Negotiate Message Protocol

**Modify files**: `public/lib/api-client.ts` (E2EE layer), add `public/lib/agent-negotiation.ts`

| Task | Description |
|------|------|
| **Negotiation Message Type** | Extend existing RWA asset_type enumeration |
| **Negotiation state machine** | `initial → offer → counter* → accept/reject → commitment` |
| **E2EE negotiation channel** | Reuse X25519 ECDH + AES-GCM encryption pipeline |
| **Arweave negotiation record** | Each round of negotiation is encrypted and stored in Arweave (for auditing) |
| **EIP-712 Commitment Signing** | After the negotiation reaches a consensus, both parties sign a structured commitment |

```typescript
// public/lib/agent-negotiation.ts

//Add RWA message type
type AgentMessageType =
| 'negotiation_offer' // Offer: { item, price, terms, deadline }
| 'negotiation_counter' // Counteroffer: { item, counterPrice, counterTerms }
| 'negotiation_accept' // Accept: { agreementHash }
| 'negotiation_reject' // Reject: { reason }
| 'commitment_proposal' // EIP-712 commitment proposal: { typedData }
| 'commitment_signed' // Signed commitment: { typedData, signature }
| 'dispute_evidence' // Dispute evidence: { evidenceHash, description }
| 'review_request'; // Human review request: { ReviewRequest }

// Negotiation state machine
type NegotiationState =
  | 'initial'
  | 'offer_sent'
  | 'offer_received'
  | 'countering'
  | 'accepted'
  | 'rejected'
| 'committed' // EIP-712 signing completed
  | 'disputed';

interface NegotiationSession {
  sessionId: string;
  myAgentId: string;
  counterpartyAgentId: string;
  state: NegotiationState;
  rounds: NegotiationRound[];
  commitment?: EIP712Commitment;
  createdAt: string;
  updatedAt: string;
}

interface NegotiationRound {
  roundNumber: number;
  sender: string;             // agentId
  messageType: AgentMessageType;
payload: object; // Specific offer/counteroffer content
  encryptedArweaveId: string; // Arweave tx id
  timestamp: string;
}

// EIP-712 Structured Commitment
interface EIP712Commitment {
  domain: {
    name: 'YalletAgentCommitment';
    version: '1';
    chainId: number;
  };
  types: {
    Commitment: Array<{ name: string; type: string }>;
  };
  value: {
buyerAgent: string; // Agent A public key
sellerAgent: string; // Agent B public key
item: string; // product/service description hash
price: string; //Amount (wei/lamports)
currency: string; // Token address
deliveryDeadline: number; // Unix timestamp
    arbitrator: string;       // Yault authority_id
    escrowRequired: boolean;
    nonce: number;
  };
  buyerSignature?: string;
  sellerSignature?: string;
}
```

**Estimated changes**: ~500 lines of TypeScript

### Phase 2 demonstrable results

> Agents in two Yallet instances automatically negotiate prices through the E2EE channel, and after reaching a consensus, both parties sign the EIP-712 commitment.

---

## Phase 3: Mobile Approval + Agent Level (2-3 days)

> **Goal**: When the Agent exceeds the policy, push it to the mobile phone for approval, support Agent-level delegation

### 3.1 ReviewRequest Protocol

**Modify file**: `dev.yallet.backend-api/src/index.js` (push channel)
**New file**: Extension side `public/lib/review-request.ts`

| Task | Description |
|------|------|
| **ReviewRequest data structure** | Define the approval request format (see below) |
| **Extension → Backend → FCM** | Policy engine generates ReviewRequest → Send to backend-api → FCM push to mobile phone |
| **Timeout mechanism** | Automatic rejection when deadline expires |
| **Upgrade Chain** | Child Agent → Parent Agent → Human |

```typescript
// public/lib/review-request.ts
interface ReviewRequest {
  requestId: string;              // UUID
  agentId: string;
  agentLabel: string;
  action: 'transfer' | 'sign' | 'approve' | 'negotiate' | 'commitment';
summary: string; // Human-readable summary (e.g. "Shopping Agent wants to spend $350 on headphones")
  details: {
    chain: string;
    to: string;
    amount: string;
    currency: string;
method?: string; // Contract method
context?: string; // Negotiation context summary
  };
  policyViolation: {
rule: string; // Violated policy rule
actual: string; // actual value
limit: string; // limit value
  };
  urgency: 'low' | 'normal' | 'high' | 'critical';
deadline: string; // ISO timestamp
escalatedFrom?: string; // Which sub-Agent it was upgraded from
  createdAt: string;
}

interface ReviewResponse {
  requestId: string;
  decision: 'approve' | 'reject' | 'modify';
modifiedAmount?: string; // if decision = 'modify'
  respondedAt: string;
  respondedVia: 'mobile' | 'extension';
  biometricVerified: boolean;
}
```

**Estimated changes**: ~300 lines of TypeScript + ~200 lines of JavaScript (backend-api)

### 3.2 Flutter Mobile: Approval interface

**New file**: `dev.yallet.mobile-app/lib/screens/agent/`

| Task | Description |
|------|------|
| **ReviewRequestScreen** | Approval details page: Agent information, operation summary, policy violation reasons, approve/reject/modify buttons |
| **AgentListScreen** | Agent list: status, budget usage, recent activity |
| **AgentDetailScreen** | Single Agent details: strategy configuration, budget details, transaction history |
| **FCM deep link** | Receive ReviewRequest push → click → directly open ReviewRequestScreen |
| **Biometric Confirmation** | High value approvals (>threshold) require fingerprint/face confirmation |
| **EmergencyFreezeButton** | Freeze all Agents with one click |

**Estimated changes**: ~1,500 lines Dart

### 3.3 Agent level management

**Modify files**: Extension `public/lib/agent-policy.ts`, Yault `server/api/accounts/`

| Task | Description |
|------|------|
| **Hierarchical relationship** | Agent can have parentAgentId (parent Agent delegates child Agent) |
| **Permission inheritance** | Child Agent permissions ≤ parent Agent permissions (automatic constraints) |
| **Upgrade Chain** | The child Agent crosses the boundary → ask the parent Agent first → the parent Agent also crosses the boundary → push to human |
| **Budget Allocation** | The parent Agent can allocate a subset of the budget to the child Agent |

```
Hierarchy example:
Human (Yault master account)
├── Shopping Agent (monthly budget $500)
│ ├── Price Comparison Agent (read only, no payment permission)
│ └── Place an order Agent (single order ≤ $100)
├── Research Agent (monthly budget $200)
│ └── Data collection sub-Agent (daily budget $20)
└── Finance Agent (monthly budget $1,000)
```

**Estimated changes**: ~400 lines of TypeScript + ~200 lines of JavaScript

### Phase 3 demonstrable results

> Shopping Agent wants to spend $350 to buy headphones (exceeding the single transaction limit of $100) → FCM is pushed to the mobile phone → ReviewRequest details are displayed → User fingerprint confirms approval → Agent executes payment.

---

## Phase 4: MCP/A2A protocol bridging (3-4 days)

> **Goal**: Yallet/Yault’s capabilities can be discovered and called by external AI Agent frameworks

### 4.1 MCP Tool Definition

**New files**: `dev.yault/server/mcp/tools.json`, `dev.yault/server/mcp/server.js`

Expose Yault API as MCP Tools:

| Tool name | Corresponding to Yault API | Parameters |
|-----------|---------------|------|
| `yault_check_balance` | `GET /api/vault/balance/:address` | address, chain |
| `yault_deposit` | `POST /api/vault/deposit` | address, amount, chain |
| `yault_redeem` | `POST /api/vault/redeem` | address, shares, chain |
| `yault_create_allowance` | `POST /api/accounts/allowances` | from, to, amount, type, frequency |
| `yault_cancel_allowance` | `PUT /api/accounts/allowances/:id/cancel` | allowance_id |
| `yault_file_dispute` | `POST /api/trigger/initiate` | wallet_id, recipient_index, reason_code, evidence_hash |
| `yault_check_budget` | `GET /api/accounts/agents/:id/budget` | agent_id |
| `yault_list_agents` | `GET /api/accounts/agents/summary` | parent_wallet_id |

```json
// MCP Tool definition example
{
  "name": "yault_create_allowance",
  "description": "Create a one-time or recurring payment allowance from one agent account to another",
  "inputSchema": {
    "type": "object",
    "properties": {
      "from_wallet_id": { "type": "string", "description": "Sender agent's Yault account ID" },
      "to_wallet_id": { "type": "string", "description": "Recipient's Yault account ID" },
      "amount": { "type": "string", "description": "Amount in underlying asset (e.g. USDC)" },
      "type": { "type": "string", "enum": ["one_time", "recurring"] },
      "frequency": { "type": "string", "enum": ["daily", "weekly", "monthly"] },
      "memo": { "type": "string", "description": "Human-readable description of the payment" }
    },
    "required": ["from_wallet_id", "to_wallet_id", "amount", "type"]
  }
}
```

**Estimated changes**: ~500 lines of JavaScript + ~300 lines of JSON Schema

### 4.2 MCP Server implementation

**New file**: `dev.yault/server/mcp/server.js`

| Task | Description |
|------|------|
| **MCP Server Entry** | Implement MCP Server protocol (stdio or SSE transmission) |
| **Tool Handler routing** | Route the MCP tool call to the corresponding Yault API |
| **Authentication Bridge** | MCP call carries Agent identity → mapped to Yault session token |
| **Policy Check Integration** | Each Tool call is first checked by the policy engine |
| **Resource Exposed** | Agent budget and transaction history are exposed as MCP Resource |

**Estimated changes**: ~400 lines of JavaScript

### 4.3 A2A Agent Card

**New file**: `dev.yault/server/a2a/agent-card.js`

| Task | Description |
|------|------|
| **Agent Card Generation** | Generate A2A Agent Card based on Yault Agent sub-account information |
| **/.well-known/agent.json** | Publish the discoverable Agent Card endpoint |
| **Task Receive** | Receive task requests from other A2A Agents |
| **Message Bridge** | A2A Message → Yallet E2EE Negotiation Channel |

```json
// A2A Agent Card example
{
  "name": "Alice's Shopping Agent",
  "description": "Autonomous shopping agent with $500/month budget on Yault",
  "url": "https://yault.app/a2a/agent/abc123",
  "provider": { "organization": "Yault", "url": "https://yault.app" },
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "purchase",
      "name": "Purchase items",
      "description": "Can purchase items up to $100/transaction, $500/month total",
      "inputModes": ["application/json"],
      "outputModes": ["application/json"]
    },
    {
      "id": "negotiate",
      "name": "Negotiate prices",
      "description": "Can negotiate with vendor agents via E2EE channel",
      "inputModes": ["application/json"],
      "outputModes": ["application/json"]
    }
  ],
  "authentication": {
    "schemes": ["ed25519"]
  },
  "defaultInputModes": ["application/json"],
  "defaultOutputModes": ["application/json"]
}
```

**Estimated changes**: ~400 lines of JavaScript

### Phase 4 demonstrable results

> External AI Agent (such as LangChain Agent) discovers Yault's `yault_check_balance` and `yault_create_allowance` tools through MCP, queries the Agent's budget and creates a payment quota.

---

## Phase 5: AP2 adaptation + end-to-end integration testing (2-3 days)

> **Goal**: Connect with Google/Coinbase AP2 payment protocol, complete link end-to-end verification

### 5.1 AP2 Adaptation Layer

**New file**: `dev.yault/server/ap2/adapter.js`

| Task | Description |
|------|------|
| **PaymentIntent receives** | AP2 PaymentIntent → Parse amount/payee/chain |
| **Budget Check** | Check Agent's budget balance and limit in Yault |
| **Vault Redemption** | Redeem the corresponding amount from the ERC-4626 vault |
| **On-chain settlement** | Complete payment through native transfer/contract call |
| **Settlement Confirmation** | Return to AP2 PaymentConfirmation |

```typescript
// AP2 adaptation process
interface AP2PaymentFlow {
// 1. Receive AP2 PaymentIntent
  intent: {
    amount: string;
currency: string; // Token address
recipient: string; // Recipient address
    chain: string;
    metadata: {
agentId: string; // Initiator Agent
purpose: string; // Payment purpose
commitmentHash?: string; // Associated EIP-712 commitment
    };
  };

// 2. Yault internal processing
  processing: {
budgetCheck: boolean; // Is the Agent's budget sufficient?
policyCheck: boolean; // Whether the policy engine allows it or not
vaultRedeem: string; // Vault redemption tx hash
nativeTransfer: string; // On-chain transfer tx hash
  };

// 3. Return to AP2 for confirmation
  confirmation: {
    status: 'completed' | 'failed' | 'pending_review';
    txHash: string;
    settledAt: string;
auditId: string; // Yault audit record ID
  };
}
```

**Estimated changes**: ~400 lines of JavaScript

### 5.2 End-to-end integration testing

| Test Scenario | Covered Components | Verification Points |
|---------|-----------|--------|
| **Agent DCA automatic execution** | Extension policy engine + Yault vault | Automatic signature within the policy, correct budget tracking |
| **Agent over-limit approval** | Extension → backend-api → FCM → Mobile | ReviewRequest is pushed to the mobile phone and executed after approval |
| **Agent Negotiation + Commitment** | Extension E2EE + EIP-712 | Two Agents negotiate to reach a consensus and sign a commitment |
| **Agent Dispute Arbitration** | Extension → Yault trigger → authority → Arweave | Complete dispute life cycle (trigger → Judgment → Cooling → Execution) |
| **MCP external call** | MCP Server → Yault API | External AI framework calls Yault tools through MCP |
| **AP2 payment** | AP2 intent → Yault redemption → on-chain transfer | The complete process from PaymentIntent to on-chain confirmation |
| **Emergency Freeze** | Mobile → Extension → Yault | All Agent operations are blocked after one-click freezing |

**Estimated changes**: ~500 lines of test code

### Phase 5 demonstrable results

> Complete demo: Agent A (Yallet) discovers Vendor Agent B through A2A → E2EE negotiates price → signs EIP-712 commitment → Yault escrow funds → Vendor delivers → releases escrow → full Arweave audit record.

---

## File list summary

### Add new file

| Project | File Path | Purpose |
|------|---------|------|
| Extension | `public/lib/agent-policy.ts` | Policy engine core |
| Extension | `public/lib/agent-budget.ts` | Budget Tracker |
| Extension | `public/lib/agent-identity.ts` | Agent identity derivation + certificate |
| Extension | `public/lib/agent-negotiation.ts` | Negotiation protocol + state machine |
| Extension | `public/lib/review-request.ts` | ReviewRequest protocol |
| Extension | `public/wallets/agent-policies.ts` | Policy Management UI |
| Mobile | `lib/screens/agent/review_request_screen.dart` | Approval interface |
| Mobile | `lib/screens/agent/agent_list_screen.dart` | Agent list |
| Mobile | `lib/screens/agent/agent_detail_screen.dart` | Agent details |
| Mobile | `lib/blocs/agent/agent_bloc.dart` | Agent state management |
| Yault | `server/api/accounts/agents.js` | Agent Account API |
| Yault | `server/mcp/tools.json` | MCP Tool definition |
| Yault | `server/mcp/server.js` | MCP Server implementation |
| Yault | `server/a2a/agent-card.js` | A2A Agent Card |
| Yault | `server/ap2/adapter.js` | AP2 adaptation layer |
| Backend-API | `src/review-request.js` | ReviewRequest push |

### Modify files

| Project | File path | Modify content |
|------|---------|---------|
| Extension | `public/background.ts` | Insert policy inspection layer + Agent scheduler |
| Extension | `public/popup.html` | Add Agent Policies UI |
| Extension | `public/lib/api-client.ts` | Add negotiation message type |
| Extension | `public/lib/acegf.ts` | Agent subpath derivation |
| Yault | `server/models/schemas.js` | SubAccount Add agent role + permissions |
| Yault | `server/api/accounts/index.js` | Agent batch creation + aggregate query |
| Yault | `contracts/src/YaultVault.sol` | Agent identification field (minimal changes) |
| Mobile | `lib/core/notifications/notification_handler.dart` | ReviewRequest deep link |
| Mobile | `lib/main.dart` | Register Agent BLoC |
| Backend-API | `src/index.js` | ReviewRequest push endpoint |

---

## Risks and Dependencies

| Risk | Mitigation |
|------|------|
| MCP specification may be updated | Adopt adapter mode, MCP changes only change `mcp/server.js` |
| The A2A specification is not yet stable | Agent Card is generated as a configurable template, and only the template changes when the specification changes |
| AP2 documentation is limited | Phase 5 can be moved later; use mock AP2 intent development first |
| Extension ↔ Mobile communication link is complex | Relayed through backend-api, no direct connection |
| Flutter ACEGF FFI compatibility | Mobile App already has ACEGF FFI binding, you can reuse it |

---

## Milestone Checkpoint

| Days | Checkpoint | Pass Criteria |
|------|--------|---------|
| Day 3 | Phase 1 completed | Agent sub-account created successfully + strategy engine automatically approves DCA transactions |
| Day 6 | Phase 2 completed | Two Agents completed negotiation through E2EE + EIP-712 commitment signing |
| Day 9 | Phase 3 completed | ReviewRequest pushed from Extension to mobile phone + biometric approval |
| Day 13 | Phase 4 completed | External AI successfully called the Yault tool through MCP |
| Day 16 | Phase 5 completed | End-to-end demo run through (negotiation → commitment → hosting → delivery → release) |

---

*Generation date: 2026-02-20*
*Implementation plan based on Yallet Chrome Extension + Yallet Flutter Mobile App + Yault Platform + Backend-API code base*
*Estimated use of Claude Code vibe coding, total construction period 11-16 working days*
