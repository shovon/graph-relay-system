# GRS One-Way Pushable JSON-Array Binding

## Status of This Memo

This document is a **binding**. It crystallizes the abstract operations of the _GRS RPC One-Way Pushable Derivative_ (`../../interface-profiles/push/rpc-push-oneway.md`) into the concrete message form fixed by the _GRS JSON-Array Message Shape_ (`../../../../encodings/json-array-message-shape.md`). It is the meeting point of the two: it takes _what each operation does_ from the former and _what a message looks like_ from the latter, and fixes the one thing neither supplies alone — the concrete wire message for each operation.

This is precisely the latitude the layering reserves for a binding. The Common Core grants it: an operation's name and the order in which its inputs are listed are abstract, and "a binding … fixes the concrete selector for each operation and the concrete layout of its inputs" (Core §5). The shape defers to it: "the operation each [selector] denotes [is] fixed by a binding outside this shape" (Shape §3.2). The derivative leaves the form open while fixing the behavior: every operation is one-way, with no response and no correlation (OneWay §2, §4). This binding occupies exactly that gap and adds nothing to the semantics.

It is normative for implementations claiming the GRS One-Way Pushable JSON-Array Binding. It depends on, without restating, everything fixed by the derivative, the shape, and the representation bindings for the carried types. Section references of the form (OneWay §N) point into `../../interface-profiles/push/rpc-push-oneway.md`; (Shape §N) into `../../../../encodings/json-array-message-shape.md`; (Push §N) into `../../interface-profiles/push/rpc-push-profile.md`; (Core §N) into `../../interface-profiles/rpc-interface.md`; (Designator §N) into `../../data-shapes/designator-string.md`; (Payload §N) into `../../data-shapes/payload-string.md`; (Relay §N) and (Architecture §N) into the respective companions. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119.

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

This binding uses all terms of the derivative (OneWay §1), the shape (Shape §1), and the Common Core (Core §1). A **message**, a **selector**, and a **positional argument** are as the shape defines them (Shape §1); an **operation** is as the derivative defines it (`Send`, `Deliver`, `NeighborhoodUpdate`; OneWay §3). A **carried type** is one of the three abstract values these operations carry — `Designator`, `Payload`, `NeighborhoodState` (Core §3) — and a **representation** of one is its concrete JSON form, fixed in Section 3.

## 2. What This Binding Fixes

The derivative fixes an operation set and its behavior; the shape fixes that a message is a JSON array `[selector, …arguments]`. Three things remain, all of which a binding owns (Core §5, Shape §3.2), and this document fixes each:

1. **A selector for each operation** (Section 5). Each operation's selector is its abstract name **verbatim**, as a string in the shape's string selector space (Shape §3.2): `"Send"`, `"Deliver"`, `"NeighborhoodUpdate"`. The binding is free to choose the token independently of the abstract spelling (Core §5); it chooses to mirror it, so that a message's selector names its operation on sight.
2. **The positional layout of each operation's inputs** (Section 4). Each abstract input becomes a positional argument; where an operation lists more than one, this binding places them in the order the derivative lists them — again a faithful choice the binding is free to make (Core §5), made for traceability.
3. **The representation of each carried type** (Section 3). `Designator` and `Payload` already have representation bindings, which this binding adopts; `NeighborhoodState` is fixed here as a JSON array of designators, the form `../../data-shapes/designator-string.md` §6 anticipates.

It adds **nothing** to the semantics. One-way operation, the absence of any response, the absence of correlation, best-effort relay, no-misdelivery, and resolution against the current neighborhood are inherited unchanged from the derivative and its parents (OneWay §3, §4; Relay §3, §4, §6). This binding is the union of the derivative's behavior and the shape's form, plus the selector-and-position assignment that is a binding's own job — and no more.

## 3. The Carried Types

### 3.1. Designator

A `Designator` is represented as a **JSON string** (RFC 8259), per `../../data-shapes/designator-string.md` and as that binding's §6 directs for a JSON encoding. Equality is exact code-point equality of the decoded string (Designator §2); a server resolves a `Send` designator against the sending node's current neighborhood by that equality (Designator §5). This binding ascribes the string no structure and a client MUST NOT either (Designator §3).

### 3.2. Payload

A `Payload` is represented as a **JSON string** (RFC 8259), per `../../data-shapes/payload-string.md`. It is the shape's opaque argument (Shape §3.4): neither the binding nor the server inspects, parses, or transforms it; it is relayed and recovered verbatim, code point for code point (Payload §3, §4). An application carrying data that is not naturally a string encodes it into the string (for example, base64) above this binding, which remains unaware it has done so (Payload §2).

### 3.3. NeighborhoodState

A `NeighborhoodState` is represented as a **JSON array** whose elements are the designator strings (Section 3.1) of the node's current out-neighbors — the form `../../data-shapes/designator-string.md` §6 anticipates ("a `NeighborhoodState` exposes each out-neighbor's designator as such"). It is fixed here as follows:

- Each element is a `Designator` (Section 3.1); the array carries one element per current out-neighbor.
- No element appears twice: per-state distinctness (Designator §4) guarantees a node's current out-neighbors bear pairwise-distinct designator strings, so the array holds no duplicate strings.
- **Order is not significant.** A `NeighborhoodState` is a set (Core §3); a receiver MUST NOT ascribe meaning to element order, nor infer anything from a change of order between two successive states. Two arrays with the same elements in any order denote the same state.
- The **empty array** `[]` is the well-formed representation of the empty neighborhood, which is a valid state and not an error (Core §3, Relay §2).

The ordering and versioning of _successive_ states remains out of scope (Core §3); under this binding a client relies on the transport's in-order delivery, so its most recently received state is its current one (Push §6, OneWay §3.3).

## 4. The Messages

Each operation is one message: a JSON array whose element 0 is the operation's selector (Section 5) and whose remaining elements are its inputs in the positions fixed below. Every message is one-way (OneWay §3); none carries an output, a response, or a correlation field (Section 7).

### 4.1. `Send` (client → server)

- **Direction**: client-initiated.
- **Message**: `[ "Send", <Designator>, <Payload> ]`
  - element 0 — selector `"Send"`;
  - element 1 — the `Designator` (Section 3.1) naming the target out-neighbor;
  - element 2 — the `Payload` (Section 3.2), opaque.
- **Example**: `[ "Send", "n7gKQ", "hello world" ]`

Semantics are unchanged from OneWay §3.1: the server resolves element 1 against the sending node's current neighborhood and relays element 2 to the denoted out-neighbor, or discards it when the designator denotes none (Relay §3, §4; Designator §5). The relay is best-effort. The message surfaces no output (Section 7); a `Send` whose designator does not resolve is discarded **silently** (OneWay §5).

### 4.2. `Deliver` (server → client)

- **Direction**: server-initiated (push).
- **Message**: `[ "Deliver", <Payload> ]`
  - element 0 — selector `"Deliver"`;
  - element 1 — the `Payload` (Section 3.2) relayed to this node by one of its in-neighbors.
- **Example**: `[ "Deliver", "hello world" ]`

This is the receiving half of the relay (OneWay §3.2, Core §4.3). The message carries **no sender designator and no reply path** — and note this is structural here, not merely unstated: the `Deliver` array has no element in which a sender could be named. A client that needs to identify an originator or answer a message constructs that within the `Payload`, above this binding (Section 7, OneWay §7). Delivery is best-effort and one-way; a client owes and the server expects no acknowledgement (Relay §6).

### 4.3. `NeighborhoodUpdate` (server → client)

- **Direction**: server-initiated (push).
- **Message**: `[ "NeighborhoodUpdate", <NeighborhoodState> ]`
  - element 0 — selector `"NeighborhoodUpdate"`;
  - element 1 — the node's new `NeighborhoodState` (Section 3.3).
- **Examples**: `[ "NeighborhoodUpdate", [ "n7gKQ", "p2Lx9" ] ]`; empty neighborhood: `[ "NeighborhoodUpdate", [] ]`

This is the path to neighborhood-state availability (OneWay §3.3, Core §4.2). The server pushes the updated state to the affected node on every neighborhood change, and SHOULD push the initial state on establishment (Push §3, §5.3). Because the transport delivers in order (Push §6), a client's most recently received `NeighborhoodUpdate` carries its current state; it stays current without asking (OneWay §6).

## 5. Selectors and Directionality

The three selectors occupy the shape's **string selector space** (Shape §3.2) and are bare — this binding applies no namespace. They are the abstract operation names verbatim:

| Operation            | Selector               | Direction       |
| -------------------- | ---------------------- | --------------- |
| `Send`               | `"Send"`               | client → server |
| `Deliver`            | `"Deliver"`            | server → client |
| `NeighborhoodUpdate` | `"NeighborhoodUpdate"` | server → client |

This binding uses **no integer selector space**: an integer at element 0 is an unrecognized selector here and is discarded (Shape §5). Because only the string space is in use, the cross-space type-confusion hazard the shape warns of (Shape §8) does not arise within this binding.

**Each selector is valid in exactly one direction**, as the table fixes. Directionality is not advisory: a receiver MUST reject — as an unrecognized message, discarded per Shape §5 — a selector arriving from the wrong side. A **server** that receives `"Deliver"` or `"NeighborhoodUpdate"` from a client, or a **client** that receives `"Send"` from the server, MUST NOT act on it. This realizes the core's self-scoping at the wire (Core §6, Push §7): the connection fixes a node's role, and a client cannot invoke a server-only push, nor redirect one, merely by naming its selector. The rejection is silent, like every other discard under this binding (Section 6, Section 7).

## 6. Well-Formedness and Receiver Handling

This binding inherits the shape's well-formedness and receiver handling wholesale (Shape §5) and specializes it to the three operations. A received message is **acceptable** under this binding when all hold:

1. it is a well-formed shape message — a JSON array of at least one element whose element 0 is a non-empty string or non-negative integer (Shape §5);
2. its selector is one of the three of Section 5, **valid in the direction it arrived** (Section 5);
3. its arguments satisfy the operation's layout (Section 4): `Send` carries a `Designator` (a JSON string) then a `Payload` (a JSON string); `Deliver` carries a `Payload` (a JSON string); `NeighborhoodUpdate` carries a `NeighborhoodState` (a JSON array of strings, Section 3.3).

Per the shape's append-only evolution (Shape §6), a receiver SHOULD **tolerate trailing arguments** beyond those a message's operation defines — ignoring them — so that a future revision MAY extend an operation by appending. A **missing** required argument, or one of the wrong JSON type (a non-string `Designator` or `Payload`, a `NeighborhoodState` that is not an array of strings), is an **invalid invocation**, not a tolerated extension.

Because every operation is one-way (Section 7), a receiver **cannot reply** to report any problem. On a malformed message, an unrecognized or wrong-direction selector, or an invalid invocation, a receiver MUST NOT act on the message and SHOULD discard it; it MAY take transport-level action (for example, closing a connection that streams malformed input), which is the transport's affair, not a response of this binding (Shape §5). Nothing is surfaced to the sender in any case — silence is the only response the interface has (Section 7), and a sender MUST NOT read meaning into it (Section 9).

A receiver MUST treat every received array as untrusted input and validate it against the three conditions above **before** dispatch (Section 9).

## 7. No Correlation, No Response

Every message under this binding is a one-way emission, and the array has **no element for a response or a correlation**. This holds by the agreement of both parents, neither of which this binding may contradict:

- the derivative forbids it — there are no responses, so nothing to correlate, and "an implementation MUST NOT introduce a correlation identifier, reply field, or request/response framing at this layer" (OneWay §4);
- the shape forbids it for the same reason and a second one — correlation is the transport's, not the message's (Shape §4).

This binding accordingly defines no fourth message and adds no slot to the three of Section 4: no acceptance decision, status, sequence number, message identifier, or reply field appears in any array. A sender is owed nothing back — not delivery, not acceptance, not discard (Section 6). Any end-to-end guarantee an application needs — acknowledgement, a reply, identification of a `Deliver`'s originator, correlation of its own request and response payloads — it constructs **within the `Payload`**, above this binding, and routes as ordinary `Send`/`Deliver` traffic (OneWay §7, Shape §7).

## 8. Conformance

This binding is conformant to each document it joins, and substitutable at the semantic level for any abstract implementation of the derivative.

- **To the derivative.** It changes no behavior: directionality, one-way operation, the absence of responses and correlation, best-effort relay, no-misdelivery, and silent discard are all preserved (OneWay §3, §4, §5). It defines no new operation, and in particular no response-eliciting operation (OneWay §6). It only gives the derivative's operations a concrete form.
- **To the shape.** Every message is a well-formed shape message — a JSON array with a valid selector and positional arguments (Shape §3, §5). It introduces no correlation field (Shape §4), commits to a single selector space as the shape recommends (Shape §3.2), and follows append-only evolution (Shape §6).
- **To the Common Core.** It exercises exactly the latitude Core §5 grants a binding — fixing the wire selector and the argument positions, and choosing them (here, faithfully) independently of the abstract names and listing order — while preserving the §4 semantics in full.

An implementation claiming this binding implements `../../interface-profiles/push/rpc-push-oneway.md` carried in `../../../../encodings/json-array-message-shape.md`, and any client or server written to the abstract derivative behaves identically when its messages take this form.

## 9. Security Considerations

This binding inherits the considerations of the derivative (OneWay §9), the shape (Shape §8), the representation bindings (Designator §7, Payload §6), the Common Core (Core §6), and Architecture §8, and weakens none. It adds only what the concrete form makes specific.

- **Directionality enforces self-scoping at the wire.** Because each selector is valid in exactly one direction (Section 5), a receiver that enforces direction prevents a peer from invoking an operation the channel does not grant it — a client cannot present `"Deliver"`/`"NeighborhoodUpdate"` to act as the server, nor the server `"Send"` to act as a node. A receiver MUST enforce this; accepting a wrong-direction selector would breach the self-scoping the connection is relied upon to provide (Core §6, Push §7).
- **Every received array is untrusted.** A receiver MUST validate well-formedness, selector, direction, and argument arity and type before dispatch (Section 6), and MUST treat a delivered `Payload` as untrusted application data, parsing it defensively above this binding (Payload §6). As the shape notes, the JSON parse is itself a resource-exhaustion vector; a receiver SHOULD bound parse depth and input size, the bound being a transport/deployment concern (Shape §8, Relay §7).
- **Silence signals nothing.** Under one-way operation a message yields no reply — no acknowledgement, rejection, or error (Section 7). A sender MUST NOT infer receipt, acceptance, processing, or failure from silence, nor build a security property on such an inference (OneWay §9).
- **A `NeighborhoodUpdate` exposes the neighbor set.** Carrying the state as a visible JSON array of designators reveals a node's current out-neighbor count to that node, and, where designators are predictable, may reveal structure (Designator §7). This binding exposes designators exactly as the companions require and adds no exposure beyond making the set a JSON array; an implementation concerned with this mints opaque, high-entropy designators (Designator §7) — a confidentiality choice above this binding, not a correctness one.
- **No confidentiality.** Every message is cleartext JSON traversing the server (Architecture §8); this binding provides no end-to-end confidentiality. A client requiring secrecy encrypts within the `Payload` (Payload §6) or relies on a confidential transport (for example, TLS/WSS), which is the transport's concern.

## 10. References

### 10.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format.
- GRS RPC One-Way Pushable Derivative (`../../interface-profiles/push/rpc-push-oneway.md`).
- GRS JSON-Array Message Shape (`../../../../encodings/json-array-message-shape.md`).
- GRS RPC Pushable Profile (`../../interface-profiles/push/rpc-push-profile.md`).
- GRS RPC Common Core (`../../interface-profiles/rpc-interface.md`).
- GRS Designator String Representation (`../../data-shapes/designator-string.md`).
- GRS Payload String Representation (`../../data-shapes/payload-string.md`).
- GRS Relay and Neighborhood Semantics (`../../../relay-and-neighborhood-semantics.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).
