# Technical Proposal
### Groupthink - How to Build It

---

## The Core Constraint

Article Seven (The Paper Test) says: software can only do things that could also be done on paper. This means the software is an accelerator, not a dependency. Every feature must have a paper equivalent. If the server goes down, every member should still have a readable copy of everything the group has ever decided.

This constraint eliminates a lot of architectural complexity. We are not building a platform that groups depend on. We are building a tool that groups can use, leave, and replace.

---

## The Record is the Product

Everything in Groupthink flows through the record. Proposals, votes, membership changes, delegation, splits. The record is an append-only log. Old entries are never changed or deleted. Every member can hold a copy. No copy is more official than another.

This is, structurally, a replicated append-only log. The question is how to replicate it.

### Why CRDTs

A CRDT (Conflict-free Replicated Data Type) is a data structure where every copy can be edited independently, and all copies merge automatically without coordination. There is no leader. There is no "master copy." Edits converge deterministically regardless of the order they arrive in.

This maps directly onto Groupthink's principles:
- "Any member can keep their own copy of the record. Every copy is equally real." (Document One, 4.4)
- "If two copies of the record got out of sync... both versions are kept as history." (Document One, 9.4)
- The Paper Test: CRDTs separate the data model from transport. The same document can sync over a server, a local WiFi network, a USB drive, or a printout that someone types back in.

The recommended library is **Automerge**, a mature CRDT implementation designed for exactly this kind of collaborative document.

**Why not Git?** Git looks decentralized, but in practice every group ends up with a "main" remote that everyone pushes to. That remote becomes a single point of failure. Git also requires coordination on concurrent writes (merge conflicts), which CRDTs eliminate by design.

**Why not pure peer-to-peer (libp2p, etc.)?** P2P requires peers to be online at the same time and requires NAT traversal, which is fragile. CRDTs decouple the data from the transport. You can use P2P as one transport when it works, and fall back to anything else when it doesn't.

**Why not federation (Matrix, ActivityPub)?** Federation assumes persistent servers with stable identities and DNS entries. Those can be seized, blocked, or shut down. CRDTs are more fundamental. You could use a federated protocol as one transport layer, but the CRDT document is the source of truth, not any server.

---

## Data Model

Each group is an Automerge document containing structured data:

```json
{
  "group": {
    "name": "The Workshop",
    "created": "ISO-8601",
    "document_zero_version": "0.2"
  },
  "record": [
    {
      "id": "unique-id",
      "type": "proposal | vote | join | leave | delegation | rulebook-change | split",
      "author": "member-public-key",
      "timestamp": "ISO-8601",
      "signature": "cryptographic-signature",
      "data": { }
    }
  ],
  "rulebook": { },
  "members": { },
  "circles": {
    "circle-name": { }
  }
}
```

The `record` array is append-only. Automerge's CRDT semantics guarantee that entries added on different devices merge without conflict. The `signature` field on each entry lets any member verify that the action was taken by a real member, even without contacting a server.

`rulebook` and `members` are derived state: they can always be rebuilt by replaying the record from the beginning. They exist for convenience, not as source of truth.

The entire document can be exported to human-readable Markdown or JSON at any time (Paper Test).

---

## Sync and Hosting

Servers exist for convenience, not authority. They are relay points, like email servers. Useful when available, not required.

### How sync works

1. **When infrastructure is available**: servers relay Automerge changes between members efficiently. A member opens the app, their local document syncs with the relay, and changes from other members arrive. This is the normal, everyday experience.

2. **When infrastructure is limited**: two phones on the same WiFi network can sync directly. A laptop and a tablet can sync over Bluetooth. The sync protocol is the same regardless of transport.

3. **When infrastructure is hostile**: someone can carry the full document state on a USB drive, sync in person, or even transmit it over radio. The CRDT merges cleanly no matter how the data traveled.

### Deployment tiers

**Tier 1: Managed Hosting (easiest)**
A hosted service (groupthink.app or similar) runs relay servers and provides a web UI. Members interact through the app. Under the hood, their local Automerge document syncs with the relay. Any member can export the full record at any time. This is how most groups will start.

**Tier 2: Self-Hosted Relay**
A group runs its own relay on any server, VPS, or Raspberry Pi. Same application, same data format, full control. The group owns its infrastructure.

**Tier 3: No Server**
Members sync directly. Local network, USB, Bluetooth, or any other transport. Maximum independence, minimum convenience.

A group can move between tiers at any time. The data format is identical everywhere. There is no migration, no export/import. You just point your app at a different relay, or stop using a relay entirely. The document is the same document.

---

## Identity

Each member holds a cryptographic keypair. The public key is their identity within a group. This is scoped per group: a person generates a separate keypair for each group they join, so nothing links their identity in Group A to their identity in Group B.

### How it works

1. You generate a keypair. Your public key is your pseudonym in this group.
2. When you join, existing members sign your public key, creating a membership credential (a vouching signature).
3. Every action you take (proposal, vote, delegation) is signed with your private key. Any member can verify it was really you.
4. You can attach a human-readable name to your key, or not.

### Why this matters

If a server is seized, there is no membership list to capture. The record contains actions taken by public keys. "Public key a7f3b2... voted yes on Proposal 12" is meaningless unless an adversary can link that key to a physical person. And since keys are different per group, compromising one group's membership reveals nothing about another.

### The Sybil problem

One person creating many fake identities is solved socially, not cryptographically: existing members vouch for new members. This is exactly how Document One's membership process works (Section 1.2). A friend group does this naturally. A larger organization might require multiple existing members to co-sign a new membership credential. This mirrors how real trust networks function.

### Why not zero-knowledge proofs?

ZK is genuinely hard to implement correctly, the tooling is immature, and the computational overhead is significant. Pseudonymous keypairs give you 90% of the privacy benefit with 10% of the complexity. If ZK tooling matures, it can be layered on later without changing the architecture.

### Why not real-name identity with encryption?

Encryption protects data in transit and at rest, but not against a compromised server operator or a legal order to decrypt. Pseudonymity means there's nothing meaningful to decrypt.

---

## The Group Creation Flow

This is where the "wizard" idea from our earlier discussion comes in:

1. **Name your group** and invite founding members
2. **Pick a template** (or start from Document One defaults):
   - Casual (friends, clubs): shorter discussion/voting periods, no sponsors needed
   - Cooperative (worker-owned, shared resources): longer periods, higher thresholds for rulebook changes
   - Governance (neighborhood councils, associations): structured circles, delegation enabled
   - Custom: start from raw Document One and configure everything
3. **Review the rulebook** the template generates
4. **Founding vote**: all founding members approve the initial rulebook (simple majority, as Document One specifies)
5. **The group is live**

Templates are convenience, not magic. Every template produces a standard rulebook that the group can amend through the normal proposal process.

---

## What the Software Does (and Doesn't Do)

**The software does:**
- Present proposals, discussions, and votes in a clean UI
- Track voting periods and deadlines automatically
- Calculate delegation and enforce caps
- Track active/inactive member status
- Maintain the append-only record
- Allow full export of all data at any time
- Support circles as nested groups
- Handle the mechanics of splitting (copy circle record, create new group)

**The software does not:**
- Make decisions for the group
- Prevent any action that could be done on paper (e.g., a member can always "vote" by telling another member in person, and that member can record it)
- Lock data behind accounts, paywalls, or proprietary formats
- Create dependencies that would break the group if the software disappeared

---

## Technology Stack (Proposed)

- **Sync engine**: Automerge (CRDT library). Each group is an Automerge document. Handles merge, conflict resolution, and change tracking.
- **Relay server**: Lightweight server (Node.js or Go) running automerge-repo for Tier 1/2. Relays changes between members. Stateless by design: if a relay dies, members still have the full document locally.
- **Frontend**: Progressive Web App (works on any device with a browser, installable, works offline). Automerge runs in the browser, so the app works without a network connection.
- **Local-first storage**: Each member's device holds the full Automerge document. IndexedDB in the browser, SQLite or filesystem for native apps.
- **Data format**: Automerge binary for sync efficiency, with JSON and Markdown exports for human reading (Paper Test).
- **Authentication**: Ed25519 keypairs per group membership. Actions are signed locally before entering the document.
- **Transport**: Relay server (WebSocket) for normal use, with fallback to local network sync, Bluetooth, or file-based exchange. The transport is pluggable because Automerge doesn't care how changes arrive.
- **Hosting**: Docker container for self-hosted relays; managed relay instances for Tier 1.

---

## Open Questions

1. **Real-time vs. async**: Automerge supports both naturally. Live sync via WebSocket gives real-time updates; periodic sync works for low-connectivity environments. The question is how much UI complexity to invest in the real-time experience (presence indicators, typing notifications, etc.).

2. **Encryption**: Should records be encrypted at rest? The Paper Test implies readability, but "any member can read it" doesn't mean "anyone in the world can read it." Group-level encryption with member keys would protect privacy while maintaining transparency within the group.

3. **Spam/abuse in managed hosting**: If Tier 1 is a public service, how do we prevent abuse without creating gatekeepers? Rate limiting and invite-based group creation are options that don't violate the articles.

4. **Mobile**: A PWA covers most cases, but native apps for iOS/Android would improve notifications and offline support. Worth the development cost?

5. **Inter-group communication**: Should groups be able to discover and interact with each other (e.g., for inter-group proposals)? The CRDT model makes this possible without federation infrastructure, but the protocol for cross-group proposals needs design.

6. **Economic model**: How is managed hosting sustained? Freemium (free for small groups, paid for large)? Donations? Cooperative ownership of the platform itself (eating our own dog food)?

