# Committing the out-edge as the chain's message

The [signed hash chain](../../signed-hash-chain/signed-hash-chain.md) leaves one thing deliberately open: what its message $m_i$ _is_. The chain treats $m_i$ as opaque bytes — the field each party commits to at its link — and fixes nothing about their meaning. This document fills that slot for the flooding overlay. It commits $m_i$ to a single, specific thing: the **out-edge designator** $P_i$ forwarded the message along. Nothing else about the chain changes; we are only saying what the party writes into the one field the chain hands to the application.

Everything the doc claims — that the verified chain reads as a route, that no hop can be rewritten or reordered — is downstream of that one commitment. Path provenance is the _consequence_ of committing the out-edge into $m_i$, not a separate mechanism.

## 1. What goes in $m_i$

The chain's abstractions map onto the overlay directly:

- **Party $P_i$** is a node the message traverses. Its self-certifying ID is the node's public key.
- **Message $m_i$** is the _out-edge designator_ chosen by $P_i$ — the edge along which $P_i$ forwarded the message to $P_{i+1}$. This is the field the chain treats as opaque bytes; committing it is this document's entire contribution. Here, those bytes name an edge.

So the commitment $P_i$ makes at its link is precise and local: _I forwarded along this out-edge._ It is not a claim about the whole route, nor about where the message goes next — just $P_i$'s own attestation of the edge it chose. As a message floods the graph, each node it reaches appends its own block $B_i = (L_i, \sigma_i)$ before forwarding, exactly as the chain prescribes. The append rule is untouched; only the message has acquired a meaning.

This uses the chain's [embedded-key type](../../signed-hash-chain/with-embedded-keys/with-embedded-keys.md), where each link $L_i = (p_i, m_i)$ carries the node's public key alongside the out-edge. For signature schemes whose keys are recoverable from a signature alone, the overlay can drop the public key from the link and let each node's identity be recovered instead — see the companion, [recovered node identity](./recovered-node-identity.md).

## 2. What the accumulated commitments prove

Run the chain's verification unchanged. If every link checks out, the ordered sequence of $(P_i, m_i)$ commitments — each party's committed out-edge, in order — reads as a route:

$$
P_0 \overset{m_0}{\longrightarrow} P_1 \overset{m_1}{\longrightarrow}\cdots\overset{m_{n-1}}{\longrightarrow}P_n
$$

Each $P_i$, in order, committed to forwarding along edge $m_i$. Because each signature commits to the entire prefix, no intermediate node can rewrite an earlier hop and no relay can reorder the sequence without invalidating a signature. The path is exactly the one recorded, and it is verifiable by anyone, with no certificate authority — the self-certifying IDs are their own trust anchor.

Note that $m_i$ matters only to $P_i$ itself: it is $P_i$'s own commitment to where it sent the message, opaque to $P_{i+1}$, which simply carries it along.

## 3. Replay

<!--
	TODO: this thing still feels superfluous, but AI assistants keep saying
	this is useful. Time will tell.
-->

Committing the out-edge proves the path was taken, not that it was taken _recently_. Inheriting the generic [caveat](../../signed-hash-chain/signed-hash-chain.md#4-caveat): without freshness bound into the first link, a whole chain of these commitments can be replayed onto a later flood.

If that matters for your flooding protocol, bind the chain's seed to something that sets _this_ flood apart from any other. The overlay does exactly that, and fixes _what_ concretely in a companion memo: [committing the flood's identity as the chain's seed](./seed-commitment.md) derives $h_{-1}$ from the flood's target and its discovery ID under a domain tag, so the first link commits to the flood it belongs to and that binding rides into every hop — deriving $h_{-1}$ from a per-flood value rather than a bare constant, the way the [omitted-key form](../../signed-hash-chain/without-embedded-keys/without-embedded-keys.md#6-caveat) prefixes freshness into the link itself. Notably the freshness need not be a dedicated envelope field — the seed is recomputed from identifiers the flood already carries — but the details are that memo's; here it is enough that once the first link commits to a per-flood value, a chain from one flood can no longer be presented as belonging to another.
