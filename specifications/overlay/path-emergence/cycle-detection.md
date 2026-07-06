# Flood safety: termination, deduplication, and loop-freedom

This is the memo the path-emergence disciplines call the *cycle-detection memo*; **flood safety** is the better name for what it actually fixes, because only some of it is about cycles. A [`send`](../send.md) at its frontier re-emits to every out-neighbor, and on a strongly-connected cyclic graph that diverges without help. This memo fixes the rules that make it converge. It applies to both disciplines wherever they flood, and it supersedes the `drafts/cycle-detection.md` sketch.

Flood safety is three separate guarantees, and conflating them is what kept this a stub:

1. **Termination** — the flood halts. Primary: remember-and-drop on the flood's identity, `(recovered origin, targetID, D)`, which bounds total emissions to O(E) — committed in §2. Backstop: an absolute **hop bound `H_max`** — committed in §1.
2. **Deduplication** — no redundant processing. Within one flood, on that same identity (§2). Across floods and retries at the message level, on the authenticated `(author, seq)` key of the [authenticated-delivery profile](../authenticated-delivery.md).
3. **Loop-freedom** — a discovered route is acyclic. This needs no check: a signed chain cannot be concatenated ([trail wire format](./trail-wire-format.md) §5), so every route is a single chain built by one flood, and remember-and-drop (§2) makes each node append itself to that flood at most once. A route therefore never revisits a node, by construction — recorded in §3.

Sections 1–2 are the committed mechanisms; §3 records why loop-freedom is free; §4 (a size cap on flooded payloads) is the remaining work.

## 1. The hop bound `H_max` (committed)

`H_max` is the absolute, unconditional ceiling on how far a flood travels — the backstop for when remember-and-drop lapses (seen-set eviction), when the graph churns, or when a node misbehaves. It is the one termination mechanism that needs no global knowledge, no trust, and no expensive per-copy recovery.

### It is a node-local threshold read from the block count, not a decrementing field

Every `send` already carries its hop count: it is the trail's **block count** `n`, derivable from the [buffer](./trail-wire-format.md). So the check adds no wire field. It lives at the frontier, one clause on the forwarding rule:

```
pos == n  and  this node is not the target:
    if n >= H_max:  drop the flood copy      # too far — refuse to extend
    else:           append (m, σ), flood onward
```

This is deliberately **not** an IP-style TTL. A mutable counter decremented per hop is envelope state any hop can rewrite — and the fatal rewrite is trivial: reset it to the maximum and the flood lives forever. The block count cannot be gamed that way: the chain is append-only and each block commits to the prior hash, so **a node cannot make `n` smaller** to feign a younger flood. The worst it can do is *inflate* `n` by padding self-signed blocks, which only errs in the safe direction — killing a copy early rather than keeping it alive, and buying the attacker nothing. Reading the length is strictly better than carrying a TTL: same information, forgery-resistant in the one direction that matters, zero extra bytes. IP needs a TTL only because an IP packet has no self-describing hop history; our trail *is* that history.

### It is set per node, and the effective cap is the minimum along a path

Each node enforces its own `H_max`. A flood dies at the first node whose `H_max` its length reaches, so the effective bound on any path is the **minimum `H_max` among that path's nodes**. This is the right safety posture — any node may cap defensively, no node can force others to accept a longer flood. For uniform reachability a deployment SHOULD publish a shared `H_max` so nodes agree; a node MAY tighten it locally to defend itself.

### Recommended default: `H_max = 64`

The bound must exceed the graph's diameter with margin, and the diameter depends on the topology. For the bounded-degree, low-diameter graphs this design targets — a random `d`-regular graph has diameter ≈ `log_{d-1}(N)`, and the [degree-3 topology](../../topologies/degree-3-graph.md) gives ≈ `log₂(N)` — the diameter at planetary scale is small:

> ~8 × 10⁹ people, all connected: `log₂(8 × 10⁹) ≈ 33` hops between the two farthest nodes.

A flood or route need not take the shortest path, and churn adds hops, so double the diameter for headroom: ≈ 66, rounded to a clean **`H_max = 64`**. At full human population on a log-diameter topology that is roughly twice the graph's diameter — generous enough that it never severs honest traffic and only ever catches pathological or adversarial floods. That "never bites in the common case" is the point: `H_max` is insurance, not the primary terminator (§2 is).

**The value is coupled to the topology.** 64 assumes a `log₂(N)` diameter. A deployment on a high-diameter topology — a ring has diameter ≈ `N/2` — must size `H_max` to *its* diameter, or 64 will cut legitimate routes. The rule is `H_max ≈ 2 × expected diameter`; 64 is that rule evaluated for a log-diameter graph at human scale.

### Optional: an origin-chosen per-flood cap

An origin wanting a tighter blast radius than the global `H_max` binds its own maximum into the [seed](./seed-commitment.md), alongside `targetID` and `D`, so it folds into every signature and is immutable — no hop can raise it. Each frontier node then enforces `min(local H_max, origin cap)`. This gives origin-controlled scope with no mutable, forgeable field, exactly the discipline the seed already follows: bind it once, transmit nothing extra.

## 2. Remember-and-drop, keyed on the recovered origin (committed)

`H_max` guarantees a flood *stops*; it does not make it *cheap*. Without suppression, a frontier flood spawns one copy per simple path — exponentially many — each ≤ `H_max` hops but the swarm catastrophic. The primary terminator is **remember-and-drop**, which collapses the flood to O(E) by having each node forward a given flood at most once. This section fixes its key, and in doing so dissolves what looked like `D`'s hardest requirement.

### The seen-set key is `(recovered origin, targetID, D)`

Each node keeps a set of flood identities it has already forwarded. On receiving a flood copy it forms the identity, and:

```
key = ( recover(block 0), targetID, D )       # origin recovered from σ₀; targetID and D from the trail preamble
if key in seen:  drop                          # already forwarded this flood — do not re-emit
else:            add key to seen;  forward
```

The same key is the destination's within-one-flood dedup: the target drops repeat copies of the same flood by it. (Message-level at-most-once *across* floods and retries is a different key — the [authenticated-delivery profile](../authenticated-delivery.md)'s `(author, seq)` — because a reused cached route carries one fixed flood identity across many messages.)

### Why the *recovered* origin, and why that removes `D`'s unpredictability requirement

The obvious cheaper keys — raw `D`, or the seed `h₋₁ = H(DOMAIN_TAG, targetID, D)` — are **forgeable**. Both are functions of public-ish, guessable inputs (the tag is a constant, the target is known, and a predictable `D` completes them), so an attacker can *reproduce a victim's future key* and mint a flood carrying it, **signed by themselves**. Keyed on `D` or `h₋₁`, that flood pre-seeds every node's set, and the victim's genuine flood is then dropped as a duplicate — censorship without touching the victim. Defending that with an *unpredictable* `D` is possible but reintroduces exactly the dependence on unguessable randomness that [RFC 6979 deterministic signing](../../crypto/secp256k1-signature-profile.md) was chosen to remove.

Keying on the **recovered origin** closes it structurally. To pre-seed a censoring entry for victim V, an attacker would need a flood whose key is V's — which means a chain that *recovers to V's public key*. Nobody but V can produce that. An attacker's floods land under their own recovered key and never collide with V's. So **the pre-seeding attack is impossible regardless of how `D` is chosen**, and `D`'s only surviving requirement is that it be **unique among one origin's floods** — a plain per-origin counter suffices. The unforgeable ingredient has to be *in the key*, and the only node-verifiable unforgeable ingredient is the one the signature recovers.

`D` MUST therefore be unique per origin, and:

- a **counter** meets that trivially, but leaks the origin's flood volume to observers and MUST persist monotonically across restarts (reset to 1 and a still-remembered `(V, 1)` drops V's fresh flood);
- a **random** value is stateless and hides volume, and is now a *preference*, not a security requirement — a weak RNG can no longer make an origin censorable.

`D` need **not** be unpredictable. Its width is whatever supplies per-origin uniqueness (it rides length-prefixed, so no fixed width is imposed on the wire); a counter or ≥8 random bytes are both fine.

### The cost: one recovery per copy

Forming the key requires **recovering block 0** — one ECDSA `Recover` per flood copy at each hop — which plain forwarding does not otherwise need (folding the running hash is only hashing). That is the price of an unforgeable key, and it is the right trade: the overlay already treats [recovery](./recovered-node-identity.md) as first-class, the cost is O(1) per copy, and it buys the deletion of a security-critical randomness requirement. There is no cheaper option that is also censorship-resistant — anything cheaper than the recovered key is reproducible by an attacker.

### Retention is the one local dial

Seen-set entries are evicted **by age** `T`, and `T` MUST exceed the maximum flood lifetime — itself bounded by `H_max × worst-case per-hop latency` — or a slow-circulating copy re-arrives after eviction and re-amplifies (with `H_max` as the catch). `T` is local policy: a node with short retention wastes its *own* bandwidth and breaks nothing for anyone else, and under memory pressure a node may evict early and lean on `H_max`. A deployment SHOULD set `T` generously above `H_max × RTT`.

### The unifying principle

The recovered key is the **scoping principal everywhere**. Flood dedup keys on `(recovered origin, D)`; message dedup keys on `(recovered author, seq)`. In both, the unforgeable recovered key does the disambiguation, so the nonce beside it — `D` or `seq` — never needs to be more than a **per-principal counter**. No globally-unique, unpredictable value is required anywhere in the design; that is the same reasoning, applied twice.

## 3. Loop-freedom (committed — it falls out of §2 and the cryptography)

There is no loop-check to invent, because a signed chain cannot be spliced or concatenated ([trail wire format](./trail-wire-format.md) §5). That single fact settles it:

- **Every route is one chain.** A node that is not the target can never hand back a route; it can only forward the live flood, appending its own block. So the only route that ever exists is the single chain one flood builds by traversing the whole of it — there is no prefix-plus-fragment, no assembled route, nothing to weld and no seam that could revisit a node.
- **That one chain cannot loop.** Remember-and-drop (§2) makes a node forward a given flood at most once, so it appends itself at most once, so the chain never revisits a node. **The design has no route loops and needs no loop-check.**

The dangerous cycles — a flood circulating forever — are killed by §2 and, as a backstop, §1; they are a *termination* concern, not a route-structure one. A stale route discovered earlier is caught later on use, as a [broken trail](./stateless/stateless.md#broken-trail), not as a loop. There is simply no loop-checking mechanism in the design, because the cryptography leaves nothing for one to do.

## 4. The remaining agenda (not yet committed)

- **Payload-size cap on flooded sends.** Because [delivery folded discovery in](../send.md), the payload now rides the flood, so a large payload amplifies across O(E) edges. A frontier-send payload cap — above which a message must wait for a discovered route and travel unicast — is the one flood-safety parameter still to fix.
