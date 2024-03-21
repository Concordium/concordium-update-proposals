.. _CIS-5:

================================================================
Draft: CIS-5: Smart Contract Wallet Standard (Chaperone Account)
================================================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Mar 17, 2024
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-5``
   * - Requires
     - :ref:`CIS-0<CIS-0>`

Abstract
========

A standard interface for defining a smart contract wallet that can hold and transfer native currency and CIS-2 tokens.
Native currency/CIS-2 tokens can be deposited into the smart contract wallet by
specifying to which public key the deposit should be assigned.

The holder of the corresponding private key is the only entity that can authorize
to transfer tokens/currency in a self-custodial manner
from the public key balance (assigned in the smart contract) to some new accounts/smart contracts/public keys.

The holder of the corresponding private key does not have to submit transactions
on chain to transfer its native currency/CIS-2 token balance,
but instead, it can generate a valid signature, identify a willing third
party to submit its signature on-chain (a service fee can be added to financially incentivize a third party to do so).

The three main actions in the smart contract that can be taken:

- *deposit*: assigns the balance to a public key within the smart contract wallet.

- *internal transfer*: assigns the balance to a new public key within the smart contract wallet.

- *withdraw*: withdraws the balance out of the smart contract wallet to a native account or smart contract.

The goal of this standard is to simplify the account creation onboarding flow on Concordium
allowing for CIS-5 smart contract wallets to be supported as first-class citizens in Concordium wallets and tooling.

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

A token ID is serialized as 1 byte for the size (``n``) of the identifier, followed by this number of bytes for the token id (``id``)::

  TokenID ::= (n: Byte) (id: Byteⁿ)

.. _CIS-5-TokenAmount:

``TokenAmount``
^^^^^^^^^^^^^^^

It is serialized using the LEB128_ variable-length unsigned integer encoding, with the additional constraint that the total number of bytes of the encoding MUST not exceed 37 bytes::

  TokenAmount ::= (x: Byte)                   =>  x                     if x < 2^7
                | (x: Byte) (m: TokenAmount)  =>  (x - 2^7) + 2^7 * m   if x >= 2^7

.. _LEB128: https://en.wikipedia.org/wiki/LEB128

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
^^^^^^^^^^^^^^^^^^

A short string (up to 64 characters) in hex-encoding.
The string describes the chain context (e.g. the genesisHash is a suitable candidate) to prevent re-playing signatures across the testnet/mainnet chain.

It is serialized as: First byte encodes the length (``n``) of the string, followed by this many bytes for the string characters::

  ChainContext ::= (n: Byte) (characters: Byteⁿ)

.. _CIS-5-SigningData:

``SigningData``
^^^^^^^^^^^^^^^

Signing data contains metadata for the signature that is used to check whether the signed message is designated for the correct chain, contract, entrypoint, and that it is not expired or has a nonce mismatch, and the serviceFee commitment is included.

It is serialized as :ref:`CIS-5-ChainContext` (``chain_context``), :ref:`CIS-5-ContractAddress` (``contract_address``), :ref:`CIS-5-EntrypointName` (``entrypoint``), :ref:`CIS-5-Nonce` (``nonce``), :ref:`CIS-5-Timestamp` (``timestamp``), :ref:`CIS-5-CCDAmount`/:ref:`CIS-5-TokenAmount` (``serviceFee``), and :ref:`CIS-5-Address` (``serviceFeeRecipient``)::

  SigningData ::= (chain_context: String) (contract_address: ContractAddress) (entrypoint: EntrypointName) (nonce: Nonce) (timestamp: Timestamp) (serviceFee: CCDAmount/tokenAmount) (serviceFeeRecipient: Address)

For each of the signature checking endpoints the signing data is as follows::

  WithdrawNativeCurrencySigningData ::= (to: Receiver) (ccdAmount: CCDAmount) (data: AdditionalData) (signingData: SigningData)

  WithdrawCis2TokensSigningData ::= (to: Receiver) (tokenAmount: tokenAmount) (tokenId: tokenID) (cis2TokenContractAddress: ContractAddress) (data: AdditionalData) (signingData: SigningData)

  InternalNativeCurrencyTransferSigningData ::= (from: PublicKeyEd25519) (to: PublicKeyEd25519) (ccdAmount: CCDAmount) (signingData: SigningData)

  InternalCis2TokensTransferSigningData ::= (from: PublicKeyEd25519) (to: PublicKeyEd25519) (tokenAmount: tokenAmount) (tokenID: TokenID) (cis2TokenContractAddress: ContractAddress) (signingData: SigningData)


Logged events
-------------

The event defined by this specification is serialized using one byte to discriminate it from other events logged by the smart contract.
Other events logged by the smart contract SHOULD NOT have a first byte colliding with the event defined by this specification.

``NonceEvent``
^^^^^^^^^^^^^^

A ``NonceEvent`` SHALL be logged for every signature checking function invoke.

The ``NonceEvent`` is serialized as: First a byte with the value of 250, followed by the :ref:`CIS-5-Nonce` (``nonce``) that is used in the SigningData, and an :ref:`CIS-5-AccountAddress` (``sponsoree``)::

  NonceEvent ::= (250: Byte) (nonce: Nonce) (sponsoree: AccountAddress)

``DepositNativeCurrencyEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``DepositNativeCurrencyEvent`` SHALL be logged ever time an amount of CCD received by the contract is assigned to a public key.

The ``DepositNativeCurrencyEvent`` is serialized as: First a byte with the value of 249, followed by the :ref:`CIS-5-CCDAmount` (``ccdAmount``), the :ref:`CIS-5-Address` (``from``), and a :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  DepositNativeCurrencyEvent ::= (249: Byte) (ccdAmount: CCDAmount) (from: Address) (to: PublicKeyEd25519)

``DepositCis2TokensEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``DepositCis2TokensEvent`` SHALL be logged ever time a token amount received by the contract is assigned to a public key.

The ``DepositCis2TokensEvent`` is serialized as: First a byte with the value of 248, followed by the
:ref:`CIS-5-TokenAmount` (``tokenAmount``), :ref:`CIS-5-TokenID` (``TokenID``),
:ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), the :ref:`CIS-5-Address` (``from``), and a :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  DepositCis2TokensEvent ::= (248: Byte) (tokenAmount: TokenAmount) (tokenId: TokenID) (cis2TokenContractAddress: ContractAddress) (from: Address) (to: PublicKeyEd25519)

``WithdrawNativeCurrencyEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``WithdrawNativeCurrencyEvent`` SHALL be logged ever time an amount of CCD held by a public key is withdrawn to an address.

The ``WithdrawNativeCurrencyEvent`` is serialized as: First a byte with the value of 247, followed by the :ref:`CIS-5-CCDAmount` (``ccdAmount``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-Address` (``to``)::

  DepositNativeCurrencyEvent ::= (247: Byte) (ccdAmount: CCDAmount) (from: PublicKeyEd25519) (to: Address)

``WithdrawCis2TokensEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``WithdrawCis2TokensEvent`` SHALL be logged ever time a token amount held by a public key is withdrawn to an address.

The ``WithdrawCis2TokensEvent`` is serialized as: First a byte with the value of 246, followed by the
:ref:`CIS-5-TokenAmount` (``tokenAmount``), :ref:`CIS-5-TokenID` (``TokenID``),
:ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-Address` (``to``)::

  WithdrawCis2TokensEvent ::= (246: Byte) (tokenAmount: TokenAmount) (tokenId: TokenID) (cis2TokenContractAddress: ContractAddress) (from: PublicKeyEd25519) (to: Address)

``InternalNativeCurrencyTransferEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``InternalNativeCurrencyTransferEvent`` SHALL be logged ever time an amount of CCD held by a public key is transferred to another public key within the contract.

The ``InternalNativeCurrencyTransferEvent`` is serialized as: First a byte with the value of 245, followed by the :ref:`CIS-5-CCDAmount` (``ccdAmount``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  InternalNativeCurrencyTransferEvent ::= (245: Byte) (ccdAmount: CCDAmount) (from: PublicKeyEd25519) (to: PublicKeyEd25519)

``InternalCis2TokensTransferEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``InternalCis2TokensTransferEvent`` SHALL be logged ever time a token amount held by a public key is transferred to another public key within the contract.

The ``InternalCis2TokensTransferEvent`` is serialized as: First a byte with the value of 244, followed by the
:ref:`CIS-5-TokenAmount` (``tokenAmount``), :ref:`CIS-5-TokenID` (``TokenID``),
:ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and the :ref:`CIS-5-PublicKeyEd25519` (``to``)::

  InternalCis2TokensTransferEvent ::= (244: Byte) (tokenAmount: TokenAmount) (tokenId: TokenID) (cis2TokenContractAddress: ContractAddress) (from: PublicKeyEd25519) (to: PublicKeyEd25519)

.. note::

  The CIS-5 events SHALL enable off-chain applications to compute off-chain all balances that the public keys are holding.

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

The function is payable and deposits/assigns the send CCDAmount (native currency) to a public key (``PublicKeyEd25519``).

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

.. _CIS-5-functions-withdrawNativeCurrency:

``withdrawNativeCurrency``
^^^^^^^^^^^^^^^^^^^^^^^^^^

The function executes a list of token withdrawals of CCDs (native currency) to native accounts and/or smart contracts out of the smart contract wallet.
When transferring CCD to a contract address, a CCD receive hook function MUST be triggered.

Parameter
~~~~~~~~~

The parameter is a list of withdrawals.

It is serialized as: 2 bytes representing the number of withdrawals (``n``) followed by the bytes for this number of withdrawals.

Each ``NativeCurrencyWithdrawal`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``),
a :ref:`CIS-5-TimeStamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-CCDAmount` (``serviceFee``), an :ref:`CIS-5-Address` (``serviceFeeRecipient``), 2 bytes representing the number of simple withdraws (``m``) followed by the bytes for this number of simple withdraws (``simple_withdraw``).

Each ``NativeCurrencyWithdrawalBatch`` is serialized as: the receiving address :ref:`CIS-2-Receiver` (``to``), the :ref:`CIS-5-CCDAmount` (``ccdAmount``), and some additional data :ref:`CIS-2-AdditionalData` (``data``)::

  NativeCurrencyWithdrawalBatch ::=  (to: Receiver) (ccdAmount: CCDAmount) (data: AdditionalData)

  NativeCurrencyWithdrawal ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (expiryTime: TimeStamp) (nonce: u64) (serviceFee: CCDAmount) (serviceFeeRecipient: Address) (m: Byte²) (simple_withdraw: NativeCurrencyWithdrawalBatchᵐ)

  NativeCurrencyWithdrawParameter ::= (n: Byte²) (withdrawal: NativeCurrencyWithdrawalⁿ)

.. _CIS-5-functions-transfer-ccd-receive-hook-parameter:

CCD Receive hook parameter
~~~~~~~~~~~~~~~~~~~~~~~~~~

The parameter for the CCD receive hook function contains information about the transfer and some additional data bytes.

It is serialized as: a :ref:`CIS-5-CCDAmount` (``ccdAmount``), a :ref:`CIS-5-PublicKeyEd25519` (``from``), and some additional data :ref:`CIS-2-AdditionalData` (``data``)::

  CCDReceiveHookParameter ::= (ccdAmount: CCDAmount) (from: PublicKeyEd25519) (data: AdditionalData)


Requirements
~~~~~~~~~~~~

- The list of withdrawals MUST be executed in order.
- The contract function MUST reject if any of the withdrawals fail to be executed.
- The function MUST reject if the signature verification fails for any withdrawal.
- The function MUST fail if the CCD balance of the ``signer`` is insufficient to do the withdrawal for any withdrawal.
- A function MUST non-strictly decrease the CCD balance of the ``signer`` public key and non-strictly increase the balance of the ``to`` address or fail for any withdrawal.
- A withdrawal back to this contract into the ``depositNativeCurrency`` entrypoint MUST be executed as a normal withdrawal.
- A withdrawal of a CCD amount of zero MUST be executed as a normal withdrawal.
- A withdrawal of any amount of CCD to a contract address MUST call a CCD receive hook function on the receiving smart contract with a :ref:`ccd receive hook parameter<CIS-5-functions-transfer-ccd-receive-hook-parameter>`.
- The contract function MUST reject if the CCD receive hook function called on the contract receiving CCDs rejects for any withdrawal.
- The balance of a public key not owning any CCD amount SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for every withdrawal if ``serviceFee!=0``.

.. warning::

  Be aware of transferring CCDs to a non-existing account address or contract address.
  This specification by itself does not include a standard that has to be followed.
  Checking the existence of an account address/ contract address would ideally be done off-chain before the message is even sent to the smart contract.

.. _CIS-5-functions-withdrawCis2Tokens:

``withdrawCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^^

The function executes a list of token withdrawals to native accounts and/or smart contracts out of the smart contract wallet.
This function MUST call the ``transfer`` function on the CIS-2 token contract for every withdrawal.

Parameter
~~~~~~~~~

The parameter is a list of withdrawals.

It is serialized as: 2 bytes representing the number of withdrawals (``n``) followed by the bytes for this number of withdrawals.

Each withdrawal is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``),
a :ref:`CIS-5-TimeStamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-TokenAmount` (``serviceFee``), an :ref:`CIS-5-Address` (``serviceFeeRecipient``),
a :ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``), and :ref:`CIS-2-TransferParameter` (``transferParameter``)::

  Cis2TokensWithdrawal ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (expiryTime: TimeStamp) (nonce: u64) (serviceFee: TokenAmount) (serviceFeeRecipient: Address) (cis2TokenContractAddress: ContractAddress) (transferParameter: TransferParameter)

  Cis2TokensWithdrawParameter ::= (n: Byte²) (withdrawal: Cis2TokensWithdrawalⁿ)

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
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for every withdrawal if ``serviceFee!=0``.

.. _CIS-5-functions-internalNativeCurrencyTransfer:

``internalNativeCurrencyTransfer``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The function executes a list of CCD internal transfers to public keys within the smart contract wallet.

Parameter
~~~~~~~~~

The parameter is a list of internal transfers.

It is serialized as: 2 bytes representing the number of transfers (``n``) followed by the bytes for this number of internal transfers.

Each ``NativeCurrencyInternalTransfer`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``),
a :ref:`CIS-5-TimeStamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-CCDAmount` (``serviceFee``), an :ref:`CIS-5-Address` (``serviceFeeRecipient``),
2 bytes representing the number of simple transfers (``m``) followed by the bytes for this number of simple transfers (``simple_transfer``).

Each ``NativeCurrencyInternalTransferBatch`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``from``), a :ref:`CIS-5-PublicKeyEd25519` (``to``), and a :ref:`CIS-5-CCDAmount` (``ccdAmount``)::

  NativeCurrencyInternalTransferBatch ::= (from: PublicKeyEd25519) (to: PublicKeyEd25519) (ccdAmount: CCDAmount)

  NativeCurrencyInternalTransfer ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (expiryTime: TimeStamp) (nonce: u64) (serviceFee: CCDAmount) (serviceFeeRecipient: Address) (m: Byte²) (simple_transfer: NativeCurrencyInternalTransferBatchᵐ)

  NativeCurrencyInternalTransferParameter ::= (n: Byte²) (transfer: NativeCurrencyInternalTransfer)


Requirements
~~~~~~~~~~~~

- The function MUST reject if the signature verification fails for any transfer.
- The function MUST fail if the CCD balance of the ``signer`` is insufficient to do the transfer for any transfer.
- A function MUST non-strictly decrease the CCD balance of the ``signer`` public key and non-strictly increase the balance of the ``to`` address or fail for any transfer.
- A transfer of a CCD amount of zero MUST be executed as a normal transfer.
- The balance of a public key not owning any CCD amount SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for every transfer if ``serviceFee!=0``.

.. _CIS-5-functions-internalCis2TokensTransfer:

``internalCis2TokensTransfer``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The function executes a list of token internal transfers to public keys within the smart contract wallet.

Parameter
~~~~~~~~~

The parameter is a list of internal transfers.

It is serialized as: 2 bytes representing the number of transfers (``n``) followed by the bytes for this number of internal transfers.

Each ``Cis2TokensInternalTransfer`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``signer``), a :ref:`CIS-5-SignatureEd25519` (``signature``),
a :ref:`CIS-5-TimeStamp` (``expiryTime``), a :ref:`CIS-5-Nonce` (``nonce``), a :ref:`CIS-5-TokenAmount` (``serviceFee``), an :ref:`CIS-5-Address` (``serviceFeeRecipient``),
2 bytes representing the number of simple transfers (``m``) followed by the bytes for this number of simple transfers (``simple_transfer``).

Each ``Cis2TokensInternalTransferBatch`` is serialized as: a :ref:`CIS-5-PublicKeyEd25519` (``from``), a :ref:`CIS-5-PublicKeyEd25519` (``to``), a :ref:`CIS-5-TokenAmount` (``tokenAmount``), a :ref:`CIS-5-TokenID` (``tokenID``), and a :ref:`CIS-5-ContractAddress` (``cis2TokenContractAddress``)::

  Cis2TokensInternalTransferBatch ::=  (from: PublicKeyEd25519) (to: PublicKeyEd25519)  (tokenAmount: tokenAmount) (tokenID: TokenID) (cis2TokenContractAddress: ContractAddress)

  Cis2TokensInternalTransfer ::= (signer: PublicKeyEd25519) (signature: SignatureEd25519) (expiryTime: TimeStamp) (nonce: u64) (serviceFee: TokenAmount) (serviceFeeRecipient: Address) (m: Byte²) (simple_transfer: Cis2TokensInternalTransferBatchᵐ)

  Cis2TokensInternalTransferParameter ::= (n: Byte²) (transfer: Cis2TokensInternalTransfer)


Requirements
~~~~~~~~~~~~

- The function MUST reject if the signature verification fails for any of the transfers.
- The function MUST fail if the token balance of the ``signer`` is insufficient to do the transfer for any transfer.
- A function MUST non-strictly decrease the token balance of the ``signer`` public key and non-strictly increase the balance of the ``to`` address or fail for any transfer.
- A transfer of a token amount of zero MUST be executed as a normal transfer.
- The balance of a public key not owning any tokens SHOULD be treated as having a balance of zero.
- The function MUST transfer the ``serviceFee`` to the ``serviceFeeRecipient`` for every transfer if ``serviceFee!=0``.

.. _CIS-5-functions-balanceOfNativeCurrency:

``balanceOfNativeCurrency``
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The function queries the CCD balances of a list of public keys.

Parameter
~~~~~~~~~

The parameter consists of a list of public keys.

It is serialized as: 2 bytes for the number of queries (``n``) and then this number of queries (``queries``).
A query is serialized as a :ref:`CIS-5-PublicKeyEd25519` (``publicKey``)::

  NativeCurrencyBalanceOfQuery ::= (publicKey: PublicKeyEd25519)

  NativeCurrencyBalanceOfParameter ::= (n: Byte²) (queries: NativeCurrencyBalanceOfQueryⁿ)

Response
~~~~~~~~

The function output response is a list of CCD amounts.

It is serialized as: 2 bytes for the number of CCD amounts (``n``) and then this number of :ref:`CIS-5-CCDAmount` (``results``)::

  NativeCurrencyBalanceOfResponse ::= (n: Byte²) (results: CCDAmountⁿ)


Requirements
~~~~~~~~~~~~

- The balance of a public key not owning any CCD  SHOULD be treated as having a balance of zero.
- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST NOT increase or decrease the CCD balance or token balance of any public key for any token type.

.. _CIS-5-functions-balanceOfCis2Tokens:

``balanceOfCis2Tokens``
^^^^^^^^^^^^^^^^^^^^^^^

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
