# GRS RPC Pull Profile

## Status of This Memo

This document is a profile of the _GRS RPC Common Core_ (`../rpc-interface.md`) for **request/response, client-initiated-only transports** — environments in which the server cannot send a message to a client except as the response to a request the client made. Plain HTTP is the archetype.

It **assumes a session** beneath it — an association with a definite beginning, a carried identity, a liveness notion, and a definite end — over the connectionless transport that does not provide one natively. How that session is supplied is out of scope here and is the implementer's responsibility, exactly as the _GRS RPC Pushable Profile_ (`../push/rpc-push-profile.md`) assumes a full-duplex connection without specifying WebSocket or TCP; `rpc-pull-session.md` is one example realization (Section 3). With a session assumed, this profile is structurally parallel to the Pushable Profile: both treat a session as a node and define only the GRS operations carried within it. The single way this profile remains heavier is the receiving half of the relay — because even with a session the server still cannot push, that half stays a client poll (Section 5).

This profile is normative for implementations claiming the GRS Pull Profile. It depends on, and does not restate, the abstract data and the `Send` semantics of the core; it is here that the core's session-and-node-lifecycle responsibility (Core §4.4) is fixed — by binding the node to an assumed session (Section 3) — and the core's neighborhood-state-availability responsibility (Core §4.2) is fixed, as a pull query. Section references (Core §N) point into `../rpc-interface.md`, and (Architecture §N) / (Relay §N) into the respective companions.

## Table of Contents

1. Terminology
2. Why This Profile Differs
3. The Session Is the Node
4. Operations 4.1. `GetNeighborhood` (client → server) 4.2. `Send` (client → server) 4.3. `Receive` (client → server)
5. The Inbox Model
6. Observing Neighborhood Changes
7. Security Considerations
8. References

## 1. Terminology

The key words "MUST", "MUST NOT", etc. are to be interpreted as described in RFC 2119. This profile uses all terms of the core (Core §1) and adds:

- **Session**: An assumed substrate association between a client and the server, with a definite beginning, a carried identity, a liveness notion refreshed by use, and a definite end (by client request or by timeout). This profile requires such a session but does not specify how it is provided; `rpc-pull-session.md` is one example realization.
- **Inbox**: A server-held buffer of payloads relayed to a node but not yet retrieved by it — the pull reconstruction of the relay's receiving half (Core §4.3).

## 2. Why This Profile Differs

A request/response transport differs from a full-duplex one in two ways the GRS layering keeps separate:

1. **No native session.** There is no connection whose lifetime is the node's. The session a node lives in must therefore be supplied by some mechanism beneath this profile, which it assumes exactly as the Pushable Profile assumes a transport connection. Providing that session — establishment, identity, liveness, teardown — is consequently the substrate's concern, **not** this profile's: this profile defines no join or leave operation, because the session's own beginning and end already are them.
2. **No server push.** Even with a session, the server still answers only when asked. The receiving half of the relay (Core §4.3) therefore cannot be a push; it remains a client poll against a server-held **inbox** (Sections 4.3, 5). This is the _sole_ remaining difference between this profile and the Pushable Profile, and it is irreducible: a session confers identity and liveness, never a channel the server can speak down.

Everything else — the self-scoping model, `Send` semantics, the security invariants — is inherited unchanged from the core; this profile additionally fixes how neighborhood state is made available (Core §4.2), by query (Section 4.1).

## 3. The Session Is the Node

This profile binds the assumed session's lifecycle to the graph: **each session is one GRS node.**

- When a session is _established_, the server creates the node and places it in the graph (Architecture §3).
- While the session lasts, every operation in Section 4 is issued **within that session**; the session identifies the node, so — symmetrically with the Pushable Profile's connection-scoped operations — none of these operations carries a node identifier or handle of its own, and each is scoped to the session's node (Core §6, self-scoping). Each such request also refreshes the session's liveness.
- When the session _ends_ — by the client's request or by liveness timeout — the node departs and the graph is repaired around its absence (Architecture §4).

This profile therefore defines no establishment or teardown of its own; those are the session's beginning and end, owned by whatever mechanism supplies it. By default a new session is a new, unrelated node, inheriting nothing from any prior one (Architecture §3, §3.2); a derivative MAY instead let a durable node identity (Architecture §3) be re-bound on a later session, distinct from the ephemeral session credential, which is not reused (`rpc-pull-session.md` §6).

To start with a current view, a client SHOULD call `GetNeighborhood` (Section 4.1) as its first request after the session is established. As an optimization, where the session mechanism allows an opaque payload on its establishment response, an implementation MAY carry the node's initial `NeighborhoodState` there, sparing that first round trip; the session mechanism remains unaware of what that payload means.

`rpc-pull-session.md` specifies one mechanism that supplies such a session; an implementation MAY use it or any other mechanism providing the same beginning/identity/liveness/end abstraction (a transport-native session, a cookie-based session, etc.). It is referenced here only as an example, not as a layer this profile depends on by name.

## 4. Operations

Each operation below is carried within an established session: the session identifies the node, and — like any request in the session — it refreshes the session's liveness. None takes a node identifier. Each inherits its semantics, where applicable, from the core.

### 4.1. `GetNeighborhood` (client → server)

- **Input**: none (the session identifies the node).
- **Output**: the node's current `NeighborhoodState`.

This is the pull mechanism by which this profile satisfies the core's neighborhood-state-availability responsibility (Core §4.2): the client obtains its current out-neighbors, and the designators it addresses `Send` with, by querying on demand. The server MUST answer with state reflecting the latest committed configuration (Relay §5). A client MAY query at any time; because this profile has no push, re-invoking it is also the _only_ way a client observes a neighborhood change (Section 6).

### 4.2. `Send` (client → server)

- **Input**: a `Designator` and a `Payload`.
- **Output**: at most an acceptance decision (Core §4.1).

As Core §4.1, with no change to relay semantics: best-effort, no-misdelivery, designator resolved against the node's current neighborhood (Relay §3, §4).

### 4.3. `Receive` (client → server)

- **Input**: none.
- **Output**: the payloads currently buffered in the node's inbox (Section 5), possibly empty.

This is the pull reconstruction of the relay's receiving half (Core §4.3). The server returns the payloads relayed to the node since its last successful `Receive` and removes them from the inbox (drain-on-read). An empty result is normal and not an error.

The server MAY hold the request open until at least one payload is available or a deployment-defined timeout elapses (long poll), or MAY return immediately with whatever is buffered (short poll); this choice is an efficiency matter and changes no guarantee. Each returned payload carries no sender designator and no reply path (Core §4.3, Architecture §3.1).

## 5. The Inbox Model

The inbox is the server-held buffer that stands in for a push channel. Its contract is exactly the core's relay floor (Core §4.3, Relay §6), realized as storage rather than delivery:

- **Best-effort retention.** The server SHOULD retain a relayed payload until the node next drains it, but the inbox is finite: under pressure (size or age limits) the server MAY drop buffered payloads. As with the relay floor, it MUST NOT _systematically_ drop conformant traffic — an inbox that discards everything as a matter of course does not conform (Relay §6).
- **Drain-on-read, at-most-once.** `Receive` removes what it returns. The profile does not define acknowledgements, cursors, or redelivery; a payload returned by `Receive` but lost before the client uses it is gone. Reliability beyond this floor is out of scope and is deferred to a future document (Relay §7).
- **Ordering deferred.** The order in which `Receive` returns buffered payloads is not fixed by this profile; ordering, like neighborhood-state versioning (Core §3), is deferred to a future document.
- **Fate on session end.** When the session ends and the node departs (Section 3), its inbox MAY be discarded; messages held for a departed node are not preserved (Relay §7).

## 6. Observing Neighborhood Changes

This profile has no neighborhood-change push. A client learns its current out-neighbors only by calling `GetNeighborhood` (Section 4.1), and learns of a _change_ only by calling it again. The recency of a client's view is therefore bounded by its own poll cadence (Relay §5). Whatever a client does between polls, the relay stays safe: a `Send` whose designator does not denote a current out-neighbor is discarded, never misdelivered (Relay §3, §4).

Push-based neighborhood notification is intentionally absent here; it is a feature of the companion Pushable Profile (`../push/rpc-push-profile.md`), which the underlying transport makes possible there and not here.

## 7. Security Considerations

This profile inherits Core §6 and Architecture §8. Any credential the session mechanism uses to bind a caller to its session belongs to that mechanism, and its protection is specified wherever that mechanism is; a deployment MUST heed those considerations, but they are not restated here. What remains specific to this profile:

- **Node authority rides on the session.** This profile realizes the core's self-scoping requirement (Core §6) through the session: the server MUST scope every operation, and every inbox drain, to the node of the session the request belongs to. Because the session is the only thing binding a caller to a node, node authority here is exactly as strong as the session mechanism's protection of whatever credential it uses; if that credential is a bearer token, its disclosure compromises the node until the session ends.
- **Designators and acceptance.** A designator is not a capability, and an acceptance decision is not a delivery signal; both are inherited unchanged from the core (Core §6).
- **Churn.** Session establishment creating per-node graph state means a flood of new sessions forces continuous graph repair (Architecture §8). Rate limiting on session establishment is the session mechanism's concern; this profile additionally inherits the architecture's general churn considerations.
- **Confidentiality.** Every payload traverses the server; this profile provides no end-to-end confidentiality (Core §6, Architecture §8).

## 8. References

### 8.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- GRS RPC Common Core (`../rpc-interface.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).
- GRS Relay and Neighborhood Semantics (`../../../relay-and-neighborhood-semantics.md`).

### 8.2. Informative References

- GRS RPC Pull Session Layer (`rpc-pull-session.md`): one example mechanism for supplying the session this profile assumes; any mechanism providing the same abstraction conforms.
- GRS RPC Pushable Profile (`../push/rpc-push-profile.md`): the companion profile for full-duplex transports.
