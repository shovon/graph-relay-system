# GRS RPC Pull Session Layer

## Status of This Memo

This document specifies **one mechanism** for a **session** over a request/response, client-initiated-only transport (HTTP is the archetype): it synthesizes the thing such a transport lacks natively but a persistent connection supplies for free — a session with a definite beginning, a carried identity, a liveness notion, and a definite end. The _GRS RPC Pull Profile_ (`rpc-pull-profile.md`) requires such a session but does not mandate this mechanism; this document is one conforming realization the profile MAY be paired with, not a layer it depends on by name.

It plays, for a connectionless transport, the role a connection plays for the Pushable Profile (`../push/rpc-push-profile.md`). The relationship is the one TCP has to an application: TCP establishes, identifies, keeps alive, and tears down a connection, and the application rides on top without re-implementing any of it. This mechanism plays TCP's role where the transport itself will not.

This document is deliberately **GRS-agnostic**. It knows nothing of nodes, neighborhoods, designators, the relay, or any other concept from `../../../architecture.md`, `../../../relay-and-neighborhood-semantics.md`, or the RPC Common Core (`../rpc-interface.md`). It carries opaque requests and exposes session lifecycle; the meaning poured into those requests, and the binding of a session's life to a node's life, belong entirely to the Pull Profile above it. An implementation MAY satisfy the Pull Profile's session requirement with any other mechanism providing the same abstraction (a transport-native session, a cookie-based session, etc.); this document specifies one conforming mechanism.

## Table of Contents

1. Terminology
2. The Session Abstraction
3. Operations 3.1. `Establish` (client → server) 3.2. In-Session Request 3.3. `Close` (client → server)
4. Liveness and Timeout
5. What This Layer Does Not Provide
6. Security Considerations
7. References

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

- **Session**: A server-tracked association between a client and the server, with a definite beginning (`Establish`) and a definite end (`Close` or timeout), over which the client issues a sequence of requests.
- **Session identifier (SID)**: An unforgeable token the server issues at establishment and the client presents on every subsequent request to associate it with the session.
- **In-session request**: Any request that carries a valid SID (Section 3.2). Its body is opaque to this layer.
- **Time-to-live (TTL)**: The interval after which, with no in-session request, the server unilaterally ends the session (Section 4).
- **Upper layer**: The specification layered on this one — here, the GRS RPC Pull Profile — which gives meaning to in-session request bodies and binds session lifecycle to its own objects.

## 2. The Session Abstraction

The layer provides the upper layer with four things, and nothing more:

1. **Establishment.** A way to begin a session and obtain a SID that names it (Section 3.1). Analogue: an active open / handshake.
2. **Identified requests.** A way to issue further requests that the server reliably attributes to that one session (Section 3.2). Analogue: data segments on an established connection.
3. **Liveness.** A bound on how long a session persists without activity, refreshed by use (Section 4). Analogue: keepalive.
4. **Teardown.** A definite end, whether requested by the client or imposed on timeout (Sections 3.3, 4). Analogue: an active close / connection reset.

It exposes session lifecycle to the upper layer as two events — _session established_ and _session ended_ — and authenticates in-session requests on the upper layer's behalf. The upper layer does the rest. This layer never inspects a request body and never originates a message to the client except as the response to a request (Section 5).

## 3. Operations

### 3.1. `Establish` (client → server)

- **Input**: none required by this layer. (Authentication or admission parameters, if any, are deployment-defined; see Section 6. The upper layer MAY attach an opaque body, which this layer does not interpret.)
- **Output**: a **session identifier**. The response body MAY also carry an opaque value supplied by the upper layer.

Begins a session and starts its TTL (Section 4). It is the only operation that does not carry a SID. The server MUST issue a fresh, unforgeable SID per `Establish` and MUST NOT let a SID name a session other than the one it was issued for, nor reattach to any earlier session (Section 6).

### 3.2. In-Session Request

- **Input**: a valid SID, plus an opaque request body supplied by the upper layer.
- **Output**: a response body supplied by the upper layer.

Any request other than `Establish` is an in-session request. The server MUST validate the SID, MUST reject a request bearing an unknown, malformed, or ended session's SID, and MUST attribute an accepted request to exactly the session its SID names — never to another (Section 6). A valid in-session request refreshes the session's TTL (Section 4).

This layer treats the request and response bodies as opaque. Their structure and meaning are the upper layer's (for the Pull Profile, they are that profile's operations and their results).

### 3.3. `Close` (client → server)

- **Input**: a valid SID.
- **Output**: none required.

Ends the session: the server invalidates the SID and emits the _session ended_ event to the upper layer. After `Close`, the SID is no longer valid and any later request bearing it MUST be rejected (Section 3.2). `Close` is the graceful counterpart to timeout (Section 4); the two are equivalent in effect, differing only in promptness. Supporting `Close` is RECOMMENDED but a client that simply stops issuing requests still ends its session by timeout.

## 4. Liveness and Timeout

Because no connection closes to signal that a client is gone, the server bounds each session by a **TTL**. `Establish` starts it; every in-session request (Section 3.2) refreshes it. A client with nothing else to send MAY keep its session alive by issuing any in-session request; a deployment MAY expose an explicit, body-less keepalive request for this purpose, but none is required beyond the requests the upper layer already makes.

If the TTL elapses with no in-session request, the server MUST end the session: it invalidates the SID and emits the _session ended_ event to the upper layer. The TTL duration is deployment-defined; the server SHOULD choose it to tolerate normal request jitter while still ending dead sessions promptly. Selecting a TTL is the pull analogue of tuning keepalive intervals on a connection.

## 5. What This Layer Does Not Provide

Two non-features are deliberate and load-bearing for the layering above:

- **No server push.** This layer does not, and over a request/response transport cannot, let the server send a message to a client on its own initiative. Every server response is the answer to a client request. A session therefore confers _identity and liveness_, not a channel the server can speak down. Anything the upper layer needs to learn from the server it MUST obtain by issuing a request; the upper layer reckons with this directly (in the Pull Profile, by polling — `rpc-pull-profile.md` §5).
- **No upper-layer semantics.** This layer assigns no meaning to request bodies and models none of the upper layer's objects. Binding a session's lifecycle to a node's lifecycle, and interpreting the requests carried within it, are the Pull Profile's concern, not this layer's.

## 6. Security Considerations

The SID is a **bearer credential**: possession of a valid SID is sufficient to act within its session, since every in-session request is attributed to the session its SID names (Section 3.2). The consequences mirror any bearer-token scheme:

- **Unforgeability.** The server MUST generate SIDs with enough entropy to make guessing or forging one infeasible.
- **Confidentiality in transit.** SIDs MUST be carried only over a confidential, integrity-protected channel (e.g. TLS). A disclosed SID is a full compromise of its session until the session ends.
- **Prompt invalidation.** The server MUST invalidate a SID on `Close` and on timeout, and MUST reject any request bearing an invalidated or unrecognized SID (Section 3.2).
- **No cross-session reuse.** A SID MUST name only the session it was issued for; it MUST NOT be reusable to reattach to a prior, ended session (Section 3.1). Any continuity of identity across sessions is explicitly not provided here and must be built above.
- **Establishment abuse.** Because `Establish` is an unauthenticated-by-default request that creates server state, it is a denial-of-service surface (flooding new sessions). Admission control and rate limiting on `Establish` are deployment-defined and RECOMMENDED; the consequences a flood of sessions has on whatever the upper layer builds per session are that layer's concern.

How a client authenticates at `Establish`, if at all, is deployment-defined and outside this layer.

## 7. References

### 7.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.

### 7.2. Informative References

- GRS RPC Pull Profile (`rpc-pull-profile.md`): the upper layer this session layer is specified to serve.
- GRS RPC Common Core (`../rpc-interface.md`): defines the session-and-node-lifecycle responsibility (Core §4.4) this layer helps the Pull Profile satisfy.
