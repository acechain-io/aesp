#AESP privacy related function description

This article introduces the new privacy-related capabilities in **dev.aesp**, which are implemented based on `dev.aesp/docs/PRIVACY_ARCHITECTURE.md` and `src/privacy/`. For readers on the Yallet extension side: Learn how AESP reduces on-chain traceability through **context-isolated temporary addresses** and how it works with policy, auditing, and Yault settlement layers.

---

## 1. Problems to be solved: Traceability on the chain

### 1.1 Phenomenon

When an Agent interacts with the Yault settlement layer under AESP, all on-chain transactions are visible to anyone:

- **Transfer out**: `Vault address → Payee`, observers can see that the funds come from the same vault.
- **Transfer**: `Payer → Vault Address`, observers can see that the funds are transferred to the same vault.
- **Aggregation Analysis**: After multiple transactions are associated with the same vault address, it can be inferred:
- Transaction mode (time, frequency, amount)
- Counterparty network (who to trade with)
- Fund size and business relationships
- For institutions: investment strategies, suppliers, salary structures, etc.

Under the agent economy, transactions are more frequent, patterns are more regular, and budget constraints are more obvious, which will amplify the above traceability risks.

### 1.2 Goals

Under the premise of **not giving up auditing capabilities**:

- Make it impossible for **on-chain observers** to attribute multiple transactions to the same subject (treasury/user).
- Allow **asset owners** to still restore the complete history in a local or authorized environment through encrypted audit records.

---

## 2. Solution overview: context-isolated temporary address

### 2.1 Core ideas

Leverage ACE-GF's **context_info (ACE-GF)** to derive multiple sets of keys/addresses from the same identity root in **different contexts**:

- Different context → HKDF outputs are **computationally uncorrelated**, and observers cannot attribute different temporary addresses to the same subject.
- Same mnemonic + same context → **Deterministic recovery**, the asset owner can re-derive and audit.

Context is composed of ordered fragments, for example:

```
agent:{agentId}, dir:out, tx:{txUUID}, seq:{n}
→ Sort and concatenate into context_info
→ Used for HKDF derived on-chain addresses
```

### 2.2 Two-hop transaction structure

**Transfer out**: Vault → Temporary transfer out address A' → Payee
The observer only sees "Unknown address → Payee" and cannot push it back to the vault.

**Transfer**: Payer → Temporary payment address B' → (Collection) → Treasury
The observer only sees "Payer → Unknown Address" and cannot push it back to the vault.

Temporary addresses are derived by AESP using `context_info`; collection is performed in batches by policy by **ConsolidationScheduler** (see below).

### 2.3 Nature of privacy (brief)

| Nature | Meaning |
|--------------|------|
| Unassociability | Addresses derived from different contexts are computationally unassociable |
| Untraceability | The correspondence between the temporary address and the vault cannot be established on the chain |
| Recoverability | Asset owner can use mnemonic + context tag record re-derivation |
| Forward security | Disclosing one temporary address does not affect other temporary addresses |
| Auditable | The asset host maintains encrypted context tag records and can decrypt and restore history |

---

## 3. Privacy Level (Privacy Level)

Not every transaction requires the utmost privacy. AESP introduces **PrivacyPolicy** in the policy, supporting three levels:

| Level | Mechanism | Cost/Overhead | Applicable Scenarios |
|----------------|--------------------------|------------------|----------|
| **transparent** | Send and receive directly using the treasury address | None | Public behavior (donation, public procurement) |
| **basic** | Fixed context address by agent (one group for each agent) | One-time derivation | Daily operations, just isolate by agent |
| **isolated** | By transaction temporary address (one context for each transaction) | One more on-chain confirmation/two hops for each transaction | Sensitive business and competitive intelligence protection |

Configurable in the policy:

- `level`: `transparent` | `basic` | `isolated`
- `auditStorage`: Audit tags are stored in `memory` | `local` | `arweave`
- `batchConsolidation`, `consolidationThreshold`, `preDerivationPoolSize`, etc. (see type definition).

---

## 4. Implementation module (src/privacy/)

### 4.1 Directory and Responsibilities

```
src/privacy/
├── index.ts — Export AddressPoolManager, ContextTagManager, ConsolidationScheduler, etc.
├── address-pool.ts — Temporary address pool: derived, pre-derived, taken from the pool
├── context-tag.ts — Context tags: creation, encryption, local/Arweave storage, cNFT minting interface
└── consolidation.ts — Consolidation scheduling: batches triggered by timing/threshold are swept back to the vault from the temporary address
```

### 4.2 AddressPoolManager (temporary address pool)

- **deriveEphemeralAddress**: Derive a temporary address (main path of `isolated`) for a single transaction; parameters include agentId, chain, direction (inbound/outbound), optional txUUID.
- **getFromPool**: Get an address from the pre-forked pool (for `basic` or to reduce latency).
- **replenishPool**: Pre-generate a batch of addresses when idle to fill the pool.
- **getAllDerivedAddresses**: Lists the addresses derived by an agent to facilitate reconciliation.
- **resolveAddressToPrivacyLevel**: The privacy level that should be used based on the address reverse check (for execution layer routing).

**Precondition**: Rely on the context derivation of ACE-GF (such as `evm_get_address_with_context`, `view_wallet_unified_with_context_wasm`, etc.). Only supported by **REV32 wallet**; Legacy UUID wallet will detect and report an error through `AddressPoolManager.supportsContextIsolation()`. You should call `supportsContextIsolation()` or catch the `REV32_REQUIRED` error before use.

### 4.3 ContextTagManager (context tag and audit)

- **createTag**: Generate `ContextTagRecord` (agentId, contextInfo, derived address, chain, direction, amount, counterparty, txHash, commitmentId, negotiationSessionId, privacyLevel, etc.) based on a transaction.
- Save the record locally first; optional:
- **ArweaveUploader**: Upload the encrypted tags to Arweave (the interface is implemented by the caller).
- **AuditNFTMinter**: Cast the Arweave results into cNFT (the interface is implemented by the caller) to facilitate attribution and recovery using Yallet's existing VA-DAR/cNFT capabilities.
- **recoverTags**, **getTagsByAgent**, etc.: recover from local cache or (after uploading/minting is implemented) from on-chain/Arweave, query by agent.

The content of the tag is encrypted using the ECIES public key of the owner's xidentity, and only the asset owner can decrypt it, making it "auditable but invisible to the outside world".

### 4.4 ConsolidationScheduler (Consolidation Scheduling)

- **scheduleConsolidation**: Set a scheduled collection (such as every 4 hours) for an agent+chain, and sweep the balance of the temporary payment address back to the vault in batches.
- **consolidateNow**: Perform a consolidation immediately.
- **getConsolidationHistory**: Query the collection record of the agent.

The actual "transfer from multiple addresses back to the treasury" on the chain is implemented by the caller **ConsolidationHandler.consolidate**, and AESP is only responsible for scheduling and recording.

---

## 5. Connection point with Yallet/extension

1. **Policy Engine**: Determined at the execution layer (PolicyEngine or execution runtime) based on the `privacyLevel` in the policy:
- `transparent` → directly use the vault address;
- `basic` / `isolated` → Get the temporary address through AddressPoolManager, then go two hops or get it from the pool.
2. **MCP Tool**: If supported by the Yault backend, `yault_deposit`, `yault_create_allowance`, etc. can add optional `context_info` parameters to align with AESP's context derivation.
3. **WASM**: It is necessary to expose context-related interfaces (such as `view_wallet_unified_with_context_wasm`, `build_vault_context`, `evm_get_address_with_context`, etc.), and the extension or backend can load the same set of WASM for reuse.
4. **Commitment and Negotiation**: The temporary address (provided by AddressPoolManager) can be written in the CommitmentBuilder/Negotiation Agreement to make the on-chain commitment consistent with the privacy address.
5. **Audit and Compliance**: The encrypted tags of ContextTagManager can be regarded as the "private version of the audit log"; if the extension side needs to display or export the audit, it can only do UI or export the decrypted and locally visible tags.

---

## 6. Costs and Limitations (Summary)

- **Audit cost per transaction**: Arweave storage is about ~$0.005/item, cNFT minting is about ~$0.0001; in high-frequency agent scenarios (such as 100 transactions/day), it is about ~$15/month.
- **Two-hop delay**: Transfer out requires one more on-chain confirmation; pre-funding can be done through **Pre-derived address pool** to reduce the first-hop wait.
- **Chain and Wallet**: The current context derivation and signature of EVM have been implemented in acegf-wallet; Solana/BTC, etc. need to complete the corresponding context signature in acegf-wallet. Only REV32 wallets support context isolation.
- **Threat Model**: It is aimed at the traceability of **on-chain graph analysis and aggregate inference**; it does not provide strong anonymity for global opponents who can "observe the entire chain status", nor does it replace coin mixers.

---

## 7. Why “controllable privacy” cannot rely solely on the application layer + ordinary HD wallet

If the bottom layer is a **standard HD wallet** (BIP32/44: `m/44'/coin'/account'/change/index`), but the **application layer** is forced to "derive an unassociable address based on context", it will usually become ugly and the security boundary will be blurred.

### 7.1 Limitations of ordinary HD

- **The path is a fixed structure**: The path space is a tree of "account/change/serial number" and does not have the "context" dimension.
- **The path itself may be on the chain or inferred**: Some chains or clients will expose/infer the path. Using "different indexes as different contexts" is equivalent to exposing the relationship to observers.
- **To be non-correlated, context must be used to make KDF**: the truly non-correlated derivation should be `KDF(seed, context_info)`, and if the context is different, the output is computationally independent. The standard BIP32 `CKDpriv` is a fixed formula and has no entry for "arbitrary context string".

Therefore, without expanding the wallet, the application layer has only two options:

1. **Abuse of paths**: For example, using `account` or `index` to encode "context" and derive a bunch of addresses. The problem is that the path space is limited, the semantics are exposed, and many wallets/chains will regard these addresses as different serial numbers of the same account. **Associability is not eliminated**, it is just "a few more addresses".
2. **The application layer makes its own KDF**: The application gets the seed or extended private key from the wallet, does `HKDF(seed, context)` within the application and then assigns the address. Although you can get non-associable addresses in this way, but:
- The key material must leave the security boundary of the wallet, and **security responsibilities and attack surfaces are pushed to the application layer**;
- Each client that supports "controllable privacy" must correctly implement and protect the same set of KDF and key logic, **implementation is repetitive and error-prone**;
- Conflicts with the best practice of "the wallet only exposes the signature/derivative interface and does not expose the seed", **the whole thing will be very ugly**.

### 7.2 Trade-offs of the current design: wallet layer supports context

AESP's privacy architecture assumes that the wallet layer (ACE-GF REV32) already supports "derivatives with context":

- Derivation and signing are completed in the wallet (WASM), **the key does not leave the wallet**;
- The application layer only passes the **context string** (such as `agent:xxx:dir:out:tx:uuid`), and the wallet internally does `HKDF(identity_root, context)` and derives the address/signature;
- The application layer only does: select the privacy level, manage the address pool, record audit labels, and adjust the collection, and **no more key derivation**.

In this way, the implementation of "controllable privacy" is relatively clean: the ugly and error-prone key logic is converged in **a** wallet implementation (acegf-wallet), and the application layer only does orchestration and policy. The price is: **You must use a wallet that supports REV32 + context**. Traditional BIP32/44-only HD wallets cannot natively support this set of capabilities; if you want to do it on traditional HD, you have to accept the ugliness and risk of the above-mentioned "application layer KDF" or "path abuse".

### 7.3 Summary

- **On the existing "HD-only wallet", it will be very ugly to implement truly controllable privacy at the application layer**: either path abuse (insufficient non-correlation), or the application layer uses seeds to do KDF (security and boundary blurring).
- **A relatively clean approach is that the wallet layer natively supports context derivation** (such as ACE-GF REV32), and the application layer only does strategy, pooling, auditing, and collection; AESP's current design follows this path.

---

## 8. Reference

- **AESP Privacy Architecture (English)**: `dev.aesp/docs/PRIVACY_ARCHITECTURE.md`
- **ACE-GF context isolation**: related spec in acegf-wallet warehouse (such as `ACEGF_CONTEXT_ISOLATION_SPEC.md`)
- **Type definition**: `dev.aesp/src/types/privacy.ts` (PrivacyLevel, PrivacyPolicy, ContextTagRecord, EphemeralAddress, ConsolidationRecord, ContextWasmFunctions, etc.)
- **VA-DAR/cNFT**: Yallet’s existing encrypted certificate and cNFT capabilities are used for the persistence and recovery of audit tags

---

*The document is based on the AESP privacy architecture v1.0 (2026-02) and the current implementation of dev.aesp, and is used as a reference for Yallet extension and product-side docking. *
