# GRS RPC One-Way Pushable Derivative

## Status of This Memo

This document is a **derivative** of the _GRS RPC Pushable Profile_ (`rpc-push-profile.md`) in the sense of Architecture §7.1: it fixes one choice that profile deliberately leaves open and changes nothing else. The Pushable Profile defines `Send` with "at most an acceptance decision" and permits it to be one-way (Push §5.1, Core §4.1); this derivative **fixes that choice to strictly one-way**, so that _every_ operation in the interface — the one client-initiated operation and both server-initiated pushes — is one-way. The interface carries no responses, and therefore defines no response correlation.

This document constrains only the **call shape** of the interface — the axis Core §5 leaves to a binding ("request/response versus one-way, and correlation of responses"). It says nothing about the transport beneath, and nothing about delivery quality: ordering, reliability, buffering, retention, and deduplication remain exactly as the Pushable Profile and any transport binding under it leave them (Relay §7). The property fixed here is a property of the interface, not of any transport that carries it.

It is normative for implementations claiming the GRS One-Way Pushable Derivative. It depends on, without restating, everything fixed by the Pushable Profile and the Common Core. Section references of the form (Push §N) point into `rpc-push-profile.md`; (Core §N) into `../rpc-interface.md`; (Relay §N) into `../../../relay-and-neighborhood-semantics.md`; (Architecture §N) into `../../../architecture.md`.

## Table of Contents

1. Terminology
2. What This Derivative Fixes
3. The One-Way Interface 3.1. `Send` (client → server) 3.2. `Deliver` (server → client) 3.3. `NeighborhoodUpdate` (server → client)
4. No Responses, No Correlation
5. Silent Discard
6. No Response-Eliciting Operations
7. Replies Are Constructed Above the Interface
8. Conformance
9. Security Considerations
10. References

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119. This derivative uses all terms of the Pushable Profile (Push §1) and the Common Core (Core §1).

A **one-way** operation is one that surfaces no output: it produces no result, acceptance decision, acknowledgement, or error returned to its invoker. A **response** is any message whose purpose is to answer a prior message. **Correlation** is the pairing of a response to the message it answers, by an identifier or by any other means; where there are no responses, there is nothing to correlate.

## 2. What This Derivative Fixes

The Pushable Profile fixes the receiving half of the relay and neighborhood-state availability as server pushes, both already one-way (Push §5.2, §5.3). The one remaining call-shape choice it leaves open is on `Send`, which it permits to surface an acceptance decision or to be one-way (Push §5.1, Core §4.1, Core §5). This derivative fixes that single choice:

> `Send` is **one-way**. It surfaces no acceptance decision and no output of any kind.

With that fixed, the interface contains no operation that produces a response (Section 3). The consequences — no correlation (Section 4), silent discard of unaddressable sends (Section 5), and no response-eliciting operations (Section 6) — follow from this one decision; they are not independent choices but restatements of it at each surface where a response might otherwise have appeared.

## 3. The One-Way Interface

Every operation is one-way. The interface defines exactly one client-initiated operation and two server-initiated pushes, and none of the three returns anything to its invoker.

### 3.1. `Send` (client → server)

- **Direction**: client-initiated.
- **Input**: a `Designator` and a `Payload`.
- **Output**: **none.** One-way.

As Push §5.1, with the acceptance decision removed. Relay semantics are unchanged: best-effort, No-Misdelivery, and resolution of the designator against the node's current neighborhood at the moment of sending (Relay §3, §4). The server still resolves and relays or discards exactly as before; this derivative changes only that the outcome of that resolution is **not surfaced** to the sender (Section 5). A sender invokes `Send` and is owed nothing back.

### 3.2. `Deliver` (server → client)

- **Direction**: server-initiated (push).
- **Input**: a `Payload`.
- **Output**: none.

Unchanged from Push §5.2. Already one-way under the profile: the receiving half of the relay, pushed as each payload arrives, carrying no sender designator and no reply path, owing and expecting no acknowledgement (Core §4.3, Architecture §3.1).

### 3.3. `NeighborhoodUpdate` (server → client)

- **Direction**: server-initiated (push).
- **Input**: the node's new `NeighborhoodState`.
- **Output**: none.

Unchanged from Push §5.3. Already one-way under the profile: the server pushes the updated state on every neighborhood change, and SHOULD push the initial state on establishment (Push §3). A client's most recently received state is its current one; it stays current without asking (Section 6).

## 4. No Responses, No Correlation

Because no operation in Section 3 produces a response, the interface defines **no request/response pairing and no correlation identifier**. There is no message whose meaning depends on being matched to a prior one; every message — a `Send` up, a `Deliver` or `NeighborhoodUpdate` down — is self-contained and interpretable on its own.

This is not the omission of a correlation facility that the interface might otherwise have wanted: correlation is the machinery for delivering responses to the right caller, and this interface has no responses to deliver. An implementation MUST NOT introduce a correlation identifier, reply field, or request/response framing at this layer, because there is nothing for it to pair. Any such mechanism an application needs belongs above the interface (Section 7).

## 5. Silent Discard

A `Send` whose designator does not denote a current out-neighbor — whether malformed, or no longer denoting one after a graph change — is discarded, exactly as the relay requires (Relay §4). Because `Send` is one-way (Section 3.1), this discard is **not reported to the sender**: the interface surfaces no discard indication. This is permitted — a discard indication is OPTIONAL in the base (Core §5, Relay §4) — and it is the deliberate consequence of fixing `Send` one-way. A sender therefore receives no per-send feedback of any kind: not delivery (which the base never promised; Core §6, Relay §6), and under this derivative not acceptance or discard either. A sender that needs to know whether a peer received something constructs that knowledge above the interface (Section 7).

## 6. No Response-Eliciting Operations

The Pushable Profile permits the server to additionally answer an on-demand neighborhood query for explicit re-sync (Core §4.2). This derivative does **not** define such an operation, and an implementation claiming this derivative MUST NOT add one or any other client-initiated operation that elicits a response — doing so would reintroduce a response into an interface this derivative fixes as response-free (Section 4). Neighborhood state stays current by the server's `NeighborhoodUpdate` push alone (Section 3.3); a client never queries for it. Re-synchronization, where wanted, is a server-initiated push, not a client-initiated request.

## 7. Replies Are Constructed Above the Interface

Removing responses from the interface removes nothing an application cannot rebuild above it, exactly where the base already places such constructions (Architecture §3.1, §3.2, Core §4.3). A `Deliver` carries no sender and no reply path; an application that needs to identify an originator, answer a message, acknowledge receipt, or correlate one of its own request payloads with a later response payload encodes all of that **within the opaque `Payload`** and routes it over the relay primitive like any other reach to a non-neighbor. The interface neither sees nor assists such schemes; it relays self-contained one-way messages and nothing more.

## 8. Conformance

This derivative is GRS-conformant by the test of Architecture §7.1: it **fixes a choice the base leaves open** (the call shape of `Send`; Push §5.1, Core §5) and strengthens nothing into a contradiction. Substitutability holds — code written to the Pushable Profile continues to work unmodified against this derivative:

- A client that already treated `Send` as one-way, or that ignored its acceptance decision, sees no change. The base requires every client to do exactly this: acceptance is not delivery, and a sender MUST NOT infer delivery from it nor from the absence of a discard indication (Core §6, Relay §6). This derivative removes a signal no conformant client was permitted to rely on.
- Every guarantee and prohibition the base fixes is preserved: No-Misdelivery, resolution against the current neighborhood, the best-effort floor, and the no-systematic-refusal rule (Relay §3, §4, §6) all continue to hold, because this derivative changes only whether `Send`'s outcome is surfaced, not how the server resolves or relays it.

A derivative is meant to add and narrow, never to subtract (Architecture §7.1); this one narrows `Send` to one-way and adds the interface-wide property that follows. It does not relax, remove, or contradict any base requirement, and it claims conformance on that basis.

## 9. Security Considerations

This derivative inherits the considerations of the Pushable Profile (Push §7), the Common Core (Core §6), and Architecture §8, and weakens none of them: it introduces no credential, no handle, and no new operation, and the self-scoping binding of caller to node by the connection is untouched (Push §7).

- **No guarantee is weakened by removing the acceptance decision.** Because the base forbids treating acceptance as delivery (Core §6), removing it takes away nothing a conformant client could have relied on for a security property.
- **The interface offers a sender no feedback channel.** Under this derivative a `Send` yields nothing back, including no discard signal for an unaddressable designator (Section 5). An application that requires confirmation, error reporting, or liveness of a peer MUST construct it above the interface (Section 7); it cannot be inferred from the interface's silence, which signals nothing (Core §4.3).

## 10. References

### 10.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- GRS RPC Pushable Profile (`rpc-push-profile.md`).
- GRS RPC Common Core (`../rpc-interface.md`).
- Graph Relay System (GRS) Protocol (`../../../architecture.md`).
- GRS Relay and Neighborhood Semantics (`../../../relay-and-neighborhood-semantics.md`).
