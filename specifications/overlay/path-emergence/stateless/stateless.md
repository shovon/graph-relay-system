# Path Emergence Logic: Stateless

This memo specifies a source-routed delivery discipline for a directed, strongly-connected network of autonomous nodes. Under it, the origin holds the complete route to a destination and stamps it onto every packet; intermediate nodes are pure forwarders that retain no routing state.

The network model it assumes:

- **Directed and strongly-connected.** Every node can reach every other node, but not necessarily by a symmetric link.
- **Forward-only sends.** A node addresses only its _out-neighbors_, each named by a local _out-edge designator_. It is never told its in-neighbors and can never reply directly to one.
- **Locally-scoped designators.** A designator is meaningful only to the node that holds it. The same value may appear at two unrelated nodes and point to different targets; a node that leaves and reconnects is issued a _new_ designator.
- **Globally unique node IDs**, derived independently of designators and not interchangeable with them.
- **No sender identity from the wire.** A receiver learns who sent a packet only by reading its contents.

Flood termination and duplicate suppression are specified in the [flood-safety memo](../cycle-detection.md) and referenced here, not re-derived. Route loops need no handling: a signed chain does not concatenate, so a discovered route is one flood's single chain and cannot revisit a node ([flood safety](../cycle-detection.md) §3).

---

## The axiom: the origin always carries the full path

A path is built end to end and **lives at the origin**. On every send, the origin stamps the complete route — the ordered list of `(node ID, out-designator)` pairs from itself to the target — into the packet header. Each hop reads the one designator that belongs to it, forwards, and advances a pointer. Nothing about forwarding requires an intermediate to remember anything.

The reason the origin must hold the _whole_ route, every time, is a deliberate worst-case assumption: **intermediate nodes are presumed not to remember the path.** Erring on the side of caution, the origin never relies on any downstream node retaining state. If an intermediate happens to recognize the way onward, that fact can only bias which out-neighbor it floods to (see below) — it can never be handed back as a route, because a signed chain does not concatenate. The origin's copy is the single authoritative copy.

This is the natural fit for locally-scoped designators. A trail is forward-only and each designator is interpretable only by its owner, so a source route works precisely because every hop reads its own entry in sequence. The origin embedding the full route is not a workaround; it is the most honest expression of the constraint.

### What the discipline buys and costs

- **Route state lives at the origin, and nowhere else.** Intermediates hold no forwarding state, so there is nothing in the interior of the network to go stale as the topology churns.
- **Staleness is concentrated and visible.** When a route dies, the whole route is dead at once, held in one place, and discovered on the next send — there is no silent, partial, hop-by-hop rot to chase down.
- **The header grows with path length.** Every packet carries the entire route. This is the price paid for stateless intermediates and is the main quantity to watch as paths lengthen.
- **The origin must be told when a route breaks**, because it is the only party holding the route and the only one that can refresh it.

---

## Flood Find

When the origin wants to deliver cargo to a target node ID it does not yet have a route for, it emits a [`send`](../../send.md) whose trail has **no blocks yet** — a preamble naming the target, and nothing more. With the route empty, the send is already at its frontier, so the origin floods the cargo to its out-neighbors, appending its `(node ID, out-designator)` entry to each copy. Every intermediate that is not the target is likewise at the frontier, so it appends its own entry and re-floods, and the route grows one pair per hop. A pure route discovery — no cargo to deliver, just a route to find — is the identical send with an empty payload. Flood termination — a discovery ID that each node remembers-and-drops on repeat, plus a hop-bound backstop `H_max` (a node-local ceiling on trail length, default 64) — is required for correctness on a cyclic graph and is specified in the [flood-safety memo](../cycle-detection.md), where `H_max` is committed and the discovery-ID rule remains its agenda.

### Recognizing the way: narrowing the flood, never splicing it

Only the target ends a discovery. A signed chain **cannot be concatenated**: every block signs the running hash of the whole prefix under one seed bound to `(targetID, D)` ([trail wire format](../trail-wire-format.md) §5), so a fresh `(origin → here)` chain and any remembered `(here → target)` fragment are chains under different seeds and cannot be welded into one. An intermediate can never hand anyone a route it did not itself traverse, so there is no such thing as a cache-answered route — a non-target node is never an answering node.

What a node's memory _can_ do is cheaper: bias its fan-out. A frontier node that recognizes one out-neighbor as lying on a known way to the target MAY append its block and re-emit on that edge alone, instead of flooding to all of them. It has not shortened the chain — it still signs and appends its own hop — it has only spent one send where a flood would have spent many, betting that the recognized edge reaches the target. If the bet is wrong or the recognition has gone stale, that copy dead-ends and is lost, and the origin re-discovers on its next send; the flood is the safe default a node falls back to whenever it does not, or no longer, trusts a shortcut. A run of such recognitions, hop after hop, collapses the flood into a de-facto unicast that still builds one clean signed chain, one real hop at a time, and terminates at the target exactly as a flood would.

### Returning the assembled route to the origin

The answering node delivers the complete assembled route back to the origin's node ID. This return is a _delivery_, not a reverse traversal: trails are forward-only, so if the answering node has no route to the origin, the return is itself a flood find. Budget the common case at two floods. The return terminates in one shot — its target is the origin, which is the endpoint. The answering node MAY cache the route home it discovers, so it can answer more cheaply next time; that installs nothing in the network's interior, and the design never depends on it.

### Filing the route at the origin

When the origin receives the assembled route, it simply files it in its own route cache against the target's node ID. There is **no hop-by-hop install**: nothing is written into the interior of the network, and there is no install walk that can break or partially complete. The next packet to that target carries the route in its header, and every intermediate forwards by reading its own entry.

---

## Forwarding a send: one pointer, two branches

The header carries the ordered route plus a position pointer, and a node forwards a `send` by reading that pointer against the number of route entries — the single rule that unifies source-routing and flooding:

- **Pointer inside the route (`pos < n`) — follow.** The node reads the current entry — its own `(node ID, out-designator)` — forwards along that designator, and advances the pointer so the next hop reads the next entry. This is the source-routed case below.
- **Pointer past the last entry (`pos == n`) — frontier.** The route is exhausted here. If the node is the target the send has arrived and is delivered; otherwise the route is still being built, so the node appends its own entry and floods onward — or, recognizing the way, re-emits to the one out-neighbor it trusts (the Flood Find case above). An unfinished route is the signal to flood forward.

The rest of this section is the follow branch.

1. Reads the current entry — its own `(node ID, out-designator)` — and forwards along that designator.
2. Advances the pointer so the next hop reads the next entry.

Node IDs in the header are not strictly required for plain forwarding (a node can just read "the next designator"), but they are carried because they enable the head-entry self-check — does the entry at `pos` recover to me? — and, via the origin recovered at the head of the trail, the flood's remember-and-drop key ([flood-safety memo](../cycle-detection.md), §2). Because the header carries _global_ node IDs, any hop can name any downstream node even though designators are local.

The concrete wire surface these mechanics compile to — the `send` RPC (which delivers along a known route and, when the route is short, floods to discover one) and the `return` RPC that carries an assembled route home — is specified in the companion stateless RPC profile.

---

## Broken Trail

A hop may find its out-designator dead — typically because the next node moved and was re-issued a designator, or because the edge itself failed. The handling here assumes failures are **loud**: a send on a dead designator returns a local error. If sends fail silently, no hop can detect a break and the only recovery is an origin-side timeout and re-find.

Given loud failures, there is one recovery, and it follows directly from the axiom that the origin owns the route.

### Re-find via the origin

The breaking node notifies the origin (a delivery to the origin's node ID — a flood find if it has no cached route there), the origin discards the stale route, re-runs flood find, and resends with a fresh full route. Intermediates stay dumb. The cost is a reverse flood plus a fresh find, but no interior state is touched and there is no partial-install hazard.

There is no in-place splice to repair the break locally. Prepending a fresh find toward the dead successor while keeping the rest of the route would join two chains under different seeds, and **a signed chain does not concatenate** ([trail wire format](../trail-wire-format.md) §5): the joined result would not verify. Origin re-find is the whole of broken-trail recovery.

---

## Notifying the origin

Because the origin is the sole holder of the route and the only party that can refresh it, notification is close to mandatory. Every notification is a delivery to the origin's node ID — a reverse flood in the worst case — so it carries a real cost, but the origin cannot function without it.

- **Break.** Notify. The origin cannot otherwise learn its cached route is dead, and an origin-side timeout alone wastes a full retry on a route already known to be broken.
- **Find succeeded (the normal case).** The return of the assembled route _is_ the notification; nothing additional is needed.

---

## State and overhead

- **Intermediate state: ideally none.** Pure forwarders. An intermediate may _optionally_ cache next-hop hints — "for target T, the way lies through out-neighbor D" — to narrow its flood fan-out, but the design never depends on it, and such caches are pure optimization that may be dropped at any time.
- **Origin state: one full route per destination it talks to.** Concentrated and visible. When it goes stale, the whole route is stale at once, discovered on the next send.
- **Header overhead: O(path length) per packet.** Every packet carries the entire route. This is the dominant cost and the main thing to watch as paths get long.

---

## Open issues and standing assumptions

- **Header size is the dominant cost.** Long paths mean large headers on every packet. If paths can be long, one mitigation is a hybrid that carries the full route only until the first successful delivery, after which intermediates _may_ cache and the origin _may_ fall back to a shorter destination-keyed header. That reintroduces distributed per-hop soft state and the silent staleness that comes with it, abandoning this design's central virtue — so adopt it only as a deliberate trade, not by accident.
- **The origin is a concentration point.** It holds and must refresh every route it uses. This is simpler to reason about than distributed soft state, but it makes the origin's route cache the thing that determines reachability.
- **Trust.** A route is a signed chain, so the origin *verifies* it — every hop [recovers](../recovered-node-identity.md) from a real signature — rather than trusting a bare claim: a malicious answerer cannot forge a route that recovers to nodes whose keys it does not hold. What signatures do **not** attest is *freshness* — an authentic route may be stale, caught only on use (broken-trail) — nor the *designators* a hop reads for forwarding, which it still takes on faith. On an untrusted network those two — staleness and header-designator trust — remain the application's to bound; the authenticity of route *structure* is the chain's.
- **Destination-side dedup.** Flooding during discovery (and retries) can deliver the same cargo to the target more than once, so a destination wanting at-most-once must deduplicate. The base `send` carries no origin ID and no message sequence on the wire, so it cannot supply a sound dedup key on its own — a plaintext origin would be spoofable, and the trail's `D` identifies a flood, not a message (a reused route shares one `D` across every message on it). The authenticated dedup key `(recovered author, seq)` is the [authenticated-delivery profile](../../authenticated-delivery.md)'s to provide, where both halves are signed; a deployment needing authenticated at-most-once adopts that profile, and one that does not should not assume the base gives it.
- **Termination and loops.** Flood termination and all cycle handling live in the [flood-safety memo](../cycle-detection.md) and are assumed available wherever referenced here. Loop-checking needs no mechanism: remember-and-drop makes each node join a flood once, so the single signed chain a discovery builds cannot revisit a node.
