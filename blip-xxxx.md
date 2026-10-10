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
| 266/267 | `option_independent_secrets` | Accepts per-commitment secrets not from a single seed | INT     | `option_channel_type` |

The even bit is only used in `channel_type`. The odd bit is set in `init` and
`node_announcement`.

A node:
  - SHOULD set `option_independent_secrets` in `init` and `node_announcement`,
    so that a peer which generates its secrets jointly can open channels with
    it.
  - if both peers set `option_independent_secrets`:
    - SHOULD include it in the `channel_type` it proposes, so that a peer that
      does not generate its per-commitment secrets with the seed-based
      algorithm of BOLT 3 can accept the channel.
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
(`per_commitment_point = per_commitment_secret * G`), no matter how the secret
was generated. This applies to every per-commitment point a node sends:
  - `first_per_commitment_point` in `open_channel`, `accept_channel`,
    `open_channel2` and `accept_channel2`.
  - `second_per_commitment_point` in `channel_ready`, `open_channel2` and
    `accept_channel2`.
  - `next_per_commitment_point` in `revoke_and_ack`.
  - `my_current_per_commitment_point` in `channel_reestablish`.

For a channel whose per-commitment secrets a node will not generate with the
seed-based algorithm of BOLT 3, that node:
  - MUST include `option_independent_secrets` in the `channel_type` of the
    `open_channel` or `open_channel2` it sends, since that message already
    carries its first per-commitment point.
  - MUST NOT accept the channel if its `channel_type` does not include
    `option_independent_secrets`.

With a peer that does not set `option_independent_secrets`, such a node either
generates that channel's secrets with the seed-based algorithm or does not open
the channel. If the peer refuses the `channel_type`, the channel is not opened,
and no per-commitment secret has been revealed.

### Changes to BOLT 3

For a channel with `option_independent_secrets`, the requirements of
[Per-commitment Secret Requirements](https://github.com/lightning/bolts/blob/master/03-transactions.md#per-commitment-secret-requirements)
on the first secret, on the I'th secret and on the receiving node are replaced
by the following, and its requirements on the seed apply only to a node that
generates its secrets with the seed-based algorithm. The per-commitment secret
of commitment number `n` still corresponds to index `2^48 - 1 - n`, the index
under which the compact representation stores it.

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
  - SHOULD be able to produce the per-commitment secret of any commitment
    number it may have reached, including after being restored from an old
    backup, so that it can check `your_last_per_commitment_secret` when its
    peer reports a later state.

A node receiving per-commitment secrets:
  - MUST NOT require that the per-commitment secrets it receives are related
    to each other in any way.
  - MUST keep each per-commitment secret it receives, so that it can produce
    the secret of every commitment transaction its peer has revoked, until
    every funding output that such a commitment transaction could spend has
    been spent irrevocably (after a splice there can be more than one) and, if
    a revoked commitment transaction spent one, until every output of that
    transaction, and of any HTLC transaction that spends it, has been
    irrevocably resolved.
  - MAY store per-commitment secrets in the compact representation in
    [Efficient Per-commitment Secret Storage](https://github.com/lightning/bolts/blob/master/03-transactions.md#efficient-per-commitment-secret-storage),
    for as long as each one it receives can be inserted into it:
    - otherwise, MUST store that secret, and every later one, individually,
      together with the number of the commitment transaction it revokes.

The commitment number is still 48 bits, so a channel with
`option_independent_secrets` still has at most 2^48 commitment transactions.

### Taproot channels

`option_independent_secrets` MAY be combined with the simple taproot channel
types of
[bolt-simple-taproot.md](https://github.com/lightning/bolts/blob/master/bolt-simple-taproot.md).
The per-commitment secrets of such a channel follow this bLIP as on any other
channel.

That extension BOLT also recommends deriving each MuSig2 verification nonce
from a second shachain, seeded from the root of the revocation secrets, so
that a node can reproduce its nonces without storing them. That scheme is
local to the node: its peer never sees the secret nonces and does not check
how they were made.

A node whose signing keys are held by several signers:
  - MUST NOT derive its MuSig2 secret nonces from a seed that any single
    signer holds.
  - MUST persist each verification nonce it sends, or whatever its signers
    need to reproduce it, before sending it, as that extension BOLT already
    requires of nonces not made with its counter-based scheme.

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

The cost falls on the receiver of such secrets, which stores a 32-byte secret
per revoked commitment, with its commitment number, instead of a fixed 49
entries. Its storage for the channel therefore grows with every update instead
of staying constant: about 32 MB of secrets for a million updates, plus each
implementation's per-record overhead, bounded only by the 48-bit commitment
number. That is the price of the option, and it is paid only on channels that
use it. A node that cannot afford it leaves the bit unset, which this bLIP
allows (the bit is a SHOULD, not a MUST), and its channels keep the compact
form; a node that sets it can still close a channel whose storage grows too
large. Some implementations already keep per-commitment data for every revoked
state in order to punish a breach (lnd's revocation log, for example), and for
them the secret adds about 38 bytes (32 plus a TLV header) to a record that
exists anyway.

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

Feature bits below 256 are reserved for the BOLTs, so this one is 266/267. A
feature vector that sets it is 34 bytes long, where the BOLT channel types in
use today fit in 7, or 11 for a simple taproot channel. That cost is paid once
per channel in `channel_type`, and in `init` and `node_announcement` as for any
other bLIP feature bit. If every node comes to need the option, moving it into
the BOLTs would also give it a bit below 256.

Taproot channels need no change to the messages, because a node's secret nonces
are never seen or checked by its peer. They do need care inside a multi-signer
node. A signer that knows another signer's secret nonce, and sees that signer's
partial signature while the signatures are combined, can compute that signer's
key share. Nonces derived from a seed that one signer holds would therefore
hand that signer the other shares, which is the same weakness the shachain has
for revocation secrets. Each signer has to make its own nonces, and the node
has to keep them, since it can no longer regenerate them from a seed.

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

A node restored from a static channel backup can produce only the secrets that
backup holds. If it holds the compact representation, the node can still punish
every state a seed-based peer revoked, but none whose secret it had to store
individually: against a peer that does not use the seed-based algorithm, it
can no longer punish a breach. Implementations should make this clear to their
users.

## Reference Implementations

Both are forks for the Bitcoin BLAKE2b chain; nothing in the change depends on
that chain. They interoperate with each other.

- lnd (Lightning Fork): `independent-secrets-blip` branch of
  https://github.com/davelund110/lightning-fork
- Core Lightning (privkeyio fork): `independent-secrets-blip` branch of
  https://github.com/davelund110/lightning
