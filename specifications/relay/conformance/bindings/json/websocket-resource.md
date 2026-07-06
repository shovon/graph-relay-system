# GRS WebSocket Resource

## Status of This Memo

This document is a companion to the _GRS WebSocket Transport Binding_ (`websocket-transport-binding.md`). That binding fixes everything about a connection except one thing it deliberately leaves open: a connection joins exactly **one graph**, but _which_ graph, and how a server is addressed, it defers here (Transport §2, §5.1). This document fixes that one thing — and fixes it almost entirely by saying where the freedom lives, not by removing it.

Its whole content reduces to one rule and its consequences: **the connection URL identifies the graph.** The URL is at once where a client connects (a locator) and an opaque identifier for the graph it joins (a natural key) — the same duality a linked-data system gives an actor IRI, which both dereferences and names. From that one rule follow a server's freedoms (it owns the URL's shape and need not encode any meaning in it), a client's single hard constraint (it MUST treat the URL as opaque), and one security decision a deployment MUST make explicitly (what a previously-unknown URL does).

A server that hosts exactly one graph needs little of this document: it exposes one URL and is done. The document earns its length only for a server that hosts many. It is normative for implementations of the GRS WebSocket Transport Binding that expose more than one graph, and its client-side opacity rule (Section 4.2) is normative for **every** client. Section references of the form (Transport §N) point into `websocket-transport-binding.md`; (Binding §N) into `oneway-binding.md`; (Designator §N) into `../../data-shapes/designator-string.md`; (Push §N) into `../../interface-profiles/push/rpc-push-profile.md`; (Architecture §N) into `../../../architecture.md`. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119.

## Table of Contents

1. Terminology
2. What This Document Fixes
3. The URL Is King
4. Opacity: A Two-Sided Contract 4.1. The Server's Freedom 4.2. The Client's Constraint
5. One Graph or Many
6. Aliasing and Identity
7. Making the Resource Known
8. An Optional HTTP Descriptor (Non-Normative)
9. Conformance
10. Security Considerations
11. References

## 1. Terminology

This document uses all terms of the transport binding (Transport §1), the JSON-array binding (Binding §1), and the architecture (Architecture §3). A **graph** is the relay graph a node joins (Architecture §3). A **connection** is a WebSocket connection on which the transport binding has been negotiated (Transport §3).

A **client**, throughout this document, is the **GRS protocol participant** — the component that holds a resource URL, opens the connection, and exchanges operations over the graph (Transport §5.2). It is _not_ the enclosing application that may drive it. Logic above the client — in particular, any construction of a resource URL from out-of-band knowledge of a server's URL scheme — is outside this document's scope; this document constrains only what the client itself is handed and does (Section 4.2).

A **resource URL** is the URI of a connection's opening WebSocket handshake — its request target (RFC 6455 §3; a URI per RFC 3986), for example `wss://example.com/some-room` or `wss://example.com`. It is the value this document makes identify a graph. "Resource URL" and "graph's URL" are used interchangeably.

To **resolve** a resource URL, here, is the server's act of associating an arriving connection with the one graph that URL denotes — not any client-side parsing, of which this document permits none (Section 4.2).

## 2. What This Document Fixes

The transport binding establishes that opening a connection is joining, and that a connection joins exactly one graph, but states that "_which_ graph the node joins is the graph the server associates with this connection" and defers the association entirely (Transport §5.1). This document supplies the missing association and nothing else:

- It fixes that the association key is the connection's **request URI** — the resource URL (Section 3).
- It fixes the **client's** obligation toward that URL — opacity (Section 4.2).
- It fixes that a server MUST make its **resource provisioning policy** explicit (Section 10).

Everything else it leaves to the server, and says so. The division of the opening handshake with the transport binding is clean and total: **the transport binding owns the `Sec-WebSocket-Protocol` header and the framing; this document owns the request URI.** They are different columns of the same handshake and never overlap (Transport §2).

## 3. The URL Is King

The rule, in full:

> A connection's **resource URL** identifies the one graph the connection joins. The server associates each connection with exactly one graph by that URL. There is exactly **one resource-URL-to-graph association per connection**, and how many such associations a server maintains — one, or many — together with what the URLs look like, is the server's affair alone.

This single rule is the whole model. It does not branch on how many graphs a server hosts: hosting one graph and hosting many are the same rule at different counts (Section 5). It ascribes the URL no structure and reserves none: a resource URL is, to this protocol, an opaque whole that a server maps to a graph and a client only ever connects to (Section 4).

The URL thereby serves two roles at once, and both are intended:

- a **locator** — it is where on the network a client opens the connection (RFC 6455 §3); and
- an **identifier** — it is an opaque, stable handle for the graph, suitable for a client to retain as a natural key (Section 4.2) the way an application stores any external identifier.

Neither role is privileged; the URL is king because it is both at once.

## 4. Opacity: A Two-Sided Contract

The URL's structure is a contract with exactly two clauses, one freeing the server and one binding the client. Stating both is the substance of this document; the server's freedom is _safe_ precisely because the client's constraint guarantees nothing depends on the structure the server is free to choose.

### 4.1. The Server's Freedom

A server is under **no obligation to encode any meaning** in a resource URL. It MAY mint URLs with human-meaningful path segments (`/some-room`), or wholly opaque ones (a UUID, a hash, a random token), or anything between, in whatever structure it likes; it MAY use the bare authority (`wss://example.com`) for a single graph; and it owes no party — least of all a client — any account of how its URLs are formed or what, if anything, their parts mean. The shape of a resource URL is the server's private affair.

This freedom mirrors the one the server already has over designators, which it mints and a client only ever echoes back without ascribing them structure (Designator §3). A resource URL is the same kind of value, one level out: server-minted, client-opaque.

### 4.2. The Client's Constraint

A client MUST treat a resource URL as an **atomic, opaque whole**. Concretely:

- it MAY connect to the URL, and MAY retain the URL **in its entirety** as an identifier — a key in storage, a label, a thing to reconnect to;
- it MUST NOT decompose the URL, parse its components, or read its scheme, authority, path, or query for application meaning;
- it MUST NOT infer from a URL's structure the identity, size, contents, or history of the graph it denotes, nor any relationship between that graph and any other, nor whether two URLs denote the same graph or different ones (Section 6).

A client comes to hold a resource URL only out-of-band (Section 7) and uses it only by connecting to it. The URL means, to a client, exactly "the place I join this graph" — no more is readable from it, because by §4.1 no more was required to be written into it.

This is the one place this document constrains a client, and it is the contract's load-bearing clause: it is what lets a server change, restructure, or randomize its URLs without breaking any client, because no conformant client was ever entitled to depend on their form.

Because "client" here is the protocol participant and not the application that may enclose it (Section 1), this constraint reaches only the component that connects. An enclosing program MAY construct a resource URL from out-of-band knowledge — including a server's **publicly published URL scheme** (for example, a URI Template) — and hand the finished URL down to the client, which still receives it whole and treats it opaquely; every clause above is satisfied. The structural coupling such a program takes on — its dependence on the server's scheme, and its exposure should the server restructure — lives **above** the client, not in it. What this clause forbids is a _client_ decomposing or reading a URL it was handed; it does not reach up to forbid an application from building one.

## 5. One Graph or Many

The rule of Section 3 covers both deployment styles without special-casing either; they are the same association at counts of one and of many.

- **One graph (n = 1).** A server that hosts a single graph exposes a single resource URL — which MAY be the bare authority `wss://example.com`, or any one URL it chooses — and every connection, whatever its request URI, joins that one graph (or is rejected per the server's policy, Section 10). Such a server needs nothing of this document beyond choosing that URL and publishing it (Section 7).
- **Many graphs (n > 1).** A server that hosts several graphs gives each its own resource URL, and a connection's URL selects among them.

A server is under **no obligation to support more than one graph.** The single-graph deployment is not an exception to the model but its n = 1 instance: one association, one URL, one graph.

## 6. Aliasing and Identity

A server MAY associate **more than one** resource URL with the same graph — aliasing — and this document neither requires nor forbids it. It follows that a resource URL identifies, strictly, a **connection handle**, not uniquely a graph: distinct URLs MAY resolve to one graph, and a client cannot tell.

Indeed a client MUST NOT try to tell. Whether two resource URLs alias one graph or denote two is unobservable to a client by §4.2 and is none of its concern: a client interacts only with the graph reachable through the URL it was given. Correspondingly, whether two superficially different URLs are "the same" — differing only in authority case, an explicit default port, `www` versus apex, a trailing slash, or a query string — is a **canonicalization** question the server answers for itself; a server recognizes its own resources by whatever rule it adopts. This document fixes no canonicalization, and needs to fix none, because the client opacity of §4.2 makes the entire question invisible to the only party the document constrains.

## 7. Making the Resource Known

Because a resource URL is opaque (Section 4.2), a client cannot derive or guess one; it must be **told**. This document mandates only _that_ a client is told, out-of-band, and specifies no mechanism.

A server that intends clients to reach a graph MUST provide some out-of-band means by which an intended client comes to hold its resource URL — a published link, a configuration value, a directory or invitation, a URL carried inside an earlier application message, anything. The channel, its form, and its access control are the deployment's to choose. This document defines **no in-band discovery**: consistent with the binding's three operations (Binding §4), there is no operation by which a client enumerates a server's graphs or asks for a URL. The client is, in the plain sense, supposed to already know.

## 8. An Optional HTTP Descriptor (Non-Normative)

The `wss` scheme opens a socket; it does not return a document. One therefore cannot dereference a resource URL the way one dereferences a linked-data identifier — there is no `GET wss://example.com/some-room` that yields a description of the graph. The identifier and locator roles of Section 3 both hold, but the _fetch-a-description_ affordance that an `https` actor IRI enjoys is simply absent from a `wss` URL.

A deployment that wants that affordance MAY supply it separately: pair a resource URL with an HTTP(S) sibling at the same authority and path, or a well-known location (RFC 8615), that returns a descriptor document for the graph. This is wholly optional, entirely outside this document's normative scope, and changes nothing about the `wss` URL, over which the opacity of §4.2 continues to govern. It is offered only to note that the dereference half of the duality, where wanted, is an HTTP sibling's job, not this protocol's.

## 9. Conformance

A conformant **server**:

- associates each connection with exactly one graph by its request URI (Section 3);
- treats the shape of its resource URLs as its own and is free to encode no meaning in them (Section 4.1);
- provides some out-of-band means for an intended client to learn a resource URL, and offers no in-band discovery (Section 7);
- makes its resource-provisioning policy explicit (Section 10).

A conformant **client** — under every deployment, single-graph or not:

- treats each resource URL as an opaque, atomic whole, retaining it only in its entirety and parsing none of it (Section 4.2);
- obtains resource URLs only out-of-band (Section 7) and infers nothing from their structure, including nothing about aliasing (Section 6).

## 10. Security Considerations

This document inherits the considerations of the transport binding (Transport §8) and its parents and adds those specific to treating a URL as a graph's identity.

- **Resource provisioning is load-bearing — define it.** A server MUST decide, and state, what an arriving connection to a **previously-unknown** request URI does: either it (a) **creates a new graph on demand**, or (b) is **rejected** because graphs are provisioned out-of-band and an unknown URL denotes nothing (the server closing the connection, Transport §5.3). The choice is not cosmetic. Auto-creation makes graph creation an **unauthenticated client capability** and a denial-of-service and resource-exhaustion vector: an attacker varies the URI to spawn unbounded graphs, each forcing graph state and repair (Architecture §8, Transport §8). A server that auto-creates SHOULD authenticate and rate-limit creation and SHOULD bound the number of live graphs; a provisioned-only server rejects unknown URIs outright. Both postures are legitimate; leaving the choice **implicit** is the error.
- **A resource URL is a capability only if the server makes it one.** If a server mints **unguessable, high-entropy** URLs and distributes them only over confidential channels, possession of the URL functions as a bearer capability to join the graph. If it mints **guessable** URLs (a readable room name), possession confers nothing and any access control MUST be enforced separately, at or above the handshake (Transport §8, Architecture §8). A server MUST NOT rely on URL secrecy for confidentiality unless it actually mints unguessable URLs and keeps them secret in transit — a hazard exactly parallel to predictable designators revealing structure (Designator §7).
- **URLs are visible; encode nothing sensitive.** The request URI travels in the opening handshake and is exposed to network observers unless the connection is `wss`, and is routinely logged by intermediaries and retained by clients. The opacity of §4.2 binds clients, not adversaries: it is an interoperability rule, not a confidentiality boundary. A server therefore MUST NOT encode sensitive information in a resource URL, and a URL used as a capability MUST be carried over `wss` and over out-of-band channels that do not leak it.

## 11. References

### 11.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- RFC 3986: Uniform Resource Identifier (URI): Generic Syntax.
- RFC 6455: The WebSocket Protocol.
- GRS WebSocket Transport Binding (`websocket-transport-binding.md`).
- GRS One-Way Pushable JSON-Array Binding (`oneway-binding.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).

### 11.2. Informative References

- RFC 8615: Well-Known Uniform Resource Identifiers (URIs).
- GRS Designator String Representation (`../../data-shapes/designator-string.md`).
- GRS RPC Pushable Profile (`../../interface-profiles/push/rpc-push-profile.md`).
