NIP-NSUB [placeholder name]
========

NSUB as delegate for NSEC/NPUB pairs (pre-alpha for feedback)
-------------------------------

`draft` `optional`

This proposed NIP introduces hierarchical key management for Nostr, enabling users to create subordinate private keys (`nsub`) derived from the master private key (`nsec`). Events signed by `nsub` keys are cryptographically tied to the same public key (`npub`)  . The model introduces a delegation proof mechanism, enabling secure delegation and revocation of signing authority to subordinate keys without compromising the master key. This design improves account security, simplifies key recovery, and supports scalable multi-client or multi-user scenarios.

---

## Problem Description

Current Nostr key management relies on a single private key (`nsec`) to authenticate and sign all user events. While this design ensures simplicity, it introduces critical vulnerabilities that can compromise account security and usability. If the `nsec` is exposed — whether through user error, malicious software, or insecure client storage — the account is effectively "burned", and recovery is impossible, forcing abandonment of the associated `npub`. This leads to significant problems:

1. **Account Compromise**:

   - A compromised `nsec` allows an attacker to impersonate the user, post malicious events, or delete existing events.
   - Once leaked, there is no way to revoke access or mitigate damage, as the `nsec` cannot be invalidated or replaced.

2. **Authentication Issues**:

   - Users must distribute their `nsec` to multiple clients or devices, increasing the risk of compromise.
   - High-value accounts (e.g., organizations, influencers, or businesses) require secure, multi-device setups, but the single-key model cannot adequately support this without significant risk.

3. **Loss of Followers and Content**:

   - When an `nsec` is compromised, users often abandon their `npub`, losing all followers, social connections, and previously signed content.
   - Migrating to a new account is disruptive and diminishes the user experience, especially for high-profile or corporate accounts.

---

## Proposed Use-case Solution

This proposal seeks to resolve the issues described above by introducing a hierarchical key system with subordinate keys, the `nsub`, and revocable delegation proofs with an optional expiry. This model allows users to:

- Securely distribute unlimited subordinate keys to team members, client applications or tracked devices.
- Revoke compromised or retired keys without abandoning the `npub` account with valuable follower and content connections.
- Support high-value, multi-device, or multi-user scenarios.

This hierarchical model aligns with Nostr’s decentralization principles by reducing reliance on centralized recovery mechanisms while maintaining user control and flexibility.

---

## Specification

### Key Hierarchy

1. **Master Private Key:** `nsec`:

   - Root key for generating and managing subordinate keys.
   - Must remain secure and offline where possible.

2. **Subordinate Private Keys:** `nsub`

   - Deterministically derived from the master private key using a hierarchical deterministic (HD) method (e.g., BIP-32 HD Wallet principles).
   - Each `nsub` is associated with a unique path or purpose (e.g., per client or per device).
   - Delegation proofs bind each `nsub` to the master `nsec`.

3. **Public Key:** `npub`

   - Remains unchanged for the user and maps to all subordinate keys.


---

### NSEC/NSUB In Use Cases

1. **Revoking a Compromised Client Key**:\
   An `nsec` may sign an event to revoke a compromised `nsub` and prevent further misuse while maintaining account integrity.

2. **Multi-Device Security**:\
   Each device is issued its own `nsub`, isolating risk and ensuring independent security.

3. **Corporate Account Management**:\
   Team members use individual `nsub` keys for controlled access. Revoked keys do not affect the entire account.

4. **Mitigating Accidental Exposure**:\
   An exposed `nsub` can be revoked while other keys remain valid, reducing the risk of total compromise. The compromise of an `nsub` does not compromise the account `npub` and its content and social relationships remain intact.

5. **High-Value Accounts**:\
   Influencers and organizations can recover from partial compromises without disrupting their presence.

5. **Content Repair**:\
   Content created by a revoked `nsub` may still be removed by a newly delegated `nsub` that is permissioned to delete the select fraudulent content.

---

### Delegation Proofs

Delegation proofs authorize subordinate keys (`nsub`) to sign events on behalf of the master key (`nsec`).

#### Format:

```json
{
  "delegator_pubkey": "<master_npub>",
  "delegatee_pubkey": "<subordinate_public_key>",
  "conditions": {
    "event_kinds": [1, 0],
    "expiration": "<expiration_timestamp>"
  },
  "purpose": "Client authentication for mobile device",
  "version": 1,
  "signature": "<signed_by_nsec>"
}
```

### Fields

- **delegator_pubkey**: The public key of the master key.
- **delegatee_pubkey**: The public key of the subordinate key.
- **conditions**: Restrictions for the subordinate key.
  - **event_kinds**: Defines the kinds of events the `nsub` can sign.
  - **expiration**: Optional expiration timestamp for the delegation proof.
- **purpose**: Context for the delegation proof (optional).
- **version**: Version number to ensure future extensibility.
- **signature**: Cryptographic signature of the delegation proof generated by the `nsec`.

---

### Event Kind Declarations and Clarifications

To implement hierarchical key management effectively, this NIP introduces two new event kinds: **Delegation Proof Event** and **Revocation Event**.

#### **Event Kind:** Delegation Proof (kind: 30080) 

Establishes the relationship between the master key (nsec) and subordinate keys (nsub) and sets any permissions and expiration parameters for the nsub.

Format: 
```json
{
  "kind": 30080,
  "delegator_pubkey": "<master_npub>",
  "delegatee_pubkey": "<subordinate_public_key>",
  "conditions": {
    "event_kinds": [1, 0],
    "expiration": "<expiration_timestamp>"
  },
  "purpose": "Client authentication for mobile device",
  "version": 1,
  "signature": "<signed_by_nsec>"
}
```

#### **Event Kind:** Revocation Event (kind: 30081) 

Allows the master key (nsec) to revoke a subordinate key (nsub), invalidating its signing authority. Informs relays for rejection of events signed by revoked keys and offers clear reasons for a revocation (e.g., key_compromised, expired).

Format:
```json
{
  "kind": 30081,
  "npub": "<master_npub>",
  "revoke_nsub": "<subordinate_public_key>",
  "reason": "Key compromised",
  "timestamp": "<timestamp>",
  "expiration": "<optional_expiration_timestamp>",
  "signature": "<signed_by_nsec>"
}
```

---

## Relay and Client Implementation

### Relay Optimization

To enhance relay performance while supporting delegation proofs and revocation events, two complementary optimization approaches are proposed:

#### **1. Caching Validated Proofs**

- **Description**: Relays cache validated delegation proofs to minimize repetitive cryptographic checks.
- **Implementation**:
  - Cache structure stores mappings of `delegatee_pubkey` to their validated conditions (e.g., expiration timestamps, event kinds).
  - Example cache entry:

```json
{
  "delegatee_pubkey": "<subordinate_public_key>",
  "valid_until": "<expiration_timestamp>",
  "event_kinds": [1, 0],
  "verified": true
}
```

- **Advantages**:
  - Reduces computational overhead during high-traffic scenarios.
  - Ensures quick lookup for relay processing.

#### **2. Efficient Cryptographic Libraries**

- **Description**: Utilize high-performance cryptographic libraries to handle signature verification for delegation proofs and revocation events.
- **Recommendation**:
  - Libraries such as `libsecp256k1` provide optimized implementations for cryptographic operations, ensuring scalability for relays handling high-volume workloads.
- **Advantages**:
  - Improves signature verification speed.
  - Reduces latency for processing delegation and revocation events.

#### Comparison of Optimization Approaches

| **Optimization** | **Focus** | **Advantages** | **Trade-offs** |
| --- | --- | --- | --- |
| **Caching Validated Proofs** | Reduces repetitive cryptographic checks | Faster relay processing for validated proofs | Requires memory for storing cached data |
| **Efficient Cryptographic Libraries** | Improves cryptographic operation speed | Enhances scalability for high traffic loads | May depend on library compatibility |

Implementing both optimizations can maximize relay performance by balancing computational efficiency and scalability. Caching ensures quick lookups, while optimized libraries handle cryptographic operations efficiently.

---

### Client UX Recommendations

Client applications may be able to off user experiences for managing hierarchical keys without ever revealing the nsec, allowing the nsec to be stored on software or hardware with dedicated use as delegation and revocation signers.

---

### Backward Compatibility

- **Existing Users**: This proposal preserves the `npub` and `nsec` structure, allowing existing users to adopt subordinate keys (`nsub`) without losing followers, content, or social connections.
- **Non-Adopting Relays**: Relays that do not implement delegation proofs (`kind: 30080`) or revocation events (`kind: 30081`) will continue to accept events signed by `nsub` keys as valid, treating them like any other signed events.
- **Adopting Relays**: Relays implementing this NIP will validate delegation proofs and enforce revocations, enhancing security while remaining interoperable with non-adopting relays.
- **Interoperability**: Events signed by `nsub` keys remain valid across all relays, ensuring seamless coexistence between adopting and non-adopting relays.

---

### Acknowledgments

- **Status**: Draft (this pre-alpha draft version is the first requests for feedback.)
- **Prepared By**: @swbratcher // swb@primal.net
- **Date**: 2024/11/28
