# Technical Proposal
### Groupthink - How to Build It

---

## The Core Constraint

D0 Article Six (The Paper Test) says: software can only do things that could also be done on paper. This means the software is an accelerator. Every feature must have a paper equivalent. If the server goes down, every member should still have a readable copy of everything the group has ever decided.

---

## The Record is the Product

Everything in Groupthink flows through the record. Discussions, consent outcomes, membership changes, splits. The record is an append-only log. Old entries are never changed or deleted. Every member can hold a copy. Every copy is equally valid.

This is, structurally, a replicated append-only log. The question is how to replicate it.

### Why CRDTs

A CRDT (Conflict-free Replicated Data Type) is a data structure where every copy can be edited independently, and all copies merge automatically without coordination. There is no leader. There is no "master copy." Edits converge deterministically regardless of the order they arrive in.

This maps directly onto Groupthink's principles:
- "Any member can keep their own copy of the record. Every copy is equally valid." (D1 A4.4)
- "If two copies of the record got out of sync... both versions are kept as history." (D1 A9.4)
- The Paper Test: CRDTs separate the data model from transport. The same document can sync over a server, a local WiFi network, a USB drive, or a printout that someone types back in.

The recommended library is **Automerge**, a mature CRDT implementation designed for exactly this kind of collaborative document.

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
      "type": "discussion | consent | stand-aside | block | set-aside | join | leave | rulebook-change | split | circle | dormancy",
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
3. Every action you take (raising a discussion, consenting, standing aside, blocking) is signed with your private key. Any member can verify it was really you.
4. You can attach a human-readable name to your key, or not.

### Why this matters

If a server is seized, there is no membership list to capture. The record contains actions taken by public keys. "Public key a7f3b2... consented to Discussion 12" is meaningless unless an adversary can link that key to a physical person. And since keys are different per group, compromising one group's membership reveals nothing about another.

### The Sybil problem

One person creating many fake identities is solved socially: existing members vouch for new members. This is exactly how Document One's membership process works (D1 A1.2). A larger organization might require multiple existing members to co-sign a new membership credential.

### Why not zero-knowledge proofs?

ZK is genuinely hard to implement correctly, the tooling is immature, and the computational overhead is significant. Pseudonymous keypairs give you 90% of the privacy benefit with 10% of the complexity. If ZK tooling matures, it can be layered on later without changing the architecture.

### Why not real-name identity with encryption?

Encryption protects data in transit and at rest, but not against a compromised server operator or a legal order to decrypt. Pseudonymity means there's nothing meaningful to decrypt.

---

## The Group Creation Flow

This is where the "wizard" idea from our earlier discussion comes in:

1. **Name your group** and invite founding members
2. **Pick a template** (or start from Document One defaults):
   - Casual (friends, clubs): shorter notice periods, minimal overhead
   - Cooperative (worker-owned, shared resources): longer notice periods, extended notice for rulebook changes
   - Governance (neighborhood councils, associations): structured circles, longer discussion periods
   - Custom: start from raw Document One and configure everything
3. **Review the rulebook** the template generates
4. **Founding consent**: all founding members discuss and consent to the initial rulebook. All founders must consent. *(See D1 A1.1.)*
5. **The group is live**

Templates are convenience. Every template produces a standard rulebook that the group can amend through the normal discussion-consent process.

---

## What the Software Does (and Doesn't Do)

**The software does:**
- Present discussions, consent status, and outcomes in a clean UI
- Track notice periods and discussion deadlines automatically
- Track active/inactive member status (D1 A1.5)
- Maintain the append-only record
- Allow full export of all data at any time
- Support circles as nested groups
- Handle the mechanics of splitting (copy circle record, create new group)
- Support anonymous resistance checks (D1 A3.7)
- Support atoms as group members (Document Two)

**The software does not:**
- Make decisions for the group
- Prevent any action that could be done on paper
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

5. **Inter-group communication**: Should groups be able to discover and interact with each other (e.g., for cross-group discussions)? The CRDT model makes this possible without federation infrastructure, but the protocol needs design.

6. **Economic model**: How is managed hosting sustained? Freemium (free for small groups, paid for large)? Donations? Cooperative ownership of the platform itself (eating our own dog food)?

