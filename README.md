# Graph Relay System (GRS)

A central server holds a directed graph whose nodes are the connected clients, keeps it **strongly connected**, and lets each client talk only to its immediate **out-neighbors**. The server relays along those edges and does no more — everything richer (replies, broadcast, end-to-end delivery to a non-neighbor) is built by clients _above_ this one primitive.

The bet is that a small, sharply-defined coordination layer — "who may talk to whom, maintained centrally" — is more useful than a server that also decides _what_ gets said. The server carries adjacency and reachability; the substance of what clients exchange over it is deliberately left open. The system is a substrate, and its utility is an emergent property of how clients use the neighbor primitive over a graph that is guaranteed to be traversable.

## The shape, in one breath

- **Directed edges.** `X → Y` means X may send to Y. It does **not** imply `Y → X`. Receiving a message grants no ability to answer it — a reply path, if needed, is a construction, not a given.
- **Strong connectivity.** From any node the server can reach every other by following out-edges. This is a structural guarantee the server _maintains_, not a feature it _exercises_; it only ensures any reachability a client wants to build is possible in principle.
- **Neighbor-scoped sends.** A client names its out-neighbors by **designator** and sends only to them. A designator that no longer denotes a current out-neighbor yields a dropped message — never a misdelivery.
- **Best-effort relay.** The server attempts delivery but guarantees none. A reliable transport (TCP, WebSocket) _may_ strengthen this; it may not weaken it.

The analogy (which is **not** the topology): the out-neighbor relation is like "follows" on a social network. Alice following Bob puts Bob in Alice's neighborhood, but not the reverse unless Bob follows back.

## Repository layout

```
specifications/   the protocol, as a layered spec corpus (start here)
grs-server/       reference WebSocket server (TypeScript / Node)
clean-updating-graph-example/   React + Vite visualization of a live graph
parking-lot/      ideas extracted from the specs, not yet re-derived from first principles
questions/        open questions / working notes
```

## The specifications

The corpus is layered: a small architectural core, companions that fix the semantics the core defers, then concrete transport bindings. Each lower layer **adds and narrows** — a _derivative_ may strengthen a guarantee or fix an open choice, but never relax or contradict what a layer above it fixed. Code written to the base contract keeps working against any conformant derivative (_substitutability_).

**Read in this order:**

1. [`architecture.md`](specifications/relay/architecture.md) — the GRS Protocol. The strongly-connected directed graph, directionality and the absence of a reply channel, server responsibilities, and what a conformant derivative may and may not do.
2. [`relay-and-neighborhood-semantics.md`](specifications/relay/relay-and-neighborhood-semantics.md) — fixes the finer semantics the architecture leaves out: neighborhood state, designators, the **No-Misdelivery** property, change notification, and the best-effort relay contract.
3. [`rpc-interface.md`](specifications/relay/conformance/interface-profiles/rpc-interface.md) — the **RPC Common Core**: the abstract data (`Payload`, `Designator`, `NeighborhoodState`), the `Send` operation, and the responsibilities that differ by transport — specialized by one of two profiles.

**Transport bindings & conformance** live in [`specifications/relay/conformance/`](specifications/relay/conformance/):

- **Profiles** — pick exactly one. `rpc-push-profile.md` for full-duplex transports where the connection _is_ the session (WebSocket, TCP); `rpc-pull-profile.md` for request/response transports (HTTP), layered on `rpc-pull-session.md` which synthesizes a session over a connectionless transport.
- **Identity** — `unique-session-identity.md` strengthens the base's ephemeral, per-session identity toward per-session uniqueness, while explicitly declining cross-session persistence.
- **`bindings/`** — concrete WebSocket bindings (`websocket-resource.md`, `websocket-transport-binding.md`).
- **`data-shapes/`** — wire representations (`designator-string.md`, `payload-string.md`, `json-array-message-shape.md`).

**Topology** — [`topologies/degree-3-graph.md`](specifications/topologies/degree-3-graph.md) describes the Degree-3 Balanced Tree (D3BT) the reference server uses to keep the graph connected and shallow: every node holds at most three symmetric slots, insertion walks toward the shallowest part of the tree, and deletion re-hangs orphaned pieces at the centroid of the largest survivor.

**Overlay** — [`overlay/`](specifications/overlay/) specifies _path emergence_: how clients construct end-to-end routing above the neighbor primitive, in [stateless](specifications/overlay/path-emergence/stateless/stateless.md) (origin carries the full source route) and [stateful](specifications/overlay/path-emergence/stateful/stateful.md) (trails accumulate hop by hop) variants. This is the canonical worked example of "reach beyond the neighborhood is a client construction," not part of the server's contract.

## The code

### `grs-server/`

A reference relay server over WebSocket (the Pushable Profile). It maintains the client graph as a Degree-3 Balanced Tree — `src/adjacency-list.mts` holds the tree primitives (link, centroid, insert/delete), and `server.mts` wires them to a `ws` `WebSocketServer`.

```bash
cd grs-server
npm install
npm test          # node --test over src/**/*.test.mts
```

### `clean-updating-graph-example/`

A React + Vite front end that visualizes a graph updating live — a way to _see_ the topology repair itself as nodes join and leave.

```bash
cd clean-updating-graph-example
npm install
npm run dev
```

## Status

This is aspirational, exploratory work: a high-level architecture meant to inspire implementations, plus a reference realization and a running visualization. The specs sketch the shape and deliberately leave detail to companion documents and to implementers.
