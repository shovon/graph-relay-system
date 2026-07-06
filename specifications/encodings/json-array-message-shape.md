# GRS JSON-Array Message Shape

## Status of This Memo

This document fixes a concrete **message shape**: the layout of a single application message as a JSON array. It is deliberately self-contained. It defines no operations, names no procedures, and depends on no other GRS document; it fixes only the _form_ a message takes on the wire — a JSON array whose first element selects an operation and whose remaining elements are that operation's positional arguments.

The shape rests on three premises (Section 2), stated here so the document can be read on its own:

1. **A reliable, message-oriented transport sits beneath it.** That layer delimits one message from the next and, where the application needs related messages paired, performs that **correlation**. It need not be an L4 transport: it may be a bespoke message-oriented protocol built over TCP, or over WebSocket, or any layer that delivers whole messages reliably and, when required, correlated. This shape therefore carries **no message identifier and no correlation field** — correlation is resolved below it, not here.
2. **An opaque application payload sits above it.** Where an operation carries application data, that data occupies one of the array's positional arguments and is opaque to this shape: not inspected, not interpreted, encoded and recovered verbatim.
3. **The protocol is fire-and-forget.** The shape defines no response, acknowledgement, or error message. A message is emitted and nothing is owed back at this layer. End-to-end acknowledgement, where an application needs it, is the application's to construct, above this shape.

It is one shape among possible others — an implementation MAY frame its messages differently — and is normative for implementations that adopt the JSON-array shape. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119.

## Table of Contents

1. Terminology
2. Design Premises
3. The Message Shape 3.1. A Message Is a JSON Array 3.2. The Selector 3.3. Positional Arguments 3.4. The Application Payload
4. Fire-and-Forget: No Response, No Correlation
5. Well-Formedness and Receiver Handling
6. Evolution and Forward Compatibility
7. Layering: What This Shape Does Not Provide
8. Security Considerations
9. References

## 1. Terminology

A **message** is one unit the sender hands to the transport for delivery and the receiver obtains whole: under this shape, exactly one JSON array.

A **selector** is the first element of that array — the value that names which operation the message invokes.

A **positional argument** is any element of the array after the selector; arguments are identified by their position, in order, not by name.

An **operation** is the named action a selector denotes. This shape does **not** define any operations; it fixes only how a message that invokes one is laid out. The set of operations, the selector each is assigned, and the number, order, and meaning of each operation's arguments are fixed by a **binding** outside this document.

A **payload** is application data an operation carries within one of its positional arguments; it is opaque to this shape (Section 3.4).

The layer beneath this shape — the one that delivers whole messages and, where needed, correlates them — is the **transport** (Section 2, premise 1). The layer above — the one that decides what operations exist and what a payload means — is the **application** (Sections 2, 7).

## 2. Design Premises

This shape is the middle of three layers, and is defined entirely by what it delegates to the other two.

**Premise 1 — a reliable, message-oriented, correlating transport beneath.** This shape assumes the layer below it delivers whole messages (it frames; this shape does not) reliably and, where the application requires related messages to be paired, correlates them. That layer is **not** required to sit at L4: it may be TCP itself, a message-oriented protocol layered over TCP, a WebSocket data channel, or any equivalent that preserves message boundaries and supplies correlation when asked. Because correlation lives there, **this shape carries no correlation identifier of its own** (Section 4).

**Premise 2 — an opaque application payload above.** Where an operation carries application data, that data is a positional argument and is opaque here: this shape neither reads nor ascribes structure to it (Section 3.4). What a payload _means_ is the application's, constructed above this shape.

**Premise 3 — fire-and-forget.** No message under this shape is a reply to another, and no message expects one. The shape defines no acknowledgement, status, or error message, and provides a sender no feedback channel. Whatever end-to-end guarantee an application needs — delivery confirmation, retry, ordering across operations, request/response pairing of its own — it builds above this shape, encoded within payloads and sent as ordinary messages (Section 7).

Each premise removes a concern from this layer and names where it lives instead. The shape that remains is small precisely because these three are delegated.

## 3. The Message Shape

### 3.1. A Message Is a JSON Array

A message is a single JSON array, as defined by RFC 8259, encoded in UTF-8 (RFC 3629). The array's first element is the selector (Section 3.2); its remaining elements, in order, are the operation's positional arguments (Section 3.3).

```
[ selector, arg0, arg1, ... ]
```

A message that carries an operation taking no arguments is the one-element array holding only the selector:

```
[ selector ]
```

The following are **illustrative** — the selectors and arguments shown belong to no operation this document defines and are present only to show the layout:

```
[ "note", "hello world" ]          // string selector, one argument
[ "ping" ]                         // string selector, no arguments
[ 7, "n7gKQ", "hello world" ]      // integer selector, two arguments
```

The top-level JSON value is **always an array**. A JSON object, a bare string, number, boolean, or null at top level is not a message under this shape (Section 5). Insignificant JSON whitespace around or within the array does not affect its meaning.

### 3.2. The Selector

The selector is element 0 of the array and names the operation the message invokes. It MUST be either:

- a **non-empty JSON string** (the _string selector space_), or
- a **JSON number that denotes a non-negative integer** (the _integer selector space_) — no fraction, no exponent yielding a non-integer, no negative value.

These are the two selector spaces this shape permits. A given message's selector lies in exactly one of them, determined by its JSON type. Which concrete values are valid selectors, and the operation each denotes, are fixed by a **binding** outside this shape (Section 1); this document fixes only the slot, the two admissible types, and the dispatch rule below.

A receiver MUST dispatch on the selector's **value and JSON type together**. The string `"0"` and the integer `0` are **distinct selectors** and MUST NOT be conflated unless a binding explicitly assigns both to the same operation. A receiver that recognizes neither the value nor its space does not dispatch (Section 5).

A binding SHOULD adopt a **single** selector space across its operation set, and a deployment SHOULD use that one space consistently. The shape permits both spaces so that _bindings may choose_ between a human-readable string form and a compact integer form — not so that one deployment must accept both and reconcile them. An integer-space binding SHOULD keep selector values within the interoperably representable integer range (`0` through `2^53 − 1`), so that a value survives any conformant JSON implementation unchanged.

### 3.3. Positional Arguments

Every element after the selector is a positional argument. Arguments are ordered and identified by position alone; this shape assigns them no names and ascribes them no meaning.

The **number of arguments, their order, and the JSON type each must take** are fixed per operation by the binding that defines that operation — never by this shape. This shape places **no type constraint** on an argument: any JSON value (string, number, boolean, null, array, or object) is a structurally valid argument as far as the shape is concerned. Whether a given value is _acceptable_ in a given position is the operation's rule, applied by the receiver above the shape (Section 5).

This is the sense in which the shape is operation-agnostic: it can carry any operation set whatever, because it fixes only that arguments are positional and follow the selector, and defers everything about _which_ arguments an operation takes to that operation's binding.

### 3.4. The Application Payload

Where an operation carries application data — a payload — that payload occupies one of the operation's positional arguments, at a position the operation's binding fixes. To this shape the payload is an **opaque argument**: the shape does not inspect it, parse it, ascribe structure to it, or constrain its JSON type beyond Section 3.3.

A payload that is naturally a JSON value MAY be carried as that value directly in its argument position; a payload that is not naturally JSON (for example, binary data) is encoded by the application into a JSON value (for example, a base64 string) before it occupies its position. Either way the shape is unaware of the distinction: it carries the argument as it carries any other, and a receiver hands the payload argument up to the application, which alone interprets it — treating it as untrusted data (Section 8).

The shape mandates no fixed position for the payload; that, like all argument layout, is the operation's binding to fix (Section 3.3).

## 4. Fire-and-Forget: No Response, No Correlation

No message under this shape is a response, and no message under this shape expects one. Every array is a one-way emission.

It follows that the array carries **no correlation identifier, sequence number, response or status element, or reply field**, and an implementation MUST NOT add one at this layer. There are two independent reasons, either of which alone would suffice, and which here coincide:

- **Correlation belongs to the transport** (Section 2, premise 1). Pairing related messages, where an application needs it, is done by the layer beneath this shape. Reintroducing a correlation field in the array would duplicate, at the wrong layer, a facility the transport already owns.
- **There are no responses to correlate** (Section 2, premise 3). The shape defines no reply message, so no message's meaning depends on being matched to a prior one. Every message — selector and arguments — is self-contained and interpretable on its own.

A sender therefore receives nothing back from this layer: not delivery, not acceptance, not rejection, not an error. The shape's only response is silence, and silence signals nothing (Section 8). An application that needs acknowledgement, error reporting, or request/response pairing of its own constructs it above the shape, encoding it within payloads and routing it as ordinary one-way messages (Section 7), exactly as it constructs any other end-to-end guarantee.

## 5. Well-Formedness and Receiver Handling

A received message is **well-formed** when all of the following hold:

1. it is valid JSON (RFC 8259) and its top-level value is an array;
2. the array has at least one element;
3. element 0 is a non-empty string or a non-negative-integer number (Section 3.2);
4. each element after the selector, if any, is any JSON value (Section 3.3).

Anything else is **malformed**: input that is not valid JSON; a top-level value that is not an array; an empty array; or a selector that is null, a boolean, an array, an object, the empty string, a non-integer number, or a negative number.

Well-formedness is structural only. A message can be well-formed yet still not be one the receiver can act on — because its selector denotes no operation the receiver implements (an **unrecognized selector**), or because its arguments do not satisfy the operation's own rules (an **invalid invocation**, judged by the operation above this shape).

Because the shape carries no response (Section 4), a receiver **cannot and does not reply** to report any of these conditions. Accordingly:

- On a **malformed** message, a receiver MUST NOT dispatch any operation and SHOULD discard the message.
- On a well-formed message with an **unrecognized selector**, a receiver MUST NOT dispatch and SHOULD discard.
- On an **invalid invocation**, the receiving operation MUST NOT act on the message as if valid; whether it discards, partially processes, or applies some operation-defined recovery is that operation's rule, above this shape.

In none of these cases does this layer surface anything to the sender. A receiver MAY take **transport-level** action — for instance, closing the connection on a stream of malformed input — but that is the transport's affair, invoked by deployment policy, not a response defined by this shape. Silence is the only response the shape has, and a sender MUST NOT read meaning into it (Section 8).

A receiver MUST treat every received array as untrusted input and validate well-formedness before dispatch (Section 8).

## 6. Evolution and Forward Compatibility

The positional array gives operations an **append-only** growth path, and this section fixes the discipline that makes it safe.

An operation is extended by **appending** new positional arguments after its existing ones. A sender MUST NOT reorder existing arguments, repurpose an existing position, or insert a new argument before an existing one; it may only append. Correspondingly, a receiver SHOULD **tolerate trailing arguments** beyond those the operation defines for the version it implements — ignoring them rather than treating the message as malformed — so that a newer sender's extended message remains processable by an older receiver. Together these let an operation grow without consuming a new selector.

A receiver that must be strict MAY instead reject a message bearing unexpected trailing arguments, but in doing so it forgoes this append-extensibility and couples senders and receivers to an exact argument count; the tolerance above is RECOMMENDED for that reason, with the smuggling caveat of Section 8 noted.

A wholly **new operation** is introduced by allocating a new selector (Section 3.2). An older receiver, not recognizing it, discards the message (Section 5); the new operation reaches only receivers that implement it.

The shape itself is **unversioned**. Where a binding needs explicit versioning, it expresses it through the **selector space** — for example, by assigning distinct selectors to distinct versions of an operation — not by adding a version element to the array, which would reintroduce structure this shape does not define and conflate version with argument position.

## 7. Layering: What This Shape Does Not Provide

This shape is small by construction; the following are deliberately **not** its concerns, and each names where it lives instead.

Below this shape, in the **transport** (Section 2, premise 1):

- **Framing** — delimiting one message from the next. This shape carries exactly one JSON array per transport message and defines no multi-message framing of its own.
- **Correlation** — pairing related messages (Section 4).
- **Reliability, ordering, and deduplication** — whether a message arrives, in what order relative to others, and whether duplicates are suppressed.

Above this shape, in the **application** (Section 2, premises 2 and 3):

- **The operation set, selector assignments, and argument schemas** — fixed by a binding (Section 1), not by this shape.
- **The meaning and internal structure of a payload** (Section 3.4).
- **End-to-end acknowledgement and any other end-to-end guarantee** — delivery confirmation, retry, request/response pairing, liveness — all constructed above the shape and sent as ordinary messages (Section 4).

What remains to this shape, and to it alone, is the layout: a JSON array, a selector in element 0, positional arguments thereafter, and the well-formedness and dispatch rules over them. Nothing more.

## 8. Security Considerations

- **Every received message is untrusted input.** A receiver MUST validate that a message is well-formed (Section 5) before dispatching, and MUST treat every argument — and especially any opaque payload (Section 3.4) — as untrusted application data, parsing it defensively above the shape. A delivered message may be malformed for the application's purposes, hostile, or crafted to exploit a parser.
- **Bound the parse.** Although framing and size limits belong to the transport (Section 7), the JSON parse is invoked at this layer. A deeply nested or very large array is a resource-exhaustion vector; a receiver SHOULD bound parse depth and input size before or during parsing. The concrete bound is deployment-defined and is enforced in concert with the transport.
- **Selector type confusion.** Because both selector spaces are permitted (Section 3.2), a receiver MUST distinguish them by JSON type and MUST NOT conflate a string selector with an integer of the same spelling (`"0"` versus `0`) unless a binding maps both to one operation. Treating them interchangeably invites an attacker to invoke an unintended operation by switching the selector's type. Adopting a single space per deployment (Section 3.2) removes the ambiguity.
- **Silence signals nothing.** Under fire-and-forget (Section 4) a message yields no reply — no acknowledgement, no rejection, no error. A sender MUST NOT infer receipt, acceptance, processing, or failure from the shape's silence, and MUST NOT build a security property on such an inference. Any confirmation an application requires is constructed above the shape (Section 7).
- **Tolerance can be abused.** The forward-compatible tolerance of trailing arguments (Section 6) means a naive receiver may ignore extra elements an attacker appends to smuggle data past inspection. A receiver for which this matters either constrains what it ignores or rejects unexpected trailing arguments outright (Section 6), accepting the loss of append-extensibility.
- **No identity and no confidentiality at this layer.** A message asserts nothing about its origin: the selector and arguments carry no authenticated identity, and possession of a selector confers no authority — the shape is not a capability. JSON is cleartext; the shape provides no end-to-end confidentiality. Authentication and confidentiality, where required, are provided by the transport (for example, a TLS/WSS channel) or constructed within the payload by the application; this shape provides neither.

## 9. References

### 9.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format.
- RFC 3629: UTF-8, a transformation format of ISO 10646.
