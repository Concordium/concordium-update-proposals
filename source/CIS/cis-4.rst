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


.. _CIS-4-Bool:

``Bool``
^^^^^^^^

A boolean is serialized as a byte with value 0 for false and 1 for true::

  Bool ::= (0: Byte) // False
         | (1: Byte) // True


.. _CIS-4-CredentialID:

``CredentialID``
^^^^^^^^^^^^^^^^

A UUID v4 identifier represented as a ``u128`` unsigned integer number.

It is serialized as 16 bytes little endian::

  CredentialID ::= (id: Byte¹⁶)


.. _CIS-4-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL and optional checksum for metadata stored outside of this contract.

Serialized in the same way as :ref:`CIS-2 MetadataUrl<CIS-2-MetadataUrl>`.

.. _CIS-4-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex both unsigned 64 bit integers.

It is serialized as: First 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``) both little endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)


.. _CIS-4-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^

An address of an account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte³²)

.. _CIS-4-EntrypointName:

``EntrypointName``
^^^^^^^^^^^^^^^^^^

A name for a smart contract function entrypoint.

It is serialized as: First 2 bytes encode the length (``n``) of the entrypoint name, followed by this many bytes for the entrypoint name (``entrypoint``)::

  EntrypointName ::= (n: Byte²) (entrypoint: Byteⁿ)

.. _CIS-4-Timestamp:

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

.. _CIS-4-PublicKeyEd25519:

``PublicKeyEd25519``
^^^^^^^^^^^^^^^^^^^^

A public key represented as an 32-byte array.

It is serialized as 32 bytes::

  PublicKeyEd25519 ::= (key: Byte³²)

.. _CIS-4-SignatureEd25519:

``SignatureEd25519``
^^^^^^^^^^^^^^^^^^^^

Signature for an Ed25519 message.

It is serialized as 64 bytes::

  SignatureEd25519 ::= (signature: Byte⁶⁴)

.. _CIS-4-SigningData:

``SigningData``
^^^^^^^^^^^^^^^

Signing data contains a metadata for the signature that is used to check whether the signed message is designated for the right contract and entrypoint, and it is not expired.

It is serialized as :ref:`CIS-4-ContractAddress` (``contract_address``), :ref:`CIS-4-EntrypointName` (``entry_point``), :ref:`CIS-4-Nonce` (``nonce``), and :ref:`CIS-4-Timestamp` (``timestamp``)::

  SigningData ::= (contract_address: ContractAddress) (entry_point: EntrypointName) (nonce: Nonce) (timestamp: Timestamp)

.. _CIS-4-SchemaRef:

``SchemaRef``
^^^^^^^^^^^^^

A URL of the credential schema.

Serialized in the same way as :ref:`CIS-2 MetadataUrl<CIS-2-MetadataUrl>`.


.. _CIS-4-CredentialType:

``CredentialType``
^^^^^^^^^^^^^^^^^^

Is an short ASCII string (up to 256 characters) describing the credential type that is used to identify which schema the credential is based on.
It corresponds to a value of the ``name`` attribute of the credential schema.

It is serialized as: First byte encodes the length (``n``) of the credential type, followed by this many bytes for the credential type string::

  CredentialType ::= (n: Byte) (credential_type: Byteⁿ)

.. _CIS-4-Commitment:

``Commitment``
^^^^^^^^^^^^^^

A vector Pedersen commitment to the credential attributes.

It is serialized as: First 2 bytes encode the length (``n``) of the commitment, followed by this many bytes for the commitment data::

  Commitment ::= (n: Byte²) (commitment: Byteⁿ)

.. _CIS-4-CredentialInfo:

``CredentialInfo``
^^^^^^^^^^^^^^^^^^

Basic data for a verifiable credential.

It is serialized as a credential holder identifier :ref:`CIS-4-PublicKeyEd25519` (``holder_id``), a flag whether the credential can be revoked by the holder :ref:`CIS-4-Bool` (``holder_revocable``), a vector Pedersen commitment to the credential attributes :ref:`CIS-4-Commitment` (``commitment``), optional timestamps :ref:`CIS-4-Timestamp` from and until the credential is valid (``valid_from`` and ``valid_until``), and the credential type :ref:`CIS-4-CredentialType` (``credential_type``). Optional timestamps are serialized as 1 byte to indicate whether a timestamp is included, if its value is 0, then no timestamp present, if the value is 1 then the :ref:`CIS-4-Timestamp` bytes follow::

  OptionTimestamp ::= (0: Byte)
                    | (1: Byte) (timestamp: Timestamp)
  CredentialInfo ::= (holder_id: PublicKeyEd25519) (holder_revocable: Bool) (commitment: Commitment) (valid_from: OptionTimestamp) (valid_until: OptionTimestamp) (credential_type: CredentialType)

.. note::
  The timestamps ``valid_from`` and ``valid_until`` are optional. If ``valid_from`` is not included (indicated by the 0 value), then the credential is considered active immediately.
  If ``valid_until`` is not included, then the credential never expires.

.. _CIS-4-functions:

Contract functions
------------------

TBD

.. _CIS-4-functions-credentialEntry:

``credentialEntry``
^^^^^^^^^^^^^^^^^^^^^^^

Query a credential entry from the registry by ID.

Parameter
~~~~~~~~~

The parameter is the credential ID.

See the serialization rules in :ref:`CIS-4-CredentialID`.

Response
~~~~~~~~

The function returns a registry entry corresponding to the credential ID parameter.

It is serialized as :ref:`CIS-4-CredentialInfo` (``credential_info``) followed by a credential schema reference :ref:`CIS-4-SchemaRef` (``schema_ref``), and a credential entry revocation nonce :ref:`CIS-4-Nonce` (``revocation_nonce``)::

  CredentialQueryResponse ::= (credential_info: CredentialInfo) (schema_ref: SchemaRef) (revocation_nonce: Nonce)


Requirements
~~~~~~~~~~~~

- The query MUST fail if the credential ID is unknown.

.. _CIS-4-functions-credentialStatus:

``credentialStatus``
^^^^^^^^^^^^^^^^^^^^^^^^

Query the status of a credential from the credential registry by ID.

Parameter
~~~~~~~~~

The parameter is the credential ID.

See the serialization rules in :ref:`CIS-4-CredentialID`.

Response
~~~~~~~~

The function returns the status of a credential.

It is serialized as::

  CredentialStatus ::= (0: Byte) // Active
                     | (1: Byte) // Revoked
                     | (2: Byte) // Expired
                     | (3: Byte) // NotActivated

Requirements
~~~~~~~~~~~~

- The query MUST fail if the credential ID is unknown.

.. _CIS-4-functions-issuer:

``issuer``
^^^^^^^^^^

Query the issuer's account address.

Response
~~~~~~~~

The function output is the issuer's account address.
It is serialized as :ref:`CIS-4-AccountAddress`.

.. _CIS-4-functions-issuerMetadata:

``issuerMetadata``
^^^^^^^^^^^^^^^^^^

Query the current token metadata URLs for a list of token IDs.

Response
~~~~~~~~

The function output is the issuer's metadata URL.

It is serialized as :ref:`CIS-2-MetadataUrl`.

.. _CIS-4-functions-registerCredential:

``registerCredential``
^^^^^^^^^^^^^^^^^^^^^^

Register a credential with the given ID.

Parameter
~~~~~~~~~

The parameter is the credential ID and credential information that is used to create an entry in the registry.

It is serialized as :ref:`CIS-4-CredentialID` (``credential_id``) followed by :ref:`CIS-4-CredentialInfo` (``credential_info``)::

  RegisterCredentialParameter ::= (credential_id: CredentialID) (credential_info: CredentialInfo)

Requirements
~~~~~~~~~~~~

- The credential registration request MUST fail if the credential ID is already present in the registry.

.. _CIS-4-functions-revokeCredentialIssuer:

``revokeCredentialIssuer``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by the issuer's request.
The issuer is authorized to revoke the credential if the transaction sender's address is the same as the return value of :ref:`CIS-4-functions-issuer`.

Parameter
~~~~~~~~~

The parameter is the credential ID :ref:`CIS-4-CredentialID` and optional string indicating the revocation reason.

It is serialized as :ref:`CIS-4-CredentialID` followed by 1 byte to indicate whether a reason is included, if its value is 0, then no reason string present, if the value is 1 then the bytes corresponding to the reason string follow::

  OptionReason ::= (0: Byte)
                 | (1: Byte) (n: Byte) (reason_string: Byteⁿ)
  RevokeCredentialIssuerParam ::= (credential_id: CredentialID) (reason: OptionReason)

.. TODO: what kind of characters are allowed? ASCII, Unicode?

Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The registration MUST fail if the credential ID is not present in the registry.
- The registration MUST fail if the credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).

.. _CIS-4-functions-revokeCredentialHolder:

``revokeCredentialHolder``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by the holders's request.

The holder is authorized to revoke the credential by verifying the signature the holder's public key.
The public key is part of :ref:`CIS-4-CredentialInfo`.

Parameter
~~~~~~~~~

It is serialized as :ref:`CIS-4-CredentialID` (``credential_id``), metadata about the signature :ref:`CIS-4-SigningData` (``signing_data``), :ref:`CIS-4-SignatureEd25519` (``signature``), and optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`::

  RevokeCredentialHolderParam ::= (credential_id: CredentialID) (signing_data: SigningData) (signature: SignatureEd25519) (reason: OptionReason)


Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The registration MUST fail if the credential ID is not present in the registry.
- The registration MUST fail if the credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).


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


