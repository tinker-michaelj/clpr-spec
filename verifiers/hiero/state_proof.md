# Hiero state proofs

The HAPI `block/stream/state_proof.proto` defines `StateProof`: repeated
`MerklePath paths = 1` and `TssSignedBlockProof signed_block_proof = 2`. A path
contains bottom-up siblings, a `next_path_index`, and at most one content arm:
explicit hash (field 3), state-item bytes (4), block-item bytes (5), or timestamp
bytes (6). A `SiblingNode` has `is_left = 1` and `hash = 2`;
`TssSignedBlockProof.block_signature` is field 1. The proposed
[HieroProof schema](hiero_proof.proto) imports this message rather than redefining it.

## Merkle commitments

HieroProof V1's state-proof arm supports **SHA-384** Merkle commitments and a
48-byte signed block root. Source nodes must use that digest configuration. A
source using SHA-256 is not compatible with this state-proof profile; a different
profile is needed. The raw-VK provenance signatures do not depend on the source
Merkle digest.

The domain bytes are independent of the protobuf field tags:

| Node | Hash input |
|---|---|
| Leaf | `SHA-384(0x00 || exact_leaf_serialization)` |
| Unary internal node | `SHA-384(0x01 || child_hash)` |
| Binary internal node | `SHA-384(0x02 || left_hash || right_hash)` |

A state leaf hashes the complete serialized `StateItem`, including its key (field
2) and value (field 3), without an extra leaf-wrapper tag. A path starting at an
explicit hash uses that hash directly. For a binary step, `is_left = true` places
the sibling before the running hash; false places it after. The CLPR producer's
`MerklePathBuilder` represents a unary step with an empty sibling hash.
`next_path_index = UINT32_MAX` terminates a path; otherwise it identifies the
continuation path. A verifier must reject out-of-range or cyclic references and
must not substitute unauthenticated content for the accumulated hash.

The source CLPR producer builds complete leaf-to-block-root paths from the state
API and extends them with the snapshot's state-to-block siblings and the
`SHA-384(0x00 || serialized_timestamp)` sibling. Paths may share continuation
segments; a receiver must use the source producer's path representation rather
than infer tree height or sibling placement from leaf type. The independent
block-spine proof format does not by itself define CLPR's leaf semantics.

## CLPR leaf semantics

`StateItem.value` contains a `StateValue` oneof. Its protobuf wire tag is
`(field_number << 3) | 2` for these length-delimited values, not a Merkle domain
prefix:

| StateValue field | Wire tag | Value | CLPR meaning |
|---|---|---|---|
| 59 | 474 | `ClprLedgerConfiguration` | Source configuration where a verifier profile uses configuration state proofs |
| 60 | 482 | `ClprChannel` | Channel identity, queue counters/hashes, status, and manifest version |
| 62 | 498 | `ClprMessageValue` | Serialized message payload and running hash after that message |
| 66 | 530 | `ClprEndpointManifest` | Source service identity, manifest version, and endpoints |

The message leaf's `StateItem.key` contains `StateKey.ClprService_I_MESSAGE_QUEUE`
(field 62), whose `ClprMessageKey` names the channel (field 1) and message ID
(field 2). The last included message establishes `anchor_message_id` and
`anchor_running_hash`. Its authenticated running hash lets the CLPR service bind
the supplied contiguous payload prefix to the receiver's existing queue history;
authenticating only arbitrary payload bytes without the queue anchor is
insufficient. A verifier may authenticate each included message leaf separately
or authenticate the final message anchor and rely on the service's running-hash
check for the prefix.

The required channel leaf and final message anchor (when messages are included),
plus any carried manifest, must converge to the same authenticated block root.
The verifier binds the channel identity to its stored context and any manifest
to the context's remote service address. Proven channel counters and status are
returned with the payloads and message anchor; the service applies replay,
running-hash, acknowledgment, progress, and lifecycle checks.

These field bindings are from
[`virtual_map_state.proto` at the source provenance integration](https://github.com/hiero-ledger/hiero-consensus-node/blob/c2b44f010ef9e264b28c30a6521e304a755cbbca/hapi/hedera-protobuf-java-api/src/main/proto/platform/state/virtual_map_state.proto).
The producer and hashing references are
[`ClprStateProofManager`](https://github.com/hiero-ledger/hiero-consensus-node/blob/c2b44f010ef9e264b28c30a6521e304a755cbbca/hedera-node/hedera-clpr-service-impl/src/main/java/com/hedera/node/app/service/clpr/impl/ClprStateProofManager.java),
[`MerklePathBuilder`](https://github.com/hiero-ledger/hiero-consensus-node/blob/c2b44f010ef9e264b28c30a6521e304a755cbbca/hedera-node/hapi-utils/src/main/java/com/hedera/node/app/hapi/utils/blocks/MerklePathBuilder.java),
and [`HashUtils`](https://github.com/hiero-ledger/hiero-consensus-node/blob/c2b44f010ef9e264b28c30a6521e304a755cbbca/hedera-node/hapi-utils/src/main/java/com/hedera/node/app/hapi/utils/blocks/HashUtils.java).
These bindings apply to the new HieroProof V1 profile. The companion legacy
`HieroVerifier` retains its separate historical manifest wire binding; this
extension does not migrate that profile. Implementers must verify bindings against
their selected profile and deployed source version; the generic HAPI StateProof
container alone does not establish CLPR compatibility.

## Authority and configuration

The ledger-ID/WRAPS profile verifies block signing authority against the pinned
network identity. The explicit-key profile instead verifies the hinTS block
signature against the stored VK using the
[state-proof arm of HieroProof V1](proof_envelope.md#state-proof-payload), including
its strictly-greater-than-one-third acceptance threshold. It must first catch up
its key via provenance-only bundles if necessary; those bundles establish no
channel or message facts.

Endpoint producers wrap their StateProof bytes in field 2 of HieroProof and set
`format_version = 1`; key catch-up uses field 3. Bare legacy StateProof bytes are
not accepted by the new profile. The pinned-key Solidity verifier decodes a
configuration control payload at setup rather than proving a configuration leaf;
channel-establishment governance authenticates that configuration. Its optional
manifest proof must verify under the deployment's initial key. See
[verifier_contract.md](verifier_contract.md).
