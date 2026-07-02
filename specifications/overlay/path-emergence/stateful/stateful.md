# Path Emergence Logic: Stateful

## Network model and assumptions

This protocol runs on a directed, strongly-connected graph of autonomous nodes under the following constraints:

- **Forward-only.** A node is given only its _out-neighbors_, addressed by a local _out-edge designator_. It is never told its in-neighbors and can never reply directly to one.
- **Locally-scoped designators.** An out-edge designator is meaningful only from the perspective of the node that holds it. The same designator value may appear at two unrelated nodes; it will never point to the same target. A designator is never reused for the same target twice — a node that leaves and reconnects is assigned a _new_ designator.
- **Two distinct identifier spaces.** Out-edge designators (per-node, local) are not node IDs. Every node also carries a globally unique node ID, derived independently of any designator.
- **No sender identity from the wire.** When a message arrives, the receiver cannot tell who sent it except by reading the packet contents.

Two consequences drive every design decision below, so they are stated up front:

1. **Trails are forward-only.** A trail is an accumulated list of `(node ID, local out-designator)` pairs. Each designator is interpreted only by the node that owns it, so the trail works as a forward source route _in the direction it was built_ — and only that direction. It cannot be walked backward: a node has no edge to, and no name for, the neighbor that sent to it.
2. **Every reply is a second routing problem.** Because trails are not reversible, "sending something back to the origin" is not a return hop. It is a from-scratch delivery to the origin's node ID, with the same cost profile as the original find. This is the single most important thing to budget for, and it is made explicit everywhere it occurs below rather than hidden behind phrases like "the regular channel."

Cycle detection, flood termination, and the heuristic applied when a cycle is detected are handled in the companion cycle-detection memo. Wherever this document relies on duplicate-suppression or loop-checking, it defers to that memo and notes the plug-in point.

---

## Flood Find

An originating node wants to deliver cargo to a target node ID but does not know which out-neighbor leads there.

It floods the cargo to all of its out-neighbors. To each copy it appends a trail entry: its own node ID paired with the out-edge designator used for that specific neighbor. Every intermediate node does the same as it re-floods, so the trail grows one `(node ID, designator)` pair per hop.

Flood termination (a discovery ID that each node remembers-and-drops on repeat, plus a hop/TTL backstop) is required for correctness on a cyclic graph and is specified in the cycle-detection memo. Without it the flood does not die.

Part-way through the flood, some node recognizes a trail _to_ the intended recipient — either it is the recipient, or it holds cached forwarding state for that destination. Call this the **answering node**.

### Returning the trail to the origin

The answering node sends the built-up trail back to the origin. **This return is itself a delivery to the origin's node ID**, not a reverse traversal of the trail. If the answering node has no cached route to the origin, the return is its own flood find. Two practical notes:

- The return terminates cleanly in one shot: its only goal is to reach the origin, and the origin is the endpoint, so there is no reply-to-the-reply.
- The return flood should install origin-ward forwarding state as it travels. One full exchange then caches both directions: origin → target _and_ answerer → origin. Subsequent traffic in either direction is cheap.

Plan for the common case to cost **two floods**, not one.

### Installing the path

When the origin receives a returned trail, it records the target node ID against the out-neighbor designator that begins the trail, then **forwards the trail along that trail** toward the answering node. Each intermediate node on the way records its own out-designator for the target as the trail passes. When the trail reaches the answering node, end-to-end forwarding state for the target exists at every hop.

Caveats on install:

- **Install is itself forwarding and can break mid-way.** If a hop on the install path has gone stale, apply the Broken Trail logic (below) recursively, and be prepared for a partially-installed path.
- **Cache-answered trails are concatenations.** When the answering node is a cached intermediate rather than the target itself, the delivered trail is `(freshly-found origin → answerer)` + `(answerer's older cached answerer → target)`. The older suffix may be stale, and the concatenation may revisit a node already on the new prefix. Loop-checking the concatenated node-ID list, and revalidating stale suffixes, is delegated to the cycle-detection memo; this is the point at which it must be invoked.
- **Multiple answers race.** Several nodes may recognize the target, so the origin may receive several trails. First-arrival (≈ lowest latency) is a fine default; keeping a runner-up as a backup route enables faster repair later.

---

## Broken Trail

An intermediate node may fail to resolve the out-neighbor for the target — a stale or changed designator, a node that moved and was re-issued a new designator, etc. Given the churn rule (reconnection yields a fresh designator), breaks are expected to be common, not rare.

### Precondition: do breaks fail loudly or silently?

Everything in this section assumes a node can _detect_ that it cannot resolve the out-neighbor — i.e., that sending on a dead or changed designator returns a local error. **This must be pinned down first, because it decides which approaches even exist:**

- If sends are fire-and-forget and a dead edge silently blackholes the packet, then no intermediate node can notice a break. "Flood find and repair" becomes impossible (you cannot repair what you cannot see), and the only surviving strategy is **Drop**, recovered by an end-to-end timeout and retry at the origin.
- If sends fail loudly, all three approaches below are available.

### The three approaches

#### Drop

The packet is lost; the recipient never receives it. Guaranteed delivery therefore requires an origin-side timeout and retry regardless of which approach is chosen, so this baseline is always present underneath.

#### Flood send

The intermediate floods the cargo onward. The recipient receives it (possibly in duplicate), but no new knowledge of a path to the recipient is established — nothing is cached, so the next packet pays the same cost.

#### Flood find and repair (then send)

The intermediate runs the same flood find the origin would, learning a fresh path to the recipient and caching it before sending. Only the repairing node learns the new path, which is fine: the origin already knows the out-neighbor that leads to this node, so the upstream portion of the route is unaffected.

This local-repair shortcut holds **only when the break is at or after the repairing node and the origin → repairer prefix is still intact.** If the break is earlier in the path, or the repairing node itself has moved and been re-issued a designator, the prefix is invalid and the failure must be kicked back to the origin for a full re-find.

---

## Notifying the origin

Whether a broken trail should notify the origin is a cost question, because — per the model above — **every notification is itself a delivery to the origin's node ID, i.e. a reverse flood**, and the notification can itself be lost.

- **Drop.** The origin already needs a timeout/retry for the drop case, so a notification here is an optional latency optimization, not a requirement. It speeds up the retry but adds a reverse-flood cost and can fail. Treat it as nice-to-have rather than the default.
- **Flood send without a find.** Worth notifying: the packet was delivered but no path was cached, so the origin is otherwise unaware that its route is degraded and will keep paying flood cost. The notification lets the origin trigger a proper find.
- **Flood find and repair.** No notification needed — the route is healed and the origin's prefix still works. Optionally inform the origin anyway if route-health visibility is wanted.

---

## Open issues and standing assumptions

- **Bounded state.** Partly handled by the network's own out-neighbor maintenance, partly not — the distinction is what the state is _keyed on_. The cached forwarding state is keyed on **destination node ID**, not on out-designator: "to reach target T, forward via D." When an out-neighbor disappears you do learn that _its designator D_ is gone (assuming failures are loud, per Broken Trail), so you can immediately evict every destination entry that forwards via D. That cleanly disposes of staleness caused by **your own out-edge dying**, and to that extent the reviewer is right: it falls out of out-neighbor maintenance for free, with no separate policy. What it does _not_ catch are the two cases the churn rule actually makes common:
  - **Downstream rot.** D is still your live out-neighbor, but a node further along the path moved and was re-issued a designator, breaking the route to T. Locally, D looks perfectly healthy and nothing disappears; the entry "T → D" is silently wrong. Your out-neighbor set gives you no signal, because the break is not on your edge.
  - **Departed targets.** T leaves or moves far away, but D stays alive serving other destinations. The designator never disappears, so nothing ever triggers removal of the now-dead T entry.

  The crux: the out-neighbor set is small and bounded, but the set of _destination IDs_ you have ever cached a route for is unbounded and grows monotonically. Out-neighbor-triggered eviction shrinks the first-hop-death class only. A staleness/TTL policy on destination entries is still needed on top of it — just less aggressively than the original "required, not optional" framing implied.

- **Destination-side dedup.** Flooding (and retries) can deliver the same cargo to the target multiple times. Deduplicate at the _destination_ on `(origin ID, message sequence)`, not only at relays.
- **Node ID generation.** With no coordinator, derive node IDs to be probabilistically unique without coordination — e.g. a sufficiently wide random value or a hash of a per-node public key.
- **Trust.** Source identity is only whatever the packet claims. A malicious node can forge an origin or poison a path-install. Acceptable on a trusted network; if the network is not trusted, authentication of trail entries and installs is required.
- **Termination and loops.** Flood termination and all cycle/loop handling (including the concatenation loop-check noted under Install) live in the cycle-detection memo and are assumed available wherever referenced here.
