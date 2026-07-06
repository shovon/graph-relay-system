# Path Emergence (Stateless): RPC Profile

This memo names the concrete RPCs the stateless path-emergence discipline runs on. It sits among three companions: the **[interface profile](../../interface-profile.md)**, which defines the packet-as-envelope and the per-RPC drop discipline relied on here; **[Delivery: the `send` RPC](../../send.md)**, which defines the delivery-and-discovery primitive this discipline realizes below; and **[Path Emergence Logic: Stateless](./stateless.md)**, which defines the delivery discipline these RPCs carry out. It adds nothing to the routing model; it pins down the wire surface.

The discipline reduces to **two selectors** — `send` and `return` — and statelessness is what fixes how each is read.

## `send` — deliver, and discover when the route is short

Statelessness reads a `send`'s pointer against its trail's block count, and that one comparison is the whole forwarding decision ([delivery binding](../../delivery-binding.md), §2):

- **`pos < n` (inside the blocks) — source-route.** The full route sits in the trail; the hop at `pos` reads its own `(node ID, out-designator)` entry, confirms the entry names it, forwards, and advances `pos`. Unicast along a known path.
- **`pos == n` (past the last block) — frontier.** The route is exhausted here. If this node is the `targetID` the preamble names, the send has arrived: it delivers the payload, unless the payload is empty, in which case discovery was the whole point and there is nothing to deliver. Otherwise the node appends its `(m, σ)` block and re-emits onward — by default to every out-neighbor, or, if it recognizes one out-neighbor as lying toward `targetID`, on that edge alone as a local optimization that skips the flood and risks a dead end if stale — each copy's `pos` set to the new block count.

So the stateless discipline needs **no distinct discovery RPC**. A first-contact delivery is a `send` whose trail has *no blocks yet*: `pos == 0 == n` at the origin, so it floods the cargo while the trail grows behind it, exactly as a pure discovery would — the two differ only in whether the payload is empty. The identifier flood termination keys on is the per-flood `D`, carried in the trail's seed preamble, not a separate field; the remember-and-drop rule over it — keyed on `(recovered origin, targetID, D)` — is committed in the [flood-safety memo](../cycle-detection.md) (§2).

## `return` — hand an assembled route home

**`return(addr-trail, pos, cargo-trail)`** — deliver a finished route back to the origin. It is addressed like a `send` and read by the same `pos`-against-`n` rule ([return binding](../../return-binding.md)): the answering node reads the origin's node ID off the head of the `cargo-trail`, seeds an `addr-trail` toward it, and source-routes home if it holds a route there or floods (a zero-block `addr-trail`, `pos = 0`) if it does not — budget two floods. When the return reaches the origin, the origin files the `cargo-trail` against its target's node ID. There is no hop-by-hop install.

`return` keeps its own selector only because its terminal act differs — file a route, rather than deliver cargo up; everything about how it *travels* is a `send`.

## What is gone

The earlier profile listed a separate `build` RPC and a `send` split into `target` and `trail` arms that had to be multiplexed side by side in one packet. Both are gone. Discovery is a `send` with a short trail; the `target` arm is a `send` with a zero-block trail; the packet that once carried a `build` beside a `send(target, …)` is now a single `send`. One selector for delivery-and-discovery, one for the route home.
