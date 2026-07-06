# The recoverable signature encoding: Bitcoin's 65-byte compact form

ECDSA over secp256k1 fixes a signature as a pair of scalars $(r, s)$ but says nothing about their concrete octet layout on the wire — fixed-width $r \Vert s$ versus DER, and whether a recovery id rides alongside, are all left open. This document fixes that layout for the *recovery* case: where a verifier reconstructs the signing key from a signature and the message it signed, rather than being handed the key separately. It commits the on-wire signature to **Bitcoin's 65-byte compact recoverable signature** — a recovery-id byte followed by fixed-width $r$ and $s$.

The choice is not arbitrary reuse. The recovery case has a specific need that a bare $(r, s)$ does not meet: it must carry the recovery id beside each signature so the recovered key is unambiguous. Bitcoin's compact format is exactly a secp256k1 signature with that id packed into a leading byte — the container the ecosystem's libraries already emit and consume as a unit. We adopt the container. We do **not** adopt Bitcoin's message-hashing convention that usually rides with it; §3 draws that line.

## 1. The 65-byte layout

A signature $\sigma = (r, s)$ together with its recovery id $v$ encodes to exactly 65 octets:

$$
\text{Compact}(v, r, s) = \text{header}(v) \mathbin\Vert \text{be32}(r) \mathbin\Vert \text{be32}(s),
$$

where $\text{be32}(x)$ is the integer $x$ written as a 32-byte big-endian unsigned value, zero-padded on the left, and $\mathbin\Vert$ is concatenation. Both $r$ and $s$ are field-order scalars for secp256k1, so each fits in 32 bytes; a scalar shorter than 32 bytes is left-padded, never truncated or minimally encoded. The result is one header byte plus $32 + 32$, a flat 65 bytes with no internal length prefixes — the field widths are constants of the curve, so nothing needs to announce them.

## 2. The header byte

The header encodes the recovery id and commits, by construction, to the compressed point form of the recovered key:

$$
\text{header}(v) = 31 + v, \qquad v \in \{0, 1, 2, 3\}.
$$

This is Bitcoin's compact-signature header with the compressed flag set — the base $27 + v$ plus $4$ for "the recovered key is used in compressed form." Setting the flag pins the recovered key to its compressed SEC1 encoding (33 octets: parity byte plus $x$), so the header always lands in $\{31, 32, 33, 34\}$. An uncompressed header ($27$–$30$) is not a second permitted encoding of the same signature; it is malformed under this binding and rejected, so a signature still maps to exactly one byte string.

The recovery id $v$ names which of the candidate points the verifier's $\text{Recover}$ reconstructs: its low bit is the parity of the nonce point $R$, and its high bit distinguishes the rare case where $R$'s $x$-coordinate exceeded the group order before reduction. Carrying $v$ inline is what lets a verifier recover the key in one call rather than recover-and-match against a candidate set.

## 3. What is *not* borrowed: the preimage

The compact envelope is a container for $(v, r, s)$ and nothing more. Bitcoin's `signmessage` also fixes what gets signed *inside* that envelope — it prepends the `"Bitcoin Signed Message:\n"` magic string and hashes with **double** SHA-256. This encoding adopts none of that.

What is signed, and recovered against, is out of the envelope's scope: the frame commits to $(v, r, s)$, and the caller supplies the 32-byte digest the signature is over. No magic prefix, no second hash. An implementer reaching for a Bitcoin library must therefore call its raw compact-signature *encoder and recoverer* over a supplied 32-byte digest, not its message-signing helper — the helper would compute a different preimage. This binding fixes the 65-byte frame and leaves the preimage to whoever uses it.

## 4. Low-$s$, and why the id is pinned with it

The encoding admits only the low-$s$ signature: an $s > n/2$ is non-canonical, and its 65-byte encoding is rejected whatever its header. That restriction is what makes the recovery id safe to carry in the clear. Malleability lets $(r, s)$ and $(r,\, n - s)$ both verify; hold $v$ fixed and flip $s$ and recovery lands on a different, unowned key. Low-$s$ removes the second signature outright, so for each message there is one canonical $s$, one permitted header, and thus one recovered key.

The compact format's virtue here is that it keeps $v$ and $s$ in one frame, so the pinning is a property of a single 65-byte object rather than of two fields an implementation might validate apart.

## 5. The base form keeps the 64-byte framing

The recovery-id byte earns its place only when the key is recovered rather than transmitted. Where the public key is carried alongside the signature instead, there is nothing for $v$ to disambiguate. The base form's signature is therefore the same fixed-width pair **without** the header — a flat 64 octets,

$$
\text{Compact}_{\text{base}}(r, s) = \text{be32}(r) \mathbin\Vert \text{be32}(s),
$$

with $r$, $s$, and the low-$s$ rule exactly as above. The two forms are distinct wire shapes: the recovery form drops the public key from what is transmitted yet its signature is one byte larger by carrying $v$, so the net saving of recovering rather than transmitting the key is the public key minus that byte.

## 6. Notes on design

- **Why borrow Bitcoin's container at all.** The alternative was to invent a framing — say, $\text{LP}(r) \Vert \text{LP}(v) \Vert \text{LP}(s)$ in a length-prefixed style of one's own. That would be self-consistent but parochial: it would not interoperate with the audited secp256k1 libraries whose recovery calls already speak the 65-byte compact layout natively. Adopting the format the tooling emits means the recovery id an implementation gets back from `sign` is already in the shape it puts on the wire, with no re-framing step to get wrong.
- **Why fixed-width, not DER.** DER is the other standard layout for an ECDSA signature. It is variable-length and carries its own type-length structure, which reopens a canonicalization surface: non-minimal lengths, negative-padding ambiguities, a parser that must reject as much as it accepts. Fixed 32-byte big-endian scalars have one encoding each; there is nothing to canonicalize. The compact format is that fixed-width choice with the recovery id already attached.
- **Why the high bit of $v$ is effectively dead.** $v \in \{2, 3\}$ signals that the nonce point's $x$-coordinate exceeded the group order before reduction — possible only for $r$ in a window of width $n - p$ below the field prime, which for secp256k1 is a stretch of astronomically small measure. Under low-$s$ a conforming signer will emit $v \in \{0, 1\}$ in all practice, and the header in $\{31, 32\}$. The $\{2, 3\}$ cases are specified only so the encoding is total, not because an implementation will meet them.
