# Interface Profile: Packets and RPCs

The two path-emergence memos — stateful and stateless — describe how delivery works: how a node floods to find a trail, how the trail comes back, how it installs or gets stamped onto a header, how breaks are repaired. Both of them talk about "flooding the **cargo**," about "an **RPC** that builds the trail," about a node "**ferrying** something back." None of them say what actually travels between two adjacent nodes to make those words true.

That is this memo's job. It defines the envelope the path-emergence disciplines are written on top of — the contract at the single hop, below the question of how trails come to exist. It deliberately specifies nothing about routing; it specifies the unit that routing is expressed in.

The whole thing rests on one move:

> A **packet** is an envelope. The unit of meaning is the **RPC**. One packet may carry several RPCs. Addressing and validity are decided **per RPC**, not per packet.

Everything below is a consequence of taking that seriously.

---

## What rides between two nodes

A packet is the raw thing that arrives on an edge: bytes, with no inherent meaning and — per the network model — no trustworthy sender identity. Meaning lives entirely in the contents. Inside the envelope are one or more RPCs, each a self-contained request: _deliver this cargo, flooding to find the route if I have none yet_, or _here is a route I assembled, file it_. An RPC is the smallest thing a receiving node acts on or declines to act on. The packet is just how some RPCs happened to travel together on one hop.

Keeping these two levels apart is what lets the rest of the design stay honest. The moment "the packet" becomes the unit of meaning, you start asking ill-formed questions — does this packet take precedence over that one, is this packet addressed to me — when the real answers are per-RPC and differ within a single envelope.

---

## Why carry more than one RPC

For a while the motivating case was sending into the unknown: a node with cargo but no route paid one RPC to discover the route and a second to deliver along it, and carried both in one packet. That pairing is gone. [Delivery folded discovery in](./send.md) — a node sending into the unknown now emits a *single* `send` whose trail is not yet complete, which floods to build the route while carrying the cargo. Delivery and discovery are one RPC, not two, and the packet that once held a discovery beside a delivery now holds one line.

What keeps the per-RPC framing earning its keep is the other direction. The **selector set is open** (below), so a node MAY multiplex an *extension* RPC — a health probe, a revocation notice, a metrics beacon — into the same envelope as a `send`, and a receiver that has never heard of the extension drops that one line and acts on the `send` beside it. Multiplexing is no longer load-bearing for the overlay's own RPCs; it is what lets the vocabulary grow without a flag day.

It is worth being careful about a symmetric-looking case, because it is _not_ two RPCs. When a delivery carries a _trail_ as its content — the discovered route riding home to the origin, a [`return`](./return-binding.md) — that is still a single RPC with the trail as its cargo, exactly as a `send` carrying ordinary cargo is one RPC. Content riding as the payload of a delivery does not make a second RPC.

---

## One flood is many packets

"Flooding" reads like a single act, and as an _RPC type_ it is one thing: build-the-trail-onward. But it is never one packet. To each out-neighbor the node appends a trail entry of its own node ID paired with the **out-edge designator for that specific neighbor** — and designators are per-edge, so the appended entry, and therefore the packet, differs for every neighbor.

The unit on the wire is the per-edge packet. A flood is the act of emitting `N` of them, identical in intent and different in their head entry. Calling it "one RPC" is fine as long as it never quietly collapses the fan-out: there is one operation, many differentiated copies, and the difference is exactly the local designator that makes the trail a forward source route.

---

## Addressing is a property of the RPC, not the packet

Different RPC types answer "is this for me?" differently, and that is the whole reason the per-RPC framing pays off.

A single `send` answers "is this for me?" in one of two ways, and which one is decided by its pointer against its trail's block count — not by a different RPC type.

- **At the frontier** (`pos` past the last block) the send names only the **destination it is hunting** — the `targetID` in the trail preamble — and *not* a next hop, because the next hop is the very thing the flood is discovering. So whoever receives it on an out-edge is a legitimate place for it to be, and no head entry has to name the receiver for the RPC to be acted on. This is how the old flood addressed itself.
- **Inside the blocks** (`pos` before the end) the send is addressed precisely: the entry at `pos` names the node meant to act next. The receiver applies a simple test — _does the entry at `pos` recover to me?_ If it does, the hop is for it; it forwards and advances the pointer. If it doesn't, the route has rotted — the packet arrived somewhere it shouldn't have. The pointer advances rather than the head being stripped, because the trail is a signed chain and cannot be truncated without breaking it.

This head-entry self-check is a delivery-integrity test, not loop detection. It asks only "was this hop meant for me," and a node that fails it knows the routed RPC in its hands is stale on arrival, independent of any cycle reasoning.

---

## Drop the RPC, not the packet

Once validity is per-RPC, the failure mode follows. When a routed RPC's head entry does not name the receiver, the instinct is to drop the packet — but the packet may be carrying other RPCs whose addressing is perfectly fine. A flood RPC sharing the same envelope is still addressed to "whoever received it," and that is this node.

So the discipline is: a node evaluates each RPC in the envelope on its own terms and discards the ones it cannot honor, not the envelope that carried them. A misaddressed `send` does not take a co-traveling RPC — an extension, say — down with it. Dropping at the granularity of the RPC is what keeps multiplexing from turning every envelope into an all-or-nothing bet.

---

## The selector, and why the set is open

Everything above treats "an RPC" as something a node recognizes and acts on without saying _how_ it recognizes it. The missing piece is small: each RPC leads with a **selector**, a tag naming which operation it is. Dispatch is per-RPC because the selector is per-RPC — a node reads the selector, finds the handler registered for it, and runs that. `send` and `return` are two such selectors; the path-emergence profiles define them, and there is nothing structurally special about them beyond being the ones written down first.

Naming the selector makes the property that matters visible: **the set of selectors is open.** No fixed enumeration of RPC types is baked into the envelope, and no registry has to bless a new one. A selector is **namespaced** — a `(namespace, name)` pair — and a namespace is a minter's own, derived the way node IDs are and with the same no-coordinator guarantee: a sufficiently-wide value, or the hash of a public key. Within its own namespace a party assigns local names as it likes, and since two parties never share a namespace without having chosen to, two independently-minted selectors never collide. The overlay's own RPCs sit in a namespace the spec reserves; everyone else works in theirs, and no one has to ask.

What keeps an open set safe is a rule the envelope already states, read one notch wider. **Drop the RPC, not the packet** covered the RPC a node cannot _honor_ — misaddressed, malformed. An RPC whose selector a node does not _recognize_ is the same case: no handler, so the node drops that one RPC and leaves its co-travelers untouched. Nor does it forward it — a node cannot re-emit semantics it does not know — so an extension RPC travels only among the nodes that implement it, growing its own reach inside the overlay without a flag day and without perturbing anyone who has never heard of it. A node written years before a selector existed is forward-compatible with it for free, by doing nothing.

So `send` and `return` are not _the_ vocabulary; they are its first entries. A new capability — a health probe, a revocation notice, a metrics beacon — is a new selector in its author's namespace, multiplexed into the very same envelopes, dropped politely by every node that does not speak it and acted on by every node that does.

---

## Plug-in points

This memo defines the envelope; it does not define what a flood needs in order to stop, or how a node decides it has seen a route before. Those belong to the cycle-detection memo, and the interface profile only reserves room for them:

- A flood / build-trail RPC carries whatever **discovery or dedup identifier** remember-and-drop termination requires **inside the trail it already carries**, not as a separate envelope field. The envelope adds no slot of its own; the [flood-safety memo](./path-emergence/cycle-detection.md) (§2) defines the rule over that identifier — remember-and-drop keyed on `(recovered origin, targetID, D)`.
- **Loop-freedom** is committed in the [flood-safety memo](./path-emergence/cycle-detection.md) (§3), and it falls out of a structural fact rather than a rule: a signed chain does not concatenate, so every trail is a single flood's chain and — with remember-and-drop making each node append itself once — cannot revisit a node. There is no loop-check to run; the profile guarantees only that the trail entries are present and well-formed enough to be verified.

The head-entry self-check above is the one validity test that lives here rather than in the cycle memo, because it concerns whether a single hop was meant for this node — delivery integrity — and not whether the route as a whole revisits itself.

---

## The concrete binding

This memo fixes the envelope in the abstract — packet, RPC, selector, per-RPC dispatch and drop — and stops short of the bytes on purpose, the way it stops short of routing. Three companion memos supply the bytes:

- [Packet framing](./packet-framing.md) fixes the envelope itself: a packet is newline-delimited ([JSONL](../encodings/json-array-message-shape.md)) message lines, one RPC per line, dispatched and dropped **per line** — the per-RPC discipline of this memo read at the byte level, where "drop the RPC, not the packet" becomes "drop the line."
- [Delivery binding](./delivery-binding.md) fixes the `send` line: selector `grs.overlay.v1/send`, a trail-and-pointer that addresses both a known route and — when the trail is short — a flood to discover one, the payload opaque.
- [Return binding](./return-binding.md) fixes the `return` line: selector `grs.overlay.v1/return`, addressed exactly as a `send`, carrying the assembled route home as its cargo.

Those selectors are names in the reserved `grs.overlay.v1` namespace §"The selector, and why the set is open" describes. Nothing in the concrete forms is new machinery: multiplexing two RPCs is two lines, and honoring one while discarding another is the per-line reading of the drop rule above.
