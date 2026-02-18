# Gitalong

Gitalong is an opinionated decentralized chat protocol that uses a simple append-only Merkle table inspired by Git to keep a decentralized timeline of a chat, over the Gossip protocol.

In short:


| Protocol | Routing | Timeline / Conflict resolution | Event format | Network model |
|-|-|-|-|-|
| Gitalong | Gossip (Lightweight, stateless, lazy-loading) | Merkle DAG (Strict hash-of-hashes, 3-pass sync, minimalistic) | CJSON | Client <-> Client |
| Matrix | Full-mesh (Heavy, complex state-resolution) | Merkle DAG (State Resolution v2, very complex | JSON | Client <-> Server <-> Server <-> Client |

## Philosophy

We do NOT want to reinvent ANY wheel. Our protocol should (to the greatest possible extent) work on well-known, reliable technologies that already exist.

Our purpose is simply to put these technologies together in a clever enough way to make a single unified instrument.

If something we're doing already exists in the form of another technology, we WILL use that technology.

### The problem with alternatives

Although alternatives like are great in their own regard, they suffer from a plethora of issues.

Matrix, for example, tries too hard to "flatten" timelines into one-dimensional lines.

Distributed systems don't work like that. Even though you CAN spend hours resolving conflicts, it's not going to get you anywhere.

Other options very commonly don't even provide syncing logic.

## Room Storage (Merkle DAG)
Our Merkle DAG implementation focuses on structural integrity while allowing for data privacy. By separating the "Timeline" (Nodes) from the "Payload" (Blobs), we can delete message content without breaking the cryptographic chain.

### Overview

We maintain two distinct tables. The `nodes` table is the immutable skeleton of the chat, while the `blobs` table holds the potentially volatile message data.

**1. The `nodes` Table (The Ledger)**
This table is strictly **append-only**. It defines the sequence of events.
| Key | Type | Description |
|-|-|-|
| **hash** | SHA256 (PK) | Primary key. A hash of `contents_ref + parent_keys[]`
| **contents_ref** | SHA256 (FK) | A hash of the `contents` CJSON object, and a primary key to its corresponding element in the `blobs` table. |

**2. The `edges` Table (The Relations)**

| Key | Type | Description |
|-|-|-|
| **node_hash** | SHA256 (FK) | The hash of the child node. |
| **parent_hash** | SHA256 (FK) | The hash of the node being built upon. |

**3. The `blobs` Table (The Storage)**
This table stores the actual CJSON message bodies. To support redaction, rows in this table may be deleted.

| Key | Type | Description |
|-|-|-|
| **hash** | SHA256 (PK) | The hash resulting from SHA256(data) |
| **data** | CJSON | The data blob |

### Methods

#### Create
`create() -> [Ok(MerkleTree), Err(err)]`
Initializes a new MerkleTree instance.

#### Initialize
`initialize(MerkleTree) -> [Ok(root_hash), Err()]`
Creates the first node in a tree (e.g., for a new room). It generates a root `node` pointing to a room-initialization `blob`.

#### Insert
`insert(parent_hash, content_blob) -> [Ok(node_hash), Err(err)]`
To maintain the tree:
1.  **Hash Content**: Calculate `content_hash = SHA256(content_blob)`.
2.  **Store Data**: Save the blob into the `blobs` table using its hash as the PK.
3.  **Link Node**: Calculate `node_hash = SHA256(parent_hash + content_hash)`.
4.  **Commit**: Save the record to the `nodes` table.

> **Note on Salting:** To prevent hash collisions and ensure privacy, all `content_blob` payloads **MUST** include a `salt` field with random data. This ensures that identical messages result in unique hashes.

#### Redact
`redact(content_hash) -> Ok()`
Deletes the entry in the `blobs` table. This "hollows out" the message. Any client crawling the `nodes` table will still see the message's place in history, but will find the content missing (returning `null`).

#### Get
`get(node_hash) -> [Ok(Node + Blob), Err(err)]`
Retrieves the node and its associated blob. If the blob has been redacted, the implementation should return the node metadata with the data field set to `null`.

---

These are all of the methods that our Merkle tree needs in order to fulfill all of the requirements of Gitalong.

Not only is it simple, it would allow the implementation to have greater control unlike with Git.

It is imperative that the table remains append-only, with the only exception that elements in the `contents` column may be redacted, as it is not essential (TODO: investigate).


## The Room 
A room is just a collection of Merkle DAGs, synchronized (?) over the Gossip protocol.

### Rules

- It is IMPERATIVE that it remains **append-only**. Participants can only add new elements, never, ever change any prior ones (with few exceptions). This is in order to avoid conflicts altogether. *Any commit that changes history **must** be rejected.*
- Changes must follow a set of rules in order to be accepted (sent by valid participant in the room, signed by sender, et cetera), though this isn't required for the MVP.

### Conflict resolution

When a fork unavoidably happens (network momentarily splits in two, users send messages at the same time), we don't try to resolve it into a single unified timeline, instead we keep the timeline as a tree.

We condense it into a "linear" timeline by choosing a branch based on a simple algorithm: we simply choose the longer side of the fork. Not because this is the "correct" side of history, but because it is simple. The other side will be collapsed behind a "thread-like" button.

In the case that there is a fork of branches with identical length, we will perform a simple tie-breaker: favor the branch who's first node has a lower alphanumeric hash. If the two hashes are equal, may God help us because they wouldn't even be different branches.

The other, shorter side of the fork is simply collapsed into a thread (client-side). We don't want to put them both in a single timeline, because they just didn't happen in one.

In a nutshell: **determinism is king.**

|Scenario|Rule|
|-|-|
|Unequal branch lengths|Favor longer branch|
|Equal branch lengths|Favor branch with lower hash|

## Message
A message is just a new node appended to the latest one, in the `contents` field of the commit lives the actual message information in CJSON. For example:
```json
// Example commit 97993ee036
{
    "type": "m.text",
    "body": "Hello, world!",
    "time": 1770908889
}
```

### Types

#### m.text
A simple plaintext (?) message.

##### Fields
- `body`: The body of the text message.
- `time`: The time (in Unix Epoch) that the message was sent.

#### m.redact
**WARNING: OUT OF SCOPE FOR MVP**
- `hash`: The hash of the message we want to redact.

### Sending

Below is an example of how a node would send a message, given a Gossip network:

#### 1. Generation
It all starts when one node (let's call it Node A) wants to send a message (add a new node), this could also be a heartbeat to show it’s still alive, or a new update it just pulled from another neighbor who has just made a change to the repository.

#### 2. Selection (Gossip)
Alice now has a different hash for its latest node, so now it is "infective." Instead of telling everyone at once (which would be full-mesh and expensive), Alice picks a small, random set of neighbors from her list.

#### 3. Exchange
Alice sends an IHAVE message to its neighbors with the hash of the latest node. The neighbors will compare their latest hash with the hash that Alice has provided.

#### 4. Replication
Bob's latest hash is HASH_D, which is different from Alice's latest hash. He checks if Alice's version of the timeline is newer or older (older would mean the node hash already exists in his message history). If Alice's timeline is newer, he will sync his timeline with Alice's.

Refer to the **Synchronization** section.

#### 5. Spread
This is where the magic happens. Alice's neighbors now all have the message. In the next cycle, they themselves will each pick 3 random neighbors and share the message.

Eventually, Alice's timeline will be replicated across every member of the room.

## Synchronization (Three-Pass Protocol)

To ensure maximum security and bandwidth efficiency, Gitalong uses a stateless three-pass sync. Let's say that Alice wants to syncronize data from Bob:

### Pass 1: Discovery (Bottom-Up)
**Direction:** Latest -> Common Ancestor
**Goal:** Identify the gap between Alice and Bob.
* Alice requests the parent of Bob's latest `node_hash`.
* Alice repeats this until she finds a `node_hash` already present in her local database.
* **Result:** A list of "missing" node hashes.

##### Why: Alice needs to know in advance the latest node they have in common.

### Pass 2: The Skeleton (Top-Down)
**Direction:** Common Ancestor → Latest
**Goal:** Verify structural integrity (Orphan Protection).
* Alice requests the nodes in order, starting from the common ancestor's child.
* For each node, Alice verifies: `SHA256(parent_hash + content_hash) == node_hash`.
* **Security:** Because it is Top-Down, Alice confirms every node links to her trusted history.
* **Result:** The `nodes` table is updated; nodes are marked as `PENDING`.

###### Why: Going from top to bottom means that even if Bob is malicious or their connection cuts short, Alice will be able to hook up all of the nodes to her tree. It is resistant to orphan branch attacks.

### Pass 3: The Meat (Bottom-Up)
**Direction:** Latest → Common Ancestor
**Goal:** Data retrieval and Privacy (Redaction Awareness).
* Alice iterates through her `PENDING` nodes starting from the most recent.
* **Redaction Check:** If Alice's skeleton (from Pass 2) contains an `m.redact` node for a specific hash, she skips that message entirely.
* **Request:** For all other nodes, Alice requests the `content_blob`.
* **Result:** The `blobs` table is populated; nodes move from `PENDING` to `COMPLETE`.

###### Why: By going from bottom to top, Alice is able to notice changes in state before she receives the content. For example, she will be aware of message redactions before she reaches redacted messages. That way, if Bob claims that he can't provide contents because they are redacted, Alice will know that it's an error or a bug and be able to reject it.

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
