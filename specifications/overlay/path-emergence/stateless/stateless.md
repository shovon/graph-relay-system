# Path Emergence Logic: Stateless

This memo specifies a source-routed delivery discipline for a directed, strongly-connected network of autonomous nodes. Under it, the origin holds the complete route to a destination and stamps it onto every packet; intermediate nodes are pure forwarders that retain no routing state.

The network model it assumes:

- **Directed and strongly-connected.** Every node can reach every other node, but not necessarily by a symmetric link.
- **Forward-only sends.** A node addresses only its _out-neighbors_, each named by a local _out-edge designator_. It is never told its in-neighbors and can never reply directly to one.
- **Locally-scoped designators.** A designator is meaningful only to the node that holds it. The same value may appear at two unrelated nodes and point to different targets; a node that leaves and reconnects is issued a _new_ designator.
- **Globally unique node IDs**, derived independently of designators and not interchangeable with them.
- **No sender identity from the wire.** A receiver learns who sent a packet only by reading its contents.

Cycle detection, flood termination, and the heuristic applied when a cycle is detected are specified in the cycle-detection memo and are referenced here, not re-derived. Wherever this document relies on duplicate suppression or loop-checking, it defers to that memo and notes the plug-in point.

---

## The axiom: the origin always carries the full path

A path is built end to end and **lives at the origin**. On every send, the origin stamps the complete route — the ordered list of `(node ID, out-designator)` pairs from itself to the target — into the packet header. Each hop reads the one designator that belongs to it, forwards, and advances a pointer. Nothing about forwarding requires an intermediate to remember anything.

The reason the origin must hold the _whole_ route, every time, is a deliberate worst-case assumption: **intermediate nodes are presumed not to remember the path.** Erring on the side of caution, the origin never relies on any downstream node retaining state. If an intermediate happens to know the remainder of the path, that fact is used only to _shorten discovery_ (see below) — never as a substitute for the origin carrying the route. The origin's copy is the single authoritative copy.

This is the natural fit for locally-scoped designators. A trail is forward-only and each designator is interpretable only by its owner, so a source route works precisely because every hop reads its own entry in sequence. The origin embedding the full route is not a workaround; it is the most honest expression of the constraint.

### What the discipline buys and costs

- **Route state lives at the origin, and nowhere else.** Intermediates hold no forwarding state, so there is nothing in the interior of the network to go stale as the topology churns.
- **Staleness is concentrated and visible.** When a route dies, the whole route is dead at once, held in one place, and discovered on the next send — there is no silent, partial, hop-by-hop rot to chase down.
- **The header grows with path length.** Every packet carries the entire route. This is the price paid for stateless intermediates and is the main quantity to watch as paths lengthen.
- **The origin must be told when a route breaks**, because it is the only party holding the route and the only one that can refresh it.

---

## Flood Find

When the origin wants to deliver cargo to a target node ID it does not yet have a route for, it floods the cargo to its out-neighbors, appending its `(node ID, out-designator)` entry to each copy. Every intermediate appends its own entry as it re-floods, so the route grows one pair per hop. Flood termination — a discovery ID that each node remembers-and-drops on repeat, plus a TTL backstop — is required for correctness on a cyclic graph and is specified in the cycle-detection memo.

### Early termination via a remembered suffix

If a node recognizes the target — it _is_ the target, or it still holds a remembered suffix toward the target — discovery stops there. This **answering node** assembles the full route: the discovered prefix `(origin → answerer)` followed by the suffix `(answerer → target)`. This is the only use of intermediate memory in the design, and it accelerates _discovery_ only, never _forwarding_. The assembled suffix may be stale; loop-checking the concatenated node-ID list and revalidating stale suffixes is the plug-in point for the cycle-detection memo.

### Returning the assembled route to the origin

The answering node delivers the complete assembled route back to the origin's node ID. This return is a _delivery_, not a reverse traversal: trails are forward-only, so if the answering node has no route to the origin, the return is itself a flood find. Budget the common case at two floods. The return terminates in one shot — its target is the origin, which is the endpoint — and should install origin-ward state as it travels, so the answering node can answer cheaply next time.

### Filing the route at the origin

When the origin receives the assembled route, it simply files it in its own route cache against the target's node ID. There is **no hop-by-hop install**: nothing is written into the interior of the network, and there is no install walk that can break or partially complete. The next packet to that target carries the route in its header, and every intermediate forwards by reading its own entry.

---

## Forwarding a source-routed packet

The header carries the ordered route plus a position pointer. A node that receives a source-routed packet:

1. Reads the current entry — its own `(node ID, out-designator)` — and forwards along that designator.
2. Advances the pointer so the next hop reads the next entry.

Node IDs in the header are not strictly required for plain forwarding (a node can just read "the next designator"), but they are carried because they enable two things: loop and duplicate validation (the plug-in point for the cycle memo), and **local repair targeting** (below). Because the header carries _global_ node IDs, any hop can name any downstream node even though designators are local.

The concrete wire surface these mechanics compile to — the `send`, `flood`, and `reportTrail` RPCs — is specified in the companion stateless RPC profile.

---

## Broken Trail

A hop may find its out-designator dead — typically because the next node moved and was re-issued a designator, or because the edge itself failed. The handling here assumes failures are **loud**: a send on a dead designator returns a local error. If sends fail silently, no hop can detect a break and the only recovery is an origin-side timeout and re-find.

Given loud failures, there are two strategies, differing in how much cleverness an intermediate is permitted.

### Default: re-find via the origin

Consistent with the axiom that the origin owns the path, the simplest handling is: the breaking node notifies the origin (a delivery to the origin's node ID — a flood find if it has no cached route there), the origin discards the stale route, re-runs flood find, and resends with a fresh full route. Intermediates stay dumb. The cost is a reverse flood plus a fresh find, but no interior state is touched and there is no partial-install hazard.

### Optimization: local splice (must still report to the origin)

Because the header carries the next hop's _node ID_, the breaking node X can attempt to repair in place: flood-find a fresh route to the successor's node ID, prepend it, and continue with the original suffix unchanged. This is sound **only when the break is the edge into the successor, not churn of the successor itself.** If the successor moved and was re-issued designators, the suffix entries it owns are also stale and the splice will deliver into a dead route — fall back to origin re-find.

Even when a local splice succeeds, the breaking node should still notify the origin so the origin can update its authoritative copy. Otherwise the invariant — _the origin always holds the current full path_ — is violated, and the origin will keep stamping the stale route on subsequent packets. The splice rescues the packet in flight; the notification preserves the design.

---

## Notifying the origin

Because the origin is the sole holder of the route and the only party that can refresh it, notification is close to mandatory. Every notification is a delivery to the origin's node ID — a reverse flood in the worst case — so it carries a real cost, but the origin cannot function without it.

- **Drop, or break with no repair.** Notify. The origin cannot otherwise learn its cached route is dead, and an origin-side timeout alone wastes a full retry on a route already known to be broken.
- **Local splice succeeded.** Notify anyway — not to recover the packet, but to keep the origin's authoritative route current (see above).
- **Find succeeded (the normal case).** The return of the assembled route _is_ the notification; nothing additional is needed.

---

## State and overhead

- **Intermediate state: ideally none.** Pure forwarders. An intermediate may _optionally_ cache suffixes to speed future discovery, but the design never depends on it, and such caches are pure optimization that may be dropped at any time.
- **Origin state: one full route per destination it talks to.** Concentrated and visible. When it goes stale, the whole route is stale at once, discovered on the next send.
- **Header overhead: O(path length) per packet.** Every packet carries the entire route. This is the dominant cost and the main thing to watch as paths get long.

---

## Open issues and standing assumptions

- **Header size is the dominant cost.** Long paths mean large headers on every packet. If paths can be long, one mitigation is a hybrid that carries the full route only until the first successful delivery, after which intermediates _may_ cache and the origin _may_ fall back to a shorter destination-keyed header. That reintroduces distributed per-hop soft state and the silent staleness that comes with it, abandoning this design's central virtue — so adopt it only as a deliberate trade, not by accident.
- **The origin is a concentration point.** It holds and must refresh every route it uses. This is simpler to reason about than distributed soft state, but it makes the origin's route cache the thing that determines reachability.
- **Trust.** The origin trusts any assembled suffix it is handed, and any hop trusts the header it is given. Source identity is still only what the packet claims; on an untrusted network, route entries and assembled suffixes need authentication, or a malicious answering node can hand the origin a poisoned route.
- **Destination-side dedup.** Flooding during discovery (and retries) can deliver duplicates; deduplicate at the destination on `(origin ID, message sequence)`.
- **Termination and loops.** Flood termination, the loop-check on remembered suffixes, and all cycle handling live in the cycle-detection memo and are assumed available wherever referenced here.
