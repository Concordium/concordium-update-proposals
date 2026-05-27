.. _CIS-8:

=====================================
CIS-8: External Key Registry Standard
=====================================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 21, 2026
   * - Draft
     - May 25, 2026
   * - Supported versions
     - | Smart contract version 1 (concordium-std v10+)
   * - Standard identifier
     - ``CIS-8``
   * - Requires
     - :ref:`CIS-0<CIS-0>`


Abstract
========

A standard interface for a smart contract that records cryptographically-proven
bindings between Concordium accounts and external blockchain public keys.

The contract supports four proof schemes that cover the major chains targeted
by Concordium integrations: Ethereum and EVM-compatible L2s
(``ethereum-personal-sign``), Solana (``solana-ed25519``), Cosmos-SDK chains
including Fetch.ai accounts (``cosmos-secp256k1``), and Fetch.ai uagent
identities (``fetch-ai-ed25519``).

CIS-8 is the cryptographic-proof companion to :ref:`CIS-8004`. CIS-8004 agents
MAY point to a CIS-8 entry via their ``external_reference`` field. CIS-8 has
no awareness of CIS-8004 — the reference is one-way.


Introduction
============

CIS-8 records a single fact per entry: a Concordium account controls the
private key corresponding to a named external public key. Each external key
is uniquely identified by the triple ``(namespace, key_type, public_key)``.
The contract verifies an off-chain cryptographic signature over a canonical
domain-separated message and, on success, records the binding.

The contract maintains only the current state of each registration. Full
history is recoverable from the emitted events.

Relationship to Other CIS Standards
-----------------------------------

- :ref:`CIS-0` — CIS-8 contracts MUST implement standard detection and
  advertise support for ``CIS-0`` and ``CIS-8``.
- :ref:`CIS-8004` — Agent Registry contracts MAY consume CIS-8 entries via
  cross-contract ``ownerOfKey`` calls when an agent declares a
  cryptographic-key external reference.

CIS-8 does not depend on any other standard.


Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in :rfc:`2119`.

Common Types
------------

.. _CIS-8-SerialisationConventions:

Serialisation conventions
^^^^^^^^^^^^^^^^^^^^^^^^^

All multi-byte integers are little-endian. The wire format of each type
below is the concatenation of its fields in the order shown, with the
following length-prefix rules for variable-length fields:

- ``String`` fields carry a ``u16`` little-endian byte-length prefix
  followed by the UTF-8 bytes.
- ``Bytes`` (raw ``Vec<u8>``) fields carry a ``u32`` little-endian
  byte-length prefix followed by the raw bytes.
- ``Vec<T>`` fields (any non-byte element type) carry a ``u32``
  little-endian element-count prefix followed by that many serialised
  ``T`` values.
- ``enum`` variants are serialised as a single ``u8`` discriminant
  (``0`` = first declared variant, ``1`` = next, etc.) followed by the
  variant's fields (if any) in declaration order.
- Fixed-size types: ``AccountAddress`` is 32 bytes, ``ContractAddress``
  is 16 bytes (``index: u64 LE`` ‖ ``subindex: u64 LE``),
  ``HashSha2256`` is 32 bytes, ``Timestamp`` is 8 bytes
  (``u64`` little-endian, milliseconds since UNIX epoch).

These are the same conventions ``concordium-std``'s default ``Serial``
derive produces. A self-delimiting encoding is intentional: implementations
MAY parse messages without an out-of-band schema.

.. _CIS-8-ExternalKeyId:

ExternalKeyId
^^^^^^^^^^^^^

A triple identifying an external public key.

.. list-table::
   :header-rows: 1
   :widths: 18 18 12 52

   * - Field
     - Type
     - Length
     - Description
   * - ``namespace_len``
     - u16 LE
     - 2
     - Byte length of ``namespace`` (UTF-8). MUST NOT exceed 128.
   * - ``namespace``
     - bytes
     - variable
     - CAIP-style chain namespace (e.g. ``"eip155:1"``,
       ``"solana:mainnet"``, ``"cosmos:fetchhub-4"``). MUST be non-empty.
   * - ``key_type_len``
     - u16 LE
     - 2
     - Byte length of ``key_type`` (UTF-8). MUST NOT exceed 64.
   * - ``key_type``
     - bytes
     - variable
     - Key-type identifier (e.g. ``"secp256k1-compressed"``,
       ``"secp256k1-uncompressed"``, ``"ed25519"``). MUST be non-empty.
   * - ``public_key_len``
     - u32 LE
     - 4
     - Byte length of ``public_key``. MUST match the size rule below.
   * - ``public_key``
     - bytes
     - variable
     - Raw external public-key bytes (NOT base58/hex encoded).

The ``public_key`` length MUST match the declared ``key_type``:

.. list-table::
   :header-rows: 1

   * - ``key_type``
     - Required ``public_key`` length
   * - ``secp256k1-compressed``
     - 33 bytes
   * - ``secp256k1-uncompressed``
     - 65 bytes
   * - ``ed25519``
     - 32 bytes

Implementations MAY support additional ``key_type`` values; if so, they MUST
document the required ``public_key`` length for each.

.. _CIS-8-Proof:

Proof
^^^^^

A signature plus the scheme used to produce it.

.. list-table::
   :header-rows: 1
   :widths: 18 18 12 52

   * - Field
     - Type
     - Length
     - Description
   * - ``scheme_len``
     - u16 LE
     - 2
     - Byte length of ``scheme`` (UTF-8).
   * - ``scheme``
     - bytes
     - variable
     - Proof scheme identifier. One of: ``"ethereum-personal-sign"``,
       ``"solana-ed25519"``, ``"cosmos-secp256k1"``,
       ``"fetch-ai-ed25519"``.
   * - ``signature_len``
     - u32 LE
     - 4
     - Byte length of ``signature``.
   * - ``signature``
     - bytes
     - variable
     - Raw signature bytes produced by signing the
       :ref:`CIS-8-CanonicalMessage` with the private key corresponding
       to ``external_key.public_key``. Length depends on the scheme
       (see :ref:`CIS-8-Scheme-EthereumPersonalSign` etc.).

The contract MUST reject any ``Proof`` whose ``scheme`` is not in the supported
set with reject reason :ref:`CIS-8-UnsupportedProofScheme`.

.. _CIS-8-MetadataEntry:

MetadataEntry
^^^^^^^^^^^^^

A single key/value metadata pair attached to a registration.

.. list-table::
   :header-rows: 1
   :widths: 18 18 12 52

   * - Field
     - Type
     - Length
     - Description
   * - ``key_len``
     - u16 LE
     - 2
     - Byte length of ``key`` (UTF-8). MUST be non-zero and MUST NOT
       exceed 64.
   * - ``key``
     - bytes
     - variable
     - UTF-8 bytes.
   * - ``value_len``
     - u16 LE
     - 2
     - Byte length of ``value`` (UTF-8). MUST NOT exceed 512.
   * - ``value``
     - bytes
     - variable
     - UTF-8 bytes.

A registration MAY carry up to 32 ``MetadataEntry`` items (see the
``Vec<MetadataEntry>`` element-count limit in :ref:`CIS-8-Registration`).
No keys are reserved at the standard level; applications define their
own conventions.

.. _CIS-8-RegistrationStatus:

RegistrationStatus
^^^^^^^^^^^^^^^^^^

A single-byte enum tag.

.. list-table::
   :header-rows: 1
   :widths: 18 18 64

   * - Tag
     - Variant
     - Meaning
   * - ``0x00``
     - ``Active``
     - The registration is the current authoritative binding for its
       ``ExternalKeyId``.
   * - ``0x01``
     - ``Revoked``
     - The registration has been revoked (by its owner) or replaced (by
       a new owner) and is no longer authoritative.

.. _CIS-8-Registration:

Registration
^^^^^^^^^^^^

The full record returned for a registered external key. Only the
``scheme`` of the proof presented at registration is retained; the
``signature`` itself is consumed at register time and not stored. This
keeps state minimal — consumers that want post-hoc signature
verification SHOULD watch :ref:`CIS-8-ExternalKeyRegistered` events,
which carry the same ``proof_scheme`` field.

.. list-table::
   :header-rows: 1
   :widths: 18 22 12 48

   * - Field
     - Type
     - Length
     - Description
   * - ``owner``
     - AccountAddress
     - 32
     - The Concordium account that last proved control of the external
       key.
   * - ``external_key``
     - ExternalKeyId
     - variable
     - The external key — serialised per :ref:`CIS-8-ExternalKeyId`.
   * - ``proof_scheme_len``
     - u16 LE
     - 2
     - Byte length of ``proof_scheme`` (UTF-8).
   * - ``proof_scheme``
     - bytes
     - variable
     - Proof scheme identifier used at the latest successful
       ``register`` call. Same string set as :ref:`CIS-8-Proof`.
   * - ``metadata_count``
     - u32 LE
     - 4
     - Number of ``MetadataEntry`` items. MUST NOT exceed 32.
   * - ``metadata``
     - MetadataEntry[]
     - variable
     - Zero or more :ref:`CIS-8-MetadataEntry` records, concatenated.
   * - ``status``
     - RegistrationStatus
     - 1
     - ``0x00`` Active or ``0x01`` Revoked
       (see :ref:`CIS-8-RegistrationStatus`).
   * - ``last_updated``
     - Timestamp
     - 8
     - ``u64`` LE milliseconds since UNIX epoch — slot time of the
       block containing the most recent state-modifying transaction for
       this registration.


By design only the current state needs to be stored. Historical
transitions MUST be recoverable from emitted events.


Canonical Signed Message
------------------------

.. _CIS-8-CanonicalMessage:

CIS-8 verifies that the holder of the external private key has authorised the
binding to a specific Concordium account on a specific contract on a specific
network. The contract reconstructs the canonical message; it MUST NEVER accept
the message bytes as a parameter.

The signed message is the concatenation, in the order shown, of the following
fields. All integer widths are little-endian. All ``String`` and ``Bytes``
fields carry an explicit ``u16`` little-endian length prefix so the encoding
is unambiguous and self-delimiting.

.. list-table::
   :header-rows: 1
   :widths: 18 18 12 52

   * - Field
     - Type
     - Length
     - Description
   * - ``prefix``
     - bytes
     - 18
     - The 18-byte UTF-8 literal ``"CIS-8/v1/canonical"``. Domain-separation
       tag; prevents signatures produced for any other purpose from being
       replayed against CIS-8.
   * - ``concordium_account``
     - AccountAddress
     - 32
     - Raw 32-byte account address. Set by the contract to
       ``ctx.sender()`` (transactions from contracts MUST be rejected with
       :ref:`CIS-8-Unauthorized`).
   * - ``contract_address``
     - ContractAddress
     - 16
     - ``index`` (``u64`` LE, 8 bytes) followed by ``subindex`` (``u64`` LE,
       8 bytes). Pins the signature to a single contract instance.
   * - ``concordium_genesis_hash``
     - HashSha2256
     - 32
     - The chain's genesis hash. The contract MUST retrieve this from the
       receive-context metadata; it MUST NOT trust a value supplied as a
       parameter. Pins the signature to a single network.
   * - ``external_namespace_len``
     - u16 LE
     - 2
     - Byte length of ``external_namespace`` (UTF-8). MUST equal
       ``external_key.namespace`` (see below).
   * - ``external_namespace``
     - bytes
     - variable
     - UTF-8 bytes of the external chain namespace.
   * - ``external_key``
     - ExternalKeyId
     - variable
     - Serialised per :ref:`CIS-8-ExternalKeyId`.
   * - ``proof_scheme_len``
     - u16 LE
     - 2
     - Byte length of ``proof_scheme`` (UTF-8).
   * - ``proof_scheme``
     - bytes
     - variable
     - UTF-8 bytes of the proof scheme identifier
       (e.g. ``"ethereum-personal-sign"``).

Proof Verification
------------------

The verifier dispatches on ``proof.scheme``. Each scheme defines how the
``CanonicalMessage::to_signed_bytes()`` payload is wrapped before hashing and
which signature primitive verifies the result.

.. _CIS-8-Scheme-EthereumPersonalSign:

ethereum-personal-sign
^^^^^^^^^^^^^^^^^^^^^^

| Algorithm: ECDSA over secp256k1
| Hash: Keccak-256
| Public key encoding: ``secp256k1-compressed`` (33 bytes) or ``secp256k1-uncompressed`` (65 bytes)
| Signature format: 65 bytes ``r (32) || s (32) || v (1)``

Verification:

1. Build the personal-sign wrapper:
   ``prefixed = b"\x19Ethereum Signed Message:\n" + ascii(len(message)) + message``
2. Compute ``digest = keccak256(prefixed)``.
3. Recover the secp256k1 public key from ``signature`` and ``digest``.
4. Compare the recovered key against ``external_key.public_key``. If
   ``key_type = "secp256k1-compressed"``, compress the recovered key before
   comparison.
5. Reject with :ref:`CIS-8-InvalidProof` on mismatch.

.. _CIS-8-Scheme-SolanaEd25519:

solana-ed25519
^^^^^^^^^^^^^^

| Algorithm: Ed25519
| Public key encoding: ``ed25519`` (32 bytes)
| Signature format: 64 bytes ``R || S``

Verification:

1. Use ``message`` directly as the signed payload (no additional wrapping).
2. Verify the Ed25519 signature against ``external_key.public_key``.
3. Reject with :ref:`CIS-8-InvalidProof` on failure.

.. _CIS-8-Scheme-CosmosSecp256k1:

cosmos-secp256k1
^^^^^^^^^^^^^^^^

For Cosmos-SDK chains including Fetch.ai accounts.

| Algorithm: ECDSA over secp256k1
| Hash: SHA-256
| Public key encoding: ``secp256k1-compressed`` (33 bytes)
| Signature format: 64 bytes ``r || s`` (no recovery byte)

Verification:

1. Wrap ``message`` in the `ADR-036 off-chain signing envelope
   <https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-036-arbitrary-signature.md>`_.
   The exact envelope shape MUST be fixed by the deploying implementation
   and documented in a contract-instance README; off-chain signers and the
   on-chain verifier MUST agree byte-for-byte.
2. Compute ``digest = sha256(envelope)``.
3. Verify the ECDSA signature against ``digest`` using
   ``external_key.public_key``.
4. Reject with :ref:`CIS-8-InvalidProof` on failure.

.. note::

   This standard does not mandate the ADR-036 envelope bytes; it requires
   only that the deployed implementation pin a choice and document it.
   Implementations that need cross-implementation interoperability SHOULD
   coordinate on a single canonical envelope. See ADR-036_ for the
   normative envelope shape.

.. _ADR-036: https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-036-arbitrary-signature.md

.. _CIS-8-Scheme-FetchAiEd25519:

fetch-ai-ed25519
^^^^^^^^^^^^^^^^

For Fetch.ai uagent framework identities (distinct from Cosmos account keys).

| Algorithm: Ed25519
| Public key encoding: ``ed25519`` (32 bytes)
| Signature format: 64 bytes

Verification: identical to :ref:`CIS-8-Scheme-SolanaEd25519`.

.. note::

   uagent addresses are derived as ``agent1`` + bech32(public_key). The
   bech32 form is informational; the contract verifies against the raw
   public key bytes only.


Entrypoints
-----------

.. _CIS-8-register:

register
^^^^^^^^

Register a new active owner for an external key, proving control via a
cryptographic signature over the :ref:`CIS-8-CanonicalMessage`.

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct RegisterParams {
        pub external_key: ExternalKeyId,
        pub proof:        Proof,
        pub metadata:     Vec<MetadataEntry>,
    }

Wire format: ``external_key`` (:ref:`CIS-8-ExternalKeyId`) ‖
``proof`` (:ref:`CIS-8-Proof`) ‖ ``metadata_count`` (``u32`` LE) ‖
zero or more :ref:`CIS-8-MetadataEntry` records concatenated.

**Authorisation.** The transaction sender MUST be an account (not a
contract); contract callers MUST be rejected with
:ref:`CIS-8-Unauthorized`. The ``concordium_account`` field of the
reconstructed canonical message is always equal to ``ctx.sender()``.

**Validation.** The contract MUST:

1. Validate ``external_key`` per :ref:`CIS-8-ExternalKeyId`.
2. Reject any ``proof.scheme`` not in the supported set with
   :ref:`CIS-8-UnsupportedProofScheme`.
3. Validate ``metadata`` per :ref:`CIS-8-MetadataEntry`.
4. Reconstruct the :ref:`CIS-8-CanonicalMessage` and verify the proof
   per the appropriate scheme; reject with :ref:`CIS-8-InvalidProof`
   on failure.

**Effect.** The contract MUST then apply the replacement rule:

a. If no existing registration for the ``ExternalKeyId``: insert a new
   ``Active`` ``Registration`` and emit
   :ref:`CIS-8-ExternalKeyRegistered`.
b. If an existing ``Active`` registration is owned by the sender:
   replace it in place (same owner re-registering) and emit
   :ref:`CIS-8-ExternalKeyRegistered`.
c. If an existing ``Active`` registration is owned by a different
   account: emit :ref:`CIS-8-ExternalKeyRevoked` (for the previous
   owner) first, then replace the registration and emit
   :ref:`CIS-8-ExternalKeyRegistered` (for the new owner). Events MUST
   be emitted in this order.

.. _CIS-8-updateMetadata:

updateMetadata
^^^^^^^^^^^^^^

Update the metadata attached to an existing active registration.

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct UpdateMetadataParams {
        pub identifier: ExternalKeyId,
        pub metadata:   Vec<MetadataEntry>,
    }

Wire format: ``identifier`` (:ref:`CIS-8-ExternalKeyId`) ‖
``metadata_count`` (``u32`` LE) ‖ zero or more
:ref:`CIS-8-MetadataEntry` records concatenated.

**Authorisation.** Sender MUST be the active owner of the resolved
registration; otherwise reject with :ref:`CIS-8-Unauthorized`.

**Effect.** Replace ``registration.metadata`` in place; update
``last_updated``. Emit :ref:`CIS-8-UpdateMetadata`.

Reject with :ref:`CIS-8-NotRegistered` if no active registration exists
for the identifier.

.. _CIS-8-revoke:

revoke
^^^^^^

Revoke the caller's active registration of an external key.

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct RevokeParams {
        pub identifier: ExternalKeyId,
    }

Wire format: ``identifier`` (:ref:`CIS-8-ExternalKeyId`).

**Authorisation.** Sender MUST be the active owner; otherwise reject
with :ref:`CIS-8-Unauthorized`.

**Effect.** Set ``status = Revoked``; update ``last_updated``. Emit
:ref:`CIS-8-ExternalKeyRevoked`.

.. _CIS-8-ownerOfKey:

ownerOfKey
^^^^^^^^^^

A read-only view returning the current :ref:`CIS-8-Registration` for the
given ``ExternalKeyId``, or ``None`` if no entry exists.

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct OwnerOfKeyParams {
        pub external_key: ExternalKeyId,
    }

    #[derive(Serialize, SchemaType)]
    pub enum OwnerQueryResult {
        None,
        Some(Registration),
    }

Parameter wire format: ``external_key`` (:ref:`CIS-8-ExternalKeyId`).

Return wire format: a single ``u8`` discriminant — ``0x00`` for ``None``
(no further bytes), ``0x01`` for ``Some`` followed by a serialised
:ref:`CIS-8-Registration`.

This entrypoint is the integration point used by :ref:`CIS-8004` cross-
contract verification (see :ref:`CIS-8004-CrossContractVerification`).

.. _CIS-8-supports:

supports
^^^^^^^^

A CIS-8 contract MUST implement :ref:`CIS-0` standard detection and MUST
return ``Support`` for both ``CIS-0`` and ``CIS-8``.


Events
------

A CIS-8 contract MUST emit the events defined in this section, each prefixed
with its tag byte. The custom ``Serial`` implementation writes the tag byte
first, then the typed payload.

.. list-table::
   :header-rows: 1

   * - Event
     - Tag
   * - :ref:`CIS-8-ExternalKeyRegistered`
     - 231
   * - :ref:`CIS-8-ExternalKeyRevoked`
     - 232
   * - :ref:`CIS-8-UpdateMetadata`
     - 233

These tag values intentionally avoid the 251..255 range reserved by
:ref:`CIS-2`, so a single contract MAY implement both CIS-8 and CIS-2 without
event-tag collisions.

.. _CIS-8-ExternalKeyRegistered:

ExternalKeyRegistered
^^^^^^^^^^^^^^^^^^^^^

Emitted when a new registration is created, when an existing owner
re-registers, or when a new owner replaces a previous one (in which case it
is preceded by an :ref:`CIS-8-ExternalKeyRevoked` for the previous owner).

.. code-block:: rust

    pub struct ExternalKeyRegisteredEvent {
        pub owner:        AccountAddress,
        pub external_key: ExternalKeyId,
        pub proof_scheme: String,
        pub metadata:     Vec<MetadataEntry>,
    }

Wire format: ``0xE7`` (tag 231) ‖ ``owner`` (32 bytes ``AccountAddress``) ‖
``external_key`` (:ref:`CIS-8-ExternalKeyId`) ‖ ``proof_scheme_len``
(``u16`` LE) ‖ ``proof_scheme`` (UTF-8 bytes) ‖ ``metadata_count``
(``u32`` LE) ‖ zero or more :ref:`CIS-8-MetadataEntry` records.

.. _CIS-8-ExternalKeyRevoked:

ExternalKeyRevoked
^^^^^^^^^^^^^^^^^^

Emitted when the active owner revokes their own registration, or when a
new owner replaces them via :ref:`CIS-8-register`.

.. code-block:: rust

    pub struct ExternalKeyRevokedEvent {
        pub owner:        AccountAddress,
        pub external_key: ExternalKeyId,
    }

Wire format: ``0xE8`` (tag 232) ‖ ``owner`` (32 bytes ``AccountAddress``)
‖ ``external_key`` (:ref:`CIS-8-ExternalKeyId`).

.. _CIS-8-UpdateMetadata:

UpdateMetadata
^^^^^^^^^^^^^^

Emitted on a successful :ref:`CIS-8-updateMetadata` call.

.. code-block:: rust

    pub struct UpdateMetadataEvent {
        pub external_key: ExternalKeyId,
        pub metadata:     Vec<MetadataEntry>,
    }

Wire format: ``0xE9`` (tag 233) ‖ ``external_key``
(:ref:`CIS-8-ExternalKeyId`) ‖ ``metadata_count`` (``u32`` LE) ‖ zero
or more :ref:`CIS-8-MetadataEntry` records.


Errors
------

Reject codes are mapped explicitly via ``From<Cis8Error> for Reject`` to the
numeric values listed below; implementations MUST NOT rely on derived
sequential numbering.

The ``-7100..`` range was chosen so CIS-8 reject codes do not collide
with other CIS standards (``-42000..`` for CIS-2 etc.) when a single
contract implements multiple standards.

.. list-table::
   :header-rows: 1

   * - Code
     - Name
     - Meaning
   * - .. _CIS-8-InvalidProof:

       -7100
     - ``InvalidProof``
     - Signature does not verify against the reconstructed canonical message.
   * - .. _CIS-8-UnsupportedProofScheme:

       -7101
     - ``UnsupportedProofScheme``
     - ``proof.scheme`` is not in the contract's supported set.
   * - .. _CIS-8-MalformedExternalKey:

       -7102
     - ``MalformedExternalKey``
     - ``external_key`` fails the validation rules in
       :ref:`CIS-8-ExternalKeyId`.
   * - .. _CIS-8-Unauthorized:

       -7103
     - ``Unauthorized``
     - Caller is not authorised — typically: caller is a contract, or caller
       is not the active owner for ``updateMetadata`` / ``revoke``.
   * - -7104
     - ``AlreadyRegistered``
     - Reserved; redundant re-registration MAY use this code, though the
       replacement rule in :ref:`CIS-8-register` instead handles same-owner
       re-registration as an in-place replacement.
   * - .. _CIS-8-NotRegistered:

       -7105
     - ``NotRegistered``
     - No active registration exists for the supplied identifier.
   * - -7106
     - ``AmbiguousIdentifier``
     - Reserved for future use.
   * - -7107
     - ``UnsupportedKeyType``
     - ``external_key.key_type`` is not supported by this contract.
   * - .. _CIS-8-InvalidMetadata:

       -7108
     - ``InvalidMetadata``
     - ``metadata`` fails the validation rules in :ref:`CIS-8-MetadataEntry`.
Administrative concerns — contract upgradeability, ownership transfer of
the contract instance itself, and any associated reject codes — are out
of scope for CIS-8 and left to the implementation. Implementations
SHOULD follow whatever conventions are appropriate for their deployment
context.


Schema Embedding
----------------

A CIS-8 contract module MUST embed its Concordium schema in the compiled
``.wasm.v1`` artifact (``cargo concordium build --schema-embed``). All
parameter, return, error, and event types MUST derive ``SchemaType``. This
ensures wallets and explorers can render contract interactions as structured
JSON.


Reference Deployments
=====================

This section is informative.

The following contract instances implement this specification on
Concordium-operated networks:

.. list-table::
   :header-rows: 1

   * - Network
     - Contract address
     - Notes
   * - Concordium DevNet P-11
     - ``<23, 0>``
     - Concordium-owned redeploy 2026-05-25
   * - Concordium TestNet
     - ``<12794, 0>``
     - Concordium-owned redeploy 2026-05-25; same module bytes as DevNet


Security Considerations
=======================

Domain separation
    The 18-byte ``CIS-8/v1/canonical`` prefix prevents a signature produced
    for any other context from being replayed against CIS-8. The
    ``contract_address`` and ``concordium_genesis_hash`` fields further pin
    the signature to a single instance on a single network.

Replacement semantics
    The replacement rule in :ref:`CIS-8-register` allows a new account to
    take over an external key by presenting a valid signature for it. This
    is the intended semantics: control of the private key is the
    authoritative ground truth. Consumers MUST NOT treat a CIS-8 entry as
    proof of historical ownership — only as proof of current control.


Copyright
=========

This document is placed in the public domain.
