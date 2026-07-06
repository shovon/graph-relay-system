# Packet Framing: Newline-Delimited RPCs

The [interface profile](./interface-profile.md) fixes the envelope in the abstract — a packet is an envelope, the **RPC** is the unit of meaning, one packet may carry several RPCs, and addressing and validity are decided **per RPC**. It deliberately says nothing about the bytes. This memo supplies the bytes, and it is a small commitment: **a packet is its RPCs, one per line, as newline-delimited JSON.** Everything the interface profile promised — multiplexing, per-RPC dispatch, drop-the-RPC-not-the-packet — falls out of that one framing without further machinery.

## 1. A packet is JSONL

A packet is a sequence of RPC messages laid down one per line and separated by a single line feed (`LF`, `U+000A`) — the framing known interchangeably as **JSON Lines** or **newline-delimited JSON** (JSONL / NDJSON). Each line is exactly one message under the [JSON-array message shape](../encodings/json-array-message-shape.md): a JSON array whose first element is the RPC's **selector** and whose remaining elements are that RPC's positional arguments, in order.

Two delimiters operate at two levels, and keeping them apart is the whole design:

- **The newline delimits one RPC from the next, within a packet.** This is precisely the "delimits one message from the next" role the message shape ([premise 1](../encodings/json-array-message-shape.md)) assumes some layer beneath it will play; here that layer is the newline.
- **The transport delimits one packet from the next.** On a message-oriented transport — a WebSocket data message is the canonical case — one transport message carries one packet. On a bare byte stream the packet grouping simply has no separate wire mark: the edge carries a continuous JSONL stream of RPCs, which is harmless, because nothing on the receiving side dispatches on the packet. Every decision the interface profile defines is per-RPC, so a packet boundary that is present frames a batch and a packet boundary that is absent costs nothing.

## 2. Multiplexing is just more lines

A packet MAY carry more than one RPC — an **extension** RPC beside a `send`, say ([interface profile](./interface-profile.md), §"Why carry more than one RPC") — and that needs nothing special here. It is two lines in one packet:

```
[ "grs.overlay.v1/send",      ... ]
[ "<some-extension-selector>", ... ]
```

A receiver reads the packet line by line. Each line it dispatches or discards **on its own terms**: a line whose selector names an operation it implements, it runs; a line it cannot parse, or whose selector it does not recognize, it drops — without touching the lines around it. That is [drop-the-RPC-not-the-packet](./interface-profile.md) read literally as **drop-the-line**. An unrecognized extension does not take a co-traveling `send` down with it, because they are two lines and nothing binds their fates together.

## 3. The line is compact — the one invariant

JSONL rests on a single rule: **no message contains an unescaped newline**, so that one message occupies exactly one line and the newline is an unambiguous frame.

- A message **MUST** be serialized compact: no insignificant whitespace, and in particular no interior newline. A JSON array of legal values never *needs* an interior newline — structural characters have none, and a newline inside a JSON string escapes to `\n` — so this rule costs nothing but forbids pretty-printing on the wire.
- Any **base64** argument **MUST** use the **unwrapped** alphabet — no line breaks inserted every 64 or 76 characters, as MIME and PEM base64 do. This is the one sharp edge. The [trail](./path-emergence/trail-wire-format.md) rides as a base64 argument (the binary buffer, encoded into a JSON string), and a wrapped blob would plant newlines mid-message and silently shred the framing. Unwrapped base64 is the only conformant form here.

Beyond that: the stream is UTF-8 (RFC 3629); a trailing `LF` after the final message is permitted; and a blank line carries no message and is ignored.

## 4. Notes on design

- **Why newline framing, and not an outer array.** Wrapping a packet's RPCs in one enclosing JSON array — `[[sel, …], [sel, …]]` — would make each RPC an *inner* array, and the packet's own element 0 would then be an array, which the [message shape](../encodings/json-array-message-shape.md)'s well-formedness (element 0 is a string or a non-negative integer) does not admit. You would need a second, packet-level shape layered over the message one. Newline framing avoids the whole second level: **every line is a first-class message-shape message**, and the packet is a pure batch with no grammar of its own. Per-line dispatch is then literally per-RPC dispatch — the property the interface profile is built around — rather than something reconstructed after un-nesting.
- **Why the transport frames packets, not RPCs.** The message shape wants a message-oriented layer beneath it. We could have made *each RPC* a transport message and let the transport delimit RPCs directly — but then two RPCs could never share a hop, and the multiplex the interface profile exists to allow would be impossible. Letting the **newline** frame RPCs and the **transport** frame packets keeps the RPC the unit of meaning while still letting several travel together. On WebSocket this is free: one frame, one packet.
- **The packet is inert on the wire, by design.** Because the interface profile makes addressing and validity per-RPC, a packet needs no header, no RPC count, no type tag — it is nothing but the lines that happened to travel together on one hop. That inertness is exactly why the loose ends above are loose ends and not problems: a blank line is ignored, a trailing newline is fine, and a transport with no message boundary simply turns the packet into an unmarked run of lines. There is no packet-level invariant left to violate.
- **base64 is the only thing that can silently break this.** Everything else that could smuggle a newline into a line is ruled out by "serialize compact." Wrapped base64 is the exception because it is a *correctly encoded* value that happens to contain newlines, so a naive encoder produces it without complaint. Stating the unwrapped requirement normatively (§3) is what turns a silent framing corruption into a conformance error.
