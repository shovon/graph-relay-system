# GRS in Full: A Top-Down Reading

## What this document is

This is the whole picture in one place, read from the top down: how a Graph Relay System hangs together, from the single idea at its root to the bytes on a socket. It is a **story, not a standard** — non-normative throughout. Every rule that an implementer must obey lives in one of the companion documents linked below; where this reading states a guarantee, the authority for it is over there, and where it shows a wire shape, the binding that fixes those bytes is the one that governs.

The system is not one design but a small family of them, assembled from choices. To tell one coherent story this document **commits to a single configuration** and follows it all the way down. Two commitments in particular:

- for the relay substrate, the **push, one-way, JSON-array over WebSocket** profile — not the pull profile the same core also admits;
- for the routing built on top, the **stateless path-emergence discipline** carried as **JSONL of JSON-array messages** — not the stateful discipline that is its sibling.

Wherever the road forks, the fork is named and then the committed branch is taken. A reader who wants the other branch will find the map at the end.

The shape of the whole is two layers over a shared foundation. Underneath sits a **relay**: a server that will carry one message one hop, to a neighbor, and no further. Above it sits an **overlay**: routing that clients build for themselves, turning a pile of single hops into delivery to anywhere. The relay is deliberately small; the overlay is where reach, replies, and durable identity come from. The rest of this document is those two layers and the seam between them.

---

## 1. The one idea

> A server maintains a directed graph whose nodes are the connected clients, keeps it strongly connected, and relays only along a node's out-edges. Everything richer is constructed by clients above that primitive.

That sentence is the entire system ([`relay/architecture.md`](relay/architecture.md)). The server decides *who may talk to whom* — it owns the adjacency — but it does not carry *what is said* any further than one directed edge. A node may address only its **out-neighbors**, each named by a **designator** the server hands it; the server relays a payload to the out-neighbor a designator denotes, or discards it, and does nothing else.

The graph is kept **strongly connected**: a directed path exists from any node to any other. But that reachability is the server's own structural guarantee, not a feature clients get to call. It exists so that any routing a client might wish to build is *possible in principle* — a path is always there to be discovered — while leaving the building to the client. In the architecture's own words, the system is a **substrate**, and "its utility is an emergent property of how clients choose to use the neighbor primitive over a graph that is guaranteed to be traversable." The overlay in Part 3 is that emergence made concrete.

---

## 2. The single hop: the relay

### What the primitive promises, and refuses

The relay exposes exactly one interaction: **send a payload to an out-neighbor, by designator.** Its contract is fixed in [`relay/relay-and-neighborhood-semantics.md`](relay/relay-and-neighborhood-semantics.md), and it is defined as much by what it refuses as by what it offers:

- **Forward-only, no reply channel.** The neighbor relation is directional. Receiving a message grants no ability to answer it — an in-edge is not a send path. To reach whoever sent to you, that node must independently be one of *your* out-neighbors, an edge that may simply not exist. This holds without exception, for replies, acknowledgements, and errors alike.
- **No-misdelivery.** A message reaches the out-neighbor its designator denoted *at the moment of sending*, or it reaches no one. It is never delivered to the wrong node. A stale designator costs a dropped message, never a misdelivery.
- **Best-effort.** Acceptance is not delivery, and the sender may not infer one from the other. The server must genuinely try, and must not systematically drop conformant traffic — but per-message, nothing is promised.
- **Thin identity.** A designator is only *distinct among a node's current out-neighbors*. It need not be globally unique, stable across neighborhood changes, or persistent across reconnection. And by default a node's identity is its connection: reconnect and you are a new node, inheriting nothing.

To send at all, a node must know its neighbors, so the server keeps each node's **neighborhood state** — the set of its current out-neighbors' designators — available, refreshed whenever the graph changes.

### The three deliberate silences

Read that list again for what is *missing*, because the overlay is built precisely out of the gaps:

1. **No reach past one hop** — no traversal, no end-to-end send to a non-neighbor.
2. **No reply path** — a receiver cannot answer a sender it has no edge toward.
3. **No durable identity** — nothing ties a node across sessions; designators are per-state handles, not names.

The architecture is emphatic that these are *left to clients*, not oversights: clients "MAY cooperatively forward an opaque payload hop-by-hop along directed edges to reach a non-neighbor — or to route a reply back to a node that has no edge toward them — treating the neighbor primitive as a transport and layering their own routing above it." That layer is the overlay. It also anticipates the identity fix by name: a derivative "MAY give a node an identity that survives session termination… one natural realization is a key pair whose public half serves as the node's durable identifier." The overlay takes that suggestion up too.

### The committed realization: push, one-way, JSON-array, over WebSocket

The abstract primitive becomes concrete through a chain of documents, each fixing exactly what the one above it left open. From top to bytes:

1. **[`relay/conformance/interface-profiles/rpc-interface.md`](relay/conformance/interface-profiles/rpc-interface.md)** — the RPC common core. Abstract operations (`Send`, and the receiving half, and neighborhood availability) and abstract data (`Designator`, `Payload`, `NeighborhoodState`). It fixes only the one client-initiated operation whose meaning never varies, `Send`, and defers the rest to a profile — because whether the server can *push* is a property of the transport.
2. **[`.../push/rpc-push-profile.md`](relay/conformance/interface-profiles/push/rpc-push-profile.md)** — the pushable profile. For full-duplex, session-oriented transports where the server can push at will. The receiving half becomes a genuine server push, `Deliver`; neighborhood changes are pushed as `NeighborhoodUpdate`; and — the elegant part — **the connection *is* the node**: opening it joins the graph, it carries identity implicitly, closing it is departure. No join operation, no session handle. *(The [pull profile](relay/conformance/interface-profiles/pull/rpc-pull-profile.md) is the sibling this story does not take: it synthesizes a session over a connectionless transport and drains messages by polling.)*
3. **[`.../push/rpc-push-oneway.md`](relay/conformance/interface-profiles/push/rpc-push-oneway.md)** — the one-way derivative. It fixes the single choice the profile left open: `Send` surfaces *nothing back* — no acceptance, no output. Now every operation in the interface is one-way, and there is nothing anywhere to correlate.
4. **[`encodings/json-array-message-shape.md`](encodings/json-array-message-shape.md)** — the envelope. A message is a JSON array whose first element is a **selector** and whose rest are positional arguments: `[ selector, arg0, arg1, … ]`. It carries no correlation field (that belongs to the transport), no response (the protocol is fire-and-forget), and grows only by *appending* arguments. This same envelope is used, unchanged, by the overlay — it is the one wire form both layers share.
5. **[`.../bindings/json/oneway-binding.md`](relay/conformance/bindings/json/oneway-binding.md)** — where the derivative meets the envelope. It assigns the three selectors and their layouts, drawing designators and payloads from the two representation bindings ([designator-string](relay/conformance/data-shapes/designator-string.md), [payload-string](relay/conformance/data-shapes/payload-string.md)) and node identity from [unique-session-identity](relay/conformance/bindings/unique-session-identity.md).
6. **[`.../bindings/json/websocket-transport-binding.md`](relay/conformance/bindings/json/websocket-transport-binding.md)** — the bytes. One shape message per WebSocket **text** message; the subprotocol token `grs.rpc-oneway.v1.list+json` negotiated in the handshake activates the whole stack; the connection lifecycle *is* the node lifecycle.
7. **[`.../bindings/json/websocket-resource.md`](relay/conformance/bindings/json/websocket-resource.md)** — the one thing the transport binding defers: **the connection's URL identifies the graph it joins.** The client treats that URL as an opaque whole.

Put a node on the wire and it looks like this. It opens a WebSocket to the resource URL for its graph — say `wss://example.com/some-room` — offering `Sec-WebSocket-Protocol: grs.rpc-oneway.v1.list+json`; the server echoes that token and the connection is now this node's whole existence. The server greets it with its neighborhood:

```
[ "NeighborhoodUpdate", [ "n7gKQ", "p2Lx9" ] ]
```

To send "hello world" to the out-neighbor that designator `"n7gKQ"` denotes, the node emits one text frame:

```
[ "Send", "n7gKQ", "hello world" ]
```

and is owed nothing back. The server resolves `"n7gKQ"` against this node's *current* neighborhood by exact string equality, relays the opaque payload verbatim if it still denotes an out-neighbor, and silently discards it otherwise. The out-neighbor receives, on its own socket:

```
[ "Deliver", "hello world" ]
```

Note what is *not* in that array: no sender, no reply handle. The receiver cannot learn who sent it or answer them through the relay. That absence is not a gap to be filled here — it is the relay keeping its promise, and the reason the overlay exists.

---

## 3. The routing that emerges: the overlay

The overlay is the client-built layer the architecture foretold. It takes the relay's single hop as its transport and constructs, above it, the three things the relay refused: **reach to any node, a way home, and durable identity.** Its network model is, deliberately, the relay's model restated — forward-only sends, locally-scoped out-edge designators, no sender identity on the wire — plus one addition that changes everything.

### The addition: durable, self-certifying node IDs

The overlay gives every node a **globally-unique node ID** that lives in a different space from designators and survives across the churn that resets designators. It realizes exactly the architecture's suggested identity fix: the ID is derived from a key pair. And it goes one better — rather than transmit the ID, it *recovers* the ID from a signature the node already had to make ([`overlay/path-emergence/recovered-node-identity.md`](overlay/path-emergence/recovered-node-identity.md), over [ECDSA on secp256k1](crypto/secp256k1-signature-profile.md)). A node's identity is thus a pure function of its signature: nothing to claim, nothing to check a claim against, nothing extra on the wire.

### The committed axiom: the origin carries the whole route

Under the **stateless** discipline ([`overlay/path-emergence/stateless/stateless.md`](overlay/path-emergence/stateless/stateless.md)), a route lives at the **origin** and nowhere else. On every send the origin stamps the complete ordered list of *(node ID, out-designator)* pairs into the packet, and each hop reads the one entry that is its own, forwards, and advances a pointer. Intermediates remember nothing. This is the honest fit for locally-scoped designators — a forward source route works precisely because every hop interprets only its own entry — and it concentrates staleness into one visible place: when a route dies it dies whole, at the origin, and is rediscovered on the next send, with no silent per-hop rot to chase. The price is a header that grows with path length. *(The [stateful discipline](overlay/path-emergence/stateful/stateful.md) is the sibling not taken: it installs forwarding state hop-by-hop instead of carrying the route, trading header size for interior state that can rot.)*

### How a route comes to exist

A node with cargo for a target it has no route to **floods**: it emits the cargo to every out-neighbor, each copy carrying a trail entry of its own *(node ID, out-designator)*, and every node that receives it does the same, so the trail grows one hop per node. Somewhere the flood reaches the target *itself* — the only node that can end a discovery, because a signed trail cannot be spliced and so no intermediate can hand back a route it did not itself traverse. That **answering node** now holds the assembled route, and **delivers it back to the origin** — which, because trails are forward-only, is itself a fresh delivery to the origin's node ID, not a walk back up the trail. The origin files the route against the target and, from then on, stamps it into the header. There is no install walk into the network's interior to break or half-complete; the whole apparatus reduces to *discover, carry, forward*. A node that already recognizes the way onward need not flood blindly — it MAY re-emit to just the one out-neighbor it trusts, still appending its own hop; that is a pure fan-out optimization, never a splice, and it falls back to a full flood the moment the guess looks stale.

A flood on a cyclic graph must be kept from circulating forever, and duplicates must be suppressed; the overlay leans on a per-flood discovery identifier that nodes remember-and-drop, plus a bound on how far a flood travels. **That a flood terminates within some bound is load-bearing and assumed throughout; the precise rule that makes it so is the normative path-emergence layer's to fix, not this reading's to enumerate.** Route loops need no such rule at all: a signed trail cannot be concatenated, so every route is one flood's single chain, and remember-and-drop lets a node join that flood only once — it can never revisit itself.

### Trails are signed hash chains

The route a flood accumulates is not a bare list — it is a **signed hash chain** ([`signed-hash-chain/signed-hash-chain.md`](signed-hash-chain/signed-hash-chain.md)), and this is what makes it trustworthy to carry. Each hop signs the out-edge it forwarded along, chained onto a running hash of everything before it ([`overlay/path-emergence/out-edge-commitment.md`](overlay/path-emergence/out-edge-commitment.md)); the chain is seeded with a value binding the flood to its target and its freshness ([`overlay/path-emergence/seed-commitment.md`](overlay/path-emergence/seed-commitment.md)). Because each signature commits to the entire prefix, the trail is **append-only and un-reorderable**, and — the overlay's chosen variant omits the public key from each link and recovers it from the signature — every hop's identity falls out of the same signature for free. What the reader gets is a route that names its own nodes, in order, that no intermediate could have rewritten, and that is welded to *this* flood toward *this* target so it cannot be replayed as a route to somewhere else.

### The committed realization: JSONL of JSON-array messages

On the wire an overlay **packet** is one or more RPCs, one per line — [JSONL](overlay/packet-framing.md) — each an array in the same [message shape](encodings/json-array-message-shape.md) the relay uses, with selectors in the reserved `grs.overlay.v1` namespace. Judging, dispatching, and dropping are all **per line**: an unrecognized or misaddressed RPC is dropped on its own without taking its line-mates down ([`overlay/interface-profile.md`](overlay/interface-profile.md)). Two selectors carry the discipline:

```
[ "grs.overlay.v1/send",   <b64-trail>, <pos>, <payload> ]
[ "grs.overlay.v1/return", <b64-trail>, <pos>, <b64-cargo-trail> ]
```

`send` ([`overlay/delivery-binding.md`](overlay/delivery-binding.md)) is the whole of delivery *and* discovery. It carries a trail and a pointer `<pos>`, and one comparison — is `<pos>` still inside the trail's blocks, or past them? — decides everything. Inside, the node follows its own entry and forwards; past the end it is at the **frontier**, where it either recognizes the target and delivers, or floods onward, appending a block of its own. So a *complete* trail is followed to its target, while a *zero-block* trail — a preamble naming only the target — is at the frontier from the origin and floods, which is how a first-contact delivery discovers its own route; the same send with an empty payload is pure discovery. There is no separate `build` selector: it folded into a short-trailed `send`, and *an unfinished trail is the signal to flood forward*. `return` ([`overlay/return-binding.md`](overlay/return-binding.md)) hands an assembled route home, addressed exactly as a `send` — its own trail-and-pointer toward the origin — carrying the discovered route as its cargo for the origin to file. The trail itself is the length-prefixed signed-hash-chain buffer ([`overlay/path-emergence/trail-wire-format.md`](overlay/path-emergence/trail-wire-format.md)), a binary object carried as one unwrapped-base64 string so its signed bytes ride untouched inside the JSON. The concrete stateless surface — which of these each step emits — is [`overlay/path-emergence/stateless/stateless-rpc.md`](overlay/path-emergence/stateless/stateless-rpc.md). A deployment that wants the *payload* signed too, not just the route, appends one signature under the optional [authenticated-delivery](overlay/authenticated-delivery.md) profile.

---

## 4. The seam: how the overlay rides the relay

The two layers meet at three exact points, and nowhere else:

- **An overlay packet is a relay payload.** The relay carries an opaque `Payload` string one hop; an overlay packet — its JSONL — *is* that string. One relay `Send` moves one overlay packet across one edge.
- **An overlay out-edge designator is a relay designator.** When the overlay says "forward along this out-edge," the forwarding is a relay `Send` to that designator's out-neighbor. The overlay's forward-only, locally-scoped edges are the relay's, because they are literally the same edges.
- **An overlay node ID is the durable identity the relay deferred.** The relay's per-session designators name neighbors within a moment; the overlay's key-recovered node IDs name nodes across time. The overlay is the "derived specification" the architecture said could supply persistent identity.

So a multi-hop overlay delivery is *N* single-hop relay sends, each carrying the packet one edge further, with the overlay's trail deciding which designator to hand the relay at each step. The relay sees only ordinary neighbor-to-neighbor traffic; it neither knows nor needs to know that a route is being walked.

The most telling fact about the seam is that **neither layer's documents mention the other.** The relay specs never say the word "overlay"; the overlay specs never say "relay." The overlay assumes only "a way to send an opaque payload to a chosen out-neighbor," which is all the relay claims to be. This is not an omission — it is the layering working. The substrate is complete without the routing, and the routing needs nothing of the substrate but its one primitive. Part 1's "emergent property of how clients choose to use the neighbor primitive" is this seam, seen from above.

---

## 5. The foundations underneath both

A few documents are neither relay nor overlay but the ground both stand on:

- **[`encodings/json-array-message-shape.md`](encodings/json-array-message-shape.md)** — the JSON-array envelope, selector-plus-positional-arguments, fire-and-forget, append-only. Both the relay's `Send`/`Deliver`/`NeighborhoodUpdate` and the overlay's `send`/`return` are messages in this one shape.
- **[`encodings/length-prefixed-encoding.md`](encodings/length-prefixed-encoding.md)** — the self-delimiting byte encoding that is both the *signing preimage* of every chain link and the layout of the trail buffer on the wire. It is what lets two implementations agree, to the byte, on what a signature covers.
- **[`signed-hash-chain/signed-hash-chain.md`](signed-hash-chain/signed-hash-chain.md)** — the append-only, self-recognizing chain, in two forms ([keys embedded](signed-hash-chain/with-embedded-keys/with-embedded-keys.md), [keys omitted](signed-hash-chain/without-embedded-keys/without-embedded-keys.md)); the overlay takes the key-omitted form and recovers identities.
- **[`crypto/secp256k1-signature-profile.md`](crypto/secp256k1-signature-profile.md)** and **[`crypto/recoverable-ecdsa-signature.md`](crypto/recoverable-ecdsa-signature.md)** — the recoverable signature scheme that makes "a node's identity is its signature" true, and its compact 65-byte wire form.

---

## 6. The honest edges

A top-down story earns trust by marking where the ground is not yet firm. Four deferrals are load-bearing enough to name:

- **Flood termination and deduplication.** The overlay *depends* on floods being bounded. The [flood-safety memo](overlay/path-emergence/cycle-detection.md) now commits both halves of termination — the absolute hop bound `H_max` (a node-local ceiling, default 64 for a planet-scale log-diameter graph) and efficient remember-and-drop, keyed on `(recovered origin, targetID, D)` so the flood nonce needs no unpredictability. Route loops need no handling at all: a signed chain does not concatenate, so every route is one flood's single chain, and remember-and-drop lets a node join it once — it can never revisit a node, so there is nothing to check. The one rule still open is a size cap on flooded payloads. This reading takes the guarantees and leaves the still-open parameter where it belongs.
- **The stateful discipline is not fully bound.** It is a real alternative to the stateless commitment taken here, but its wire surface — an install operation that writes forwarding state at each hop — has no binding yet. A deployment choosing stateful cannot yet build it from the wire docs alone.
- **Neighborhood-state ordering.** The relay makes each state reflect the latest committed configuration, but how a client tells a newer state from an older one is deferred to a future layer.
- **Trust.** The relay authenticates a node only as far as its connection; the overlay's trail entries are self-certifying as *route structure*, but a node still trusts the header it is handed, and the *payload* a `send` carries is unsigned in the base. Authenticating the payload — who authored it, and that it arrived unaltered — is the optional [authenticated-delivery](overlay/authenticated-delivery.md) profile's to supply.

---

## 7. The map: a reading order

For a reader who wants to follow the whole thing in the source, in roughly the order this story met it:

```
relay/
  architecture.md                              — the one idea: a relayed, strongly-connected graph
  relay-and-neighborhood-semantics.md          — designators, no-misdelivery, best-effort
  conformance/
    interface-profiles/rpc-interface.md        — abstract Send / receive / neighborhood
    interface-profiles/push/rpc-push-profile.md    — connection = node; server pushes
    interface-profiles/push/rpc-push-oneway.md     — Send returns nothing
    bindings/json/oneway-binding.md            — the three selectors and layouts
    bindings/json/websocket-transport-binding.md   — one text frame per message; the subprotocol
    bindings/json/websocket-resource.md        — the URL is the graph
    data-shapes/{designator,payload}-string.md — designator and payload as strings
    bindings/unique-session-identity.md        — per-session, graph-unique node identifiers

overlay/
  interface-profile.md                         — packet = envelope of per-line RPCs
  send.md                                       — the send primitive's meaning
  packet-framing.md                             — a packet is JSONL
  delivery-binding.md / return-binding.md      — send / return on the wire (build folded into send)
  path-emergence/
    stateless/stateless.md                     — the committed discipline
    stateless/stateless-rpc.md                 — its concrete RPC surface
    out-edge-commitment.md / recovered-node-identity.md / seed-commitment.md
    trail-wire-format.md                        — the signed chain as a byte buffer
    cycle-detection.md                          — flood safety: H_max, remember-and-drop; loop-freedom is free (no splicing); payload cap its agenda

encodings/       — json-array-message-shape.md, length-prefixed-encoding.md
signed-hash-chain/ — the chain, embedded- and omitted-key forms
crypto/          — secp256k1 profile, recoverable ECDSA signature
```

Start at `relay/architecture.md` with the one idea; end knowing that everything else is either the substrate keeping its narrow promise, or the routing a client grows on top to make that promise reach.
