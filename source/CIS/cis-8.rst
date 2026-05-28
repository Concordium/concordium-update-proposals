.. _CIS-8:

==============================
CIS-8: External Key Registry
==============================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 21, 2026
   * - Draft
     - May 25, 2026
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-8``
   * - Requires
     - :ref:`CIS-0<CIS-0>`


Abstract
========

A standard interface for registering a public link between a Concordium account and an external public key used on another blockchain.
A registration asserts that a Concordium account controls a specified external public key, proven by a cryptographic signature produced with that external key.

The standard is chain-agnostic: it supports external blockchains including Ethereum, Solana, Fetch.ai, and Cosmos-SDK chains using CAIP-style chain namespace identifiers, per-chain-family proof scheme identifiers, and chain-specific public key encodings.

Verifier-anchored assertions for non-cryptographic external identifiers such as platform handles or HTTPS endpoints are out of scope.

The interface defines the following:

- Data structures for external key identifiers, ownership proofs, and registrations.
- Entrypoints for registering, updating, revoking, and querying external key links.
- Logged events for each mutating operation.
- Standard rejection error codes.


Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.


General types and serialization
--------------------------------

.. note::

   Integers are encoded in little-endian unless stated otherwise.

.. _CIS-8-String:

``String``
^^^^^^^^^^

A variable-length UTF-8 encoded string.

It is serialized as: 2 bytes for the length (``n``) of the string in little-endian, followed by ``n`` bytes for the UTF-8 encoding of the string::

  String ::= (n: Byte2) (value: Byten)

.. _CIS-8-Bytestring:

``Bytestring``
^^^^^^^^^^^^^^

A variable-length byte array.

It is serialized as: 2 bytes for the length (``n``) in little-endian, followed by ``n`` bytes::

  Bytestring ::= (n: Byte2) (value: Byten)

.. _CIS-8-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^

An address of a Concordium account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte32)

.. _CIS-8-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a Concordium smart contract instance.
It consists of an index and a subindex, both unsigned 64-bit integers.

It is serialized as: 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``), both little-endian::

  ContractAddress ::= (index: Byte8) (subindex: Byte8)

.. _CIS-8-BlockHash:

``BlockHash``
^^^^^^^^^^^^^

A 32-byte block hash identifying a Concordium block.
Used in the :ref:`canonical signed message<CIS-8-CanonicalSignedMessage>` to carry the genesis block hash and prevent cross-network replay.

It is serialized as 32 bytes::

  BlockHash ::= (hash: Byte32)

.. _CIS-8-Timestamp:

``Timestamp``
^^^^^^^^^^^^^

A point in time given in milliseconds since Unix epoch, represented as an unsigned 64-bit integer.

It is serialized as 8 bytes in little-endian::

  Timestamp ::= (milliseconds: Byte8)

.. _CIS-8-ExternalKeyId:

``ExternalKeyId``
^^^^^^^^^^^^^^^^^

An identifier for a public key on an external blockchain, consisting of a CAIP-style chain namespace, a key type identifier, and the raw public key bytes.

The ``namespace`` field is a CAIP-style chain identifier (e.g., ``eip155:1``, ``solana:mainnet``, ``cosmos:fetchhub-4``).

The ``key_type`` field identifies the public key format and encoding.
Standard values defined by this specification are:

- ``secp256k1-compressed`` - 33 bytes; used by Ethereum and Cosmos chains.
- ``secp256k1-uncompressed`` - 65 bytes; alternate Ethereum encoding.
- ``ed25519`` - 32 bytes; used by Solana and Fetch.ai uagents.

The ``public_key`` field is the raw encoded public key bytes.
Encoding is determined by ``key_type``.

It is serialized as a :ref:`CIS-8-String` (``namespace``), a :ref:`CIS-8-String` (``key_type``), and a :ref:`CIS-8-Bytestring` (``public_key``)::

  ExternalKeyId ::= (namespace: String) (key_type: String) (public_key: Bytestring)

.. _CIS-8-Proof:

``Proof``
^^^^^^^^^

A cryptographic ownership proof, produced by signing the :ref:`canonical signed message<CIS-8-CanonicalSignedMessage>` with the external key being registered.

The ``scheme`` field is a proof scheme identifier (e.g., ``ethereum-personal-sign``, ``solana-ed25519``, ``cosmos-secp256k1``, ``fetch-ai-ed25519``).
The ``signature`` field contains the signature bytes as produced by that proof scheme.

It is serialized as a :ref:`CIS-8-String` (``scheme``) followed by a :ref:`CIS-8-Bytestring` (``signature``)::

  Proof ::= (scheme: String) (signature: Bytestring)

.. _CIS-8-MetadataEntry:

``MetadataEntry``
^^^^^^^^^^^^^^^^^

An application-defined key-value string pair associated with a registration.

It is serialized as a :ref:`CIS-8-String` (``key``) followed by a :ref:`CIS-8-String` (``value``)::

  MetadataEntry ::= (key: String) (value: String)

.. _CIS-8-RegistrationStatus:

``RegistrationStatus``
^^^^^^^^^^^^^^^^^^^^^^

The lifecycle status of a registration.

It is serialized as 1 byte with value 0 for ``Active`` and 1 for ``Revoked``::

  RegistrationStatus ::= (0: Byte) // Active
                       | (1: Byte) // Revoked

.. _CIS-8-Registration:

``Registration``
^^^^^^^^^^^^^^^^

A complete record of an external key registration, including the linked Concordium account, the external key, the proof scheme used at registration time, application-defined metadata, the current status, and the time of the last status change.

It is serialized as an :ref:`CIS-8-AccountAddress` (``owner``), an :ref:`CIS-8-ExternalKeyId` (``external_key``), a :ref:`CIS-8-String` (``proof_scheme``), 2 bytes for the number of metadata entries (``m``), followed by ``m`` :ref:`CIS-8-MetadataEntry` records (``metadata``), a :ref:`CIS-8-RegistrationStatus` (``status``), and a :ref:`CIS-8-Timestamp` (``last_updated``)::

  Registration ::= (owner: AccountAddress) (external_key: ExternalKeyId) (proof_scheme: String)
                   (m: Byte2) (metadata: MetadataEntrym) (status: RegistrationStatus)
                   (last_updated: Timestamp)


.. _CIS-8-CanonicalSignedMessage:

Canonical signed message
------------------------

The canonical signed message is the message that MUST be signed with the external private key as the ownership proof in a :ref:`registerExternalKey<CIS-8-functions-registerExternalKey>` call.

The message MUST NOT include a nonce or expiry timestamp.
Replay protection is provided by the Concordium transaction itself, which is bound to the sender account, a sequential nonce, and an expiry time at the protocol level.
Cross-contract and cross-network replay is prevented by the inclusion of the ``contract_address`` and ``concordium_genesis_hash`` fields.

The message bytes MUST be prefixed with the 18-byte ASCII domain separation tag ``CIS-8/v1/canonical``.

It is serialized as the 18 raw ASCII bytes of the domain separation tag, followed by an :ref:`CIS-8-AccountAddress` (``concordium_account``), a :ref:`CIS-8-ContractAddress` (``contract_address``), a :ref:`CIS-8-BlockHash` (``concordium_genesis_hash``), a :ref:`CIS-8-String` (``external_namespace``), an :ref:`CIS-8-ExternalKeyId` (``external_key``), and a :ref:`CIS-8-String` (``proof_scheme``)::

  CanonicalSignedMessage ::= ("CIS-8/v1/canonical": Byte18)
                              (concordium_account: AccountAddress)
                              (contract_address: ContractAddress)
                              (concordium_genesis_hash: BlockHash)
                              (external_namespace: String)
                              (external_key: ExternalKeyId)
                              (proof_scheme: String)


Proof verification
------------------

This section specifies the proof schemes that a conformant CIS-8 contract SHOULD support.
A contract MUST reject with ``UnsupportedProofScheme`` for any scheme identifier it does not implement.
A contract MAY support additional proof schemes beyond those listed here.

.. _CIS-8-proof-ethereum-personal-sign:

``ethereum-personal-sign``
^^^^^^^^^^^^^^^^^^^^^^^^^^

- **Algorithm**: ECDSA over secp256k1.
- **Hash**: Keccak-256.
- **Public key encoding**: secp256k1 compressed (33 bytes) or uncompressed (65 bytes).
- **Message construction**: Let ``m`` be the :ref:`canonical signed message<CIS-8-CanonicalSignedMessage>` bytes.
  Construct ``"\x19Ethereum Signed Message:\n" + ascii(len(m)) + m`` and hash with Keccak-256.
- **Signature**: 65 bytes (r || s || v). Verify by public-key recovery and comparison against the supplied ``public_key``.

.. _CIS-8-proof-solana-ed25519:

``solana-ed25519``
^^^^^^^^^^^^^^^^^^

- **Algorithm**: Ed25519.
- **Public key encoding**: 32 bytes.
- **Signature**: 64 bytes (R || S).
- **Message construction**: Sign the :ref:`canonical signed message<CIS-8-CanonicalSignedMessage>` bytes directly
  (the ``CIS-8/v1/canonical`` domain prefix is included as part of the message).

.. _CIS-8-proof-cosmos-secp256k1:

``cosmos-secp256k1``
^^^^^^^^^^^^^^^^^^^^

- **Algorithm**: ECDSA over secp256k1.
- **Hash**: SHA-256.
- **Public key encoding**: secp256k1 compressed (33 bytes).
- **Message construction**: Wrap the :ref:`canonical signed message<CIS-8-CanonicalSignedMessage>` bytes
  in an ADR-036 off-chain envelope, then hash with SHA-256.
- **Signature**: Standard ECDSA (r || s). Verify against the supplied compressed public key.

.. _CIS-8-proof-fetch-ai-ed25519:

``fetch-ai-ed25519``
^^^^^^^^^^^^^^^^^^^^

- **Algorithm**: Ed25519.
- **Public key encoding**: 32 bytes (Fetch.ai uagent ed25519 public key).
- **Signature**: 64 bytes (R || S).
- **Message construction**: Identical to :ref:`solana-ed25519<CIS-8-proof-solana-ed25519>`.

Logged events
-------------

The events defined by this specification are serialized using one byte to discriminate the different events.
A custom event SHOULD NOT have a first byte colliding with any of the events defined by this specification.

If account B replaces account A as the active owner of an external key, the contract MUST emit
``ExternalKeyRevoked`` for account A before emitting ``ExternalKeyRegistered`` for account B.

.. _CIS-8-events-ExternalKeyRegistered:

``ExternalKeyRegistered``
^^^^^^^^^^^^^^^^^^^^^^^^^

An ``ExternalKeyRegistered`` event MUST be logged whenever an external key is successfully registered or re-registered.

It is serialized as: first a byte with the value of 231, followed by the :ref:`CIS-8-AccountAddress` (``owner``) and the :ref:`CIS-8-ExternalKeyId` (``external_key``)::

  ExternalKeyRegistered ::= (231: Byte) (owner: AccountAddress) (external_key: ExternalKeyId)

.. _CIS-8-events-ExternalKeyRevoked:

``ExternalKeyRevoked``
^^^^^^^^^^^^^^^^^^^^^^

An ``ExternalKeyRevoked`` event MUST be logged whenever an active registration is revoked, whether by an explicit :ref:`revoke<CIS-8-functions-revoke>` call or by displacement through a new :ref:`registerExternalKey<CIS-8-functions-registerExternalKey>` call.

It is serialized as: first a byte with the value of 232, followed by the :ref:`CIS-8-AccountAddress` (``owner``) and the :ref:`CIS-8-ExternalKeyId` (``external_key``)::

  ExternalKeyRevoked ::= (232: Byte) (owner: AccountAddress) (external_key: ExternalKeyId)

.. _CIS-8-events-UpdateMetadata:

``UpdateMetadata``
^^^^^^^^^^^^^^^^^^

An ``UpdateMetadata`` event MUST be logged whenever the metadata of an active registration is successfully updated.

It is serialized as: first a byte with the value of 233, followed by the :ref:`CIS-8-AccountAddress` (``owner``), the :ref:`CIS-8-ExternalKeyId` (``external_key``), 2 bytes for the number of metadata entries (``m``), and then ``m`` :ref:`CIS-8-MetadataEntry` records (``metadata``)::

  UpdateMetadata ::= (233: Byte) (owner: AccountAddress) (external_key: ExternalKeyId)
                     (m: Byte2) (metadata: MetadataEntrym)

.. _CIS-8-functions:

Contract functions
------------------

A smart contract implementing this standard MUST export the following functions:

- :ref:`CIS-8-functions-supports`
- :ref:`CIS-8-functions-registerExternalKey`
- :ref:`CIS-8-functions-updateMetadata`
- :ref:`CIS-8-functions-revoke`
- :ref:`CIS-8-functions-ownerOfKey`


.. _CIS-8-functions-supports:

``supports``
^^^^^^^^^^^^

As specified in :ref:`CIS-0<CIS-0>`.

.. _CIS-8-functions-registerExternalKey:

``registerExternalKey``
^^^^^^^^^^^^^^^^^^^^^^^

Register or replace the active owner of an external key with a cryptographic ownership proof.
The proof MUST be signed with the external private key over the :ref:`canonical signed message<CIS-8-CanonicalSignedMessage>` constructed from the transaction context and the supplied parameters.

If an active registration already exists for the supplied external key under a different owner, the previous registration is displaced: the contract MUST emit ``ExternalKeyRevoked`` for the displaced owner before emitting ``ExternalKeyRegistered`` for the new owner.

Parameter
~~~~~~~~~

The parameter consists of an :ref:`CIS-8-ExternalKeyId` (``external_key``), a :ref:`CIS-8-Proof` (``proof``), 2 bytes for the number of metadata entries (``m``), and then ``m`` :ref:`CIS-8-MetadataEntry` records (``metadata``)::

  RegisterExternalKeyParam ::= (external_key: ExternalKeyId) (proof: Proof) (m: Byte²) (metadata: MetadataEntryᵐ)

Requirements
~~~~~~~~~~~~

- The ``proof.scheme`` MUST be supported by the contract; otherwise the contract MUST reject with ``UnsupportedProofScheme``.
- The ``external_key.key_type`` MUST be supported by the contract; otherwise the contract MUST reject with ``UnsupportedKeyType``.
- The ``external_key`` MUST be well-formed for its declared ``key_type``; otherwise the contract MUST reject with ``MalformedExternalKey``.
- The ``proof`` MUST verify against the canonical signed message reconstructed from the transaction context and the supplied parameters; otherwise the contract MUST reject with ``InvalidProof``.
- If the caller is already the active owner of the supplied external key the contract MUST reject with ``AlreadyRegistered``.
- If ``metadata`` violates implementation-defined limits the contract MUST reject with ``InvalidMetadata``.
- On success the contract MUST emit :ref:`ExternalKeyRegistered<CIS-8-events-ExternalKeyRegistered>`.

.. _CIS-8-functions-updateMetadata:

``updateMetadata``
^^^^^^^^^^^^^^^^^^

Update the metadata of an existing active registration.
The caller MUST be the current active owner of the registration.

Parameter
~~~~~~~~~

The parameter consists of an :ref:`CIS-8-ExternalKeyId` (``external_key``), 2 bytes for the number of metadata entries (``m``), and then ``m`` :ref:`CIS-8-MetadataEntry` records (``metadata``)::

  UpdateMetadataParam ::= (external_key: ExternalKeyId) (m: Byte2) (metadata: MetadataEntrym)

Requirements
~~~~~~~~~~~~

- The contract MUST reject with ``NotRegistered`` if no active registration exists for the supplied external key.
- The contract MUST reject with ``Unauthorized`` if the caller is not the active owner of the registration.
- If ``metadata`` violates implementation-defined limits the contract MUST reject with ``InvalidMetadata``.
- On success the contract MUST emit :ref:`UpdateMetadata<CIS-8-events-UpdateMetadata>`.

.. _CIS-8-functions-revoke:

``revoke``
^^^^^^^^^^

Revoke an active registration.
The caller MUST be the current active owner of the registration.

Parameter
~~~~~~~~~

The parameter consists of the :ref:`CIS-8-ExternalKeyId` (``external_key``) to revoke::

  RevokeParam ::= (external_key: ExternalKeyId)

Requirements
~~~~~~~~~~~~

- The contract MUST reject with ``NotRegistered`` if no active registration exists for the supplied external key.
- The contract MUST reject with ``Unauthorized`` if the caller is not the active owner.
- On success the contract MUST emit :ref:`ExternalKeyRevoked<CIS-8-events-ExternalKeyRevoked>`.

.. _CIS-8-functions-ownerOfKey:

``ownerOfKey``
^^^^^^^^^^^^^^

Query the active registration for a given external key identifier.

Parameter
~~~~~~~~~

The parameter consists of the :ref:`CIS-8-ExternalKeyId` (``external_key``) to query::

  OwnerOfKeyParam ::= (external_key: ExternalKeyId)

Response
~~~~~~~~

The response is 1 byte indicating whether an active registration was found.
If the value is 0, no active registration exists and no further bytes follow.
If the value is 1, the full :ref:`CIS-8-Registration` (``registration``) follows::

  OwnerOfKeyResponse ::= (0: Byte)                                 // not found
                       | (1: Byte) (registration: Registration)    // found




.. _CIS-8-errors:

Rejection errors
----------------

A smart contract following this specification MUST use the following error codes to reject under the described conditions:

.. list-table::
  :header-rows: 1

  * - Name
    - Error code
    - Description
  * - ``InvalidProof``
    - -7100
    - The supplied proof does not verify against the reconstructed canonical signed message.
  * - ``UnsupportedProofScheme``
    - -7101
    - The proof scheme identifier is not supported by this contract.
  * - ``MalformedExternalKey``
    - -7102
    - The external key identifier is malformed or its encoding does not match the declared ``key_type``.
  * - ``Unauthorized``
    - -7103
    - The sender is not the active owner of the registration.
  * - ``AlreadyRegistered``
    - -7104
    - The sender is already the active owner of the supplied external key.
  * - ``NotRegistered``
    - -7105
    - No active registration exists for the supplied identifier.
  * - ``AmbiguousIdentifier``
    - -7106
    - The supplied identifier resolves ambiguously.
  * - ``UnsupportedKeyType``
    - -7107
    - The supplied ``key_type`` is not supported by this contract.
  * - ``InvalidMetadata``
    - -7108
    - The metadata violates implementation-defined limits or encoding requirements.

Rejecting using an error code from the table above MUST only occur in a situation as described in the corresponding error description.

The smart contract implementing this specification MAY introduce custom error codes other than the ones specified in the table above.
