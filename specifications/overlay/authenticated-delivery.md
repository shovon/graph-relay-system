# Authenticated Delivery: a signed-`send` profile

The base [delivery binding](./delivery-binding.md) carries an opaque payload and signs nothing about it: a `send` attests neither who authored the cargo nor that it arrived unaltered. That is deliberate — the overlay authenticates *routes* (a trail is a [signed hash chain](../signed-hash-chain/signed-hash-chain.md)) but treats the payload the way the relay treats it, as bytes it will not interpret. Payload authenticity is an application policy, and the base substrate declines to impose one.

This document is the derivative for deployments that *do* want that policy at the overlay. It is the overlay's analogue of the move the [relay makes for identity](../relay/architecture.md): the base leaves a thing open on purpose, and a derived spec closes it for those who need it, without the base having committed anyone. **A conformant base implementation ignores everything here; this profile is opt-in and adds no requirement to the base wire.**

What it provides: **origin authenticity and end-to-end integrity of the payload**, tied to the route's own origin, plus **authenticated at-most-once delivery**. What it deliberately does not provide is set out in §6.

## 1. What it adds to the wire

The profile appends **one trailing argument** to the base `send` line, under the append-only rule of the [message shape](../encodings/json-array-message-shape.md) (§6):

```
[ "grs.overlay.v1/send", <b64-trail>, <pos>, <payload>, <b64-auth> ]
```

- `<b64-trail>`, `<pos>`, `<payload>` — exactly as the [base binding](./delivery-binding.md) fixes them, with one added constraint (§2): under this profile `<payload>` **MUST** be carried as unwrapped-base64 bytes, never as a bare JSON value, so the signed bytes are unambiguous.
- `<b64-auth>` (argument 3) — the **authentication envelope**, a fixed-layout binary buffer, unwrapped-base64 into a JSON string exactly as the trail is:

  ```
  <b64-auth> = base64( seq(8) ‖ σ(65) )
  ```

  an 8-octet big-endian sequence number followed by the 65-octet [recoverable signature](../crypto/recoverable-ecdsa-signature.md). Nothing else is transmitted, because — as with the [chain seed](./path-emergence/seed-commitment.md) — every other input to the signature is either a constant or a field the receiver already holds (§2). Authorship and freshness for 73 bytes.

The selector is unchanged. A base (four-element) `send` and an authenticated (five-element) `send` are the *same operation*; the trailing argument is the authentication, not a different message. A receiver operating this profile MUST decide, as a deployment posture, whether it **enforces** (a `send` lacking a well-formed `<b64-auth>` is dropped, [per line](./interface-profile.md#drop-the-rpc-not-the-packet)) or **verifies opportunistically** (an unsigned `send` is delivered as the base allows, a signed one is checked). Enforce is fail-closed and is the recommended posture for a deployment that adopts this profile at all.

## 2. What the signature covers

The signed value is a single [SHA-256](../crypto/secp256k1-signature-profile.md) digest over the [length-prefixed encoding](../encodings/length-prefixed-encoding.md) of four fields:

$$
\text{digest} = \text{SHA-256}\big(\,\text{LP}(\texttt{DOMAIN\_TAG\_DELIVER}),\ \text{LP}(\text{targetID}),\ \text{LP}(\text{SHA-256}(\text{payload})),\ \text{LP}(\text{seq})\,\big)
$$

and $\sigma$ is the [recoverable ECDSA signature](../crypto/recoverable-ecdsa-signature.md) over that digest, low-$s$ per the [signature profile](../crypto/secp256k1-signature-profile.md) (§3) so it recovers exactly one principal. Each field earns its place:

- **`DOMAIN_TAG_DELIVER`** — the US-ASCII octets of `grs.overlay.v1/authenticated-send` (33 bytes, no terminator). Domain separation: a delivery signature can never be replayed as a [trail link](./path-emergence/out-edge-commitment.md) or a [chain seed](./path-emergence/seed-commitment.md), whose tag is the distinct `grs.overlay.v1/trail-seed`.
- **`targetID`** — the destination node ID, read from the [trail preamble](./path-emergence/trail-wire-format.md) (§2), which every `send` now carries. Binding: an authenticated `send` is authored *for this destination* and cannot be captured and redirected at another. It is not transmitted for the signature's sake — the preamble already holds it, and the recipient supplies its own ID as "me."
- **`SHA-256(payload)`** — a hash of the application bytes, so the digest is a fixed 32 octets regardless of cargo size. Integrity: any alteration in flight breaks the signature.
- **`seq`** — the 8-octet sequence number carried in the envelope, u64 big-endian ([encoding](../encodings/length-prefixed-encoding.md), §2). Freshness: it is what makes replay detectable (§4).

Because `DOMAIN_TAG_DELIVER` and `targetID` are recomputed rather than sent, the only new bytes on the wire are `seq` and $\sigma$.

## 3. The author is recovered, and checked against the route's origin

The profile does not carry an author ID. It **recovers** one, the same way the overlay recovers a [node identity](./path-emergence/recovered-node-identity.md) from a trail link: the receiver reconstructs the digest of §2 from what it holds — the constant tag, its own node ID, the payload on the wire, the `seq` from the envelope — and runs `Recover(σ, digest)`, yielding the author's public key in [compressed SEC1 form](../crypto/secp256k1-signature-profile.md) (§2), 33 octets. Nothing to claim, nothing to check a claim against.

Because a `send` under the unified binding *also* carries the trail whose first block recovers to the route's origin, the target holds **two** recovered identities and can check them against each other:

```
route_origin   = recover(trail.block[0])       # who built the route
payload_author = recover(σ)                     # who signed the bytes
```

**Whose keys these are, and whether they must match, is the deployment's binding:**

- **Principal is the node.** The signing key is the same key that recovers as the origin at the head of this node's trails. The deployment requires `route_origin == payload_author` and **rejects a mismatch** — welding payload authorship to the route origin *without* welding the payload into the chain, which would cost the trail its reusability (§6). This is the posture that also turns a first-contact flood-send, anonymous under the base, into an attributable one: the recipient recovers *who* to reply to.
- **Principal is an application identity distinct from the node.** The overlay recovers it, hands `(principal, payload)` up, and leaves `route_origin` as separate information; the application decides whether it trusts that principal. The overlay does not conflate the two identity spaces, and this profile does not force it to.

The overlay hands the recovered author up alongside the payload; it does not itself decide whether the author is *authorized*, only who it is.

## 4. Freshness and authenticated at-most-once delivery

Flooding and retries deliver the same cargo to a destination more than once ([stateless discipline](./path-emergence/stateless/stateless.md)); a destination that wants at-most-once must deduplicate. Under this profile the dedup key is **`(recovered author, seq)`**, and both halves are sound: the **author** is recovered from the signature, not read from a spoofable field, and the **seq** is inside the signed digest, so an attacker can neither forge a fresh `(author, seq)` without the author's key nor relabel a captured `send` with a new `seq` — changing `seq` invalidates $\sigma$.

**Why `seq`, and not the trail's `D`.** It is tempting to reuse the per-flood discovery ID `D`, which already rides in the trail, as the nonce. But `D` identifies a *flood*, and a *reused* cached trail carries one fixed `D` across every message source-routed along it — so `D` cannot tell message #2 from message #3 over the same route. `seq` is per-*message*, transmitted in the envelope, and works identically whether the send floods (first contact) or follows a cached route. Two nonces, two jobs: `D` bounds the flood, `seq` dedups the message; they never collide.

Sequence discipline (per-author monotonic, per-`(author, target)` monotonic, or a random nonce with a remembered-set) is a deployment choice; the profile fixes only that `seq` is 8 octets, signed, and the second half of the dedup key.

## 5. Only the target verifies

The payload signature is **created once by the origin and verified once by the target — end to end.** Intermediate hops never touch it: they follow-or-flood the `send` by the base `pos`-against-block-count rule, carrying `<payload>` and `<b64-auth>` as opaque riders. Crucially the envelope is **not a chain link** — it does not fold into the trail's running hash — so the trail stays payload-blind and therefore reusable: the origin re-signs the payload per message (one signature) while reusing the cached route with no re-flood.

## 6. What it does not provide

- **No confidentiality.** The payload is signed, not encrypted; every hop still reads it. Secrecy is a separate concern and not this profile's.
- **No route authentication beyond the trail's own.** Authorship here is deliberately *path-independent* — the signature says nothing about which trail carried the cargo, which is correct: who authored a payload should not depend on how it travelled. Route integrity remains the [trail's](./path-emergence/trail-wire-format.md) job, and the two are orthogonal, which is exactly what lets the payload signature stay separable and the route stay reusable.
- **No delivery guarantee.** Relay carriage is still [best-effort](../relay/relay-and-neighborhood-semantics.md); a signature authenticates a `send` that arrives, and does nothing to make it arrive.
- **No authorization.** The profile establishes *who* authored the payload, never that they were *permitted* to send it. That judgment is the application's, over the author the profile hands it.

## 7. Notes on design

- **Append, do not fork.** A new selector would split delivery into two operations and force every receiver to dispatch twice for one semantics. Appending the authentication as a trailing argument keeps one `send`, keeps the reserved namespace's name set minimal, and lets a base receiver tolerate the extra argument — the append-only shape working as intended. The cost is that opportunistic verification permits a silent downgrade to unsigned; a deployment that cares closes that by taking the *enforce* posture (§1).
- **Recover, don't transmit — twice over.** The author is recovered from $\sigma$, and the `targetID` and tag are recomputed, so the envelope is 73 bytes regardless of who signs. This is the [seed commitment's](./path-emergence/seed-commitment.md) frugality applied to delivery: bind everything, transmit only what cannot be recomputed.
- **Why the payload is signed here and not welded into the trail.** The cheaper-looking option is to fold `SHA-256(payload)` into the trail's seed, so the origin's existing block-0 signature would authenticate the payload for free. It is rejected: the running hash folds the seed into *every* link, so a payload-welded trail is bound to that one payload and can never be cached and reused for the next message — and reuse is the entire point of discovering a route. Keeping the payload signature separate from the chain is what lets the route be reused and the bytes be trusted at the same time. The routing chain is payload-blind on purpose; this is the signature that is not.
