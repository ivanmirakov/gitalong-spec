---
title: "gitalong-core-sync"
---

# Synchronization (Three-Pass Protocol)

To ensure maximum security and bandwidth efficiency, Gitalong uses a stateless three-pass sync. Let's say that Alice wants to syncronize data from Bob:

## Pass 1: Discovery (Bottom-Up)
**Direction:** Latest -> Common Ancestor
**Goal:** Identify the gap between Alice and Bob.
* Alice requests the parent of Bob's latest `node_hash`.
* Alice repeats this until she finds a `node_hash` already present in her local database.
* **Result:** A list of "missing" node hashes.

!!! note "Why?"
    Alice needs to know in advance the latest node they have in common.

#### Pass 2: The Skeleton (Top-Down)
**Direction:** Common Ancestor → Latest
**Goal:** Verify structural integrity (Orphan Protection).
* Alice requests the nodes in order, starting from the common ancestor's child.
* For each node, Alice verifies: `SHA256(parent_hash + content_hash) == node_hash`.
* **Security:** Because it is Top-Down, Alice confirms every node links to her trusted history.
* **Result:** The `nodes` table is updated; nodes are marked as `PENDING`.

!!! note "Why?"
    Going from top to bottom means that even if Bob is malicious or their connection cuts short, Alice will be able to hook up all of the nodes to her tree. It is resistant to orphan branch attacks.

## Pass 3: The Meat (Bottom-Up)

!!! warning "Considerations"
    This will most likely be replaced by a bulk download of content blobs, done in parallel and lazily by adding all "pending" nodes to a download queue which will be used to fetch content blobs as needed.

**Direction:** Latest → Common Ancestor
**Goal:** Data retrieval and Privacy (Redaction Awareness).
* Alice iterates through her `PENDING` nodes starting from the most recent.
* **Redaction Check:** If Alice's skeleton (from Pass 2) contains an `m.redact` node for a specific hash, she skips that message entirely.
* **Request:** For all other nodes, Alice requests the `content_blob`.
* **Result:** The `blobs` table is populated; nodes move from `PENDING` to `COMPLETE`.

!!! note "Why?"
    Going from top to bottom means that even if Bob is malicious or their connection cuts short, Alice will be able to hook up all of the nodes to her tree. It is resistant to orphan branch attacks.
