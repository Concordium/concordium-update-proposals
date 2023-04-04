.. _CIS-3:

=====================================
CIS-3: Sponsored Transaction Standard
=====================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Apr 04, 2023
   * - Final
     - Apr 15, 2023
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

The interface provides an entrypoint for querying the smart contract about which
entrypoints can be invoked with a sponsored transaction.
The smart contract responds with either: not supported, or supported.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

Logged events
-------------

This standard does not introduce any additional events. Entrypoints invoked via the `permit` function using the sponsored
transaction mechanism MUST emit the same events as if they were invoked
by the sponsoree directly without the sponsored transaction mechanism.

Contract function
-----------------

A smart contract implementing CIS-3 MUST export the functions: :ref:`CIS-3-functions-permit` and :ref:`CIS-3-functions-supportsPermit`.

.. _CIS-3-functions-permit:

``permit``
^^^^^^^^^^

Verifies an ed25519 signature from a sponsoree and authorizes the sponsor to execute the logic of
specific entrypoints on behalf of the sponsoree. The sponsored transaction mechanism replaces the
authorization checks conducted on the `sender/invoker` variable with signature verification.

Parameter
~~~~~~~~~

The parameter (``PermitParam``) contains a two-level signature map and a signer account that created the signature
together with the message that was signed.

.. note::

    The CIS3 standard supports multi-sig accounts which is the purpose of the two-level signature map. A basic account (no multi-sig account) SHOULD have its signature at the key 0 in both maps.

The message (``PermitMessage``) contains a contract_address (``ContractAddress``), entry_point (``OwnedEntrypointName``), nonce (``u64``), timestamp (``Timestamp``), and the payload (``PermitPayload``).
This message structure enables the sponsoree the authorize the sponsor to act on its behalf in the given scope.

The payload (``PermitPayload``) is serialized as: The first byte to distinguish the entrypoint followed by the parameter intended for that entrypoint::

  PermitPayload  ::= (0: Byte) (Entrypoint_1_Parameter) // First entrypoint supported
                   | (1: Byte) (Entrypoint_2_Parameter) // Second entrypoint supported
                   | (2: Byte) (Entrypoint_3_Parameter) // Third entrypoint supported
                   ...

  PermitMessage ::= (contract_address: ContractAddress) (entry_point: OwnedEntrypointName) (nonce: u64) (timestamp: Timestamp) (payload: PermitPayload)

  PermitParam ::= (signature: BTreeMap<u8, BTreeMap<u8, SignatureEd25519>>) (signer: AccountAddress) (message: PermitMessage)

.. note::

    The PermitPayload is an enum and the CIS-3 standard specifies that the first byte is used to distinguish between the different entrypoints.
    This definition enables the serialization of 255 different entrypoints which should be sufficient in practice.

Requirements
~~~~~~~~~~~~

- The requirements specified for an entrypoint MUST persist even if invoked via the `permit` function. E.g. a smart contract implementing an `updateOperator/transfer` function from the CIS-2 standard, if these entrypoints are invoked via the `permit` function, the sponsored transaction invoke MUST adhere to the CIS-2 standard as well.
- The signature verification MUST have some replay attack prevention mechanisms (e.g. nonces).
- An invoke MUST fail if the signature was intended for a different contract.
- An invoke MUST fail if the signature was intended for a different entrypoint.
- An invoke MUST fail if the signature is expired.
- An invoke MUST fail if the signature can not be validated. The smart contract logic SHOULD practice its best efforts to ensure that only the sponsoree can generate and authorize its intended action with a valid signature.

.. _CIS-3-functions-supportsPermit:

``supportsPermit``
^^^^^^^^^^^^^^^^^^

Query supported entrypoints by the `permit` function given a list of entrypoints.
The response contains a corresponding result for each entrypoint, where the result is either
"Entrypoint is not supported and can not be invoked via the `permit` function using the sponsored transaction mechanim" or "Entrypoint is supported and can be invoked via the `permit` function using the sponsored transaction mechanim".

Parameter
~~~~~~~~~

The parameter consists of a list of entrypoints.

It is serialized as: 2 bytes for the number (little endian) of the entrypoints and then this number of ``OwnedEntrypointNames``::

  SupportsPermitQueryParams ::= (n : Byte²) (names: OwnedEntrypointNameⁿ)

Response
~~~~~~~~

The function output is a list of support results, where the order of the support results matches the order of ``OwnedEntrypointNames`` in the parameter.

It is serialized as: 2 bytes for the number (little endian) of results (``n``) and then this number of support results (``results``).
A support result is serialized as either: a byte with value ``0`` for "Entrypoint is not supported", a byte with the value ``1`` for "Entrypoint is support by this contract"::

  SupportResult ::= (0 : Byte)  // Entrypoint is not supported
                  | (1 : Byte)  // Entrypoint is supported by this contract

  SupportsResponse ::= (n : Byte²) (results: SupportResultⁿ)

Requirements
~~~~~~~~~~~~

- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST reject if any of the queries fail.
- The contract function MUST be non-mutative.

Limitations
===========

A number of limitations are important to be aware of:

- Only accounts can generate a valid Ed25519 signature using public-private key cryptography. Smart contracts can not be a sponsoree as defined in this CIS-3 standard.

- To validate a signature, the smart contract needs to have access to its corresponding public key. Concordium smart contracts currently have no way to query the corresponding public key(s) of an account within the smart contract code. For the time being a `public_key_registry` is recommended to be added to the smart contract to only allow the owner of the contract instance (or the account itself) to register a public key for a given account. The Concordium team is working on exposing the public key within the smart contract code and this feature is planned to be included in the next protocol update which will result in an update to this standard.
