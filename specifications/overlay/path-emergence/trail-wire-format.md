# The flood trail on the wire

The overlay carries its trail — the [signed hash chain](../../signed-hash-chain/signed-hash-chain.md) in its [key-omitted form](../../signed-hash-chain/without-embedded-keys/without-embedded-keys.md), with the out-edge as each [message](./out-edge-commitment.md) and node identities [recovered](./recovered-node-identity.md) from the signatures — as a byte buffer inside a flood packet. The chain fixes what gets _signed_; how the assembled links are _serialized_ to travel between nodes it leaves open, the same way it leaves the seed and the payload open. This document is the flood's choice for that serialization.

Only one of the two byte strings is ever the chain's business, and the distinction is the reason this document lives here rather than beside the chain. [Length-prefixed encoding](../../encodings/length-prefixed-encoding.md) pins the bytes fed _into_ $H$ and $\text{Sign}$ — the _signed input_ — and it must be generic and fixed once, or no two implementations would agree on what a signature covers. The _transmitted_ bytes are a different string and a different kind of decision: the signed input for a link is $(i, h_{i-1}, m_i)$ and cannot contain $\sigma_i$, while the block $B_i = (m_i, \sigma_i)$ on the wire must. Nothing forces a single transmitted form — a carrier could pack the links into JSON, a columnar frame, or the flat buffer below — so the chain leaves it to whoever moves the trail. That whoever is the flood.

## 1. Transmit only the underivable

The chain refuses to transmit what it can derive — the sequence number is a position, the running hash is a fold — and the buffer is the disciplined extension of that refusal. Exactly two things are sent; the rest is recomputed.

| carried on the wire | recomputed, never sent |
| --- | --- |
| the blocks $B_0, \ldots, B_n$ | sequence number $i$ — the block's position |
| the seed inputs, `targetID` and `D` | running hash $h_i$ — folded forward from $h_{-1}$ |
| | node ID $p_i$ — recovered from $\sigma_i$ |

Everything the reader needs bottoms out at $h_{-1}$, and $h_{-1} = H(\texttt{DOMAIN\_TAG}, \text{targetID}, D)$ [bottoms out](./seed-commitment.md) at `targetID` and `D` — the only inputs neither constant nor already in a block. Those two, and the blocks, are the buffer.

## 2. The buffer

$$
\text{buffer} = \underbrace{\text{LP}(\text{targetID}) \mathbin\Vert \text{LP}(D)}_{\text{seed inputs}} \mathbin\Vert \underbrace{\text{LP}(m_0) \mathbin\Vert \text{LP}(\sigma_0)}_{B_0} \mathbin\Vert \cdots \mathbin\Vert \underbrace{\text{LP}(m_n) \mathbin\Vert \text{LP}(\sigma_n)}_{B_n}
$$

Every field is length-prefixed per the [encoding](../../encodings/length-prefixed-encoding.md) — a `u32` big-endian length then the bytes — so each is self-delimiting whatever its width. The buffer is **flat**: the two seed fields, then the blocks as field-pairs, in seed-to-head order. There are no wrappers around the seed region or around a block; the reader knows the grammar — two seed fields, then $(m, \sigma)$ pairs — and finds every boundary from the prefixes alone.

- **`targetID`, `D`** — the seed inputs, from which the reader recomputes $h_{-1}$. They ride here, not in a separate envelope field, because the trail is ferried and cached away from the packet it was born in; a seed input that lived only in the envelope would be gone the moment the trail travels. These bytes are the **one canonical copy** — the flood's target-recognition and its remember-and-drop termination read `targetID` and `D` from here, so there is no second copy to disagree.
- **$m_i$** — the out-edge designator $P_i$ committed to. Variable-length, at the mercy of the medium's designator format, hence length-prefixed.
- **$\sigma_i$** — the [65-byte recoverable signature](../../crypto/recoverable-ecdsa-signature.md).

Appending a hop appends one $(\text{LP}(m), \text{LP}(\sigma))$ pair at the tail and touches nothing before it.

This uniform length-prefixed layout is the **one and only** wire form of a trail. There is no alternate, "tightened," or prefix-shed encoding to negotiate: every field — including the fixed-width `targetID` and each $\sigma_i$ — carries its `u32` prefix, so two implementations read and write exactly these bytes. Every field is under a signature or folded into the running hash (§4), so the whole buffer is tamper-evident end to end: change any byte and some link fails to recover or verify.

## 3. Reading it

One parsing primitive, from the [encoding](../../encodings/length-prefixed-encoding.md): read a `u32` length, take that many bytes, advance. Every field announces its length, so the read is strictly forward — no delimiters, no backtracking. A reader keeps a cursor and does one pass: read `targetID` and `D`, then read fields in pairs until the buffer is exhausted.

The pass is a parse and a verification at once, because a chain is sequential: block $i$'s signed input contains $h_{i-1}$, so it cannot be checked without folding everything before it. The cursor therefore advances in lockstep with a running-hash accumulator — read the seed inputs and set $h \leftarrow H(\texttt{DOMAIN\_TAG}, \text{targetID}, D)$; then per block, recover or verify against the current $h$ and fold $h \leftarrow H(i, h, m_i, \sigma_i)$ before the next. It streams: nothing but the accumulator, and whichever recovered identities the reader chooses to keep, need be retained. The flood RPC frames the buffer's length, so exhaustion is a well-defined stop.

## 4. From wire bytes to a verdict

For each block the reader holds the position $i$ and the accumulator $h_{i-1}$; it rebuilds the signed input $(i, h_{i-1}, m_i)$ — length-prefixed as the [encoding](../../encodings/length-prefixed-encoding.md) prescribes — and runs $\text{Recover}$ (or $\text{Vrfy}$, for a participant checking its own link), then folds $\sigma_i$ into $h_i$.

This commits the trail to the **running-hash** form: links chain through the accumulator $h_{i-1}$, not through $\sigma_{i-1}$. [Recovered node identity](./recovered-node-identity.md) writes the signed input the same way — $(i, h_{i-1}, m_i)$, the recovery reading of the [key-omitted chain](../../signed-hash-chain/without-embedded-keys/without-embedded-keys.md)'s $\text{Sign}(i, h_{i-1}, m_i)$ — so the two describe the same bytes.

## 5. A route is always one chain; a chain does not concatenate

The buffer above is one chain: one seed, one running hash, blocks appended at the tail. A **chain does not concatenate with another.** Every $\sigma_i$ commits, through $h_{i-1}$, to *this* chain's seed $h_{-1} = H(\texttt{DOMAIN\_TAG}, \text{targetID}, D)$, so a chain grown from a different seed carries an incompatible running-hash lineage: its blocks cannot be appended to the first and still verify, and there is no operation that welds two signed chains into one.

So a single chain spanning a route exists only when *one* flood traverses the whole of it, every hop signing against the same seed. There is no way to take a freshly-discovered `(origin → here)` chain and a remembered `(here → target)` fragment and present them as one route: they are chains under different seeds, and nothing welds them. This is the cryptographic fact behind the routing rule that **only the target ends a discovery** — a non-target node cannot hand anyone a route it did not itself traverse, so it [floods onward](./stateless/stateless.md) (narrowing its fan-out to a recognized out-neighbor if it can), and the route the origin eventually files and source-routes is always this one, whole, single-seed chain. It is also why [loop-checking](./cycle-detection.md) needs no mechanism: there is never a seam at which one route could re-enter another.

## 6. Notes on design

- **Why this is an overlay document, not a chain one.** The chain fixes the signed input because the cryptography is defined over it; it cannot fix the transmitted form without also fixing things it deliberately leaves open — here, that the seed is `(targetID, D)` and the message is a designator. A generic serializer would have to treat the seed as an opaque, self-delimiting blob, because it could not know the seed's arity; the flood _does_ know it, which is exactly why the buffer above is flat — `targetID` then `D`, with no opaque wrapper. Concreteness is what removes the wrapper, and concreteness is the overlay's to supply. It is also why the seed-versus-block framing asymmetry a generic layout would carry simply is not here.
- **Robust to how the seed is minted.** The buffer fixes only that `targetID` and `D` ride in the preamble; whether `D` and $h_{-1}$ turn out to be the same object or distinct ones changes what those bytes _are_, not where they sit.
