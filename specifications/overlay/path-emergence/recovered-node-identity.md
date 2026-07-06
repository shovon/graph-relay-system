# Recovered node identity: dropping the public key from the link

This is a companion to [committing the out-edge as the chain's message](./out-edge-commitment.md). That document fixed the chain's message $m_i$ as the out-edge a node forwarded along, using the [embedded-key type](../../signed-hash-chain/with-embedded-keys/with-embedded-keys.md) — where every link $L_i = (p_i, m_i)$ carries the node's public key $p_i$ alongside the committed edge. This document commits the overlay to the chain's other type — [keys omitted from the link](../../signed-hash-chain/without-embedded-keys/without-embedded-keys.md), in which $p_i$ is **not** encoded in the link at all — and, because the overlay's signature scheme supports it, _recovers_ each node's identity from the signature. That recovery is the overlay's own addition: the chain construction gives each node self-recognition of its own hop, and recovering the omitted key is what lifts that into a route readable, and nameable, by anyone.

The commitment is narrow and worth stating plainly: for signature schemes where a verifier can recover the signing key from the signature alone, a node does not transmit its identity — it lets the identity be recovered from the signature it already had to produce. The out-edge is then the _whole_ of what a link carries.

## 1. What the link becomes

Under this variant the link sheds the public key, leaving only the committed out-edge:

$$
L_i = (m_i), \qquad B_i = (L_i, \sigma_i),
$$

with the recurrence unchanged in shape. Where the base commitment writes _(node's public key, out-edge)_ into each link, here a link is nothing but the out-edge designator $P_i$ forwarded along. The node's self-certifying ID is the same derivation of $p_i$ as before — only now $p_i$ is _recovered_ from the signature rather than read out of the link:

$$
\forall i : \quad p_i = \text{Recover}\big(\sigma_i, (i, h_{i-1}, m_i)\big).
$$

Everything downstream of "obtain $p_i$" is exactly the verification of the base commitment: the ordered $(P_i, m_i)$ sequence still reads as a route, and the append-only, no-reorder guarantees still come from the running hash $h_{i-1}$ riding inside each signature. The route reading is untouched; only where each node's identity _comes from_ has changed.

## 2. What this buys the overlay

The property that matters most survives, and arguably sharpens. The recovered key _is_ the node's identity: $P_i$ is defined as whoever holds the key that this hop's signature recovers to, with nothing transmitted alongside to corroborate or contradict it. There is no separate "this hop claims to be node X" field for a verifier to check against the signature — the identity is a pure function of the signature and the prefix it commits to. That is self-certification in its most literal form, and it costs the flood one fewer field per hop: a link on the wire is just the out-edge and a signature.

## 3. What it costs

The saving is not free, and the price is specific enough to state.

- **A recovery-capable scheme is a hard prerequisite.** This variant is only available where a $\text{Recover}$ operation genuinely exists — ECDSA over secp256k1 is the canonical case (it is how Ethereum derives addresses from a key that is never transmitted). Ed25519 as standardly specified offers no key recovery; an overlay built on it cannot use this variant and must keep $p_i$ in the link per the base commitment. Committing to "no public key in the link" is therefore also committing to a recovery-capable signature scheme; the two are one decision. The overlay's concrete choice is [ECDSA over secp256k1](../../crypto/secp256k1-signature-profile.md), the curve on which recovery is a first-class, battle-tested operation — with the mirror-image wrinkle that no platform speaks it, so the overlay bundles the curve and signs for itself (that doc's §4). Its on-wire form is [Bitcoin's 65-byte compact signature](../../crypto/recoverable-ecdsa-signature.md), which carries the recovery id inline.
- **Signature malleability becomes an identity concern — sharpest at the last hop.** Some recoverable schemes are malleable: for ECDSA, $(r, s)$ and $(r, n - s)$ both verify and in general recover _different_ keys — that is, present as _different_ nodes. Inside the chain this is self-limiting, because $\sigma_i$ is bound into $\sigma_{i+1}$, so tampering with any non-final hop's signature breaks the hop that follows it. But the **terminal hop** — the last node the flood reached — has nothing chaining over it, and an overlay that keys, deduplicates, or indexes hops by their signature should pin a canonical form (e.g. low-$s$ normalization) so that a hop maps to exactly one recovered node.
- **A hop's node is known only _after_ checking it.** There is no recovering the identity without first verifying the signature, so a receiver cannot pre-filter "is this hop from the neighbor I expected?" ahead of the recovery. This is no worse than the base commitment, which is already self-certifying rather than identity-directed, but it is worth saying: the variant gives no addressable node ID up front.

## 4. Mixing forms

Whether a link carries $p_i$ is part of its shape, not a detail a verifier can infer from the bytes — the two forms produce different $E(L_i)$ and are indistinguishable on the wire. The overlay's recommended stance is the chain's: fix one form for an entire flood chain. An overlay that genuinely needs some hops to recover and others to carry the key must signal the choice in an application-level framing the chain does not define; the chain itself only fixes the recurrence, and both forms instantiate the same one.
