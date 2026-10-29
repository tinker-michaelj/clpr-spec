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
