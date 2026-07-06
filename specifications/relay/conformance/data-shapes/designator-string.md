# GRS Designator String Representation

## Status of This Memo

This document is a concrete **representation binding** for the `Designator` abstract type of the _GRS RPC Common Core_ (`../interface-profiles/rpc-interface.md`, Core §3), whose semantics are fixed by _GRS Relay and Neighborhood Semantics_ (`../../relay-and-neighborhood-semantics.md`, Relay §2). It fixes one decision those documents deliberately leave open (Core §3, §5; Relay §7): the **concrete form a designator takes**. Under this binding, a designator is a **string**.

It fixes representation only. It changes none of the designator's semantics — the per-state distinctness requirement and the no-misdelivery guarantee hold exactly as the companions state them; this document restates them in terms of the string form and adds nothing to them. In particular, it does **not** prescribe how a server generates designators: the requirement that a node's out-neighbors be distinctly designated is preserved, but the _mechanism_ by which an implementation satisfies it is left to the implementer (Section 4). Like the companions, it fixes nothing about what a designator means from one neighborhood state to the next; that is left to a future layer (Relay §2, §4).

It is one binding among possible others — an implementation MAY represent a designator differently — and is normative for implementations that adopt the string representation. Section references of the form (Core §N) point into `../interface-profiles/rpc-interface.md`, (Relay §N) into `../../relay-and-neighborhood-semantics.md`, and (Architecture §N) into `../../architecture.md`.

## Table of Contents

1. Terminology
2. Representation
3. Opacity and Permitted Operations
4. Distinct Designation
5. Resolution and Matching
6. Encoding
7. Security Considerations
8. References

## 1. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

This document uses the terms **Server**, **Client**, **Node**, **out-neighbor**, **neighborhood**, **neighborhood state**, and **Designator** as defined in Architecture §2 and refined in Relay §2. A **designator string** is the concrete realization of a `Designator` this binding fixes.

## 2. Representation

A designator is a **string**: a finite sequence of Unicode code points. This binding places no constraint on its contents; a future specification MAY restrict the permitted strings (for example, by reserving the empty string as a sentinel), but no such restriction is imposed here.

- Two designator strings are **equal** if and only if they are identical code-point sequences. Equality is exact: no case folding, no Unicode normalization, and no leading/trailing whitespace trimming is performed by either the server or the client when comparing, issuing, or resolving a designator. A server that generates designators and a client that echoes them back MUST preserve them byte-for-meaning verbatim, so that an issued designator compares equal to the same designator presented on a later `Send`.
- The maximum length of a designator string is not fixed here; it is bounded by the transport binding or deployment (Relay §7). A server SHOULD keep designators short, as they are server-generated and a client only ever echoes them back (Section 3).

This binding ascribes **no internal structure** to the string. Any structure a particular server's generation scheme imposes (Section 4) is private to that server and MUST NOT be relied upon by a client (Section 3).

## 3. Opacity and Permitted Operations

A designator string is an **opaque token**. A client MUST treat it as such: it MUST NOT parse it, interpret its bytes, infer a neighbor's identity, position, or count from it, order designators, or synthesize a designator of its own. The only operations a client may perform on a designator are:

1. **Select** a designator verbatim from its _current_ neighborhood state (Relay §5), to name the out-neighbor that state associates with it.
2. **Pass** that designator verbatim as the address of a `Send` (Core §4.1).
3. **Compare for equality**, _within a single neighborhood state_, to tell that state's out-neighbors apart — the distinctness this binding guarantees (Section 4).

This binding fixes no meaning for a designator beyond the single neighborhood state in which it appears, and likewise fixes no relationship between designators drawn from different states. Whether designators may be compared across states, and what such a comparison would mean, is not defined here; a client that needs continuity of identity across neighborhood changes constructs it above this interface (Architecture §3.2), and a future layer MAY give designators cross-state meaning (Relay §2).

## 4. Distinct Designation

Within a node's **single current neighborhood state**, the server MUST assign **pairwise-distinct** designator strings to that node's out-neighbors — no two out-neighbors of the same node share a designator in the same state — so that each out-neighbor can be addressed unambiguously (Relay §2).

**How a server achieves this distinctness is implementation-defined.** This binding fixes that designators are distinct strings, not how those strings are minted. An implementation is free to choose any scheme that yields the required per-state distinctness; by way of non-normative example:

- small integers scoped to the neighborhood, rendered as decimal strings (`"0"`, `"1"`, `"2"`, …);
- per-edge random tokens;
- UUIDs;
- an opaque hash of a server-internal neighbor key.

The distinctness requirement is the _only_ uniqueness this binding mandates. Consistent with Relay §2, a server is **not** required to make a designator globally unique, persistent across reconnection, or stable across a node's neighborhood changes. A server MAY reuse a string to denote a different out-neighbor in a later state, and MAY change the string that denotes a given out-neighbor from one state to the next.

An implementation MAY provide guarantees stronger than per-state distinctness — stability across states, global uniqueness, unguessability (Section 7), or persistence across a node's session boundary, the last where a derived identity model supplies a durable node identity (Relay §2, Architecture §3) — and, if it does, SHOULD document them. Absent such a documented guarantee, this binding provides none beyond per-state distinctness, and what may be built on a stronger guarantee is a matter for the layer that defines it (Relay §2).

## 5. Resolution and Matching

The server resolves the designator carried on a `Send` by **exact string equality** (Section 2) against the sending node's _current_ neighborhood state:

- if exactly one current out-neighbor is denoted by an equal designator, the server relays the payload to that out-neighbor;
- otherwise — no current out-neighbor's designator is equal, because the string is malformed, was drawn from an earlier neighborhood state, or simply never denoted one — the server **discards** the payload.

Because matching is exact equality against the current neighborhood, a designator that does not denote a present out-neighbor yields discard, never misdelivery and never reach to a non-neighbor (Relay §3, §4). Per-state distinctness (Section 4) guarantees the match is unambiguous: at most one current out-neighbor can satisfy it.

## 6. Encoding

This binding fixes the abstract form (a string of Unicode code points); the wire encoding is the transport binding's concern (Core §5). Under a JSON or JSON-RPC encoding, a designator is carried as a JSON string (RFC 8259) and a `NeighborhoodState` exposes each out-neighbor's designator as such; the exact-equality rule of Section 2 then applies to the decoded code-point sequence, not to the on-wire octets. A transport binding that carries text in another form MUST preserve the designator's code-point sequence so that equality (Section 2) is decided identically on both sides.

## 7. Security Considerations

This binding inherits the considerations of Core §6 and Architecture §8 and adds only what the string representation makes concrete.

- **Designators are not capabilities, and their secrecy is not the safety mechanism.** No-misdelivery rests on resolving every `Send` against the caller's _current_ neighborhood (Section 5, Relay §3), not on a designator being unguessable. A client can only `Send` as its own node (Core §6, self-scoping) and only to that node's current out-neighbors; a guessed or forged string therefore relays only if it already denotes one of the caller's _own_ current out-neighbors, and is otherwise discarded. Consequently an implementation MAY use predictable designators (e.g. `"0"`, `"1"`, `"2"`) without enabling misdelivery or reach to a non-neighbor.
- **Predictable designators may leak structure.** A scheme whose designators expose neighbor counts, positions, or churn (e.g. dense integers) may reveal information about a node's neighborhood to its holder. An implementation that wishes to avoid this MAY mint opaque, high-entropy designators instead; this is a confidentiality choice, not a correctness one (the previous point still holds either way).
- **Oversized input.** Because a client echoes designators back on `Send` (Section 3), a server MUST guard against resource exhaustion from over-long or malformed designator strings on input, independent of the (non-)match that resolution (Section 5) will produce. Concrete length limits are a transport/deployment concern (Relay §7).

## 8. References

### 8.1. Normative References

- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels.
- GRS RPC Common Core (`../interface-profiles/rpc-interface.md`).
- GRS Relay and Neighborhood Semantics (`../../relay-and-neighborhood-semantics.md`).
- Graph Relay System (GRS) Protocol (`../../architecture.md`).

### 8.2. Informative References

- RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format.
- GRS RPC Pull Profile (`../interface-profiles/pull/rpc-pull-profile.md`) and GRS RPC Pushable Profile (`../interface-profiles/push/rpc-push-profile.md`): the profiles whose `Send` and neighborhood-state operations carry designators in this representation.
