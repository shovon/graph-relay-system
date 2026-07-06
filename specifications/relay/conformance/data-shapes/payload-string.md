# GRS Payload String Representation

## Status of This Memo

This document is a concrete **representation binding** for the `Payload` abstract type of the _GRS RPC Common Core_ (`../interface-profiles/rpc-interface.md`, Core §3). It fixes one decision the core and its companions deliberately leave open (Core §3, §5): the concrete form of the value a node sends to an out-neighbor — the **thing the relay carries**. Under this binding, that value is a **string**.

A payload is **what is carried, not the thing that carries it.** How a payload is framed on the wire, bounded in size, and packed into whatever envelope conveys it across the transport is a separate concern, deliberately out of scope here and deferred (Section 5). This binding speaks only to the carried value: that it is a string, that the server does not interpret it, and that it is relayed verbatim.

It is the mirror image of the _GRS Designator String Representation_ (`designator-string.md`). That binding makes a server-minted handle opaque to the **client**; this one makes a client-supplied payload opaque to the **server**. The two share a shape — a string, preserved by exact code points, with no structure ascribed to it — but point their non-interpretation rule in opposite directions: there, the client MUST NOT read what the server minted; here, the server MUST NOT read what the client sent.

It fixes representation only. It changes none of the relay's semantics — best-effort delivery, no-misdelivery, and the server's content-agnostic role hold exactly as the companions state them (Architecture §6, Relay §3, §6). In particular it imposes **no constraint on a payload's contents**: what a payload _means_ is the application's, constructed above this interface (Architecture §3.2), and the server neither inspects nor interprets it.

It is one binding among possible others — an implementation MAY represent a payload differently (for example, as an opaque octet string) — and is normative for implementations that adopt the string representation. Section references of the form (Core §N) point into `../interface-profiles/rpc-interface.md`, (Relay §N) into `../../relay-and-neighborhood-semantics.md`, and (Architecture §N) into `../../architecture.md`.

## Table of Contents

1. Terminology
2. Representation
3. Opacity and the Server's Non-Interpretation
4. Verbatim Relay
5. Carriage, Framing, and Size Are Out of Scope
6. Security Considerations
7. References

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

This document uses the terms **Server**, **Client**, **Node**, **out-neighbor**, **in-neighbor**, and **relay** as defined in Architecture §2, and **Payload** as defined in Core §3. A **payload string** is the concrete realization of a `Payload` this binding fixes — the carried value, not the means of carriage.

## 2. Representation

A payload is a **string**: a finite sequence of Unicode code points. This binding places **no constraint on its contents**. Any sequence of code points is a well-formed payload, including:

- the **empty string**, which is a well-formed payload and not an error; this binding ascribes no meaning to it and reserves it for nothing;
- a string whose code points spell structure of the application's choosing — JSON, a URL, a base64-encoded binary blob. That structure is private to the communicating clients and is invisible to the server (Section 3). An application that needs to send data that is not naturally a string encodes it **within** the payload string (for example, base64); the server is unaware it has done so.

This binding ascribes **no internal structure** to the string, and imposes **no length bound of its own**. Any bound on a payload's size belongs to the layer that carries it, not to the carried value, and is out of scope here (Section 5).

## 3. Opacity and the Server's Non-Interpretation

A payload string is an **opaque value** to the server. The server is a relay (Architecture §6), not a reader: it conveys a payload from a sending node to one of its out-neighbors without depending on what the payload contains. Accordingly, the server MUST NOT:

- parse, decode, or otherwise interpret the payload's code points, or attribute any meaning to them;
- make a relay decision — to deliver, to discard, to order, to buffer, or to drop — as a function of the payload's **contents**. The relay decision is a function of the sending node and the `Send` designator alone (Relay §4); the payload is never an input to it;
- transform the payload in transit — no Unicode normalization, no case folding, no whitespace trimming, no truncation, and no re-encoding that changes the code-point sequence.

The only thing the server does with a payload is **relay it verbatim** to the out-neighbor the `Send` designator denotes (Section 4), or **discard** it when that designator denotes no current out-neighbor (Relay §4) — a decision that, again, never consults the payload.

This non-interpretation is a property an implementation SHOULD enforce **structurally**, not merely by policy: where practical, the code path that resolves a `Send` against the sender's neighborhood (Relay §4) should not receive the payload as an input it could branch on, so that opacity holds by construction rather than by discipline.

A receiving client, by contrast, **does** interpret the payload — that is the point of the relay — but it does so above this interface, and treats the payload as untrusted application data (Section 6).

## 4. Verbatim Relay

When the server relays a payload (Relay §4, §6), it delivers to the receiving node the **identical code-point sequence** the sender supplied. The relay is a transparent pipe: the payload a receiver obtains compares equal, code point for code point, to the payload the sender sent.

- No normalization, case folding, trimming, truncation, or re-encoding is performed by the server on the relay path; whatever entered is what is delivered (Section 3).
- This preservation holds end to end across the server's mediation, exactly as `designator-string.md` §2 requires for designators. Whatever layer carries the payload across the transport (Section 5) MUST preserve its code-point sequence so that the sender's and receiver's views are identical.

Verbatim relay is what lets two clients pour their own structure into the payload (Architecture §3.2) and rely on it arriving unmolested, while the server remains oblivious to that structure (Section 3). Preservation is a guarantee about _fidelity_, not _arrival_: the relay remains best-effort, and a payload MAY be dropped entirely (Relay §6), but a payload that is delivered is delivered unchanged.

## 5. Carriage, Framing, and Size Are Out of Scope

This binding fixes the carried value — a string of Unicode code points (Section 2) — and **nothing about how that value is carried**. The following all belong to the layer that conveys a payload across the transport (the envelope, and the transport binding beneath it), and are deliberately left open here, consistent with Relay §7 and Core §5:

- how a payload is framed and encoded on the wire (for example, as a JSON string under a JSON encoding, or as a length-delimited frame), and how its code-point sequence is recovered on the far side;
- any **maximum payload size**, and the rejection or handling of an over-size payload (Relay §7, Core §3);
- what constitutes a well-formed carrying frame, and the treatment of one that is malformed.

These are concerns of the **envelope**, not of the payload. The single obligation this binding places on whatever carries a payload is the preservation rule of Section 4: the carried code-point sequence MUST arrive identical to the one that was sent. Everything else about carriage is for that layer to fix, and this document takes no position on it.

## 6. Security Considerations

This binding inherits the considerations of Core §6 and Architecture §8 and adds only what the string representation of the **carried value** makes concrete. Considerations arising from how a payload is _carried_ — resource exhaustion from over-large payloads, malformed framing, and the like — belong to the carrying layer (Section 5) and are not treated here.

- **Opacity is non-interpretation, not confidentiality.** The server does not _act on_ a payload's contents (Section 3), but it is structurally _able to read_ them: every payload traverses the server (Architecture §8), which sees the cleartext string. This binding provides **no end-to-end confidentiality**. A client requiring secrecy from the server, or from anyone able to observe it, MUST encrypt within the payload (Architecture §3.2); the server then relays the ciphertext verbatim (Section 4), as oblivious to it as to any other string.
- **Payloads are untrusted application data at the receiver.** Because the server never interprets a payload, it makes **no guarantee about what a delivered payload contains**: it may be malformed for the application's purposes, hostile, or crafted to exploit a parser. A receiving client MUST treat every delivered payload as untrusted input and validate or parse it defensively, above this interface. Receipt also confers no sender identity and no reply path (Core §4.3, Architecture §3.1); any origin claim _inside_ the payload is likewise unauthenticated application data unless the application secures it.

## 7. References

### 7.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- GRS RPC Common Core (`../interface-profiles/rpc-interface.md`).
- GRS Relay and Neighborhood Semantics (`../../relay-and-neighborhood-semantics.md`).
- Graph Relay System (GRS) Protocol (`../../architecture.md`).

### 7.2. Informative References

- GRS Designator String Representation (`designator-string.md`): the companion representation binding this one mirrors, fixing the `Designator` type as a string and making it opaque to the client as this binding makes the `Payload` opaque to the server.
- GRS RPC Pull Profile (`../interface-profiles/pull/rpc-pull-profile.md`) and GRS RPC Pushable Profile (`../interface-profiles/push/rpc-push-profile.md`): the profiles whose `Send`, `Deliver`, and `Receive` operations carry payloads in this representation.
