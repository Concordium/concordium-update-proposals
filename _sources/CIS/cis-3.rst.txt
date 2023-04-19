.. _CIS-3:

=====================================
CIS-3: Sponsored Transaction Standard
=====================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Apr 04, 2023
   * - Final
     - Apr 19, 2023
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-3``

Abstract
========

A standard for how a smart contract can enable sponsored transactions.
This specification defines a mechanism for a sponsor address to submit a transaction
to a smart contract on behalf of a sponsoree account. The sponsor address (invoker to the smart contract)
pays for the transaction fees while the sponsoree has to sign its intended action in a message to authorize the sponsor to
execute a specific action on its behalf.

Additionally, the interface provides an entrypoint for querying the smart contract about which
entrypoints can be invoked with a sponsored transaction.
The smart contract responds with either: not supported, or supported.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

General types and serialization
-------------------------------

.. _CIS-3-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex both unsigned 64 bit integers.

It is serialized as: First 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``) both little endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)

.. _CIS-3-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^

An address of an account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte³²)

.. _CIS-3-EntrypointName:

``EntrypointName``
^^^^^^^^^^^^^^^^^^

A name for a smart contract function entrypoint.

It is serialized as: First 2 bytes encode the length (``n``) of the entrypoint name, followed by this many bytes for the entrypoint name (``entrypoint``)::

  EntrypointName ::= (n: Byte²) (entrypoint: Byteⁿ)

.. _CIS-3-Timestamp:

``Timestamp``
^^^^^^^^^^^^^

A timestamp given in milliseconds since unix epoch.
It consists of an unsigned 64 bit integer.

It is serialized as 8 bytes in little endian::

  Timestamp ::= (milliseconds: Byte⁸)

.. _CIS-3-Nonce:

``Nonce``
^^^^^^^^^

An unsigned 64 bit integer number that increases sequentially to protect against replay attacks.

It is serialized as 8 bytes in little endian::

  Nonce ::= (nonce: Byte⁸)

.. _CIS-3-SignatureEd25519:

``SignatureEd25519``
^^^^^^^^^^^^^^^^^^^^

Signature for an Ed25519 message.

It is serialized as 64 bytes::

  SignatureEd25519 ::= (signature: Byte⁶⁴)

.. _CIS-3-TwoLevelSignatureMap:

``TwoLevelSignatureMap``
^^^^^^^^^^^^^^^^^^^^^^^^

It consists of two maps. Each map stores key-value pairs, where the keys are unsigned 8 bit integers in both maps.
The value in the outer map is the inner map and the value in the inner map is of type ``SignatureEd25519``.

It is serialized as: For each map, the first byte encodes the size (``n`` for the outer map and ``m`` for the inner map)
of the map, followed by this many key-value pairs::

  InnerMap ::= (m: Byte) (key: Byte, value: SignatureEd25519)ᵐ
  TwoLevelSignatureMap ::= (n: Byte) (key: Byte, value: InnerMap)ⁿ

Logged events
-------------

Entrypoints invoked via the `permit` function using the sponsored
transaction mechanism MUST log the same events as if they were invoked
by the sponsoree directly without the sponsored transaction mechanism.
In addition, the `permit` function SHALL log an event with the `nonce` and the `sponsoree` address every time
the `permit` function is invoked.

The event defined by this specification is serialized using one byte to discriminate it from other events logged by the smart contract.
Other events logged by the smart contract SHOULD NOT have a first byte colliding with the event defined by this specification.

``NonceEvent``
^^^^^^^^^^^^^^

A ``NonceEvent`` SHALL be logged for every `permit` function invoke.

The ``NonceEvent`` is serialized as: First a byte with the value of 250, followed by the :ref:`CIS-3-Nonce` (``nonce``) that was used in the PermitMessage, and an :ref:`CIS-3-AccountAddress` (``sponsoree``)::

  NonceEvent ::= (250: Byte) (nonce: Nonce) (sponsoree: AccountAddress)

Contract function
-----------------

A smart contract implementing CIS-3 MUST export the functions: :ref:`CIS-3-functions-permit` and :ref:`CIS-3-functions-supportsPermit`.

.. _CIS-3-functions-permit:

``permit``
^^^^^^^^^^

Verifies an ed25519 signature from a sponsoree and authorizes the sponsor to execute the logic of
specific entrypoints on behalf of the sponsoree. The sponsored transaction mechanism replaces the
authorization checks conducted on the `sender/invoker` variable with signature verification.
That is, the sponsoree needs to sign its intended action and the signature is verified in the smart contract.

Parameter
~~~~~~~~~

The parameter (``PermitParam``) contains a two-level signature map and a signer account that created the signature
together with the message that was signed.

.. note::

    The CIS3 standard supports multi-sig accounts which is the purpose of the two-level signature map. A basic account (no multi-sig account) SHOULD have its signature at the key 0 in both maps.

The message (``PermitMessage``) contains a contract_address (``ContractAddress``), entry_point (``EntrypointName``), nonce (``Nonce``), timestamp (``Timestamp``), and the payload (``PermitPayload``).
This message structure enables the sponsoree the authorize the sponsor to act on its behalf in the given scope.

The payload (``PermitPayload``) is serialized as: First 2 bytes encode the length (``n``) of the payload, followed by this many bytes for the payload (``entrypoint_parameter``)::

  PermitPayload ::= (n: Byte²) (entrypoint_parameter: Byteⁿ)

  PermitMessage ::= (contract_address: ContractAddress) (nonce: Nonce) (timestamp: Timestamp) (entry_point: EntrypointName) (payload: PermitPayload)

  PermitParam ::= (signature: TwoLevelSignatureMap) (signer: AccountAddress) (message: PermitMessage)

Requirements
~~~~~~~~~~~~

- The requirements specified for an entrypoint and the outcome of the invoke MUST be the same as if it was invoked directly by the sponsoree. E.g. a smart contract implementing an `updateOperator/transfer` function from the CIS-2 standard, if these entrypoints are invoked via the `permit` function, the sponsored transaction invoke MUST adhere to the CIS-2 standard as well and create the same outcome as if the sponsoree invokes the `updateOperator/transfer` function directly.
- The PermitMessage MUST include a nonce to protect against replay attacks. The sponsoree's nonce is sequentially increased every time a `PermitMessage` (signed by the sponsoree) is successfully executed in the `permit` function. The `permit` function MUST only accept a `PermitMessage` if it has the next nonce following the sequential order.
- An invoke MUST fail if the signature was intended for a different contract.
- An invoke MUST fail if the signature was intended for a different entrypoint.
- An invoke MUST fail if the signature is expired.
- An invoke MUST fail if the signature can not be validated. The smart contract logic SHOULD practice its best efforts to ensure that only the sponsoree can generate and authorize its intended action with a valid signature.

.. _CIS-3-functions-supportsPermit:

``supportsPermit``
^^^^^^^^^^^^^^^^^^

Query supported entrypoints by the `permit` function given a list of entrypoints.
The response contains a corresponding result for each entrypoint, where the result is either
"Entrypoint is not supported and can not be invoked via the `permit` function using the sponsored transaction mechanism" or "Entrypoint is supported and can be invoked via the `permit` function using the sponsored transaction mechanism".

Parameter
~~~~~~~~~

The parameter consists of a list of entrypoints.

It is serialized as: 2 bytes for the number (little endian) of the entrypoints and then this number of ``EntrypointNames``::

  SupportsPermitQueryParams ::= (n : Byte²) (names: EntrypointNameⁿ)

Response
~~~~~~~~

The function output is a list of support results, where the order of the support results matches the order of ``EntrypointNames`` in the parameter.

It is serialized as: 2 bytes for the number (little endian) of results (``n``) and then this number of support results (``results``).
A support result is serialized as either: A byte with value ``0`` for "Entrypoint is not supported" or a byte with the value ``1`` for "Entrypoint is supported by this contract"::

  SupportResult ::= (0 : Byte)  // Entrypoint is not supported by the `permit` function
                  | (1 : Byte)  // Entrypoint is supported by the `permit` function

  SupportsResponse ::= (n : Byte²) (results: SupportResultⁿ)

Requirements
~~~~~~~~~~~~

- The number of results in the response MUST correspond to the number of queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST be non-mutative.

Limitations
===========

A number of limitations are important to be aware of:

- Only accounts can generate a valid Ed25519 signature using public-private key cryptography. Smart contracts can not be a sponsoree as defined in this CIS-3 standard.

- To validate a signature, the smart contract needs to have access to its corresponding public key. Concordium smart contracts currently have no way to query the corresponding public key(s) of an account within the smart contract code. For the time being a `public_key_registry` is recommended to be added to the smart contract to only allow a trusted party to register a public key for a given account. The Concordium team is working on exposing the public key within the smart contract code and this feature is planned to be included in the next protocol update.
