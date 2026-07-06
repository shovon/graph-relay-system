# Delivery Binding: the `send` RPC on the wire

[Delivery: the `send` RPC](./send.md) fixes what delivery *means* — `send(trail, pos, payload)`, one primitive that both delivers along a known route and, when the route is not yet complete, floods to discover the rest. [Packet framing](./packet-framing.md) fixes that a packet is newline-delimited [message-shape](../encodings/json-array-message-shape.md) lines. This memo is the join of the two: the concrete line a `send` is.

The selector is `grs.overlay.v1/send`, a name in the overlay's reserved namespace ([interface profile](./interface-profile.md), §"The selector"). One line is one delivery.

## 1. The message

A `send` is a message-shape array of four elements:

```
[ "grs.overlay.v1/send", <b64-trail>, <pos>, <payload> ]
```

- element 0 — the selector.
- `<b64-trail>` (argument 0) — the [trail buffer](./path-emergence/trail-wire-format.md), unwrapped-base64 into a JSON string. The binary buffer is unchanged; base64 is only how a byte string rides in a JSON value, and it **MUST** be unwrapped so no line break lands mid-message ([packet framing](./packet-framing.md), §3). The signed bytes stay the buffer's; [length-prefixed encoding](../encodings/length-prefixed-encoding.md) remains the signing preimage, untouched by this carriage. The buffer's preamble carries the `targetID` and the discovery ID `D`, so the destination the send is aimed at rides *inside the trail itself*, never as a separate argument.
- `<pos>` (argument 1) — a non-negative integer, the source-route pointer: the index of the block whose hop acts next. Its relationship to the number of blocks is what the [stateless discipline](./path-emergence/stateless/stateless-rpc.md) reads as *follow* versus *frontier* (§2). The binding fixes the slot and its type; the discipline fixes its use — a stateful discipline MAY omit or reinterpret it.
- `<payload>` (argument 2) — the cargo, opaque to the overlay (§3). It **MAY** be empty, which carries no cargo and makes the send a pure route discovery — what a separate `build` RPC used to be.

There is no tagged destination arm. The earlier `target | trail` sum is gone: a send that knows only its destination carries a **zero-block trail** — a preamble naming `targetID`, with no hops yet — and `pos = 0`, which the discipline reads as "frontier at the origin," i.e. flood. A send that holds a complete route carries the full trail and `pos = 0`, which is read as "follow." Both are the same four-element line; nothing branches on shape.

## 2. `pos` against the block count is the whole dispatch

A receiver decodes the trail buffer far enough to know its **block count** `n` ([trail wire format](./path-emergence/trail-wire-format.md), §3), then branches on `pos`:

- **`pos < n` — follow.** Read block `pos`; its `(node ID, out-designator)` names this node and its next out-edge. Confirm the node ID recovers to this node (the head-entry self-check, [interface profile](./interface-profile.md), §"Addressing"), relay the whole line to that out-neighbor with `pos` incremented, and stop. A block `pos` whose node ID is not this node is a rotted route — the line is dropped, [per line](./interface-profile.md), and the discipline's break handling takes over.
- **`pos == n` — frontier.** The route is exhausted here. If this node is the `targetID` in the preamble, deliver `<payload>` up — unless it is empty, in which case there is nothing to deliver — and, if a route is being discovered, hand it home with a [`return`](./return-binding.md). Otherwise append this node's `(m, σ)` block to the buffer ([trail wire format](./path-emergence/trail-wire-format.md), §2), and re-emit — one line per out-edge, differentiated by the appended edge designator ([interface profile](./interface-profile.md), §"One flood is many packets") — each with `pos` set to the new block count. A node that recognizes one out-edge as leading toward `targetID` MAY instead re-emit on that edge alone, still appending its block; this skips the flood at the price of a dead end if the recognition is stale, and changes only the fan-out, never the appended chain. Appending is refused once `n` reaches the hop bound `H_max` ([flood safety](./path-emergence/cycle-detection.md), §1), the node-local ceiling that caps how far a flood travels.

`pos > n` cannot arise from a conformant sender and is dropped.

## 3. The payload is opaque

`<payload>` is application cargo, and the overlay does not interpret it ([message shape](../encodings/json-array-message-shape.md), §3.4). A payload that is naturally a JSON value MAY ride as that value directly; a payload that is not — binary application data — is unwrapped-base64 into a JSON value before it takes its position. Either way the overlay hands it up untouched at the target and never branches on it in transit. An empty payload — the JSON empty string `""` — carries no cargo and marks the send as pure discovery.

A `send` whose payload fails an application-level check has **its line** discarded, and any co-traveling RPC on another line is untouched — delivery drops at the granularity of the RPC ([packet framing](./packet-framing.md), §2; [interface profile](./interface-profile.md), §"Drop the RPC, not the packet").

## 4. Notes on design

- **One rule, no arm tag.** Modelling delivery and discovery as a single `send` whose behaviour turns on `pos` versus the block count means a receiver dispatches once, on one selector, and reads one integer against one count. The `target` case that used to need its own arm is just a trail that has not grown any blocks yet, so the tag it carried is redundant — the trail's own emptiness says the same thing. Fewer selectors, one forwarding predicate, and the `target` anomaly (a destination expressed as a bare ID rather than a degenerate trail) is gone rather than merely relabelled.
- **Why the pointer, and why its meaning is deferred.** Source routing cannot advance without a pointer, and the frontier test needs it too — "have we run off the end of the known route" is precisely `pos == n`. So the byte has to be on the wire. But *whether* a discipline reads it as a source-route cursor, and what "past the end" triggers, is the discipline's to fix; the binding reserves the slot and says only that it is a non-negative integer index.
- **base64, not a splayed trail.** The trail is a signed hash chain — a cryptographic object whose canonical form is [its buffer](./path-emergence/trail-wire-format.md). Exploding it into nested JSON would gain nothing and would invite an implementation to reserialize the signed bytes; carrying the whole buffer as one unwrapped-base64 string keeps the signed preimage in one place and the wire honest about what it is holding.
- **Authenticated delivery appends, it does not replace.** A deployment that wants the payload signed adds one trailing argument to this same line, under the append-only rule of the [message shape](../encodings/json-array-message-shape.md) (§6); the [authenticated-delivery profile](./authenticated-delivery.md) fixes it. A base receiver ignores the extra argument; nothing here changes.
