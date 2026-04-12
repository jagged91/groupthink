# Technical Proposal
### Groupthink - How to Build It — Updated for Version 0.6

---

## The Core Constraint

The Paper Principle (Documents/Core/2.Implementation.md) says: software can only do things that could also be done on paper. This means the software is an accelerator. Every feature must have a paper equivalent. If the server goes down, every member should still be able to recreate a readable copy of everything the group has ever decided.

---

## The Record is the Product

Everything in Groupthink flows through the record. Discussions, consent outcomes, membership changes, splits. The record is an append-only log. Old entries are never changed or deleted. Every member can hold a copy. Every copy is equally valid.

This is, structurally, a replicated append-only log. The question is how to replicate it.

### Why CRDTs

A CRDT (Conflict-free Replicated Data Type) is a data structure where every copy can be edited independently, and all copies merge automatically without coordination. There is no leader. There is no "master copy." Edits converge deterministically regardless of the order they arrive in.

This maps directly onto Groupthink's principles:
- "Any member can keep their own copy. Every copy is equally valid." (Documents/Core/2.Implementation.md § The Record)
- The append-only record is open to all members and written plainly. CRDTs give this property automatic convergence across copies.
- The Paper Principle: CRDTs separate the data model from transport. The same document can sync over a server, a local WiFi network, a USB drive, or a printout that someone types back in.

The recommended library is **Automerge**, a mature CRDT implementation designed for exactly this kind of collaborative document.

---

## Data Model

Each group is an Automerge document containing structured data:

```json
{
  "group": {
    "name": "The Workshop",
    "created": "ISO-8601",
    "framework_version": "0.6",
    "functions": ["stewardship"],
    "epoch": 0,
    "epoch_threshold_bytes": 10485760,
    "prior_epochs": []
  },
  "record": [
    {
      "id": "unique-id",
      "type": "see entry types below",
      "author": "member-public-key",
      "timestamp": "ISO-8601",
      "signature": "cryptographic-signature",
      "deliberation_ref": "automerge-document-hash (optional)",
      "data": { }
    }
  ],
  "rules": { },
  "members": { },
  "circles": {
    "circle-name": { }
  }
}
```

### Record entry types

The `type` field on each record entry identifies the action. Entry types are organised by the Core and by each activated function:

**Core:** `discussion` · `consent` · `stand-aside` · `set-aside` · `join` · `vouch` · `leave` · `rule-change` · `split` · `circle` · `dormancy` · `dispute` · `correction` · `function-activate` · `function-deactivate` · `epoch-close` · `standing-decision` · `standing-decision-revoke`

**Stewardship:** `commons-identify` · `invocation` · `invocation-response` · `steward-assign` · `steward-recall`

**Federation:** `federation-join` · `federation-leave` · `voice-designate` · `voice-recall` · `compact-form` · `compact-leave` · `deliberation-round` · `commons-federation-convene` · `commons-federation-stake-declare` · `commons-federation-finding`

**Provision:** `provision-acknowledge` · `provision-assessment` · `alternative-map`

**Defense:** `defense-concern` · `defense-authorize` · `defense-action` · `defense-review` · `emergency-action` · `mutual-defense-invoke` · `mutual-defense-confirm` · `mutual-defense-refuse`

**Accountability:** `harm-claim` · `structural-examination` · `repair-proposal` · `repair-consent` · `separation` · `restoration` · `standing-review` · `standing-suspension` · `standing-restoration`

A group only encounters the entry types relevant to its activated functions. A group with no functions activated uses only Core entry types.

---

The `record` array is append-only. Automerge's CRDT semantics guarantee that entries added on different devices merge without conflict. The `signature` field on each entry lets any member verify that the action was taken by a real member, even without contacting a server.

The `deliberation_ref` field is optional. When present, it contains the content-addressed hash of a separate Automerge document that holds the full deliberation for that entry. See *Document Architecture* below.

`rules` and `members` are derived state: they can always be rebuilt by replaying the record from the beginning. They exist for convenience, not as source of truth.

The `functions` array in the group metadata records which functions (Stewardship, Federation, Provision, Defense, Accountability) the group has activated. Activating or deactivating a function is itself a record entry.

The `epoch`, `epoch_threshold_bytes`, and `prior_epochs` fields support the epoch system described in *Scaling and Epochs* below.

The entire document can be exported to human-readable Markdown or JSON at any time (Paper Principle).

---

## Functions

The v0.6 framework is modular. Every group starts with the Core. Beyond that, a group activates functions based on what it does. Each function brings additional values, rights, and implementation requirements — and corresponding record entry types.

### Stewardship — when a group manages shared resources

The group identifies the commons it stewards (what it is, its boundaries, who depends on it, what it can sustain, who maintains it). Stakeholders — including people outside the group — have standing to invoke the commons, pausing a decision and requiring the group to examine evidence that the decision exceeds what the resource can sustain. The software handles: recording commons identification, routing invocations, tracking stewardship roles, and recording contested-use resolutions.

### Federation — when groups relate to other groups

Groups can form voluntary federations (shared governance of shared concerns), commons-based federations (coordination scoped by a shared physical resource), compacts (bilateral promises with no governing body), or mutual defense compacts (coordinated response to threats of irreversible harm). Voluntary federations use iterative deliberation: voices carry positions between the federation and member groups across multiple rounds until consent emerges or the matter is set aside. Commons-based federations use stake-weighted voice (proportional to dependence on and impact on the resource) rather than one-group-one-voice. Standing decisions allow groups to pre-authorize positions on anticipated scenarios, enabling rapid coordination without sacrificing sovereignty. The software handles: voice designation and recall, deliberation rounds, consent tracking across groups, compact formation and records, standing decision management, stake declaration and assessment for commons-based federations, and mutual defense invocation workflows. Federation is the most architecturally significant function because it requires coordination *between* Automerge documents.

### Provision — when a group supplies essential needs

When a group provides shelter, food, water, energy, healthcare, livelihood, or safety, voluntary association is only voluntary if leaving is survivable. The group must acknowledge its provision, ensure genuine exit (transition support, portable resources, alternative mapping), and conduct periodic structural health checks. The software handles: recording provision acknowledgment, prompting health checks, maintaining alternative maps, and flagging when a member may lack alternatives.

### Defense — when irreversible harm must be prevented

The most dangerous and most constrained function. Activates in response to specific, identified threats of irreversible harm, and deactivates when the threat is resolved. Every defensive action requires community authorization (with specific expiry), is constrained by proportionality, and is subject to post-action review by people with no stake in the outcome. Emergency action — the only case where action precedes authorization — must be the minimum necessary and must be reported immediately. The software handles: concern routing, authorization with expiry tracking, action logging, mandatory review workflows, and role rotation tracking.

### Accountability — when harm must be addressed

Accountability responds to harm that has occurred — the space between soft mechanisms (reputation, exit) and defense (preventing irreversible harm in progress). The function begins with structural examination: what conditions produced the harm? It proceeds to repair: what does making it right look like? When voluntary repair is not forthcoming, the primary tool is structured separation — withdrawal of specific relationships and access, with defined duration, proportionality, and a restoration path. For confirmed floor violations, network-standing consequences apply: the finding is published, the group's Groupthink standing is suspended, and restoration is defined. The software handles: harm claim routing, structural examination workflows, repair tracking, separation management with expiry and renewal prompts, independent review coordination, network-standing publication, and restoration workflows.

### Function activation

A group activates a function through its normal decision process. The activation is recorded as a `function-activate` entry. From that point, the software surfaces the relevant workflows and entry types. Deactivation works the same way. A group's profile — the set of functions it has activated — is simply a description of what it does.

---

## Document Architecture

The framework requires that the record includes discussions — not just outcomes. The reasoning behind decisions, the concerns raised, the positions taken, and how disagreements were resolved are all part of the record (Documents/Core/0.Values.md § Toward maximum transparency; Documents/Core/1.Rights.md § The right to know).

But deliberation is large and unpredictable. A contentious decision in a 200-member group could produce hundreds of messages. If all of that lives in the main Automerge document, a few active discussions could consume an entire epoch.

The paper equivalent is instructive. A group that meets in person keeps minutes — a structured account of what was discussed, what concerns were raised, and what was decided. The full conversation happened in the room. The minutes capture its structure and substance. Both are part of the group's history, but they're not in the same book.

### Two document types

**The group document** is the main Automerge document described in the data model above. It contains the structured record — concise, well-typed entries that capture the substance of every action. For decisions, this means:

- Who proposed the discussion and what they proposed
- What concerns were raised (each concern is a named, attributed entry)
- How concerns were addressed (resolved, withdrawn, or led to modification)
- Who consented, who stood aside (with stand-aside reasoning)
- The outcome
- A `deliberation_ref` — the content-addressed hash of the full deliberation document

A structured decision entry is roughly 1–2 KB regardless of how much discussion happened. The group document stays lean. It syncs fast. It's always on every member's device.

**Deliberation documents** are separate Automerge documents, one per discussion. They contain the full conversation — every message, every response, the raw back-and-forth. They have their own append-only structure and their own signatures. They are linked from the group document by the `deliberation_ref` hash.

Deliberation documents are part of the record. They are commons. They are open to all members and to those affected by the decision. They are append-only and verifiable. But they are loaded on demand. A member reading the record sees the structured summary immediately. The full deliberation loads when they choose to open it.

### What this means in practice

- A member on a phone sees: "Discussion #47: Proposal to change meeting schedule. 3 concerns raised, all addressed. Consent reached, 2 stand-asides. [View full discussion]"
- Tapping "View full discussion" fetches the deliberation document — perhaps 200 KB of conversation.
- The 200 KB never bloated the group document. It never slowed sync. But it's always available.

### The honesty

This is a pragmatic separation. The framework says the record includes discussions. This architecture says "yes, and the full discussion is always available, but stored in a linked document rather than inline." This honours the spirit — transparency, accessibility, the right to understand — without making the system unusable. The software must make deliberation documents genuinely easy to access: one tap, no friction, no sense that the history is hidden.

---

## Scaling and Epochs

### The problem

Automerge documents work well up to roughly 10–50 MB. Beyond that, initial sync slows, mobile devices struggle, and the offline-first promise degrades. A casual group of 20 people with occasional membership changes will never hit this ceiling — a membership entry with vouching is roughly 2.3 KB with Automerge overhead, meaning a membership-only group could handle 4,000+ joins before reaching 10 MB. That group is fine for centuries.

But an active group that holds discussions, manages shared resources, and makes regular decisions can produce 5,000–50,000 entries per year. With deliberation documents stored separately, the group document stays leaner — but it will still grow. An active cooperative might reach 10 MB in 1–3 years. A large governance body could reach it in months.

### Epochs

An epoch is a closed, immutable segment of the group's record. When the group document crosses a size threshold, the current epoch closes and a new one begins.

**What an epoch contains:**

1. The record entries from that period.
2. A snapshot of derived state at the moment the epoch closed — members, rules, functions, circles, as they existed when the epoch ended.

The snapshot is critical. Without it, reconstructing current state would require replaying every prior epoch from the beginning. With it, the active document is self-contained: it has the checkpoint snapshot plus the entries since that checkpoint. Enough to derive current state without touching any archived epoch.

**How an epoch closes:**

1. The software detects that the active Automerge document has crossed the epoch threshold (default: 10 MB).
2. The group is notified. Depending on the group's rules, closure may require consent or may be pre-authorised automatically (a decision made when the group configures its rules).
3. The current derived state is snapshotted into an `epoch-close` record entry.
4. The closed epoch is finalised — its Automerge document is compacted and given a content-addressed hash.
5. The hash is recorded in the new active document's `prior_epochs` array.
6. The new active document starts from the snapshot, with a fresh record. This is Epoch N+1.

**Numbering:** Epoch 0 is the group's first epoch — the one created when the group is founded. Most groups will never leave Epoch 0.

### Archive stewardship

The group's record is a commons (Documents/Core/2.Implementation.md § The record is a commons). Archived epochs are part of that commons. The group is responsible for ensuring they remain available.

How archive storage works depends on the deployment tier:

- **Tier 1 (managed hosting):** the relay stores archived epochs. The platform operator is effectively a steward. The group can leave at any time because epochs are portable, immutable Automerge documents.
- **Tier 2 (self-hosted):** the group's own relay stores them. Whoever maintains the server is the archive steward.
- **Tier 3 (no server):** the group decides who stores archived epochs. Multiple members should hold redundant copies.

**Minimum redundancy:** every archived epoch must be held by at least N sources (where N is set by the group's rules, with a sensible default — 3). If the number of holders drops below N, the software flags it, similar to how provision health checks flag when alternatives are lacking.

**Any member can request any archived epoch from any source that holds it** — a relay, a peer, another member. Since epochs are immutable and content-addressed, any copy is verifiable. No source can tamper with an archived epoch without the hash changing.

### Splits and epochs

When a group splits, the splitting members take the full history — including all prior epochs. This is required by the framework: "They take a copy of their own record and rules" (Documents/Core/2.Implementation.md § Splitting), and the record is a commons that cannot be withheld.

In practice:

1. The new group receives references (content-addressed hashes) to all parent epochs, plus the right to access them from any source.
2. The new group's Epoch 0 starts fresh — with a snapshot of derived state at the split point and a `split` entry as its first record entry, referencing the parent group and the epoch/entry where the split occurred.
3. The new group does not need to store all parent epochs locally. They access them on demand. The right to access is unconditional.
4. Both groups can verify shared history — since archived epochs are immutable and content-addressed, either group can confirm their histories match up to the split point.

### Configurable thresholds

The epoch threshold is a method decision — groups decide how to implement it. The software provides:

- **Default: 10 MB.** Works on mobile, syncs quickly on modest connections.
- **Minimum: 1 MB.** Below this, epoch management overhead outweighs the benefit.
- **No maximum.** A group on dedicated infrastructure can set 100 MB or higher.

The threshold is stored in the group's rules and changeable through the normal decision process. The software should note the trade-off: a higher threshold means members need more capable devices to participate, which is a tension with the direction toward maximum equality.

---

## Sync and Hosting

Servers exist for convenience, not authority. They are relay points, like email servers. Useful when available, not required.

### How sync works

1. **When infrastructure is available**: servers relay Automerge changes between members efficiently. A member opens the app, their local document syncs with the relay, and changes from other members arrive. This is the normal, everyday experience.

2. **When infrastructure is limited**: two phones on the same WiFi network can sync directly. A laptop and a tablet can sync over Bluetooth. The sync protocol is the same regardless of transport.

3. **When infrastructure is hostile**: someone can carry the full document state on a USB drive, sync in person, or even transmit it over radio. The CRDT merges cleanly no matter how the data traveled.

### Deployment tiers

**Tier 1: Managed Hosting (easiest)**
A hosted service runs relay servers and provides a web UI. Members interact through the app. Under the hood, their local Automerge document syncs with the relay. Any member can export the full record at any time. This is how most groups will start.

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
3. Every action you take (raising a discussion, consenting, standing aside) is signed with your private key. Any member can verify it was really you.
4. You can attach a human-readable name to your key, or not.

### Why this matters

If a server is seized, there is no membership list to capture. The record contains actions taken by public keys. "Public key a7f3b2... consented to Discussion 12" is meaningless unless an adversary can link that key to a physical person. And since keys are different per group, compromising one group's membership reveals nothing about another.

### The Sybil problem

One person creating many fake identities is solved socially: existing members vouch for new members. This is exactly how Core membership works — a current member vouches for a new person and attests that they are a distinct individual not already a member under another name (Documents/Core/2.Implementation.md § Membership). A larger organization might require multiple existing members to co-sign a new membership credential.

### Why not zero-knowledge proofs?

ZK is genuinely hard to implement correctly, the tooling is immature, and the computational overhead is significant. Pseudonymous keypairs give you 90% of the privacy benefit with 10% of the complexity. If ZK tooling matures, it can be layered on later without changing the architecture.

### Why not real-name identity with encryption?

Encryption protects data in transit and at rest, but not against a compromised server operator or a legal order to decrypt. Pseudonymity means there's nothing meaningful to decrypt.

---

## The Group Creation Flow

(Just a suggestion)

1. **Name your group** and invite founding members
2. **Pick a template** (or start from Core defaults):
   - Casual (friends, clubs): shorter notice periods, minimal overhead
   - Cooperative (worker-owned, shared resources): longer notice periods, extended notice for rule changes
   - Governance (neighborhood councils, associations): structured circles, longer discussion periods
   - Custom: start from raw Core and configure everything
3. **Activate functions**: does the group steward shared resources? Federate with other groups? Provide essentials? The wizard surfaces relevant functions based on the template, and the group can activate or deactivate them at any time.
4. **Review the rules** the template generates
5. **Founding consent**: all founding members discuss and consent to the initial rules. All founders must consent. *(Documents/Core/2.Implementation.md § Membership)*
6. **The group is live**

Templates are convenience. Every template produces a standard set of rules that the group can amend through the normal discussion-consent process.

---

## What the Software Does (and Doesn't Do)

**The software does:**
- Present discussions, consent status, and outcomes in a clean UI
- Track notice periods and discussion deadlines automatically
- Maintain the append-only record
- Allow full export of all data at any time
- Support circles as nested groups
- Handle the mechanics of splitting (copy record, create new group)
- Support function activation and deactivation (Stewardship, Federation, Provision, Defense)
- Surface function-specific workflows: invocations (Stewardship), iterative deliberation and voice management (Federation), structural health checks (Provision), defense authorization and review (Defense)
- Prompt periodic assessments where the framework requires them (provision health checks, stewardship capacity reviews, defense authorization expiry)
- Support compacts and federation membership between groups

**The software does not:**
- Make decisions for the group
- Prevent any action that could be done on paper
- Lock data behind accounts, paywalls, or proprietary formats
- Create dependencies that would break the group if the software disappeared
- Enforce floors — floors are human commitments; the software records claims, routes them to deliberation, and tracks outcomes

---

## Technology Stack (Proposed)

- **Sync engine**: Automerge (CRDT library). Each group is an Automerge document. Handles merge, conflict resolution, and change tracking.
- **Relay server**: Lightweight server (Node.js or Go) running automerge-repo for Tier 1/2. Relays changes between members. Stateless by design: if a relay dies, members still have the full document locally.
- **Frontend**: Progressive Web App (works on any device with a browser, installable, works offline). Automerge runs in the browser, so the app works without a network connection.
- **Local-first storage**: Each member's device holds the full Automerge document. IndexedDB in the browser, SQLite or filesystem for native apps.
- **Data format**: Automerge binary for sync efficiency, with JSON and Markdown exports for human reading (Paper Principle).
- **Authentication**: Ed25519 keypairs per group membership. Actions are signed locally before entering the document.
- **Transport**: Relay server (WebSocket) for normal use, with fallback to local network sync, Bluetooth, or file-based exchange. The transport is pluggable because Automerge doesn't care how changes arrive.
- **Hosting**: Docker container for self-hosted relays; managed relay instances for Tier 1.

---

## Open Questions

1. **Real-time vs. async**: Automerge supports both naturally. Live sync via WebSocket gives real-time updates; periodic sync works for low-connectivity environments. The question is how much UI complexity to invest in the real-time experience (presence indicators, typing notifications, etc.).

2. **Encryption**: Should records be encrypted at rest? The Paper Principle implies readability, but "any member can read it" doesn't mean "anyone in the world can read it." Group-level encryption with member keys would protect privacy while maintaining transparency within the group.

3. **Spam/abuse in managed hosting**: If Tier 1 is a public service, how do we prevent abuse without creating gatekeepers? Rate limiting and invite-based group creation are options that don't violate the Core values.

4. **Mobile**: A PWA covers most cases, but native apps for iOS/Android would improve notifications and offline support. Worth the development cost?

5. **Federation sync protocol**: The framework now defines a full Federation module (Documents/Federation/) with voices, iterative deliberation, compacts, and federations of federations. The technical question is how federation-level records relate to group-level records. Options: (a) a federation is itself an Automerge document that member groups sync with, (b) groups exchange signed messages through a relay without a shared document, (c) a hybrid where the federation has a shared record but groups maintain sovereign copies. This is the most architecturally significant open question.

6. **Group discovery**: Groups are self-declaring ("a group exists when its members say it does"). But federation and compacts require groups to find each other. How do groups discover one another without creating a central registry that becomes a gatekeeper? Options include distributed hash tables, relay-based directories, or simply out-of-band introductions.

7. **Economic model**: How is managed hosting sustained? Freemium (free for small groups, paid for large)? Donations? Cooperative ownership of the platform itself?

8. **Decision method plugins**: The framework deliberately leaves decision methods open — consent-based, majority voting, sortition, delegation. The software needs a strategy/plugin interface for decision processes, since "the group decides how to decide" creates a bootstrapping problem in code. What is the minimal decision engine that supports multiple methods?

9. **Provision dependency tracking**: When a group provides essentials, the framework requires structural health checks and alternative mapping. How much of this can the software automate vs. prompt? Can the system detect when a member's dependency on the group has increased (e.g., they've deactivated other group memberships)?

10. **Network-standing propagation**: How are network-standing consequences (floor violation findings, standing suspensions, restorations) propagated across groups? Options: (a) published in federation documents and relayed through existing sync, (b) a separate "standing registry" Automerge document shared across the network, (c) gossip protocol where findings propagate peer-to-peer. The standing registry option creates a potential central point of failure; the gossip option may be slow and inconsistent. This needs careful design.

11. **Commons-based federation findings distribution**: How are sustainability findings from a commons-based federation published to non-participating groups that draw from the same resource? These groups have standing (they depend on the resource) but are not syncing the federation document. Options include: direct notification through relays, publication to a resource-scoped document that any stakeholder can access, or reliance on social channels (which may be insufficient for urgent findings).

12. **Stake quantification**: In commons-based federations, voice is weighted by stake. How is stake quantified and kept honest? Self-declaration with challenge rights is the starting point (Documents/Stewardship/2.Implementation.md), but this is vulnerable to exaggeration. Options: peer attestation (other stakeholders confirm each other's declared stake), measurement-based verification (for physical resources where use can be metered), or periodic audits. The choice likely varies by resource type.

13. **Trust record visibility**: How does the software surface the positive incentive structure — making a group's trust record visible and meaningful to potential partners? A group's record is append-only and verifiable, but raw records are not easily digestible. Options: computed trust metrics (risky — who defines the algorithm?), curated summaries (who curates?), or simply making records searchable and letting groups evaluate each other directly. The framework's commitment to transparency suggests the last option, but usability may require some form of structured summary.

