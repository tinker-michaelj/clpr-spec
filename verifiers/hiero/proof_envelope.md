# Hiero proof envelope, version 1

This is an opt-in proof profile for a verifier whose trust anchor is a particular
hinTS verification key (VK). It supports authenticated successor authorization
without a recursive WRAPS proof. It does not change the ledger-ID/WRAPS profile.
A channel's verifier determines its profile; receivers MUST NOT infer the profile
from attacker-supplied proof bytes or fall back to another profile on failure.

## Placement and encoding

`ClprBundleContent.bundle_payload` remains opaque and unchanged. Its contents for
this profile are the serialized `HieroProof` in [hiero_proof.proto](hiero_proof.proto).
The envelope belongs beside `StateProof` in HAPI's `block/stream` package, with the
provenance messages in the same schema. This PR specifies the proposed HAPI
addition; publication in HAPI and endpoint construction are separate integrations.
`StateProof`, its signed block proof, and the outer CLPR transport are not redefined.

Version 1 requires `format_version = 1`, followed by exactly one nonempty payload.
The envelope and provenance messages MUST use ascending field order, shortest
uint32 varints, and exactly the fields defined here. Duplicate singular fields,
unknown fields, duplicate/conflicting payload arms, missing required fields,
truncation, overflow, and trailing data MUST be rejected. Repeated links retain
order. These restrictions are stricter than a generic protobuf parser: a generated
`oneof` parser alone does not enforce them. StateProof's existing encoding rules
continue to apply inside the state-proof arm. Future extensions use a new envelope
version; unknown versions MUST fail closed, with no retry as a bare StateProof.

## Key-provenance payload

`initial_verification_key` MUST equal the channel's current raw VK byte for byte.
There MUST be between 1 and 8 links, in forward order. For each link:

1. Validate the successor VK's canonical encoding, group membership, and domain.
2. Verify `signature` under the preceding VK over **exactly the raw bytes** of
   `next_verification_key`. Do not hash, prefix, ABI-encode, or append epoch IDs.
3. Require signers' authenticated weight to be strictly greater than one third of
   the preceding key's total weight (aBFT threshold). Weight at exactly one third
   MUST fail. The aggregate must prove the selected parties' weight, not merely a
   BLS pairing under a caller-supplied aggregate public key.
4. Reject the successor if it equals any VK already in this bundle, including the
   initial VK. Then use it as the authority for the following link.

The V1 crypto suite is hiero-cryptography 3.23.0's hinTS V2: a 1,096-byte uncompressed
VK and a 1,248-byte aggregate signature, with the library's canonical scalar and
point encodings and V2 Fiat–Shamir domains. Other encodings/suites require a new
version. A verifier MUST validate all links before returning any update.

Success returns `has_channel_state = false`, no messages, no manifest, no channel
sync data or message anchor, the final raw VK as `new_trust_anchor`, and its 32-byte
Keccak-256 digest as `new_trust_anchor_id`. The CLPR service commits only these two
authority fields, atomically. The identifier is a cache/routing hint; it is not an
epoch counter, signature target, or independent source of authority.

Endpoints may split longer chains into batches of at most eight links, additionally
bounded by receiver `max_sync_bytes` and execution gas. They MUST begin each batch
at the receiver's actual current VK, and reread it after a competing update. A
stale or already applied batch fails the initial-key comparison. Do not concatenate
state proofs to this payload; submit a state-proof bundle after catch-up.

## Signed authorization is sufficient

Acceptance means **the predecessor signing authority authorized this successor**.
It does not prove that the source network adopted the candidate roster, that an
epoch number advanced, that this is the latest key, or that no competing candidate
was authorized. Signatures collected before roster adoption are sufficient under
this profile. Epoch/construction IDs and the provenance queue's inclusion in source
state are not authenticated by these raw-VK signatures.

Endpoints SHOULD submit the adopted chain observed on the source network for
availability, but this is not an on-chain security predicate. Multiple valid
successor authorizations can form branches; a verifier holding one successor need
not accept a sibling. A source signing authority must not authorize a backward
edge, reuse an old key as a successor, or endorse an unsafe candidate. The service
stores only its current anchor, so the within-bundle cycle check cannot detect
historical cycles across separate transactions. Requiring proof of adoption,
chronological monotonicity, or uniqueness would need additional signed material
or authenticated state evidence and a different proof profile.

## State-proof payload

Verify the unchanged StateProof's Merkle paths and authenticate the 48-byte block
root with hinTS under the **current stored VK**. V1 accepts a block signature of
`VK (1096 bytes) || hinTS signature (1248 bytes)`, or the native 13,712-byte combined
TSS representation whose first 2,344 bytes are that prefix. In the latter form the
remaining 11,368 WRAPS bytes are not authority evidence in this profile; the
prefix VK MUST still match the stored anchor exactly. No arbitrary suffix lengths
are permitted. All required paths must converge to the authenticated block root;
the proven channel identity must match the channel context, and carried endpoint
manifests must match its remote service address.

Success returns `has_channel_state = true` and ordinary proven channel data and
messages. It returns no trust-anchor change. A state-proof bundle with no messages
still requires actual channel-state evidence; it is distinct from provenance-only
absence. Normal CLPR replay, running-hash, ack, progress and status rules apply.

## Compatibility and bootstrap

Select and trust the initial VK out of band when configuring the verifier/channel.
A key supplied only by an untrusted bundle cannot bootstrap itself. A pinned-key
verifier may decode the usual configuration control payload and seed the anchor
from its deployment configuration; decoding that payload does not make it a state
proof. Channel-establishment governance retains responsibility for that config.
An optional config-time manifest proof must use the state-proof envelope and the
pinned initial key.

The shared verifier interface adds an explicit final `has_channel_state` boolean.
All verifiers used by that service must implement it; older return tuples cannot
be silently treated as state-bearing or authority-only. Deployment of the revised
service and matching verifiers must be coordinated. This does not migrate existing
channels from ledger-ID anchors to VK anchors, or change the native CLPR precompile
ABI automatically.

## Required conformance cases

- Five successive native-library-signed links advance E−5 to E; E verifies a later
  state proof. The same chain split into batches yields the same final key.
- Wrong initial key, missing/reordered/forged links, self/cyclic links, exact-one-third
  weight, malformed keys/signatures, >8 links, and stale replay are rejected.
- Unknown versions, legacy bare proofs, conflicting arms, nonminimal varints,
  duplicates, truncation and trailing bytes are rejected without fallback.
- An invalid final link leaves the original anchor unchanged.
- Authority-only updates on ACTIVE, PAUSED, CLOSING and DRAINED channels preserve
  all queue/ack counters and hashes, messages, configuration, manifests, status,
  and application effects. They do not auto-resume or close a channel.
- A false presence flag with nonempty channel data/messages/manifest, an empty
  successor, empty successor ID, or unchanged authority is rejected by the service.
