# Committing the flood's identity as the chain's seed

The [signed hash chain](../../signed-hash-chain/signed-hash-chain.md) leaves two slots for the application to fill. One is the per-link message $m_i$; [committing the out-edge as the chain's message](./out-edge-commitment.md) fills it with the out-edge a node forwarded along. The other is the seed $h_{-1}$ — the value $P_0$ chains its first link onto — which the chain [holds out of scope on purpose](../../signed-hash-chain/signed-hash-chain.md#3-the-base-case-is-out-of-scope), because the seed is exactly where freshness, domain separation, and anti-replay get bound in, and the right value is the carrying application's to choose. This document is that choice for the flood. It is the companion to out-edge-commitment, filling the chain's other open slot.

The gap it closes was named in [out-edge-commitment §3](./out-edge-commitment.md#3-replay): a trail proves the path was taken, not that it was taken _recently_, so with a constant seed a whole chain can be replayed onto a later flood. The fix flagged there — seed the chain with a per-flood nonce — is what this memo commits concretely.

## 1. What goes in $h_{-1}$

The seed is the hash of three fields:

$$
h_{-1} = H\big(\texttt{DOMAIN\_TAG},\ \text{targetID},\ D\big),
$$

combined by the chain's [length-prefixed encoding](../../encodings/length-prefixed-encoding.md) like every other hashed tuple. Each field does one job:

- **`DOMAIN_TAG`** — a fixed protocol-and-version constant, committed here to the octets of the US-ASCII string `grs.overlay.v1/trail-seed` (25 bytes, no terminator; it enters the seed as one length-prefixed field like every other, so its length is carried and no delimiter is needed). Domain separation: the chain's signatures are meaningful only in this context, and cannot be lifted into any other use of the same signature scheme. The tag carries its protocol version (`v1`) for exactly this reason — a future revision that changes what a trail signature covers MUST mint a new tag, so that chains from two protocol versions can never be made to verify against each other.
- **`targetID`** — the node ID the flood is hunting. Semantic binding: every hop's signature now attests not merely "I forwarded" but "I forwarded _in the flood seeking target T_." A trail cannot be re-presented as a route toward a different target.
- **`D`** — the per-flood discovery ID. Freshness: it is what ties the whole chain to one flood instance rather than to the route in the abstract.

The origin is deliberately absent. The origin is $P_0$, and $\sigma_0$ already recovers to it; binding it into the seed as well would be redundant.

## 2. What seeding this way buys

Because the running hash folds $h_{-1}$ into every link, each signature commits to the seed transitively: whatever the seed binds, the whole chain binds. Every hop's signature therefore carries all three bindings at once —

- to the **flood**, through `D`: the trail belongs to one flood instance, not to the route in the abstract;
- to the **target**, through `targetID`: it is a route toward _this_ destination, and cannot be re-presented as a route toward another;
- to the **context**, through `DOMAIN_TAG`: it is valid only within this protocol, inert anywhere else.

It is worth being exact about where the replay boundary actually falls. Binding `D` does not, by itself, _reject_ a replayed trail: replayed verbatim with its original `D`, an old trail still verifies, because it is genuinely signed over that seed. What binding buys is that the chain's freshness becomes _exactly_ `D`'s freshness — an attacker cannot take a real old trail and dress it as a current flood by swapping in a fresh discovery ID, because every signature is welded to the `D` that seeded it: change `D` and the whole chain reconstructs differently, each hop [recovering](./recovered-node-identity.md) to a different identity, so the trail no longer names the nodes it claims to. Rejecting a stale-but-genuine `D` is the one remaining job, and §4 hands it to `D`'s owner.

## 3. It costs nothing on the wire

$h_{-1}$ is never transmitted. Every input to it is either a constant (`DOMAIN_TAG`) or a field the flood packet already carries for other reasons — the `targetID` a flood names so a receiver can recognize the destination it is hunting (the [envelope](../interface-profile.md) reserves both), and the discovery ID `D` it carries for remember-and-drop termination. Each node recomputes the seed from what it already holds. This is the chain's own discipline — the sequence number and the running hash are derived, not sent — extended to the seed: freshness for zero additional bytes.

## 4. The freshness contract

The anti-replay this memo provides reduces entirely to one property of `D`: the chain is exactly as fresh as its discovery ID. This memo therefore _requires_ of `D`, without minting it, only that it be **unique among one origin's floods** — so two of an origin's floods never share a seed.

It does **not** require `D` to be unpredictable. An earlier draft did, to defend against an attacker pre-seeding nodes' drop-sets with a guessed future `D`; the [flood-safety memo](./cycle-detection.md) (§2) closes that attack structurally instead, by keying remember-and-drop on `(recovered origin, targetID, D)` — an identity an attacker cannot reproduce for someone else's key, because it cannot sign as that origin — so a plain per-origin counter is safe. This keeps flooding consistent with the [signature profile's](../../crypto/secp256k1-signature-profile.md) choice of deterministic nonces: nowhere does the design depend on unguessable randomness. Minting `D`, keying the drop-set, and rejecting an already-seen one are the flood-safety memo's business; this memo states only the uniqueness requirement.

## 5. Notes on design

- **A flood is a flood.** The seed recipe is uniform across every flood — the forward flood ferrying cargo and the return flood ferrying a built-up trail compute $h_{-1}$ the identical way. There is no forward-versus-return discriminator in the seed, and deliberately so: two distinct floods already differ by their `D` (and by their target — the forward flood seeks the recipient, the return seeks the origin), and the trail a return flood carries rides as its _payload_, never spliced into its own trail. A role tag would separate nothing already separated, and would couple the generic flood-forwarding path to knowing which kind of flood it is in merely to compute a seed — the RPC-role coupling the envelope refuses, where one RPC's identity must not reach into how the packet is forwarded.
- **The dedup identity is not the seed.** $h_{-1}$ is constant across a flood's copies and distinct across floods, which once made it a candidate to double as the remember-and-drop key. The [flood-safety memo](./cycle-detection.md) (§2) took a different key — `(recovered origin, targetID, D)` — because $h_{-1}$, like raw `D`, is reproducible by an attacker (it hashes only a public tag, a known target, and `D`), whereas the recovered origin is not. So the seed and the flood's dedup identity stay distinct objects: the seed binds freshness into the signatures, and the dedup key adds the unforgeable origin that defeats pre-seeding. `D` is the ingredient common to both, and need only be a per-origin counter.
- **Freshness only.** Seeding this way binds a chain to one flood; it adds no guarantee the chain does not already provide. Ordering and integrity are the chain's, and the rest of the freshness story — that this defends against replay but not against a live flood in progress — inherits the [shared caveat](../../signed-hash-chain/signed-hash-chain.md#4-caveat).
