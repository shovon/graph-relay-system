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

**The base case is out of scope.** The recurrence bottoms out at $\sigma_{-1}$: the seed value that $P_0$ chains its first link onto when it computes $\sigma_0 = \text{Sign}_{s_0}(\sigma_{-1}, E(L_0))$. What that seed actually _is_ — the empty string, a fixed domain-separation constant, a genesis digest, or a per-session nonce (see [§5](#5-caveat)) — is deliberately left unspecified here. The choice is not incidental: $\sigma_{-1}$ is precisely where freshness, domain separation, and anti-replay guarantees get bound into the chain, and the right value depends on what the carrying application demands rather than on the chain construction itself. This document therefore fixes only the _shape_ of the recurrence and treats $\sigma_{-1}$ as an opaque, externally supplied initial value; nailing it down is the job of a separate specification.

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

## 4. Notes on design

- **"Only the party that wrote $m_i$ needs to interpret it."** Correct, and it falls out naturally: $m_i$ is signed by $P_i$ alone. $P_{i + 1}$ does not need to understand it; from every other party's perspective it is an opaque commitment, simply carried along.
- **Self-certifying.** Because the ID is based on $p_i$, anyone can verify a link against its claimed ID without external trust. This is the Mazières/SFS sense of "self-certifying."
- **Append-only integrity.** Chaining via $\sigma_{i-1}$ inside each signature is what stops a malicious intermediate party from rewriting history, or an intermediary from reordering links. Each signature is a commitment to the _entire prefix_.

## 5. Caveat

This proves ordering and integrity, not freshness. Without a nonce or timestamp bound into the first link, a chain can be replayed wholesale. If replay matters to your application, fold a session nonce into the chain's seed — i.e., seed $\sigma_0$'s input with it instead of starting from the bare payload.
