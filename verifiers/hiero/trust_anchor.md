# Hiero trust anchors

Hiero supports distinct trust profiles. A channel's selected verifier fixes the
profile; anchors from one profile cannot be interpreted as the other.

## Ledger identity and WRAPS

The genesis network ID (`ledger_id`) commits to the genesis address book and hinTS
VK. Recursive WRAPS proofs authenticate later network IDs and keys against that
fixed identity. Normal roster rotation does not replace the ledger ID. An
implementation may cache a verified WRAPS result in its opaque anchor, but its
ultimate authority remains the pinned ledger ID.

## Explicit hinTS key provenance

The [version-one proof profile](proof_envelope.md) instead starts with an explicitly
trusted raw hinTS VK. Each valid predecessor-key signature over a successor's raw
VK bytes authorizes the next key. Catch-up updates the stored anchor to the last
verified VK, and `trust_anchor_id` is its Keccak-256 digest. The service stores no
intermediate keys or unauthenticated epoch numbers.

A provenance signature establishes authorization, not roster adoption. Competing
candidate endorsements, historical key reuse and availability consequences are
specified in [Signed authorization is sufficient](proof_envelope.md#signed-authorization-is-sufficient).
Only a verifier result may advance the stored authority after channel setup.
Because submission is permissionless, endpoint adherence to the adopted queue
cannot prevent a third party from installing another validly authorized candidate.
There is no admin re-anchor; a stranded channel's unacknowledged obligations may
also prevent closure. Recovery for future traffic requires a new channel.

The verifier checks canonical key encoding, group membership, and the supported
domain shape. It does not independently prove consistency between the domain and
`Z`, between total weight and `W`, or the CRS construction/trapdoor assumptions.
For a successor, these properties are trusted through the predecessor authority's
signature on the exact bytes; for the initial pin they must be established out
of band. Neither a valid point encoding nor a key digest establishes them.

Bootstrap keys/ledger IDs must be authenticated independently of untrusted proof
bytes. Changing a channel from the ledger-ID profile to the key profile is a trust
policy change and is not performed by this extension.
