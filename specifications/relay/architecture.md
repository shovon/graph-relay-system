# Graph Relay System (GRS) Protocol

## Status of This Memo

This document is an informational specification for a Graph Relay System (GRS) protocol. It describes an aspirational architecture for a centralized server-based system that maintains a graph of connected clients. Distribution of this memo is unlimited.

This document does not specify a standard for the Internet community. It sketches a high-level architecture meant to inspire further development, and deliberately leaves detail to companion specifications. Implementers are encouraged to derive their own detailed specifications based on this outline, and to realize it with whatever tools they prefer.

## Abstract

The Graph Relay System (GRS) is a conceptual protocol for a client-server architecture in which a central server maintains an internal directed graph whose nodes are the connected clients. The server keeps the graph strongly connected — a directed path exists from any node to any other — while restricting each client's reach to its immediate out-neighbors. A client may send only to the neighbors the graph grants it, and the server relays only along those out-edges; everything richer is constructed by clients above this primitive. This document outlines the core principles without prescribing implementation details, aiming to foster innovation in graph-based relay systems.

## Table of Contents

1. Introduction
2. Terminology
3. Architecture Overview 3.1. Directionality and the Absence of a Reply Channel 3.2. Scope of Interaction (Non-Normative)
4. Server Responsibilities
5. Client Capabilities and Constraints
6. Communication Rules
7. Extensibility and Derivatives 7.1. Conformance of Derivatives
8. Security Considerations
9. IANA Considerations
10. Acknowledgments
11. References

## 1. Introduction

Some systems need a central authority to decide who may talk to whom, but should not have to carry what is said. The Graph Relay System (GRS) is built for that shape. A server maintains a directed graph of connected clients and keeps it strongly connected, so that the server can reach any node from any other by following the graph's edges, while each client may interact only with its immediate out-neighbors. The server's own part is deliberately small: it relays just enough for a node to reach the neighbors the graph grants it — enough, for instance, for two neighbors to negotiate a direct channel of their own — and anything richer, or any reach beyond the immediate neighborhood, is left for clients to construct above this primitive. The contribution is that coordination layer itself: a centrally maintained structure of adjacency and reachability, with the substance of what clients exchange over it deliberately left open.

This protocol is aspirational, focusing on high-level concepts to guide the creation of practical implementations. It assumes a client-server medium for core operations but allows for flexibility in derivative works. The goal is to keep the graph strongly connected, with changes propagated promptly, while direct communication is limited to graph neighbors.

This document does not mandate specific transport mechanisms, data formats, or algorithms. The finer semantics of how state and messages move — neighborhood state, designators, delivery, and consistency — are specified separately in a companion document, _GRS Relay and Neighborhood Semantics_ (`relay-and-neighborhood-semantics.md`), so that this overview can stay focused on the architecture's shape.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

- **Server**: The central entity that maintains the graph and manages client connections.
- **Client**: An entity connected to the server, represented as a node in the graph.
- **Node**: A representation of a client within the server's internal graph.
- **Out-neighbor**: A node Y is an out-neighbor of node X when a directed edge runs from X to Y. X's **neighborhood** is its set of out-neighbors.
- **In-neighbor**: A node X is an in-neighbor of node Y when X has Y as an out-neighbor. Because edges are directional, a node's in-neighbors and out-neighbors are distinct sets that need not coincide.
- **Reachability**: A structural property of the graph: from any node, the server can reach every other node by following directed edges. A graph with this property is **strongly connected**. Reachability describes server traversal; it does not grant clients the ability to reach non-neighbors.
- **Designator**: A handle by which the server lets a node denote one of its out-neighbors. Its detailed semantics — in particular how a node's out-neighbors are distinguished and how a designator is resolved — are specified in the companion _Relay and Neighborhood Semantics_ document, which deliberately leaves how designators are allocated, and what (if anything) they mean across neighborhood changes, to a future layer.
- **Graph**: A directed graph maintained by the server, where nodes correspond to clients and edges define direct communication allowances. A server MAY maintain several such graphs, disjoint from one another (Section 3).

## 3. Architecture Overview

GRS operates in a client-server model where clients establish a session to a single server. Each active session establishes a node in the server's internal directed graph. The server is responsible for constructing and maintaining this graph such that it remains strongly connected, so that the server can reach every node by following out-edges from any starting node (for example, to broadcast to or traverse the entire client set).

An example topology is a directed n-cycle, where n is the number of nodes: every node has out-degree 1 and in-degree 1, and following out-edges from any node traverses all n nodes before returning to the start. The server MAY instead use any other topology — for instance one with lower diameter — as long as the graph remains strongly connected. The choice of topology is left open; companion specifications may describe particular ones.

A server MAY maintain more than one such graph at once — for instance to serve unrelated client populations from a single deployment. When it does, the graphs MUST be **disjoint**: no edge runs between them, and every node belongs to exactly one graph — so the server's reachability traversal stays within a graph, and no node's neighborhood ever names a node of another. Strong connectivity, relay, and every rule below then apply within each graph independently, with no coordination required between graphs. (A single-graph deployment is unaffected by this clause.)

Two boundary cases follow directly from this definition. An empty graph (zero connected clients) trivially satisfies strong connectivity. A single-node graph is likewise trivially strongly connected; that node has an empty neighborhood, and the server does not create self-loops, so a lone client has no one to communicate with until a second client joins.

Strong connectivity is a steady-state property. A node joining or leaving necessarily disturbs the graph (for example, removing a node from an n-cycle breaks the cycle), so the invariant holds between updates rather than during them. When a change occurs, the server MUST restore strong connectivity and SHOULD minimize the window during which the graph is in a transitional state. Clients MAY observe transient neighborhood states while a transition is in progress.

A node's existence is, by default, bounded by its connection. When the connection closes, or the server otherwise determines the node has departed, the node ceases to exist and the graph is repaired around its absence. Under this default, a client that later reconnects thereby establishes a _new_ node, unrelated to any it previously occupied: the server treats it as a wholly new participant — inheriting no designation, position, or neighbors from before — and MAY place the node at any position in the graph. In the base architecture the server has no concept of client "sameness" across connections and is under no obligation, and has no general means, to recognize that two connections belong to the same client.

Whether a node's identity persists across this boundary is a deferred axis: the base provides the connection-scoped, ephemeral identity just described, and neither requires nor forbids more. A derived specification MAY give a node an identity that survives session termination and reinitiation — one natural realization is a client-held credential, such as a key pair whose public half serves as the node's durable identifier, presented and proven on each session so that the same identifier denotes the same node across reconnections. A derivative that does so assumes the obligations such persistence entails, above all authenticating the identity (Section 8); whatever identity a derivative does not provide remains, as in the default, a construction left to clients (Section 3.2).

The architecture MUST be implemented over a client-server medium, such as a session-oriented protocol. This ensures centralized control and simplifies graph management.

By way of analogy (this does not describe the system's topology): the out-neighbor relation is like following on a social network. If Alice follows Bob, Bob is in Alice's neighborhood, but Alice is not in Bob's unless Bob also follows her.

## 3.1. Directionality and the Absence of a Reply Channel

The neighbor relation is directional: a node may send only to its out-neighbors, and the server relays only along out-edges. One consequence is worth stating plainly, because it is where readers most often expect an exception:

**Receiving a message grants no ability to answer it.** An in-edge is not a send path. For a node to reach the node that sent it a message, that sender must independently be one of its _own_ out-neighbors — a reverse edge that may simply not exist. When it does not, a reply (if the application needs one) is constructed above the relay primitive (Section 3.2), like any other reach to a non-neighbor. This holds without exception — for replies, acknowledgements, and errors alike: a node's directly reachable set is exactly its current out-neighbors, at every instant.

## 3.2. Scope of Interaction (Non-Normative)

This protocol defines exactly one interaction primitive: relay from a node to its out-neighbors (Section 6). It deliberately defines no others — no traversal and no end-to-end delivery between non-neighboring nodes.

Strong connectivity (Section 3) is maintained as a structural guarantee, not as a feature the protocol itself exercises. Its purpose is to ensure that a directed path always exists between any pair of nodes, so that any reachability a client may wish to construct is _possible_ in principle. Whether, and how, to construct such reachability is left entirely to clients and is outside the scope of this document.

For example, clients MAY cooperatively forward an opaque payload hop-by-hop along directed edges to reach a non-neighbor — or to route a reply back to a node that has no edge toward them — treating the neighbor primitive as a transport and layering their own routing above it. To the server these are ordinary neighbor-to-neighbor messages; it neither provides nor is aware of such schemes. This is one possibility among many; the protocol neither prescribes nor privileges it.

Identity continuity is another such construction. Under the base architecture's default, the server treats every connection as a new node (Section 3), so any sense in which a reconnecting client is the "same" participant as one that left is established out-of-band — for example, by a higher-layer protocol that uses the relay as its delivery medium — and not by the server, which under that default neither tracks nor can vouch for sameness across connections. A derived specification MAY instead make identity persist at the substrate (Section 3), in which case that continuity becomes the derivative's to define and secure; absent such a derivative, it remains the application-level construction described here.

In this sense the system is a substrate. Its utility is an emergent property of how clients choose to use the neighbor primitive over a graph that is guaranteed to be traversable.

## 4. Server Responsibilities

The server acts as the authoritative maintainer of the graph. It MUST:

- Accept sessions from clients and represent each as a node.
- Construct and dynamically update the graph to keep it strongly connected.
- Handle additions, removals, and modifications of nodes (e.g., due to client disconnections or topology optimizations).
- Detect when a node has departed or become unreachable (for example, via connection close, heartbeat, or timeout) so that the graph can be repaired. The detection mechanism is implementation-defined.
- Make each affected node's updated neighborhood state available once a change is committed, and relay messages from a node to its out-neighbors — and only its out-neighbors. The precise guarantees governing state availability and message delivery are specified in the companion _Relay and Neighborhood Semantics_ document.

The server SHOULD aim for efficiency in graph maintenance but is not required to optimize for specific metrics like latency or bandwidth.

## 5. Client Capabilities and Constraints

Clients interact solely with the server and, through it, with their graph neighbors. Clients:

- MUST establish a connection to the server to join the graph.
- SHOULD be prepared to receive updates about neighborhood changes.
- SHOULD be prepared to receive messages relayed by the server.

Clients MAY:

- Request the current state of their neighborhood from the server.
- Send messages, via the server, to any node in their neighborhood — that is, to any of their current out-neighbors, and to no one else (Section 3.1).
- Receive messages, via the server, from any of their in-neighbors. Because the neighbor relation is not symmetric, the set of nodes a client can send to and the set it can receive from need not coincide, and receiving from a node does not imply being able to send to it (Section 3.1).

How a client names its out-neighbors, how long such a name remains valid, and what happens to a message addressed to a node that is no longer an out-neighbor are governed by the companion _Relay and Neighborhood Semantics_ document.

## 6. Communication Rules

Communication in GRS is graph-constrained:

- A node may send a message only to its out-neighbors; the server relays it along that directed edge and discards anything addressed elsewhere. This permission is directional and admits no reply exception (Section 3.1): that X may send to Y does not imply Y may send to X.
- Relay is **best-effort**: the server attempts to deliver a validly addressed message but does not guarantee it.

This places all routing at the edges: the server relays from a node to its out-neighbors and does no more; any reach beyond the immediate neighborhood is constructed by clients above this primitive (Section 3.2). It bounds what the _server_ does, not what clients can achieve.

The detailed delivery contract — what "best-effort" guarantees and excludes, how stale addressing is handled, and what is left to the transport (buffering, ordering, retention, message size, and the like) — is specified in the companion _Relay and Neighborhood Semantics_ document. A specification binding GRS to a concrete transport is expected to fix the remaining choices.

## 7. Extensibility and Derivatives

This specification focuses on the core client-server paradigm. However, core implementations MUST adhere to the client-server model to preserve simplicity and control.

Implementers are encouraged to develop detailed protocols, including transport bindings, message formats, and error handling, to realize GRS in practice. Such a document is a **derivative**: it builds on this architecture and its companions, fixing choices they deliberately leave open or strengthening guarantees they state as floors, without restating or replacing what they already fix. The companion specifications are themselves derivatives in this sense — `relay-and-neighborhood-semantics.md` fixes the relay and designator semantics this overview defers, and `unique-session-identity` strengthens the base's ephemeral, per-session identity (Section 3) in one direction the companions permit.

## 7.1. Conformance of Derivatives

A derivative is GRS-conformant when everything written against this architecture and its companions still holds under it. The intent is to welcome derivatives that add structure for their own microcosm, while excluding those that quietly repudiate what the base fixes. Concretely, a derivative:

- MUST preserve every guarantee and prohibition this document and its companions fix. The properties the base mandates — strong connectivity (Section 3), the directional send permission and its no-reply consequence (Sections 3.1, 6), No-Misdelivery and resolution against the current neighborhood (_Relay and Neighborhood Semantics_ §3, §4), and the relay's anti-abuse floor (_Relay and Neighborhood Semantics_ §6) — MUST continue to hold for any client that relies on them.
- MAY strengthen any property the base leaves as a non-guarantee, and MAY fix any choice the base leaves open. Best-effort delivery is a **floor, not a ceiling**: a derivative binding GRS to a reliable, ordered transport (for example TCP or WebSocket) MAY guarantee delivery and ordering above that floor (_Relay and Neighborhood Semantics_ §6, §7), just as a derivative MAY make a designator stable, globally unique, or persistent across sessions beyond the base's per-state distinctness (_Relay and Neighborhood Semantics_ §2; designator-string §6). Strengthening a non-guarantee does not contradict the base, because nothing written against the base assumed the _absence_ of the stronger property.
- MUST NOT relax, remove, or contradict a requirement stated here or in a companion, nor redefine a term these documents define (Section 2). A derivative that, for instance, permits misdelivery, lets a node send to a non-neighbor, or has the server make relay decisions from a payload's contents is not a GRS derivative but a different protocol, and MUST NOT claim conformance.
- SHOULD, where it strengthens a guarantee, document both the stronger guarantee it provides and — where an axis offers more than one direction — the directions it does _not_ take, so that what may be built atop the derivative is clear. The `unique-session-identity` binding is written this way: it strengthens identity toward per-session uniqueness — distinct for the lifetime of the node's graph, with reuse across a server restart available only as an opt-in optimization where every holder dies with the server — while explicitly declining cross-session persistence.

The test is **substitutability**: code written to the base contract MUST continue to work, unmodified, against any conformant derivative. A derivative adds and narrows; it never subtracts. A profile that strengthens the best-effort relay toward reliable, in-order delivery passes this test — a client that tolerated loss still works when loss never happens. A document that instead repudiates a base requirement — however reasonable its alternative — is outside this family and SHOULD NOT borrow its conformance language.

## 8. Security Considerations

GRS assumes a trusted server, as it holds authoritative control over the graph. Potential risks include:

- Server compromise, leading to graph manipulation or message interception.
- Client spoofing, where unauthorized entities join as nodes, or assume a designation or identity another client relies on.
- Denial-of-service attacks targeting the server or specific nodes, including connection churn intended to force continuous graph repair.
- Loss of confidentiality: because every message is relayed through the server, the server can read all client-to-client traffic. GRS does not provide end-to-end confidentiality; implementations requiring it must layer it above the relay.

Implementations SHOULD incorporate authentication, encryption, and access controls so that unauthorized joins and spoofed designations or identities can be rejected. Specific security mechanisms are left to derivative specifications.

A derivative that gives nodes a persistent identity (Section 3) sharpens the spoofing concern above: once a durable identifier denotes the same node across sessions, the server MUST authenticate any claim to that identifier, or an attacker can assume an identity other nodes rely on. The cost depends on where the identity's durability lives. A client-held, self-authenticating credential — for example a public key proven by possession of the matching private key on each session — keeps the server soft-state, verifying a fresh proof per connection and holding only current routing, at the usual key-disclosure risk. A server-remembered identity instead obliges the server to hold durable, authenticated identity state that survives restarts, enlarging both its persistence requirements and its attack surface. The base mandates neither; a derivative that opts into persistence owns the consequences of its choice.

GRS designates a single server (Section 3), which is therefore a single point of failure; the graph is held in the server's state, and persistence, replication, and recovery after a server restart are implementation-defined.

## 9. IANA Considerations

This document has no IANA actions.

## 10. Acknowledgments

This aspirational specification draws from concepts in graph theory and networked systems. Thanks to the broader community for inspiring structured communication architectures.

## 11. References

### 11.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.

### 11.2. Informative References

- GRS Relay and Neighborhood Semantics (`relay-and-neighborhood-semantics.md`): companion specification fixing neighborhood-state availability, designator semantics, and the delivery contract.
- GRS RPC Common Core (`conformance/interface-profiles/rpc-interface.md`): companion specification fixing the common abstract data, operation semantics, and server responsibilities of the relay interface, specialized by one of two transport profiles:
  - GRS RPC Pull Profile (`conformance/interface-profiles/pull/rpc-pull-profile.md`): for request/response transports (e.g. HTTP), layered on the GRS RPC Pull Session Layer (`conformance/interface-profiles/pull/rpc-pull-session.md`), which synthesizes a session over a connectionless transport.
  - GRS RPC Pushable Profile (`conformance/interface-profiles/push/rpc-push-profile.md`): for full-duplex transports (e.g. WebSocket, TCP), where the connection is itself the session.
