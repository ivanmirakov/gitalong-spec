---
title: "gitalong-core-room"
---

# The Room 

## Overview

### What is it?

A room is a collection of trees in a Gitalong **Merkle DAG** working together to offer functionality.

### What does it do?

It is a local-only representation of a room. It defines interfaces for interacting with it, mainly appending events. It is completely network-agnostic and may even be used offline. A room implementation isn't very useful by itself, mainly serving as a building block for a higher-level implementation.

### How does it do it?

By using a Merkle DAG instance provided by a [gitalong-core-dag](gitalong-core-dag) implementation. A room implementation wraps around this instance and provides a higher-level API for interacting with the room.

A room begins with the `state` tree, acting as the "main" tree and source of truth for the room. Its job is to define everything related to the room (title, permissions, membership), room metadata, and metadata (room configuration, [power levels](../../ext/gitalong-ext-powerlevels)...).

Any other trees in the room are **fully optional**, and thus are required to specify a reference to the latest state tree node for each new node.

A chat room, for example, will have a state tree (what the room is, who joined, who is banned). This is the *main* tree. But additionally, it will have a **timeline tree** (the messages). Since the timeline tree is not a main tree, every node needs to hold a reference of the latest state node that the author knew of when appending it.

This approach makes Gitalong mostly resistant to state-lag attacks, where an attacker tries to manipulate the room state by appending events before the room has caught up with the latest state.

### Examples

#### Linear History (Simple)

--8<-- "includes/diagram-linear.mmd"

#### Forked History (Complex)

--8<-- "includes/diagram-forked.mmd"

### Rules

- It is IMPERATIVE that it remains **append-only**. Participants can only add new elements, never ever change any prior ones (with few exceptions). This is in order to avoid conflicts altogether. *Any commit that changes history **must** be rejected.*
- The referenced state node MUST be the same or newer than of the parent with the newest state node reference.
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

#### Message
A message is just a new node appended to the latest one, in the `contents` field of the commit lives the actual message information in CBOR. For example:

```json
// Example commit 97993ee036
{
    "type": "m.text",
    "body": "Hello, world!",
    "time": 1770908889
}
```

##### Types

###### m.text
A simple plaintext (?) message.

####### Fields
- `body`: The body of the text message.
- `time`: The time (in Unix Epoch) that the message was sent.

###### m.redact
**WARNING: OUT OF SCOPE FOR MVP**
- `hash`: The hash of the message we want to redact.

##### Sending

Below is an example of how a node would send a message, given a Gossip network:

###### 1. Generation
It all starts when one node (let's call it Node A) wants to send a message (add a new node), this could also be a heartbeat to show it’s still alive, or a new update it just pulled from another neighbor who has just made a change to the repository.

###### 2. Selection (Gossip)
Alice now has a different hash for its latest node, so now it is "infective." Instead of telling everyone at once (which would be full-mesh and expensive), Alice picks a small, random set of neighbors from her list.

###### 3. Exchange
Alice sends an IHAVE message to its neighbors with the hash of the latest node. The neighbors will compare their latest hash with the hash that Alice has provided.

###### 4. Replication
Bob's latest hash is HASH_D, which is different from Alice's latest hash. He checks if Alice's version of the timeline is newer or older (older would mean the node hash already exists in his message history). If Alice's timeline is newer, he will sync his timeline with Alice's.

Refer to the **Synchronization** section.

###### 5. Spread
This is where the magic happens. Alice's neighbors now all have the message. In the next cycle, they themselves will each pick 3 random neighbors and share the message.

Eventually, Alice's timeline will be replicated across every member of the room.
