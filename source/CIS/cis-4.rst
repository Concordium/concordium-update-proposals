.. _CIS-4:

===================================
CIS-4: Credential Registry Standard
===================================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 1, 2023
   * - Standard identifier
     - ``CIS-4``


Abstract
========

A standard interface for verifiable credential (VC) registries.
A registry keeps track of public VC data and manages the VC lifecycle.
The interface provides functionality for the following roles of users:

- *credential issuers* can register and revoke credentials;
- *credential holders* can revoke holder-revocable credentials by signing a revocation message;
- *verifiers* can query credential status and data that are used to check a verifiable presentation requested from a holder;
- *revocation authorities* can revoke credentials by signing a revocation message.

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


.. _CIS-4-CredentialHolderId:

``CredentialHolderId``
^^^^^^^^^^^^^^^^^^^^^^

A credential identifier is a holder's public key.
It is expected that for each issued credential a unique public key is generated.
The credential identifier also identifies the credential holder and can be used to prove ownership of the credential by the holder.

It is serialized as :ref:`CIS-4-PublicKeyEd25519`.


.. _CIS-4-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL and optional checksum for metadata stored outside of this contract.

It is serialized as: 2 bytes for the length of the metadata url (``n``) and then this many bytes for the url to the metadata (``url``) followed by an optional checksum.
The checksum is serialized by 1 byte to indicate whether a hash of the metadata is included.
If its value is 0, then there is no hash, if the value is 1 then 32 bytes for a SHA256 hash (``hash``) follows::

  MetadataChecksum ::= (0: Byte)
                     | (1: Byte) (hash: Byte³²)

  MetadataUrl ::= (n: Byte²) (url: Byteⁿ) (checksum: MetadataChecksum)

.. _CIS-4-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex both unsigned 64-bit integers.

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
It consists of an unsigned 64-bit integer.

It is serialized as 8 bytes in little endian::

  Timestamp ::= (milliseconds: Byte⁸)

.. _CIS-4-Nonce:

``Nonce``
^^^^^^^^^

An unsigned 64-bit integer number that increases sequentially to protect against replay attacks.

It is serialized as 8 bytes in little endian::

  Nonce ::= (nonce: Byte⁸)

.. _CIS-4-PublicKeyEd25519:

``PublicKeyEd25519``
^^^^^^^^^^^^^^^^^^^^

A public key is represented as a 32-byte array.

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

It is serialized as :ref:`CIS-4-ContractAddress` (``contract_address``), :ref:`CIS-4-EntrypointName` (``entrypoint``), :ref:`CIS-4-Nonce` (``nonce``), and :ref:`CIS-4-Timestamp` (``timestamp``)::

  SigningData ::= (contract_address: ContractAddress) (entrypoint: EntrypointName) (nonce: Nonce) (timestamp: Timestamp)

.. _CIS-4-SchemaRef:

``SchemaRef``
^^^^^^^^^^^^^

A URL of the credential schema.

Serialized in the same way as :ref:`CIS-2 MetadataUrl<CIS-2-MetadataUrl>`.


.. _CIS-4-CredentialType:

``CredentialType``
^^^^^^^^^^^^^^^^^^

Is a short ASCII string (up to 256 characters) describing the credential type that is used to identify which schema the credential is based on.
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

It is serialized as a credential holder identifier :ref:`CIS-4-PublicKeyEd25519` (``holder_id``), a flag whether the credential can be revoked by the holder :ref:`CIS-4-Bool` (``holder_revocable``), a vector Pedersen commitment to the credential attributes :ref:`CIS-4-Commitment` (``commitment``), a :ref:`CIS-4-Timestamp` from which the credential is valid (``valid_from``), an optional :ref:`CIS-4-Timestamp` until which the credential is valid (``valid_until``), and the credential type :ref:`CIS-4-CredentialType` (``credential_type``). The optional timestamp is serialized as 1 byte to indicate whether a timestamp is included, if its value is 0, then no timestamp is present, if the value is 1 then the :ref:`CIS-4-Timestamp` bytes follow::

  OptionalTimestamp ::= (0: Byte)
                      | (1: Byte) (timestamp: Timestamp)
  CredentialInfo ::= (holder_id: CredentialHolderId) (holder_revocable: Bool) (commitment: Commitment) (valid_from: Timestamp)
                     (valid_until: OptionTimestamp) (credential_type: CredentialType) (metadata_url: MetadataUrl)

.. note::
  The timestamp ``valid_until`` is optional; if it is not included (indicated by the 0 value), then the credential never expires.


.. _CIS-4-CredentialStatus:

``CredentialStatus``
^^^^^^^^^^^^^^^^^^^^

The status of a verifiable credential.

It is serialized as 1 byte where ``0`` correponds to the status ``Active``, ``1`` corresponds to  ``Revoked``, ``2`` corresponds to  ``Expired``, ``3`` corresponds to ``NotActivated``::

  CredentialStatus ::= (0: Byte) // Active
                     | (1: Byte) // Revoked
                     | (2: Byte) // Expired
                     | (3: Byte) // NotActivated

.. _CIS-4-functions:

Contract functions
------------------

A smart contract implementing this standard MUST export the following functions:

- :ref:`CIS-4-functions-credentialEntry`
- :ref:`CIS-4-functions-credentialStatus`
- :ref:`CIS-4-functions-issuer`
- :ref:`CIS-4-functions-issuerMetadata`
- :ref:`CIS-4-functions-registerCredential`
- :ref:`CIS-4-functions-revokeCredentialIssuer`
- :ref:`CIS-4-functions-revokeCredentialHolder`
- :ref:`CIS-4-functions-revokeCredentialOther`
- :ref:`CIS-4-functions-registerRevocationKeys`
- :ref:`CIS-4-functions-removeRevocationKeys`
- :ref:`CIS-4-functions-revocationKeys`


.. _CIS-4-functions-credentialEntry:

``credentialEntry``
^^^^^^^^^^^^^^^^^^^

Query a credential entry from the registry by ID.

Parameter
~~~~~~~~~

The parameter is the credential ID.

See the serialization rules in :ref:`CIS-4-CredentialHolderId`.

Response
~~~~~~~~

The function returns a registry entry corresponding to the credential ID parameter.

It is serialized as :ref:`CIS-4-CredentialInfo` (``credential_info``) followed by a credential schema reference :ref:`CIS-4-SchemaRef` (``schema_ref``), and a credential entry revocation :ref:`CIS-4-Nonce` (``revocation_nonce``)::

  CredentialQueryResponse ::= (credential_info: CredentialInfo) (schema_ref: SchemaRef) (revocation_nonce: Nonce)


Requirements
~~~~~~~~~~~~

- The query MUST fail if the credential ID is not present in the registry.

.. _CIS-4-functions-credentialStatus:

``credentialStatus``
^^^^^^^^^^^^^^^^^^^^^^^^

Query the status of a credential from the credential registry by ID.

Parameter
~~~~~~~~~

The parameter is the credential ID.

See the serialization rules in :ref:`CIS-4-CredentialHolderId`.

Response
~~~~~~~~

The function returns the status of a credential.

See the serialization rules in :ref:`CIS-4-CredentialStatus`

Requirements
~~~~~~~~~~~~

- The query MUST fail if the credential ID is not present in the registry.
- The credential status MUST be ``Expired`` if the credential is not revoked, the field ``valid_until`` was present in :ref:`CIS-4-CredentialInfo` when registering the credential, and ``valid_until < now``.
- The credential status MUST NOT be ``Expired`` if the field ``valid_until`` was not present in :ref:`CIS-4-CredentialInfo` when registering the credential.
- The credential status MUST be ``Acive`` if the credential is not revoked, and does not qualify as ``Expired`` or ``NotActivated``.

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

It is serialized as :ref:`CIS-4-CredentialHolderId` (``credential_id``) followed by :ref:`CIS-4-CredentialInfo` (``credential_info``)::

  RegisterCredentialParameter ::= (credential_id: CredentialHolderId) (credential_info: CredentialInfo)

Requirements
~~~~~~~~~~~~

- The credential registration request MUST fail if the credential ID is already present in the registry.
- After successful registration:
    - Querying the credential by its ID with :ref:`CIS-4-functions-credentialEntry` MUST succeed.
    - Querying the credential status by ID with :ref:`CIS-4-functions-credentialStatus` MUST succeed and MUST NOT return ``Revoked`` (See the possible values for the status in :ref:`CIS-4-CredentialStatus`).

.. _CIS-4-functions-revokeCredentialIssuer:

``revokeCredentialIssuer``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by the issuer's request.
The issuer is authorized to revoke a credential if the transaction sender's address is the same as the return value of :ref:`CIS-4-functions-issuer`.

Parameter
~~~~~~~~~

The parameter is the credential ID :ref:`CIS-4-CredentialHolderId` and optional string indicating the revocation reason.

It is serialized as :ref:`CIS-4-CredentialHolderId` followed by 1 byte to indicate whether a reason is included.
If its value is 0, then no reason string is present, if the value is 1 then the bytes corresponding to the reason string follow::

  OptionalReason ::= (0: Byte)
                   | (1: Byte) (n: Byte) (reason_string: Byteⁿ)
  RevokeCredentialIssuerParam ::= (credential_id: CredentialHolderId) (reason: OptionReason)

.. TODO: what kind of characters are allowed? ASCII, Unicode?

Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The revocation MUST fail if:
    - The sender of the transaction is not the issuer.
    - The credential ID is not present in the registry.
    - The credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).

.. _CIS-4-functions-revokeCredentialHolder:

``revokeCredentialHolder``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by the holders's request.

The holder is authorized to revoke a credential by verifying the signature with the holder's public key.
It replaces the authorization based on checking the transaction sender address with signature verification.
The public key is part of :ref:`CIS-4-CredentialInfo` that is used when registering a credential with the :ref:`CIS-4-functions-registerCredential` entrypoint.

Parameter
~~~~~~~~~

It is serialized as :ref:`CIS-4-SignatureEd25519` (``signature``) and message data ``RevocationDataHolder`` consisting of :ref:`CIS-4-CredentialHolderId` (``credential_id``), metadata about the signature :ref:`CIS-4-SigningData` (``signing_data``), and an optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`::

  RevocationDataHolder ::= (credential_id: CredentialHolderId) (signing_data: SigningData) (reason: OptionReason)
  RevokeCredentialHolderParam ::= (signature: SignatureEd25519) (data : RevocationDataHolder)


Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The ``RevokeCredentialHolderParam``'s ``signing_data`` MUST include a nonce to protect against replay attacks.
  The holders's nonce is sequentially increased every time a revocation request is successfully executed.
  The function MUST only accept a ``RevokeCredentialHolderParam`` if it has the next nonce following the sequential order.
- The revocation MUST fail if:
    - The credential ID is not present in the registry.
    - The credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).
    - The credential is not holder-revocable.
    - The signature was intended for a different contract.
    - The signature was intended for a different entrypoint.
    - The signature is expired.
    - The signature cannot be validated.
      The smart contract logic SHOULD practice its best efforts to ensure that only the holder can generate and authorize a revocation request with a valid signature.

.. _CIS-4-functions-revokeCredentialOther:

``revokeCredentialOther``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by a revocation authority request.
A revocation authority is any entity that holds a private key corresponding to the public key registered by the issuer.
A revocation authority is authorized to revoke a credential by verifying the signature with the public key of the given identifier.

This entrypoint gives a general way of adding revocation rights to external entities.
It replaces the authorization checks conducted on the ``sender/invoker`` variable with signature verification.
In particular, it enables the issuer to provide a service for selected entities to revoke credentials without paying for revocation transactions.


Parameter
~~~~~~~~~

It is serialized as :ref:`CIS-4-SignatureEd25519` (``signature``) and message data ``RevocationDataOther`` consisting of :ref:`CIS-4-CredentialHolderId` (``credential_id``), metadata about the signature :ref:`CIS-4-SigningData` (``signing_data``), a revocation public key :ref:`CIS-4-PublicKeyEd25519` , and an optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`::

  RevocationDataOther ::= (credential_id: CredentialHolderId) (signing_data: SigningData) (revocation_key: PublicKeyEd25519) (reason: OptionReason)
  RevokeCredentialHolderParam ::= (signature: SignatureEd25519) (data : RevocationDataOther)


Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The ``RevokeCredentialOtherParam``'s ``signing_data`` MUST include a nonce to protect against replay attacks.
  The revocation authority's nonce is sequentially increased every time a revocation request is successfully executed.
  The function MUST only accept a ``RevokeCredentialOtherParam`` if it has the next nonce following the sequential order.
- The revocation MUST fail if:
    - The credential ID is not present in the registry.
    - The revocation key in not present in the registry.
    - The credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).
    - The signature was intended for a different contract.
    - The signature was intended for a different entrypoint.
    - The signature is expired.
    - The signature can not be validated.
      The smart contract logic SHOULD practice its best efforts to ensure that only the revocation authority can generate and authorize a revocation request with a valid signature.

.. _CIS-4-functions-registerRevocationKeys:

``registerRevocationKeys``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Register public keys that can be used by revocation authorities.

Parameter
~~~~~~~~~

It is serialized as First 2 bytes encode the length (``n``) the vector of kesy, followed by this many :ref:`CIS-4-PublicKeyEd25519` keys::

  RegisterPublicKeyParameters ::= (n: Byte²) (key: PublicKeyEd25519)ⁿ

Requirements
~~~~~~~~~~~~

- The revocation MUST fail if:
    - The sender of the transaction is not the issuer.
    - Some of the keys are already registered.
- The smart contract MUST prevent resetting the nonce associated with a public key.
  For example, the contract logic could keep track of all keys seen by the contract and avoid reusing the same keys even after the keys were made unavailable by calling :ref:`CIS-4-functions-removeRevocationKeys`.


.. _CIS-4-functions-removeRevocationKeys:

``removeRevocationKeys``
^^^^^^^^^^^^^^^^^^^^^^^^

Make a list of public keys unavailable to revocation authorities.

Parameter
~~~~~~~~~

It is serialized as: First 2 bytes encode the length (``n``) the vector of keys, followed by this many :ref:`CIS-4-PublicKeyEd25519` keys::

  RegisterPublicKeyParameters ::= (n: Byte²) (key: PublicKeyEd25519)ⁿ

Requirements
~~~~~~~~~~~~

- The revocation MUST fail if:
    - The sender of the transaction is not the issuer.
    - Some of the keys are not present in the registry.


.. _CIS-4-functions-revocationKeys:

``revocationKeys``
^^^^^^^^^^^^^^^^^^

Query revocation keys.

Response
~~~~~~~~

The function output a list of available revocation keys.
Valid signatures with the corresponding private keys can be used to revoke any credential in the registry.

It is serialized as: First 2 bytes encode the length (``n``) the vector of keys, followed by this many :ref:`CIS-4-PublicKeyEd25519` keys::

  RegisterPublicKeyParameters ::= (n: Byte²) (key: PublicKeyEd25519)ⁿ


Logged events
-------------

The events defined by this specification are serialized using one byte to discriminate the different events.
A custom event SHOULD NOT have a first byte colliding with any of the events defined by this specification.

.. _CIS-4-register-credential-transfer:

``RegisterCredentialEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``RegisterCredentialEvent`` event MUST be logged when a new credential is registered.
The event records the credential identifier, the credential type, and the corresponding schema reference.

The ``RegisterCredentialEvent`` event is serialized as: first a byte with the value of 255, followed by :ref:`CIS-4-CredentialHolderId` (``crednetial_id``), a reference to the credential schema :ref:`CIS-4-SchemaRef` (``schema_ref``), a credential type :ref:`CIS-4-CredentialType` (``credential_type``) ::

  CredentialEventData ::= (credential_id: CredentialHolderId) (schema_ref: SchemaRef) (credential_type: CredentialType)
  RegisterCredentialEvent ::= (249: Byte) (data: CredentialEventData)

``RevokeCredentialEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^

A ``RevokeCredentialEvent`` event MUST be logged when a credential is revoked.
The event records the credential identifier, who requested the revocation (the holder, the issuer or a revocation authority), and an optional string with a short comment on the revocation reason.

The ``RevokeCredentialEvent`` event is serialized as: first a byte with the value of 254, followed by :ref:`CIS-4-CredentialHolderId` (``crednetial_id``), a ``revoker``, and an optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`; ``revoker`` is serialized as 1 byte to indicate who sent the revocation request ( 0 - issuer, 1 - holder, 2 -revocation authority); if the first byte is 2, then it is followed by a public key :ref:`CIS-4-PublicKeyEd25519` of the revoker::

  Revoker ::= (0: Byte)                         // Issuer
            | (1: Byte)                         // Holder
            | (2: Byte) (key: PublicKeyEd25519) // Other
  RevokeCredentialEvent ::= (248: Byte) (credential_id: CredentialHolderId) (revoker: Revoker) (reason: OptionReason)


.. _CIS-4-events-IssuerMetadata:

``IssuerMetadata``
^^^^^^^^^^^^^^^^^^

A ``IssuerMetadata`` event MUST be logged when setting the metadata url of the issuer.
This also applies to contract initialization.
It consists of a URL for the location of the metadata for the issuer with an optional SHA256 checksum of the content.

The ``IssuerMetadata`` event is serialized as: first a byte with the value of 253, followed by :ref:`CIS-2-MetadataUrl` (``metadata``)::

  IssuerMetadata ::= (247: Byte) (metadata: MetadataUrl)


.. _CIS-4-events-CredentialMetadataEvent:

``CredentialMetadataEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``CredentialMetadataEvent`` event MUST be logged when updating the credential metadata.
It consist of a credential ID and a URL for the location of the metadata for this credential with an optional SHA256 checksum of the content.

The ``CredentialMetadataEvent`` event is serialized as: first a byte with the value of 252, followed by the token ID :ref:`CIS-2-TokenID` (``id``), and then a :ref:`CIS-2-MetadataUrl` (``metadata``)::

  CredentialMetadataEvent ::= (246: Byte) (id: CredentialHolderId) (metadata: MetadataUrl)


``CredentialSchemaRefEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``CredentialSchemaRefEvent`` event MUST be logged when updating the credential schema reference for a credential type.
It consist of a credential type and a URL for the location of the schema for this credential with an optional SHA256 checksum of the content.

The ``CredentialSchemaRefEvent`` event is serialized as: first a byte with the value of 251, followed by the token ID :ref:`CIS-4-CredentialType` (``type``), and then a :ref:`CIS-4-SchemaRef` (``schema_ref``)::

  CredentialSchemaRefEvent ::= (245: Byte) (type: CredentialType) (schema_ref: SchemaRef)

``RevocationKeyEvent``
^^^^^^^^^^^^^^^^^^^^^^

A ``RevocationKeyEvent`` event MUST be logged when registering a new or removing an existing revocation key.
It consist of the key and the action performed with the key (registration or removal).

The ``RevocationKeyEvent`` event is serialized as: first a byte with the value of 250, followed by the key bytes :ref:`CIS-4-PublicKeyEd25519` and 1 byte encoding the action (0 for ``Register``, 1 for ``Remove``)::

  RevocationKeyAction ::= (0: Byte)    // Register
                        | (1: Byte)    // Remove
  RevocationKeyEvent ::= (244: Byte) (action: RevocationKeyAction)


.. _CIS-4-issuer-metadata-json:

Issuer metadata JSON
--------------------

The issuer metadata is stored off-chain and MUST be a JSON (:rfc:`8259`) file.

All of the fields in the JSON file are optional, and this specification reserves a number of field names, shown in the table below.

.. list-table:: Issuer metadata JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``name``
    - string
    - The name to display for the issuer.
  * - ``description`` (optional)
    - string
    - A description for the issuer.
  * - ``thumbnail`` (optional)
    - URL JSON object
    - An image URL to a small image for displaying the issuer.
  * - ``display`` (optional)
    - URL JSON object
    - An image URL to a large image for displaying the issuer.
  * - ``url``
    - string (:rfc:`3986`) [``uri-reference``]
    - A URL of the issuer's website.
  * - ``attributes`` (optional)
    - JSON array of Attribute JSON objects
    - Assign a number of attributes to the issuer.
      Attributes can be used to include extra information about the issuer.

Optionally a SHA256 hash of the JSON file can be logged with the :ref:`CIS-4-events-IssuerMetadata` event for checking integrity.
Since the metadata JSON file could contain URLs, a SHA256 hash can optionally be associated with the URL.
To associate a hash with a URL the JSON value is an object:

.. list-table:: URL JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``url``
    - string (:rfc:`3986`) [``uri-reference``]
    - A URL.
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
    - Value of the attribute.


Example issuer metadata
^^^^^^^^^^^^^^^^^^^^^^^

TBD

Credential metadata JSON
------------------------

The credential metadata is stored off-chain and MUST be a JSON (:rfc:`8259`) file.

.. list-table:: Credential metadata JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``title``
    - string
    - The name to display for the credential.
  * - ``logo``
    - URL JSON object
    - An image URL for displaying the credential.
  * - ``background_color``
    - string
    - A hex code of the background color for displaying the credential.
  * - ``image`` (optional)
    - URL JSON object
    - A background image URL for displaying the credential.

Where URL JSON object the same as in :ref:`CIS-4-events-IssuerMetadata`.

Optionally a SHA256 hash of the JSON file can be logged with the :ref:`CIS-4-events-CredentialMetadataEvent` event for checking integrity.


Example credential metadata
^^^^^^^^^^^^^^^^^^^^^^^^^^^

TBD

.. _CIS-4-smart-contract-limitations:

Smart contract limitations
==========================

A number of limitations are important to be aware of:

- The byte size of smart contract function parameters are limited to at most 65535 B.
- Each logged event is limited to 0.5 KiB.
- The total size of the smart contract module is limited to 512 KiB.


