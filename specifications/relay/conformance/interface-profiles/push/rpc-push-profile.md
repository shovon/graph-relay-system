# GRS RPC Pushable Profile

## Status of This Memo

This document is a profile of the _GRS RPC Common Core_ (`../rpc-interface.md`) for **full-duplex, session-oriented transports** — environments in which the server can send a message to a client at any time, not only in response to a request, and in which a persistent connection underlies the session. WebSocket and raw TCP are the archetypes.

It specializes the two responsibilities the core defers (Core §4.3, §4.4) in the direct way such a transport allows: the receiving half of the relay is a genuine server push, and a node's lifetime is the connection's lifetime. This makes it the lighter of the two profiles — the architecture's "node existence is coterminous with the session" (Architecture §3) holds literally here, so no establishment or teardown procedure is needed. It is normative for implementations claiming the GRS Pushable Profile, and depends on, without restating, the abstract data and the `Send` semantics of the core; it is here that the core's neighborhood-state-availability responsibility (Core §4.2) is fixed, primarily by push. Section references (Core §N) point into `../rpc-interface.md`; (Architecture §N) and (Relay §N) into the respective companions.

## Table of Contents

1. Terminology
2. Why This Profile Differs
3. The Connection Is the Session
4. Liveness and Departure
5. Operations 5.1. `Send` (client → server) 5.2. `Deliver` (server → client) 5.3. `NeighborhoodUpdate` (server → client)
6. Ordering
7. Security Considerations
8. References

## 1. Terminology

The key words "MUST", "MUST NOT", etc. are to be interpreted as described in RFC 2119. This profile uses all terms of the core (Core §1). A **server-initiated** message (a _push_) is one the server sends to a client on its own initiative over the open connection, without a prior request to answer; `Deliver` (Section 5.2) and `NeighborhoodUpdate` (Section 5.3) are the two such messages this profile defines.

## 2. Why This Profile Differs

A full-duplex, session-oriented transport removes the two limitations that shape the Pull Profile:

1. **The server can push.** The receiving half of the relay (Core §4.3) is therefore a real server-originated delivery (`Deliver`, Section 5.2) — no inbox, no polling. The same capability lets the server push neighborhood changes (`NeighborhoodUpdate`, Section 5.3) rather than make the client re-query for them.
2. **The connection is persistent.** Node existence (Core §4.4) is coterminous with the connection: opening it establishes the node, the connection itself identifies the node on every operation, and closing it (or detecting its failure) is departure. No session handle, `Join`, or TTL plumbing is needed.

Everything else — self-scoping, `Send` semantics, the security invariants — is inherited unchanged from the core; this profile additionally fixes how neighborhood state is made available (Core §4.2), entirely by push (Section 5.3).

## 3. The Connection Is the Session

Opening a connection to the server establishes the caller's node; the server places it in the graph (Architecture §3). There is **no `Join` operation and no session handle**: the connection itself is the node's identity, and the server scopes every operation to the node on whose connection it arrives (Core §6, self-scoping). Closing the connection destroys the node (Section 4).

By default the server attaches no identity across connections: a client that reconnects establishes a _wholly new_ node, inheriting nothing from any it previously held (Architecture §3, §3.2). Any identity continuity is then constructed above this interface — unless a derivative provides a persistent node identity at the substrate (Architecture §3), re-bound when the client reconnects and proves it.

On establishment the server SHOULD push the node's initial `NeighborhoodState` via `NeighborhoodUpdate` (Section 5.3) so the client begins with a current view from the start.

## 4. Liveness and Departure

Departure is signalled by the connection. The server MUST treat a cleanly closed connection as departure: the node ceases to exist and the graph is repaired around its absence (Architecture §4). To detect a connection that has failed without a clean close (a half-open connection), the server SHOULD use a transport liveness mechanism — WebSocket ping/pong, TCP keepalive, or an application-level heartbeat — and treat a node whose connection fails such a check as departed. No client operation is required to leave; closing the connection suffices.

## 5. Operations

Client-initiated operations carry no node identifier — the connection identifies the node (Section 3). Server-initiated messages are delivered on that same connection.

### 5.1. `Send` (client → server)

- **Input**: a `Designator` and a `Payload`.
- **Output**: at most an acceptance decision (Core §4.1); MAY be one-way.

As Core §4.1, with no change to relay semantics: best-effort, no-misdelivery, designator resolved against the node's current neighborhood (Relay §3, §4).

### 5.2. `Deliver` (server → client)

- **Direction**: server-initiated (push).
- **Input**: a `Payload`.
- **Output**: none. One-way; the relay guarantees no acknowledgement, and a client owes none.

This is the receiving half of the relay (Core §4.3): the server pushes to a node each payload relayed to it by one of its in-neighbors, as it arrives. The payload carries **no sender designator** and no reply path — receiving a message confers no ability to answer it (Architecture §3.1). Delivery is best-effort (Relay §6); a client MUST NOT assume every relayed message will arrive, nor that its absence signals anything.

### 5.3. `NeighborhoodUpdate` (server → client)

- **Direction**: server-initiated (push).
- **Input**: the node's new `NeighborhoodState`.
- **Output**: none. One-way.

This is the push path to neighborhood-state availability (Core §4.2, Relay §5). When a node's neighborhood changes — an out-edge added, removed, or retargeted — the server MUST push the updated `NeighborhoodState` to the affected node, and SHOULD do so promptly after the change is committed (Relay §5). The server SHOULD also push the initial state on establishment (Section 3). Because this profile pushes every change and the transport delivers them in order (Section 6), a client's most recently received state is always its current one — it stays current without polling.

## 6. Ordering

This profile assumes the transport delivers messages on a connection **in order** (WebSocket and TCP both do). Under that assumption the most recently received `NeighborhoodState` is the current one, and a client may treat each `NeighborhoodUpdate` as superseding the last without needing an explicit version. This profile therefore defines no neighborhood-state versioning of its own; should it ever be carried over an unordered transport, the versioning needed to distinguish newer state from older is deferred, with the rest of state versioning, to a future document (Core §3). Ordering of relayed `Deliver` payloads beyond what the connection provides is likewise out of scope (Relay §7).

## 7. Security Considerations

This profile inherits Core §6 and Architecture §8, and adds little of its own because it introduces no bearer credential:

- **The connection binds caller to node.** This profile realizes the core's self-scoping requirement (Core §6) through the connection itself: the server MUST scope every operation, and every `Deliver`/`NeighborhoodUpdate` push, to the node of the connection it arrives on or departs over. There is no handle to leak (contrast the Pull Profile), so node authority cannot be transferred by replaying a token; it rests entirely on control of the connection. The connection SHOULD therefore run over a confidential, integrity-protected channel (e.g. TLS/WSS), and authentication, where required, is performed at or above connection establishment (Architecture §8).
- **Connection churn.** Rapid connect/disconnect cycling forces continuous graph repair (Architecture §8); rate limiting on connection establishment is deployment-defined and RECOMMENDED.
- **Confidentiality.** As in the core, every payload traverses the server; this profile provides no end-to-end confidentiality (Core §6, Architecture §8).

## 8. References

### 8.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- GRS RPC Common Core (`../rpc-interface.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).
- GRS Relay and Neighborhood Semantics (`../../../relay-and-neighborhood-semantics.md`).

### 8.2. Informative References

- GRS RPC Pull Profile (`../pull/rpc-pull-profile.md`): the companion profile for request/response transports.
