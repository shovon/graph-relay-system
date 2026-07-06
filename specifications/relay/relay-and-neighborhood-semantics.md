# GRS Relay and Neighborhood Semantics

## Status of This Memo

This document is a companion specification to the _Graph Relay System (GRS) Protocol_ (`architecture.md`). Where the architecture overview fixes the _shape_ of the system — a server maintaining a strongly connected directed graph, relaying only along out-edges — this document fixes the finer semantics that overview deliberately leaves out: how a node names its out-neighbors, how neighborhood state is made current, and what the server's relay does and does not guarantee.

It is normative for implementations that claim to implement these aspects of GRS, but it is still one binding among possible others; a concrete transport binding is expected to fix the choices this document leaves open. Section references of the form (Architecture §N) point into `architecture.md`.

## Table of Contents

1. Terminology
2. Neighborhood State and Designators
3. The No-Misdelivery Property
4. Resolution Against the Current Neighborhood
5. Change Notification and State Availability
6. The Relay Contract
7. What the Transport Fixes

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

This document uses the terms **Server**, **Client**, **Node**, **out-neighbor**, **in-neighbor**, **neighborhood**, and **Designator** as defined in Architecture §2, refining the last three here.

## 2. Neighborhood State and Designators

A node's **neighborhood state** is the set of its current out-neighbors, each exposed in whatever form the designation scheme uses. This set is what the server makes available to the node (Section 5), and it is what a client uses to address relayed messages (Section 6).

A **designator** is the handle by which the server lets a node denote one of its out-neighbors. This revision fixes deliberately little about it:

- Within a single current neighborhood state, designators MUST be **distinct among the node's out-neighbors**, so that each out-neighbor can be addressed unambiguously. A client MUST be able to determine, from its neighborhood state, how to designate each node it is permitted to send to.
- That per-state distinctness is the **only** property this revision guarantees. A designator is **not** required to be globally unique, persistent across reconnection, or stable across a node's neighborhood changes, and the server MAY reassign or reuse one freely as the graph changes.

How designators are allocated, and whether a designator carries any meaning from one neighborhood state to the next, is **left to a future layer** and unspecified here. This revision neither grants a designator cross-state meaning nor places any obligation on how a client treats one it obtained from an earlier state; it fixes only the server-side resolution that makes the relay safe (Sections 3 and 4).

A derivative MAY strengthen a designator beyond per-state distinctness — for instance making it stable across a node's neighborhood changes, globally unique, or even persistent across the node's session boundary, the last where a derived identity model (such as a public key used as a durable node identifier; Architecture §3) makes the same designator denote the same node across reconnection. The base neither requires nor precludes any of these; whichever a derivative documents, the server-side resolution above keeps the relay safe regardless.

The neighborhood state MAY be empty: a node with no out-neighbors (Architecture §3) has nothing it can address, and that is a well-formed state, not an error.

## 3. The No-Misdelivery Property

The relay's central safety guarantee is that a message never reaches the wrong node:

> A message MUST be delivered only to the node its designator denoted **at the moment of sending**, or not delivered at all. It MUST NOT be delivered to any other node.

Equivalently: a send is only ever relayed to the out-neighbor its designator denotes at the moment of sending, or it is discarded — never delivered elsewhere. This holds together with the directionality rule of Architecture §3.1: because the only addressable targets are out-neighbors, and a designator resolves (Section 4) to at most one of them or to nothing, there is no path by which a message reaches a node that is not, at that instant, an out-neighbor of the sender.

The practical guarantee is therefore strong but narrow: a `Send` carries no risk of _misdelivery_, only a risk of _non-delivery_. A designator that no longer denotes a current out-neighbor costs at most a dropped message (Section 4); it can never cause one to land on the wrong recipient.

## 4. Resolution Against the Current Neighborhood

The server resolves the designator carried on a `Send` against the sending node's neighborhood **at the moment of sending**, and acts on the result in exactly one of two ways:

- if the designator denotes a current out-neighbor of the sender, the server relays the payload along that out-edge (subject to the best-effort contract of Section 6);
- otherwise — the designator is malformed, or simply does not denote a current out-neighbor — the server **discards** the payload. It is never relayed, and never routed to any other node (Section 3).

Because a graph change may add, remove, or retarget a node's out-edges at any time, a designator the server issued earlier may no longer denote a current out-neighbor, or may have come to denote a different one. This revision says nothing about how a client should treat such a designator, and imposes no obligation on it: whatever a designator means across neighborhood states, and how a client ought to track that, is for a future layer to define (Section 2). The server's only commitment is the resolution above — relay to a current out-neighbor or discard — which holds No-Misdelivery (Section 3) regardless of what the client believed the designator denoted.

## 5. Change Notification and State Availability

When a node's neighborhood changes — that is, when one of its out-edges is added, removed, or retargeted — the server MUST make the updated neighborhood state available to that affected node once the change is **committed**. The state the server makes available, whether pushed or returned in response to a query, MUST reflect the latest committed configuration.

The server MAY satisfy this in either of two ways, and the protocol is indifferent to the choice:

- **Push.** The server actively delivers the new state to the affected node. When it does, it SHOULD do so promptly after commit.
- **Pull.** The server ensures the node can look the state up on demand. Under a pull model the recency of a node's view is bounded by its own query cadence, not by a server obligation to notify.

Clients SHOULD be able to query the server for their current neighborhood state at any time, and the server MUST respond accurately, reflecting the latest committed configuration. The availability of updated state — by whichever mechanism — is what makes the system eventually consistent.

**Ordering across successive states is deferred.** How a client tells a newer neighborhood state from an older one — versioning, ordering, or freshness of successive states — is not fixed by this revision and is left to a future layer (consistent with the deferral in `conformance/interface-profiles/rpc-interface.md`, Core §3). This document fixes only that each state the server makes available reflects the latest committed configuration at the moment it is made available (above); it places no requirement on a client to reconcile one state against another.

No specific mechanism for making state available is prescribed: push notifications, polling, or other methods are all permitted, provided the availability, recency, and ordering requirements above are met.

## 6. The Relay Contract

The server relays a validly addressed message from a node to the out-neighbor it names (Architecture §6). The contract on that relay is **best-effort**, with one anti-abuse floor:

- **Best-effort.** Acceptance does not imply delivery, and a sender MUST NOT treat it as such. The server SHOULD attempt to deliver an accepted message to its out-neighbor, but the protocol guarantees neither delivery nor any signal of delivery or its failure. Where a binding surfaces a discard indication (Section 4), that indication reports only the server's acceptance decision — not the message's fate at the receiver — and a sender cannot infer delivery from its absence.
- **No systematic refusal.** The server MUST make a genuine attempt to relay a validly addressed message; it MUST NOT systematically or arbitrarily decline to do so. A server that accepts validly addressed traffic and discards it as a matter of course does not conform, even where best-effort otherwise permits a given message to be dropped.

Together these bound the server from both sides: it owes no per-message delivery guarantee, but it may not hollow out the relay by dropping conformant traffic wholesale.

Best-effort is a **floor, not a ceiling**. A derivative that binds GRS to a reliable, ordered transport MAY guarantee delivery and ordering above this floor without ceasing to conform, provided it preserves No-Misdelivery (Section 3) and the no-systematic-refusal rule above; what it MUST NOT do is relax them. This is the general rule for conformant strengthening (Architecture §7.1), of which delivery reliability (Section 7) is one axis.

## 7. What the Transport Fixes

Delivery _quality_ beyond the best-effort floor is deliberately left open and follows from the transport in use. The following are all implementation-defined, and a specification binding GRS to a concrete transport is expected to fix them:

- Reachability of the receiver at the instant of relay.
- Buffering of pending messages, and retention and drop policy.
- The fate of messages held for a node that departs.
- Ordering of relayed messages.
- Reliability beyond the best-effort floor (e.g., retransmission) and deduplication.
- Maximum message size.

Clients requiring end-to-end guarantees — reliable delivery, ordering, acknowledgement, or a reply path to a node with no edge toward them — construct them above the relay primitive (Architecture §3.2), not by expecting them from the server.

## References

### Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- Graph Relay System (GRS) Protocol (`architecture.md`).

### Informative References

- GRS RPC Common Core (`conformance/interface-profiles/rpc-interface.md`): maps the state-availability and relay semantics fixed here onto abstract operations, specialized by the Pull (`conformance/interface-profiles/pull/rpc-pull-profile.md`) and Pushable (`conformance/interface-profiles/push/rpc-push-profile.md`) transport profiles.
