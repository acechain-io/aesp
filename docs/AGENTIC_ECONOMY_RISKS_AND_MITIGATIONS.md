# Agentic Economy Risks and Responses

When Agent automatically executes payments, negotiations, and commitments, it will bring risks such as mispayments, excess/drained wallets, and privacy leaks. This article sorts out risk types by risk type and provides ** ex-ante constraints, in-process protection, and ex-post remedial measures that correspond to the existing framework (strategy, audit, and execution).

---

## 1. Quick overview of risk classification

| Type | Typical Scenario | Consequences |
|------|----------|------|
| **Mistaken payment/wrong payment** | Automatic payment to the wrong object, wrong amount, repeated payment, payment of canceled bills | Fund loss, disputes, high recovery costs |
| **Excess/spent** | The policy is too wide, the single-day/single-transaction limit is invalid, multiple agents superimpose overspending, maliciousness or bugs lead to continuous transfers | Wallets are emptied, affecting life/business |
| **Privacy Leak** | Agent or vendor obtains balance/history/identity and leaks it; E2EE is bypassed; logs are crawled | Identity and asset exposure, targeted fraud, compliance issues |
| **Identity/Authorization Abuse** | The key or token is stolen during the delegation period; the remote signature request is forged or replayed | Unauthorized transactions and commitments are counterfeited |
| **Irreversibility and Dispute** | Confirmed on the chain, the other party does not admit the account, cross-border/cross-jurisdictional areas | Difficult to recover, high legal costs |

The following provides responses based on "before/during/after" and "who will do it", and is linked to the strategy, auditing, and execution in "AGENT_FRAMEWORK_INTERFACES".

---

## 2. Wrong payment/wrong payment

### 2.1 Typical reasons

- The automatic payment rules are written incorrectly (address/amount/duplicate); the invoice is inconsistent with the payment object; the other party forges or tampered with the invoice.
- The policy conditions are too loose (such as allowList is too large, token/chain is not restricted).

### 2.2 Prior constraints (strategy layer)

- **Whitelist**: `allowListAddresses` / `allowListXidentities` only contains **confirmed** payees; new payees must be **manually confirmed** for the first time, and then automatic rules can be added.
- **Amount and frequency**: `maxAmountPerTx`, `maxAmountPerDay`, `maxPerMonth` (expandable PolicyConditions) are strictly limited; it is recommended that large-amount automatic payments are not allowed or approved separately.
- **Invoice Verification**: When automatically paying an invoice, verify the **invoice hash/signature** and the identity of the other party; the amount, payment address and invoice must be consistent before execution (see RWA/Notes invoice flow).
- **Single valid/validity period**: Use `validUntil` or a one-time token for the "pay this invoice" type strategy. It will become invalid after payment to avoid repeated payments.

### 2.3 Protection during events

- **Second verification before execution**: Before executing the automatic payment, the framework will use the **current policy** to calculate again: the payee is in the whitelist, the amount is ≤ single and has not exceeded the date limit, and the invoice has not been paid; if any of the items is not satisfied, execution will be **rejected** and recorded in the audit.
- **Required for audit**: Automatically write `recordExecution` (who, when, to whom, how much, policy id) for each payment; supports query by "Payee/Day/Strategy" to facilitate the discovery of abnormal patterns.

### 2.4 Post-remediation

- **Dispute and Cancellation**: If it is not confirmed on the **chain** or if the other party supports revocation** (such as some payment channels), you can go through the "cancellation/reversal" process; if it has been confirmed on the chain, it can only be handled through **negotiation, arbitration, and insurance**.
- **Immediate shutdown of the policy**: When a user or organization discovers a wrong payment, it will **immediately disable** the automatic payment of the policy or the entire manufacturer (see "Shutdown and Revocation" below); to avoid continued wrong payments.
- **Product layer**: Provides a list of "recent automatic payments", one-click "dispute/contact the other party", and exports audit records to customer service/lawyers.

---

## 3. Overage / Spend all your wallet

### 3.1 Typical reasons

- The policy limit is set too high; multiple vendors/multiple strategies are superimposed, and the total expenditure in a single day exceeds expectations; bugs or malicious logic lead to loops/batch transfers; the entrusted backend is compromised and abused.

### 3.2 Prior constraints (strategy layer)

- **Hard limit**: `maxAmountPerTx`, `maxAmountPerDay` (and optional `maxAmountPerMonth`) **required** and set conservative values; the framework **rejects** policies without limits or with limits of 0/infinity.
- **Total Expenditure Cap**: The framework layer supports "**Global Daily Expenditure Cap**" (across all vendors/strategies). Users can set a "Daily Automatic Payment Total Cap" and check "Today's automatic expenditure + this transaction ≤ cap" before executing any strategy.
- **Retain Balance**: The strategy can include `minBalanceAfter` (the balance of the main account or the designated token shall not be less than X after execution); if it is not satisfied, the transaction will not be executed to avoid emptying the wallet.
- **Cooling and Frequency**: Same payee/same strategy **Minimum interval** (such as 1 hour, 1 day) or **Maximum N transactions in a single day** to prevent out-of-control continuous transfers.

### 3.3 Protection during events

- **Pre-execution summary check**:
- The amount of this transaction ≤ the single transaction of this strategy, the daily remaining balance of this strategy, and the **global daily remaining**;
- If `minBalanceAfter` is configured, the balance after simulation execution is ≥ this value;
Otherwise, it will be rejected and audited.
- **Delegation-type backend**: The delegation period is as short as possible (such as 24h), and the delegation token is bound to the **day limit/policy** of the wallet, and the server executes according to the same set of limits; the backend does not "bypass" the framework to sign by itself.
- **Circuit breaker**: If the same strategy or the same wallet triggers multiple executions in a short period of time (such as within 1 minute), the strategy can be automatically **suspended** and the user notified (suspected abnormality).

### 3.4 Post-remediation

- **Shutdown and revocation**: Users can "disable all automatic payments" or "disable a certain manufacturer/strategy" with one click; under the delegation type, **immediately revoke** the delegation token, and all subsequent requests from the background will be rejected.
- **Audit and Liability**: Complete audit records (who performed what operation under which policy) are used for accountability and insurance claims; if a framework/vendor bug is found, bug bounty or contractual liability can be used.
- **Product layer**: low balance reminder, optional "secondary confirmation" before automatic payment of large amounts, regular "automatic payment summary" emails/pushes.

---

## 4. Privacy leakage

### 4.1 Typical reasons

- Agent or vendor service obtains "balance, transaction history, identity" and then leaks it (hacked, abused, insufficient compliance); E2EE is bypassed by implementation defects or malicious endpoints; audits/logs are accessed or crawled without authorization.

### 4.2 Prior constraints (minimum visible)

- **On-demand disclosure**: Agent/Vendor** can only get the minimum information** required to perform the current operation. For example: to automatically pay an invoice, you only need "the invoice amount + payee + whether this wallet can pay", and do not need "all balances, all history"; the framework side provides **restricted API** (such as `checkCanPay(amount, token)` returns boolean), which does not expose the specific balance.
- **Policy and Identity**: The policy engine calculates "whether execution is allowed" on **local or trusted end**; if the policy is on the manufacturer's backend, it will query through the context of **encryption or desensitization** (such as only transmitting the "amount range" and "whether the counterparty is in the whitelist"), without transmitting the complete on-chain address or history.
- **E2EE default**: Communication with the other party/the other party's agent is always E2EE** (existing xidentity + encrypted channel); the manufacturer's backend **does not hold user keys** in the "remote signature" mode, and tries not to persist plaintext sensitive data.

### 4.3 Protection during events

- **Channel and Endpoint**: Communication uses TLS + E2EE double insurance; sensitive operations (such as signature requests) have nonce/expiration time to prevent replay; manufacturer endpoints need to be authenticated to avoid requests being hijacked to malicious endpoints.
- **Log desensitization**: Mnemonic words/private keys and incomplete addresses will not be recorded in the audit log; "policy id, amount, token, counterparty hash or short id" can be recorded; the original identity and balance are only visible in the authorized "user/compliance audit" view.

### 4.4 Post-remediation

- **Authorization revocation**: After the user revokes the authorization for a certain manufacturer/agent, the party **will no longer receive new data**; historical data will be retained or deleted according to the privacy policy (compliance).
- **Leak Response**: If the leak is confirmed, the product side supports "rotating identity/address" (if the architecture allows), changing passwords/recovery methods, and notifying affected users; reporting to supervision if necessary.

---

## 5. Identity/authorization abuse (commission theft, request forgery)

### 5.1 Typical reasons

- Delegation tokens or short-term keys are stolen; remote signature requests are forged or replayed; users mistakenly click on malicious links to authorize objects they should not authorize.

### 5.2 Before and during the event

- **Delegation scope binding**: The delegation token is bound to **wallet + policy id + limit**; when the server executes it, it verifies that "this request belongs to the policy and limit allowed by the token", otherwise it is rejected.
- **Short-term Validity**: The delegation is set to have a short validity period (such as 24h) and will automatically expire upon expiration; sensitive operations tend to use **remote signature** rather than long-term delegation.
- **Remote signature**: The request contains nonce, timestamp, and requestId; the client displays "who, what to do, and the amount" before allowing the user to approve it to prevent forgery and replay.
- **Revocation**: Users can "revoke the delegation" or "revoke a manufacturer" at any time; after revocation, the old token will become invalid immediately, and the next request to the background will be rejected.

### 5.3 Afterwards

- Same as "excess/spent": shutdown, audit, liability tracing; if it is the platform/manufacturer's responsibility, go for insurance or contract.

---

## 6. Irreversibility and disputes (confirmed on the chain, cross-border)

### 6.1 Beforehand

- **Large Amount/First Time**: Large amount or first time payment object **not fully automatic**, must be manually confirmed or delayed (for example, it can be canceled within 24 hours).
- **Commitment instead of immediate payment**: You can use "on-chain/off-chain commitment" to lock in your intention first, and then pay after the agreed time or conditions are met, which can reduce the situation of "mispayment is irreversible".

### 6.2 Afterwards

- **Dispute Process**: The product provides "Mark Dispute → Export Audit and Evidence → Contact the Other Party/Arbitration"; connects with the "Arbitration Agent/Arbitration Agreement" in "AGENTIC_ECONOMY_VISION".
- **Insurance and Reserve**: Provide optional insurance (mispayment, theft) for "automatic payment/entrustment"; or require manufacturers/custodians to pay deposits/reserves for first payment and then recovery.

---

## 7. Correspondence with existing frameworks

| Risk | Main response | Framework side response |
|------|----------|------------|
| Mispayment | Whitelist, amount/frequency limit, invoice verification, single valid | `PolicyConditions` (allowList*, maxAmount*, validUntil); verification before execution + `recordExecution` |
| Overage/spent out | Hard limit, global cap, reserved balance, cooling, circuit breaker | Extended `PolicyConditions` (minBalanceAfter, maxPerDay global); policy engine summary check; shutdown/revocation of delegation |
| Privacy leak | Minimum visibility, E2EE, log desensitization, authorization revocation | Restricted API (checkCanPay, etc.); E2EE channel; audit structure does not leave clear text sensitive fields |
| Authorization abuse | Delegation binding strategy + quota, short-term validity, revocation | Delegation token is bound to strategy/limit; `validUntil`; invalid if revoked |
| Irreversible/Dispute | Large amount of labor, promised delayed payment, dispute process, insurance | Strategy `allowAutoApprove` is distinguished by amount/object; audit export; docking with arbitration/insurance |

---

## 8. Suggested landing order

1. **Policy layer**: Mandatory `maxAmountPerTx` / `maxAmountPerDay`, whitelist; expand the "global daily cap" "minBalanceAfter"; strictly follow the policy + summary verification before execution.
2. **Audit**: A required audit is automatically executed every time; query and export by policy/day are supported; plaintext keys/full addresses are not recorded in the logs.
3. **Shutdown**: Users can disable a certain strategy/a certain manufacturer/all automatic payments with one click; revocation under delegation type will become invalid.
4. **Product**: Automatic payment summary, low balance and large amount reminders, dispute entry and audit export.
5. **Optional**: Circuit breaker (suspension if automatically executed multiple times in a short period of time), insurance/reserve, and docking with arbitration agreement.

In this way, risks such as mispayments, money spent, privacy leaks, and authorization abuse** form a closed loop through** policy constraints, pre-execution inspections, audits and shutdowns, disputes, and insurance**; the framework side only needs to strengthen verification and expansion conditions on the existing policy and execution interfaces to support the above responses.
