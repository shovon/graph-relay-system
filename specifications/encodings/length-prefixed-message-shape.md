# GRS Length-Prefixed Message Shape

## Status of This Memo

This document fixes a concrete **message shape**: the layout of a single application message as a sequence of length-prefixed octet fields. It defines no operations and names no procedures; it fixes only the _form_ a message takes on the wire — a flat run of fields, each prefixed with its own length, of which the first selects an operation and the rest are that operation's positional arguments. For the encoding of a single field it adopts the length-prefixed encoding (`./length-prefixed-encoding.md`), which is that encoding's single home; on every other point it stands alone and depends on no further GRS document.

The shape rests on three premises (Section 2), stated here so the document can be read on its own:

1. **A reliable, message-oriented transport sits beneath it.** That layer delimits one message from the next and, where the application needs related messages paired, performs that **correlation**. It need not be an L4 transport: it may be a bespoke message-oriented protocol built over TCP, or over WebSocket, or any layer that delivers whole messages reliably and, when required, correlated. This shape therefore carries **no message identifier and no correlation field** — correlation is resolved below it, not here.
2. **An opaque application payload sits above it.** Where an operation carries application data, that data occupies one of the message's fields and is opaque to this shape: not inspected, not interpreted, carried and recovered octet-for-octet.
3. **The protocol is fire-and-forget.** The shape defines no response, acknowledgement, or error message. A message is emitted and nothing is owed back at this layer. End-to-end acknowledgement, where an application needs it, is the application's to construct, above this shape.

It is one shape among possible others — an implementation MAY frame its messages differently — and is normative for implementations that adopt the length-prefixed field shape. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in RFC 2119.

## Table of Contents

1. Terminology
2. Design Premises
3. The Message Shape 3.1. A Message Is a Sequence of Length-Prefixed Fields 3.2. The Field Encoding 3.3. The Selector 3.4. Positional Arguments 3.5. The Application Payload
4. Fire-and-Forget: No Response, No Correlation
5. Well-Formedness and Receiver Handling
6. Evolution and Forward Compatibility
7. Layering: What This Shape Does Not Provide
8. Security Considerations
9. References

## 1. Terminology

A **message** is one unit the sender hands to the transport for delivery and the receiver obtains whole: under this shape, exactly one octet string laid out as the field sequence of Section 3.

A **field** is one length-prefixed octet run within that string: a length, then exactly that many octets of content.

A **selector** is the first field of the message — the value that names which operation the message invokes.

A **positional argument** is any field after the selector; arguments are identified by their position, in order, not by name.

An **operation** is the named action a selector denotes. This shape does **not** define any operations; it fixes only how a message that invokes one is laid out. The set of operations, the selector each is assigned, and the number, order, and meaning of each operation's arguments are fixed by a **binding** outside this document.

A **payload** is application data an operation carries within one of its fields; it is opaque to this shape (Section 3.5).

The layer beneath this shape — the one that delivers whole messages and, where needed, correlates them — is the **transport** (Section 2, premise 1). The layer above — the one that decides what operations exist and what a payload means — is the **application** (Sections 2, 7).

## 2. Design Premises

This shape is the middle of three layers, and is defined entirely by what it delegates to the other two.

**Premise 1 — a reliable, message-oriented, correlating transport beneath.** This shape assumes the layer below it delivers whole messages (it frames; this shape does not) reliably and, where the application requires related messages to be paired, correlates them. That layer is **not** required to sit at L4: it may be TCP itself, a message-oriented protocol layered over TCP, a WebSocket data channel, or any equivalent that preserves message boundaries and supplies correlation when asked. Because correlation lives there, **this shape carries no correlation identifier of its own** (Section 4).

**Premise 2 — an opaque application payload above.** Where an operation carries application data, that data is a field and is opaque here: this shape neither reads nor ascribes structure to it (Section 3.5). What a payload _means_ is the application's, constructed above this shape.

**Premise 3 — fire-and-forget.** No message under this shape is a reply to another, and no message expects one. The shape defines no acknowledgement, status, or error message, and provides a sender no feedback channel. Whatever end-to-end guarantee an application needs — delivery confirmation, retry, ordering across operations, request/response pairing of its own — it builds above this shape, encoded within payloads and sent as ordinary messages (Section 7).

Each premise removes a concern from this layer and names where it lives instead. The shape that remains is small precisely because these three are delegated.

## 3. The Message Shape

### 3.1. A Message Is a Sequence of Length-Prefixed Fields

A message is a single octet string holding zero or more fields laid end to end, with no wrapper, delimiter, or padding between them. Each field carries its own length (Section 3.2), so the boundaries are recovered from the octets alone: a reader keeps a cursor, reads a field, advances past it, and repeats until the message's octets are exhausted. The first field read is the selector (Section 3.3); each field after it, in order, is a positional argument (Section 3.4).

```
message  =  field(selector)  field(arg0)  field(arg1)  ...
```

A message that carries an operation taking no arguments is the single-field string holding only the selector:

```
message  =  field(selector)
```

The message is **flat**: this shape ascribes a field no internal structure and nests nothing. A field's content is an opaque octet run; whether those octets themselves spell a further length-prefixed structure is a binding's business (Section 3.4), invisible here.

### 3.2. The Field Encoding

A single field is encoded by the **length-prefixed encoding** (`./length-prefixed-encoding.md`): its length as a 32-bit unsigned integer in big-endian (network) byte order, immediately followed by exactly that many octets of content. Writing `LP(a)` for that encoding of an octet string `a`,

```
LP(a)  =  u32( |a| )  ||  a
```

exactly as `./length-prefixed-encoding.md` §1 fixes it — `u32(n)` is `n` as a four-octet big-endian unsigned integer, `|a|` is the octet length of `a`, and `||` is concatenation. This shape adopts that encoding unchanged for each of its fields and adds nothing to it; the field slots of Section 3.1 are `LP(selector)`, `LP(arg0)`, and so on. The prefix is exactly four octets and is not negotiated per field or per message. The content may be any octet string, including the **empty string**, whose encoding `LP(ε)` is the four octets `00 00 00 00` and nothing more — a well-formed field of length zero (`./length-prefixed-encoding.md` §2). A field of `2^32` octets or more cannot be represented and is an encoding error (ibid.); every field this shape carries sits far below that ceiling.

This fixed, self-delimiting encoding is the whole of the shape's mechanism, and it is what makes the field sequence unambiguous: because each field announces its own length, distinct sequences never collide on the same octet string (`./length-prefixed-encoding.md` §3) — no reader can slide octets across a field boundary, merge two fields into one, or split one into two without changing the message. The reader's single primitive — read four octets as a length `n`, take the next `n` octets as content, advance — recovers exactly the sequence the sender wrote.

### 3.3. The Selector

The selector is the first field of the message and names the operation the message invokes. It MUST be a **non-empty** octet string: its length prefix is nonzero.

This shape ascribes the selector's octets **no type and no structure**: they are an opaque, non-empty run compared by exact octet equality. A receiver MUST dispatch on the selector's octets as a whole. Which concrete octet values are valid selectors, the operation each denotes, and whether those octets are to be read as a compact integer tag, a textual name, or anything else, are fixed by a **binding** outside this shape (Section 1); this document fixes only the slot, that it is a non-empty octet string, and the exact-equality dispatch rule. A receiver that recognizes the selector's octets in neither value nor length does not dispatch (Section 5).

### 3.4. Positional Arguments

Every field after the selector is a positional argument. Arguments are ordered and identified by position alone; this shape assigns them no names and ascribes them no meaning.

The **number of arguments, their order, and the octet content each must take** are fixed per operation by the binding that defines that operation — never by this shape. This shape places **no constraint** on an argument's content: any octet string, empty or not, is a structurally valid argument as far as the shape is concerned. In particular, a binding MAY give a field internal structure of its own — for instance, letting an argument's octets themselves be a nested run of length-prefixed sub-fields — and read that structure above this shape; to the shape the field remains one opaque octet run whose length its prefix already fixed. Whether a given field is _acceptable_ in a given position is the operation's rule, applied by the receiver above the shape (Section 5).

This is the sense in which the shape is operation-agnostic: it can carry any operation set whatever, because it fixes only that arguments are length-prefixed fields following the selector, and defers everything about _which_ arguments an operation takes to that operation's binding.

### 3.5. The Application Payload

Where an operation carries application data — a payload — that payload occupies one of the operation's fields, at a position the operation's binding fixes. To this shape the payload is an **opaque field**: the shape does not inspect it, parse it, ascribe structure to it, or constrain its octets beyond Section 3.4.

Because a field carries arbitrary octets, a payload that is naturally binary is carried directly in its field, and a payload that is naturally text is carried as its octets under whatever character encoding the application chose; either way the shape is unaware of the distinction. It carries the field as it carries any other, and a receiver hands the payload field up to the application, which alone interprets it — treating it as untrusted data (Section 8).

The shape mandates no fixed position for the payload; that, like all argument layout, is the operation's binding to fix (Section 3.4).

## 4. Fire-and-Forget: No Response, No Correlation

No message under this shape is a response, and no message under this shape expects one. Every message is a one-way emission.

It follows that the message carries **no correlation identifier, sequence number, response or status field, or reply field**, and an implementation MUST NOT add one at this layer. There are two independent reasons, either of which alone would suffice, and which here coincide:

- **Correlation belongs to the transport** (Section 2, premise 1). Pairing related messages, where an application needs it, is done by the layer beneath this shape. Reintroducing a correlation field would duplicate, at the wrong layer, a facility the transport already owns.
- **There are no responses to correlate** (Section 2, premise 3). The shape defines no reply message, so no message's meaning depends on being matched to a prior one. Every message — selector and arguments — is self-contained and interpretable on its own.

A sender therefore receives nothing back from this layer: not delivery, not acceptance, not rejection, not an error. The shape's only response is silence, and silence signals nothing (Section 8). An application that needs acknowledgement, error reporting, or request/response pairing of its own constructs it above the shape, encoding it within payloads and routing it as ordinary one-way messages (Section 7), exactly as it constructs any other end-to-end guarantee.

## 5. Well-Formedness and Receiver Handling

A received message is **well-formed** when all of the following hold:

1. it parses cleanly as a field sequence (Section 3.2): reading a four-octet length then that many content octets, repeatedly, consumes the message's octets **exactly** — no length prefix runs past the end of the message, and no octets remain after the final field;
2. it holds at least one field;
3. the first field is non-empty (Section 3.3);
4. each field after the selector, if any, is a well-formed field of any content (Section 3.4).

Anything else is **malformed**: a truncated length prefix (fewer than four octets remain where a length is expected); a length that overruns the message's remaining octets; trailing octets that do not begin a complete field; a message of zero fields; or a selector field of length zero.

Well-formedness is structural only. A message can be well-formed yet still not be one the receiver can act on — because its selector denotes no operation the receiver implements (an **unrecognized selector**), or because its arguments do not satisfy the operation's own rules (an **invalid invocation**, judged by the operation above this shape).

Because the shape carries no response (Section 4), a receiver **cannot and does not reply** to report any of these conditions. Accordingly:

- On a **malformed** message, a receiver MUST NOT dispatch any operation and SHOULD discard the message.
- On a well-formed message with an **unrecognized selector**, a receiver MUST NOT dispatch and SHOULD discard.
- On an **invalid invocation**, the receiving operation MUST NOT act on the message as if valid; whether it discards, partially processes, or applies some operation-defined recovery is that operation's rule, above this shape.

In none of these cases does this layer surface anything to the sender. A receiver MAY take **transport-level** action — for instance, closing the connection on a stream of malformed input — but that is the transport's affair, invoked by deployment policy, not a response defined by this shape. Silence is the only response the shape has, and a sender MUST NOT read meaning into it (Section 8).

A receiver MUST treat every received message as untrusted input and validate well-formedness before dispatch (Section 8).

## 6. Evolution and Forward Compatibility

The positional field sequence gives operations an **append-only** growth path, and this section fixes the discipline that makes it safe.

An operation is extended by **appending** new fields after its existing ones. A sender MUST NOT reorder existing fields, repurpose an existing position, or insert a new field before an existing one; it may only append. Correspondingly, a receiver SHOULD **tolerate trailing fields** beyond those the operation defines for the version it implements — ignoring them rather than treating the message as malformed — so that a newer sender's extended message remains processable by an older receiver. Together these let an operation grow without consuming a new selector.

A receiver that must be strict MAY instead reject a message bearing unexpected trailing fields, but in doing so it forgoes this append-extensibility and couples senders and receivers to an exact field count; the tolerance above is RECOMMENDED for that reason, with the smuggling caveat of Section 8 noted.

A wholly **new operation** is introduced by allocating a new selector (Section 3.3). An older receiver, not recognizing it, discards the message (Section 5); the new operation reaches only receivers that implement it.

The shape itself is **unversioned**. Where a binding needs explicit versioning, it expresses it through the **selector** — for example, by assigning distinct selectors to distinct versions of an operation — not by adding a version field to the message, which would reintroduce structure this shape does not define and conflate version with argument position.

## 7. Layering: What This Shape Does Not Provide

This shape is small by construction; the following are deliberately **not** its concerns, and each names where it lives instead.

Below this shape, in the **transport** (Section 2, premise 1):

- **Framing** — delimiting one message from the next. This shape lays out exactly one field sequence per transport message and defines no multi-message framing of its own; the transport says where one message's octets end and the next begins.
- **Correlation** — pairing related messages (Section 4).
- **Reliability, ordering, and deduplication** — whether a message arrives, in what order relative to others, and whether duplicates are suppressed.

Above this shape, in the **application** (Section 2, premises 2 and 3):

- **The operation set, selector assignments, and argument schemas** — fixed by a binding (Section 1), not by this shape.
- **The meaning and internal structure of a payload** (Section 3.5).
- **End-to-end acknowledgement and any other end-to-end guarantee** — delivery confirmation, retry, request/response pairing, liveness — all constructed above the shape and sent as ordinary messages (Section 4).

What remains to this shape, and to it alone, is the layout: a run of length-prefixed fields, a selector first, positional arguments thereafter, and the well-formedness and dispatch rules over them. Nothing more.

## 8. Security Considerations

- **Every received message is untrusted input.** A receiver MUST validate that a message is well-formed (Section 5) before dispatching, and MUST treat every field — and especially any opaque payload (Section 3.5) — as untrusted application data, parsing it defensively above the shape. A delivered message may be malformed for the application's purposes, hostile, or crafted to exploit a parser.
- **Bound the parse.** Although framing and size limits belong to the transport (Section 7), the field parse is invoked at this layer. A declared field length is attacker-controlled and precedes the octets it counts; a receiver MUST NOT trust a length prefix before checking it against the octets actually remaining (Section 5), and a receiver SHOULD bound the total message size and any per-field length it will accept before allocating against it. A large declared length is a resource-exhaustion vector; the concrete bound is deployment-defined and is enforced in concert with the transport.
- **Selector matching is exact.** Because the selector is opaque octets dispatched by exact equality (Section 3.3), a receiver MUST match its whole octet content — length and bytes together — and MUST NOT dispatch on a prefix, a truncation, or a case-folded or otherwise normalized form of it. Any interpretation of the selector's octets as an integer or a name is the binding's, applied only after an exact match.
- **Silence signals nothing.** Under fire-and-forget (Section 4) a message yields no reply — no acknowledgement, no rejection, no error. A sender MUST NOT infer receipt, acceptance, processing, or failure from the shape's silence, and MUST NOT build a security property on such an inference. Any confirmation an application requires is constructed above the shape (Section 7).
- **Tolerance can be abused.** The forward-compatible tolerance of trailing fields (Section 6) means a naive receiver may ignore extra fields an attacker appends to smuggle data past inspection. A receiver for which this matters either constrains what it ignores or rejects unexpected trailing fields outright (Section 6), accepting the loss of append-extensibility.
- **No identity and no confidentiality at this layer.** A message asserts nothing about its origin: the selector and arguments carry no authenticated identity, and possession of a selector confers no authority — the shape is not a capability. The octets are carried in the clear; the shape provides no end-to-end confidentiality. Authentication and confidentiality, where required, are provided by the transport (for example, a TLS/WSS channel) or constructed within the payload by the application; this shape provides neither.

## 9. References

### 9.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- GRS Length-Prefixed Encoding (`./length-prefixed-encoding.md`): the single-field encoding this shape adopts (Section 3.2).
