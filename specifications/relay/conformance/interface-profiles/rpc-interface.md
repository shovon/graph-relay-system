# GRS RPC Common Core

## Status of This Memo

This document is a companion specification to the _Graph Relay System (GRS) Protocol_ (`../../architecture.md`) and its _Relay and Neighborhood Semantics_ companion (`../../relay-and-neighborhood-semantics.md`). It fixes the **common core** of the GRS remote-procedure interface: the parts that do not depend on how a client and server exchange messages — the abstract data, the semantics of the operations a client invokes on the server, and the two server-owed responsibilities the relay rests on.

It deliberately does **not** fix how server-originated events reach a client, nor how a node is established and its departure detected. Those depend on whether the transport can carry server-initiated messages, and they are fixed by one of two companion **profiles**, exactly one of which an implementation adopts:

- **GRS RPC Pull Profile** (`pull/rpc-pull-profile.md`) — for request/response, client-initiated-only transports (e.g. HTTP), where the server cannot push and a session must be synthesized from independent requests.
- **GRS RPC Pushable Profile** (`push/rpc-push-profile.md`) — for full-duplex, session-oriented transports (e.g. WebSocket, raw TCP), where the server can push and a connection _is_ the session.

This document is normative for both profiles. Where the two profiles differ, this core states the _responsibility_ and defers the _mechanism_ to the profile; a profile MUST fix each such mechanism concretely (it is not left open within a profile). Section references of the form (Architecture §N) point into `../../architecture.md`; (Relay §N) into `../../relay-and-neighborhood-semantics.md`.

## Table of Contents

1. Terminology
2. Interface Model
3. Abstract Data
4. Operations and Responsibilities 4.1. `Send` (client → server) 4.2. Neighborhood State Availability (mechanism deferred to profile) 4.3. The Receiving Half of the Relay (mechanism deferred to profile) 4.4. Sessions and Node Lifecycle (mechanism deferred to profile)
5. What a Profile Fixes
6. Security Considerations
7. References

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

This document uses the terms **Server**, **Client**, **Node**, **out-neighbor**, **in-neighbor**, **neighborhood**, **neighborhood state**, and **Designator** as defined in Architecture §2 and refined in Relay §2.

A **procedure** (or **operation**) is an abstract, named action with a direction, zero or more inputs, and zero or more outputs. A **client-initiated** procedure is invoked by a client and serviced by the server. A **server-originated** event is one whose occurrence the server decides (a relayed message arriving for a node; a neighborhood change). The core defines client-initiated operations and the _requirement_ that server-originated events reach the client; a profile fixes the mechanism by which they do.

A **session** is the association between a client and the server for the lifetime of one node: it begins when the node is established, identifies the node on every operation, and ends when the node departs. A session is provided by the layer beneath the RPC profile — a native transport connection, or a session layer over a connectionless transport (Section 4.4).

A **profile** is one of the two companion documents above. An implementation MUST adopt exactly one profile and MUST implement this core as that profile specializes it.

## 2. Interface Model

A client participates in GRS as a single node (Architecture §3). Every operation in this core is implicitly scoped to _the caller's own node_: no operation takes a "self" node identifier as a parameter, and a client never names itself — only, by designator, its out-neighbors. This keeps the interface aligned with directionality (Architecture §3.1): the only nodes a client can name are its current out-neighbors. _How_ the caller's node identity is carried — by the session the operation belongs to, whether a native transport connection or a session synthesized by a session layer — is a profile concern (Section 4.4), but the self-scoping property holds regardless.

The interface has exactly two responsibilities, mirroring the two the server owes a client (Architecture §4):

1. **Make neighborhood state available** to the node — its set of current out-neighbors and their designators (Relay §5).
2. **Carry the relay primitive** in both directions — the sending half (`Send`, Section 4.1) and the receiving half (Section 4.3).

Only the sending half is client-initiated and identical in semantics across profiles, so the core fixes it as a concrete operation (Section 4.1). The other three concerns — making neighborhood state available, the receiving half, and node lifecycle — differ fundamentally by transport, so the core states each as a responsibility and each profile fixes the mechanism (Sections 4.2–4.4).

## 3. Abstract Data

This document defines the following abstract types. A profile, or a transport binding beneath it, fixes their concrete representation.

- **Payload**: An opaque, application-defined value carried by the relay. The interface neither inspects nor interprets it. Its maximum size is transport-defined (Relay §7).
- **Designator**: A handle denoting one out-neighbor, with the semantics of Relay §2 — distinct among a node's current out-neighbors, but not necessarily globally unique, persistent, or stable across neighborhood changes. What a designator means from one neighborhood state to the next is left to a future layer (Relay §2).
- **NeighborhoodState**: The set of the node's current out-neighbors, each represented by its designator (Relay §2). The set MAY be empty, which is a well-formed state and not an error (Relay §2). The ordering and versioning of successive neighborhood states — how a client tells a newer state from an older one — is intentionally out of scope for this revision and is deferred to a future document.

## 4. Operations and Responsibilities

### 4.1. `Send` (client → server)

- **Direction**: client-initiated.
- **Input**: a `Designator` and a `Payload`, plus the caller's node identity (Section 4.4).
- **Output**: at most an **acceptance decision** (accepted for relay attempt, or discarded). A profile MAY define this operation as one-way, surfacing no output at all.

The server relays the payload to the out-neighbor the designator denotes **at the moment of sending**, or discards it; it MUST NOT deliver it to any other node (No-Misdelivery, Relay §3). A designator that does not denote a current out-neighbor — whether malformed, or no longer denoting one after a graph change — results in discard, never misdelivery (Relay §4).

The relay is **best-effort** (Architecture §6, Relay §6):

- An acceptance decision, where surfaced, reports **only the server's acceptance**, not the message's fate at the receiver. A sender MUST NOT infer delivery from acceptance, nor from the absence of a discard indication.
- The server MUST make a genuine attempt to relay a validly addressed message and MUST NOT systematically decline to do so (Relay §6).

This is the protocol's one interaction primitive (Architecture §3.2): the interface defines no traversal, no broadcast, and no end-to-end send to a non-neighbor. Reach beyond the immediate neighborhood, and any reply path, is constructed by clients above this operation (Architecture §3.1, §3.2).

### 4.2. Neighborhood State Availability (mechanism deferred to profile)

A client can address `Send` (Section 4.1) only by designating a current out-neighbor, so the server MUST make a node's current neighborhood state available to it (Architecture §4, Relay §5). Whatever mechanism a profile fixes, the state made available MUST reflect the latest committed configuration at the moment it is made available (Relay §5).

How that state reaches the client depends on whether the transport can carry server-originated messages, and each profile MUST fix one mechanism:

- In a **pull** transport, the client obtains the state on demand by querying (`GetNeighborhood`, defined by `pull/rpc-pull-profile.md`); the recency of its view is bounded by its own query cadence, and re-querying is the only way it observes a change.
- In a **pushable** transport, the server pushes the updated state to the affected node whenever it changes (`NeighborhoodUpdate`, defined by `push/rpc-push-profile.md`), and MAY additionally answer an on-demand query for explicit re-sync.

The ordering and versioning of successive neighborhood states remains out of scope (Section 3).

### 4.3. The Receiving Half of the Relay (mechanism deferred to profile)

`Send` (Section 4.1) has no observable effect unless the destination node can obtain the payloads relayed to it by its in-neighbors. The server therefore MUST provide a receiving half, and every profile MUST fix exactly one mechanism for it:

- In a **pushable** transport, the server delivers each relayed payload to the destination node as a server-originated event (`Deliver`, defined by `push/rpc-push-profile.md`).
- In a **pull** transport, the server buffers relayed payloads for the node and the client drains them by polling (`Receive`, defined by `pull/rpc-pull-profile.md`).

Whichever mechanism a profile fixes, it MUST honor the same invariants this core and the companions impose on the relay: best-effort delivery with no acknowledgement guarantee (Relay §6), and no sender designator or reply path implied by receipt (Architecture §3.1). A relayed payload is handed to the receiver with no obligation to identify its originator; a client requiring origin or a reply path constructs it within the `Payload`, above this interface (Architecture §3.2).

### 4.4. Sessions and Node Lifecycle (mechanism deferred to profile)

A node exists for the duration of a **session** (Section 1): the session establishes the node, the node's identity on every operation is the session it belongs to, and the session's end is the node's departure, after which the graph is repaired (Architecture §3, §4). The server MUST be able to detect that end so it can repair the graph. Crucially, the RPC profile does not itself define how a session begins or ends — it treats session establishment as node creation and session end as departure, and obtains the session from the layer beneath it. What provides that layer depends on the transport, and each profile MUST fix it:

- Where the transport supplies a **persistent connection** (pushable), that connection _is_ the session: opening it establishes the node, the connection carries node identity implicitly, and the server detects departure from the connection closing or failing a liveness check. Establishment and teardown are the transport's own (e.g. a TCP handshake and close), so the profile defines no session operations of its own.
- Where the transport supplies **no persistent connection** (pull), the session is synthesized by a **session layer** beneath the profile: that layer establishes a session, carries its identity on each request, refreshes its liveness, and tears it down by explicit close or by timeout. The RPC profile layers on that session exactly as the pushable profile layers on a connection, and likewise defines no session operations of its own.

Either way, under the base default the server attaches no identity to a node across its lifetime boundary: a node that departs and a later one that arrives are unrelated, and no session identity, designation, or position carries over (Architecture §3, §3.2). A derived specification MAY instead define a durable node identity that a later session re-binds (Architecture §3); where it does not, identity continuity, if an application needs it, is constructed above this interface.

## 5. What a Profile Fixes

Beyond the per-profile mechanisms of Sections 4.3 and 4.4, the following are left to a profile (and to any transport binding beneath it), consistent with Relay §7:

- The encoding and framing of operations and of the abstract data of Section 3, including the concrete form of a `Designator` and of `Payload`, and the maximum payload size.
- The call mechanics of each operation: request/response versus one-way, and correlation of responses.
- Whether, and how, an acceptance decision (Section 4.1) is surfaced, and whether any discard indication is reported (Relay §4).
- Ordering of relayed payloads, buffering and retention of pending messages, the fate of messages held for a departing node, and any reliability or deduplication beyond the best-effort floor (Relay §6, §7).

The way this document and its profiles **name** operations and **list** their abstract inputs is itself part of what is left open. An operation's name — `Send`, and those each profile adds (`Deliver`, `NeighborhoodUpdate`, `GetNeighborhood`, `Receive`, and the like) — is a label this specification uses to refer to the operation, not the token that selects it on the wire. The order in which an operation's inputs are listed — for `Send`, a `Designator` then a `Payload` — states _what the operation takes_, not the positions those inputs occupy in any concrete message. A binding (a profile, or a transport binding beneath it) fixes the concrete selector for each operation and the concrete layout of its inputs, and MAY choose both independently of the spelling and listing order used here, provided the operation's semantics (Section 4) are preserved. Nothing in how an operation is named or its inputs are ordered in this document constrains a concrete message shape.

Clients requiring end-to-end guarantees — reliable delivery, ordering, acknowledgement, identity continuity across sessions, or a reply path to a node with no edge toward them — construct them above this interface (Architecture §3.2), not by expecting them from these operations.

## 6. Security Considerations

This core inherits the considerations of Architecture §8; each profile adds those specific to how it identifies a node (Section 4.4).

- **Self-scoping is authority.** Because every operation is scoped to the caller's own node (Section 2) and no operation accepts a "self" identifier, a client cannot name or act as another node merely by argument. A profile MUST ensure a caller can invoke operations only as its own node, so that `Send` cannot be issued on behalf of, and the receiving half cannot be redirected to, a node other than the caller's own. The means by which a profile binds a caller to a node — a transport connection, or a session-layer session — is therefore security-critical (Section 4.4).
- **Resolution is against the current neighborhood.** A profile MUST resolve a `Send` designator against the sending node's _current_ neighborhood, so that a designator which does not denote a current out-neighbor — whether stale, guessed, or malformed — yields discard, never misdelivery (Relay §3, §4) and never reach to a non-neighbor. Designators are therefore not capabilities: holding or guessing one grants nothing beyond what the caller's current neighborhood already allows.
- **Acceptance is not delivery.** Because `Send` may surface an acceptance decision (Section 4.1), implementers and clients MUST NOT treat it as a delivery signal; doing so would reintroduce a guarantee the relay explicitly disclaims (Relay §6).
- **Confidentiality.** Every payload traverses the server (Architecture §8); this interface provides no end-to-end confidentiality. Clients requiring it encrypt within `Payload`, above this interface.

Authentication of the caller, admission control, and rejection of spoofed designations or identities are shared between the profile and Architecture §8.

## 7. References

### 7.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- Graph Relay System (GRS) Protocol (`../../architecture.md`).
- GRS Relay and Neighborhood Semantics (`../../relay-and-neighborhood-semantics.md`).

### 7.2. Informative References

- GRS RPC Pull Profile (`pull/rpc-pull-profile.md`): specializes this core for request/response, client-initiated-only transports. It layers on the GRS RPC Pull Session Layer (`pull/rpc-pull-session.md`), which synthesizes the session a connectionless transport does not natively provide.
- GRS RPC Pushable Profile (`push/rpc-push-profile.md`): specializes this core for full-duplex, session-oriented transports, where the transport connection is itself the session.
