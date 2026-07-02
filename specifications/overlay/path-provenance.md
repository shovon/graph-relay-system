# Path provenance over the flooding overlay

This applies the generic [signed hash chain](../signed-hash-chain/signed-hash-chain.md) to one problem: proving the exact path a message took as it flooded the graph. The chain supplies the machinery — self-certifying IDs, signed links, append-only integrity, splice- and reorder-resistance. This document supplies the meaning: it says what the parties and the payload _are_, and what the verified chain lets you conclude about routing.

## 1. Instantiation

The chain's abstractions map onto the overlay directly:

- **Party $P_i$** is a node the message traverses. Its self-certifying ID is the node's public key.
- **Payload $m_i$** is the _out-edge designator_ chosen by $P_i$ — the edge along which $P_i$ forwarded the message to $P_{i+1}$. This is the field the chain treats as opaque bytes; here it names an edge.

As a message floods the graph, each node it reaches appends its own block $B_i = (L_i, \sigma_i)$ before forwarding, exactly as the chain prescribes. Nothing about the append rule changes; only the payload has acquired a meaning.

## 2. What a verified chain proves

Run the chain's verification unchanged. If every link checks out, the ordered sequence of $(P_i, m_i)$ commitments reads as a route:

$$
P_0 \overset{m_0}{\longrightarrow} P_1 \overset{m_1}{\longrightarrow}\cdots\overset{m_{n-1}}{\longrightarrow}P_n
$$

Each $P_i$, in order, committed to forwarding along edge $m_i$. Because each signature commits to the entire prefix, no intermediate node can rewrite an earlier hop and no relay can reorder the sequence without invalidating a signature. The path is exactly the one recorded, and it is verifiable by anyone, with no certificate authority — the self-certifying IDs are their own trust anchor.

Note that $m_i$ matters only to $P_i$ itself: it is $P_i$'s own attestation of where it sent the message, opaque to $P_{i+1}$, which simply carries it along.

## 3. Replay

The chain proves the path was taken, not that it was taken _recently_. Inheriting the generic [caveat](../signed-hash-chain/signed-hash-chain.md#6-caveat): without freshness bound into the first link, a whole provenance chain can be replayed onto a later flood. If that matters for your flooding protocol, seed the chain with a per-flood session nonce so a chain from one flood cannot be presented as belonging to another.
