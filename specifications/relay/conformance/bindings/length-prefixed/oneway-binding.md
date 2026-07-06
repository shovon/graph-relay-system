# GRS One-Way Length-Prefixed Binding

## Status of This Memo

This document is a **binding**. It crystallizes the abstract operations of the _GRS RPC One-Way Pushable Derivative_ (`../../interface-profiles/push/rpc-push-oneway.md`) into the concrete message form fixed by the _GRS Length-Prefixed Message Shape_ (`../../../../encodings/length-prefixed-message-shape.md`). It is the meeting point of the two: it takes _what each operation does_ from the former and _what a message looks like_ from the latter, and fixes the one thing neither supplies alone — the concrete wire message for each operation.

This is precisely the latitude the layering reserves for a binding. The Common Core grants it: an operation's name and the order in which its inputs are listed are abstract, and "a binding … fixes the concrete selector for each operation and the concrete layout of its inputs" (Core §5). The shape defers to it: "the operation each [selector] denotes [is] fixed by a binding outside this shape" (Shape §3.3). The derivative leaves the form open while fixing the behavior: every operation is one-way, with no response and no correlation (OneWay §2, §4). This binding occupies exactly that gap and adds nothing to the semantics.

It is normative for implementations claiming the GRS One-Way Length-Prefixed Binding. It depends on, without restating, everything fixed by the derivative, the shape, and the representation bindings for the carried types. Section references of the form (OneWay §N) point into `../../interface-profiles/push/rpc-push-oneway.md`; (Shape §N) into `../../../../encodings/length-prefixed-message-shape.md`; (Push §N) into `../../interface-profiles/push/rpc-push-profile.md`; (Core §N) into `../../interface-profiles/rpc-interface.md`; (Designator §N) into `../../data-shapes/designator-string.md`; (Payload §N) into `../../data-shapes/payload-string.md`; (Relay §N) and (Architecture §N) into the respective companions. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119.

## Table of Contents

1. Terminology
2. What This Binding Fixes
3. The Carried Types 3.1. Designator 3.2. Payload 3.3. NeighborhoodState
4. The Messages 4.1. `Send` (client → server) 4.2. `Deliver` (server → client) 4.3. `NeighborhoodUpdate` (server → client)
5. Selectors and Directionality
6. Well-Formedness and Receiver Handling
7. No Correlation, No Response
8. Conformance
9. Security Considerations
10. References

## 1. Terminology

This binding uses all terms of the derivative (OneWay §1), the shape (Shape §1), the Pushable Profile (Push §1), and the Common Core (Core §1). In particular a **message**, a **field**, a **selector**, and a **positional argument** are as the shape defines them (Shape §1); an **operation** is as the derivative defines it (`Send`, `Deliver`, `NeighborhoodUpdate`; OneWay §3). A **carried type** is one of the three abstract values these operations carry — `Designator`, `Payload`, `NeighborhoodState` (Core §3) — and a **representation** of one is its concrete field content, fixed in Section 3.

An **octet** is an eight-bit byte. `LP(a)` denotes the length-prefixed encoding of the octet string `a` — its four-octet big-endian length then its octets — as the shape adopts it for each field (Shape §3.2). `utf8(s)` denotes the UTF-8 octets of a string `s` (RFC 3629); `LP(ε)` is the length-zero field `00 00 00 00`.

## 2. What This Binding Fixes

The derivative fixes an operation set and its behavior; the shape fixes that a message is a flat run of length-prefixed fields, of which the first is the selector and the rest are positional arguments (Shape §3.1). Three things remain, all of which a binding owns (Core §5, Shape §3.3), and this document fixes each:

1. **A selector for each operation** (Section 5). Each operation is assigned a **single-octet tag** — `Send` is `0x01`, `Deliver` is `0x02`, `NeighborhoodUpdate` is `0x03` — carried as the message's non-empty first field (Shape §3.3). The binding is free to choose the selector's octets independently of the abstract spelling (Core §5); it chooses one compact octet per operation, since the shape treats the selector as opaque octets and a message need carry no more than one to name its operation.
2. **The positional layout of each operation's inputs** (Section 4). Each abstract input becomes a positional argument field; where an operation lists more than one, this binding places them in the order the derivative lists them — a faithful choice the binding is free to make (Core §5), made for traceability.
3. **The representation of each carried type** (Section 3). `Designator` and `Payload` already have representation bindings, which this binding adopts and renders into field content; `NeighborhoodState` is fixed here as a single field holding a self-delimiting run of designator sub-fields.

It adds **nothing** to the semantics. One-way operation, the absence of any response, the absence of correlation, best-effort relay, no-misdelivery, and resolution against the current neighborhood are inherited unchanged from the derivative and its parents (OneWay §3, §4; Relay §3, §4, §6). This binding is the union of the derivative's behavior and the shape's form, plus the selector-and-position assignment that is a binding's own job — and no more.

## 3. The Carried Types

### 3.1. Designator

A `Designator` is a string (Designator §2). This binding represents it as a field whose content is the string's **UTF-8** octets (RFC 3629): `LP(utf8(designator))`. Equality is exact code-point equality of the decoded string (Designator §2); because UTF-8 is a deterministic encoding of a code-point sequence, that equality is decided identically on the field's octets, and a server resolves a `Send` designator against the sending node's current neighborhood by it (Designator §5). This binding preserves the octets verbatim end to end so that an issued designator compares equal to the same designator presented on a later `Send` (Designator §2), and ascribes the string no structure — a client MUST NOT either (Designator §3).

### 3.2. Payload

A `Payload` is a string (Payload §2). This binding represents it as a field whose content is the string's **UTF-8** octets (RFC 3629): `LP(utf8(payload))`. It is the shape's opaque argument (Shape §3.5): neither the binding nor the server inspects, parses, or transforms it; it is relayed and recovered verbatim, code point for code point (Payload §3, §4). Because the field carries the payload's octets directly with no escaping, the **empty payload** is `LP(ε)`, the length-zero field `00 00 00 00`, well-formed and not an error (Payload §2). An application carrying data that is not naturally a string encodes it into the string above this binding, which remains unaware it has done so (Payload §2).

### 3.3. NeighborhoodState

A `NeighborhoodState` is a **set** of the designators of a node's current out-neighbors (Core §3). This binding represents it as a **single field** whose content is a self-delimiting run of designator sub-fields — each an inner `LP(utf8(designator))` (Section 3.1) laid end to end — read until that field's octets are exhausted, exactly the nested-field structure the shape permits an argument to carry (Shape §3.4). It is fixed here as follows:

- The outer field's content holds one inner designator sub-field per current out-neighbor, in no fixed order.
- No designator appears twice: per-state distinctness (Designator §4) guarantees a node's current out-neighbors bear pairwise-distinct designator strings, so the run holds no duplicate.
- **Order is not significant.** A `NeighborhoodState` is a set (Core §3); a receiver MUST NOT ascribe meaning to sub-field order, nor infer anything from a change of order between two successive states. Two runs with the same designators in any order denote the same state.
- The **empty neighborhood** is `LP(ε)`, the length-zero outer field `00 00 00 00`: a field present but holding no inner sub-fields. It is a valid state and not an error (Core §3, Relay §2).

The ordering and versioning of _successive_ states remains out of scope (Core §3); under this binding a client relies on the transport's in-order delivery, so its most recently received state is its current one (Push §6, OneWay §3.3).

## 4. The Messages

Each operation is one message: a field sequence whose first field is the operation's selector (Section 5) and whose remaining fields are its inputs in the positions fixed below (Shape §3.1). Every message is one-way (OneWay §3); none carries an output, a response, or a correlation field (Section 7). Below, `LP(...)` is written left to right in the order the octets appear on the wire.

### 4.1. `Send` (client → server)

- **Direction**: client-initiated.
- **Message**: `LP(0x01)  LP(utf8(Designator))  LP(utf8(Payload))`
  - field 0 — selector `0x01`;
  - field 1 — the `Designator` (Section 3.1) naming the target out-neighbor;
  - field 2 — the `Payload` (Section 3.2), opaque.

Semantics are unchanged from OneWay §3.1: the server resolves field 1 against the sending node's current neighborhood and relays field 2 to the denoted out-neighbor, or discards it when the designator denotes none (Relay §3, §4; Designator §5). The relay is best-effort. The message surfaces no output (Section 7); a `Send` whose designator does not resolve is discarded **silently** (OneWay §5).

### 4.2. `Deliver` (server → client)

- **Direction**: server-initiated (push).
- **Message**: `LP(0x02)  LP(utf8(Payload))`
  - field 0 — selector `0x02`;
  - field 1 — the `Payload` (Section 3.2) relayed to this node by one of its in-neighbors.

This is the receiving half of the relay (OneWay §3.2, Core §4.3). The message carries **no sender designator and no reply path** — and note this is structural here, not merely unstated: the `Deliver` message has no field in which a sender could be named. A client that needs to identify an originator or answer a message constructs that within the `Payload`, above this binding (Section 7, OneWay §7). Delivery is best-effort and one-way; a client owes and the server expects no acknowledgement (Relay §6).

### 4.3. `NeighborhoodUpdate` (server → client)

- **Direction**: server-initiated (push).
- **Message**: `LP(0x03)  LP(NeighborhoodState)`
  - field 0 — selector `0x03`;
  - field 1 — the node's new `NeighborhoodState` (Section 3.3), the outer field whose octets are the run of designator sub-fields.

This is the path to neighborhood-state availability (OneWay §3.3, Core §4.2). The server pushes the updated state to the affected node on every neighborhood change, and SHOULD push the initial state on establishment (Push §3, §5.3). Because the transport delivers in order (Push §6), a client's most recently received `NeighborhoodUpdate` carries its current state; it stays current without asking (OneWay §6). The empty neighborhood is the length-zero outer field (Section 3.3).

## 5. Selectors and Directionality

The three selectors are **single-octet tags**, each the non-empty first field of its message (Shape §3.3):

| Operation            | Selector (field 0 content) | Direction       |
| -------------------- | -------------------------- | --------------- |
| `Send`               | `0x01`                     | client → server |
| `Deliver`            | `0x02`                     | server → client |
| `NeighborhoodUpdate` | `0x03`                     | server → client |

Each selector is one octet, so field 0 is `00 00 00 01` followed by the tag octet. A receiver dispatches by exact octet equality on the selector field's content (Shape §3.3); a first field whose content is any other octet string, of any length, is an unrecognized selector here and is discarded (Shape §5). This binding assigns no other selectors and reserves the remaining single-octet values for future operations of a compatible revision (Section 6 of the shape governs how a new operation is introduced).

**Each selector is valid in exactly one direction**, as the table fixes. Directionality is not advisory: a receiver MUST reject — as an unrecognized message, discarded per Shape §5 — a selector arriving from the wrong side. A **server** that receives `0x02` or `0x03` from a client, or a **client** that receives `0x01` from the server, MUST NOT act on it. This realizes the core's self-scoping at the wire (Core §6, Push §7): the connection fixes a node's role, and a client cannot invoke a server-only push, nor redirect one, merely by presenting its tag. The rejection is silent, like every other discard under this binding (Section 6, Section 7).

## 6. Well-Formedness and Receiver Handling

This binding inherits the shape's well-formedness and receiver handling wholesale (Shape §5) and specializes it to the three operations. A received message is **acceptable** under this binding when all hold:

1. it is a well-formed shape message — a field sequence that parses to exact octet consumption, holds at least one field, and whose first field is non-empty (Shape §5);
2. its selector is one of the three of Section 5, **valid in the direction it arrived** (Section 5);
3. its arguments satisfy the operation's layout (Section 4): `Send` carries a `Designator` field then a `Payload` field; `Deliver` carries a `Payload` field; `NeighborhoodUpdate` carries a `NeighborhoodState` field whose content parses as a run of length-prefixed designator sub-fields to exact exhaustion (Section 3.3).

Per the shape's append-only evolution (Shape §6), a receiver SHOULD **tolerate trailing fields** beyond those a message's operation defines — ignoring them — so that a future revision MAY extend an operation by appending. A **missing** required field, or a `NeighborhoodState` field whose inner run does not parse to exact exhaustion, is an **invalid invocation**, not a tolerated extension.

Because every operation is one-way (Section 7), a receiver **cannot reply** to report any problem. On a malformed message, an unrecognized or wrong-direction selector, or an invalid invocation, a receiver MUST NOT act on the message and SHOULD discard it; it MAY take transport-level action (for example, closing a connection that streams malformed input), which is the transport's affair, not a response of this binding (Shape §5). Nothing is surfaced to the sender in any case — silence is the only response the interface has (Section 7), and a sender MUST NOT read meaning into it (Section 9).

A receiver MUST treat every received message as untrusted input and validate it against the three conditions above **before** dispatch (Section 9).

## 7. No Correlation, No Response

Every message under this binding is a one-way emission, and the message has **no field for a response or a correlation**. This holds by the agreement of both parents, neither of which this binding may contradict:

- the derivative forbids it — there are no responses, so nothing to correlate, and "an implementation MUST NOT introduce a correlation identifier, reply field, or request/response framing at this layer" (OneWay §4);
- the shape forbids it for the same reason and a second one — correlation is the transport's, not the message's (Shape §4).

This binding accordingly defines no fourth message and adds no field to the three of Section 4: no acceptance decision, status, sequence number, message identifier, or reply field appears in any message. A sender is owed nothing back — not delivery, not acceptance, not discard (Section 6). Any end-to-end guarantee an application needs — acknowledgement, a reply, identification of a `Deliver`'s originator, correlation of its own request and response payloads — it constructs **within the `Payload`**, above this binding, and routes as ordinary `Send`/`Deliver` traffic (OneWay §7, Shape §7).

## 8. Conformance

This binding is conformant to each document it joins, and substitutable at the semantic level for any abstract implementation of the derivative.

- **To the derivative.** It changes no behavior: directionality, one-way operation, the absence of responses and correlation, best-effort relay, no-misdelivery, and silent discard are all preserved (OneWay §3, §4, §5). It defines no new operation, and in particular no response-eliciting operation (OneWay §6). It only gives the derivative's operations a concrete form.
- **To the shape.** Every message is a well-formed shape message — a field sequence with a non-empty selector and positional argument fields (Shape §3, §5). It introduces no correlation field (Shape §4), and follows append-only evolution (Shape §6).
- **To the Common Core.** It exercises exactly the latitude Core §5 grants a binding — fixing the wire selector and the argument positions, and choosing them independently of the abstract names and listing order — while preserving the §4 semantics in full.

An implementation claiming this binding implements `../../interface-profiles/push/rpc-push-oneway.md` carried in `../../../../encodings/length-prefixed-message-shape.md`, and any client or server written to the abstract derivative behaves identically when its messages take this form.

## 9. Security Considerations

This binding inherits the considerations of the derivative (OneWay §9), the shape (Shape §8), the representation bindings (Designator §7, Payload §6), the Common Core (Core §6), and Architecture §8, and weakens none. It adds only what the concrete form makes specific.

- **Directionality enforces self-scoping at the wire.** Because each selector is valid in exactly one direction (Section 5), a receiver that enforces direction prevents a peer from invoking an operation the channel does not grant it — a client cannot present `0x02`/`0x03` to act as the server, nor the server `0x01` to act as a node. A receiver MUST enforce this; accepting a wrong-direction selector would breach the self-scoping the connection is relied upon to provide (Core §6, Push §7).
- **Every received message is untrusted, and lengths precede their octets.** A receiver MUST validate well-formedness, selector, direction, and argument layout before dispatch (Section 6), and MUST treat a delivered `Payload` as untrusted application data, parsing it defensively above this binding (Payload §6). Each field's length prefix, and each designator sub-field's within a `NeighborhoodState`, is attacker-controlled and precedes the octets it counts; a receiver MUST check every declared length against the octets actually remaining before acting on it, and SHOULD bound the message and per-field sizes it accepts, the bounds being a transport/deployment concern (Shape §8, Relay §7).
- **Silence signals nothing.** Under one-way operation a message yields no reply — no acknowledgement, rejection, or error (Section 7). A sender MUST NOT infer receipt, acceptance, processing, or failure from silence, nor build a security property on such an inference (OneWay §9).
- **A `NeighborhoodUpdate` exposes the neighbor set.** Carrying the state as a visible run of designator sub-fields reveals a node's current out-neighbor count to that node, and, where designators are predictable, may reveal structure (Designator §7). This binding exposes designators exactly as the companions require and adds no exposure beyond laying the set out as a field; an implementation concerned with this mints opaque, high-entropy designators (Designator §7) — a confidentiality choice above this binding, not a correctness one.
- **No confidentiality.** Every message is carried in the clear and traverses the server (Architecture §8); this binding provides no end-to-end confidentiality. A client requiring secrecy encrypts within the `Payload` (Payload §6) or relies on a confidential transport (for example, TLS/WSS), which is the transport's concern.

## 10. References

### 10.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- RFC 3629: UTF-8, a transformation format of ISO 10646.
- GRS RPC One-Way Pushable Derivative (`../../interface-profiles/push/rpc-push-oneway.md`).
- GRS Length-Prefixed Message Shape (`../../../../encodings/length-prefixed-message-shape.md`).
- GRS RPC Pushable Profile (`../../interface-profiles/push/rpc-push-profile.md`).
- GRS RPC Common Core (`../../interface-profiles/rpc-interface.md`).
- GRS Designator String Representation (`../../data-shapes/designator-string.md`).
- GRS Payload String Representation (`../../data-shapes/payload-string.md`).
- GRS Relay and Neighborhood Semantics (`../../../relay-and-neighborhood-semantics.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).
