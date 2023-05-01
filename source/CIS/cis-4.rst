.. _CIS-4:

===================================
CIS-4: Credential Registry Standard
===================================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 1, 2023

Abstract
========

A standard interface for verified credential registries.
The interface provides entrypoints for managing credentials (register, update, revoke), querying credential status and data.
The interface also includes entrypoints for managing of public keys for the issuer and revocation authorities

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

General types and serialization
-------------------------------

``Bool``
^^^^^^^^

A boolean is serialized as a byte with value 0 for false and 1 for true::

  Bool ::= (0: Byte) // False
         | (1: Byte) // True


.. _CIS-4-CredentialID:

``CredentialID``
^^^^^^^^^^^^^^^^

A UUID v4 identifier represented as a `u128` unsigned integer number.

It is serialized as little endian 16 bytes::

  CredentialID ::= (id: Byte¹⁶)


.. _CIS-4-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL or DID and optional checksum for metadata stored outside of this contract.

Serialized in the same way as CIS-2 :ref:`CIS-2 MetadataUrl<CIS-2-MetadataUrl>`.

``Timestamp``
^^^^^^^^^^^^^

A timestamp given in milliseconds since unix epoch.
It consists of an unsigned 64 bit integer.

It is serialized as 8 bytes in little endian::

  Timestamp ::= (milliseconds: Byte⁸)

.. _CIS-4-Nonce:

``Nonce``
^^^^^^^^^

An unsigned 64 bit integer number that increases sequentially to protect against replay attacks.

It is serialized as 8 bytes in little endian::

  Nonce ::= (nonce: Byte⁸)

.. _CIS-4-SignatureEd25519:

``SignatureEd25519``
^^^^^^^^^^^^^^^^^^^^

Signature for an Ed25519 message.

It is serialized as 64 bytes::

  SignatureEd25519 ::= (signature: Byte⁶⁴)


.. _CIS-4-functions:

Contract functions
------------------

TBD

.. _CIS-4-functions-viewCredentialEntry:

``viewCredentialEntry``
^^^^^^^^^^^^^^^^^^^^^^^

Query a credential entry from the registry by ID.

Parameter
~~~~~~~~~

The parameter is the credential ID.

See the serialization rules in :ref:`CIS-4-CredentialID`.


Response
~~~~~~~~

The function returns a registry entry corresponding to the credential ID parameter.

It is serialized as::

  Commitment ::= (n: Byte²) (commitment_bytes: Byteⁿ)
  OptionTimestamp ::= (0: Byte)
                    | (1: Byte) (timestamp: Timestamp)
  CredentialEntryResponse ::= (holder_id: CredentialID) (holder_revocable: Bool) (commitment: Commitment) (schema_ref: MetadataUrl) (valid_from: OptionTimestamp)  (valid_from: OptionTimestamp) (revocation_nonce: Nonce)


Requirements
~~~~~~~~~~~~

.. _CIS-4-functions-viewCredentialStatus:

``viewCredentialStatus``
^^^^^^^^^^^^^^^^^^^^^^^^

Query the status of a credential from the credential registry by ID.

It is serialized as::

  CredentialStatus ::= (0: Byte) // Active
                     | (1: Byte) // Revoked
                     | (2: Byte) // Expired
                     | (3: Byte) // NotActivated

Parameter
~~~~~~~~~

The parameter is the credential ID.

See the serialization rules in :ref:`CIS-4-CredentialID`.

Response
~~~~~~~~



Requirements
~~~~~~~~~~~~

``viewIssuerMetadata``
^^^^^^^^^^^^^^^^^^^^^^

Query the current token metadata URLs for a list of token IDs.

Parameter
~~~~~~~~~


Response
~~~~~~~~

The function output is the issuer's metadata URL.

It is serialized as :ref:`CIS-2-MetadataUrl`.

Logged events
-------------

A custom event SHOULD NOT have a first byte colliding with any of the events defined by this specification.

.. _CIS-4-register-credential-transfer:

``RegisterCredentialEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``RegisterCredentialEvent`` event MUST be logged when a new credential is issued.

The ``RegisterCredentialEvent`` event is serialized as: TBD::

  RegisterCredentialEvent ::= ...


Rejection errors
----------------

A smart contract following this specification MAY reject using the following error codes:

.. list-table::
  :header-rows: 1

  * - Name
    - Error code
    - Description
  * - TBD
    - TBD
    - TBD


Rejecting using an error code from the table above MUST only occur in a situation as described in the corresponding error description.

The smart contract implementing this specification MAY introduce custom error codes other than the ones specified in the table above.


Issuer metadata JSON
--------------------

The token metadata is stored off-chain and MUST be a JSON (:rfc:`8259`) file.

All of the fields in the JSON file are optional, and this specification reserves a number of field names, shown in the table below.

.. list-table:: Issuer metadata JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``name`` (optional)
    - string
    - The name to display for the issuer.
  * - ``symbol`` (optional)
    - string
    - Short text to display for the issuer.
  * - ``description`` (optional)
    - string
    - A description for the issuer.
  * - ``thumbnail`` (optional)
    - URL JSON object
    - An image URL to a small image for displaying the issuer.
  * - ``display`` (optional)
    - URL JSON object
    - An image URL to a large image for displaying the issuer.
  * - ``attributes`` (optional)
    - JSON array of Attribute JSON objects
    - Assign a number of attributes to the issuer.
      Attributes can be used to include extra information about the issuer.

Optionally a SHA256 hash of the JSON file can be logged with the TokenMetadata event for checking integrity.
Since the metadata JSON file could contain URLs, a SHA256 hash can optionally be associated with the URL.
To associate a hash with a URL the JSON value is an object:

.. list-table:: URL JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``url``
    - string (:rfc:`3986`, DID) [``uri-reference``]
    - A URL or DID.
  * - ``hash`` (optional)
    - string
    - A SHA256 hash of the URL content encoded as a hex string.

Attributes are objects with the following fields:

.. list-table:: Attribute JSON object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``type``
    - string
    - Type for the value field of the attribute.
  * - ``name``
    - string
    - Name of the attribute.
  * - ``value``
    - string
    - Value of the attrbute.


Example issuer metadata
^^^^^^^^^^^^^^^^^^^^^^^

TBD

.. _CIS-4-smart-contract-limitations:

Smart contract limitations
==========================

A number of limitations are important to be aware of:

- The byte size of smart contract function parameters are limited to at most 65535 B.
- Each logged event is limited to 0.5 KiB.
- The total size of the smart contract module is limited to 512 KiB.


