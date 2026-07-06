# A signed hash chain with embedded keys

This is one of the two [signed hash chain](../signed-hash-chain.md) types, and the default one. Every link carries the party's public key $p_i$ explicitly: a verifier is handed the key and checks the signature against it. This assumes nothing about the signature scheme beyond "you can verify a signature given a public key," so any scheme works — Ed25519 included. For the other type — where $p_i$ is dropped from the link entirely, saving its bytes on the wire at the cost of making the chain checkable in full only by its own participants — see [keys omitted from the link](../without-embedded-keys/without-embedded-keys.md).

## 1. Primitives

Each party $P_i$ holds a keypair $\big( s_i, p_i \big)$ for any asymmetric signature scheme (e.g. Ed25519). The _self-certifying ID_ is the public key itself, or some derivation thereof, so that the ID _is_ the means of checking signatures from $P_i$. Because the key travels in the link, any verifier — participant or not — checks a link against the key it was handed, with nothing to reconstruct from the signature and no dependency on the scheme supporting such reconstruction.

The payload $m_i$, the [encoding convention](../signed-hash-chain.md#2-shared-primitives), the running hash $h_i$, and the [base case](../signed-hash-chain.md#3-the-base-case-is-out-of-scope) are the chain's shared primitives; this document fixes only what is specific to the embedded form.

## 2. The chain

At its core, we have

$$
L_i = (p_i, m_i), \qquad B_i=(L_i, \sigma_i)
$$

Given

$$
\sigma_i = \text{Sign}_{s_i}(i, h_{i - 1}, p_i, m_i)
$$

With

$$
h_i = H(i, h_{i - 1}, p_i, m_i, \sigma_i)
$$

Where $i$ is the sequence number — the position of $B_i$ in the chain, not a transmitted field. (Note the subscript on $\text{Sign}_{s_i}$ is the secret key from $P_i$'s keypair $(s_i, p_i)$, distinct from the sequence number.) Unlike the [key-omitted form](../without-embedded-keys/without-embedded-keys.md), $p_i$ is present both in the transmitted link $L_i$ and inside the signed input — a party signs over its own public key, which is harmless and lets any verifier check the signature against the key it was handed.

Note that $B_i$ carries only $L_i$ and $\sigma_i$; the sequence number and hash are derived. The sequence number comes from the position that the block $B_i$ occupies in the chain, and the hash by walking forward from the seed. Neither is transmitted.

## 3. Verification

A verifier walks the chain from the seed, rebuilding the running hash as it goes. It reconstructs each signed input from scratch — trusting no transmitted sequence number or hash. The sequence number is the link's position $i$; the prefix hash $h_{i-1}$ is the value the verifier itself computed one step earlier, bottoming out at the externally supplied seed $h_{-1}$.

At each link it checks the signature against the link's own public key,

$$
\forall i : \quad \text{Vrfy}_{p_i}\big((i, h_{i-1}, p_i, m_i),\ \sigma_i\big) \overset{?}{=} 1,
$$

and then folds the block into the running hash before advancing to the next link,

$$
h_i = H(i, h_{i-1}, p_i, m_i, \sigma_i).
$$

The ID here is a _transmitted_ claim, yet the link is still self-certifying: the verifier checks the signature against that very claimed key, so a substituted or wrong key simply fails $\text{Vrfy}$ — there is no external attestation to trust. Integrity comes from the accumulator. Each $\sigma_i$ commits to $h_{i-1}$, a digest of the entire prefix; tamper with any earlier block and the verifier's recomputed $h_{i-1}$ no longer matches, so the following link's signature fails to verify. This is self-limiting in the usual way: it pins every link but the terminal one, which nothing chains over.

If every link verifies, you have proof that each $P_i$, in order, committed to payload $m_i$ atop the entire chain that preceded it. The verified object is the ordered sequence of $(P_i, m_i)$ commitments — no more, no less. Any reading beyond that (this is a path, this is a ledger) is the application's to impose.

## 4. Notes on design

- **Why keep $p_i$ in the link at all.** Carrying the key is the right default because it makes every link checkable — and nameable — by anyone, from the bytes alone, assuming nothing of the signature scheme beyond verifiability. The [key-omitted form](../without-embedded-keys/without-embedded-keys.md) saves those bytes, but its integrity is then self-recognition — a participant confirming its own link — and any third-party reading of the chain becomes contingent on the scheme's ability to reconstruct the missing key. When you want universal, scheme-independent verifiability, this is the form to reach for.
- **"Only the party that wrote $m_i$ needs to interpret it."** Correct, and it falls out naturally: $m_i$ is signed by $P_i$ alone. $P_{i+1}$ does not need to understand it; from every other party's perspective it is an opaque commitment, simply carried along.
- **Self-certifying.** Because the ID is based on $p_i$, anyone can verify a link against its claimed ID without external trust. This is the Mazières/SFS sense of "self-certifying."
- **Append-only integrity.** Chaining via $h_{i-1}$ inside each signature is what stops a malicious intermediate party from rewriting history, or an intermediary from reordering links. Because $h_{i-1}$ is a running digest of everything before it, each signature is a commitment to the _entire prefix_.

## 5. Caveat

Ordering and integrity, not freshness: see the [shared caveat](../signed-hash-chain.md#4-caveat). Without a nonce or timestamp bound into the first link, a chain can be replayed wholesale; if replay matters, derive $h_{-1}$ from a session nonce instead of a bare constant.
