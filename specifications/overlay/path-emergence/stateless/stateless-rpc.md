# Path Emergence (Stateless): RPC Profile

This memo names the concrete RPCs the stateless path-emergence discipline runs on. It sits between two companions: the **interface profile**, which defines the packet-as-envelope and the per-RPC drop discipline relied on here, and **Path Emergence Logic: Stateless**, which defines the delivery discipline these RPCs carry out. It adds nothing to the routing model; it pins down the wire surface.

The discipline reduces to **two RPCs on the wire**, plus a third procedure that is a named use of one of them.

**`send(target | trail, payload)`** — deliver `payload`. The first argument is a sum, and which arm you hold decides how the packet travels:

- **`send(trail, payload)`** routes. The full `trail` sits in the header with a position pointer; each hop reads its own `(node ID, out-designator)` entry, forwards, and advances. `target` is implicit — the trail ends at it — so it is never named separately.
- **`send(target, payload)`** holds only a destination ID and no route to stamp, so it does not travel on its own. It rides with a discovery (below).

**`flood(target, trail)`** — discover a route to `target`. `trail` is the prefix accumulated so far; every hop appends its own entry and re-floods until a node is the target or holds a remembered suffix toward it. `target` is the destination the flood hunts, never a next hop: a flood names what it is looking for, not who carries it onward. Whatever discovery/dedup identifier termination needs rides here too — the plug-in point for the cycle-detection memo.

**`reportTrail(trail)`** — hand an assembled route back to the origin. This is not a third primitive. The assembled `trail` begins with the origin's own entry, so the answering node reads the origin's node ID straight off the head and calls `send(originOf(trail), payload = trail)` — a delivery to the origin carrying the route as its cargo. If it has no cached route to the origin, that `send` is itself a `flood`; budget two floods. The origin files the trail against the target's node ID. There is no hop-by-hop install.

## Sending without a route multiplexes two RPCs

When the origin holds only a `target`, the packet it emits to each out-neighbor carries two RPCs side by side: a `flood(target, trail)` to discover a route, and a `send(target, payload)` to deliver the cargo. They are not in tension — one says _find me a route for next time_, the other says _and get this there now_ — and per the interface profile they are evaluated independently at every hop. The discovery propagates; the delivery is satisfied when it reaches the target. (If the flood terminates early at an intermediate holding a suffix rather than at the target itself, that node completes the delivery by routing the payload along the suffix — `send(suffix, payload)` — while it reports the assembled trail home.) Either RPC can be dropped on its own: a payload that is malformed or fails a check has its `send` discarded while the `flood` keeps discovering; a discovery already seen is dropped under the dedup rule without that alone killing the send. The packet is only the envelope; the decision is made at the RPC.

The one pairing that never occurs is `flood` with `send(trail, …)`. Holding a finished trail while asking for one to be built is a contradiction, not a packet — it is the two arms of the `target | trail` sum, and a node is only ever in one of them.
