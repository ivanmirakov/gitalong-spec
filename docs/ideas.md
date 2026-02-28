## Ideas

### 1. Must-have

* **Dual-Tree Architecture:** For chat, the protocol is split into two distinct Merkle DAG trees:
    * **State Tree:** A high-priority, low-volume tree containing the room information and "rules" (ACLs, Roles (?), Room Metadata).
    * **Timeline Tree:** A high-volume tree containing the actual "Meat" (chat messages, media references).


* **The State-Pointer:** Every node in a tree other than state (like the timeline) must contain a `state_ref` (hash) pointing to the specific State Tree node that was active when the message was created. Nodes that "downgrade" the state_ref will be rejected.

* **Sovereign Identity:** A user’s identity is an Ed25519 Private Key. The identity is separate from the node (a node is a "dumb" replicator). It should naturally be portable and shareable.

* **Identity vs Peer:** The **Identity** is the "user" (signer), while the **Peer** is a dumb actor (replicator). Nodes can host multiple identities.

* **State-Weighted Fork Choice:** In the event of a network split, peers will ALWAYS prioritize the branch with the **newest** state version should they be different, not just the longest chain. This prevents state-lag spam attacks.

### 2. Should-have

* **Founder Identity:** The room founder is the **root of trust**. A room's identity is tied to its founder's public key.

* **Transitive Trust:** Trust flows from the Founder to Admins via signed state changes. If you trust the Founder, you trust whoever they designate.

* **Rule of Strictest Rules:** Nodes assume a "safe" baseline (for example, room is read-only) unless a signed State Node explicitly changes that rule.

### 3. May-have

* **Lazy Loading:** Because of the tree split, clients can easily fetch state history (lightweight, important) and start chatting immediately, backfilling timeline (big, big) as needed.

* **Role-Based Access Control (RBAC):** Instead of power levels, users are assigned to role hashes (empty references).

* **Distributed Auditing:** Every peer independently validates every incoming node against the referenced ACLs. "Math" decides validity, not a central server.

* **ACL-Reference Logic:** ACLs (Read, Write, Redact) keep their own individual list of role hashes. For example, let's say we have a room for announcements:
    * Read: MEMBER_HASH, ADMIN_HASH
    * Write: ADMIN_HASH

### 4. Nice-to-have

* **Profiles as Rooms:** A user’s profile is its own room. Chat rooms only store a reference to this room through an m.profile.ref event appended to the state tree.

* **Hybrid-Federated Model:** Allow peers to put themselves behind domains. Rooms can set policies to exclude P2P peers.

* **The Bouncer:** A specialized peer that must co-sign nodes for them to be considered valid in public rooms.

* **Optional Proof-of-Work (PoW):** Defined by the State Tree. Difficulty can be dialed up during attacks to make spamming economically impossible.
q
