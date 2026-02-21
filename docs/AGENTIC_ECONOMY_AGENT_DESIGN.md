# Autonomous Agent design refinement in AI Agentic economy

Based on Yallet's existing capabilities (E2EE communication + asset holding + signature verification), three specific agent forms and their access methods to the existing architecture are refined.

---

## 1. Quick overview of existing Yallet capabilities (reusable components)

| Capabilities | Existing implementation locations | Usage |
|------|--------------|------|
| **E2EE Communication** | `api-client.ts` (encryptedFetch,
| **Asset Holding** | Multi-chain address (Solana/EVM/BTC, etc.), balance, Transfer/Swap/Stake | Currency holding, transfer, on-chain transactions |
| **Signature/Verification** | `background.ts` → `requestApproval` → `approve.html` → WASM (Solana/EVM signature); `handleSendTransaction` / `handlePersonalSign` / `handleSignTypedData` | dApp sends transaction/signature request → user confirmation → returns signature or sent transaction |
| **RWA on-chain certificate** | `rwa/mint-queue.ts`, `encryptAndMintAsset`, cNFT | Encrypted Note/Invoice/File/Photo on the chain, can bring recipient, recipient can decrypt |

Agent design is to add a **strategy layer** of "who has the right to execute on behalf of the user under what conditions" on top of the above capabilities, as well as an optional "reasoning/planning" layer (this solution does not assume that reasoning must be within the plug-in).

---

## 2. Three specific Agent forms

### 2.1 Auto-Payment

**Meaning**: When the user's preset conditions are met, the wallet **automatically** completes the transfer or authorized payment without pop-up windows every time.

**Typical scenario**:
- Recurring payments: Transfer Y USDC to address A on X day every month.
- Conditional payment: automatic payment when "invoice amount ≤ Z and the other party's address is in the whitelist".
- Threshold/Budget: Automatically pass when single transaction ≤ N, daily accumulation ≤ M, otherwise manual confirmation will still be required.

**Interconnection with existing architecture**:
- **Policy Storage**: Save "automatic payment rules" in `chrome.storage.local` or backend (if cross-device is required), for example:
  - `{ type: 'recurring', toAddress, amount, token, cron, maxPerMonth }`
  - `{ type: 'invoice_auto', conditions: { maxAmount, allowList }, scope: 'per_contact' | 'global' }`
- **Execution path**: not through dApp, but **background or offscreen** internal timing/event driven:
- Arrive at the point or receive the "invoice to be paid" event → check the rules → if it is hit and does not exceed the limit, **directly call the existing transfer logic** (same as Transfer: getDecryptedMnemonic → sign → send).
- **Key Point**: Today’s transfer must go through `requestApproval` (pop-up window). To be "automatic", you need to add a **pop-up-free path**, and it can only be taken when "the current operation matches an enabled strategy":
- Option A: Implement `executeTransferIfPolicyAllows(payload, policyId)` in the background. After internally verifying that the payload is consistent with the policy and has not exceeded the limit, directly adjust the signature + sending logic from the same source as `handleSendTransaction` (without going through requestApproval).
- Option B: Keep "Issue approval every time", but let another agent (local or server) click "Approve" on behalf of the user - technically feasible, but the experience is poor and not recommended.
- **Audit**: Each automatic payment writes a local/on-chain checkable record (such as RWA Note or local log only) to facilitate user review and dispute handling.

**Summary**: Automatic payment = **Policy engine (rules + limits)** + **Existing transfer and signature capabilities**, new "pop-up-free execution path when policy allows" and auditing.

---

### 2.2 Delegated Negotiation

**Meaning**: The user authorizes the agent to conduct multiple rounds of E2EE negotiations (price, terms, delivery conditions, etc.) with the other party (person or agent) within a given range, and when an agreement is reached, the **same wallet** signs the commitment or payment.

**Typical scenario**:
- The user sets "purchase a certain service with a budget of 500 USDC, and the terms are negotiable"; after multiple rounds of E2EE messages between the agent and the other agent, the agent reaches "400 USDC + 30 days SLA" and generates an **off-chain commitment** (signatures from both parties) or **on-chain commitment** (smart contract/signature message).
- Invoice negotiation: When the other party sends an encrypted invoice, the agent automatically replies "accept" or "counteroffer"; when accepted, it automatically pays (return to 2.1) or generates a commitment.

**Interconnection with existing architecture**:
- **E2EE Channel**: Already have RWA Notes (encrypted + optional recipient) + on-chain cNFT or backend push. Reusable:
- Treat "negotiation dialogue" as a Note or special message type, `recipientXidentity` = the xidentity of the other party (or the other party's agent); the content is structured JSON (such as `{ type: 'offer', amount, terms, nonce }`).
- If the negotiation is between "user ↔ counterparty", you can continue to use the existing Notes to send and receive; if it is "user agent ↔ counterparty agent", you need to assign an identity to the agent: **The same xidentity in the same wallet** is enough, the agent is just a process that "automatically sends/receives messages under this identity".
- **Signed Commitment**: A "commitment" is required at the end of the negotiation that can be verified:
- **Off-chain**: Both parties do `personal_sign` or EIP-712 for the same payload (such as `{ agreementId, amount, terms, timestamp }`); Yallet side uses **requestApproval** or **policy to allow time signing** (the same as the pop-up-free path of 2.1, but here it is sign instead of sendTransaction).
- **On-chain**: If a smart contract is used to store a "commitment", the agent calls `handleSendTransaction` to send a transaction that calls the contract if allowed by the policy.
- **Who runs the "negotiation logic"**: Multiple rounds of if-then, price comparison, and counter-offer can be executed by the cloud or local agent services; Yallet is only responsible for: **(1) using the user wallet identity to send and receive E2EE messages, (2) signing the commitment/transaction as allowed by the user or policy**. That is, Yallet = identity + communication endpoint + pen, the brain can be external.

**Summary**: Entrusted negotiation = **E2EE message (Notes/Extended Agreement)** + **Structured negotiation payload** + **Sign/send ability when committing**; the policy can stipulate "automatically sign only when the amount/the other party is within the allowed range".

---

### 2.3 On-Chain Commitment

**Meaning**: The user or agent signs "a certain statement" as a wallet, so that anyone can verify "the address promised something at a certain moment", which is used for credit, SLA, guarantee, etc.

**Typical scenario**:
- Commitment to pay: Sign `{ "I will pay", amount, to, deadline }`; the other party or the contract can verify the signature and then perform the contract first, and then demand payment when it is due.
- Commitment to deliver: Sign "deliver something before T", and store the certificate on the chain or off the chain.
- Authorized agent: Sign `{ "I authorize agent to spend up to X until date D" }`. The agent can sign on behalf of D before D and within X (combined with 2.1).

**Interconnection with existing architecture**:
- **Existing capabilities**: `handlePersonalSign`, `handleSignTypedData` (EIP-712) already support "signing any message/structured data". On-chain commitment = make these signature results **public** (send to the other party, write on-chain or IPFS), without sending on-chain transactions (unless the commitment itself needs to be on-chain).
- **Structured Format**: It is recommended to unify a "commitment" schema (such as EIP-712 domain + type) to facilitate parsing by the verification party. For example:
  - `type: 'YalletCommitment', payload: { commitmentType, amount?, to?, deadline?, nonce }`
- **Agent usage**: When the agent decides to "make a commitment on behalf of the user", it calls the same `personal_sign` / `eth_signTypedData_v4` path as the dApp; if the user has been pre-authorized (policy), it takes the **pop-up-free signature path** (same as 2.1); otherwise, the requestApproval window still pops up.
- **Signature Verification**: The signature verification party uses the public key corresponding to the address on the chain to verify the signature; the Yallet side can provide a UI or API (read-only) to "verify whether a certain commitment is issued by this wallet".

**Summary**: On-chain commitment = **Existing personal_sign/EIP-712** + **Unified commitment schema** + **Signature path under the strategy** + **Signature verification and certificate storage**.

---

## 3. Access method with existing Yallet architecture (unification)

### 3.1 Policy layer (Policy)

- **Location**: It can be placed in the background of the extension or in a separate "agent policy" module; if cross-device/backup is required, it can be encrypted and stored in the backend.
- **Content**: Each strategy contains:
  - **scope**：auto_payment | delegated_negotiation | commitment；
- **conditions**: amount limit, whitelist address, time window, counterparty xidentity, etc.;
- **allow_auto_approve**: When true, when the request completely matches the policy and does not exceed the limit, it can be executed directly (signature or transfer) without pop-up window**.
- **Verification**: Before performing any "pop-up free" operation, you must: **(1) find a policy with matching scope and allow_auto_approve; (2) payload satisfies conditions; (3) does not exceed the policy or global limit/number of times**. Otherwise fallback to requestApproval (or reject).

### 3.2 Pop-up-free execution path (Agent-Only Execution)

- **Current situation**: All `handleSendTransaction` / `handlePersonalSign` / `handleSignTypedData` etc. are executed after being approved by `requestApproval` → user click.
- **Extensions**:
- Before calling `requestApproval`, check the policy first: if there is a matching policy and automatic execution is allowed, then **not** open the approve window, directly:
- Get mnemonic (through the existing getDecryptedMnemonic / Passkey process; if the agent is running in a session that has been unlocked by the user, the cache can be reused),
- Calls the same signing/sending logic as approve (like existing EVM/Solana signing and broadcasting),
-Write audit log.
- If there is no matching strategy or the conditions are not met, the original requestApproval process remains unchanged.
- **Security**: The addition, deletion and modification of the policy must be explicitly confirmed by the user (such as a pop-up window when "adding automatic payment rules"); the policy itself can be set to "validity period" or "single authorization".

### 3.3 Reuse of E2EE and RWA

- **Send encrypted message**: The existing `queueRWAMint` + `recipientXidentity` already supports "encrypted Note for specified xidentity". The "offer/counteroffer" in entrustment negotiation can be modeled as metadata of a dedicated Note type or extended RWA, and the sending path remains unchanged.
- **Receive encrypted messages**: The existing sync + decryption process is supported; the agent side only needs to parse the content by the background logic when "receiving a new note/notification". If it is a negotiation message, the next round of replies will be triggered (the replies are still sent through the same set of RWA).
- **Identity**: Agent and user share the same wallet (same xidentity, same address), and no separate "agent sub-identity" is issued; the difference is only in "who triggered the action" (user manual vs. policy automatic).

### 3.4 Auditing and Observability

- For all automatic payments and signing commitments triggered by strategies, it is recommended to:
- Record locally (such as IndexedDB or chrome.storage): time, policy ID, payload summary, on-chain tx hash (if any);
- Optional: Write a "visible only to you" Note in the RWA as evidence to facilitate future reconciliations or disputes with the other party.

---

## 4. Implementation sequence suggestions

1. **Policy data model + policy verification function** (no execution, only judgment of "whether automatic execution is allowed").
2. **Automatic payment**: Implement "transfer path when policy allows" in the background + share signature/send logic with existing Transfer, first support the simplest rules (such as single address + amount limit + period).
3. **On-chain commitment**: Define the Yallet commitment schema (EIP-712), support the "sign only, do not send transactions" commitment flow in the approve or policy path, and provide a signature verification entrance.
4. **Delegated Negotiation**: Expand the "Negotiation Type" message and parsing on RWA Notes; the external or local agent is responsible for multiple rounds of logic, and Yallet is responsible for sending and receiving E2EE and signing/paying when an agreement is reached.

---

## 5. Relationship with “Running AI in Plugins”

- **Inference/Planning** can be a small model in the cloud, locally or in the plug-in; **Yallet plug-in** mainly provides:
- **Strategy configuration and verification**,
- **Pop-up-free execution when policy allows**,
- **E2EE sending and receiving and signature verification**,
- **Audit log**.
- In this way, Yallet as the "agent's economic and identity carrier" remains unchanged: currency holding, E2EE, and signature verification are all in the wallet; "whether and when to execute" are jointly decided by the policy + external or local AI, and execution is completed by the wallet with the permission of the policy.

---

*Document version: Summarized based on the existing Yallet code structure (background approval flow, RWA, api-client E2E, passkey/WASM signature); the specific API naming and storage schema can be determined during implementation. *
