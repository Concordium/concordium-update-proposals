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

General types and serialization
-------------------------------

.. note::

  Integers are encoded in little-endian unless stated otherwise.

  Variable-length ``String`` fields carry a 2-byte little-endian length
  prefix followed by the UTF-8 bytes. Variable-length byte sequences and
  vectors carry a 4-byte little-endian length-or-count prefix followed
  by the elements. This matches the default ``Serial`` derive produced
  by ``concordium-std``.

  Fixed-size types referenced below have the following widths:

  - ``AccountAddress`` = 32 bytes
  - ``ContractAddress`` = 16 bytes (``index: Byte⁸`` ‖ ``subindex: Byte⁸``)
  - ``HashSha2256`` = 32 bytes
  - ``Timestamp`` = ``u64`` little-endian = 8 bytes (milliseconds since
    UNIX epoch)

.. _CIS-8-ExternalKeyId:

``ExternalKeyId``
^^^^^^^^^^^^^^^^^

A triple identifying an external public key.

It is serialized as: 2 bytes for the length of the namespace (``n_ns``)
followed by ``n_ns`` UTF-8 bytes (``namespace``), then 2 bytes for the
length of the key type (``n_kt``) followed by ``n_kt`` UTF-8 bytes
(``key_type``), then 4 bytes for the length of the public key (``n_pk``)
followed by ``n_pk`` raw bytes (``public_key``)::

  ExternalKeyId ::= (n_ns: Byte²) (namespace: Byteⁿ_ⁿˢ)
                    (n_kt: Byte²) (key_type: Byteⁿ_ᵏᵗ)
                    (n_pk: Byte⁴) (public_key: Byteⁿ_ᵖᵏ)

The ``namespace`` MUST be non-empty and MUST NOT exceed 128 bytes. It is
a CAIP-style chain namespace (e.g. ``"eip155:1"``, ``"solana:mainnet"``,
``"cosmos:fetchhub-4"``).

The ``key_type`` MUST be non-empty and MUST NOT exceed 64 bytes. The
``public_key`` length MUST match the declared ``key_type``:

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

Implementations MAY support additional ``key_type`` values; if so, they
MUST document the required ``public_key`` length for each.

.. _CIS-8-Proof:

``Proof``
^^^^^^^^^

A signature plus the scheme used to produce it.

It is serialized as: 2 bytes for the length of the scheme (``n_s``)
followed by ``n_s`` UTF-8 bytes (``scheme``), then 4 bytes for the length
of the signature (``n_sig``) followed by ``n_sig`` raw bytes
(``signature``)::

  Proof ::= (n_s: Byte²) (scheme: Byteⁿ_ˢ) (n_sig: Byte⁴) (signature: Byteⁿ_ˢⁱᵍ)

The ``scheme`` identifier MUST be one of ``"ethereum-personal-sign"``,
``"solana-ed25519"``, ``"cosmos-secp256k1"``, ``"fetch-ai-ed25519"``.
The contract MUST reject any ``Proof`` whose ``scheme`` is not in the
supported set with reject reason :ref:`CIS-8-UnsupportedProofScheme`.

The ``signature`` is the raw signature bytes produced by signing the
:ref:`CIS-8-CanonicalMessage` with the private key corresponding to
``external_key.public_key``. Its length depends on the scheme; see the
scheme-specific subsections under :ref:`CIS-8-ProofVerification`.

.. _CIS-8-MetadataEntry:

``MetadataEntry``
^^^^^^^^^^^^^^^^^

A single key/value metadata pair attached to a registration.

It is serialized as: 2 bytes for the length of the key (``n_k``)
followed by ``n_k`` UTF-8 bytes (``key``), then 2 bytes for the length
of the value (``n_v``) followed by ``n_v`` UTF-8 bytes (``value``)::

  MetadataEntry ::= (n_k: Byte²) (key: Byteⁿ_ᵏ) (n_v: Byte²) (value: Byteⁿ_ᵛ)

The ``key`` MUST be non-empty and MUST NOT exceed 64 bytes. The
``value`` MUST NOT exceed 512 bytes. No keys are reserved at the standard
level; applications define their own conventions.

.. _CIS-8-RegistrationStatus:

``RegistrationStatus``
^^^^^^^^^^^^^^^^^^^^^^

A single-byte enum tag::

  RegistrationStatus ::= (0: Byte)   // Active
                       | (1: Byte)   // Revoked

``Active`` indicates the registration is the current authoritative binding
for its ``ExternalKeyId``. ``Revoked`` indicates the registration has
been revoked (by its owner) or replaced (by a new owner) and is no longer
authoritative.

.. _CIS-8-Registration:

``Registration``
^^^^^^^^^^^^^^^^

The full record returned for a registered external key. Only the
``scheme`` of the proof presented at registration is retained; the
``signature`` itself is consumed at register time and not stored. This
keeps state minimal — consumers that want post-hoc signature
verification SHOULD watch :ref:`CIS-8-ExternalKeyRegistered` events,
which carry the same ``proof_scheme`` field.

It is serialized as: 32 bytes for the ``owner`` ``AccountAddress``,
followed by a serialised :ref:`CIS-8-ExternalKeyId` (``external_key``),
then 2 bytes for the length of the proof scheme (``n_ps``) followed by
``n_ps`` UTF-8 bytes (``proof_scheme``), then 4 bytes for the number of
metadata entries (``n_m``) followed by ``n_m`` serialised
:ref:`CIS-8-MetadataEntry`\ s (``metadata``), then a single
:ref:`CIS-8-RegistrationStatus` byte (``status``), then 8 bytes for the
``last_updated`` ``Timestamp``::

  Registration ::= (owner: Byte³²)
                   (external_key: ExternalKeyId)
                   (n_ps: Byte²) (proof_scheme: Byteⁿ_ᵖˢ)
                   (n_m: Byte⁴) (metadata: MetadataEntryⁿ_ᵐ)
                   (status: RegistrationStatus)
                   (last_updated: Byte⁸)

The ``metadata`` element count ``n_m`` MUST NOT exceed 32.

The ``last_updated`` ``Timestamp`` is the ``u64`` little-endian slot
time, in milliseconds since the UNIX epoch, of the block containing the
most recent state-modifying transaction for this registration.

By design only the current state needs to be stored. Historical
transitions MUST be recoverable from emitted events.


.. _CIS-8-CanonicalMessage:

Canonical Signed Message
------------------------

CIS-8 verifies that the holder of the external private key has authorised the
binding to a specific Concordium account on a specific contract on a specific
network. The contract reconstructs the canonical message; it MUST NEVER accept
the message bytes as a parameter.

The signed message is serialized as: the 18-byte UTF-8 literal
``"CIS-8/v1/canonical"`` (``prefix``); followed by the 32-byte
``AccountAddress`` of the bound account (``concordium_account``);
followed by the 16-byte ``ContractAddress`` of the registry contract
(``contract_address``); followed by the 32-byte ``HashSha2256`` chain
genesis (``concordium_genesis_hash``); followed by 2 bytes for the
namespace length (``n_ens``) and ``n_ens`` UTF-8 bytes
(``external_namespace``); followed by a serialised :ref:`CIS-8-ExternalKeyId`
(``external_key``); followed by 2 bytes for the proof scheme length
(``n_ps``) and ``n_ps`` UTF-8 bytes (``proof_scheme``)::

  CanonicalMessage ::= (prefix: Byte¹⁸)                              // "CIS-8/v1/canonical"
                       (concordium_account: Byte³²)
                       (contract_address: Byte¹⁶)                    // index ‖ subindex, each u64 LE
                       (concordium_genesis_hash: Byte³²)
                       (n_ens: Byte²) (external_namespace: Byteⁿ_ᵉⁿˢ)
                       (external_key: ExternalKeyId)
                       (n_ps: Byte²)  (proof_scheme: Byteⁿ_ᵖˢ)

The 18-byte UTF-8 prefix ``"CIS-8/v1/canonical"`` provides domain
separation: signatures produced for any other purpose cannot be replayed
against CIS-8. The ``contract_address`` and ``concordium_genesis_hash``
fields pin the signature to a single contract instance on a single
network.

The contract MUST retrieve ``concordium_genesis_hash`` from the receive
context's metadata; it MUST NOT trust a genesis hash supplied as a
parameter.

The contract MUST set ``concordium_account = ctx.sender()`` (an
``AccountAddress``); calls from contract senders MUST be rejected with
:ref:`CIS-8-Unauthorized`.

The ``external_namespace`` field MUST equal ``external_key.namespace``;
it is included as a separate field so that the signed bytes commit to
the namespace independently of the ExternalKeyId framing.

.. _CIS-8-ProofVerification:

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


Contract functions
------------------

A smart contract implementing CIS-8 MUST export the following functions:
:ref:`CIS-8-register`, :ref:`CIS-8-updateMetadata`, :ref:`CIS-8-revoke`,
:ref:`CIS-8-ownerOfKey`, and the :ref:`CIS-0` :ref:`CIS-8-supports`
standard-detection function.

.. _CIS-8-register:

``register``
^^^^^^^^^^^^

Register a new active owner for an external key, proving control via a
cryptographic signature over the :ref:`CIS-8-CanonicalMessage`.

.. _CIS-8-RegisterParameter:

Parameter
~~~~~~~~~

A serialised :ref:`CIS-8-ExternalKeyId` (``external_key``), followed by
a serialised :ref:`CIS-8-Proof` (``proof``), followed by 4 bytes for the
number of metadata entries (``n_m``) and ``n_m`` serialised
:ref:`CIS-8-MetadataEntry`\ s (``metadata``)::

  RegisterParameter ::= (external_key: ExternalKeyId)
                        (proof: Proof)
                        (n_m: Byte⁴) (metadata: MetadataEntryⁿ_ᵐ)

Requirements
~~~~~~~~~~~~

- The transaction sender MUST be an account (not a contract). Contract
  callers MUST be rejected with :ref:`CIS-8-Unauthorized`.
- The ``concordium_account`` field of the reconstructed
  :ref:`CIS-8-CanonicalMessage` is always equal to ``ctx.sender()``.
- The contract MUST validate ``external_key`` per
  :ref:`CIS-8-ExternalKeyId` (length limits, ``public_key`` size
  matching ``key_type``).
- The contract MUST reject any ``proof.scheme`` not in the supported
  set with :ref:`CIS-8-UnsupportedProofScheme`.
- The contract MUST validate ``metadata`` per
  :ref:`CIS-8-MetadataEntry`.
- The contract MUST reconstruct the :ref:`CIS-8-CanonicalMessage` and
  verify the proof per the scheme-specific verification under
  :ref:`CIS-8-ProofVerification`; reject with :ref:`CIS-8-InvalidProof`
  on failure.
- The contract MUST then apply the replacement rule:

  a. If no existing registration for the ``ExternalKeyId``: insert a
     new ``Active`` ``Registration`` and emit
     :ref:`CIS-8-ExternalKeyRegistered`.
  b. If an existing ``Active`` registration is owned by the sender:
     replace it in place (same owner re-registering) and emit
     :ref:`CIS-8-ExternalKeyRegistered`.
  c. If an existing ``Active`` registration is owned by a different
     account: emit :ref:`CIS-8-ExternalKeyRevoked` (for the previous
     owner) first, then replace the registration and emit
     :ref:`CIS-8-ExternalKeyRegistered` (for the new owner). Events
     MUST be emitted in this order.

.. _CIS-8-updateMetadata:

``updateMetadata``
^^^^^^^^^^^^^^^^^^

Update the metadata attached to an existing active registration.

.. _CIS-8-UpdateMetadataParameter:

Parameter
~~~~~~~~~

A serialised :ref:`CIS-8-ExternalKeyId` (``identifier``) followed by 4
bytes for the number of metadata entries (``n_m``) and ``n_m``
serialised :ref:`CIS-8-MetadataEntry`\ s (``metadata``)::

  UpdateMetadataParameter ::= (identifier: ExternalKeyId)
                              (n_m: Byte⁴) (metadata: MetadataEntryⁿ_ᵐ)

Requirements
~~~~~~~~~~~~

- The sender MUST be the active owner of the resolved registration;
  otherwise reject with :ref:`CIS-8-Unauthorized`.
- If no active registration exists for the identifier, reject with
  :ref:`CIS-8-NotRegistered`.
- On success, replace ``registration.metadata`` in place; update
  ``last_updated``; emit :ref:`CIS-8-UpdateMetadataEvent`.

.. _CIS-8-revoke:

``revoke``
^^^^^^^^^^

Revoke the caller's active registration of an external key.

.. _CIS-8-RevokeParameter:

Parameter
~~~~~~~~~

A serialised :ref:`CIS-8-ExternalKeyId` (``identifier``)::

  RevokeParameter ::= (identifier: ExternalKeyId)

Requirements
~~~~~~~~~~~~

- The sender MUST be the active owner; otherwise reject with
  :ref:`CIS-8-Unauthorized`.
- If no active registration exists for the identifier, reject with
  :ref:`CIS-8-NotRegistered`.
- On success, set ``status = Revoked``; update ``last_updated``; emit
  :ref:`CIS-8-ExternalKeyRevoked`.

.. _CIS-8-ownerOfKey:

``ownerOfKey``
^^^^^^^^^^^^^^

A read-only view returning the current :ref:`CIS-8-Registration` for the
given ``ExternalKeyId``, or ``None`` if no entry exists.

This entrypoint MAY be used by external contracts to look up the current
registration for a given key.

Parameter
~~~~~~~~~

A serialised :ref:`CIS-8-ExternalKeyId` (``external_key``)::

  OwnerOfKeyParameter ::= (external_key: ExternalKeyId)

Response
~~~~~~~~

A single byte discriminant — ``0`` for the absent case (no further
bytes), or ``1`` followed by a serialised :ref:`CIS-8-Registration`::

  OwnerQueryResult ::= (0: Byte)
                     | (1: Byte) (registration: Registration)

.. _CIS-8-supports:

``supports``
^^^^^^^^^^^^

A CIS-8 contract MUST implement :ref:`CIS-0` standard detection and MUST
return ``Support`` for both ``CIS-0`` and ``CIS-8``.


Logged events
-------------

The events defined by this specification are serialized using one byte
to discriminate the different events. A custom event SHOULD NOT have a
first byte colliding with any of the events defined by this
specification.

.. list-table::
   :header-rows: 1

   * - Event
     - Tag
   * - :ref:`CIS-8-ExternalKeyRegistered`
     - 231
   * - :ref:`CIS-8-ExternalKeyRevoked`
     - 232
   * - :ref:`CIS-8-UpdateMetadataEvent`
     - 233

These tag values intentionally avoid the 251..255 range reserved by
:ref:`CIS-2`, so a single contract MAY implement both CIS-8 and CIS-2
without event-tag collisions.

.. _CIS-8-ExternalKeyRegistered:

``ExternalKeyRegistered``
^^^^^^^^^^^^^^^^^^^^^^^^^

Emitted when a new registration is created, when an existing owner
re-registers, or when a new owner replaces a previous one (in which case
it is preceded by an :ref:`CIS-8-ExternalKeyRevoked` for the previous
owner).

It is serialized as: 1 byte for the event tag, followed by the 32-byte
``owner`` ``AccountAddress``, followed by a serialised
:ref:`CIS-8-ExternalKeyId` (``external_key``), followed by 2 bytes for
the proof scheme length (``n_ps``) and ``n_ps`` UTF-8 bytes
(``proof_scheme``), followed by 4 bytes for the number of metadata
entries (``n_m``) and ``n_m`` serialised
:ref:`CIS-8-MetadataEntry`\ s::

  ExternalKeyRegisteredEvent ::= (231: Byte)
                                 (owner: Byte³²)
                                 (external_key: ExternalKeyId)
                                 (n_ps: Byte²) (proof_scheme: Byteⁿ_ᵖˢ)
                                 (n_m: Byte⁴) (metadata: MetadataEntryⁿ_ᵐ)

.. _CIS-8-ExternalKeyRevoked:

``ExternalKeyRevoked``
^^^^^^^^^^^^^^^^^^^^^^

Emitted when the active owner revokes their own registration, or when a
new owner replaces them via :ref:`CIS-8-register`.

It is serialized as: 1 byte for the event tag, followed by the 32-byte
``owner`` ``AccountAddress``, followed by a serialised
:ref:`CIS-8-ExternalKeyId` (``external_key``)::

  ExternalKeyRevokedEvent ::= (232: Byte)
                              (owner: Byte³²)
                              (external_key: ExternalKeyId)

.. _CIS-8-UpdateMetadataEvent:

``UpdateMetadata``
^^^^^^^^^^^^^^^^^^

Emitted on a successful :ref:`CIS-8-updateMetadata` call.

It is serialized as: 1 byte for the event tag, followed by a serialised
:ref:`CIS-8-ExternalKeyId` (``external_key``), followed by 4 bytes for
the number of metadata entries (``n_m``) and ``n_m`` serialised
:ref:`CIS-8-MetadataEntry`\ s::

  UpdateMetadataEvent ::= (233: Byte)
                          (external_key: ExternalKeyId)
                          (n_m: Byte⁴) (metadata: MetadataEntryⁿ_ᵐ)


Errors
------

Reject codes MUST take the explicit numeric values listed below;
implementations MUST NOT rely on derived sequential numbering.

The ``-7100..`` range was chosen so CIS-8 reject codes do not collide
with other CIS standards (``-42000..`` for CIS-2 etc.) when a single
contract implements multiple standards.

.. _CIS-8-InvalidProof:
.. _CIS-8-UnsupportedProofScheme:
.. _CIS-8-MalformedExternalKey:
.. _CIS-8-Unauthorized:
.. _CIS-8-NotRegistered:
.. _CIS-8-InvalidMetadata:

.. list-table:: Reject Codes
   :header-rows: 1

   * - Code
     - Name
     - Meaning
   * - -7100
     - ``InvalidProof``
     - Signature does not verify against the reconstructed canonical message.
   * - -7101
     - ``UnsupportedProofScheme``
     - ``proof.scheme`` is not in the contract's supported set.
   * - -7102
     - ``MalformedExternalKey``
     - ``external_key`` fails the validation rules in
       :ref:`CIS-8-ExternalKeyId`.
   * - -7103
     - ``Unauthorized``
     - Caller is not authorised — typically: caller is a contract, or caller
       is not the active owner for ``updateMetadata`` / ``revoke``.
   * - -7104
     - ``AlreadyRegistered``
     - Reserved; redundant re-registration MAY use this code, though the
       replacement rule in :ref:`CIS-8-register` instead handles same-owner
       re-registration as an in-place replacement.
   * - -7105
     - ``NotRegistered``
     - No active registration exists for the supplied identifier.
   * - -7106
     - ``AmbiguousIdentifier``
     - Reserved for future use.
   * - -7107
     - ``UnsupportedKeyType``
     - ``external_key.key_type`` is not supported by this contract.
   * - -7108
     - ``InvalidMetadata``
     - ``metadata`` fails the validation rules in :ref:`CIS-8-MetadataEntry`.


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
