.. _CIS-5:

=====================================
CIS-5: Smart Contract Wallet Standard
=====================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Mar 17, 2024
   * - Final
     - May 23, 2024
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-5``
   * - Requires
     - :ref:`CIS-0<CIS-0>`

Abstract
========

A standard interface for defining a smart contract wallet that can hold and transfer CCDs and CIS-2 tokens.
CCDs/CIS-2 tokens can be deposited into the smart contract wallet by
specifying to which public key the deposit should be assigned.

The holder of the corresponding private key does not submit transactions
on chain to transfer/withdraw its CCD/CIS-2 token balances,
but instead, it can generate a valid signature, identify a willing third
party to submit its signature in a transaction on-chain (a service fee can be added to financially incentivize a third party to do so).

The three main actions in the smart contract that can be taken:

- *deposit*: assigns the balance to a public key within the smart contract wallet.

- *internal transfer*: assigns the balance to a new public key within the smart contract wallet.

- *withdraw*: withdraws the balance out of the smart contract wallet to a native account or smart contract.

The goal of this standard is to create an alternative simplified onboarding flow
without the inconvenience of a native account creation on Concordium.
Allowing for CIS-5 smart contract wallets to be supported as first-class citizens in Concordium wallets and tooling.

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

The token ID is from the CIS-2 standard :ref:`CIS-2-TokenID`.
A token ID is serialized as 1 byte for the size (``n``) of the identifier, followed by this number of bytes for the token id (``id``)::

  TokenID ::= (n: Byte) (id: Byteⁿ)

.. _CIS-5-TokenAmount:

``TokenAmount``
^^^^^^^^^^^^^^^

The token amount is from the CIS-2 standard :ref:`CIS-2-TokenAmount`.
It is serialized using the LEB128_ variable-length unsigned integer encoding, with the additional constraint that the total number of bytes of the encoding MUST not exceed 37 bytes::

  TokenAmount ::= (x: Byte)                   =>  x                     if x < 2^7
                | (x: Byte) (m: TokenAmount)  =>  (x - 2^7) + 2^7 * m   if x >= 2^7

.. _LEB128: https://en.wikipedia.org/wiki/LEB128

.. _CIS-5-ExternalTokenAmount:

``CIS5-ExternalTokenAmount``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A CIS-2 token amount on a blockchain defined by the amount, token id and smart contract it represents.

It is serialized as a :ref:`CIS-5-TokenAmount` (``tokenAmount``), a :ref:`CIS-5-TokenId` (``tokenId``), and a :ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``)::

  ExternalTokenAmount ::=  (tokenAmount: TokenAmount) (tokenId: TokenId) (cis2TokenContractAddress: ContractAddress)

.. _CIS-5-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^

An address of an account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte³²)

.. _CIS-5-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex, both unsigned 64-bit integers.

It is serialized as: First 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``), both little-endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)

.. _CIS-5-Address:

``Address``
^^^^^^^^^^^

Is either an :ref:`CIS-5-AccountAddress` or a :ref:`CIS-5-ContractAddress`.

It is serialized as: The first byte indicates whether it is an account address or a contract address.
In case the first byte is 0 then an :ref:`CIS-5-AccountAddress` (``address``) follows.
In case the first byte is 1 then a :ref:`CIS-5-ContractAddress` (``address``) follows::

  Address ::= (0: Byte) (address: AccountAddress)
            | (1: Byte) (address: ContractAddress)


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

An Ed25519 public key is represented as a 32-byte array.

It is serialized as 32 bytes::

  PublicKeyEd25519 ::= (key: Byte³²)

.. _CIS-5-SignatureEd25519:

``SignatureEd25519``
^^^^^^^^^^^^^^^^^^^^

Signature for an Ed25519 message.

It is serialized as 64 bytes::

  SignatureEd25519 ::= (signature: Byte⁶⁴)

.. _CIS-5-ChainContext:

``ChainContext``
^^^^^^^^^^^^^^^^

The chain context consists of the genesis hash and the contract address to prevent re-playing signatures across the testnet/mainnet chain and smart contracts.

It is serialized as: First 32 bytes defining the genesisHash (its hex string representation converted into bytes),
followed by 8 bytes for the index of the smart contract address and 8 bytes for the subindex of the smart contract address::

  ChainContext ::= (genesisHash: Byte³²) (index: Byte⁸) (subindex: Byte⁸)


Logged events
-------------

The event defined by this specification is serialized using one byte to discriminate it from other events logged by the smart contract.
Other events logged by the smart contract SHOULD NOT have a first byte colliding with the event defined by this specification.

``NonceEvent``
^^^^^^^^^^^^^^

A ``NonceEvent`` SHALL be logged every time a signature is successfully processed and considered valid by the contract.

The ``NonceEvent`` is serialized as: First a byte with the value of 250, followed by the :ref:`CIS-5-Nonce` (``nonce``) that is used in the SigningData, and an :ref:`CIS-5-PublicKeyEd25519` (``sponsoree``)::

  NonceEvent ::= (250: Byte) (nonce: Nonce) (sponsoree: PublicKeyEd25519)

``DepositCcdEvent``
^^^^^^^^^^^^^^^^^^^

A ``DepositCcdEvent`` SHALL be logged every time an amount of CCD received by the contract is assigned to a public key.

The ``DepositCcdEvent`` is serialized as: First a byte with the value of 249, followed by the :ref:`CIS-5-CCDAmount` (``ccdAmount``), the :ref:`CIS-5-Address` (``from``), and a :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  DepositCcdEvent ::= (249: Byte) (ccdAmount: CCDAmount) (from: Address) (to: PublicKeyEd25519)

``DepositCis2TokensEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``DepositCis2TokensEvent`` SHALL be logged every time a token amount received by the contract is assigned to a public key.

The ``DepositCis2TokensEvent`` is serialized as: First a byte with the value of 248, followed by the
:ref:`CIS-5-TokenAmount` (``tokenAmount``), :ref:`CIS-5-TokenID` (``TokenID``),
:ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), the :ref:`CIS-5-Address` (``from``), and a :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  DepositCis2TokensEvent ::= (248: Byte) (tokenAmount: TokenAmount) (tokenId: TokenID) (cis2TokenContractAddress: ContractAddress) (from: Address) (to: PublicKeyEd25519)

``WithdrawCcdEvent``
^^^^^^^^^^^^^^^^^^^^

A ``WithdrawCcdEvent`` SHALL be logged every time an amount of CCD held by a public key is withdrawn to an address.

The ``WithdrawCcdEvent`` is serialized as: First a byte with the value of 247, followed by the :ref:`CIS-5-CCDAmount` (``ccdAmount``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-Address` (``to``)::

  WithdrawCcdEvent ::= (247: Byte) (ccdAmount: CCDAmount) (from: PublicKeyEd25519) (to: Address)

``WithdrawCis2TokensEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``WithdrawCis2TokensEvent`` SHALL be logged every time a token amount held by a public key is withdrawn to an address.

The ``WithdrawCis2TokensEvent`` is serialized as: First a byte with the value of 246, followed by the
:ref:`CIS-5-TokenAmount` (``tokenAmount``), :ref:`CIS-5-TokenID` (``TokenID``),
:ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-Address` (``to``)::

  WithdrawCis2TokensEvent ::= (246: Byte) (tokenAmount: TokenAmount) (tokenId: TokenID) (cis2TokenContractAddress: ContractAddress) (from: PublicKeyEd25519) (to: Address)

``TransferCcdEvent``
^^^^^^^^^^^^^^^^^^^^

A ``TransferCcdEvent`` SHALL be logged every time an amount of CCD held by a public key is transferred to another public key within the contract.

The ``TransferCcdEvent`` is serialized as: First a byte with the value of 245, followed by the :ref:`CIS-5-CCDAmount` (``ccdAmount``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  TransferCcdEvent ::= (245: Byte) (ccdAmount: CCDAmount) (from: PublicKeyEd25519) (to: PublicKeyEd25519)

``TransferCis2TokensEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``TransferCis2TokensEvent`` SHALL be logged every time a token amount held by a public key is transferred to another public key within the contract.

The ``TransferCis2TokensEvent`` is serialized as: First a byte with the value of 244, followed by the
:ref:`CIS-5-TokenAmount` (``tokenAmount``), :ref:`CIS-5-TokenID` (``TokenID``),
:ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  TransferCis2TokensEvent ::= (244: Byte) (tokenAmount: TokenAmount) (tokenId: TokenID) (cis2TokenContractAddress: ContractAddress) (from: PublicKeyEd25519) (to: PublicKeyEd25519)

.. note::

  The collection of all CIS-5 events emitted by a smart contract instance implementing CIS-5 SHALL enable off-chain applications to compute off-chain all non-zero balances that the public keys are holding.

.. _CIS-5-functions:

Contract functions
------------------

A smart contract implementing this standard MUST export the following functions:

- :ref:`CIS-5-functions-depositCcd`
- :ref:`CIS-5-functions-depositCis2Tokens`
- :ref:`CIS-5-functions-withdrawCcd`
- :ref:`CIS-5-functions-withdrawCis2Tokens`
- :ref:`CIS-5-functions-transferCcd`
- :ref:`CIS-5-functions-transferCis2Tokens`
- :ref:`CIS-5-functions-ccdBalanceOf`
- :ref:`CIS-5-functions-cis2BalanceOf`


.. _CIS-5-functions-depositCcd:

``depositCcd``
^^^^^^^^^^^^^^

The function is payable and deposits/assigns the send CCDAmount to a public key (``PublicKeyEd25519``).

Parameter
~~~~~~~~~

The parameter is a ``PublicKeyEd25519``.

See the serialization rules in :ref:`CIS-5-PublicKeyEd25519`.

.. _CIS-5-functions-depositCis2Tokens:

``depositCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^

This function SHOULD be called through the receive hook mechanism (:ref:`CIS-2-Receive-Hook-Function`)
of a CIS-2 token contract. The function deposits/assigns the send CIS-2 token amount to a public key (``PublicKeyEd25519``).

.. note::

  The ``depositCis2Tokens`` function can be called by any smart contract. It is up to the exact implementation of the smart contract wallet whether it should trust the caller or not.
  The smart contract wallet is not required to check if the invoking contract is a CIS-2 token contract or has some reasonable receive hook logic implemented.
  If no additional authorization is added to this function, similar caution should be applied as if you would directly interact with any CIS-2 token contract.
  Only interact with a CIS-2 token contract or value its recorded token balance if you checked its smart
  contract logic or reasonable social reputation is given to the project/CIS-2 token contract.

Parameter
~~~~~~~~~

The parameter is the :ref:`CIS-2-functions-transfer-receive-hook-parameter` (``OnReceivingCis2Params``) and the
``data`` field of the ``OnReceivingCis2Params`` SHALL encode a ``PublicKeyEd25519``.

See the serialization rules in :ref:`CIS-2-functions-transfer-receive-hook-parameter`
and the serialization rules in :ref:`CIS-5-PublicKeyEd25519`.

Requirements
~~~~~~~~~~~~

- The function SHOULD check that a contract is the caller since only a contract can implement a receive hook mechanism.

.. _CIS-5-functions-withdrawCcd:

``withdrawCcd``
^^^^^^^^^^^^^^^

The function executes a list of token withdrawals of CCDs to native accounts and/or smart contracts out of the smart contract wallet.
When transferring CCD to a contract address, a CCD receive hook function MUST be triggered.

Parameter
~~~~~~~~~

The parameter is a list of withdrawals.

It is serialized as: 2 bytes representing the number of withdrawals (``n``) followed by the bytes for this number of ``withdrawals``.

Each ``WithdrawBatchCcdAmount`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``), and  a ``WithdrawMessageCcdAmount`` (``message``).

Each ``WithdrawMessageCcdAmount`` is serialized as: an :ref:`CIS-5-EntrypointName` (``entryPoint``), a :ref:`CIS-5-Timestamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-PublicKeyEd25519` (``serviceFeeRecipient``), a :ref:`CIS-5-CCDAmount` (``serviceFee``), 2 bytes representing the number of simple withdraws (``m``) followed by the bytes for this number of simple withdraws (``simpleWithdraws``).

Each ``WithdrawCcdAmount`` is serialized as: the receiving address :ref:`CIS-2-Receiver` (``to``), the :ref:`CIS-5-CCDAmount` (``withdrawAmount``), and some additional data :ref:`CIS-2-AdditionalData` (``data``)::

  WithdrawCcdAmount ::=  (to: Receiver) (withdrawAmount: CCDAmount) (data: AdditionalData)

  WithdrawMessageCcdAmount ::= (entryPoint: EntrypointName) (expiryTime: Timestamp) (nonce: Nonce) (serviceFeeRecipient: PublicKeyEd25519) (serviceFee: CCDAmount) (m: Byte²) (simpleWithdraws: WithdrawCcdAmountᵐ)

  WithdrawBatchCcdAmount ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (message: WithdrawMessageCcdAmount)

  WithdrawParameterCcdAmount ::= (n: Byte²) (withdrawals: WithdrawBatchCcdAmountⁿ)

.. _CIS-5-functions-transfer-ccd-receive-hook-parameter:

CCD Receive hook parameter
~~~~~~~~~~~~~~~~~~~~~~~~~~

The parameter for the CCD receive hook function contains some additional data bytes.

It is serialized as: some additional data :ref:`CIS-2-AdditionalData` (``data``)::

  CCDReceiveHookParameter ::= (data: AdditionalData)

Generating a valid signature
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To generate a valid signature for the entry point, the following bytes have to be signed::

  WithdrawCCDSigningData ::= (chainContext: ChainContext) (param: WithdrawParameterCcdAmount)

Requirements
~~~~~~~~~~~~

- The list of withdrawals MUST be executed in order.
- The contract function MUST reject if any of the withdrawals fail to be executed.
- The function MUST reject if the signature verification fails for any withdrawal.
- The function MUST fail if the CCD balance of the ``signer`` is insufficient to do the withdrawal for any withdrawal.
- A function MUST non-strictly decrease the CCD balance of the ``signer```s public key and non-strictly increase the balance of the ``to`` address or fail for any withdrawal.
- A withdrawal back to this contract into the ``depositCcd`` entrypoint MUST be executed as a normal withdrawal.
- A withdrawal of a CCD amount of zero MUST be executed as a normal withdrawal.
- A withdrawal of any amount of CCD to a contract address MUST call a CCD receive hook function on the receiving smart contract with a :ref:`ccd receive hook parameter<CIS-5-functions-transfer-ccd-receive-hook-parameter>`.
- The contract function MUST reject if the CCD receive hook function called on the contract receiving CCDs rejects for any withdrawal.
- The balance of a public key not owning any CCD amount SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for each batch withdrawal if ``serviceFee!=0``.

.. warning::

  Be aware of transferring CCDs to a non-existing account address or contract address which results in an error on Concordium.
  Checking the existence of an account address/ contract address would ideally be done off-chain before the message is even sent to the smart contract.

.. _CIS-5-functions-withdrawCis2Tokens:

``withdrawCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^^

The function executes a list of token withdrawals to native accounts and/or smart contracts out of the smart contract wallet.
This function MUST call the ``transfer`` function on the CIS-2 token contract for every withdrawal.

Parameter
~~~~~~~~~

The parameter is a list of withdrawals.

It is serialized as: 2 bytes representing the number of withdrawals (``n``) followed by the bytes for this number of ``withdrawals``.

Each ``WithdrawBatchTokenAmount`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``), and  a ``WithdrawMessageTokenAmount`` (``message``).

Each ``WithdrawMessageTokenAmount`` is serialized as: an :ref:`CIS-5-EntrypointName` (``entryPoint``), a :ref:`CIS-5-Timestamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-PublicKeyEd25519` (``serviceFeeRecipient``), a :ref:`CIS-5-ExternalTokenAmount` (``serviceFee``), 2 bytes representing the number of simple withdraws (``m``) followed by the bytes for this number of simple withdraws (``simpleWithdraws``).

Each ``WithdrawTokenAmount`` is serialized as: the receiving address :ref:`CIS-2-Receiver` (``to``), the :ref:`CIS-5-ExternalTokenAmount` (``withdrawAmount``), and some additional data :ref:`CIS-2-AdditionalData` (``data``)::

  WithdrawTokenAmount ::=  (to: Receiver) (withdrawAmount: ExternalTokenAmount) (data: AdditionalData)

  WithdrawMessageTokenAmount ::= (entryPoint: EntrypointName) (expiryTime: Timestamp) (nonce: Nonce) (serviceFeeRecipient: PublicKeyEd25519) (serviceFee: ExternalTokenAmount) (m: Byte²) (simpleWithdraws: WithdrawTokenAmountᵐ)

  WithdrawBatchTokenAmount ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (message: WithdrawMessageTokenAmount)

  WithdrawParameterTokenAmount ::= (n: Byte²) (withdrawals: WithdrawBatchTokenAmountⁿ)

Generating a valid signature
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To generate a valid signature for the entry point, the following bytes have to be signed::

  WithdrawTokensSigningData ::= (chainContext: ChainContext) (param: WithdrawParameterTokenAmount)

Requirements
~~~~~~~~~~~~

- The list of withdrawals MUST be executed in order.
- The contract function MUST reject if any of the withdrawals fail to be executed.
- The function MUST reject if the signature verification fails for any withdrawal.
- This function MUST call the ``transfer`` function on the CIS-2 token contract for every withdrawal.
- The function MUST fail if the token balance of the ``signer`` is insufficient to do the withdrawal for any withdrawal.
- A function MUST non-strictly decrease the token balance of the ``signer`` public key and non-strictly increase the balance of the ``to`` address or fail for any withdrawal.
- A withdrawal back to this contract into the ``depositCis2Tokens`` entrypoint MUST be executed as a normal withdrawal.
- A withdrawal of a token amount of zero MUST be executed as a normal withdrawal.
- The balance of a public key not owning any tokens SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for each batch withdrawal if ``serviceFee!=0``.

.. _CIS-5-functions-transferCcd:

``transferCcd``
^^^^^^^^^^^^^^^
The function executes a list of CCD transfers to public keys within the smart contract wallet.

Parameter
~~~~~~~~~

The parameter is a list of transfers.

It is serialized as: 2 bytes representing the number of transfers (``n``) followed by the bytes for this number of ``transfers``.

Each ``TransferBatchCcdAmount`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``), and  a ``TransferMessageCcdAmount`` (``message``).

Each ``TransferMessageCcdAmount`` is serialized as: an :ref:`CIS-5-EntrypointName` (``entryPoint``), a :ref:`CIS-5-Timestamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-PublicKeyEd25519` (``serviceFeeRecipient``), a :ref:`CIS-5-CCDAmount` (``serviceFee``), 2 bytes representing the number of simple transfers (``m``) followed by the bytes for this number of simple transers (``simpleTransfers``).

Each ``TransferCcdAmount`` is serialized as: the receiving :ref:`CIS-5-PublicKeyEd25519` (``to``), and the :ref:`CIS-5-CCDAmount` (``transferAmount``)::

  TransferCcdAmount ::=  (to: PublicKeyEd25519) (transferAmount: CCDAmount)

  TransferMessageCcdAmount ::= (entryPoint: EntrypointName) (expiryTime: Timestamp) (nonce: Nonce) (serviceFeeRecipient: PublicKeyEd25519) (serviceFee: CCDAmount) (m: Byte²) (simpleTransfers: TransferCcdAmountᵐ)

  TransferBatchCcdAmount ::=  (signer: PublicKeyEd25519) (signature: SignatureEd25519) (message: TransferMessageCcdAmount)

  TransferParameterCcdAmount ::= (n: Byte²) (transfers: TransferBatchCcdAmount)

Generating a valid signature
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To generate a valid signature for the entry point, the following bytes have to be signed::

  TransferCCDSigningData ::= (chainContext: ChainContext) (param: TransferParameterCcdAmount)

Requirements
~~~~~~~~~~~~

- The function MUST reject if the signature verification fails for any transfer.
- The function MUST fail if the CCD balance of the ``signer`` is insufficient to do the transfer for any transfer.
- A function MUST non-strictly decrease the CCD balance of the ``signer`` public key and non-strictly increase the balance of the ``to`` public key or fail for any transfer.
- A transfer of a CCD amount of zero MUST be executed as a normal transfer.
- The balance of a public key not owning any CCD amount SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for each batch transfer if ``serviceFee!=0``.

.. _CIS-5-functions-transferCis2Tokens:

``transferCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^^

The function executes a list of token transfers to public keys within the smart contract wallet.

Parameter
~~~~~~~~~

The parameter is a list of transfers.

It is serialized as: 2 bytes representing the number of transfers (``n``) followed by the bytes for this number of ``transfers``.

Each ``TransferBatchTokenAmount`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``), and  a ``TransferMessageTokenAmount`` (``message``).

Each ``TransferMessageTokenAmount`` is serialized as: an :ref:`CIS-5-EntrypointName` (``entryPoint``), a :ref:`CIS-5-Timestamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-PublicKeyEd25519` (``serviceFeeRecipient``), a :ref:`CIS-5-ExternalTokenAmount` (``serviceFee``), 2 bytes representing the number of simple transfers (``m``) followed by the bytes for this number of simple transfers (``simpleTransfers``).

Each ``TransferTokenAmount`` is serialized as: the receiving :ref:`CIS-5-PublicKeyEd25519` (``to``), the :ref:`CIS-5-ExternalTokenAmount` (``transferAmount``)::

  TransferTokenAmount ::=  (to: PublicKeyEd25519) (transferAmount: ExternalTokenAmount)

  TransferMessageTokenAmount ::= (entryPoint: EntrypointName) (expiryTime: Timestamp) (nonce: Nonce) (serviceFeeRecipient: PublicKeyEd25519) (serviceFee: ExternalTokenAmount) (m: Byte²) (simpletransfers: TransferTokenAmountᵐ)

  TransferBatchTokenAmount ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (message: TransferMessageTokenAmount)

  TransferParameterTokenAmount ::= (n: Byte²) (transfers: TransferBatchTokenAmountⁿ)


Generating a valid signature
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To generate a valid signature for the entry point, the following bytes have to be signed::

  TransferTokensSigningData ::= (chainContext: ChainContext) (param: TransferParameterTokenAmount)

Requirements
~~~~~~~~~~~~

- The function MUST reject if the signature verification fails for any of the transfers.
- The function MUST fail if the token balance of the ``signer`` is insufficient to do the transfer for any transfer.
- A function MUST non-strictly decrease the token balance of the ``signer`` public key and non-strictly increase the balance of the ``to`` public key or fail for any transfer.
- A transfer of a token amount of zero MUST be executed as a normal transfer.
- The balance of a public key not owning any tokens SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for each batch transfer if ``serviceFee!=0``.

.. _CIS-5-functions-ccdBalanceOf:

``ccdBalanceOf``
^^^^^^^^^^^^^^^^

The function queries the CCD balances of a list of public keys.

Parameter
~~~~~~~~~

The parameter consists of a list of public keys.

It is serialized as: 2 bytes for the number of public keys (``n``) and then this number of :ref:`CIS-5-PublicKeyEd25519` (``publicKeys``)::

  CCDBalanceOfParameter ::= (n: Byte²) (publicKeys: PublicKeyEd25519ⁿ)

Response
~~~~~~~~

The function output response is a list of CCD amounts.

It is serialized as: 2 bytes for the number of CCD amounts (``n``) and then this number of :ref:`CIS-5-CCDAmount` (``results``)::

  CCDBalanceOfResponse ::= (n: Byte²) (results: CCDAmountⁿ)


Requirements
~~~~~~~~~~~~

- The balance of a public key not owning any CCD  SHOULD be treated as having a balance of zero.
- The number of results in the response MUST correspond to the number of the public keys in the parameter.
- The order of results in the response MUST correspond to the order of public keys in the parameter.
- The contract function MUST NOT increase or decrease the CCD balance or token balance of any public key for any token type.

.. _CIS-5-functions-cis2BalanceOf:

``cis2BalanceOf``
^^^^^^^^^^^^^^^^^

The function queries the token balances of a list of public keys for given token IDs, and CIS-2 token contract addresses.

Parameter
~~~~~~~~~

The parameter consists of a list of token ID, CIS-2 token contract address, and public key triplets.

It is serialized as: 2 bytes for the number of queries (``n``) and then this number of queries (``queries``).
A query is serialized as a :ref:`CIS-5-TokenID` (``tokenID``), a :ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), and a :ref:`CIS-5-PublicKeyEd25519` (``publicKey``)::

  Cis2TokensBalanceOfQuery ::= (tokenID: TokenID) (cis2TokenContractAddress: ContractAddress) (publicKey: PublicKeyEd25519)

  Cis2TokensBalanceOfParameter ::= (n: Byte²) (queries: Cis2TokensBalanceOfQueryⁿ)

Response
~~~~~~~~

The function output response is a list of token amounts.

It is serialized as: 2 bytes for the number of token amounts (``n``) and then this number of :ref:`CIS-5-TokenAmount` (``results``)::

  Cis2TokensBalanceOfResponse ::= (n: Byte²) (results: TokenAmountⁿ)

Requirements
~~~~~~~~~~~~

- The balance of a public key not owning any amount of a token type SHOULD be treated as having a balance of zero.
- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST NOT increase or decrease the CCD balance or token balance of any public key for any token type.
