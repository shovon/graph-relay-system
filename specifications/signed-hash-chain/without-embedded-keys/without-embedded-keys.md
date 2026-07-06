# A signed hash chain with keys omitted from the link

This is one of the two [signed hash chain](../signed-hash-chain.md) types. Here the link carries no public key at all: a party's identity is simply not transmitted. A link is the payload it committed to and a signature over that payload and the whole prefix — nothing else. This is the smaller-on-the-wire form. For the other type — where $p_i$ rides in every link and a verifier is handed the key to check against — see [keys embedded in the chain](../with-embedded-keys/with-embedded-keys.md).

The distinction between the two types is exactly this and nothing more: **whether the public key is written into the link.** Everything else — the payload's opacity, the [encoding convention](../signed-hash-chain.md#2-shared-primitives), the running-hash recurrence, and the [base case](../signed-hash-chain.md#3-the-base-case-is-out-of-scope) — is the chain's shared machinery. This document fixes only what follows from leaving the key out.

## 1. Primitives

Each party $P_i$ holds a keypair $(s_i, p_i)$ for an asymmetric signature scheme. **Any scheme will do** — the construction assumes nothing beyond "sign with $s_i$, verify with $p_i$." A party's self-certifying identity is its key, or a derivation thereof, exactly as in the embedded form; the single difference is that this key is never written into the link and never travels beside the signature.

Leaving the key out has one immediate consequence, worth stating before the mechanics rather than after: a verifier holds no key for a link it did not sign, so from the bytes alone it cannot check such a link. Whether a non-participant can nonetheless reconstruct a party's key — and so verify or name a link it had no part in — is a question about the _signature scheme_, not about this construction, and it is out of scope here; the [parent's account of the two types](../signed-hash-chain.md#1-two-types-keys-in-the-link-or-keys-omitted) draws that boundary. What [§3](#3-verification) shows is that the construction's _own_ integrity guarantee asks for no such reconstruction at all.

## 2. The chain

At its core,

$$
B_i = (m_i, \sigma_i)
$$

given

$$
\sigma_i = \text{Sign}_{s_i}(i, h_{i - 1}, m_i)
$$

with

$$
h_i = H(i, h_{i - 1}, m_i, \sigma_i).
$$

Here $i$ is the sequence number — the position of $B_i$ in the chain, not a transmitted field. (The subscript on $\text{Sign}_{s_i}$ is $P_i$'s secret key, distinct from the sequence number.) Nothing in a link is redundant: the key is deliberately omitted, and neither the sequence number nor the hash is transmitted either — the first is the block's position, the second is rebuilt by walking forward from the seed. A link is therefore about as bare as a signed commitment gets: a payload, and a signature binding that payload to the entire chain before it.

## 3. Verification

Verification here is not "check every link." With the keys omitted, a verifier holds no means to check a link it did not sign — so the guarantee the construction offers is not universal but **participant-local**: a verifier is one of the chain's own parties, the holder of some $s_k$, and what it establishes is that its _own_ link came through intact. The surprise is how far that single check reaches.

The verifier walks from the seed, folding each block into the running hash as opaque bytes — the fold needs no key —

$$
h_i = H(i, h_{i - 1}, m_i, \sigma_i),
$$

rebuilding $h_{k-1}$, the prefix digest as it stood when $P_k$ signed. Then, at its own link and only there, it checks the one signature it has the key for:

$$
\text{Vrfy}_{p_{\text{self}}}\big(\sigma_k, (k, h_{k-1}, m_k)\big) \ \text{ and } \ m_k = m_{\text{self}}.
$$

There is no per-link identity to work out and no other signature to check — the fold takes no keys, and the verification takes only $p_{\text{self}}$, which the verifier already holds.

**Why that one check anchors the whole prefix.** $P_k$ signed $\sigma_k$ over $(k, h_{k-1}, m_k)$, and $h_{k-1}$ is the digest of the _entire prefix_. For a tampered chain to still verify under $p_{\text{self}}$ at position $k$ over $P_k$'s own message, an attacker would need a signature valid under $p_{\text{self}}$ over a _different_ prefix hash — which takes $s_k$, which it does not hold. So "my link still verifies, carrying my message" forces $h_{k-1}$ to be the value $P_k$ signed, which by collision resistance forces the whole prefix $B_0, \ldots, B_{k-1}$ to be the authentic one. **Self-recognition anchors the entire chain up to and including the verifier's own link — out of the single signature it already had to make.**

**And it anchors nothing past that link.** $\sigma_k$ commits to the prefix but says nothing about what follows it. An attacker can keep the authentic prefix and $P_k$'s link, truncate the tail, and append arbitrary links under keys of its choosing; $P_k$ holds no reference to weigh them against. Past its own link the verifier is, bluntly, out of luck — it cannot tell an authentic suffix from a forged one. The reach of self-recognition is exactly this: _prefix through my link, anchored; suffix, unknowable._

**What tampering produces.** Because the fold is over opaque bytes and each link chains the last, editing an earlier $m_j$ or reordering a block does not yield an _invalid_ chain — the hashes still chain, and bytes are still bytes. It yields a _different, internally consistent_ one. Nothing in the chain contradicts itself; the only thing that catches the edit is a party downstream of it finding that its own link no longer comes back as it signed it. Absent such a party, an omitted-key chain carries no internal grounds for rejection at all — detection is always some participant recognizing the corruption, or the absence, of _itself_.

The verified object, then, is narrow and honest. To $P_k$ it is a proof that the prefix it signed over stands exactly as it signed it, and that its own committed payload $m_k$ is intact — no more. It is not, at the construction level, a named route or a public ledger: the keys that would name the other parties are simply not on the wire. Any richer reading is the application's to build on top, with whatever its setting and its signature scheme afford it.

## 4. The tradeoff

Omitting the key buys a smaller wire: a link is a payload and a signature, and the identity the embedded form spends bytes on rides for free inside the signature instead. The price is specific, and it is the mirror image of that saving.

- **Verification is participant-local.** With no key in the link, the construction lets a party check only its own participation and the prefix behind it. It does not, on its own, make the chain checkable or readable by anyone else. Whether a non-participant can do more turns entirely on the signature scheme (see [§1](#1-primitives)) — a dependency the [embedded form](../with-embedded-keys/with-embedded-keys.md) does not carry.
- **No scheme is otherwise demanded.** Beyond that, this form asks nothing special: self-recognition works under any signature scheme the embedded form would accept. The choice between the two is therefore not "which scheme can I use" but "do I need every link checkable by anyone, or only each party checkable to itself."

## 5. Notes on design

- **"Only the party that wrote $m_i$ needs to interpret it."** Correct, and it falls out naturally: $m_i$ is signed by $P_i$ alone. $P_{i + 1}$ does not need to understand it; from every other party's perspective it is an opaque commitment, simply carried along.
- **Self-certifying, at its most literal for a participant.** There is no separate "claimed ID" field anywhere in the link, so nothing is transmitted for a verifier to weigh the signature against — and nothing to corroborate or contradict it. A participant checks a link against the one key it holds, its own; the ID is a pure function of the signature and the prefix it commits to. This is the Mazières/SFS sense of "self-certifying," sharpened by carrying no identity on the wire at all.
- **Append-only integrity.** Chaining via $h_{i-1}$ inside each signature is what stops a malicious intermediate party from rewriting history, or an intermediary from reordering links. Because $h_{i-1}$ is a running digest of everything before it, each signature is a commitment to the _entire prefix_ — which is exactly what lets a single self-recognition check reach back across all of it.

## 6. Caveat

Ordering and integrity, not freshness: see the [shared caveat](../signed-hash-chain.md#4-caveat). Without a nonce or timestamp bound into the first link, a chain can be replayed wholesale; if replay matters, derive $h_{-1}$ from a session nonce instead of a bare constant.
