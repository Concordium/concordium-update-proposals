.. _CIS-0:

=========================
CIS-0: Standard Detection
=========================

.. list-table::
   :stub-columns: 1

   * - Created
     - Jul 14, 2022
   * - Draft version
     - 1
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-0``

.. warning::

   This standard is still a draft and significant changes might still be made.

Abstract
========

A standardized interface for how a smart contract indicate supported smart contract standards.
Additionally, this specification includes a list of identifiers for each standard used on the Concordium blockchain.

The interface provides an entrypoint for querying the smart contract about whether it supports a given standard. The smart contract responds with either: not supported, supported or supported by some contract address.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

.. _CIS-0-standard-identifer:

Standard identifer
-------------------

Each standard MUST have a unique identifier.
An identifier is a variable-length ASCII string up to 255 characters.

The identifier is serialized as: first 1 byte for the length followed this many bytes for the ASCII encoding of the identifier::

  StandardIdentifier ::= (n: Byte) (id: Byteⁿ)

The table below contains the current standards and associated standard identifier.

Contract function
-----------------

A smart contract implementing CIS-0 MUST export the following function: :ref:`CIS-0-functions-supports`.

.. _CIS-0-functions-supports:

``supports``
^^^^^^^^^^^^

Query supported standards using a list of standard identifers. The response contains a corresponding result for each standard identifier, where the result is either "Standard is not supported", "Standard is supported by this contract" or "Standard is supported by using some contract address".

.. note::

   The result of "Standard is supported by using some contract address" can be used to add support of smart contract standards in another smart contract.
   This could be to split up logic or to later add support for a new standard.

Parameter
~~~~~~~~~

The parameter consists of a list of standard identifers.

It is serialized as: 2 bytes for the number (little endian) of standard identifers and then this number of :ref:`CIS-0-standard-identifer` identifers::

  SupportsParameter ::= (n : Byte²) (ids: StandardIdentifierⁿ)

Response
~~~~~~~~

The function output is a list of support results, where the order of the support results matches the order of standard identifers in the parameter.

It is serialized as: 2 bytes for the number (little endian) of results (``n``) and then this number of support results (``results``).
A support result is serialized as either: a byte with value ``0`` for "Standard is not supported", a byte with the value ``1`` for "Standard is support by this contract" or a byte with value ``2`` for "Standard is supported by using another contract address" followed by a single byte encoding the number of contract addresses, then this number of contract addresses (``address``).
A contract address is serialized as: first 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``) both little endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)

  SupportResult ::= (0 : Byte)                             // Standard is not supported
                  | (1 : Byte)                             // Standard is supported by this contract
                  | (2 : Byte) (n: Byte) (address: ContractAddressⁿ)  // Standard is supported by using one of these contract addresses.

  SupportsResponse ::= (n : Byte²) (results: SupportResultⁿ)

Requirements
~~~~~~~~~~~~

- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST reject if any of the queries fail to produce a result.
- The result for querying the support of ``CIS-0`` for a smart contract implementing this specification MUST NOT be "Standard is not supported".
