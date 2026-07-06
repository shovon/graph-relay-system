# A self-certifying signed hash chain

This is a hash chain of signed links — a cryptographic, append-only chain. Each party that extends the chain appends its own self-certifying identifier together with an opaque payload it commits to. The result is a tamper-evident, verifiable record of exactly which parties extended the chain, in what order, and what each one committed to — requiring no certificate authority.

The chain is a primitive. It says nothing about what the payload _means_; that is the business of whatever application carries the chain.

## 1. Two types: keys in the link, or keys omitted

A party's self-certifying identifier is its public key $p_i$, or some derivation thereof, so that the ID _is_ the means of checking signatures from $P_i$. There are two choices about whether that key rides in the link, and they differ enough — on the wire, and in who can verify what — to be worth specifying as two distinct chains rather than a flag on one:

- **[Keys embedded in the chain](./with-embedded-keys/with-embedded-keys.md).** Every link carries $p_i$ explicitly. The verifier is handed the key and checks the signature against it. This is the default: it assumes nothing about the signature scheme beyond "you can verify a signature given a public key," so any scheme works — Ed25519 included — and every link is checkable by anyone, from the bytes alone.

- **[Keys omitted from the link](./without-embedded-keys/without-embedded-keys.md).** The link carries no $p_i$ at all; a party's identity is not transmitted. This is smaller on the wire. Its integrity rests on _self-recognition_: a party confirms its own link — and, transitively, the entire prefix it signed over — using only the key it already holds, which asks nothing of the signature scheme. What it does _not_ hand you for free is third-party readability. Whether someone who is _not_ a link's signer can check or name that link turns on whether the signature scheme lets an omitted key be reconstructed from a signature alone. Some schemes do — ECDSA over secp256k1 is the canonical case (it is how Ethereum derives an address from a key that is never transmitted; see [recoverable ECDSA](../crypto/recoverable-ecdsa-signature.md)) — and some, Ed25519 among them, do not. That reconstruction is a property of the _scheme_, applied by whatever application wants a fully readable chain; the chain construction itself neither performs it nor depends on it.

The two types share everything below this section — the payload's opacity, the encoding convention, the running-hash recurrence, the base case, and the freshness caveat. They differ only in whether $p_i$ rides in the link and, correspondingly, in who can verify: hand the embedded form to anyone and every link checks out, whereas the omitted form is checkable in full only by the chain's own participants — or by anyone, but only where the scheme permits reconstructing the missing keys. Because whether $p_i$ is present is part of the link's shape, the two produce different signed inputs and cannot be told apart from the bytes alone; a verifier must know, per link, which type it is decoding. The cheap and recommended stance is to fix one type for an entire chain — and, if an application genuinely needs to mix them, to signal the choice out of band, since the chain itself does not.

## 2. Shared primitives

Each party $P_i$ holds a keypair $(s_i, p_i)$ for an asymmetric signature scheme. Both types accept any such scheme for their own integrity; the omitted-key form can _additionally_ be read by non-participants when the scheme happens to support reconstructing a key from a signature, but it requires nothing of the scheme to let a participant verify its own link.

The payload $m_i$ is an opaque, application-defined byte string that $P_i$ commits to at its link. The chain treats it as bytes and never interprets it; its meaning is fixed entirely by the application. Whatever an application needs each party to attest to — a forwarding edge, a state transition, a timestamp, a document digest — rides here.

**The running hash.** Both types thread an accumulator $h_i$ through the chain: a digest folding in every link so far. Each signature commits to $h_{i-1}$ — the digest of the entire prefix — which is what makes the chain append-only and reorder-resistant. The sequence number $i$ is the link's position in the chain, not a transmitted field; it is bound into both the signature and the hash so that a link cannot be lifted to a different position. The exact tuple that goes under $\text{Sign}$ and $H$ is the one thing the two types differ on, and each type's document states its own.

**Encoding convention.** Throughout, $H$, $\text{Sign}$, $\text{Vrfy}$, and $\text{Recover}$ are _variadic over the individual components_, never over a single pre-concatenated string. The arguments are combined by an unambiguous, injective encoding — length-prefixed, or otherwise self-delimiting — so that distinct argument tuples never collide on the same input. [`../encodings/length-prefixed-encoding.md`](../encodings/length-prefixed-encoding.md) fixes that encoding concretely. Concretely, $H(\text{"AB"}, \text{"C"}) \ne H(\text{"A"}, \text{"BC"})$. Plain concatenation ($a \mathbin\Vert b$) does **not** qualify and must not be used: it lets an adversary slide bytes across a field boundary, yielding a different logical tuple with an identical hashed-or-signed input — defeating exactly the splicing- and reordering-resistance the chain exists to provide. That fixes the bytes fed _to_ the primitives. How the chain is then _serialized_ to travel — the transmitted bytes, a different string from this signed input — is the carrying application's to fix, like the seed and the payload; the chain only requires that a verifier can rebuild the signed input from whatever is transmitted.

## 3. The base case is out of scope

The recurrence bottoms out at $h_{-1}$: the seed value that $P_0$ chains its first link onto. What that seed actually _is_ — the empty string, a fixed domain-separation constant, a genesis digest, or a per-session nonce (see [§4](#4-caveat)) — is deliberately left unspecified here. The choice is not incidental: $h_{-1}$ is precisely where freshness, domain separation, and anti-replay guarantees get bound into the chain, and the right value depends on what the carrying application demands rather than on the chain construction itself. This document therefore fixes only the _shape_ of the recurrence and treats $h_{-1}$ as an opaque, externally supplied initial value; nailing it down is the job of a separate specification.

Each subsequent party $P_i$ (for $i \geq 1$) receives the chain $C_{i-1} = \big( B_0, B_1, \ldots, B_{i-1} \big)$ and appends its block $B_i$, yielding

$$
C_n = \big( B_0, B_1, \ldots, B_n \big).
$$

The exact shape of $B_i$ — and of the signed input that produces $\sigma_i$ — is the type's business; see [embedded keys](./with-embedded-keys/with-embedded-keys.md) or [keys omitted from the link](./without-embedded-keys/without-embedded-keys.md).

## 4. Caveat

This proves ordering and integrity, not freshness. Without a nonce or timestamp bound into the first link, a chain can be replayed wholesale. If replay matters to your application, fold a session nonce into the chain's seed — i.e., derive $h_{-1}$ from it instead of starting from a bare constant.

## 5. Notes on design

- **"Only the party that wrote $m_i$ needs to interpret it."** Correct, and it falls out naturally: $m_i$ is signed by $P_i$ alone. $P_{i+1}$ does not need to understand it; from every other party's perspective it is an opaque commitment, simply carried along.
- **Self-certifying.** Because the ID is based on $p_i$, a link can be checked against its ID without external trust. This is the Mazières/SFS sense of "self-certifying." The two types realize it differently — the embedded form hands every verifier the key to check against; the omitted form leaves a participant to check against the key it already holds, and a non-participant to reconstruct one only where the scheme allows — but both make the ID a function of the signature rather than an external attestation.
- **Append-only integrity.** Chaining via $h_{i-1}$ inside each signature is what stops a malicious intermediate party from rewriting history, or an intermediary from reordering links. Because $h_{i-1}$ is a running digest of everything before it, each signature is a commitment to the _entire prefix_.
