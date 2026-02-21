# Yallet Agent Framework — Pure background scene design

The previous article "AGENT_FRAMEWORK_INTERFACES.md" uses **browser extension** as the runtime; this article specifically discusses the Agent framework under **pure background** (server, no browser). The pure backend is indeed much richer than "only in the plug-in" in terms of computing power, integration surface, persistence and scale. The interface concept can be reused, and the runtime and key model are different.

---

## 1. Plug-in vs pure backend: an overview of the differences

| Dimension | Browser plug-in | Pure backend |
|------|------------|--------|
| **Operating environment** | Extensions on the user device (Chrome/Edge), start and stop according to the browser/user behavior | Server/cloud function/container, 7×24 resident, can be multi-instance and horizontally scalable |
| **Key location** | The key is in the plug-in and is cached in the memory after the user unlocks it; the plug-in needs to be unlocked again | The key can be "entrusted to the background" or "never in the background": delegated type, remote signature type, HSM/MPC, etc. |
| **Pop-up window/manual confirmation** | Natural pop-up window; no pop-up window when policy allows | No UI, either policy/authorization is given in advance, or "notify user to App/Extension confirmation" |
| **Computing power and integration** | Limited to a single machine, single tab/background, cannot occupy the CPU for a long time | Can run heavy inference, large models, queues, DB, external API, multi-chain index, etc. |
| **Network and Storage** | Restricted by host_permissions and storage | Any network access, any DB/cache/message queue |
| **Who deploys** | Users install extensions; manufacturers can make additional extensions or connect extensions through SDK | Manufacturers build their own services or use the "Agent hosting backend" provided by Yallet |

Therefore: **Pure backend is more suitable for "Agent brain"** (multiple rounds of reasoning, scheduling, and integration with the off-chain world), while **Plug-ins are more suitable for "user-side keys and instant confirmation"**. The two can cooperate: the background runs logic, and when signature/transfer is required, it is completed through "remote signature" or "entrusted key".

---

## 2. Two key models of pure backend (determining what the “framework” looks like)

### 2.1 Delegation (Key Delegation)

- **Meaning**: The user entrusts the "signature right" to the backend within a certain period of time; the backend can sign/transfer on behalf of the user during the delegation period without asking the user every time.
- **Examples of implementation**:
- The user clicks "Authorize Agent Service for 24 Hours" in the extension/App → the extension sends the **encrypted key material** (or short-term token) to the manufacturer's backend; the backend can use it to sign within 24 hours.
- Or: The key material is encrypted by the user and stored in "Yallet Escrow" or vendor custody. The backend applies for a signature to the custody service through a rights-bearing token. The custody service verifies the policy and executes the signature and returns the result. The key does not leave custody.
- **Features**: The backend can actively initiate transactions/signatures with low latency, suitable for "automatic payments, scheduled tasks, and instant commitments in multiple rounds of negotiations"; however, there is a risk that "if the backend is compromised, it can be abused during the commission period", which requires strict strategies and audits.

### 2.2 Remote Signing / Wallet as a Service

- **Meaning**: **The key is never in the background**; when the background wants to sign, a signature request is initiated to the "user's wallet (extension/App)" or "Yallet official signature service". After the user (or user's default policy) approves it on the client side, the signature is completed on the client side, and only the result is sent back to the background.
- **Examples of implementation**:
- Call the "Yallet Signing API" in the background: send the payload to be signed to the API; the API sends the request to the user extension or app through push/long connection. After the user clicks approval, the extension signs and uploads the result, and the API returns to the background.
- Or: The "Yallet extension" installed by the user has a corresponding bridge service in the background. The background communicates with the bridge, and the bridge communicates with the extension. The signature is always completed within the extension.
- **Features**: The key can never be obtained in the background, which is more secure; however, each signature has the delay and dependence of "end-side confirmation" (user is online, extension is opened, etc.), which is suitable for "non-high-frequency, acceptable delays of a few seconds to minutes" scenarios.

**summary**:
- **Delegation type** → The background "self" can sign, the framework interface is similar to the plug-in version, except that the policy and execution are run in the background, and the key/signature module is in the background or managed service.
- **Remote signature type** → The background only initiates a "signature request" and is actually executed on the client side (or Yallet signature service). The framework needs an additional layer of abstraction of "request → client side → result callback".

The following describes how to design the framework and how to reuse/extend the interface according to "delegation type" and "remote signature type" respectively.

---

## 3. Interface reuse and expansion of pure background framework

The **concepts** defined above still hold true in the background and can be directly reused or slightly expanded:

- **Policy**: `IAgentPolicyProvider`, `AgentPolicy`, `PolicyConditions` - unchanged; only the implementation is "backend service" or "backend + database".
- **Execution requests**: `AgentExecutionRequest`, `ExecutionAction` (transfer / sign_personal / sign_typed_data / send_transaction) - unchanged; the background constructs and calls the execution layer when needed.
- **Auditing and Quotas**: `IAgentAuditReader`, `recordExecution`, `getUsageToday` — unchanged; implementation falls on backend DB/storage.
- **E2EE messages**: `AgentMessage`, `sendAgentMessage`, handlers distributed by `type` - unchanged; the background can send and receive encrypted messages (using the same set of X25519/Ed25519), and the other party can be the client, the background of other manufacturers, or on-chain/off-chain services.

**Points that need to be added or enhanced** (purely unique to the backend):

1. **Key/Signature Abstraction**
- Delegation type: There is a "signature module" interface inside the framework, which is injected by the manufacturer or implemented by Yallet (for example, retrieving the delegation key from the secure storage and calling the WASM/library signature that has the same origin as the plug-in).
- Remote signature type: "Signature module" inside the framework = calls the "remote signature API" and polls/long-lives the results, without directly holding the key.

2. **Identity and E2EE**
- The background still represents the "user wallet identity" (same xidentity, same address on the chain).
- E2EE key: Under the entrusted mode, the backend can hold the entrusted x25519/ed25519 or derived materials; under the remote signature mode, the encryption and decryption of E2EE can be completed by the client side, and the backend only forwards the ciphertext or "decrypted payload" provided by the client side and then used by the business logic (depending on the product selection).

3. **Events and Triggers**
- The background can actively poll, subscribe to on-chain events, consume message queues, receive Webhooks, and run cron. Therefore, "when to initiate Execution and when to send E2EE messages" is much richer than the plug-in; the framework only needs to define "execution entrance" and "message entrance", and the event source is accessed by the manufacturer itself.

4. **Multi-tenancy and multi-wallet**
- A backend instance may serve multiple users (or multiple policies, multiple vendors); the framework must support "isolation by walletId / userId / vendorId" policies, audits, and limits.

The following uses a block diagram + interface each of "delegation type" and "remote signature type" to supplement the explanation.

---

## 4. Delegated background framework (keys are in the background or hosted)

```
Manufacturer's backend (your service)
├──Business logic (negotiation, pricing, scheduled tasks, API)
├── Implement IAgentPolicyProvider (read policy from DB/Configuration)
├── Implement/register IAgentMessageHandler (handle negotiation, invoice, etc.)
└── Call AgentFramework.requestExecution / sendAgentMessage
                    │
                    ▼
Yallet Agent framework (Node/Go SDK or built-in library)
├── Policy engine: getPolicies → checkAutoApprove → recordExecution
├── Execution layer: transfer / sign_personal / sign_typed_data / send_tx
│ (Use delegation key internally or call "Yallet managed signature service")
├── E2EE: encryption and decryption, sendAgentMessage, distribution of MessageHandler by type
└── Audit/limit storage (DB or you provide IAgentAuditStore)
                    │
                    ▼
Key and on-chain
├── Delegated key (encrypted DB or HSM) or
└──Yallet managed signature service (only session token is taken in the background and private key is not accessed)
```

- **Interface**: consistent with the plug-in version, `requestExecution`, `sendAgentMessage`, `IAgentPolicyProvider`, `IAgentMessageHandler`; an additional "signature implementation" injection point (such as `ISigningBackend`) is added, and the manufacturer or Yallet provides a delegated/managed implementation.
- **Strategy Source**: Completely controlled by the manufacturer's backend (DB, configuration center, management backend), and can be made more detailed than in the plug-in (such as by user, by contract, by amount segment, by counterparty, etc.).
- **Scenarios**: automatic payment, scheduled settlement, "the other party signs the commitment immediately upon acceptance" in multiple rounds of negotiations, batch issuance of on-chain commitments, etc., **low latency, does not rely on users being online**.

---

## 5. Remote signature background framework (the key is never in the background)

```
Manufacturer's backend
├──Business logic
  ├── IAgentPolicyProvider / IAgentMessageHandler
└── Call AgentFramework.requestExecution / sendAgentMessage
                    │
                    ▼
Yallet Agent Framework (SDK or Agent Gateway provided by Yallet)
├── Strategy engine (same as above)
├── Execution layer: not signing directly, but
│ 1. Construct AgentExecutionRequest
│ 2. Call IRemoteSigningClient.createSigningRequest(req)
│ 3. Wait/polling/Webhook to get the signature result or txHash
└── E2EE: If a private key is required for decryption, the "decryption request" is also sent remotely (the client side decrypts and returns plain text or only returns the necessary fields for the business)
                    │
                    ▼
Yallet client side (Extension/App/Official signature page)
├── Receive a request to be signed (push / long connection / polling)
├── Show to user (or silently approve if policy allows)
└── Upload the result after signing → The framework then calls back to the background
```

- **New interface**:
  - `IRemoteSigningClient.createSigningRequest(req): Promise<{ requestId, statusUrl? }>`  
- Poll `statusUrl` or receive Webhook in the background, and get `signedPayload` / `txHash` or `rejected`.
- **Policy**: Still using the same set of `AgentPolicy`; "Whether pop-up-free windows are allowed" is determined on the **device side** (the policy engine in the extension/App). The background only initiates a request, and the client side decides whether to sign directly or give pop-ups to the user.
- **Scenario**: High security requirements, key delegation cannot be accepted; processes that "complete the signature when the user opens the app/extension" or "complete within a few minutes" (such as asynchronous settlement, commitments requiring manual spot checks, etc.) are acceptable.

---

## 6. What is the "richer" feature of pure backend compared to plug-ins?

- **Residency and Scale**: Run multiple rounds of negotiations 7×24, schedule automatic payments, monitor on-chain/off-chain events and trigger execution, and are not affected by closing the browser.
- **Computing Power**: Large models, complex strategies, batch verification, multi-chain indexing and matching can all be done in the background.
- **Integration surface**: DB, queue, CRM, payment gateway, other chains, third-party APIs, and the backend can all be directly connected; plug-ins can only use limited APIs and storage.
- **Key model**: You can either "entrust" to exchange for low latency, or "remote signature" to exchange for zero-touch keys; the key in the plug-in can only be on the terminal side, and there are no two options.
- **Multi-tenant**: One backend serves multiple users/multi-wallets, and policies and audits are isolated by user/wallet; the plug-in is naturally the "current user".
- **Event-driven**: The background itself subscribes to events, queues, and cron to trigger execution and send messages, rather than relying solely on user operations or content scripts within the extension.

Therefore: **The same set of "policy + execution + message + audit" interfaces will appear "much richer in scenarios" when placed on a pure backend**; plug-ins are more suitable for "user-side keys and instant confirmation", and the backend is more suitable for "Agent brain" and "large-volume/resident/integrated" capabilities.

---

## 7. Unified abstraction: one set of interfaces, two runtimes

It can be abstracted in this way to facilitate manufacturers to reuse the same set of business logic as much as possible:

- **Core Interface** (defined previously):
  `IAgentPolicyProvider`、`AgentExecutionRequest`/`requestExecution`、`AgentMessage`/`sendAgentMessage`/`IAgentMessageHandler`、`IAgentAuditReader`。  
These are runtime independent and can be implemented once by the vendor and reused in different runtimes (see below).

- **Runtime Abstraction**:
  - `IExecutionRuntime`：  
    - `requestExecution(req): Promise<AgentExecutionResult>`  
- Plug-in implementation = strategy engine within the current extension + pop-up/free window + wallet core.
- Backend delegation implementation = backend policy engine + delegation key/escrow signature.
- Background remote signature implementation = background policy engine + create a remote signature request and wait for the result.
  - `IMessagingRuntime`：  
    - `sendAgentMessage(msg)`、`registerMessageHandler(handler)`  
- Plug-in implementation = existing RWA/Notes channel.
- Backend implementation = the same protocol over HTTP/WebSocket/queue, communicating with the client or other backends.

When manufacturers write "policy + message processing + when to initiate execution", they only rely on `IExecutionRuntime` and `IMessagingRuntime`; when deploying, select "plug-in runtime" or "background runtime" (delegated type/remote signature type), and you can reuse the same set of logic under "browser only" or "pure background" or "browser + background hybrid".

---

## 8. Suggested order of implementation (pure backend)

1. **Define and implement the "backend use" Execution + Messaging runtime** (choose one of the delegate type or the remote signature type first), so that `requestExecution` / `sendAgentMessage` is available in Node (or Go).
2. **Policy and Audit**: The same set of `AgentPolicy`, `IAgentPolicyProvider`, and audit structure, and DB is used in the background to store policies and audit records; optionally, Yallet provides "hosted policy + audit" services, and the manufacturer only implements business policy logic.
3. **Key/Signature**: The delegation type requires "delegated delivery + secure storage + signature implementation" or docking Yallet managed signature; the remote signature type requires "end-side request queue + result callback" protocol and implementation.
4. **E2EE and messages**: The same `AgentMessage` format and encryption method (X25519 + business payload) are reused between the backend and the client, and between the backend and the backend. The channel can use HTTPS/WebSocket or message queue, and there is no need to use RWA cNFT (unless you want to store certificates on the chain).

In this way, the same set of framework interfaces can be used in both browser plug-ins and pure backends. Pure backends are indeed much richer in terms of key models, persistence, computing power, and integration than "just plug-ins." Using the two key models and runtime abstractions in this article, "pure backends" can be incorporated into the same framework design.
