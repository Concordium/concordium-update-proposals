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

1. Concordium DID Identifier
=============================

Method Name
-----------

The Concordium DID method name is ``ccd``.
A DID identifier that uses this method must begin with the following prefix: ``did:ccd``.
Per DID specification [DID] this string must be in lowercase.

Identifier Syntax
-----------------

Concordium DID identifiers are defined by the following ABNF_:

.. code-block:: ABNF

  ccd-did = prefix ":" *1(network ":") (acctype / pkctype / scitype)
  prefix  = %s"did:ccd"
  network = "testnet" / "mainnet"
  acctype = "acc:" 50(base58char)
  pkctype = "pkc:" 64(base58char)
  scitype = "sci:" index *1(“:” subindex)
  index = 1*DIGIT
  subindex = 1*DIGIT
  base58char = "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9" /
               "A" / "B" / "C" / "D" / "E" / "F" / "G" / "H" / "J" /
               "K" / "L" / "M" / "N" / "P" / "Q" / "R" / "S" / "T" /
               "U" / "V" / "W" / "X" / "Y" / "Z" / "a" / "b" / "c" /
               "d" / "e" / "f" / "g" / "h" / "i" / "j" / "k" / "m" /
               "n" / "o" / "p" / "q" / "r" / "s" / "t" / "u" / "v" /
               "w" / "x" / "y" / "z"

.. note::
    The network part of a CCD DID is optional.
    *If omitted the network is assumed to be "mainnet" by default*.

.. note::
    The ``subindex`` part of the smart contract instance ``sci`` is optional.
    *If omitted the subindex is assumed to be `0` by default.*

Examples
--------

An account on testnet:

``did:ccd:testnet:acc:3ZFGxLtnUUSJGW2WqjMh1DDjxyq5rnytCwkSqxFTpsWSFdQnNn``

A smart contract instance on the default network (``mainnet``):

``did:ccd:sci:12:0``

A public key related to ``mainnet``:

``did:ccd:mainnet:pkc:0c7f4421e44a4385850b883e3bbf098f5a9853ef6f1a862c2ce2856381b5f5e3``

2. Concordium DID Documents
===========================

.. TODO add formal DID documents

Account DID
-----------

.. code-block:: json

  {
   "@context": [
     "https://www.w3.org/ns/did/v1",
     "Concordium DID URI"
   ],
   "id": "did:ccd:NET:acc:ADDR",
   "accountCredentials": [
     {
       "credId": "ZZZZ",
       "credentialPublicKeys": {
         "keys": [
           {
             "id": "did:ccd:pkc:XX#key-0",
             "type": "Ed25519VerificationKey2020",
             "publicKeyMultibase": "XX"
           }
         ],
         "threshold": "NN"
       }
     }
   ],
   "accountThreshold": "KK",
   "authentication": [
        "TODO"
    ],
    "verificationMethod" : "TODO"
  }

- Authentication via account credentials (this probably needs the definition of new verification method)

Smart Contract Instance DID
---------------------------

.. code-block:: json

  {
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "Concordium DID URI"
    ],
    "id": "did:ccd:sci:IND:SUBIND",
    "owner": "did:ccd:NET:acc:ADDR"
  }

Where ``IND`` and ``SUBIND`` are the contract index and subindex.
``NET`` and ``ADDR`` correspond to the network and to the owner's account address.

- Authentication?

Public Key Cryptography DID
---------------------------

.. code-block:: json

  {
    "@context": [
      "https://www.w3.org/ns/did/v1",
      "Concordium DID URI"
    ],
    "id": "did:ccd:pkc:XX",
    "publicKey": [
      {
        "id": "did:ccd:pkc:XX#key-0",
        "type": "Ed25519VerificationKey2020",
        "publicKeyMultibase": "XX"
      }
    ],
    "authentication": [
      {
        "publicKey": "did:ccd:pkc:XX#key-0"
      }
    ]
  }

3. Concordium DID Operations
=============================

Concordium DIDs are managed on the Concordium blockchain.

Create
------

Account DID
^^^^^^^^^^^

An account DID can be created by opening an account on the ``network`` blockchain.
The resulting DID is ``did:ccd:network:acc:<accountaddr>`` where ``<accountaddr>`` is the base58 encoded account address.

Smart Contract Instance DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A smart contract instance DID can be created by deploying a smart contract module on the ``network`` blockchain. The resulting DID is ``did:ccd:network:sci:<index>:<subindex>`` where ``<index>``, ``<subindex>`` are the index and the subindex of the instance.

Public Key Cryptography DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A public key cryptography DID can be created by generating a fresh Ed25519 key pair. The resulting DID is ``did:ccd:network:pkc:<pk>`` where ``<pk>`` is the base58 encoded public key. These DIDs are not registered on the blockchain.

Read
----

Account DID
^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:network:acc:accaddr``

can be resolved by looking up the account with address  ``accaddr`` on blockchain ``network``.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetAccountInfo``
See the details in the `gRPC v2 documentation <https://developer.concordium.software/concordium-grpc-api/#v2%2fconcordium%2fservice.proto>`_.

From the command line, ``concordium-client`` allows to retrieve the data in the following way:

.. code-block:: console

    $concordium-client raw GetAccountInfo <account-address>

.. TODO add more details?


Smart Contract Instance DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document information for a DID of the form

``did:ccd:network:sci:index:subindex``

can be resolved by looking up the smart contract instance with indices ``index``, ``subindex`` on blockchain ``network``.
This includes a lookup of the owner's account.

Data required to construct the DID document can be acquired by using the gRPC interface command ``GetInstanceInfo``

See the details in the `gRPC v2 documentation <https://developer.concordium.software/concordium-grpc-api/#v2%2fconcordium%2fservice.proto>`_.

From the command line, ``concordium-client`` allows to retrieve the data in the following way:

.. code-block:: console

    $concordium-client contract show <contract-index>

.. TODO add more details?

Public Key Cryptography DID
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The DID document corresponding to a DID of the form

``did:ccd:network:pkc:pk``

can be constructed directly from the DID without any lookup necessary.

.. TODO Add construction here?

Update
------

At this time Concordium does not support the update of DID documents.

.. TODO Technically the account based DIDs are updateable, add something about it?

Deactivate
----------

At this time Concordium does not support deactivation of DID documents.


.. _`w3c-did-core-v1.0`: https://www.w3.org/TR/did-core/
.. _DID Primer : https://github.com/WebOfTrustInfo/rebooting-the-web-of-trust-fall2017/blob/master/topics-and-advance-readings/did-primer.md
.. _DID Spec: https://w3c-ccg.github.io/did-spec/
.. _ABNF: https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form
