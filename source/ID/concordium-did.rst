.. _concordium_did:

==================================
Concordium DID Method Version 1.1
==================================

About
=====

The Concordium DID method specification conforms to the requirements specified in `Decentralized Identifiers (DIDs) v1.0 <w3c-did-core-v1.0_>`_ published by the W3C Credentials Community Group.
For more information about DIDs and DID method specifications, please see the `DID Primer`_ and `DID Spec`_.

Abstract
=========

Concordium is a layer-1 blockchain with identity built into the core protocol.
We distinguish the following types of DIDs:

- **Account DID** refers to accounts on the Concordium blockchain.
- **Concordium Credential DID** refers to a Concordium credential object.
- **Smart Contract DID** refers to smart contract instances on the Concordium blockchain.
- **Public Key DID** refers to a subject that knows the corresponding secret key.
- **Identity Provider DID** refers to an Identity Provider - an organization, approved by Concordium, that performs off-chain identification of users.
- **Encrypted Identity Credenital DID** refers to a subject that has registered at an identity provider with the corresponding holder identifier.

Status of This Document
=======================

This is a draft document and may be updated.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

Concordium DID Identifier
=========================

Method Name
-----------

The Concordium DID method name is ``ccd``.
A DID identifier that uses this method must begin with the following prefix: ``did:ccd``.
Per DID specification [DID] this string must be in lowercase.

Identifier Syntax
-----------------

Concordium DID identifiers are defined by the following ABNF_:

.. code-block:: ABNF

  ccd-did = prefix ":" *1(network ":") (acctype / pkctype / scitype / idptype / credtype / encidtype)
  prefix  = %s"did:ccd"
  network = "testnet" / "mainnet"
  acctype = "acc:" 50(base58char)
  pkctype = "pkc:" 64(base16char)
  scitype = "sci:" index *1(“:” subindex)
  idptype = "idp:" index
  credtype = "cred:" 96(base16char)
  encidtype = "encidcred" *(base16char)
  index = 1*DIGIT
  subindex = 1*DIGIT
  base16char = HEXDIG
  base58char = "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9" /
               "A" / "B" / "C" / "D" / "E" / "F" / "G" / "H" / "J" /
               "K" / "L" / "M" / "N" / "P" / "Q" / "R" / "S" / "T" /
               "U" / "V" / "W" / "X" / "Y" / "Z"

.. note::
    In the ABNF_ grammar, string literals are case-insensitive, unless explicitly specified using the ``%s`` prefix.

.. note::
    The network part of a CCD DID is optional.
    *If omitted the network is assumed to be "mainnet" by default*.

.. note::
    The ``subindex`` part of the smart contract instance ``sci`` is optional.
    *If omitted the subindex is assumed to be `0` by default.*

Smart Contract Instance Reference Syntax
----------------------------------------

A DID reference is a DID with additional data, such as a path and query parameters.
Concordium smart contract entrypoints can be addressed using DID references.

.. code-block:: ABNF

  ccd-sc-ref = "/" entrypoint *1("/" parameter)
  parameter = base16char
  entrypoint = 1*(ALPHA / DIGIT / punctuation)
  punctuation = "!" / DQUOTE / "#" / "$" / "%" / "&" / "'" / "(" /
                ")" / "*" / "+" / "," / "-" / "." / "/" / ";" /
                "<" / "=" / ">" / "?" / "@" / "[" / "\" / "]" /
                "^" / "_" / "`" / "{" / "|" / "}" / "~"

See the details about *dereferencing* a smart contract instance reference in :ref:`sci-reference`.

Examples
--------

An account on testnet:

``did:ccd:testnet:acc:3ZFGxLtnUUSJGW2WqjMh1DDjxyq5rnytCwkSqxFTpsWSFdQnNn``


A Concordium credential on mainnet

``did:ccd:mainnet:cred:9aa3641a212da36a9ffae6e6085b9cf486ca9b44fa059aa74565b0a1c0f7052d8e71168beccf299d767f3961b33aaae2``

A smart contract instance on the default network (``mainnet``):

``did:ccd:sci:12:0``

A public key:

``did:ccd:pkc:0c7f4421e44a4385850b883e3bbf098f5a9853ef6f1a862c2ce2856381b5f5e3``

A smart contract instance with the ``issuerKeys`` entrypoint that does not take any parameters

``did:ccd:sci:321/issuerKeys``

A smart contract instance with the ``credentialEntry`` entrypoint taking a parameter

``did:ccd:sci:123/credentialEntry/ee763364dc1a47d6aa4cc6bdb005e2b2``


Concordium DID Documents
========================

Account DID
-----------

The goal of the Account DID Document is to provide information about the account authentication data, including a possibility to reference particular pieces of data, such as public keys.
In order to do that, it specifies a `DID verification method <did-vefication-method_>`_ that reflects the account authentication data: public keys grouped into credentials.

The Account DID Document MUST contain the following data:

- ``id`` - the DID of the account.
- ``verificationMethod`` - the account's verification method.
  It is a nested :ref:`threshold scheme <concordium-did-verification-method>` requiring ``T`` out of ``M`` credentials to sign; each credential uses its own threshold scheme requiring ``R_i`` out of ``N_i`` keys to sign, where ``i = 1..M``.and ``j = 1..N_i``.
  The credentials are identified by a `DID fragment`_ ``#credential-i``, and the keys in each credentials by ``#key-j-i`` where ``i = 1..M`` and ``j = 1..N_i``.
- ``authentication`` - authentication method for the account.

The document MAY include any other public data of a Concordium account.

.. note::

  A `DID fragment`_ allows for referencing a particular credential, or a key in the Account DID Document.
  The fragment is used to locate the (unique) JSON object by matching the DID URL with the object's ``id`` property.

.. seealso::

  `Dereferencing a DID URL`_ in the W3C Credentials Community Group draft report.


.. code-block:: json

  {
    "id": "did:ccd:NET:acc:ADDR",
    "verificationMethod": [
      {
        "id": "did:ccd:NET:acc:ADDR#acc-vm",
        "controller": "did:ccd:NET:acc:ADDR",
        "type": "VerifiableCondition2021",
        "blockchainAccountId": "ADDR",
        "threshold": "T",
        "conditionThreshold": [
          {
            "verificationMethod": [
              {
                "id": "did:ccd:NET:acc:ADDR#credential-1",
                "controller": "did:ccd:NET:acc:ADDR",
                "type": "VerifiableCondition2021",
                "threshold": "R_1",
                "conditionThreshold": [
                  {
                    "id": "did:ccd:NET:acc:ADDR#key-1-1",
                    "type": "Ed25519VerificationKey2020",
                    "controller": "did:ccd:NET:acc:ADDR",
                    "publicKeyMultibase": "fXX"
                  },
                  "...",
                  {
                    "id": "did:ccd:NET:acc:ADDR#key-N_1-1",
                    "type": "Ed25519VerificationKey2020",
                    "controller": "did:ccd:NET:acc:ADDR",
                    "publicKeyMultibase": "fYY"
                  }
                ]
              }
            ]
          },
          "...",
          {
            "verificationMethod": [
              {
                "id": "did:ccd:NET:acc:ADDR#credential-M",
                "controller": "did:ccd:NET:acc:ADDR",
                "type": "VerifiableCondition2021",
                "threshold": "N",
                "conditionThreshold": [
                  {
                    "id": "did:ccd:NET:acc:ADDR#key-1-M",
                    "type": "Ed25519VerificationKey2020",
                    "controller": "did:ccd:NET:acc:ADDR",
                    "publicKeyMultibase": "fVV"
                  },
                  "...",
                  {
                    "id": "did:ccd:NET:acc:ADDR#key-N_M-M",
                    "type": "Ed25519VerificationKey2020",
                    "controller": "did:ccd:NET:acc:ADDR",
                    "publicKeyMultibase": "fZZ"
                  }
                ]
              }
            ]
          }
        ]
      }
    ],
    "authentication": [
      "#acc-vm"
    ]
  }

.. note::
  The ``publicKeyMultibase`` field contains a public key prefixed with ``f`` that denotes the base16 encoding.
  See `The Multibase Encoding Scheme`_.


Concordium Credential DID
-------------------------

The goal of the Concordium Credential DID Document is to provide information about Concordium credentials, including a possibility to reference particular pieces of data, such as public keys.
In order to do that, it specifies a `DID verification method <did-vefication-method_>`_ that reflects the credential authentication data.

The Concordium Credential DID Document MUST contain the following data:

- ``id`` - the DID of the credential.
- ``verificationMethod`` - the credential's verification method.
- ``authentication`` - authentication method for the credential.

The document MAY include any other public data of a Concordium credential.

The following document defines a Concordium credential with ID ``CRED``.
The credential has ``N`` keys and uses a threshold signature scheme requiring ``T`` signatures.

.. code-block:: json

  {
    "id": "did:ccd:NET:cred:CRED",
    "verificationMethod": [
      {
        "did:ccd:NET:cred:CRED#credential-vm"
        "type": "VerifiableCondition2021",
        "threshold": "T",
        "conditionThreshold": [
          {
            "id": "did:ccd:NET:cred:CRED#key-1",
            "type": "Ed25519VerificationKey2020",
            "controller": "did:ccd:NET:cred:CRED",
            "publicKeyMultibase": "fXX"
          },
          "...",
          {
            "id": "did:ccd:NET:cred:CRED#key-N",
            "type": "Ed25519VerificationKey2020",
            "controller": "did:ccd:NET:cred:CRED",
            "publicKeyMultibase": "fYY"
          }
        ]
      }
    ],
    "authentication": [
      "#credential-vm"
    ]
  }


Smart Contract Instance DID
---------------------------

The goal of the Smart Contract Instance DID is to provide metadata about the contract instance.
At the moment, it contains an account address of the initialization transaction sender, and the list of the contract's entrypoints.

The Smart Contract Instance DID Document MUST contain the following data:

- ``id`` - the DID of the smart contract instance.
- ``creator`` - a DID of an account that initialized the contract instance represented as a JSON object containing fields ``id`` and ``account``.
- ``entrypoints`` - a list on the contract's entrypoints. Each entrypoint is an object containing fields ``id`` and ``name``.

The document MAY include any other public data of a smart contract instance.

.. code-block:: json

  {
    "id": "did:ccd:sci:IND:SUBIND",
    "owner": {
      "id": "did:ccd:sci:IND:SUBIND#creator",
      "account": "did:ccd:NET:acc:ADDR"
    }
    "entrypoints": [
      { "id": "did:ccd:sci:IND:SUBIND#entrypoint-issuerKeys",
        "name": "issuerKeys"
      },
      { "id": "did:ccd:sci:IND:SUBIND#entrypoint-revocationKey",
        "name": "revocationKey"
      }
    ]
  }

Where ``IND`` and ``SUBIND`` are the contract index and subindex.
``NET`` and ``ADDR`` correspond to the network and to the owner's account address respectively.


.. _concordium-did-pkc:

Public Key Cryptography DID
---------------------------

The goal of the Public Key Cryptography DID is to represent a public key and the corresponding signature verification method.

The Public Key Cryptography DID Document MUST contain the following data:

- ``id`` - the DID of the public key.
- ``verificationMethod`` - specifies a `DID verification method <did-vefication-method_>`_ for verifying a signature corresponding to the public key.
- ``authentication`` - authentication method for the key.

.. code-block:: json

  {
    "id": "did:ccd:pkc:XX",
    "verificationMethod": [
      {
        "id": "did:ccd:pkc:XX#key-0",
        "type": "Ed25519VerificationKey2020",
        "controller": "did:ccd:NET:pkc:PK",
        "publicKeyMultibase": "fXX"
      }
    ],
    "authentication": [
      {
        "did:ccd:pkc:XX#key-0"
      }
    ]
  }

Identity Provider DID
---------------------

The goal of the Identity Provider DID is identify a Concordium identity provider.
An identity provider (IDP) is an organization, approved by Concordium, that performs off-chain identification of users.
Identity providers are used in the account creation process to issue an identity.
Identity provider DIDs can represent an issuer of a verifiable credential.

The Identity Provider DID Document MUST contain the following data:

- ``id`` - the DID of the IDP.
- ``name`` - the IDP name.
- ``url`` - A link to more information about the IDP.
- ``description`` - A free form description the IDP.
- ``verificationMethod`` - specifies a `DID verification method <did-vefication-method_>`_ for verifying a signature corresponding to the public key.

.. code-block:: json

  {
    "id": "did:ccd:testnet:idp:0",
    "name": "Concordium testnet IP",
    "url": "",
    "description": "Concordium testnet identity provider",
    "verificationMethod": [
      {
        "id": "did:ccd:testnet:idp:0#cdi-key",
        "type": "Ed25519VerificationKey2020",
        "controller": "did:ccd:testnet:idp:0",
        "publicKeyMultibase": "fXX"
      }
    ]
  }

Encrypted Identity Credential DID
---------------------------------

The goal of the Encrypted Identity Credential DID is to represent a pseudonymous subject that has registered at an identity provider.
The subject can be identified via identity disclosure.

The Encrypted Identity Credential DID Document MUST contain the following data:

- ``id`` - the DID of the subject.
- ``idp`` - specifies the identity provider that issued the identity credential

.. code-block:: json

  {
    "id": "did:ccd:encidcredXX",
    "idp": "did:ccd:idp:YY"
  }

Concordium DID Operations
=========================

Concordium DIDs are managed on the Concordium blockchain.

Create
------

Account DID
^^^^^^^^^^^

An account DID can be created by `opening an account <concordium-accounts_>`_ on the ``NET`` blockchain.
The resulting DID is ``did:ccd:NET:acc:ADDR`` where ``ADDR`` is the base58 encoded account address.

Concordium Credential DID
^^^^^^^^^^^^^^^^^^^^^^^^^

A Concordium Credential DID is created as part of the account opening process, or by adding a Concordium credential to an existing account.

Smart Contract Instance DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A smart contract instance DID can be created by `deploying a smart contract module <deploy-module_>`_ and `initializing a smart contract instance <initialize-contract-instance_>`_ on the ``NET`` blockchain.
The resulting DID is ``did:ccd:NET:sci:IND:SUBIND`` where ``IND``, ``SUBIND`` are the index and the subindex of the instance.

Public Key Cryptography DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A public key cryptography DID can be created by generating a fresh Ed25519 key pair.
The resulting DID is ``did:ccd:NET:pkc:PK`` where ``PK`` is the base16 encoded public key.
These DIDs are not registered on the blockchain.

Identity Provider DID
^^^^^^^^^^^^^^^^^^^^^

Identity providers can be added as a `chain update <https://docs.rs/concordium_base/1.2.0/concordium_base/updates/index.html>`_ transaction of type `UpdateAddIdentityProvider <https://docs.rs/concordium_base/1.2.0/concordium_base/updates/enum.UpdateType.html#variant.UpdateAddIdentityProvider>`_.

Encrypted Identity Credential DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An encrypted identity credential DID can be created as part of a verifiable presentation that is directly based on the identity credential.

Read
----

Account DID
^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:NET:acc:ADDR``

can be resolved by looking up an account with address ``ADDR`` on blockchain ``NET``.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetAccountInfo``.

See the details in the `gRPC v2 documentation`_.

From the command line, ``concordium-client`` allows retrieving the data in the following way:

.. code-block:: console

    $concordium-client raw GetAccountInfo ADDR

.. TODO update, once we have a DID resolver


Concordium Credential DID
^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:NET:cred:CRED``

can be resolved by looking up a credential with ID ``CRED`` on blockchain ``NET``.

Data required to construct the DID document can be acquired by using the same gRPC interface command ``GetAccountInfo`` as for Concordium account DIDs.

.. TODO update, once we have a DID resolver


Smart Contract Instance DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:NET:sci:IND:SUBIND``

can be resolved by looking up a smart contract instance with indices ``IND``, ``SUBIND`` on blockchain ``NET``.
This includes a lookup of the owner's account.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetInstanceInfo``.

See the details in the `gRPC v2 documentation`_.

From the command line, ``concordium-client`` allows retrieving the data in the following way:

.. code-block:: console

  $concordium-client contract show IND

.. TODO update, once we have a DID resolver


.. _sci-reference:

Smart Contract Instance Reference
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Dereferencing* the smart contract DID reference invokes the specified entrypoint.

Dereferencing a DID reference of the form

``did:ccd:NET:sci:IND:SUBIND/EP[/PAR]``

can be done by using the gRPC interface command ``InvokeInstance``.
The entrypoint is considered a *view*: no state changes are persisted, only the result of the invocation is returned to the caller.
The parameter ``PAR`` is passed to the entrypoint.

The result of the invocation is the return value produced by the contract or an error if the invocation fails.

If the contract contains an embedded schema, then the response is the following:

.. code-block:: json

  {
    "type" : "json",
    "data": "JSON"
  }

Where ``JSON`` is the JSON rendering of the response.

If the contract does not contain an embedded schema, then the response is the following:

.. code-block:: json

  {
    "type" : "raw",
    "data": "BASE16DATA"
  }

Where ``BASE16DATA`` is a base16-encoded return value.

From the command line, ``concordium-client`` allows invoking a smart contract instance in the following way:

.. code-block::

  $concordium-client contract invoke IND --entrypoint EP --parameter-binary param.bin

The base16 encoding of the ``param.bin`` file corresponds to ``PAR``.

See the details in the `gRPC v2 documentation`_.

.. seealso::

  `Dereferencing a DID URL`_ in the W3C Credentials Community Group draft report.

.. TODO update, once we have a DID resolver

Public Key Cryptography DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document corresponding to a DID of the form

``did:ccd:NET:pkc:PK``

can be constructed directly from the DID without any lookup necessary.

.. note::

  The ``NET`` part is optional and currently there is no difference how the documents are generated for different networks.
  In the future, however, the ``vefiricationMethod`` as it specified in :ref:`concordium-did-pkc` might depend on the network.

Identity Provider DID
^^^^^^^^^^^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:NET:idp:INDEX``

can be resolved by looking up an identity provider ``INDEX`` on blockchain ``NET``.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetIdentityProviders``.

See the details in the `gRPC v2 documentation`_.

From the command line, ``concordium-client`` allows retrieving the data in the following way:

.. code-block:: console

    $concordium-client raw GetIdentityProviders


Encrypted Identity Credential DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document corresponding to a DID of the form

``did:ccd:NET:encidcredENCID``

can be constructed from a `ConcordiumIdBasedCredential` verifiable presentation that contains ``did:ccd:NET:encidcredENCID`` as the ``id`` of the `credentialSubject`.
The ``idp`` field is set to the `issuer` field of the verifiable presentation.

Update
------

Account DID
^^^^^^^^^^^

It is possible to update Account DID documents by sending an `update credentials transaction <https://docs.rs/concordium_base/1.2.0/concordium_base/transactions/construct/fn.update_credentials.html>`_.
This type of transaction allows for adding/removing credentials and changing the signature threshold.

Concordium Credential DID
^^^^^^^^^^^^^^^^^^^^^^^^^

It is possible to update Concordium Credential DID documents by sending an `update credentials keys transaction <https://docs.rs/concordium_base/1.2.0/concordium_base/transactions/send/fn.update_credential_keys.html>`_.
This type of transaction allows for updating the keys associated with a Concordium credential and the corresponding signature threshold.


Deactivate
----------

Concordium Credential DID
^^^^^^^^^^^^^^^^^^^^^^^^^

A Concordium Credential DID can be deactivated by removing the corresponding credential with an `update credentials transaction <https://docs.rs/concordium_base/1.2.0/concordium_base/transactions/construct/fn.update_credentials.html>`_.

Security Considerations
=======================

The ``did:ccd`` method is built on top the Concordium blockchain, a public permissionless distributed ledger (DL).
Security of the DID method reduces to the security of the underlying blockchain protocol.
This concerns attacks such as eavesdropping, replay, message insertion, deletion, modification, denial of service, amplification, and man-in-the-middle.

Parties SHOULD run a full node of the underlying blockchain protocol to ensure that they can read and write securely to the DL.

Authorization is performed by means of digital signature keys.
Leakage of private keys allows an attacker to take control.
Therefore, parties MUST handle private keys with care.


Privacy Considerations
=======================

DIDs SHOULD be assumed to be pseudonyoums and public as they might be stored on the underlying DL.
Correlation attacks MAY be possible if information assocciated to DIDs is published.
It is therefore NOT RECOMMENDED to reuse PKC DIDs.


Appendices
==========

.. _concordium-did-verification-method:


Threshold Verification Method
-----------------------------

The threshold verification method used in Concordium DID Documents is based on a `ConditionalProof verification method <https://w3c-ccg.github.io/verifiable-conditions/>`_.
This is a new type of verification method under development.
``ConditionalProof`` features several extensions such as logical operations (``and``, ``or``), threshold and weighted threshold.
Note that the method is not yet a W3C standard and currently has a *draft* status.

The example below shows the ``2-out-of-3`` signature verification method.
It uses the ``ConditionalProof2022`` verification method.
It specifies ``conditionThreshold`` with three keys ``key-1``, ``key-2`` and ``key-3``; each signature can be verified using ``Ed25519VerificationKey2020``.
The document that uses the ``2-out-of-3`` method is valid if it has at least two valid signatures.

.. code-block:: json

  {
    "id": "did:example:123#2-out-of-3",
    "controller": "did:example:123",
    "type": "ConditionalProof2022",
    "threshold": 2,
    "conditionThreshold": [
      {
        "id": "did:example:123#key-1",
        "type": "Ed25519VerificationKey2020",
        "controller": "...",
        "publicKeyMultibase": "..."
      },
      {
        "id": "did:example:123#key-2",
        "type": "Ed25519VerificationKey2020",
        "controller": "...",
        "publicKeyMultibase": "..."
      },
      {
        "id": "did:example:123#key-3",
        "type": "Ed25519VerificationKey2020",
        "controller": "...",
        "publicKeyMultibase": "..."
      }
    ]
  }


.. _w3c-did-core-v1.0: https://www.w3.org/TR/did-core/
.. _DID Primer : https://github.com/WebOfTrustInfo/rebooting-the-web-of-trust-fall2017/blob/master/topics-and-advance-readings/did-primer.md
.. _DID Spec: https://w3c-ccg.github.io/did-spec/
.. _DID fragment: https://w3c.github.io/did-core/#dfn-did-fragments
.. _did-vefication-method: https://w3c.github.io/did-core/#verification-methods
.. _ABNF: https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form
.. _concordium-accounts: https://docs.concordium.com/en/mainnet/docs/protocol/manage-accounts.html
.. _gRPC v2 documentation: https://docs.concordium.com/concordium-grpc-api/#v2%2fconcordium%2fservice.proto
.. _deploy-module: https://docs.concordium.com/en/mainnet/docs/smart-contracts/guides/deploy-module.html
.. _initialize-contract-instance: https://docs.concordium.com/en/mainnet/docs/smart-contracts/guides/initialize-contract.html
.. _Dereferencing a DID URL: https://w3c-ccg.github.io/did-resolution/#dereferencing
.. _The Multibase Encoding Scheme: https://datatracker.ietf.org/doc/html/draft-multiformats-multibase-03
