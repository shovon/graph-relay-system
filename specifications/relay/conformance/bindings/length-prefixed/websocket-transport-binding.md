# GRS Length-Prefixed WebSocket Transport Binding

## Status of This Memo

This document is a **transport binding**. It carries the _GRS One-Way Length-Prefixed Binding_ (`oneway-binding.md`) over **WebSocket** (RFC 6455), and fixes the one layer the stack above it has, until now, only named: the transport beneath the message shape (Shape §2, premise 1). Everything above — what each operation does, and what each message looks like — is already fixed by the binding and its parents; this document fixes only how those messages cross a WebSocket connection, and under what name a peer agrees to speak them.

It fixes three things and no more:

1. a **WebSocket subprotocol name** that, when negotiated, places this entire stack in effect (Section 3);
2. the **framing** the shape delegates — one message per WebSocket binary message (Section 4);
3. the **connection lifecycle** — that opening a connection is joining, that the connection fixes a node's role and identity, and that closing it is departure (Section 5) — realizing, at the wire, what the Pushable Profile and its choreography already mandate.

It deliberately does **not** fix _which graph_ a connection joins, _how a server is addressed_, or _what a connection URL looks like_. A connection under this binding joins exactly **one graph**; how many graphs a server hosts and how each is named is the concern of a companion resource document, deferred and out of scope here (Section 2). This binding is complete and correct for a server that hosts a single graph, which needs that companion not at all.

It is normative for implementations claiming the GRS Length-Prefixed WebSocket Transport Binding. Section references of the form (Binding §N) point into `oneway-binding.md`; (Shape §N) into `../../../../encodings/length-prefixed-message-shape.md`; (Push §N) into `../../interface-profiles/push/rpc-push-profile.md`; (Choreography §N) into `../push-based-choreography.md`; (Core §N) into `../../interface-profiles/rpc-interface.md`; (Relay §N) into `../../../relay-and-neighborhood-semantics.md`; (Architecture §N) into `../../../architecture.md`. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119.

## Table of Contents

1. Terminology
2. What This Binding Fixes, and What It Defers
3. Subprotocol Negotiation 3.1. The Subprotocol Name 3.2. Offer and Selection 3.3. Versioning by a New Name
4. Message Framing 4.1. One Message per WebSocket Binary Message 4.2. Text Messages and Control Frames
5. The Connection 5.1. Opening a Connection Is Joining 5.2. The Connection Fixes a Node's Role 5.3. Liveness and Departure
6. Ordering
7. Conformance
8. Security Considerations
9. References

## 1. Terminology

This binding uses all terms of the length-prefixed binding it carries (Binding §1), the shape (Shape §1), the Pushable Profile (Push §1), and the Common Core (Core §1). In particular a **message**, a **field**, a **selector**, and a **positional argument** are as the shape defines them (Shape §1); the operations `Send`, `Deliver`, and `NeighborhoodUpdate`, and the carried types `Designator`, `Payload`, and `NeighborhoodState`, are as the binding fixes them (Binding §3, §4).

This document additionally uses the vocabulary of RFC 6455. A **WebSocket connection** is the bidirectional channel established by a successful opening handshake. A **frame** is a single unit of the WebSocket framing protocol (RFC 6455 §5.2); a **message** in the WebSocket sense is one or more frames the protocol reassembles into a whole (RFC 6455 §5.4). Where the two senses of "message" could collide, this document writes **shape message** for the length-prefixed field sequence (Shape §1) and **WebSocket message** for the reassembled WebSocket unit. The **subprotocol name** is the token negotiated in the `Sec-WebSocket-Protocol` header (RFC 6455 §1.9, §11.3.4).

A **graph** is the relay graph a node joins (Architecture §3); under this binding a connection joins exactly one of them (Section 2).

## 2. What This Binding Fixes, and What It Defers

The shape rests on the premise of "a reliable, message-oriented transport" beneath it that "delimits one message from the next" (Shape §2, premise 1; §7). WebSocket is such a transport: it is reliable and ordered (it runs over TCP), it is message-oriented (it frames), and it is full-duplex and session-oriented in exactly the sense the Pushable Profile requires (Push §2). This binding fixes how the length-prefixed binding rides it, and is the union of that binding's form with WebSocket's carriage — nothing of the semantics above is touched.

What this binding fixes is enumerated in the Status of This Memo and detailed in Sections 3–6. What it **defers** is as deliberate, and is of two kinds:

- **Upward, to the binding and its parents.** The operation set, the selectors, the argument layout, directionality, one-way operation, the absence of correlation and response, best-effort relay — all are inherited unchanged (Binding §2). This document restates none of them; it cites them.
- **Sideways, to a resource document.** A connection under this binding joins exactly one graph, but _which_ graph — how the server is addressed, what a connection URL is, whether one server hosts one graph or many, and how a client comes to know the address — is the concern of a companion resource document, not this one. This binding owns the **`Sec-WebSocket-Protocol` header and the framing**; the request URI is a separate column of the same opening handshake. A single-graph deployment reads this document alone.

## 3. Subprotocol Negotiation

### 3.1. The Subprotocol Name

This binding is identified by the WebSocket subprotocol name:

```
grs.rpc-oneway.v1.fields+lp
```

The name is a single HTTP token (RFC 7230 §3.2.6, as RFC 6455 §4.1 requires of a subprotocol name) and is matched by **exact, case-sensitive octet equality** of the whole string. Its internal structure — the dotted segments and the `+lp` suffix — is a human-readable mnemonic only: it records the project (`grs`), the carried binding (`rpc-oneway`), the version (`v1`), and the message form (a positional field sequence, `fields`, length-prefixed, `lp`). A receiver MUST NOT parse the name for meaning, derive behavior from any segment, or accept a name that differs in any octet. The whole token is the agreement; its parts are documentation.

Negotiating this name places the entire stack in effect: a peer that selects it agrees to exchange GRS One-Way Length-Prefixed messages (Binding §4) as shape messages (Shape §3), framed per Section 4, over a connection governed by Section 5.

### 3.2. Offer and Selection

A client that wishes to speak this binding MUST include `grs.rpc-oneway.v1.fields+lp` among the tokens of the `Sec-WebSocket-Protocol` header in its opening handshake (RFC 6455 §4.1). It MAY offer other subprotocol names alongside, in its own order of preference.

A server that speaks this binding and selects it MUST echo exactly `grs.rpc-oneway.v1.fields+lp` in the `Sec-WebSocket-Protocol` header of its handshake response (RFC 6455 §4.2.2, §11.3.4). A server MUST NOT select a name the client did not offer.

If the server's response does not select this name, **this binding is not in effect** on the connection, and neither peer may assume the other will honor it. A client that requires this binding MUST treat its absence from the response as a failed negotiation and SHOULD close the connection rather than transact; a server that does not implement this binding MUST NOT select its name.

### 3.3. Versioning by a New Name

The name carries `v1`, and that is the binding's hedge against its own regret. Should a future revision change the message form, the selectors, or anything else fixed above this transport — the kind of change the shape anticipates but does not itself version (Shape §6) — that revision is introduced as a **new, distinct subprotocol name** (for example, one bearing `v2`), never by reinterpreting this one. The two names then negotiate independently: a deployment MAY offer and select both during a migration window, and a peer that speaks only `v1` continues to interoperate with any other `v1` peer unchanged, indefinitely.

This is the negotiation seam working as intended (RFC 6455 §1.9): the wire form is bound to a name, not to WebSocket's identity, so the form can be replaced without disturbing the transport and without a flag day. A receiver MUST NOT attempt to "upgrade" or reinterpret a `v1` connection as anything else; the name it agreed to is the name it speaks until the connection closes.

## 4. Message Framing

### 4.1. One Message per WebSocket Binary Message

The shape requires its transport to delimit one shape message from the next and defines no framing of its own (Shape §2 premise 1, §7). WebSocket supplies that delimiting: each WebSocket message is a self-delimiting unit (RFC 6455 §5.4). This binding fixes the mapping as **one shape message per WebSocket message** — exactly one length-prefixed field sequence (Shape §3.1) per reassembled WebSocket message.

That WebSocket message MUST be a **binary** message (RFC 6455 §5.6): a shape message is an octet string (Shape §3.1, §3.2), and the WebSocket binary type is precisely an opaque application octet payload. A sender MUST place each shape message in its own binary message and MUST NOT pack two or more shape messages into one, nor split one shape message across two. Whether that binary message is transmitted whole or fragmented across several WebSocket frames (RFC 6455 §5.4) is the transport's affair and below this binding: the receiver reassembles the frames into one binary message and parses that as one shape message.

A received binary message is handled exactly as the binding directs (Binding §6): parsed as a shape message, validated for well-formedness, selector, direction, and argument layout, and dispatched or silently discarded. Nothing in the framing alters that handling.

### 4.2. Text Messages and Control Frames

This binding carries shape messages **only** in binary messages. A **text** WebSocket message (RFC 6455 §5.6) carries no shape message under this binding; a receiver MUST NOT interpret one as an operation. It SHOULD discard a text message, and MAY close the connection on receiving one (RFC 6455 §7.4.1, e.g. status 1003 "unacceptable data"), as transport/deployment policy — never surfacing anything to the sender, consistent with the silence of every other discard (Binding §6, §7).

WebSocket **control frames** — Ping, Pong, and Close (RFC 6455 §5.5) — are the transport's own and are never shape messages. They are processed by the WebSocket layer (Section 5.3 uses Ping/Pong for liveness and Close for departure) and MUST NOT be dispatched as operations. An application payload (Binding §3.2) is never carried in a control frame; it is always a field within a binary-framed shape message.

## 5. The Connection

### 5.1. Opening a Connection Is Joining

A WebSocket connection on which this binding has been negotiated (Section 3.2) **is** the node. Completing the opening handshake establishes the caller's node; the server places it in the graph and repairs the graph to keep it strongly connected (Push §3, Choreography §3, Architecture §3). There is no `Join` operation, no session handle, and no establishment message: the connection itself is the node's identity for its lifetime (Push §3).

_Which_ graph the node joins is the graph the server associates with this connection; this binding requires only that it be exactly one, and defers the addressing entirely to a companion resource document (Section 2). Within that one graph, the server SHOULD push the node's initial `NeighborhoodState` as the first `NeighborhoodUpdate` after establishment, so the client begins with a current view (Push §3, §5.3; Choreography §3; Binding §4.3). That initial state MAY be the empty neighborhood — the length-zero state field — which is well-formed and not an error (Binding §3.3, Relay §2).

By default the server attaches no identity across connections: a client that reconnects establishes a wholly new node (Push §3, Architecture §3.2). Any continuity is constructed above this binding.

### 5.2. The Connection Fixes a Node's Role

The binding's directionality (Binding §5) is realized over the connection: the connection has two ends, and each end is one role for the connection's lifetime. The **client end** may send only `Send` and may receive only `Deliver` and `NeighborhoodUpdate`; the **server end** the converse. A receiver MUST reject — silently discard, as an unrecognized message (Binding §5, §6) — any shape message bearing a selector valid only in the other direction. The connection thus fixes the node's role at establishment and a peer cannot assume the other's role merely by presenting its selector tag; this is the core's self-scoping realized at the wire (Core §6, Push §7, Binding §5).

### 5.3. Liveness and Departure

Departure is signalled by the connection, exactly as the profile requires (Push §4). A receiver MUST treat a **clean WebSocket close** (RFC 6455 §5.5.1, §7) — initiated by either end — as the node's departure: the node ceases to exist and the graph is repaired around its absence (Push §4, Architecture §4). No operation is needed to leave; closing the connection suffices.

To detect a connection that has failed without a clean close (a half-open connection), a server SHOULD use the WebSocket **Ping/Pong** mechanism (RFC 6455 §5.5.2, §5.5.3) as its liveness check — the transport-native instance of the mechanism the profile recommends (Push §4) — and MUST treat a node whose connection fails that check as departed, repairing the graph as on a clean close. The interval and timeout are deployment-defined.

## 6. Ordering

WebSocket delivers the messages of a connection **in order** (RFC 6455 §5.1) — the property the profile assumes of its transport (Push §6). Under it, a client's most recently received `NeighborhoodUpdate` always carries its current `NeighborhoodState`, and a client treats each as superseding the last without any version field (Push §6, Binding §3.3). This binding therefore introduces no neighborhood-state versioning and no per-message sequence number; it relies on WebSocket's ordering exactly as the profile relies on the transport's. Ordering of relayed `Deliver` payloads beyond what the connection provides remains out of scope (Push §6, Relay §7).

## 7. Conformance

An implementation claiming this binding:

- negotiates the subprotocol name `grs.rpc-oneway.v1.fields+lp` per Section 3, selecting it by exact-octet match and treating its absence as a failed negotiation;
- carries each shape message in exactly one WebSocket **binary** message (Section 4.1), and interprets neither text messages nor control frames as operations (Section 4.2);
- treats the connection as the node — opening as joining, the connection as identity and role, clean close or failed liveness check as departure (Section 5);
- preserves, unchanged, every semantic of the length-prefixed binding it carries (Binding §8) and of that binding's parents.

Such an implementation realizes `oneway-binding.md` over WebSocket, and is substitutable at the semantic level for any other transport binding of the same length-prefixed binding: a client and server written to the binding behave identically when its messages cross a WebSocket connection under this document.

## 8. Security Considerations

This binding inherits the considerations of the length-prefixed binding (Binding §9), the shape (Shape §8), the Pushable Profile (Push §7), the Common Core (Core §6), and Architecture §8, and weakens none. It adds what WebSocket carriage makes specific.

- **Run over TLS.** A connection SHOULD use the secure WebSocket scheme (`wss://`), so that the channel is confidential and integrity-protected. This binding provides no end-to-end confidentiality of its own — every payload is cleartext octets traversing the server (Architecture §8, Binding §9) — and `wss://` is the transport-level protection the stack assumes where secrecy or tamper-resistance is required (Push §7, Shape §8).
- **The subprotocol name is not a credential.** Selecting `grs.rpc-oneway.v1.fields+lp` authenticates nothing and grants no authority; it only agrees a wire form. Node authority rests entirely on control of the connection (Push §7), and authentication, where required, is performed at or above the opening handshake (Architecture §8) — never inferred from subprotocol negotiation.
- **Check the Origin where browsers are clients.** WebSocket has no same-origin policy of its own; a browser will open a connection to any reachable server, carrying an `Origin` header (RFC 6455 §10.2). A server that grants any authority on the basis of a browser session MUST validate `Origin` (and apply whatever cross-site request defenses it would apply to an HTTP endpoint), or it is exposed to cross-site WebSocket hijacking. The concrete policy is deployment-defined.
- **Every binary message is untrusted input.** A receiver MUST validate each shape message — well-formedness, selector, direction, and argument layout — before dispatch (Binding §6, §9), and MUST treat a delivered `Payload` as untrusted application data (Binding §3.2). Each field's declared length precedes the octets it counts and is attacker-controlled (Shape §8); a receiver MUST NOT allocate against a length before checking it against the octets remaining, and SHOULD bound the maximum WebSocket message size it will accept (RFC 6455 permits this) and any per-field length, the bounds being deployment concerns.
- **Connection churn.** Because opening a connection is joining and closing it is departure (Section 5), rapid connect/disconnect cycling forces continuous graph repair (Push §7, Architecture §8). Rate limiting on connection establishment is RECOMMENDED and deployment-defined.
- **Silence still signals nothing.** Under one-way operation no message yields a reply (Binding §7). Neither the WebSocket layer's acknowledgements nor its liveness Pongs are application-level acknowledgements; a sender MUST NOT infer receipt, acceptance, or processing of a shape message from any transport-level signal (Binding §9).

## 9. References

### 9.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- RFC 6455: The WebSocket Protocol.
- RFC 7230: Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing.
- GRS One-Way Length-Prefixed Binding (`oneway-binding.md`).
- GRS Length-Prefixed Message Shape (`../../../../encodings/length-prefixed-message-shape.md`).
- GRS RPC Pushable Profile (`../../interface-profiles/push/rpc-push-profile.md`).
- GRS RPC Common Core (`../../interface-profiles/rpc-interface.md`).
- GRS Relay and Neighborhood Semantics (`../../../relay-and-neighborhood-semantics.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).

### 9.2. Informative References

- GRS Pushable Choreography (`../push-based-choreography.md`).
