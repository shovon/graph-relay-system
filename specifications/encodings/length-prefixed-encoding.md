# Length-prefixed encoding

This document fixes one concrete **length-prefixed encoding**: an unambiguous, injective, self-delimiting encoding of a sequence of fields, in which each field is prefixed with its length and the length is a 32-bit integer. The primitive (§1) is generic — any format that must lay a sequence of fields down as one byte string from which the fields are recovered exactly can adopt it — and this document is its single home.

Its motivating consumer is the signed hash chain. The chain in [`../signed-hash-chain/signed-hash-chain.md`](../signed-hash-chain/signed-hash-chain.md) requires that every multi-argument application of $H$, $\text{Sign}$, and $\text{Vrfy}$ combine its components by "an unambiguous, injective encoding — length-prefixed, or otherwise self-delimiting"; this document removes that latitude and pins the encoding concretely. Everything below that speaks of a hashed or signed tuple is that consumer's use of the primitive, not a limit on it: a field is a field, whatever a caller does with the encoded string.

## 1. The encoding

For a byte string $a$, write $|a|$ for its length in bytes. The _length-prefixed encoding_ of a single field is

$$
\text{LP}(a) = \text{u32}\big(|a|\big) \mathbin\Vert a,
$$

where $\text{u32}(n)$ is $n$ written as a 32-bit unsigned integer in big-endian (network) byte order, and $\mathbin\Vert$ is byte-string concatenation.

A variadic application of a primitive to several fields _is_ that primitive applied to the concatenation of the fields' length-prefixed encodings, in order. That is, for $f \in \{H, \text{Sign}, \text{Vrfy}, \text{Recover}\}$,

$$
f(a_0, a_1, \ldots, a_{k-1}) = f\big(\text{LP}(a_0) \mathbin\Vert \text{LP}(a_1) \mathbin\Vert \cdots \mathbin\Vert \text{LP}(a_{k-1})\big).
$$

The variadic form on the left is shorthand for the single-argument form on the right; the chain's primitives are never handed a bare tuple, only the byte string this expands to. So, for the [embedded-key](../signed-hash-chain/with-embedded-keys/with-embedded-keys.md) form,

$$
h_i = H(i, h_{i-1}, p_i, m_i, \sigma_i),\qquad \sigma_i = \text{Sign}_{s_i}(i, h_{i-1}, p_i, m_i),
$$

and for the [key-omitted](../signed-hash-chain/without-embedded-keys/without-embedded-keys.md) form, which drops $p_i$ from the link,

$$
h_i = H(i, h_{i-1}, m_i, \sigma_i),\qquad \sigma_i = \text{Sign}_{s_i}(i, h_{i-1}, m_i),
$$

each carry the length prefixes above, and any verifier recomputes exactly these bytes before it checks the signature.

## 2. Constraints

- **Width and byte order.** The length prefix is exactly four bytes, unsigned, big-endian. This is fixed; it is not negotiated per field or per call.
- **Maximum field length.** A field of $2^{32}$ bytes or more cannot be represented and is an encoding error. Every field in this protocol — a hash digest, a public key, an edge designator, a signature — sits many orders of magnitude below that ceiling, so the limit is never reached in practice; it is stated only to make the encoding total and unambiguous.
- **The sequence number.** The position $i$ is a field like any other, but an integer rather than a byte string, so it is first rendered as a fixed-width **unsigned 64-bit big-endian** integer and then length-prefixed as that 8-byte field — its length prefix is therefore always $\text{u32}(8)$. A chain never approaches $2^{64}$ links, so the width is not a practical limit; fixing it makes the encoding of $i$ total and unambiguous, like every other field.
- **The empty field.** $\epsilon$ encodes as $\text{LP}(\epsilon) = \text{u32}(0)$ — four zero bytes and nothing more. So a genesis seed $h_{-1} = \epsilon$ from the overview's [base case](../signed-hash-chain/signed-hash-chain.md#3-the-base-case-is-out-of-scope) is the four-byte string `00 00 00 00`, distinct from every non-empty field.

## 3. Injectivity

A decoder reads four bytes to obtain a length $n$, takes the next $n$ bytes as the field, and repeats until the input is exhausted. Because each field carries its own length, the field boundaries are recovered exactly, and with them the original tuple. The map from tuples to byte strings is therefore injective: distinct tuples never share an encoding.

This holds across differing arities, not just differing splits at a fixed arity. The three field lists below all encode differently —

$$
(\text{"AB"}, \text{"C"}),\qquad (\text{"A"}, \text{"BC"}),\qquad (\text{"ABC"}),
$$

— because the first field's length prefix already disagrees (`2`, `1`, `3` respectively). That is exactly the separation the chain depends on: no adversary can slide bytes across a field boundary, merge two fields into one, or split one into two without changing the encoded string, and therefore without changing the hash or invalidating the signature.

## 4. Notes on design

- **Why a fixed width, not a varint.** A fixed-width prefix has exactly one encoding per length, so there is nothing to canonicalize and nothing to reject. A variable-length integer would shave a couple of bytes off a prefix that is already negligible against the fields it guards, and it would buy that with a parsing routine plus a malleability surface — non-minimal varints that an implementation must detect and refuse, or else reopen the ambiguity this encoding exists to close.
- **Why 32 bits, not 64.** The fields are a digest, a key, a designator, a signature. Four gibibytes of headroom is already far past absurd for any of them; an eight-byte prefix would be four wasted bytes on every field forever.
- **Why big-endian.** Network byte order is the conventional choice for wire formats, and the prefix stays legible in a hex dump. Either endianness is equally injective — the only thing that matters is that independent implementations commit to the _same_ one, and this is it.
