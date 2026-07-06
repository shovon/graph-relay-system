# Return Binding: the `return` RPC on the wire

[Delivery: the `send` RPC](./send.md) folds discovery into delivery: a send whose trail is not yet complete floods to build it. What remains genuinely separate is handing a *finished* route back to the origin that asked for it — because the receiver does something delivery-specific with it: it **files the trail against the target's node ID** rather than handing opaque cargo up to an application. That is `return`, and this memo fixes its line. It is the companion to the [delivery binding](./delivery-binding.md), which covers `send`.

The selector is `grs.overlay.v1/return`, a name in the overlay's reserved namespace ([interface profile](./interface-profile.md), §"The selector").

## 1. `return` mirrors `send`

A `return` is addressed **exactly as a `send` is** — it is a delivery whose cargo happens to be an assembled route — so it carries the same trail-and-pointer the delivery binding fixes, plus the route it is bringing home:

```
[ "grs.overlay.v1/return", <b64-addr-trail>, <pos>, <b64-cargo-trail> ]
```

- `<b64-addr-trail>` (argument 0) — the **addressing trail**, toward the *origin*. Same [trail buffer](./path-emergence/trail-wire-format.md) and same base64 carriage as a `send`'s trail; its preamble names the **origin's node ID** as `targetID`. The answering node reads that node ID straight off the head of the cargo trail (below), where the origin's own entry sits, and seeds the addressing trail with it.
- `<pos>` (argument 1) — the source-route pointer, read against the addressing trail's block count by the very same rule the [delivery binding](./delivery-binding.md) (§2) fixes: `pos < n` follows a known route home, `pos == n` is the frontier. So if the answering node already holds a route to the origin it source-routes the return along it; if it does not, the addressing trail is **zero-block** with `pos = 0` and the return *floods* toward the origin, building its route home as it goes. "The return is itself a flood if there is no cached route" is no longer a special clause — it is the frontier rule applied to a return.
- `<b64-cargo-trail>` (argument 2 — the payload slot) — the route being brought home, the `origin → target` [trail](./path-emergence/trail-wire-format.md) that discovery built: a single signed chain, always ([trail wire format](./path-emergence/trail-wire-format.md) §5). This is the object the origin files. It rides in the position a `send`'s `<payload>` occupies, because to the addressing machinery it *is* the payload; only the selector marks that it is a route to file rather than cargo to deliver.

Two trails, two jobs, and they must not be confused: the **addressing trail** routes the return *home* (answerer → origin) and is followed-or-flooded like any send; the **cargo trail** is the discovered route *out* (origin → target) that the return exists to deliver. They point in opposite directions and one is the other's payload.

## 2. What the origin does on arrival

When the return reaches the frontier and this node is the origin named in the addressing trail's preamble, the receiver does the `return`-specific thing: it **files the cargo trail in its route cache against the cargo trail's `targetID`**. There is no hop-by-hop install — nothing is written into the interior of the network, and the next `send` to that target simply carries the filed trail with `pos = 0` and is followed. Filing is the whole terminal action; it is the one thing that distinguishes `return` from `send`, and the reason it keeps its own selector.

## 3. Notes on design

- **Why a distinct selector at all, if the addressing is identical.** Everything about *moving* a return is a `send`: same trail, same pointer, same follow-or-flood. The only difference is the terminal act — file a route versus deliver cargo up — and that difference is not visible in the addressing, so it cannot be inferred from shape. The selector is what tells the receiver which terminal act to run. Mirroring the addressing while keeping the selector is exactly the split that says "same transport, different destination semantics."
- **Why the destination is derived, not carried twice.** The origin's identity is already present, at the head of the cargo trail (its own entry, block 0). The answering node reads it there and seeds the addressing trail's preamble with it; it is never passed as a separate argument. Passing it twice would create two copies that could disagree — the same trap the [trail wire format](./path-emergence/trail-wire-format.md) (§2) avoids by keeping `targetID` and `D` in one canonical place.
- **Budget two floods.** In the common first-contact case neither the outbound discovery nor the homeward return has a cached route, so each floods: one flood to find the target, one to carry the route home. The return, having reached the origin, MAY leave origin-ward state behind it so the answering node can answer more cheaply next time — a pure optimization the origin never depends on.
