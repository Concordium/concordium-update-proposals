.. _CIS-5:

=====================================
CIS-5: Smart Contract Wallet Standard
=====================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Mar 17, 2024
   * - Final
     - Mar 28, 2024
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-5``
   * - Requires
     - :ref:`CIS-0<CIS-0>`

Abstract
========

A standard interface for defining a smart contract wallet that can hold and transfer native currency and cis2 tokens.
Native currency/Cis2 tokens can be deposited into the smart contract wallet by
specifying to which public key the deposit should be assigned.

The holder of the corresponding private key is the only entity that can authorize
to transfer tokens/currency in a self-custodial manner
from the public key balance (assigned in the smart contract) to some new accounts/smart contracts/public keys.

The holder of the corresponding private key does not have to submit transactions
on chain to transfer its native currency/cis2 token balance,
but instead, it can generate a valid signature, identify a willing third
party to submit its signature on-chain (a service fee can be added to financially incentivize a third party to do so).

The three main actions in the smart contract that can be taken:

- *deposit*: assigns the balance to a public key within the smart contract wallet.

- *internal transfer*: assigns the balance to a new public key within the smart contract wallet.

- *withdraw*: withdraws the balance out of the smart contract wallet to a native account or smart contract.

The goal of this standard is to simplify the account creation onboarding flow on Concordium
allowing for CIS5 smart contract wallets to be supported as first-class citizens on Concordium wallets and tooling.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

General types and serialization
-------------------------------

.. note::

  Integers are encoded in little-endian unless stated otherwise.

.. _CIS-5-TokenID:

``TokenID``
^^^^^^^^^^^

- The token ID SHALL be able to encode all possible tokenIds from the :ref:`CIS-2-TokenID` (``tokenId``)

A token ID is serialized as 1 byte for the size (``n``) of the identifier, followed by this number of bytes for the token id (``id``)::

  TokenID ::= (n: Byte) (id: Byteⁿ)

.. note::

  Token IDs can be as small as a single byte (by setting the first byte to the value 0) or as big as 256 bytes leaving more than 10^614 possible token IDs.
  The token ID in the CIS5 standard needs to be able to encode up to 256 bytes long tokenIds.

.. _CIS-5-TokenAmount:

``TokenAmount``
^^^^^^^^^^^^^^^

- The token amount SHALL be able to encode all possible tokenAmounts from the :ref:`CIS-2-TokenAmount` (``amount``). An amount of a token type is an unsigned integer up to 2^256 - 1.

It is serialized using the LEB128_ variable-length unsigned integer encoding, with the additional constraint of the total number of bytes of the encoding MUST not exceed 37 bytes::

  TokenAmount ::= (x: Byte)                   =>  x                     if x < 2^7
                | (x: Byte) (m: TokenAmount)  =>  (x - 2^7) + 2^7 * m   if x >= 2^7

.. _LEB128: https://en.wikipedia.org/wiki/LEB128


.. _CIS-5-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL and optional checksum for metadata stored outside of this contract.

It is serialized as: 2 bytes for the length (``n``) of the metadata URL in little-endian and then this many bytes for the URL to the metadata (``url``) followed by an optional checksum.
The checksum is serialized by 1 byte to indicate whether a hash of the metadata is included.
If its value is 0, then there is no hash; if the value is 1, then 32 bytes for a SHA256 hash (``hash``) follows::

  MetadataChecksum ::= (0: Byte)
                     | (1: Byte) (hash: Byte³²)

  MetadataUrl ::= (n: Byte²) (url: Byteⁿ) (checksum: MetadataChecksum)

.. _CIS-5-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex, both unsigned 64-bit integers.

It is serialized as: First 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``), both little-endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)


.. _CIS-5-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^

An address of an account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte³²)

.. _CIS-5-EntrypointName:

``EntrypointName``
^^^^^^^^^^^^^^^^^^

A name for a smart contract function entrypoint.

It is serialized as: First 2 bytes encode the length (``n``) of the entrypoint name in little-endian, followed by this many bytes for the entrypoint name (``entrypoint``)::

  EntrypointName ::= (n: Byte²) (entrypoint: Byteⁿ)

.. _CIS-5-Timestamp:

``Timestamp``
^^^^^^^^^^^^^

A timestamp given in milliseconds since Unix epoch.
It consists of an unsigned 64-bit integer.

It is serialized as 8 bytes in little-endian::

  Timestamp ::= (milliseconds: Byte⁸)

.. _CIS-5-Nonce:

``Nonce``
^^^^^^^^^

An unsigned 64-bit integer number that increases sequentially to protect against replay attacks.

It is serialized as 8 bytes in little-endian::

  Nonce ::= (nonce: Byte⁸)

.. _CIS-5-CCDAmount:

``CCDAmount``
^^^^^^^^^^^^^

An unsigned 64-bit integer number.

It is serialized as 8 bytes in little-endian::

  CCDAmount ::= (ccdAmount: Byte⁸)

.. _CIS-5-PublicKeyEd25519:

``PublicKeyEd25519``
^^^^^^^^^^^^^^^^^^^^

A public key is represented as a 32-byte array.

It is serialized as 32 bytes::

  PublicKeyEd25519 ::= (key: Byte³²)

.. _CIS-5-SignatureEd25519:

``SignatureEd25519``
^^^^^^^^^^^^^^^^^^^^

Signature for an Ed25519 message.

It is serialized as 64 bytes::

  SignatureEd25519 ::= (signature: Byte⁶⁴)

.. _CIS-5-SigningData:

``SigningData``
^^^^^^^^^^^^^^^

Signing data contains metadata for the signature that is used to check whether the signed message is designated for the correct contract and entrypoint, and that it is not expired.

It is serialized as :ref:`CIS-5-ContractAddress` (``contract_address``), :ref:`CIS-5-EntrypointName` (``entrypoint``), :ref:`CIS-5-Nonce` (``nonce``), and :ref:`CIS-5-Timestamp` (``timestamp``)::

  SigningData ::= (contract_address: ContractAddress) (entrypoint: EntrypointName) (nonce: Nonce) (timestamp: Timestamp)

Logged events
-------------

The event defined by this specification is serialized using one byte to discriminate it from other events logged by the smart contract.
Other events logged by the smart contract SHOULD NOT have a first byte colliding with the event defined by this specification.

``NonceEvent``
^^^^^^^^^^^^^^

A ``NonceEvent`` SHALL be logged for every signature checking function invoke.

The ``NonceEvent`` is serialized as: First a byte with the value of 250, followed by the :ref:`CIS-5-Nonce` (``nonce``) that was used in the PermitMessage, and an :ref:`CIS-5-AccountAddress` (``sponsoree``)::

  NonceEvent ::= (250: Byte) (nonce: Nonce) (sponsoree: AccountAddress)

``DepositNativeCurrencyEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``DepositNativeCurrencyEvent`` SHALL be logged for every `depositNativeCurrency` function invoke.

The ``DepositNativeCurrencyEvent`` is serialized as: First a byte with the value of 249, followed by the :ref:`CIS-5-CCDAmount` (``amount``), and an :ref:`CIS-5-PublicKeyEd25519` (``beneficiary``)::

  DepositNativeCurrencyEvent ::= (249: Byte) (amount: CCDAmount) (beneficiary: PublicKeyEd25519)

``DepositCis2TokensEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``DepositCis2TokensEvent`` SHALL be logged for every `depositCis2Tokens` function invoke.

The ``DepositCis2TokensEvent`` is serialized as: First a byte with the value of 248, followed by the :ref:`CIS-5-CCDAmount` (``amount``), and an :ref:`CIS-5-PublicKeyEd25519` (``beneficiary``)::

  DepositCis2TokensEvent ::= (248: Byte) (amount: CCDAmount) (beneficiary: PublicKeyEd25519)



.. _CIS-5-functions:

Contract functions
------------------

A smart contract implementing this standard MUST export the following functions:

- :ref:`CIS-5-functions-depositNativeCurrency`
- :ref:`CIS-5-functions-depositCis2Tokens`
- :ref:`CIS-5-functions-withdrawNativeCurrency`
- :ref:`CIS-5-functions-withdrawCis2Tokens`
- :ref:`CIS-5-functions-internalNativeCurrencyTransfer`
- :ref:`CIS-5-functions-internalCis2TokensTransfer`
- :ref:`CIS-5-functions-balanceOfNativeCurrency`
- :ref:`CIS-5-functions-balanceOfCis2Tokens`


.. _CIS-5-functions-depositNativeCurrency:

``depositNativeCurrency``
^^^^^^^^^^^^^^^^^^^^^^^^^

The function is payable and deposits/assigns the send CCDAmount (native currency) to a beneficiary (PublicKeyEd25519).

Parameter
~~~~~~~~~

The parameter is a PublicKeyEd25519.

See the serialization rules in :ref:`CIS-5-PublicKeyEd25519`.

Requirements
~~~~~~~~~~~~

- The function MUST emit a `NonceEvent` and a `DepositEvent`.

.. _CIS-5-functions-depositCis2Tokens:

``depositCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^
.. _CIS-5-functions-withdrawNativeCurrency:

``withdrawNativeCurrency``
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _CIS-5-functions-withdrawCis2Tokens:

``withdrawCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^^
.. _CIS-5-functions-internalNativeCurrencyTransfer:

``internalNativeCurrencyTransfer``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. _CIS-5-functions-internalCis2TokensTransfer:

``internalCis2TokensTransfer``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. _CIS-5-functions-balanceOfNativeCurrency:

``balanceOfNativeCurrency``
^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. _CIS-5-functions-balanceOfCis2Tokens:

``balanceOfCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^^^^^
