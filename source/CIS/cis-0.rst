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

A standardized interface for how a smart contract publish supported smart contract standards.
Additionally, this specification includes a list of identifiers for each standard used on the Concordium blockchain.

The interface provides an entrypoint for querying the smart contract, whether it supports a given standard. The smart contract respond with either: not supported, supported or supported by some contract address.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

.. _CIS-0-standard-identifer:

Standard identifer
-------------------

Each standard MUST have a unique identifier.
An identifier is a variable-length ASCII string up to 100 characters.

The identifier is serialized as: first 1 byte for the length followed this many bytes for the ASCII encoding of the identifier::

  StandardIdentifier ::= (n: Byte) (id: Byteⁿ)


The table below contains the current standards and associated standard identifier.

.. list-table::
   :header-rows: 1
   :widths: 2 1 1

   * - Standard
     - Identifier
     - Encoded
   * - :ref:`CIS-0`
     - ``CIS-0``
     - ``054349532d30``
   * - :ref:`CIS-1`
     - ``CIS-1``
     - ``054349532d31``
   * - :ref:`CIS-2`
     - ``CIS-2``
     - ``054349532d32``

.. note::

   The above table will be expanded with identifers, when new standards are introduced.

Contract functions
------------------

A smart contract implementing CIS-0 MUST export the following function: :ref:`CIS-0-functions-supports`.

.. _CIS-0-functions-supports:

``supports``
^^^^^^^^^^^^

Query supported standards using a list of standard identifers.

Parameter
~~~~~~~~~

The parameter consists of a list of standard identifers.

It is serialized as: 2 bytes for the number (little endian) of standard identifers and then this number of :ref:`CIS-0-standard-identifer` identifers::

  SupportsParameter ::= (n : Byte²) (ids: StandardIdentifierⁿ)

Response
~~~~~~~~

The function output is a list of support results, where the order of the support results matches the order of standard identifers in the parameter.

It is serialized as: 2 bytes for the number (little endian) of results (``n``) and then this number of support results (``results``).
A support result is serialized as either: a byte with value ``0`` for "Standard is not supported", a byte with the value ``1`` for "Standard is support by this contract" or a byte with value ``2`` for "Standard is supported by using another contract address" followed by the contract address (``address``).
A contract address is serialized as: first 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``) both little endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)

  SupportResult ::= (0 : Byte)                             // Standard is not supported
                  | (1 : Byte)                             // Standard is supported by this contract
                  | (2 : Byte) (address: ContractAddress)  // Standard is supported by using another contract address

  SupportsResponse ::= (n : Byte²) (results: SupportResultⁿ)

Requirements
~~~~~~~~~~~~

- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The result for querying the support of ``CIS-0`` for a smart contract implementing this specification MUST NOT be "Standard is not supported".

