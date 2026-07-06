# A cryptographic counter

You have a fixed, ordered list of parties, identified by their secp256k1 public keys $K_0, K_1, K_2, \ldots$ — a roster that already exists, built for some purpose of its own. You want a single running **count** that only those parties may advance, each in turn, and you want to be able to _prove_ afterward that it was advanced by exactly those parties, in exactly that order — without keeping a bloated log of who did what.

The whole trick is that the count need not be a bare integer that anyone could bump. It can be a value that only the next authorized party is able to produce, because producing it requires that party's signature over the current state. Advance it $n$ times and you hold not just the number $n$ but a proof of the $n$ authorized advances that reached it.

## 1. State

Two values, and no more, constitute the live counter:

- **The count $c$** — an ordinary integer. It is both the number of increments performed so far and the index of the next authorized incrementer: the party permitted to make increment $c$ is the one holding $K_c$.
- **The head $h$** — a single 32-byte rolling commitment, a cryptographic summary of every increment so far. It starts at a seed value $h_{-1}$ (see [§5](#5-freshness) on what to seed it with).

Everything else — the roster, and the record of past increments — either pre-exists or is carried by the incrementers themselves ([§4](#4-what-is-actually-stored)).

## 2. Incrementing

To perform increment $i$ — permitted only to the holder of $K_i$, i.e. when $c = i$ — that party signs the current state with its secret key $s_i$:

$$
\sigma_i = \text{Sign}_{s_i}(i, h_{i-1}).
$$

The counter then folds the signature into a fresh head and advances the count:

$$
h_i = H(i, h_{i-1}, \sigma_i), \qquad c \leftarrow i + 1.
$$

Two details carry the weight:

- **The signature commits to the previous head $h_{i-1}$**, which is itself a commitment to every increment before it. So $\sigma_i$ commits to the _entire history_ up to $i$, not merely to the act of incrementing once. This is what makes the record reorder- and splice-resistant: alter or move any earlier increment and $h_{i-1}$ changes, so $\sigma_i$ no longer fits where it sits.
- **$(i, h_{i-1})$ must be combined by an injective encoding** — length-prefixed, or otherwise self-delimiting — before signing, never by raw concatenation. Concatenating $i \mathbin\Vert h_{i-1}$ would let an adversary shift bytes across the boundary between the two fields and present a different $(i, h_{i-1})$ with an identical signed input. The same discipline applies to the arguments of $H$.

Note that the count $i$ is bound _into_ the signature but need not be transmitted alongside it: it is just the position of $\sigma_i$ in the sequence of increments, recovered by counting.

## 3. Verifying who incremented, and in what order

To verify, check each increment's signature against the roster key for its position. A verifier holding the seed $h_{-1}$, the roster $(K_0, K_1, \ldots)$, and the stream of signatures $(\sigma_0, \ldots, \sigma_n)$ walks forward from the seed, at each step requiring

$$
\forall\, i \le n : \quad \text{Vrfy}_{K_i}\big((i, h_{i-1}),\ \sigma_i\big) = 1,
$$

and then folding the head forward, $h_i = H(i, h_{i-1}, \sigma_i)$, before advancing. The prefix head $h_{i-1}$ it needs at each step is the one it computed the step before, bottoming out at the seed.

That single check does three things at once:

- **Authorization** — increment $i$ is verified against $K_i$, a roster key, so no outsider incremented.
- **Order** — increment $i$ must satisfy $K_i$ _specifically_, so the increments went in the roster's order.
- **Integrity** — because each $\sigma_i$ commits to $h_{i-1}$, tampering with or reordering any earlier increment changes every later head, so a later signature no longer verifies.

If every check passes, the count is $n + 1$, and you have proof that the first $n + 1$ roster members, in order, each advanced it once.

## 4. What is actually stored

The point of the exercise is to add little. Accounting honestly:

- **The count is not added data.** It is an ordinary integer, the length of the sequence, derived by counting increments.
- **The construction adds one value: the head $h$** — a single 32-byte rolling commitment, overwritten at each increment. That is the counter's entire standing footprint.
- **The roster pre-exists** and is reused as-is; it is not a cost this construction introduces.
- **The proof is the stream of signatures**, one per increment, each produced by the incrementer at the moment it increments — carried by the parties, not held as counter state.

There is a tradeoff worth stating plainly. Keep only the head $h$ and the count and you have a compact live summary — enough to check that a _freshly presented_ full stream of signatures reproduces $h$ — but you cannot reconstruct the history from the head alone; that requires retaining the signatures. Keep the signatures and you can re-derive and re-verify the whole run against the roster at any time. Either way the counter's own live state is a single hash plus an integer.

## 5. Freshness

This proves _who_ incremented and _in what order_, not _when_. Nothing above binds the count to a particular occasion, so a whole run of increments can be replayed — presented again as if fresh, or a longer run rolled back to a shorter prefix and re-advanced. If that matters, seed the head with a per-epoch nonce — derive $h_{-1}$ from it rather than from a bare constant — so a run from one epoch cannot pass as another's.
