---
title: "Home"
---

# Gitalong

Gitalong is an **opinionated**, **truly distributed**, **non-linear** chat protocol of many ambitions.

## What makes it different?

We hate complexity. That's why Gitalong uses a simple append-only Merkle DAG inspired by Git to keep a decentralized timeline of a chat, over the Gossip protocol. Here is what Gitalong aims to offer:

- **Seamless distribution**: Our protocol is distributed, our users don't have to know.
- **Data sovereignty**: Your data should be yours, that's why we believe in a system where you have full control over it.
- **Right to be forgotten**: Most protocols fail to provide proper mechanisms for data redaction. Gitalong is designed with redaction in mind.
- **Resilience**: Embrace network splits, prevent spam before it happens.
- **Privacy**: Gitalong is designed with privacy in mind, using decentralized, peer-to-peer communication and strong cryptographic primitives.
- **Efficiency**: Transmit the least amount of data possible, allow for rooms to reach hundreds of participants while keeping a reasonable bandwidth usage, opting for an eventual-consistency model.
- **Extensibility**: Gitalong is designed to be extensible, allowing for easy integration of new features and protocols.

Here is a comparison table with Matrix:

| Protocol | Routing | Timeline / Conflict resolution | Event format | Networking model |
|-|-|-|-|-|
| Gitalong | Gossip (Lightweight, stateless, lazy-loading) | Merkle DAG (Strict hash-of-hashes, 3-pass sync, minimalistic) | CBOR | Peer <-> Peer |
| Matrix | Full-mesh (Heavy, complex state-resolution) | Merkle DAG (State Resolution v2, very complex | JSON | Client <-> Server <-> Server <-> Client |

## Is it just for chat?

Absolutely not! As a matter of fact, [chat is a protocol extension](ext/gitalong-ext-chat).

In a lower level, you could say that Gitalong is a distributed database. Though chat is the main objective, the possibilities are limitless.

---

We aren't here to reinvent wheels, that's why our protocol should (to the greatest possible extent) work on well-known, reliable technologies that already exist.

Our purpose is simply to put these technologies together in a clever enough way to make a single unified instrument.

If something we're doing already exists in the form of another technology, we *will* use that technology!

### Awesome! Where do I start?

* [gitalong-core-dag](spec/core/gitalong-core-dag) (Drafted): The "dumb" storage layer.
* [gitalong-core-room](spec/core/gitalong-core-room) (Drafting): The permission layer. How events are authored and validated in a room.
* [gitalong-core-sync](spec/core/gitalong-core-sync) (Drafting): The networking layer (how do you distribute events, how do you sync them).
* [gitalong-ext-chat](spec/core/gitalong-ext-chat) (Drafting): The Application Layer. Turning the room into an Instant Messenger (what's a message).
