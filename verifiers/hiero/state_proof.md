# Hiero state proofs

The existing HAPI `block/stream/state_proof.proto` is authoritative for StateProof.
It contains repeated Merkle paths and a signed block proof. Paths can be chained
and contain state-item, block-item or timestamp leaves, or explicit hashes, together
with sibling orientation. The signed block proof authenticates the resulting
48-byte block root. This CLPR extension does not redefine that schema or its
Merkle hashing rules.

For a CLPR bundle, the verifier authenticates the Channel leaf and each included
ClprMessageValue leaf to the same signed block root. It binds the channel identity
to the stored channel context, extracts the proven message IDs and running hashes,
and verifies any carried endpoint manifest's service identity. CLPR's service
performs the subsequent queue, replay, acknowledgment and status checks.

The ledger-ID/WRAPS profile verifies block signing authority against the pinned
network identity. The explicit-key profile verifies the hinTS block signature
against the stored VK using the [state-proof arm of HieroProof V1](proof_envelope.md#state-proof-payload).
It must first catch up its key via provenance-only bundles if necessary. Those
bundles contain no StateProof and establish no channel or message facts.

The proposed [HieroProof schema](hiero_proof.proto) imports StateProof rather than
adding CLPR-specific provenance fields inside it. Endpoint producers wrap their
existing StateProof bytes in field 2 and supply `format_version = 1`; key catch-up
uses field 3 instead. Bare legacy StateProof bytes are not accepted by the new
profile. Config-proof handling remains profile-specific; see
[verifier_contract.md](verifier_contract.md).
