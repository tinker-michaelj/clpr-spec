# Explicit Hiero key provenance and absent channel state

Status: proposed.

A peer whose trusted hinTS key is several rotations behind needs to advance trust
before it can verify a current block signature. The CN provenance queue supplies
predecessor signatures over successor VK bytes; these signatures do not include a
channel-state proof or prove adoption of the candidate roster.

We select signed successor authorization as sufficient for this opt-in verifier
profile. Requiring adoption would require additional evidence beyond those
signatures. The decision deliberately accepts the candidate-branch and key-reuse
consequences documented in the normative [HieroProof V1 profile](../verifiers/hiero/proof_envelope.md).
This is a named exception to CLPR's default requirement for state-proven successor
authority; it does not relax the other verifier profiles.

Submission is permissionless: an unrelated party may submit a valid authorization
published in a proof vote before adoption, despite endpoints following the adopted
provenance queue. Selecting an unadopted or abandoned branch can irreversibly strand
the receiver. No administrative re-anchor or force-close mechanism is introduced.
Unacknowledged outbound messages and required responses prevent normal drain/close
progress when no acceptable peer proof can arrive; message accounting may remain
locked. A new channel pinned to a recent authenticated key restores future traffic,
not settlement of the old channel's obligations.

Both provenance and block-state signatures use strictly greater than one third
of the pinned key's weight. This relies on fewer than one third Byzantine weight
and honest signers endorsing only valid successors/block roots; it does not assert
that the network's signature collection threshold is also one third.

A versioned HieroProof envelope selects unchanged StateProof bytes or a bounded
key-provenance chain. The envelope is specified beside StateProof in HAPI rather
than in the ledger-neutral CLPR transport or in the consensus node's state queue.
This repo owns the protocol definition; the proposed proto is ready for a separate
HAPI publication. The existing WRAPS/ledger-ID verifier remains a separate profile.

The common verifier result gains `has_channel_state`. Inferring absence from zero
counters is unsafe: zero is legitimate proven state, and processing synthetic
zero counters against a nonempty channel can trigger replay, ack, or lifecycle
rules. A false flag requires an otherwise empty result and a genuine authority
change. The service writes only the anchor and ID, returning before channel
processing. A state-bearing empty-message bundle still follows the normal path.

This is a coordinated verifier/service ABI revision, with no shared storage layout
change. No old-ABI fallback, automatic anchor-profile migration, or claim of
roster adoption is introduced. Endpoints must implement the envelope, retain
rotation evidence, and submit state proofs separately after catch-up.
