# AESP Privacy Architecture: Context-Isolated Ephemeral Addresses

> Solving the on-chain traceability problem for agent economic transactions

---

## 1. Problem Statement

### The Traceability Problem

When AI agents operating under AESP interact with the Yault settlement layer, all on-chain transactions become publicly visible:

```
                   ┌────────────────────┐
                   │  Yault Vault        │
                   │  (known address)    │
                   └──────┬─────────────┘
              ┌───────────┼────────────┐
              ▼           ▼            ▼
         Payment A   Payment B    Payment C
         to Vendor1  to Vendor2   to Vendor3
```

**Outbound:** `VaultAddress → Recipient` — any observer can see funds originate from the same vault.

**Inbound:** `Payer → VaultAddress` — any observer can see funds flow into the same vault.

**Aggregate analysis:** Multiple transactions linked to the same vault address enable full fund-flow profiling, exposing:
- Transaction patterns (timing, frequency, amounts)
- Counterparty networks (who the entity transacts with)
- Capital flow volumes and business relationships
- For institutions: investment strategies, supplier relationships, payroll structures

This is especially dangerous for institutional users (DAOs, funds, enterprises), where competitors can derive strategic intelligence from on-chain analysis.

### Why This Matters for Agent Economics

AESP introduces a paradigm where AI agents transact autonomously on behalf of humans/institutions. This amplifies the traceability problem because:

1. **High transaction frequency** — Agents may execute dozens to hundreds of transactions daily
2. **Deterministic patterns** — Agent behavior is more predictable than human behavior
3. **Policy-driven amounts** — Budget limits and per-transaction caps create identifiable signatures
4. **Counterparty clustering** — Agents negotiating with specific vendor agents create stable relationship graphs

---

## 2. Solution: Context-Isolated Ephemeral Addresses

### Core Idea

Leverage the `context_info` feature from ACE-GF (MSCIKDF) to derive per-transaction ephemeral addresses that are **cryptographically unlinkable** to each other and to the vault, while remaining **deterministically recoverable** by the owner.

### Foundation: ACE-GF Context Isolation

ACE-GF's `hkdf_expand_with_context` function (implemented in Rust, compiled to WASM) appends a context string to HKDF info labels:

```
identity_root → HKDF(info="ACEGF-REV32-V1-SECP256K1-EVM:{context}") → chain_seed
```

Different `context` values produce **computationally independent** keys from the same `identity_root`. An external observer cannot link addresses derived from different contexts to the same entity.

The context is built from sorted segments:

```
build_vault_context(&["agent:a3f2...", "dir:out", "tx:550e8400-...", "ts:2026-02"])
→ "agent:a3f2...:dir:out:ts:2026-02:tx:550e8400-..."
```

### Two-Hop Transaction Architecture

#### Outbound Payment Flow

```
Vault → Ephemeral Outgoing Address (A') → Recipient
          ↑
  derived from: context_info = build_vault_context(&[
    "agent:{agentId}",
    "dir:out",
    "tx:{txUUID}",
    "seq:{sequenceNumber}"
  ])
```

1. AESP derives an ephemeral address A' using a unique context_info
2. Vault transfers funds to A'
3. A' sends payment to the actual recipient
4. Observer sees: `UnknownAddress → Recipient` — cannot link back to vault

#### Inbound Payment Flow

```
Payer → Ephemeral Incoming Address (B') → Vault
          ↑
  derived from: context_info = build_vault_context(&[
    "agent:{agentId}",
    "dir:in",
    "tx:{txUUID}",
    "seq:{sequenceNumber}"
  ])
```

1. AESP derives an ephemeral address B' for receiving
2. B' is provided to the payer as the payment destination
3. Funds arrive at B' → AESP sweeps to vault
4. Observer sees: `Payer → UnknownAddress` — cannot link to vault

### Privacy Properties

| Property | Guarantee |
|----------|-----------|
| **Unlinkability** | HKDF guarantees computationally independent outputs for different info strings |
| **Untraceability** | No on-chain link between ephemeral addresses and the vault |
| **Deterministic recovery** | Owner can re-derive all ephemeral addresses from mnemonic + context tags |
| **Forward secrecy** | Compromising one ephemeral address reveals nothing about others |
| **Auditability** | Owner maintains encrypted records of all context tags |

---

## 3. Context Tag Record System

### The Audit Problem

While ephemeral addresses break external traceability, the **owner** must be able to reconstruct the full transaction history. This requires maintaining a mapping from context_info tags to their transaction metadata.

### Encrypted cNFT Records on Arweave

Each transaction generates a `ContextTagRecord`:

```typescript
interface ContextTagRecord {
  id: string;                   // UUID
  agentId: string;              // Which agent performed the transaction
  contextInfo: string;          // The full context string used for derivation
  derivedAddress: string;       // The ephemeral address that was derived
  chain: ChainId;               // Which blockchain
  direction: 'inbound' | 'outbound';
  amount: string;               // Transaction amount
  token: string;                // Token identifier
  counterpartyAddress: string;  // The other party's address
  txHash?: string;              // On-chain transaction hash (when available)
  commitmentId?: string;        // Linked AESP commitment (if applicable)
  negotiationSessionId?: string; // Linked negotiation session (if applicable)
  timestamp: number;            // Unix timestamp
  privacyLevel: PrivacyLevel;   // Which privacy tier was used
}
```

### Storage Pipeline

```
ContextTagRecord
  → ECIES encrypt with owner's xidentity pubkey
    → Upload to Arweave (permanent, immutable storage)
      → Mint as Solana cNFT via Metaplex Bubblegum
        → Owner can recover by: mnemonic → xidentity → decrypt → full history
```

This integrates with Yallet's existing VA-DAR protocol — the same infrastructure that stores encrypted photos, files, and notes as cNFTs.

### Cost Analysis

| Component | Per-Record Cost | Notes |
|-----------|----------------|-------|
| Arweave storage | ~$0.005 | ~0.5-1 KB per encrypted record |
| cNFT mint (Solana) | ~$0.0001 | Compressed NFT via Merkle tree |
| **Total per transaction** | **~$0.005** | |

**At 100 agent transactions/day: ~$0.50/day ≈ $15/month** — acceptable audit cost.

---

## 4. Tiered Privacy Policy

Not all transactions require the same privacy level. AESP introduces a `PrivacyPolicy` dimension within the agent policy framework:

### Privacy Levels

| Level | Mechanism | Cost Overhead | Use Case |
|-------|-----------|---------------|----------|
| `transparent` | Direct vault transactions | None | Public actions (donations, open purchases) |
| `basic` | Per-agent fixed context address | One-time setup | Routine operations where agent-level separation is sufficient |
| `isolated` | Per-transaction ephemeral address | +1 tx fee per operation | Sensitive commercial transactions, competitive intelligence protection |

### Policy Configuration

```typescript
interface PrivacyPolicy {
  level: 'transparent' | 'basic' | 'isolated';
  auditStorage: 'memory' | 'local' | 'arweave';
  batchConsolidation: boolean;
  consolidationThreshold: number;
  preDerivationPoolSize: number;
}
```

### Optimization Strategies

#### Pre-derivation Pool

For `isolated` level, pre-derive a pool of ephemeral addresses during idle time:

```
Pool: [A'₁, A'₂, A'₃, ..., A'ₙ]  (pre-funded from vault)
```

When a payment is needed, draw from the pool → eliminates the first-hop latency. Pool is replenished asynchronously.

#### Batch Consolidation

For inbound payments, accumulated funds in ephemeral addresses are swept to the vault in batches:

```
Schedule: Every 4 hours (configurable)
  Scan all inbound ephemeral addresses with balance > 0
  Batch transfer back to vault
  Record all context tags
```

This reduces the number of consolidation transactions while still providing timely fund availability.

#### Chain-Specific Considerations

| Chain | Two-Hop Cost | Recommendation |
|-------|-------------|----------------|
| Solana | ~$0.002 | Use `isolated` freely — costs are negligible |
| Ethereum L2 (Arbitrum, Polygon, Base) | ~$0.02-0.10 | Use `isolated` for sensitive transactions |
| Ethereum L1 | ~$1-10 | Use `basic` by default, `isolated` only for high-value |

---

## 5. Implementation in AESP

### New Module: `src/privacy/`

```
src/privacy/
├── index.ts           — Module exports
├── address-pool.ts    — Ephemeral address pool management
├── context-tag.ts     — ContextTagRecord creation and encrypted storage
└── consolidation.ts   — Batch fund consolidation scheduler
```

### Address Pool Manager

Manages pre-derived ephemeral addresses:

- `deriveEphemeralAddress(agentId, direction, txUUID)` — derive a new ephemeral address
- `getFromPool(agentId, direction)` — get a pre-derived address from the pool
- `replenishPool(agentId)` — pre-derive addresses to fill the pool
- `getAllDerivedAddresses(agentId)` — list all derived addresses for reconciliation

### Context Tag Manager

Manages encrypted audit records:

- `createTag(record)` — create and encrypt a context tag record
- `uploadToArweave(encryptedTag)` — upload to permanent storage
- `mintAuditNFT(arweaveTxId)` — mint cNFT for ownership tracking
- `recoverTags(mnemonic)` — recover all tags from mnemonic (via cNFT discovery)
- `getTagsByAgent(agentId)` — query local tag cache

### Consolidation Scheduler

Manages fund sweeping:

- `scheduleConsolidation(agentId, intervalMs)` — set up periodic sweeping
- `consolidateNow(agentId)` — immediate sweep of all ephemeral addresses
- `getConsolidationHistory(agentId)` — audit log of consolidation events

### Integration Points

1. **PolicyEngine** — evaluates `privacyLevel` to decide direct vs. ephemeral address routing
2. **MCP Tools** — `yault_deposit`, `yault_create_allowance` accept optional `context_info` parameter
3. **WASM Bridge** — exposes `view_wallet_unified_with_context_wasm` and `build_vault_context`
4. **CommitmentBuilder** — can include ephemeral addresses in EIP-712 commitments

---

## 6. Security Considerations

### Threat Model

| Threat | Mitigation |
|--------|------------|
| On-chain graph analysis | Ephemeral addresses break transaction graph connectivity |
| Timing correlation | Batch consolidation with randomized delays |
| Amount correlation | Future: amount splitting for high-value transfers |
| Metadata leakage | Context tags are E2EE encrypted; only owner can decrypt |
| Ephemeral key compromise | Each key is derived independently; no cross-contamination |
| Context tag loss | Arweave provides permanent storage; cNFT provides discoverability |

### Chain Support Status (ACE-GF)

Context-aware signing has been verified against the acegf-wallet codebase:

| Chain | Address Derivation | Transaction Signing | WASM Export |
|-------|-------------------|--------------------|----|
| **EVM** (Ethereum, Polygon, Arbitrum, Base) | YES | YES (EIP-1559, EIP-191, EIP-712) | YES (passphrase + PRF) |
| **Solana** | YES (core seed available) | **GAP** (SolanaSigner not wired) | NO |
| **Bitcoin** | YES (core seed available) | **GAP** (BitcoinSigner not wired) | NO |
| **Cosmos** | YES (core seed available) | **GAP** (no signer module) | NO |
| **Polkadot** | YES (core seed available) | **GAP** (no signer module) | NO |

The core `derive_from_rev32_with_context` in `acegf_core.rs` produces private key seeds for ALL chains. The EVM signer (`evm_signer.rs`) has full context-aware signing functions. Other chain signers need equivalent wiring (straightforward additions mirroring the EVM implementation).

**Important constraint:** Context isolation requires REV32 wallets (new format). Legacy UUID wallets reject non-empty context with `AcegfError::InvalidFormat`.

### Limitations

1. **Two-hop latency** — Outbound payments require an extra on-chain confirmation
2. **L1 gas costs** — Ethereum mainnet makes the two-hop pattern expensive for small amounts
3. **Pool management** — Pre-funded ephemeral addresses tie up capital
4. **Non-EVM chains** — Solana/BTC/Cosmos context-aware signing needs acegf-wallet additions
5. **REV32 only** — Legacy UUID wallets cannot use context isolation
6. **Not a mixer** — This does not provide anonymity against a global adversary who can observe all chain state simultaneously; it provides unlinkability against practical on-chain analysis

### Future Enhancements

- **Stealth addresses** — ERC-5564 integration for receiver-generated one-time addresses
- **Amount splitting** — Break large transactions into random-sized chunks
- **Cross-chain routing** — Use bridge protocols to add cross-chain unlinkability
- **Zero-knowledge proofs** — Prove transaction validity without revealing amount/parties

---

## 7. References

- **ACE-GF Context Isolation Spec**: `/dev.acegf-wallet/docs/ACEGF_CONTEXT_ISOLATION_SPEC.md`
- **MSCIKDF Patent**: PCT/IB2025/061532, US 63/913,956
- **VA-DAR Protocol**: Yallet's Vendor-Agnostic Deterministic Artifact Resolution
- **ERC-4626**: Tokenized vault standard (Yault vaults)
- **EIP-5564**: Stealth Addresses (future integration)
- **Metaplex Bubblegum**: Solana compressed NFT program

---

*AESP Privacy Architecture v1.0 — February 2026*
