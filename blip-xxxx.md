```
bLIP: ?
Title: Independent per-commitment secrets
Status: Draft
Author: Dave Lund <dave@flowrate.com>
Created: 2026-10-06
License: CC0
```

## Abstract

This bLIP defines `option_independent_secrets`, a `channel_type` add-on under
which per-commitment secrets are not required to come from a single seed. Each
side may generate its secrets by any method, and each side keeps every secret
it receives.

This lets a node generate and reveal its secrets k-of-n, so that no single
signer can hand the channel peer the secret of the node's current commitment.

## Copyright

This bLIP is licensed under the CC0 license.

## Specification

### Feature bits

| Bits    | Name                         | Description                                           | Context | Dependencies          |
|---------|------------------------------|-------------------------------------------------------|---------|-----------------------|
| 264/265 | `option_independent_secrets` | Accepts per-commitment secrets not from a single seed | INT     | `option_channel_type` |

The even bit is only used in `channel_type`. The odd bit is set in `init` and
`node_announcement`.

A node:
  - SHOULD set `option_independent_secrets` in `init` and `node_announcement`,
    so that a peer which generates its secrets jointly can open channels with
    it.
  - if both peers set `option_independent_secrets`:
    - MAY include it in the `channel_type` it proposes.
    - if it does not generate its per-commitment secrets with the seed-based
      algorithm of BOLT 3:
      - MUST include it in the `channel_type` it proposes.
  - if it sets `option_independent_secrets`:
    - MUST accept a `channel_type` that adds `option_independent_secrets` to a
      `channel_type` it would otherwise accept.

### Changes to BOLT 2

For a channel with `option_independent_secrets`, the following requirement is
added to the receiver of `revoke_and_ack`.

A receiving node:
  - if `option_independent_secrets` applies to the channel:
    - MUST NOT fail the channel or close the connection because the
      `per_commitment_secret` is not related to earlier secrets as
      described in [BOLT #3](https://github.com/lightning/bolts/blob/master/03-transactions.md#per-commitment-secret-requirements).

The other requirements on `revoke_and_ack` are unchanged: in particular, the
receiver still fails the channel if `per_commitment_secret` is not a valid
secret key or does not generate the previous `per_commitment_point`.

### Per-commitment points

On a channel with `option_independent_secrets`, each per-commitment point is
computed from its per-commitment secret exactly as in BOLT 3
(`per_commitment_point = per_commitment_secret * G`), however the secret was
generated. This applies to every per-commitment point a node sends:
  - `first_per_commitment_point` in `open_channel`, `accept_channel`,
    `open_channel2` and `accept_channel2`.
  - `second_per_commitment_point` in `channel_ready`, `open_channel2` and
    `accept_channel2`.
  - `next_per_commitment_point` in `revoke_and_ack`.
  - `my_current_per_commitment_point` in `channel_reestablish`.

A node that does not generate its per-commitment secrets with the seed-based
algorithm of BOLT 3:
  - MUST include `option_independent_secrets` in the `channel_type` of the
    `open_channel` or `open_channel2` it sends, since that message already
    carries its first per-commitment point.
  - MUST NOT accept a channel whose `channel_type` does not include
    `option_independent_secrets`.

If the peer refuses the `channel_type`, the channel is not opened, and no
per-commitment secret has been revealed.

### Changes to BOLT 3

For a channel with `option_independent_secrets`, the requirements of
[Per-commitment Secret Requirements](https://github.com/lightning/bolts/blob/master/03-transactions.md#per-commitment-secret-requirements)
on the first secret, on the I'th secret and on the receiving node are replaced
by the following.

A node generating its per-commitment secrets:
  - MAY generate them by any method, including:
    - the seed-based algorithm of BOLT 3.
    - a key derivation function applied to a private seed and the commitment
      number.
    - a cryptographically secure random number generator.
    - a threshold protocol run by several signers.
  - MUST generate each per-commitment secret such that it cannot be guessed by
    its peer, including from any secrets it has already revealed.
  - MUST generate each per-commitment secret as a valid secp256k1 secret key.
  - MUST NOT use the same per-commitment secret for two commitment
    transactions.
  - MUST be able to reveal the per-commitment secret for every per-commitment
    point it has sent, including after a restart: for example, by persisting
    the secret, or whatever its signers need to reveal it jointly, before it
    sends the point.
  - MUST be able to produce the last per-commitment secret it revealed, in
    order to check `your_last_per_commitment_secret` in `channel_reestablish`.

A node receiving per-commitment secrets:
  - MUST NOT require that the per-commitment secrets it receives are related
    to each other in any way.
  - MUST keep each per-commitment secret it receives, so that it can produce
    the secret of every commitment transaction its peer has revoked, until the
    funding output has been spent irrevocably and, if a revoked commitment
    transaction spent it, until the outputs of that transaction are resolved.
  - MAY store per-commitment secrets in the compact representation in
    [Efficient Per-commitment Secret Storage](https://github.com/lightning/bolts/blob/master/03-transactions.md#efficient-per-commitment-secret-storage),
    for as long as each one it receives can be inserted into it:
    - otherwise, MUST store that secret, and every later one, individually,
      together with the number of the commitment transaction it revokes.

The commitment number is still 48 bits, so a channel with
`option_independent_secrets` still has at most 2^48 commitment transactions.

### Taproot channels

`option_independent_secrets` MAY be combined with the simple taproot channel
types. The per-commitment secrets of such a channel follow this bLIP as on any
other channel.

The simple taproot channel proposal also recommends deriving each MuSig2
verification nonce from a second shachain, seeded from the root of the
revocation secrets, so that a node can reproduce its nonces without storing
them. That scheme is local to the node: its peer never sees the secret nonces
and does not check how they were made.

A node whose signing keys are held by several signers:
  - MUST NOT derive its MuSig2 secret nonces from a seed that any single
    signer holds.
  - MUST persist each verification nonce it sends, or whatever its signers
    need to reproduce it, before sending it, as the simple taproot channel
    proposal already requires of nonces not made with its counter-based
    scheme.

## Motivation

The seed-based algorithm of BOLT 3 lets a receiver keep every secret in 49
entries, but it ties all of a node's secrets to one value. The seed cannot be
split across several signers with elliptic curve arithmetic, because each
derivation step is SHA256, so a node that wants its signing keys held k-of-n
still has to give the whole seed to every signer that may need to revoke a
commitment. Any one of those signers can then give the seed to the channel
peer, who learns the secret for the node's current commitment and can claim
its outputs through the revocation path as soon as the node broadcasts that
commitment. A "2-of-3" node is then really 1-of-1.

Independent secrets remove that coupling. A node can generate each secret
jointly, with no signer ever holding it, and reveal it only once enough signers
agree that the commitment is revoked.

## Rationale

The cost falls on the receiver of such secrets, which stores 32 bytes per
revoked commitment instead of a fixed 49 entries. A node already keeps
per-commitment data for every revoked state in order to punish a breach, so
this adds to existing storage rather than creating a new kind.

A single-signer node gains nothing from the option and loses nothing either: it
may keep using the seed-based algorithm, so it can still regenerate its own
secrets.

Nor does its peer pay for that node's secrets: secrets from the seed-based
algorithm can all be inserted into the compact representation, so the receiver
can keep its fixed size for as long as they keep coming that way. Insertion
checks each secret against the ones before it, so the first secret which cannot
be inserted shows that the sender is not using that algorithm, and every secret
inserted before it can still be produced. That secret and every later one have
to be stored individually, since the compact representation would lose them.
This also means a static backup that carries the compact representation can
still punish any state revoked by a seed-based peer.

A generator that does not derive its secrets from a seed must keep them, or the
means to reveal them, from the moment it sends their points. Losing a secret
whose point was sent leaves the node unable to revoke that commitment, and so
unable to move the channel past it.

The option is a `channel_type` add-on rather than a node-wide flag, so it is
agreed per channel like other channel features, and a channel's rules never
change after it opens.

Taproot channels need no further change to the protocol, because their nonces
are never checked by the peer. They do need care inside a multi-signer node. A
signer that knows another signer's secret nonce, and sees that signer's
partial signature while the signatures are combined, can compute that
signer's key share. Nonces derived from a seed that one signer holds would
therefore hand that signer the other shares, which is the same weakness the
shachain has for revocation secrets. Each signer has to make its own nonces,
and the node has to keep them, since it can no longer regenerate them from a
seed.

The idea was first proposed by ZmnSCPxj on Delving Bitcoin in April 2026
("no_more_shachains", option 2A in his
[summary of the options for a k-of-n node](https://delvingbitcoin.org/t/towards-a-k-of-n-lightning-network-node/2395)).

## Universality

Nothing here depends on the chain. It is a bLIP rather than a BOLT because it
is optional and only concerns the two peers of a channel. If it is widely
adopted, and multi-signer nodes need every peer to accept it, it can move into
the BOLTs and be made compulsory there.

## Backwards Compatibility

Peers that don't set the bit open channels as before. A channel only uses the
add-on if both peers agree to it in `channel_type`.

A node that has open channels with the add-on must not be downgraded to a
version without it: the older version would expect the peer's next secret to
extend a shachain, and fail the channel.

## Reference Implementations

Both are forks for the Bitcoin BLAKE2b chain; nothing in the change depends on
that chain. They interoperate with each other.

- lnd (Lightning Fork): `independent-secrets-blip` branch of
  https://github.com/davelund110/lightning-fork
- Core Lightning (privkeyio fork): `independent-secrets-blip` branch of
  https://github.com/davelund110/lightning
