# GRS Unique Per-Session Node Identifiers

## Status of This Memo

This document is a concrete **identity binding** for GRS. It answers, with a single definite rule, the identity-persistence axis the architecture leaves deferred (Architecture §3): a node's identity is **ephemeral and per-session** — minted fresh when a node is established, and never carried across a node's lifetime boundary — and additionally **unique and never reused within its graph**: distinct from every identifier any other node in the same graph holds or has ever held, for the entire lifetime of that graph, and never minted a second time. It exists for implementers and derivers who want a fixed rule to build on rather than the open choice the base leaves them. Underneath the rule is a single safety property it exists to guarantee: that the server never places a node in a position to mistake one out-neighbor for another (Section 2).

It takes the base architecture's _default_ — connection/session-scoped identity, a reconnect being a wholly new node (Architecture §3) — and strengthens it in exactly one direction the companions already permit (Relay §2; designator-string §4): the per-session identifier is guaranteed distinct from every identifier any other node in the same graph holds or will ever hold, and a retired identifier is never minted again. This restores the **global uniqueness** that designator-string §4 names as an optional stronger guarantee, scoped here to the graph (Section 2). It does **not** take the other direction the axis offers: it provides **no** persistence across sessions (Section 7). Global uniqueness of the _identifier_ is not continuity of _identity_ — a returning client is still a wholly new node bearing a new identifier, one merely guaranteed distinct from the old, not the old one restored. A derivative wanting durable identity — for example a public key serving as a durable node identifier (Architecture §3) — is a different binding, answering the same axis the other way.

This identifier serves as the node's **designator** — the handle by which its in-neighbors address it (Architecture §2, Relay §2) — so the binding fixes designator semantics as a consequence of fixing identity. It is **representation-agnostic** and composes with a representation binding such as `../data-shapes/designator-string.md` (under which the identifier is a string). It is one binding among possible others, and is normative for implementations that adopt it. Section references of the form (Architecture §N) point into `../../architecture.md`, (Relay §N) into `../../relay-and-neighborhood-semantics.md`, and (Core §N) into `../interface-profiles/rpc-interface.md`.

## Table of Contents

1. Terminology
2. The Rule
3. Generation
4. Lifetime and Retirement
5. The Designator Is the Node's Identity
6. Resolution and No-Misdelivery
7. What This Binding Does Not Provide
8. Security Considerations
9. References

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

This document uses the terms **Server**, **Client**, **Node**, **out-neighbor**, **in-neighbor**, **neighborhood**, **neighborhood state**, and **Designator** as defined in Architecture §2 and refined in Relay §2, and **session** as in Core §4.4. It adds:

- A **node identifier** is the value this binding fixes: the per-session, graph-unique, never-reused identity of one node, which also serves as that node's designator.
- A **graph** is the maximal relay domain within which a designator is resolved — the set of nodes that any one node may, over its lifetime, hold as out-neighbors (Architecture §2, §3). It is the scope across which this binding requires uniqueness (Section 2). The base architecture describes a single such graph; a server MAY host several **disjoint** graphs, between which no edge or neighborhood ever reaches (Architecture §3).
- An **epoch** is a single continuous run of the server: a restart that ends every live node and resets the server's minting state begins a new one. The epoch is relevant only to the optional per-run relaxation of Section 2; it is not the scope of the rule.

## 2. The Rule

**Governing invariant.** The server MUST NEVER place a node in a position where it could mistake one out-neighbor for another — neither two distinct out-neighbors within a single neighborhood state, nor, across the node's lifetime, a current out-neighbor for one the node previously held under the same designator. This is the property the binding exists to guarantee, and the graph and the node's lifetime are the two boundaries across which it must hold (Sections 5, 6). Everything below is in its service.

The three guarantees that follow are the **sufficient conditions** this binding adopts to uphold that invariant. They are not the only conditions that could — the mechanism is left open (Section 3), and a derivative MAY satisfy the invariant another way — but an implementation that meets them cannot violate it. When the server establishes a node — that is, at the start of each session (Core §4.4) — it assigns the node a **node identifier** carrying three guarantees:

1. **Per-session.** Exactly one identifier is minted per node, at establishment. A node never changes identifier during its lifetime, and a new session is a new node with a new identifier (Architecture §3).
2. **Unique within the graph.** The identifier is distinct from that of every other node in the same graph — every contemporary, and every node the graph has ever held or will ever hold — not merely among concurrently live nodes, and not merely within a single run of the server.
3. **Never reused within the graph.** Once a node departs, its identifier is retired permanently and is never minted again for any later node in that graph (Section 4).

Those three together are the entire rule. Everything else in this document either says how to satisfy them (Section 3), what follows from them (Sections 4–6), or what they deliberately do not give you (Section 7). Two qualifications fix the _scope_ of guarantees 2 and 3.

**The scope is one graph.** "Within the graph" is exactly the reach across which a designator can be resolved against a node (Section 6). Where a server hosts several disjoint graphs, uniqueness is required only within each; this is a **floor, not a ceiling**. A server MAY mint from an **independent identifier space per graph** — even reusing the same value in two graphs — or MAY equally enforce uniqueness more broadly, across all its graphs or globally, at no cost to correctness. Either way, no designator is ever resolved across a graph boundary (Section 6), so a cross-graph collision can confuse no node and cross-graph uniqueness buys none any guarantee it could observe: it is permitted, occasionally simpler to build, and otherwise overkill. This rests on graphs being disjoint addressing domains — no node's neighborhood ever names a node of another graph (Architecture §3). A deployment that bridges two graphs MUST widen uniqueness to span the bridged domain (Section 8).

**An optional relaxation, per run.** A deployment in which every holder of an identifier necessarily dies whenever the server loses its minting state — for example one whose sessions live only in server memory, so a restart ends them all — MAY relax guarantees 2 and 3 from "for the lifetime of the graph" to "within a single **epoch**," reusing identifiers freely across runs. This is safe because the restart that resets the minting state also ends every holder that could be confused, so no surviving node can mistake the re-minted value's new bearer for its old one — the governing invariant still holds (Section 8). It is an optimization for a constrained identifier space, nothing more. A deployment whose holders can outlive the server's minting state — and any deployment that cannot _prove_ they cannot — MUST NOT take this relaxation; it relies on the baseline graph-wide guarantee instead (Section 8).

## 3. Generation

How the server mints identifiers satisfying Section 2 is implementation-defined; this binding fixes the guarantees, not the mechanism. Three families suffice, by way of non-normative example:

- **Random (RECOMMENDED).** A token drawn from a space large enough that collision over the lifetime of the graph is negligible (e.g. a version-4 UUID). It satisfies graph-wide uniqueness and never-reuse directly: a fresh draw after a restart is no more likely to collide with a pre-restart identifier than with a live one, so it carries the baseline guarantee with no persisted state and no dependence on holder lifetime. It is also the choice that avoids the structure leak of Section 8, and is the recommended default.
- **Persistent monotonic.** A counter whose high-water mark is durably recorded and never decreases across restarts — and, in a multi-instance deployment, coordinated across instances. Graph-wide uniqueness and never-reuse follow from never counting backward over the graph's lifetime. The cost is durable, crash-safe, possibly shared minting state; the benefit is compact, ordered identifiers.
- **Reset monotonic — the per-run relaxation.** A counter that resets at each restart, rendered as the identifier. It keeps no record of issued identifiers and needs no persistence — its sole appeal — but it satisfies _only_ the relaxed, per-epoch form of guarantees 2–3 (Section 2), and is correct **only** for a deployment that can prove every holder dies with the minting state (Section 8). A deployment whose holders may outlive a restart MUST NOT use it. A deployment that prefers identifiers visibly distinct across runs MAY prefix a per-restart epoch label (e.g. `epoch ‖ counter`); a strictly increasing epoch label turns this scheme into the persistent-monotonic family above and recovers the baseline guarantee.

Two constraints make any of these correct, and are the load-bearing requirements behind Section 2:

- **Derive only from server-side, allocation-fresh state.** An identifier's seed MUST come from the server's own minting process — a counter tick, a fresh random draw — and MUST NOT be derived from any client-supplied or client-stable attribute. Deriving from a client attribute both risks collisions (two like clients colliding) and would smuggle in cross-session identity (Architecture §3, §3.2), which this binding does not provide (Section 7).
- **The space need only be large enough for its scope.** The identifier space MUST be large enough that it is never exhausted within the scope across which it must be unique — the lifetime of one graph under the baseline, or a single run under the relaxation. Per-graph scoping (Section 2) bounds this to one graph's population over that scope, not the server's cross-graph total, so a server hosting many graphs need not size any one space for the whole deployment. A scheme that could wrap and re-mint a value _within_ that scope would violate never-reuse (Section 2).

## 4. Lifetime and Retirement

A node identifier is born with the node and dies with it:

- **Stable for the lifetime.** The identifier does not change while the node exists, including across any number of the node's neighborhood changes (Architecture §3). An in-neighbor holding it may rely on it denoting the same node for as long as that node lives.
- **Retired into vacancy.** When the node departs (Core §4.4), its identifier is retired. Under the baseline, retirement is **permanent for the lifetime of the graph, and to no one**: the identifier becomes a tombstone, not a reservation. It is never minted for a later node in that graph (Section 2) and — crucially — it is never reattached to a returning client, because there is no notion of a returning client (Architecture §3). A later session is a new node with a new identifier. (Under the per-run relaxation of Section 2, retirement is permanent only within the run; a later epoch may re-mint the same value for an unrelated node, which is harmless because the run boundary ended every holder — Section 8.)

This is the _cross-lifetime_ half of the governing invariant (§2), and is what makes a stale identifier **unambiguously dead within its graph**. For as long as a client that learned an identifier still lives, that identifier either still denotes the same live node or denotes nothing at all; it can never have come to denote a _different_ node — so the holder can never mistake a new out-neighbor for one it previously addressed under that designator. The "absolute certainty that a previously held identifier is no longer the same node," which the architecture requires before a handle may be treated as stale (Architecture §3), is here structural and — under the baseline — holds even across a server restart: within the graph a retired identifier resolves to nothing, and is never re-minted to mean something else (Section 8).

## 5. The Designator Is the Node's Identity

Under this binding, a node's identifier **is** its designator: every in-neighbor that has the node as an out-neighbor denotes it by that one identifier (Relay §2). Two consequences:

- **Per-state distinctness holds for free** — this is the _within-state_ half of the governing invariant (§2). Because every node's identifier is unique within its graph, and a node's out-neighbors are necessarily nodes of that same graph, the designators of any node's out-neighbors are automatically distinct within every neighborhood state — the base requirement (Relay §2) is met with no further effort, and no node can mistake two of its current out-neighbors for each other.
- **The designator carries cross-state meaning, within a lifetime.** Unlike the base designator, which fixes no meaning beyond a single neighborhood state (designator-string §3), this identifier is stable across the node's lifetime (Section 4). A client MAY therefore compare it across its own successive neighborhood states to recognize "the same out-neighbor" as the graph changes around it — the cross-state meaning the base leaves to a future layer (Relay §2), supplied here for the node's lifetime. Under the baseline this remains sound even for a client whose own node outlives a server restart, since the identifier is unique for the whole lifetime of the graph; a deployment that takes the per-run relaxation (Section 2) instead relies on no node's lifetime spanning a restart, which is precisely the condition that relaxation demands (Section 8).

Because all of a node's in-neighbors share the one identifier for it, two of them can — out of band — discover that they address a common out-neighbor. This is inherent in giving the node a single identity and is intended; it is also an information exposure the per-edge base designator does not have (Section 8).

Whether the server also discloses a node's _own_ identifier to it is out of scope here; doing so bears nothing on self-scoping (Core §6), which rests on the session, not on naming.

## 6. Resolution and No-Misdelivery

Resolution is unchanged from the base: the server resolves a `Send` designator against the sender's **current** neighborhood and relays to the denoted out-neighbor, or discards (Relay §3, §4). This binding does not touch that rule; it only constrains what identifiers the server may issue.

The interaction is wholly favorable. Because an identifier is never reused within the graph (Section 4), a `Send` carrying a stale identifier — one whose node has departed — resolves to nothing and is **discarded**, exactly as the base requires, with no possibility that it now denotes a different node. No-misdelivery (Relay §3) holds as before — indeed it holds regardless of reuse, since the server always resolves against the sender's _current_ neighborhood (Relay §4) — and the one residual hazard the base tolerates, a designator silently coming to denote a _different_ out-neighbor after reuse, is eliminated **for the lifetime of every node that could hold it** (Section 5). A client therefore needs no version or generation counter to tell a stale identifier from a current one: stale resolves to discard. This is the relay-facing expression of the governing invariant (§2) — the server never resolves a held designator to an out-neighbor other than the one its holder means. Under the baseline this holds unconditionally, whether or not any holder survives a restart, because the value is simply never re-minted; under the per-run relaxation the same safety rests instead on the restart ending every holder that could misread it (Section 8).

This is also why per-graph scoping (Section 2) is sufficient. Resolution is inherently confined to the sender's neighborhood, which lies wholly within one graph; an identifier value reused for a node of a _different_ graph is never consulted while resolving this graph's sends, so a cross-graph collision can confuse no one — the spatial counterpart of the way a cross-epoch collision cannot reach a holder under the per-run relaxation.

## 7. What This Binding Does Not Provide

- **No persistence across sessions.** This is the _ephemeral_ answer to the identity-persistence axis (Architecture §3). A node that departs and a client that later returns are unrelated nodes with unrelated identifiers; nothing an in-neighbor held survives the departed node's death. Graph-wide uniqueness guarantees the returning node's identifier _differs_ from the old one — never that the old one is restored. A deployment needing durable identity wants the _other_ answer to the axis — a persistent-identity binding (e.g. a public key as a durable identifier; Architecture §3) — not this one.
- **No participant identity.** That two sessions belong to the same real-world client is neither expressed nor expressible here; the server has no such concept (Architecture §3). Any continuity of _participant_ identity is constructed above the relay (Architecture §3.2), independent of these identifiers.
- **No ordering or versioning of neighborhood state.** An identifier names a node, not a state; it says nothing about which of two neighborhood states is newer. State ordering remains deferred (Core §3).

## 8. Security Considerations

This binding inherits Core §6 and Architecture §8, and — where it is realized as a string — the designator considerations of designator-string §7, adding only what fixing identity makes concrete.

- **Identifiers are not capabilities.** As in the base (designator-string §7), no-misdelivery rests on resolving every `Send` against the caller's _current_ neighborhood, not on an identifier being unguessable. A guessed or forged identifier relays only if it already denotes one of the caller's own current out-neighbors, and is otherwise discarded. Predictable identifiers (e.g. a monotonic counter) are therefore safe for correctness.
- **A predictable identifier leaks structure.** A monotonic scheme exposes node creation order and rough population — and, because the identifier is unique across the graph, lets holders correlate a shared out-neighbor (Section 5). A deployment wishing to avoid this SHOULD mint high-entropy random identifiers (Section 3), which are also the recommended default; this is a confidentiality choice, not a correctness one.
- **Uniqueness is unconditional under the baseline; the relaxation reintroduces a condition.** Under the baseline (graph-wide uniqueness — Section 2), whether a holder survives a restart does not affect correctness: a surviving holder's identifier is never re-minted, so it can neither misdeliver (Section 6) nor be mistaken for a new node (Section 5). The per-run relaxation (Sections 2, 3) trades this safety away in exchange for a reset-able counter, and is correct only where **every holder dies with the server's minting state**. The configuration that breaks it is a holder that _survives_ that loss — e.g. a Pull session deliberately engineered to outlive a restart while a reset counter runs beneath it (`../interface-profiles/pull/rpc-pull-profile.md` §3). Such a deployment MUST use the baseline (random, or persistent-monotonic), not the relaxation. Holders that die with the server — every Pushable-Profile connection (`../interface-profiles/push/rpc-push-profile.md` §3), and any Pull session bound to server-side state (`../interface-profiles/pull/rpc-pull-profile.md` §3) — MAY take the relaxation. When in doubt, do not: the baseline costs a large random space or a persisted counter and removes the question entirely.
- **Per-graph scoping assumes disjoint graphs.** Minting an independent identifier space per graph (Section 2) is safe only because resolution never crosses a graph boundary (Section 6): each `Send` is resolved within the sender's neighborhood, which lies wholly within one graph, so a value reused in another graph is never consulted. A deployment that lets an edge or a node bridge two graphs breaks this assumption and MUST make uniqueness span the bridged domain.

## 9. References

### 9.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- Graph Relay System (GRS) Protocol (`../../architecture.md`).
- GRS Relay and Neighborhood Semantics (`../../relay-and-neighborhood-semantics.md`).
- GRS RPC Common Core (`../interface-profiles/rpc-interface.md`).

### 9.2. Informative References

- GRS Designator String Representation (`../data-shapes/designator-string.md`): the representation binding this one composes with; under it, an identifier is a string, and §4's "global uniqueness" and "stability across states" are the strengthenings this binding makes mandatory — the former scoped here to the graph (Section 2).
- GRS Payload String Representation (`../data-shapes/payload-string.md`): a sibling cross-cutting binding.
- GRS RPC Pull Profile (`../interface-profiles/pull/rpc-pull-profile.md`) and GRS RPC Pushable Profile (`../interface-profiles/push/rpc-push-profile.md`): the transport profiles whose session/connection lifecycle determines whether a holder can outlive a restart (Section 8) — and so whether the per-run relaxation (Section 2) is available to that deployment or the baseline graph-wide uniqueness is required.
- RFC 4122: A Universally Unique IDentifier (UUID) URN Namespace — one example random mechanism for Section 3.
