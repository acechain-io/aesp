# Yallet: Digital Sovereign Terminal in the Age of Agentic Economy - Feasibility Analysis

> This article systematically analyzes the feasibility of Yallet as a "Digital Sovereign Terminal" in Agentic Economy from five dimensions: technical architecture, existing capabilities, market positioning, protocol compatibility, and competitive landscape.

---

## 1. Core proposition

### 1.1 What is "Digital Sovereign Terminal"

In Agentic Economy, AI Agents perform economic decisions - shopping, investing, negotiating, signing contracts, and payments - on behalf of humans. But humans cannot completely relinquish control. **Digital Sovereign Terminal (DST)** is the entrance for humans to exercise sovereignty:

- **Identity Anchor**: All Agent’s economic behaviors are ultimately traced back to human key identities
- **Strategy Center**: Humans set the Agent's behavioral boundaries (budget, whitelist, authorization level) through the terminal
- **Approval Gateway**: Operations outside the policy scope require human approval on the terminal
- **Audit Window**: Humans can view the economic activity history of all Agents through the terminal
- **Emergency Control**: Humans can freeze or revoke the permissions of any Agent at any time

**Analogy**: If Agentic Economy were a country, DST would be the citizen's "digital passport + mobile banking + control panel".

### 1.2 Why Yallet

Yallet is not an agentic product designed from scratch. It is a **multi-chain non-custodial wallet** that is already running in a production environment, with 22 functional modules, 10+ blockchain support, complete E2EE communication infrastructure, and a cNFT-based encrypted asset management system. What this article will argue is: **Yallet's existing architecture is naturally adapted to all the needs of DST. The upgrade path from "current wallet" to "digital sovereign terminal" is incremental, not reconstructive. **

---

## 2. Technical feasibility analysis

### 2.1 Identity layer: from "wallet address" to "sovereign identity"

#### Already capable

| Capabilities | Implementation details | Code location |
|------|---------|---------|
| **BIP39/BIP32/BIP44 hierarchical deterministic derivation** | WASM module (acegf.wasm) supports derivation of Solana (Ed25519), EVM (ECDSA), Bitcoin (UTXO), Cosmos, Polkadot addresses from a single mnemonic | `acegf.ts` + `acegf.wasm` |
| **xidentity Unified Identity** | `yallet_getAddresses()` RPC method returns 7 fields JSON: Solana, EVM, Bitcoin, Cosmos, Polkadot, xaddress, xidentity | `FEASIBILITY_yallet_getAddresses.md` |
| **Passkey/WebAuthn authentication** | PRF (Pseudo-Random Function) derives hardware-bound keys, no need to remember passwords | `passkey.ts` |
| **VA-DAR decentralized backup** | Password + mnemonic phrase never leaves the device; encrypted backups are stored in IPFS and indexes are stored on the Solana chain | `vadar-security.md` |
| **Ed25519 Signature** | All critical operations (authentication, transactions, RWA encryption) use deterministically derived Ed25519 keys | `api-client.ts` |

#### Upgrade path to DST identity layer

**Status quo**: Yallet's xidentity is already a multi-chain unified identity identifier - the address of all chains is deterministically derived from a mnemonic phrase. This itself is a prototype of a "digital sovereign identity".

**Need to add**:

1. **Agent identity derivation**: Assign each Agent a sub-path (such as `m/44'/501'/0'/agent_index'`) in the BIP44 path tree so that each Agent has an independent key pair but can be traced back to the same sovereign identity. This is a natural capability of BIP32-level deterministic wallets - Yallet's WASM module already supports arbitrary path derivation.

2. **Agent identity registration**: Each Agent's public key + policy constraints + human authorized signature, packaged into an "Agent identity certificate". It can be minted on the chain as a cNFT (Yallet has a complete cNFT minting process), or it can be stored in Arweave (Yallet has an Arweave upload channel).

3. **DID (decentralized identification) compatibility**: xidentity can be packaged into W3C DID format (`did:yallet:<xidentity>`), so that the Agent identity can be discovered and verified by the MCP/A2A protocol.

**Feasibility Assessment**: **High**. Core cryptography infrastructure is in place (WASM derivation + Ed25519 signing + cNFT minting + Arweave storage), with incremental work of ~200-400 lines of TypeScript.

### 2.2 Strategy layer: from "Transaction Approval" to "Agent Strategy Engine"

#### Already capable

Yallet’s current transaction approval process:

```
dApp request → content.ts forwarding → background.ts queue → approve.html popup → user approval/rejection
```

Key features of this process:
- **Request Queue**: `background.ts` maintains the pending approval queue, with a 5-minute timeout
- **Independent Approval Window**: Each approval request is presented in an independent Chrome window (`approve.html`) to prevent UI hijacking
- **Parameter visualization**: Approval pop-up window displays complete transaction parameters (target address, amount, Gas, method signature)
- **Security check**: address poisoning detection, malicious address blacklist, contract detection, transaction simulation (eth_call)
- **Permission Management**: List of connected sites (`connectedSites`), which can be disconnected individually

#### Upgrade path to DST policy engine

**Current situation**: Yallet's approval process is "manual approval one by one" - a pop-up window confirms each dApp request. In Agentic Economy, agents may perform hundreds of operations every day, and case-by-case approval is not feasible.

**Need to add**:

1. **Policy Engine**

Insert a policy check layer before the approval queue in `background.ts`:

```typescript
// Policy rule data structure (already designed in AGENT_DESIGN.md)
interface AgentPolicy {
agentId: string; // Agent identity
  scope: 'auto_payment' | 'negotiation' | 'commitment';
  conditions: {
maxAmountPerTx: number; // Single transaction limit
maxAmountPerDay: number; //Daily limit
maxAmountPerWeek: number; // Weekly limit
allowListAddresses: string[]; // whitelist address
allowListChains: string[]; // Allowed chains
allowListMethods: string[]; // Allowed contract methods
minBalanceAfter: number; // Minimum balance after transaction
requireReviewBeforeFirstPay: boolean; // The first payment requires approval
timeWindow: { start: string; end: string }; // The time window in which operations are allowed
  };
escalation: 'block' | 'ask_parent' | 'ask_human'; // Out-of-bounds processing
}
```

**Decision Logic**:
- Request matching policy conditions → **Automatic approval** (Agent executes freely, no pop-up window)
- The request exceeds the policy but can be upgraded → **Notify the parent Agent or push the mobile phone for approval**
- The request seriously crosses the boundary → **Directly blocked**

This does not require refactoring the existing approval process - just add a `checkPolicy()` layer before `handleApproval()` in `background.ts`. Requests that match the policy are resolved directly, and those that do not match the policy go through the original pop-up window process.

2. **Policy Storage and Synchronization**

Policies are stored in `chrome.storage.local` (shared storage layer with existing settings) in JSON format. Policy changes are signed with Ed25519 (tamper-proof) and optionally synced to Arweave (audit-immutable).

3. **Dynamic Budget Tracking**

Add a sliding window counter in `background.ts` to track the daily/weekly/monthly expenditure of each Agent:

```typescript
// The existing chrome.storage.local already supports persistence
interface AgentBudgetTracker {
  agentId: string;
dailySpent: number; // Rolling 24 hours spent
weeklySpent: number; // Spending within rolling 7 days
monthlySpent: number; // Spending within rolling 30 days
lastReset: string; // ISO timestamp
  transactions: { amount: number; timestamp: string; txHash: string }[];
}
```

**Feasibility Assessment**: **High**. The existing approval process is a complete "request → check → decision → execution" pipeline, and the policy engine adds a layer of automated decision-making before the "check" step. The core data structure has been designed in `AGENT_DESIGN.md`, and the implementation workload is about 600-800 lines of TypeScript.

### 2.3 Communication layer: from "E2EE encrypted notes" to "Agent negotiation channel"

#### Already capable

Yallet’s E2EE infrastructure is one of its most unique technology assets:

| Capability | Implementation | Usage |
|------|------|------|
| **X25519 ECDH key exchange** | `api-client.ts` - `initE2E()` | Establish a shared secret with the server |
| **AES-GCM symmetric encryption** | 128-bit key, random nonce | Message payload encryption |
| **XChaCha20-Poly1305** | WASM module (acegf) | RWA asset data encryption |
| **Ed25519 Signature** | Mnemonic derivation key | Message integrity verification |
| **Arweave Persistent Storage** | Irys/ArDrive Turbo | Immutable storage of encrypted data |
| **cNFT metadata** | Metaplex Bubblegum | Asset reference chain anchoring |
| **Recipient Encryption** | Public key encryption based on xidentity | Directed sending of crypto assets |

**Supported crypto asset types**: Notes (tagged rich text), Invoices (with line items, tax rates, PDF/CSV export), Photos, Files, Contacts (vCard), Profile, Settings.

#### Upgrade path to Agent negotiation channel

The core capability required for inter-agent negotiation is **structured E2EE message exchange** - which is exactly what the Yallet RWA system is already doing.

**Current situation**: Yallet's RWA system encrypts structured data (JSON envelope) and stores it in Arweave, and the receiver decrypts it through the xidentity public key. This process can be directly reused as the underlying communication of the Agent negotiation protocol:

```
Current RWA process:
Sender → Construct JSON envelope → X25519 encryption → Upload Arweave → Mint cNFT → Receiver decrypt

Agent negotiation process (reuse):
Agent A → Construct negotiation message (offer/counteroffer/acceptance) → X25519 encryption → Storage → Agent B decryption → Reply
```

**Need to add**:

1. **Negotiation message type extension**

Currently RWA supports 7 asset types (note/invoice/photo/file/contact/profile/settings). Added new types for Agent negotiations:

```typescript
//Add RWA type
type AgentMessageType =
| 'negotiation_offer' // offer
| 'negotiation_counter' // Counteroffer
| 'negotiation_accept' // accept
| 'negotiation_reject' // Reject
| 'commitment_proposal' // EIP-712 commitment proposal
| 'commitment_signed' // Signed commitment
| 'dispute_evidence' // Dispute evidence
| 'review_request'; // Approval request (push to mobile phone)
```

2. **Real-time channel (optional)**

Currently RWA is asynchronous (Arweave storage + polling). For high-frequency negotiation scenarios, a WebSocket real-time channel (based on the E2EE layer of `api-client.ts`) can be added while retaining Arweave as an immutable audit log.

3. **Negotiation state machine**

Each round of negotiation is a finite state machine: `initial → offer → counter* → accept/reject`. State transitions are recorded locally (IndexedDB, Yallet already has) + Arweave (immutable auditing).

**Feasibility Assessment**: **High**. Yallet's E2EE + Arweave + cNFT communication pipeline is a capability that very few wallets on the market have. Competing products (MetaMask, Phantom, Trust Wallet) do not have end-to-end encrypted messaging systems. Incremental effort ~400-600 lines of TypeScript (new message type + state machine).

### 2.4 Execution layer: from "transaction signature" to "Agent economic execution"

#### Already capable

Economic operations currently supported by Yallet:

| Operation type | Supported chain | Implementation method |
|---------|---------|---------|
| **Native Token Transfer** | Solana, 10 EVM chains | `solana.ts`, `ethereum.ts` |
| **Token transfer** | ERC-20 (all EVM), SPL (Solana) | Token standard contract call |
| **DEX EXCHANGE** | Solana (Jupiter), EVM (1inch/0x) | Aggregator API + Signature |
| **Staking** | Solana (Marinade mSOL), Ethereum (Lido stETH) | Protocol contract call |
| **Cross-chain bridging** | All supported chains | LI.FI routing |
| **NFT Casting** | Solana (Metaplex Bubblegum) | cNFT Compression Casting |
| **Contract interaction** | All EVM chains | Common `eth_sendTransaction` |
| **WalletConnect** | EVM + Solana | SignClient v2 Protocol |

#### Upgrade path to Agent execution layer

**Status quo**: Yallet is already a complete multi-chain transaction signature engine. In Agentic Economy, the types of operations that Agents need to perform highly overlap with the types of operations currently supported by Yallet:

| Operations required by Agent | Yallet’s existing capabilities | Gap |
|----------------|------------------|------|
| Payment for goods/services | Token transfer + DEX currency exchange | None - already available |
| Subscription payment | Contract interaction (approve + transferFrom) | Need to add scheduled trigger |
| Cross-chain payment | LI.FI bridge | None - already available |
| DeFi operations (pledge/liquidity) | Marinade + Lido + general contract call | More protocol adaptations need to be added |
| Sign on-chain commitment | Ed25519/ECDSA signature | Need to add EIP-712 structured signature |
| dApp Authorization | WalletConnect v2 | None - Already Available |

**Need to add**:

1. **Timing Executor**

Use Chrome Extension's `chrome.alarms` API (Yallet has been used for notification polling, with an interval of 5 minutes) to add an Agent scheduled task executor:

```typescript
// Reuse existing chrome.alarms infrastructure
chrome.alarms.create('agent-scheduler', { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener(async (alarm) => {
  if (alarm.name === 'agent-scheduler') {
    const tasks = await getScheduledAgentTasks();
    for (const task of tasks) {
      if (task.nextExecution <= Date.now()) {
        const policy = await getAgentPolicy(task.agentId);
        if (checkPolicy(policy, task.execution)) {
await executeTask(task); // Reuse existing signature engine
        }
      }
    }
  }
});
```

2. **EIP-712 Structured Signature**

Used for signing commitments between agents (commodity delivery commitments, SLA commitments, service contracts). Yallet already supports `eth_signTypedData_v4` (in WalletConnect), which needs to be opened as an independent API.

3. **Batch execution optimization**

The Agent may need to perform multiple operations in one transaction (such as "exchange currency + payment + mint receipt"). You can take advantage of Solana's native multi-instruction transactions (Yallet already supports it) or EVM's multi-call contract (multicall).

**Feasibility Assessment**: **High**. Yallet's transaction signature engine already covers most types of operations performed by the Agent economy. The incremental work is focused on the timed executor (~300 lines) and the EIP-712 open API (~200 lines).

### 2.5 Security layer: from "transaction security" to "Agent security boundary"

#### Already capable

Yallet’s current security system:

| Security capabilities | Implementation |
|---------|------|
| **Address poisoning detection** | Leading zero heuristic check |
| **Malicious Address Blacklist** | Extensible API Integration |
| **Contract Address Detection** | eth_getCode Check |
| **Trade Simulation** | eth_call pre-execution |
| **Token authorization scan** | ERC-20 approve scan + unlimited authorization tokens |
| **Authorization Revoke** | Construct revoke transaction with one click |
| **Security Alarm History** | Hierarchical alarms (critical/high/medium/low) + history records |
| **Content Security Policy** | CSP header enforcement |
| **Provider injection protection** | window.ethereum is not writable to prevent malicious overwriting |
| **Approval Timeout** | Automatically rejected if no operation is performed for 5 minutes |
| **Mnemonic cache expiration** | Automatically clear key material in memory in 5 minutes |

#### Upgrade path to Agent security boundary

In Agentic Economy, security threats have been upgraded from "users being phished" to "Agents being deceived/hijacked/abused". Yallet's security architecture needs to be expanded from "protecting humans from making mistakes" to "constraining agents from crossing boundaries."

**Need to add**:

1. **Agent Behavior Breaker (Circuit Breaker)**

The Agent automatically stops when it triggers an abnormal mode within a short period of time:

```typescript
interface CircuitBreakerConfig {
maxTxPerMinute: number; //Maximum number of transactions per minute (anti-batch malicious)
maxFailedTxPerHour: number; // Maximum number of failed transactions per hour (anti-retry attacks)
maxValuePerHour: number; // Maximum transfer amount per hour
suspiciousPatterns: RegExp[]; // Suspicious contract method signature
autoLockDuration: number; // Automatic locking duration after circuit breaker (ms)
}
```

This reuses the existing security alert infrastructure - when an Agent triggers a circuit breaker condition, a `critical` level alert is generated and the Agent's execution permission is automatically suspended.

2. **Agent Sandbox Isolation**

Each Agent's operations are performed in an independent context:
- Independent strategy check
- Independent budget tracking
- Independent signing key (derived via BIP44 subpath)
- Mutually invisible communication channels

Chrome Extension's `chrome.storage` namespace naturally supports isolating data by key prefix (such as `agent_001_policy`, `agent_001_budget`).

3. **Human Emergency Control (Kill Switch)**

```typescript
// Freeze all Agents with one click
async function emergencyFreeze(): Promise<void> {
  await chrome.storage.local.set({ AGENT_GLOBAL_FREEZE: true });
// Clear all pending agent tasks
  chrome.alarms.clearAll();
// Push notification to mobile terminal
  await pushNotification('EMERGENCY_FREEZE', { timestamp: Date.now() });
}
```

**Feasibility Assessment**: **High**. Yallet's existing security infrastructure (alarm system + permission management + signature isolation) provides the core components of the Agent security boundary. Circuit breakers and sandboxes are incremental features that do not require modifications to the existing security architecture.

### 2.6 Audit layer: from "transaction history" to "Agent behavior audit"

#### Already capable

- **Transaction History**: filter by chain/status, block explorer link
- **Security Alarm History**: hierarchical alarm records
- **RWA Asset Log**: Crypto asset minting/synchronization records
- **Arweave immutable storage**: permanent storage of encrypted data
- **Notification System**: Background polling + browser notifications

#### Upgrade path

Agent behavior audit = existing transaction history + strategic decision log + negotiation record. Storage infrastructure for all three is in place:

- **Local audit**: `chrome.storage.local` + IndexedDB (large data volume such as negotiation records)
- **On-chain audit**: Transaction hash naturally exists in the blockchain
- **Immutable Auditing**: Arweave (Integrated for RWA data)
- **Visualization**: Add "Agent Activity" tab to 22 UI tabs

**Feasibility Assessment**: **High**. The audit layer is the easiest to implement because it is essentially "record + display" and all the underlying storage and UI frameworks are already in place.

---

## 3. Architecture Adaptability Analysis

### 3.1 Advantages of Chrome Extension Architecture

As a Chrome Extension rather than a mobile app or desktop application, Yallet has unique advantages in DST scenarios:

1. **Symbiosis with the browser**: Agent needs to interact with dApp (WalletConnect, EIP-1193 injection), Chrome Extension is the only form that can directly inject Web3 Provider into the web page context.

2. **Background resident**: `background.ts` (Service Worker) persists while the browser is running, and can run the Agent scheduler, listen to events, and execute scheduled tasks.

3. **Isolation security model**: Chrome Extension has an independent execution context (content script, background script, popup), which naturally provides the basis for Agent sandbox isolation.

4. **Rich storage layer**: `chrome.storage.local` (persistence), `chrome.storage.session` (session level), IndexedDB (big data), WASM memory (key operation) - covering all the storage needs of Agent.

5. **Cross-platform**: Chrome Extension can run on Windows, macOS, Linux, ChromeOS, and can be developed to cover multiple platforms at once.

### 3.2 Limitations of Chrome Extension architecture and complementation with mobile terminals

Chrome Extension has some inherent limitations, but the Yallet mobile terminal (Flutter App, `dev.yallet.mobile-app`) has implemented a complete complementary solution:

| Extension restrictions | Mobile terminal already has capabilities | Complementary effects |
|------|------|---------|
| **Service Worker may be killed** | Flutter App background resident + FCM push wake-up | Double-ended keep-alive, no execution interruption |
| **No native push** | Firebase FCM has been fully integrated (including iOS APNS), supporting foreground/backend message processing and deep linking | Agent approval requests are pushed to mobile phones in real time |
| **No biometrics** | `local_auth` Integrated fingerprint/face recognition, BiometricSetupScreen complete process | High value operation biometric confirmation |
| **Computing power is limited** | ACEGF FFI native library (non-WASM), Ed25519 signature performance is higher | The mobile terminal can bear heavier encryption operations |
| **Manifest V3 limitations** | Flutter has no such limitation and can dynamically load policies | Complex policy updates are distributed through the mobile terminal |

### 3.3 Dual terminal architecture (Extension + Mobile) - implemented

The complete form of DST is a dual-terminal architecture of **extension + mobile terminal**. **Both ends are close to production ready**:

```
┌─────────────────────────────────┐
│ Chrome Extension (Yallet) │ ← TypeScript, 22 function modules
│  ┌─────────────────────────────┐ │
│ │ Agent policy engine + automatic execution │ │ ← 99% of operations are automatically completed here
│ │ Transaction Signature Engine (WASM) │ │
│ │ E2EE Communication + Negotiation │ │
│ │ dApp Integration (EIP-1193/WC) │ │
│ │ Agent behavior monitoring + audit │ │
│  └─────────────────────────────┘ │
└──────────────┬──────────────────┘
               │ ReviewRequest (FCM Push)
               ▼
┌─────────────────────────────────┐
│ Mobile App (Yallet Flutter) │ ← Dart, 254 files, BLoC architecture
│ ┌─────────────────────────────┐ │ Development completion 80-90%
│ │ ✅ Firebase FCM Push Notifications │ │ ← Approval requests delivered in real time
│ │ ✅ Biometric authentication (fingerprint/face) │ │ ← Secondary confirmation of high-value operations
│ │ ✅ Ed25519 Signature (ACEGF FFI) │ │ ← Native Signature Engine
│ │ ✅ RWA Crypto Asset Management │ │ ← Notes/Invoices/Photos/Files
│ │ ✅ NFT Minting (Arweave) │ │ ← Compatible with Extension format
│ │ ✅ QR scan code payment │ │ ← Offline payment scenario
│ │ ✅ Drift local database │ │ ← 8 data entities complete CRUD
│ │ ✅ Gmail OAuth email integration │ │ ← Automatically encrypted storage of attachments
│  └─────────────────────────────┘ │
└─────────────────────────────────┘
```

**Key facts**: The mobile version is not a "future plan" - it is a Flutter application with **254 Dart files, 76 dependency packages, and a complete BLoC architecture**, which has been implemented:

| Mobile capabilities | Implementation status | DST role |
|-----------|---------|---------|
| Wallet authentication (mnemonic/password/Passkey) | ✅ Implemented | Agent owner authentication |
| Biometrics (fingerprint/face) | ✅ Implemented | Secondary confirmation of high-value approvals |
| FCM push notification | ✅ Implemented (including deep linking) | Agent ReviewRequest real-time delivery |
| Ed25519 signature (ACEGF FFI) | ✅ Implemented | Mobile transaction signature |
| RWA crypto assets | ✅ Implemented (compatible with Extension format) | Agent work output viewing/management |
| NFT Minting | ✅ Fulfilled | Agent Commitment/Receipt Minting |
| Payment QR code | ✅ Implemented | Offline Agent payment scenario |
| Secure Storage (Keychain/AES-GCM) | ✅ Implemented | Key Material Protection |
| Subscription Management (RevenueCat) | ✅ Implemented | Agent Service Subscription |

The **ReviewRequest protocol** has been designed in `PERSONAL_AGENT_HIERARCHY_AND_MOBILE.md`:

```typescript
interface ReviewRequest {
  requestId: string;
  agentId: string;
  agentLabel: string;
  action: 'transfer' | 'sign' | 'approve' | 'negotiate';
summary: string; // human readable summary
details: object; // complete parameters
  urgency: 'low' | 'normal' | 'high' | 'critical';
deadline: string; // ISO timestamp, automatically rejected after timeout
escalatedFrom?: string; // Which sub-Agent it was upgraded from
}
```

---

## 4. Protocol compatibility analysis

### 4.1 Compatibility with MCP (Model Context Protocol)

MCP defines how Agent uses tools (Tools), access resources (Resources), and obtains prompts (Prompts).

**Yallet’s role as MCP Host/Client**:

```
MCP Server (Tool Provider) Yallet (MCP Client/Host)
┌────────────────────┐          ┌────────────────────┐
│ DEX Aggregator API │ ←Tool── │ Agent calls swap() │
│ NFT Market API │ ←Tool── │ Agent calls buy_nft()│
│ On-chain data query API │ ←Res─── │ Agent reads balance │
└────────────────────┘          └────────────────────┘
```

**Compatibility Path**: Yallet's `api-client.ts` already encapsulates all external API calls (Jupiter, 1inch, LI.FI, Helius, etc.). By exposing these encapsulations as MCP Tool definitions (JSON Schema), Yallet becomes an MCP Host - managing Agent's access to external tools, while controlling call frequency and budget through the policy engine.

### 4.2 Compatibility with A2A (Agent-to-Agent)

A2A defines how Agents discover each other (Agent Cards), assign tasks (Tasks), and exchange messages (Messages).

**A2A Adaptation for Yallet**:

- **Agent Card**: generated based on xidentity + policy summary, published through `/.well-known/agent.json`
- **Task Receiving**: Receive task requests from other Agents through Yallet’s notification system (FCM push + polling already exists)
- **Message exchange**: Yallet's E2EE communication channel naturally adapts to A2A's Message exchange requirements - and is more secure than A2A's native specification (A2A does not force E2EE)

### 4.3 Compatibility with AP2/x402

AP2 (Agent Payments Protocol) and x402 (HTTP 402 micropayment) are Agent payment protocols.

**Yallet's role**: Yallet is not a competitor to AP2 - Yallet is the **signature terminal** for AP2 transactions. When the Agent initiates a payment intention through AP2, the final on-chain signature requires a wallet to complete. Yallet is that wallet, providing both policy checking (whether the amount is within budget) and audit trail.

---

## 5. Competitive Landscape Analysis

### 5.1 Comparison of Agentic capabilities of existing wallets

| Capabilities | MetaMask | Phantom | Trust Wallet | Coinbase Wallet | **Yallet** |
|------|----------|---------|-------------|----------------|---------|
| Multi-chain support | EVM only | Sol+EVM | Multi-chain | EVM+Sol | **Sol+10 EVM+BTC** |
| Dual terminal (Extension+Mobile) | ✅ (Extension+Mobile) | ✅ (Extension+Mobile) | Mobile only | ✅ (Extension+Mobile) | **✅ Extension + Flutter App** |
| E2EE Messages | ❌ | ❌ | ❌ | ❌ | **✅ X25519+AES-GCM** |
| Cryptoasset Storage | ❌ | ❌ | ❌ | ❌ | **✅ cNFT+Arweave** |
| Strategy Engine | ❌ | ❌ | ❌ | ❌ | **✅ (Design Complete)** |
| Agent Identity Derivation | ❌ | ❌ | ❌ | ❌ | **✅ BIP44 Subpath** |
| Structured Signatures | EIP-712 | ❌ | Limited | EIP-712 | **✅ EIP-712** |
| Immutable Auditing | ❌ | ❌ | ❌ | ❌ | **✅ Arweave** |
| Biometric Approval | ❌ | ❌ | Limited | ❌ | **✅ Fingerprint + Face (Flutter)** |
| FCM Push Notifications | ❌ | ❌ | Basics | Basics | **✅ With Deep Links** |
| dApp Integration | ✅ | ✅ | ✅ | ✅ | **✅ EIP-1193+WC** |
| Open Source | Part | ❌ | ❌ | Part | **Fully Own** |

**Key differences**: Although MetaMask/Phantom also has dual terminals, their mobile terminals are only copies of the Extension function, without E2EE communication, no encrypted storage, no policy engine, and no Agent identity management. **Yallet is the only wallet that has the five-layer capabilities of "signature + communication + storage + policy + push approval", and both ends share the ACEGF cryptography infrastructure and RWA data format. **

### 5.2 Potential competitors

1. **Coinbase Wallet + AP2**: Coinbase is pushing the AP2 payment protocol and may upgrade Coinbase Wallet to an Agent payment terminal. But Coinbase Wallet is custodial/semi-custodial and lacks Yallet’s non-custodial security model and E2EE capabilities.

2. **MetaMask Snaps**: MetaMask’s plug-in system can theoretically expand Agent capabilities. But Snaps is an additional layer on top of MetaMask, not a native architecture; and MetaMask only supports EVM.

3. **Specialized Agent Framework** (such as LangChain, AutoGPT): These are AI frameworks, not wallets. They can schedule Agent logic, but cannot execute on-chain signatures - ultimately a wallet is still required as the execution terminal. **Yallet does not compete with AI frameworks but complements them. **

### 5.3 The irreplaceability of Yallet

In the DST role, Yallet’s core irreplaceable capabilities are:

1. **The key never leaves the client**: All signatures are completed in WASM, and the private key does not touch the network. This is the foundation of Agent's economic security - if the key is on the server side, Agent is compromised = funds are stolen.

2. **E2EE Negotiation Channel**: The content of negotiations between agents is not visible to the server. When AI Agents process sensitive business information (prices, contract terms, medical data), E2EE is not an "optional plus" but a necessity for compliance.

3. **cNFT encrypted assets**: Agent’s work results (reports, invoices, contracts) are stored in encrypted cNFT form and can only be decrypted by authorization. This creates an "encrypted archive of Agent work output".

---

## 6. Implementation Roadmap

### Phase 1: Agent Strategy Engine MVP (4-6 weeks)
- Add policy checking layer in `background.ts`
- Implement basic policy rules (amount limit, whitelist, time window)
- Budget tracker (daily/weekly/monthly sliding window)
- UI: Add "Agent Policies" configuration panel to Settings page
- **can be demonstrated**: an Agent that automatically performs DCA (regular purchase), automatically signs within the budget, and pops up a window for approval when the budget is exceeded.

### Phase 2: Agent identity + negotiation channel (4-6 weeks)
- BIP44 subpath Agent identity derivation
- Agent identity certificate (cNFT or Arweave storage)
- Negotiation message type extension (based on existing RWA encrypted channel)
- Negotiation state machine
- **can be demonstrated**: Agents in two Yallet instances automatically negotiate and reach consensus through the E2EE channel

### Phase 3: Agent level + mobile approval integration (4-6 weeks)
- Agent hierarchical management (parent/child Agent authority delegation)
- ReviewRequest protocol implementation
- Mobile approval interface (Flutter App already has FCM + biometrics foundation, and adds a ReviewRequest page)
- Extension → Mobile approval link opened
- **Can be demonstrated**: Shopping Agent exceeds budget → FCM pushed to mobile phone → Biometric confirmation → One-click approval

### Phase 4: MCP/A2A Protocol Bridging (4-6 weeks)
- Yallet tool API exposed as MCP Tool definition
- Agent Card generation and release
- A2A Task reception and distribution
- **Can be demonstrated**: External AI Agent discovers Yallet's swap tool through MCP, and requests Yallet Agent to perform swap through A2A

**Total Time Estimate**: 16-24 weeks (calculated from launch of Yallet MVP). Phase 3 is shortened by 2 weeks because the mobile version already has FCM + biometrics foundation.

---

## 7. Risks and Mitigation

| Risk | Level | Mitigation |
|------|------|---------|
| **Chrome Extension was removed from the shelves by Google** | Medium | Simultaneous development of Firefox Extension; core logic is in WASM, portable |
| **Service Worker unstable** | Low | chrome.alarms keep-alive + state persistence; key operations idempotent design |
| **AI model security** (Agent is injected with malicious instructions) | High | The policy engine is independent of the AI ​​model at the wallet layer; even if the AI ​​is compromised, the policy restrictions still take effect |
| **High user awareness threshold** | High | Progressive guidance: first do simple Agent (DCA, gas optimization), and then open to complex scenarios |
| **Protocol standard fragmentation** (MCP/A2A/AP2 continues to evolve) | Medium | Using the Adapter Pattern, protocol changes only require modification of the adaptation layer |
| **Privacy Compliance** (GDPR/CCPA) | Medium | E2EE natural privacy protection; Agent behavior logs can be stored locally and not uploaded to the server |

---

## 8. Conclusion

### Feasibility Rating: ⬛⬛⬛⬛⬛ (5/5 — Very High Feasibility)

**Core arguments**:

1. **Complete technical foundation**: Identity (BIP44 + xidentity), communication (E2EE + Arweave), execution (multi-chain signature engine), security (alarm + permission + simulation) - Yallet has basic implementation of the four major technical layers required by DST.

2. **Incremental upgrade rather than refactoring**: Each step of the upgrade from "Wallet" to "DST" is to add modules to the existing architecture without rewriting the core code. The policy engine is an extension of `background.ts`, Agent Negotiation is an extension of RWA communication, and Agent Identity is a BIP44-derived extension.

3. **Unique competitive barriers**: The combination of E2EE messaging + cNFT encrypted storage + non-custodial signatures is unique in the market. Competing products (MetaMask, Phantom) to catch up with this combination will require building the E2EE and RWA subsystems from scratch.

4. **Accurate ecological niche**: Yallet does not try to make an AI model, does not try to make an Agent orchestration framework, and does not try to make a payment protocol - it is positioned as a "terminal for humans to exercise digital sovereignty" and complements rather than competes with the MCP/A2A/AP2/AI framework.

5. **Dual-terminal architecture is ready**: Chrome Extension (TypeScript, 22 functional modules) + Flutter Mobile App (Dart, 254 files, BLoC architecture, development completion 80-90%). The mobile terminal has implemented Firebase FCM push, biometric authentication, ACEGF FFI native signature, RWA encrypted asset management - **DST dual-terminal architecture is not a plan, but a realized capability**. Extension's Service Worker limitations are fully compensated by the mobile terminal's background resident + FCM push; the biometrics that Extension lacks are supplemented by the mobile terminal's `local_auth`. Both ends share the same cryptographic infrastructure (ACEGF) and RWA data format to ensure a consistent user experience.

---

*Generation date: 2026-02-20*
*Comprehensive feasibility assessment based on Yallet Chrome Extension code base + Yallet Flutter Mobile App code base + 13 agentics documents + Yault platform analysis*
