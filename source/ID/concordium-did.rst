.. _concordium_did:

==================================
Concordium DID Method Version 1.0
==================================

About
=====

The Concordium DID method specification conforms to the requirements specified in `Decentralized Identifiers (DIDs) v1.0 <w3c-did-core-v1.0_>`_ [DID] published by the W3C Credentials Community Group.
For more information about DIDs and DID method specifications, please see the `DID Primer`_ and `DID Spec`_.

Abstract
=========

Concordium is a layer-1 blockchain with identity built into the core protocol.
We distinguish the following types of DIDs:

- **Account DID** refer to accounts on the Concordium blockchain.
- **Smart Contract DID** refer to smart contracts instances on the Concordium blockchain.
- **Public Key DID** refers to a subject that knows the corresponding secret key.

Status of This Document
=======================

This is a draft document and may be updated.

[Add more legal text here?]

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

  ccd-did = prefix ":" *1(network ":") (acctype / pkctype / scitype / idptype)
  prefix  = %s"did:ccd"
  network = "testnet" / "mainnet"
  acctype = "acc:" 50(base58char)
  pkctype = "pkc:" 64(base16char)
  scitype = "sci:" index *1(“:” subindex)
  idptype = "idp:" index
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

A DID reference is a DID with additional data, such as query parameters.
Concordium smart contract entrypoints can be addressed using DID references.

.. code-block:: ABNF

  ccd-sc-ref = "/" entrypoint *1("?" "parameter" "=" 1*(base16char)) *1("&" "standard" "=" standard)
  entrypoint = 1*(ALPHA / DIGIT / punctuation)
  standard = "CIS" "-" 1*(DIGIT)
  punctuation = "!" / DQUOTE / "#" / "$" / "%" / "&" / "'" / "(" /
                ")" / "*" / "+" / "," / "-" / "." / "/" / ";" /
                "<" / "=" / ">" / "?" / "@" / "[" / "\" / "]" /
                "^" / "_" / "`" / "{" / "|" / "}" / "~"

See the details about *dereferencing* a smart contract instance reference in :ref:`sci-reference`.

Examples
--------

An account on testnet:

``did:ccd:testnet:acc:3ZFGxLtnUUSJGW2WqjMh1DDjxyq5rnytCwkSqxFTpsWSFdQnNn``

A smart contract instance on the default network (``mainnet``):

``did:ccd:sci:12:0``

A public key:

``did:ccd:pkc:0c7f4421e44a4385850b883e3bbf098f5a9853ef6f1a862c2ce2856381b5f5e3``

A smart contract instance with the ``viewIssuerKeys`` entrypoint that does not take any parameters

``did:ccd:sci:321/viewIssuerKeys``

A smart contract instance with the ``viewCredentialData`` entrypoint taking a parameter

``did:ccd:sci:123/viewCredentialData?parameter=ee763364dc1a47d6aa4cc6bdb005e2b2``


Concordium DID Documents
========================

.. TODO add formal DID documents

Account DID
-----------

The goal of the Account DID Document is to provide information about the account authentication data, including a possibility to reference particular pieces of data, such as public keys.
In order to do that, it specifies a `DID verification method <did-vefication-method_>`_ that reflects the account authentication data: public keys grouped into credentials.

The Account DID Document MUST contain the following data:

- ``@context`` - the attribute that expresses context information.
- ``id`` - the DID of the account.
- ``verificationMethod`` - the account's verification method.
  It is a nested :ref:`threshold scheme <concordium-did-verification-method>` requiring at ``T`` out of ``M`` credentials to sign; each credential uses its own threshold scheme requiring ``R_i`` out of ``N_i`` keys to sign, where ``i = 1..M``.and ``j = 1..N_i``.
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
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "Concordium DID URI"
    ],
    "id": "did:ccd:NET:acc:ADDR",
    "verificationMethod": [
      {
        "id": "did:ccd:NET:acc:ADDR#acc-1",
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
      "#acc-1"
    ]
  }

.. note::
  The ``publicKeyMultibase`` field contains a public key prefixed with ``f`` that denotes the base16 encoding.
  See `The Multibase Encoding Scheme`_.


Smart Contract Instance DID
---------------------------

The goal of the Smart Contract Instance DID is to provide meta-data about the contract instance.
At the moment, the main piece of meta-data is the Concordium account that send the initialization transaction.

The Smart Contract Instance DID Document MUST contain the following data:

- ``@context`` - the attribute that expresses context information.
- ``id`` - the DID of the smart contract instance.
- ``creator`` - a DID of an account that initialized the contract instance represented as a JSON object containing fields ``id`` and ``account``.
- ``entrypoints`` - a list on the contract's entrypoints.

The document MAY include any other public data of a smart contract instance.

.. code-block:: json

  {
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "Concordium DID URI"
    ],
    "id": "did:ccd:sci:IND:SUBIND",
    "owner": {
      "id": "did:ccd:sci:IND:SUBIND#creator",
      "account": "did:ccd:NET:acc:ADDR"
    }
    "entrypoints": [
      { "id": ""did:ccd:sci:IND:SUBIND#entrypoint-viewIssuerKeys",
        "name": "viewIssuerKeys"
      },
      { "id": ""did:ccd:sci:IND:SUBIND#entrypoint-viewRevocationKey",
        "name": "viewRevocationKey"
      },
  }

Where ``IND`` and ``SUBIND`` are the contract index and subindex.
``NET`` and ``ADDR`` correspond to the network and to the owner's account address.


.. _concordium-did-pkc:

Public Key Cryptography DID
---------------------------

The goal of the Public Key Cryptography DID is to represent a public key and the corresponding signature verification method.

The Public Key Cryptography DID Document MUST contain the following data:

- ``@context`` - the attribute that expresses context information.
- ``id`` - the DID of the public key.
- ``verificationMethod`` - specifies a `DID verification method <did-vefication-method_>`_ for verifying a signature corresponding to the public key.
- ``authentication`` - authentication method for the key.

.. code-block:: json

  {
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "Concordium DID URI"
    ],
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

The goal of the Identity Provider DID is identify a Concodrium identity provider (IDP).
An identity provider is an organization, approved by Concordium, that performs off-chain identification of users.
IDPs are used in the account creation process to issue an identity.
IDP DIDs can represent an issuer of a verifiable credential.

The Identity Provider DID Document MUST contain the following data:
- ``@context`` - the attribute that expresses context information.
- ``id`` - the DID of the IDP.
- ``name`` - the IDP name.
- ``url`` - A link to more information about the IDP.
- ``description`` - A free form description the IDP.
- ``verificationMethod`` - specifies a `DID verification method <did-vefication-method_>`_ for verifying a signature corresponding to the public key.

.. code-block:: json

  {
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "Concordium DID URI"
    ],
    "id": "did:ccd:testnet:idp:3",
    "name": "Digital Trust Solutions TestNet",
    "url": "https://www.digitaltrustsolutions.nl",
    "description": "Identity verified by Digital Trust Solutions on behalf of Concordium",
    "verificationMethod": [
      {
        "id": "did:ccd:testnet:idp:3#cdi-key",
        "type": "Ed25519VerificationKey2020",
        "controller": "did:ccd:NET:pkc:PK",
        "publicKeyMultibase": "fXX"
      }
    ]
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

Smart Contract Instance DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A smart contract instance DID can be created by `deploying a smart contract module <deploy-module_>`_ and `initializing a smart contract instance <initialize-contract-instance_>`_ on the ``NET`` blockchain.
The resulting DID is ``did:ccd:NET:sci:IND:SUBIND`` where ``IND``, ``SUBIND`` are the index and the subindex of the instance.

Public Key Cryptography DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A public key cryptography DID can be created by generating a fresh Ed25519 key pair.
The resulting DID is ``did:ccd:NET:pkc:PK`` where ``PK`` is the base16 encoded public key.
These DIDs are not registered on the blockchain.

Read
----

Account DID
^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:NET:acc:ADDR``

can be resolved by looking up the account with address ``ADDR`` on blockchain ``NET``.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetAccountInfo``.

See the details in the `gRPC v2 documentation`_.

From the command line, ``concordium-client`` allows to retrieve the data in the following way:

.. code-block:: console

    $concordium-client raw GetAccountInfo ADDR

.. TODO update, once we have a DID resolver


Smart Contract Instance DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:NET:sci:IND:SUBIND``

can be resolved by looking up the smart contract instance with indices ``IND``, ``SUBIND`` on blockchain ``NET``.
This includes a lookup of the owner's account.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetInstanceInfo``.

See the details in the `gRPC v2 documentation`_.

From the command line, ``concordium-client`` allows for retrieving the data in the following way:

.. code-block:: console

  $concordium-client contract show IND

.. TODO update, once we have a DID resolver


.. _sci-reference:

Smart Contract Instance Reference
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Dereferencing* the smart contract DID reference invokes the specified entrypoint.

Dereferencing a DID reference of the form

``did:ccd:NET:sci:IND:SUBIND/EP?parameter=PAR[&standard=STD]``

can be done by using the gRPC interface command ``InvokeInstance``.
The entrypoint is considered a *view*: no state changes are persisted, only the result of the invocation is returned to the caller.
The parameter ``PAR`` is passed to the entrypoint.

The result of the invocation is the return value produced by the contract or an error, if the invocation failed.
The optional query parameter ``standard=STD`` specifies the formatting of the return value.
If not specified, the return value is in the JSON format corresponding to the embedded smart contract schema.
If a contract does not have an embedded schema, the following JSON is returned:

.. code-block:: json

  {
    "contractBinaryResponse" : "BASE16DATA"
  }

``BASE16DATA`` is a base16-encoded return value.

As an example, consider CIS-4 Concordium smart contract standard that specifies a verifiable credential registry.
The special formatting rules apply to the following entrypoints:

- ``viewRevocationKey`` - the entrypoint return value is expected to be a pair of an array of bytes for a public key ``XX`` and a ``u64`` value (nonce) ``N``.
  The output is a DID document with a verification method based on the public key ``XX``.
  The bytes of ``XX`` are represented in the base16 encoding in the document.
  The key index ``K`` corresponds to the input parameter to the ``viewRevocationKey`` entrypoint.

  .. code-block:: json

    {
      "@context": [
        "https://www.w3.org/ns/did/v1",
        "Concordium DID URI"
      ],
      "id": "did:ccd:NET:sci:IND:SUBIND/viewRevocationKey?parameter=PAR&standard=CIS-4",
      "nonce": "N",
      "verificationMethod": [
        {
          "id": "did:ccd:NET:sci:IND:SUBIND/viewRevocationKey?parameter=PAR&standard=CIS-4#key-K",
          "type": "Ed25519VerificationKey2020",
          "publicKeyMultibase": "fXX"
        }
      ]
    }

- ``viewIssuerKeys`` - the entrypoint return value is expected to be a list of pairs: key index, key bytes.
  For example, ``[(0, XX); (1, YY); (2, ZZ)]``.
  The output is a DID document with a list of verification methods based on public keys ``XX``, ``YY`` and ``ZZ``.
  The bytes of the keys are represented in the base16 encoding in the document.
  The key indices correspond to the first components of the each pair in the list.

  .. code-block:: json

    {
      "@context": [
        "https://www.w3.org/ns/did/v1",
        "Concordium DID URI"
      ],
      "id": "did:ccd:NET:sci:IND:SUBIND/viewIssuerKeys?format=CIS-4",
      "verificationMethod": [
        {
          "id": "did:ccd:NET:sci:IND:SUBIND/viewIssuerKeys?format=CIS-4#key-0",
          "type": "Ed25519VerificationKey2020",
          "publicKeyMultibase": "fXX"
        },
        {
          "id": "did:ccd:NET:sci:IND:SUBIND/viewIssuerKeys?format=CIS-4#key-1",
          "type": "Ed25519VerificationKey2020",
          "publicKeyMultibase": "fYY"
        },
        {
          "id": "did:ccd:NET:sci:IND:SUBIND/viewIssuerKeys?format=CIS-4#key-2",
          "type": "Ed25519VerificationKey2020",
          "publicKeyMultibase": "fZZ"
        }
      ]
    }

.. TODO should the binary return values be allowed? What if the contract doesn't have an embedded schema?
.. TODO formatting keys and lists of keys is ad hoc, maybe we can do better in the future.

From the command line, ``concordium-client`` allows for invoking a smart contract instance in the following way:

.. code-block::

  $concordium-client contract invoke IND --entrypoint EP --energy 3000000 --parameter-json param.json

The base64 encoding of the ``param.json`` file corresponds to ``PAR``.

See the details in the `gRPC v2 documentation`_.

.. seealso::

  `Dereferencing a DID URL`_ in the W3C Credentials Community Group draft report.

.. TODO: write about binary vs JSON data
.. TODO update, once we have a DID resolver

Public Key Cryptography DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document corresponding to a DID of the form

``did:ccd:NET:pkc:PK``

can be constructed directly from the DID without any lookup necessary.

.. note::

  The ``NET`` part is optional and currently there is no difference how the documents are generated for different networks.
  In the future, however, the ``vefiricationMethod`` as it specified in :ref:`concordium-did-pkc` might depend on the network.

Update
------

At this time Concordium does not support the update of DID documents.

.. TODO Technically the account based DIDs are updateable, add something about it?

Deactivate
----------

At this time Concordium does not support deactivation of DID documents.


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
.. _concordium-accounts: https://developer.concordium.software/en/mainnet/net/references/manage-accounts.html
.. _gRPC v2 documentation: https://developer.concordium.software/concordium-grpc-api/#v2%2fconcordium%2fservice.proto
.. _deploy-module: https://developer.concordium.software/en/mainnet/smart-contracts/guides/deploy-module.html
.. _initialize-contract-instance: https://developer.concordium.software/en/mainnet/smart-contracts/guides/initialize-contract.html
.. _Dereferencing a DID URL: https://w3c-ccg.github.io/did-resolution/#dereferencing
.. _The Multibase Encoding Scheme: https://datatracker.ietf.org/doc/html/draft-multiformats-multibase-03
