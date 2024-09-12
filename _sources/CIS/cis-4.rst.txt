.. _CIS-4:

===================================
CIS-4: Credential Registry Standard
===================================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 1, 2023
   * - Final
     - Oct 8, 2023
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-4``


Abstract
========

A standard interface for verifiable credential (VC) registries.
A registry keeps track of public VC data and manages the VC lifecycle.
The interface provides functionality for the following roles of users:

- *credential issuers* can register and revoke credentials, manage revocation keys;
- *credential holders* can revoke holder-revocable credentials by signing a revocation message;
- *verifiers* can query credential status and data that are used to check a verifiable presentation requested from a holder;
- *revocation authorities* can revoke credentials by signing a revocation message.

Each contract instance MUST store public data of VCs of the same type.
A *credential type* is a string that corresponds to the name of the VCs JSON schema, which credentials in the registry are based on.
A `VCs JSON schema <https://w3c.github.io/vc-json-schema/>`_ is a `JSON schema <http://json-schema.org/>`_ describing the attributes of a VC.
Attributes are sequentially numbered and have their numbers recorded in the additional ``index`` field in the schema.
See VC schema examples and the corresponding VCs `here <https://github.com/Concordium/concordium-web3id/tree/main/examples/json-schemas>`_.

.. TODO: refer to the Concordium VC Data Model documentation, once we have it.

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

It is serialized as: 2 bytes for the length (``n``) of the metadata URL in little-endian and then this many bytes for the URL to the metadata (``url``) followed by an optional checksum.
The checksum is serialized by 1 byte to indicate whether a hash of the metadata is included.
If its value is 0, then there is no hash; if the value is 1, then 32 bytes for a SHA256 hash (``hash``) follows::

  MetadataChecksum ::= (0: Byte)
                     | (1: Byte) (hash: Byte³²)

  MetadataUrl ::= (n: Byte²) (url: Byteⁿ) (checksum: MetadataChecksum)

.. _CIS-4-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex, both unsigned 64-bit integers.

It is serialized as: First 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``), both little-endian::

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

It is serialized as: First 2 bytes encode the length (``n``) of the entrypoint name in little-endian, followed by this many bytes for the entrypoint name (``entrypoint``)::

  EntrypointName ::= (n: Byte²) (entrypoint: Byteⁿ)

.. _CIS-4-Timestamp:

``Timestamp``
^^^^^^^^^^^^^

A timestamp given in milliseconds since Unix epoch.
It consists of an unsigned 64-bit integer.

It is serialized as 8 bytes in little-endian::

  Timestamp ::= (milliseconds: Byte⁸)

.. _CIS-4-Nonce:

``Nonce``
^^^^^^^^^

An unsigned 64-bit integer number that increases sequentially to protect against replay attacks.

It is serialized as 8 bytes in little-endian::

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

Signing data contains metadata for the signature that is used to check whether the signed message is designated for the correct contract and entrypoint, and that it is not expired.

It is serialized as :ref:`CIS-4-ContractAddress` (``contract_address``), :ref:`CIS-4-EntrypointName` (``entrypoint``), :ref:`CIS-4-Nonce` (``nonce``), and :ref:`CIS-4-Timestamp` (``timestamp``)::

  SigningData ::= (contract_address: ContractAddress) (entrypoint: EntrypointName) (nonce: Nonce) (timestamp: Timestamp)

.. _CIS-4-SchemaRef:

``SchemaRef``
^^^^^^^^^^^^^

A URL of the credential schema.

Serialized in the same way as :ref:`CIS-4-MetadataUrl`.


.. _CIS-4-CredentialType:

``CredentialType``
^^^^^^^^^^^^^^^^^^

A short string (up to 256 characters) in UTF-8 encoding.
The string describes the credential type that is used to identify which schema the credential is based on.
It corresponds to a value of the ``name`` attribute of the credential schema.

It is serialized as: First byte encodes the length (``n``) of the credential type, followed by this many bytes for the credential type string::

  CredentialType ::= (n: Byte) (credential_type: Byteⁿ)

.. _CIS-4-CredentialInfo:

``CredentialInfo``
^^^^^^^^^^^^^^^^^^

Basic data for a verifiable credential.

It is serialized as a credential holder identifier :ref:`CIS-4-PublicKeyEd25519` (``holder_id``), a flag whether the credential can be revoked by the holder :ref:`CIS-4-Bool` (``holder_revocable``), a :ref:`CIS-4-Timestamp` from which the credential is valid (``valid_from``), an optional :ref:`CIS-4-Timestamp` until which the credential is valid (``valid_until``), and a reference to the credential metadata :ref:`CIS-4-MetadataUrl` (``metadata_url``).
The optional timestamp is serialized as 1 byte to indicate whether a timestamp is included. If its value is 0, then no timestamp is present; if the value is 1, then the :ref:`CIS-4-Timestamp` bytes follow::

  OptionalTimestamp ::= (0: Byte)
                      | (1: Byte) (timestamp: Timestamp)
  CredentialInfo ::= (holder_id: CredentialHolderId) (holder_revocable: Bool) (valid_from: Timestamp)
                     (valid_until: OptionTimestamp) (metadata_url: MetadataUrl)

.. note::
  The timestamp ``valid_until`` is optional; if it is not included (indicated by the 0 tag), then the credential never expires.


.. _CIS-4-CredentialStatus:

``CredentialStatus``
^^^^^^^^^^^^^^^^^^^^

The status of a verifiable credential.

It is serialized as 1 byte where ``0`` correponds to the status ``Active``, ``1`` corresponds to  ``Revoked``, ``2`` corresponds to  ``Expired``, ``3`` corresponds to ``NotActivated``::

  CredentialStatus ::= (0: Byte) // Active
                     | (1: Byte) // Revoked
                     | (2: Byte) // Expired
                     | (3: Byte) // NotActivated

See requirements for :ref:`CIS-4-functions-credentialStatus` for details of how statuses are returned.

.. _CIS-4-functions:

Contract functions
------------------

A smart contract implementing this standard MUST export the following functions:

- :ref:`CIS-4-functions-credentialEntry`
- :ref:`CIS-4-functions-credentialStatus`
- :ref:`CIS-4-functions-issuer`
- :ref:`CIS-4-functions-registryMetadata`
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
- The credential status MUST be ``NotActivated`` if ``now < valid_from``, where ``valid_from`` is the corresponding value from :ref:`CIS-4-CredentialInfo` provided when registering the credential.
- The credential status MUST be ``Acive`` if the credential is not revoked, and does not qualify as ``Expired`` or ``NotActivated``.

.. _CIS-4-functions-issuer:

``issuer``
^^^^^^^^^^

Query the issuer's public key.
The corresponding private key is used to sign the public part of verifiable credentials issued by this issuer.

Response
~~~~~~~~

The function output is the issuer's public key.
It is serialized as :ref:`CIS-4-PublicKeyEd25519`.

.. _CIS-4-functions-registryMetadata:

``registryMetadata``
^^^^^^^^^^^^^^^^^^^^

Query the registry's metadata.

Response
~~~~~~~~

The function output is the issuer's metadata URL, the credential type and schema for the credentials stored in the registry.

It is serialized as the issuer's :ref:`CIS-4-MetadataUrl` (``issuer_metadata``) followed by the credential type of the registry :ref:`CIS-4-CredentialType` (``credential_type``) and the corresponding credential JSON schema reference :ref:`CIS-4-SchemaRef` (``credential_schema``)::

  MetadataResponse := (issuer_metadata: MetadataUrl) (credential_type: CredentialType) (credential_schema: SchemaRef)

.. _CIS-4-functions-registerCredential:

``registerCredential``
^^^^^^^^^^^^^^^^^^^^^^

Register public data for a new credential.

.. note::
  This standard does not specify how the issuer is authenticated.
  Implementations can use various mechanisms.
  For example, the transaction sender's address is checked against the issuer's account address stored in the contract.
  Another option is to use ``auxiliary_data`` to implement a signature-based authentication mechanism.

Parameter
~~~~~~~~~

The parameter is credential information that is used to create an entry in the registry.

It is serialized as :ref:`CIS-4-CredentialInfo` (``credential_info``), followed by auxiliary data, which is serialized as 2 bytes to encode the length (``n``) of the vector of keys in little-endian, followed by this many bytes of data::

  AuxData ::= (n: Byte²) (data: Byteⁿ)
  RegisterCredentialParameter ::= (credential_info: CredentialInfo) (auxiliary_data: AuxData)

Requirements
~~~~~~~~~~~~

- The credential registration request MUST fail if the credential ID is already present in the registry.
- After successful registration, querying the credential by its ID with :ref:`CIS-4-functions-credentialEntry` MUST succeed.

.. _CIS-4-functions-revokeCredentialIssuer:

``revokeCredentialIssuer``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by the issuer's request.

.. note::
  This standard does not specify how the issuer is authenticated.
  Implementations can use various mechanisms.
  For example, the transaction sender's address is checked against the issuer's account address stored in the contract.
  Another option is to use ``auxiliary_data`` to implement a signature-based authentication mechanism.

Parameter
~~~~~~~~~

The parameter is the credential ID :ref:`CIS-4-CredentialHolderId` and an optional string in the UTF-8 encoding that indicates the revocation reason.

It is serialized as :ref:`CIS-4-CredentialHolderId` followed by 1 byte to indicate whether a reason is included.
If its value is 0, then no reason string is present; if the value is 1, then the bytes corresponding to the reason string follow.
The optional revocation reason is followed by auxiliary data, which is serialized as 2 bytes to encode the length (``n``) of the data vector in little-endian, followed by this many bytes of data::

  OptionalReason ::= (0: Byte)
                   | (1: Byte) (n: Byte) (reason_string: Byteⁿ)
  AuxData ::= (n: Byte²) (data: Byteⁿ)
  RevokeCredentialIssuerParam ::= (credential_id: CredentialHolderId) (reason: OptionalReason) (auxiliary_data: AuxData)

Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The revocation MUST fail if any of the following conditions are met:
    - The credential ID is not present in the registry.
    - The credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).

.. _CIS-4-functions-revokeCredentialHolder:

``revokeCredentialHolder``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Revoke a credential by the holders's request.

The holder is authorized to revoke a credential by verifying the signature with the holder's public key.
The public key is part of :ref:`CIS-4-CredentialInfo` that is used when registering a credential with the :ref:`CIS-4-functions-registerCredential` entrypoint.

Parameter
~~~~~~~~~

It is serialized as :ref:`CIS-4-SignatureEd25519` (``signature``) and data, for which the signature is computed ``RevocationDataHolder`` (``message``), consisting of :ref:`CIS-4-CredentialHolderId` (``credential_id``), metadata about the signature :ref:`CIS-4-SigningData` (``signing_data``), and an optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`::

  RevocationDataHolder ::= (credential_id: CredentialHolderId) (signing_data: SigningData) (reason: OptionalReason)
  RevokeCredentialHolderParam ::= (signature: SignatureEd25519) (message : RevocationDataHolder)

Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The message to be signed MUST be produced in the following way:
    - Start with the bytes of the domain separation string ``WEB3ID:REVOKE``.
    - Append ``RevocationDataHolder`` bytes from the input parameter.
- The ``RevokeCredentialHolderParam``'s ``signing_data`` MUST include a nonce to protect against replay attacks.
  The holders's nonce is sequentially increased every time a revocation request is successfully executed.
  The function MUST only accept a ``RevokeCredentialHolderParam`` if it has the next nonce following the sequential order.
- The revocation MUST fail if any of the following conditions are met:
    - The credential ID is not present in the registry.
    - The credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).
    - The credential is not holder-revocable.
    - The signature was intended for a different contract.
    - The signature was intended for a different entrypoint.
    - The signature is expired.
    - The signature cannot be validated.
      The smart contract logic SHOULD use all possible efforts to ensure that only the holder can generate and authorize a revocation request with a valid signature.

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

It is serialized as :ref:`CIS-4-SignatureEd25519` (``signature``) and data, for which the signature is computed ``RevocationDataHolder`` (``message``) consisting of :ref:`CIS-4-CredentialHolderId` (``credential_id``), metadata about the signature :ref:`CIS-4-SigningData` (``signing_data``), a revocation public key :ref:`CIS-4-PublicKeyEd25519` , and an optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`::

  RevocationDataOther ::= (credential_id: CredentialHolderId) (signing_data: SigningData) (revocation_key: PublicKeyEd25519) (reason: OptionalReason)
  RevokeCredentialHolderParam ::= (signature: SignatureEd25519) (message : RevocationDataOther)

Requirements
~~~~~~~~~~~~

- If revoked successfully, the credential status MUST change to ``Revoked`` (see :ref:`CIS-4-functions-credentialStatus`).
- The message to be signed MUST be produced in the following way:
    - Start with the bytes of the domain separation string ``WEB3ID:REVOKE``.
    - Append ``RevocationDataOther`` bytes from the input parameter.
- The ``RevokeCredentialOtherParam``'s ``signing_data`` MUST include a nonce to protect against replay attacks.
  The revocation authority's nonce is sequentially increased every time a revocation request is successfully executed.
  The function MUST only accept a ``RevokeCredentialOtherParam`` if it has the next nonce following the sequential order.
- The revocation MUST fail if any of the following conditions are met:
    - The credential ID is not present in the registry.
    - The revocation key in not present in the registry.
    - The credential status is not one of ``Active`` or ``NotActivated`` (see :ref:`CIS-4-functions-credentialStatus`).
    - The signature was intended for a different contract.
    - The signature was intended for a different entrypoint.
    - The signature is expired.
    - The signature can not be validated.
      The smart contract logic SHOULD use all possible efforts to ensure that only the revocation authority can generate and authorize a revocation request with a valid signature.

.. _CIS-4-functions-registerRevocationKeys:

``registerRevocationKeys``
^^^^^^^^^^^^^^^^^^^^^^^^^^

Register public keys that can be used by revocation authorities.

.. note::
  This standard does not specify how the issuer is authenticated.
  Implementations can use various mechanisms.
  For example, the transaction sender's address is checked against the issuer's account address stored in the contract.
  Another option is to use ``auxiliary_data`` to implement a signature-based authentication mechanism.

Parameter
~~~~~~~~~

It is serialized as First 2 bytes encode the length (``n``) of the vector of keys in little-endian, followed by this many :ref:`CIS-4-PublicKeyEd25519` keys.
The revocation keys are followed by auxiliary data, which is serialized as 2 bytes to encode the length (``m``) of the data vector in little-endian, followed by this many bytes of data::

  AuxData ::= (m: Byte²) (data: Byteᵐ)
  RegisterPublicKeyParameters ::= (n: Byte²) (key: PublicKeyEd25519)ⁿ (auxiliary_data: AuxData)

Requirements
~~~~~~~~~~~~

- The revocation MUST fail if some of the keys are already registered.
- The smart contract MUST prevent resetting the nonce associated with a public key.
  For example, the contract logic could keep track of all keys seen by the contract and avoid reusing the same keys even after the keys were made unavailable by calling :ref:`CIS-4-functions-removeRevocationKeys`.


.. _CIS-4-functions-removeRevocationKeys:

``removeRevocationKeys``
^^^^^^^^^^^^^^^^^^^^^^^^

Make a list of public keys unavailable to revocation authorities.

.. note::
  This standard does not specify how the issuer is authenticated.
  Implementations can use various mechanisms.
  For example, the transaction sender's address is checked against the issuer's account address stored in the contract.
  Another option is to use ``auxiliary_data`` to implement a signature-based authentication mechanism.

Parameter
~~~~~~~~~

It is serialized as: First 2 bytes encode the length (``n``) of the vector of keys in little-endian, followed by this many :ref:`CIS-4-PublicKeyEd25519` keys.
The revocation keys are followed by auxiliary data, which is serialized as 2 bytes to encode the length (``m``) of the data vector in little-endian, followed by this many bytes of data::

  AuxData ::= (m: Byte²) (data: Byteᵐ)
  RegisterPublicKeyParameters ::= (n: Byte²) (key: PublicKeyEd25519)ⁿ (auxiliary_data: AuxData)

Requirements
~~~~~~~~~~~~

- The revocation MUST fail if some of the keys are not present in the registry.


.. _CIS-4-functions-revocationKeys:

``revocationKeys``
^^^^^^^^^^^^^^^^^^

Query revocation keys.

Response
~~~~~~~~

The function outputs a list of available revocation keys.
Valid signatures with the corresponding private keys can be used to revoke any credential in the registry.

It is serialized as: First 2 bytes encode the length (``n``) of the vector of keys in little-endian, followed by this many :ref:`CIS-4-PublicKeyEd25519` keys::

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

The ``RegisterCredentialEvent`` event is serialized as: first a byte with the value of 249, followed by :ref:`CIS-4-CredentialHolderId` (``crednetial_id``), a reference to the credential schema :ref:`CIS-4-SchemaRef` (``schema_ref``), a credential type :ref:`CIS-4-CredentialType` (``credential_type``), and a reference to the credential metadata :ref:`CIS-4-MetadataUrl` (``metadata_url``) ::

  CredentialEventData ::= (credential_id: CredentialHolderId) (schema_ref: SchemaRef) (credential_type: CredentialType) (metadata_url: MetadataUrl)
  RegisterCredentialEvent ::= (249: Byte) (data: CredentialEventData)

``RevokeCredentialEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^

A ``RevokeCredentialEvent`` event MUST be logged when a credential is revoked.
The event records the credential identifier who requested the revocation (the holder, the issuer, or a revocation authority), and an optional string with a short comment on the revocation reason.

The ``RevokeCredentialEvent`` event is serialized as: first a byte with the value of 248, followed by :ref:`CIS-4-CredentialHolderId` (``crednetial_id``), a ``revoker``, and an optional revocation reason (``reason``), serialized similarly to :ref:`CIS-4-functions-revokeCredentialIssuer`. ``revoker`` is serialized as 1 byte to indicate who sent the revocation request ( 0 - issuer, 1 - holder, 2 -revocation authority); if the first byte is 2, then it is followed by a public key :ref:`CIS-4-PublicKeyEd25519` of the revoker::

  Revoker ::= (0: Byte)                         // Issuer
            | (1: Byte)                         // Holder
            | (2: Byte) (key: PublicKeyEd25519) // Other
  RevokeCredentialEvent ::= (248: Byte) (credential_id: CredentialHolderId) (revoker: Revoker) (reason: OptionalReason)


.. _CIS-4-events-IssuerMetadata:

``IssuerMetadata``
^^^^^^^^^^^^^^^^^^

A ``IssuerMetadata`` event MUST be logged when setting the metadata URL of the issuer.
It consists of a URL for the location of the metadata for the issuer with an optional SHA256 checksum of the content.

The ``IssuerMetadata`` event is serialized as: first a byte with the value of 247, followed by :ref:`CIS-2-MetadataUrl` (``metadata``)::

  IssuerMetadata ::= (247: Byte) (metadata: MetadataUrl)


.. _CIS-4-events-CredentialMetadataEvent:

``CredentialMetadataEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``CredentialMetadataEvent`` event MUST be logged when updating the credential metadata.
It consist of a credential ID and a URL for the location of the metadata for this credential with an optional SHA256 checksum of the content.

The ``CredentialMetadataEvent`` event is serialized as: first a byte with the value of 246, followed by :ref:`CIS-4-CredentialHolderID` (``id``), and then a :ref:`CIS-4-MetadataUrl` (``metadata``)::

  CredentialMetadataEvent ::= (246: Byte) (id: CredentialHolderId) (metadata: MetadataUrl)


``CredentialSchemaRefEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``CredentialSchemaRefEvent`` event MUST be logged when updating the credential schema reference for a credential type.
It consist of a credential type and a URL for the location of the schema for this credential with an optional SHA256 checksum of the content.

The ``CredentialSchemaRefEvent`` event is serialized as: first a byte with the value of 245, followed by :ref:`CIS-4-CredentialType` (``type``), and then a :ref:`CIS-4-SchemaRef` (``schema_ref``)::

  CredentialSchemaRefEvent ::= (245: Byte) (type: CredentialType) (schema_ref: SchemaRef)

``RevocationKeyEvent``
^^^^^^^^^^^^^^^^^^^^^^

A ``RevocationKeyEvent`` event MUST be logged when registering a new or removing an existing revocation key.
It consists of the key and the action performed with the key (registration or removal).

The ``RevocationKeyEvent`` event is serialized as: first a byte with the value of 244, followed by the key bytes :ref:`CIS-4-PublicKeyEd25519` and 1 byte encoding the action (0 for ``Register``, 1 for ``Remove``)::

  RevocationKeyAction ::= (0: Byte)    // Register
                        | (1: Byte)    // Remove
  RevocationKeyEvent ::= (244: Byte) (public_key: PublicKeyEd25519) (action: RevocationKeyAction)

.. _CIS-4-issuer-metadata-json:

Issuer metadata JSON
--------------------

The issuer metadata is stored off-chain and MUST be a JSON (:rfc:`8259`) file.

This specification reserves a number of field names, shown in the table below.
Fields not marked with ``optional`` MUST be included.

.. list-table:: Issuer metadata JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``name``
    - string
    - The name to display for the issuer.
  * - ``icon``
    - URL JSON object
    - An image URL for displaying the issuer.
  * - ``description`` (optional)
    - string
    - A description for the issuer.
  * - ``url`` (optional)
    - string (:rfc:`3986`) [``uri-reference``]
    - A URL of the issuer's website.

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

Example issuer metadata
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: json

  {
    "name": "Concordium",
    "icon" : {
      "url":  "https://concordium.com/wp-content/uploads/2022/07/Concordium-1.png",
      "hash": "1c74f7eb1b3343a5834e60e9a8fce277f2c7553112accd42e63fae7a09e0caf8"
      }
    "description": "A public-layer 1, science-backed blockchain",
    "url": "https://concordium.com"
  }

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
    - An image URL for displaying the credential. The RECOMMENDED size of the image is 40x40.
  * - ``backgroundColor``
    - string
    - A hex code of the background color for displaying the credential.
  * - ``image`` (optional)
    - URL JSON object
    - A background image URL for displaying the credential. The RECOMMENDED size of the image is 327x120.
  * - ``localization`` (optional)
    - JSON object with locales as field names (:rfc:`5646`) and field values are URL JSON objects linking to JSON files.
      Credential issuers SHOULD provide localization files for all the languages they want to support.
    - URLs to JSON files with localized token metadata.

.. TODO: check the actual image sizes before finalizing the standard.

Where the URL JSON object is the same as in :ref:`CIS-4-events-IssuerMetadata`.

Optionally a SHA256 hash of the JSON file can be logged with the :ref:`CIS-4-events-CredentialMetadataEvent` event for checking integrity.


Example credential metadata
^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: json

 {
    "title": "Concordium Employment",
    "logo" : {
      "url":  "https://concordium.com/wp-content/uploads/2022/07/Concordium-1.png",
      "hash": "1c74f7eb1b3343a5834e60e9a8fce277f2c7553112accd42e63fae7a09e0caf8"
      }
    "backgroundColor": "#000000",
    "image": {
      "url": "https://concordium.com/employment/vc-background.png",
    }
    "localization": {
      "da-DK": {
        "url": "https://location.of/the/danish/metadata.json",
        "hash": "624a1a7e51f7a87effbf8261426cb7d436cf597be327ebbf113e62cb7814a34b"
      }
    }
  }

Note that that URL addresses for images can come with or without the ``hash`` attribute.
In the example, the ``image`` attribute value does not specify ``hash``.
In this case, no content integrity check will be performed.

The Danish localization JSON file could be:

.. code-block:: json

  {
    "employer": "Arbejdsgiver",
    "employedFrom": "Ansat fra",
    "employedUntil": "Ansat indtil"
  }


.. _CIS-4-smart-contract-limitations:

Smart contract limitations
==========================

A number of limitations are important to be aware of:

- The byte size of smart contract function parameters are limited to at most 65535 B.
- Each logged event is limited to 0.5 KiB.
- The total size of the smart contract module is limited to 512 KiB.


