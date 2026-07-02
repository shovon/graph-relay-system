# A self-certifying signed hash chain

This is a hash chain of public keys with signed links — a cryptographic, append-only chain. Each party that extends the chain appends its own self-certifying identifier together with an opaque payload it commits to. The result is a tamper-evident, verifiable record of exactly which parties extended the chain, in what order, and what each one committed to — requiring no certificate authority.

The chain is a primitive. It says nothing about what the payload _means_; that is the business of whatever application carries the chain. For one such application, see [path provenance in the flooding overlay](../overlay/path-provenance.md), which instantiates the payload as a forwarding edge and reads the verified chain as a proof of the path a message took.

## 1. Primitives

Each party $P_i$ holds a keypair $\big( s_i, p_i \big)$ for an asymmetric signature scheme (e.g. Ed25519). The _self-certifying ID_ is the public key itself, or some derivation thereof, so that the ID _is_ the means of checking signatures from $P_i$.

The payload $m_i$ is an opaque, application-defined byte string that $P_i$ commits to at its link. The chain treats it as bytes and never interprets it; its meaning is fixed entirely by the application. Whatever an application needs each party to attest to — a forwarding edge, a state transition, a timestamp, a document digest — rides here.

**Encoding convention.** Throughout, $\text{Sign}$, and $\text{Vrfy}$ are _variadic over the individual components_, never over a single pre-concatenated string. The arguments are combined by an unambiguous, injective encoding — length-prefixed, or otherwise self-delimiting — so that distinct argument tuples never collide on the same input. [`length-prefixed-encoding.md`](./length-prefixed-encoding.md) fixes that encoding concretely. Concretely, $H(\text{"AB"}, \text{"C"}) \ne H(\text{"A"}, \text{"BC"})$. Plain concatenation ($a \mathbin\Vert b$) does **not** qualify and must not be used: it lets an adversary slide bytes across a field boundary, yielding a different logical tuple with an identical hashed-or-signed input — defeating exactly the splicing- and reordering-resistance the chain exists to provide. (In this construction $\sigma_{i-1}$ and $p_i$ are fixed-length and only the trailing $m_i$ varies, so naive concatenation happens to be unambiguous here — but $m_i$ is the one attacker-chosen field, so relying on that coincidence rather than stating the convention is a trap.)

## 2. The chain

At its core, we have

$$
L_i = (p_i, m_i), \qquad B_i=(L_i, \sigma_i)
$$

Given

$$
\sigma_i = \text{Sign}_{s_i}(\sigma_{i - 1},E(L_i))
$$

**The base case is out of scope.** The recurrence bottoms out at $\sigma_{-1}$: the seed value that $P_0$ chains its first link onto when it computes $\sigma_0 = \text{Sign}_{s_0}(\sigma_{-1}, E(L_0))$. What that seed actually _is_ — the empty string, a fixed domain-separation constant, a genesis digest, or a per-session nonce (see [§6](#6-caveat)) — is deliberately left unspecified here. The choice is not incidental: $\sigma_{-1}$ is precisely where freshness, domain separation, and anti-replay guarantees get bound into the chain, and the right value depends on what the carrying application demands rather than on the chain construction itself. This document therefore fixes only the _shape_ of the recurrence and treats $\sigma_{-1}$ as an opaque, externally supplied initial value; nailing it down is the job of a separate specification.

Each subsequent party $P_i$ (for $i \geq 1$) receives the chain $C_{i - 1} = \big( B_0,B_1,\ldots,B_{i - 1} \big)$ and appends $B_i$, yielding

$$
C_n = \big( B_0,B_1,\ldots,B_n \big)
$$

## 3. Verification

A verifier reconstructs each signed input from scratch and checks every link:

$$
\forall i : \quad \text{Vrfy}_{p_i}((\sigma_{i-1}, E(L_i)), \sigma_i) \overset{?}{=} 1
$$

If all checks pass, you have proof that each $P_i$, in order, committed to payload $m_i$ atop the entire chain that preceded it. The verified object is the ordered sequence of $(P_i, m_i)$ commitments — no more, no less. Any reading beyond that (this is a path, this is a ledger) is the application's to impose.

## 4. Alternative: omitting the public key when it is recoverable

The base construction carries $p_i$ explicitly inside every link. That is the right default, because it makes no assumption about the signature scheme beyond "you can verify a signature given a public key." But some schemes give you more than that: from the signature and the signed message alone, a verifier can _recover_ the public key that produced it. ECDSA with public-key recovery is the canonical example — it is how, for instance, secp256k1 signatures are handled on Ethereum, where addresses are derived from a key that is never transmitted, only recovered. For any such scheme, transmitting $p_i$ and committing to it is redundant work: the verifier already holds everything it needs to reconstruct the key. This section defines the variant that drops it.

The link and its signature become

$$
L_i = (m_i), \qquad B_i = (L_i, \sigma_i),
$$

with the recurrence unchanged in shape,

$$
\sigma_i = \text{Sign}_{s_i}(\sigma_{i - 1}, E(L_i)),
$$

except that $E(L_i)$ now encodes only the payload $m_i$ — the public key is nowhere in the signed input. Verification correspondingly recovers the key instead of being handed it:

$$
\forall i : \quad p_i = \text{Recover}\big(\sigma_i, (\sigma_{i-1}, E(L_i))\big),
$$

and the self-certifying ID of $P_i$ is the same derivation of $p_i$ as before — only now applied to the recovered key rather than a transmitted one. Everything downstream of "obtain $p_i$" is identical to [§3](#3-verification): the ordering and append-only guarantees come from $\sigma_{i-1}$ riding inside each signature exactly as in the base scheme, and they are untouched by where $p_i$ comes from.

**This is not merely an optimization; for a recovery-based verifier it is forced.** Look at the circularity the base scheme would otherwise impose. There, a verifier needs $p_i$ in hand to rebuild $E(L_i)$ before it can check the link. If $p_i$ is instead to be _recovered from_ the signature over that very message, then the message must not itself depend on $p_i$ — otherwise you would need the answer before you could pose the question. Omitting $p_i$ from $L_i$ is precisely what breaks that circle. So "recover the key" and "leave the key out of the payload" are not two independent choices you happen to make together; the first requires the second. This is why the variant is defined as its own scheme rather than offered as a flag on the base one.

**What you keep.** The property that matters most survives intact, and arguably sharpens. The recovered key _is_ the identity: $P_i$ is defined as whoever holds the key that this signature recovers to, with nothing transmitted alongside to corroborate or contradict it. That is self-certification in its most literal form — there is no separate "claimed ID" field that a verifier must check against the signature, because the ID is a pure function of the signature and the prefix it commits to.

**What it costs.**

- **A recovery-capable scheme is a hard prerequisite.** Ed25519 as standardly specified offers no public-key recovery; you cannot use this variant with it, and should keep $p_i$ in the payload per the base scheme. The variant is only available where a $\text{Recover}$ operation genuinely exists.
- **Signature malleability becomes an identity concern.** Some recoverable schemes are malleable — for ECDSA, $(r, s)$ and $(r, n - s)$ are both valid, and in general they recover _different_ keys, hence present as _different_ parties. Within the chain this is self-limiting: $\sigma_i$ is bound into $\sigma_{i+1}$, so tampering with a non-final link's signature invalidates the link that follows it. But an application that indexes, deduplicates, or otherwise keys links by their signature — or that treats the terminal link specially, since nothing chains over it — should pin down a canonical form (e.g. the low-$s$ normalization) so that a link maps to exactly one recovered identity.
- **Identity is known only _after_ checking.** There is no way to pre-filter "is this link from the party I expected?" without first recovering the key. This is no worse than the base scheme, which is already self-certifying rather than identity-directed, but it is worth stating: the variant does not give you an addressable ID up front.

**Mixing the two.** Whether $p_i$ is present is part of the link's shape, so a verifier must know, per link, which form it is decoding — the two produce different $E(L_i)$ and cannot be told apart from the bytes alone. The cheap and recommended stance is to fix one form for an entire chain. If an application genuinely needs to mix schemes across parties (some recover, some do not), the choice must be signaled out of band or in an application-level framing that the chain does not define; the chain itself only fixes the recurrence, and both forms instantiate the same recurrence.

## 5. Notes on design

- **"Only the party that wrote $m_i$ needs to interpret it."** Correct, and it falls out naturally: $m_i$ is signed by $P_i$ alone. $P_{i + 1}$ does not need to understand it; from every other party's perspective it is an opaque commitment, simply carried along.
- **Self-certifying.** Because the ID is based on $p_i$, anyone can verify a link against its claimed ID without external trust. This is the Mazières/SFS sense of "self-certifying."
- **Append-only integrity.** Chaining via $\sigma_{i-1}$ inside each signature is what stops a malicious intermediate party from rewriting history, or an intermediary from reordering links. Each signature is a commitment to the _entire prefix_.

## 6. Caveat

This proves ordering and integrity, not freshness. Without a nonce or timestamp bound into the first link, a chain can be replayed wholesale. If replay matters to your application, fold a session nonce into the chain's seed — i.e., seed $\sigma_0$'s input with it instead of starting from the bare payload.
