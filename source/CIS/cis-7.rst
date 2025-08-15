.. _CIS-7:

===================================
CIS-7: Protocol-level Tokens (PLTs)
===================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Jun 25, 2025
   * - Final
     - TBD
   * - Supported versions
     - | Protocol version 9 or newer
   * - Standard identifier
     - ``CIS-7``


Abstract
========

A standard interface for protocol-level tokens (PLTs).
The interface defines the data formats used when interacting with PLTs.
These data are encoded in Concise Binary Object Representation (CBOR) as defined in :rfc:`8949`.
In particular, the interface defines the following:

- Initialization parameters
- Transactions
- Events
- Reject reasons
- Token module state
- Account state

Introduction
============

Protocol-level tokens (PLTs) provide chain-native support for tokens (other than CCD) without depending on smart contracts for the implementation.
Each PLT is governed by a Token Module, which manages the token's lifecycle, including creation, transfer, minting, and burning of tokens.
There may be a single Token Module that manages all PLTs, or there may be multiple Token Modules, each managing one or more PLTs.

The Token Module defines a number of interfaces for interaction with PLTs.
At each interface, data is exchanged in Concise Binary Object Representation (CBOR) format.
The CBOR format is defined in :rfc:`8949`.
This document defines schemas for the data exchanged at each interface, including initialization parameters, transactions, events, reject reasons, token module state, and account state.

The Token Module interacts with the state of the chain through the Token Kernel, which provides basic operations for updating account balances and state, as well as generating events.
This document does not specify the behavior or interface of the Token Kernel.

The outside world interacts with the Token Module through transactions and queries.
This document does not specify the serialization format of transactions, or the `API for querying the node <https://docs.concordium.com/concordium-grpc-api/>`_.
The CreatePLT chain update transaction carries CBOR-encoded :ref:`initialization parameters<CIS-7-InitializationParameters>`.
The TokenUpdate account transaction carries a CBOR-encoded :ref:`list of transaction operations<CIS-7-Transactions>`.
The GetTokenInfo gRPC query returns the CBOR-encoded :ref:`Token Module state<CIS-7-TokenModuleState>`.
The GetAccountInfo gRPC query returns the CBOR-encoded :ref:`token account state<CIS-7-AccountState>`.
The GetBlockTransactionEvents and GetBlockItemStatus gRPC queries return CBOR-encoded :ref:`events<CIS-7-TokenModuleEvents>` and :ref:`reject reasons<CIS-7-RejectReasons>` for TokenUpdate transactions.

Terminology and Conventions
---------------------------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

The schemas for CBOR data are presented using Concise Data Definition Language (CDDL) as defined in :rfc:`8610`.

Example data values are presented in CBOR diagnostic notation as described in :rfc:`8949`.

Specification
=============

Except as otherwise specified, the Token Module MUST accept data in preferred serialization as defined in :rfc:`8949`.
The Token Module SHOULD accept well-formed CBOR data that is not in preferred serialization, but conforms to the required schema.
The Token Module MUST produce data in preferred serialization, and SHOULD produce data in Deterministically Encoded CBOR.


Common Types
------------

.. _CIS-7-TokenID:

Token ID
^^^^^^^^

::

    token-id = text .size (1..128) .regexp "[a-zA-Z0-9\-.%]+"

A Token ID is a text string that uniquely identifies a PLT.
It is a non-empty string of 1 to 128 characters, consisting of alphanumeric characters, as well as the characters "-", ".", and "%".
It is RECOMMENDED that Token IDs are limited to 6 characters, and are purely alphabetic.
Implementations MUST match Token IDs in a case-insensitive manner.

The Token ID is used to identify the PLT in transactions, events and queries.
It is not typically encoded in CBOR.

.. _CIS-7-TokenModuleHash:

Token Module Hash
^^^^^^^^^^^^^^^^^

::

    sha256-hash = bytes .size(32)

    token-module-hash = sha256-hash

A Token Module Hash identifies a particular implementation of a Token Module.
It should be a 32-byte SHA256 hash.

The Token Module Hash is used to identify the Token Module implementation in transactions, events and queries.
It is not typically encoded in CBOR.

Protocol version 9 introduces a single Token Module implementation, referred to as TokenModuleV0, which is identified by the Token Module Hash ``5c5c2645db84a7026d78f2501740f60a8ccb8fae5c166dc2428077fd9a699a4a``.

.. _CIS-7-TokenDecimals:

Token Decimals
^^^^^^^^^^^^^^
::

    token-decimals = int .range (0..255)

The token decimals for a PLT is an integer that specifies the number of decimal places in the token's representation.
It is a constant that is determined when the PLT is created.
As of protocol version 9, it is constrained to be between 0 and 255 (inclusive).

.. _CIS-7-TokenAmount:

Token Amount
^^^^^^^^^^^^
::

  token-amount = decfrac

A token amount is represented as a CBOR decimal fraction.
A decimal fraction is a tagged pair of a base-10 exponent and significand.
From the CDDL prelude (:rfc:`8610`)::

  decfrac = #6.4([e10: int, m: integer])

For ``token-amount``, as of protocol version 9, the exponent (``e10``) must be between -255 and 0 (inclusive).
The significand (``m``) must be between 0 and 2^64-1 = 18446744073709551615 (inclusive); that is, the significand is a 64-bit unsigned integer.

A ``token-amount`` for a given PLT MUST be expressed with the exponent ``e10`` being the negation of the :ref:`CIS-7-TokenDecimals`.
Thus, the following ``token-amount``\s are not equivalent::

  4([-2, 100])      -- 1.00
  4([-6, 1000000])  -- 1.000000

Memo
^^^^
::

    memo = raw-memo / cbor-memo

    raw-memo = bytes .size (0..256)

    cbor-memo = #6.24(raw-memo)

A ``memo`` is a byte string of up to 256 bytes.
The ``memo`` can be represented either directly as a byte string (``raw-memo``), or as a byte string tagged as CBOR-encoded data (``cbor-memo``).
The tag 24 is defined in :rfc:`8949#section-3.4.5.1` to denote that the enclosed byte string represents CBOR-encoded data.

The tagged ``cbor-memo`` format SHOULD NOT be used unless the memo data itself is valid CBOR.
The purpose of the tagged ``cbor-memo`` is to hint to a decoder that the contents is interpreted as CBOR, and allow it to displayed in decoded form, where appropriate.

Account Address
^^^^^^^^^^^^^^^
::

    tagged-account-address = #6.40307(untagged-account-address)

    untagged-account-address = {
        ; If the info (1) field is present, it must indicate CCD.
        ? 1: tagged-ccd-coininfo,
        ; The type (2) field is not supported.
        ; The data (3) field must be the 32-byte representation of a Concordium address
        3: bytes .size 32
    }

    ; A subtype of the tagged-coininfo type from BCR-2020-007
    tagged-ccd-coininfo = #6.40305(ccd-coininfo)

    ccd-coininfo = {
        ; The type (1) field is the SLIP44 code for Concordium
        1: 919
        ; The network (2) field is not supported.
    }

Accounts are represented by ``tagged-account-address``, which is based on the UR Type Definition for Cryptocurrency Addresses as defined in `BCR-2020-009 <https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-009-address.md>`_.
The tag 40307 denotes a cryptocurrency address.
The ``untagged-account-address`` consists of an optional info field (key ``1``) that indicates the address is specifically a Concordium address.
The type field (key ``2``) defined by BCR-2020-009 is not supported for Concordium account addresses, and is therefore omitted.
The data field (key ``3``) is required and MUST be the 32-byte representation of the Concordium account address.

When present, the info field SHOULD hold the value ``40305({1: 919})``.
The tag 40305 denotes a coin info type as defined in `BCR-2020-007 <https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-007-hdkey.md>`_.
The info field MAY be omitted.
Decoders SHOULD assume that a tagged cryptocurrency address with no info field represents a Concordium address.

The coin info structure consists of the type field (key ``1``) which holds value 919 for Concordium, which is the code assigned under `SLIP44 <https://github.com/satoshilabs/slips/blob/master/slip-0044.md>`_.
The network field (key ``2``) is not supported and therefore omitted.

When rendering a ``tagged-account-address`` in a human-readable format, it SHOULD be displayed in the standard base58 check encoding.

Smart Contract Address
^^^^^^^^^^^^^^^^^^^^^^

A Concordium smart contract address is a pair of unsigned 64-bit integers: the *index* and *subindex*.
A smart contract address is conventionally represented as <*index*, *subindex*>.
Smart contract addresses with a subindex other than 0 are unused, but reserved for future use.

::

    ; A Concordium smart contract address.
    tagged-contract-address = #6.40919(contract-address)

    contract-address = contract-address-index-only / contract-address-index-subindex

    ; A smart contract address represented as the contract index only.
    ; The subindex is implied to be 0.
    contract-address-index-only = uint

    ; A smart contract address represented as a pair of the index and subindex.
    contract-address-index-subindex = [
        index: uint,
        subindex: uint
    ]

A smart contract address is represented by ``tagged-contract-address``.
The tag 40919 denotes a Concordium smart contract address.
The smart contract address is either represented as only the index, in which case the subindex is implicitly 0, or as an ordered pair of the index and subindex.

A smart contract address with subindex 0 has two possible representations.
Encoders SHOULD use the ``contract-address-index-only`` representation for such addresses.
Decoders MUST accept the ``contract-address-index-only`` representation.
Decoders MUST accept the ``contract-address-index-subindex`` representation with subindex 0, unless deterministic encoding is required.

.. _CIS-7-MetadataURL:

Metadata URL
^^^^^^^^^^^^

::

    metadata-url = {
        ; A string field representing the URL
        "url": text,
        ; An optional sha256 checksum value tied to the content of the URL
        ? "checksumSha256": sha256-hash
        ; Additional fields may be included for future extensibility, e.g. another hash algorithm.
        * text => any
    }

A ``metadata-url`` encodes a URL that identifies metadata, together with an optional sha256 checksum of the contents of the metadata.
When the ``checksumSha256`` field is present, tools SHOULD confirm that the computed sha256 hash of the data retrieved from the URL specified by the ``url`` field matches the contents of the ``checksumSha256`` field.

.. _CIS-7-InitializationParameters:

Initialization Parameters
-------------------------

The initialization parameters are used when creating a new PLT instance.
They are included as part of the CreatePLT chain update transaction.
They are passed to the Token Module to initialize the state.
Note that the CreatePLT chain update includes additional parameters that are separate from the initialization parameters: the :ref:`CIS-7-TokenID`, the :ref:`CIS-7-TokenModuleHash`, and the :ref:`CIS-7-TokenDecimals`.

The format and semantics of the initialization parameters may differ between Token Module implementations.
The initializations parameters for a conforming implementation MUST be represented as a CBOR map conforming to the following schema:
::

    token-initialization-parameters = {
        ; The name of the token
        ? "name": text,
        ; A URL pointing to the token metadata
        ? "metadata": metadata-url,
        ; The governance account of the token
        ? "governanceAccount": tagged-account-address,
        ; Whether the token enforces an allow list
        ? "allowList": bool .default false,
        ; Whether the token enforces a deny list
        ? "denyList": bool .default false,
        ; The initial supply of the token. If not present, no tokens are minted initially.
        ? "initialSupply": token-amount,
        ; Whether the token is mintable
        ? "mintable": bool .default false,
        ; Whether the token is burnable
        ? "burnable": bool .default false,
        ; Additional fields
        * text => any
    }

The schema defines a number of standardized fields, while allowing for additional fields that may be defined by future standards.
The semantics of the standardized fields are defined below.

The TokenModuleV0 implementation requires the ``name``, ``metadata``, and ``governanceAccount`` fields.
The ``allowList``, ``denyList``, ``initialSupply``, ``mintable``, and ``burnable`` fields are optional.
All other fields are prohibited.


.. _CIS-7-name:

``name``
^^^^^^^^

The full name of the token.

.. _CIS-7-metadata:

``metadata``
^^^^^^^^^^^^

A URL pointing to the token metadata JSON object, and optionally a hash of the metadata.

.. _CIS-7-governanceAccount:

``governanceAccount``
^^^^^^^^^^^^^^^^^^^^^

The singular *governance account* that is permitted to perform mint, burn, pause, and list-update governance operations, if they are enabled for the token.
A PLT with a governance account MUST NOT allow accounts other than the governance account to perform token-governance operations.
The :ref:`token module state <CIS-7-TokenModuleState>` MUST indicate the governance account, if it exists.

Token Modules may implement different access control mechanisms (such as role-based access control) that permit different accounts to perform token-governance operations.
Such mechanisms are not specified in the current standard, but may be incompatible with having a singular governance account as defined above.

.. _CIS-7-allowList:

``allowList``
^^^^^^^^^^^^^

Whether the PLT enforces an allow list.
A PLT that enforces an allow list is subject to the following:

* Transfers MUST be rejected unless both the sender and receiver accounts belong to the allow list.

* The :ref:`token module state <CIS-7-TokenModuleState>` MUST indicate that the allow list is enforced.

* Accounts with no :ref:`account state <CIS-7-AccountState>` implicitly MUST NOT belong to the allow list.

* Accounts that have an account state MUST report whether the account belongs to the allow list.

* The ``addAllowList`` and ``removeAllowList`` operations SHOULD be implemented.

* When an account is added to the allow list, an ``addAllowList`` Token Module Event MUST be emitted.

* When an account is removed from the allow list, a ``removeAllowList`` Token Module Event MUST be emitted.

If the value is not specified, the PLT MUST NOT enforce an allow list.

.. _CIS-7-denyList:

``denyList``
^^^^^^^^^^^^

Whether the PLT implements a deny list.
A PLT that implements a deny list is subject to the following:

* Transfers MUST be rejected if either the sender or receiver account belongs to the deny list.

* The :ref:`token module state<CIS-7-TokenModuleState>` MUST indicate that the deny list is enforced.

* Accounts with no :ref:`account state<CIS-7-AccountState>` implicitly MUST NOT belong to the deny list.

* Accounts that have an account state MUST report whether the account belongs to the deny list.

* The ``addDenyList`` and ``removeDenyList`` operations SHOULD be implemented.

* When an account is added to the deny list, an ``addDenyList`` Token Module Event MUST be emitted.

* When an account is removed from the deny list, a ``removeDenyList`` Token Module Event MUST be emitted.

If the value is not specified, the PLT MUST NOT enforce a deny list.

``initialSupply``
^^^^^^^^^^^^^^^^^

The initial supply of the PLT that is minted when the token is created.
If this is not specified, no initial supply is minted.

.. _CIS-7-mintable:

``mintable``
^^^^^^^^^^^^

Whether the PLT supports the ``mint`` transaction operation.

.. _CIS-7-burnable:

``burnable``
^^^^^^^^^^^^

Whether the PLT supports the ``burn`` transaction operation.

.. _CIS-7-Transactions:

Transactions
------------

A Token Update transaction identifies a PLT by its Token ID and carries a CBOR-encoded payload that consists of a list of token operations (``token-update-transaction``).
The Token Module MUST execute the token operations in sequence.
If any of the token operations fails, the entire transaction SHOULD fail with the reject reason indicating the cause of failure of the first failing operation.
Energy fees SHOULD be charged for each operation up to and including the first failing operation.

If a Token Update transaction cannot be deserialized, the transaction SHOULD fail with the reject reason ``deserializationFailure``.
A token amount that does not conform to the :ref:`CIS-7-TokenDecimals` SHOULD be considered a deserialization failure.
TokenModuleV0 deserializes the transaction in its entirety before executing any of the operations, and thus no charge is levied for any operations if deserialization fails.

::

    token-update-transaction = [ * token-operation ]

    token-operation = token-transfer
        / token-mint
        / token-burn
        / token-update-list
        / token-pause
        / token-unpause

The token operations presented here are those implemented by TokenModuleV0.
Different Token Module implementations may implement a different set of operations.
However, the payload MUST always consist of a CBOR list of token operations.
Each token operation MUST consist of a map with a single key that identifies the operation type.

The semantics of each token operation SHOULD be the same across all Token Modules which implements it.
In particular, implementations MUST conform to the schema for the token operations defined in this document.
An implementation MUST NOT use the operation types ``transfer``, ``mint``, ``burn``, ``addAllowList``, ``removeAllowList``, ``addDenyList``, ``removeDenyList``, ``pause``, or ``unpause`` for any other operation than those defined below.

``transfer``
^^^^^^^^^^^^
::

    ; A token transfer operation. This transfers a specified amount of tokens from the sender account
    ; (implicit) to the recipient account.
    token-transfer = {
        ; The operation type is "transfer".
        "transfer": {
            ; The amount of tokens to transfer.
            "amount": token-amount,
            ; The recipient account.
            "recipient": tagged-account-address,
            ; An optional memo.
            ? "memo": memo
        }
    }

The token transfer operation transfers a specified amount of tokens from the sender account to the recipient account, generating a transfer event.
The transfer event MUST record the ``memo`` if one is provided.
(Note that, while the ``memo`` in the transaction may be explicitly tagged as CBOR encoded, the generated transfer event does not retain this tagging.)

The transfer operation MUST fail if any of the following conditions are met:

- the token is paused;
- the recipient account does not exist;
- the token has an allow list and the sender is not on the allow list;
- the token has an allow list and the recipient is not on the allow list;
- the token has a deny list and the sender is on the deny list;
- the token has a deny list and the recipient is on the deny list; or
- the sender's balance is insufficient to complete the transfer.

The reject reason SHOULD indicate which condition caused the failure.
If multiple conditions apply, the reject reason can indicate any of them.

``mint`` and ``burn``
^^^^^^^^^^^^^^^^^^^^^
::

    ; Mint a specified amount to the sender account.
    token-mint = {
        ; The operation type is "mint".
        "mint": token-supply-update-details
    }

    ; Burn a specified amount from the sender account.
    token-burn = {
        ; The operation type is "burn".
        "burn": token-supply-update-details
    }

    ; Specifies the details of a mint/burn operation.
    token-supply-update-details = {
        ; The amount of tokens to either mint or burn.
        "amount": token-amount
    }

The mint operation increases the total supply of the token by the specified amount, issuing the new tokens to the sender account.
The burn operation decreases the total supply of the token by the specified amount, deducting the tokens from the sender account.
These operations are considered token-governance operations, and thus are not available to all accounts.

The mint operation MUST fail if any of the following conditions holds:

- the token has a governance account, and the sender account is not the governance account;
- the token is paused;
- the token is not mintable; or
- minting would cause the total supply of the token to exceed the maximum representable value for the token.

The burn operation MUST fail if any of the following conditions holds:

- the token has a governance account, and the sender account is not the governance account;
- the token is paused;
- the token is not burnable; or
- the balance of the sender account is less than the specified amount to burn.

The reject reason SHOULD indicate which condition caused the failure.
If multiple conditions apply, the reject reason can indicate any of them.

``addAllowList``, ``removeAllowList``, ``addDenyList``, and ``removeDenyList``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
::

    ; Update an allow or a deny list by adding or removing an account from it.
    token-update-list =
        token-add-allow-list
        / token-remove-allow-list
        / token-add-deny-list
        / token-remove-deny-list

    ; Add an account to the allow list.
    token-add-allow-list = {
        ; The operation type is "addAllowList".
        "addAllowList": token-list-update-details
    }

    ; Remove an account from the allow list.
    token-remove-allow-list = {
        ; The operation type is "removeAllowList".
        "removeAllowList": token-list-update-details
    }

    ; Add an account to the deny list.
    token-add-deny-list = {
        ; The operation type is "addDenyList".
        "addDenyList": token-list-update-details
    }

    ; Remove an account from the deny list.
    token-remove-deny-list = {
        ; The operation type is "removeDenyList".
        "removeDenyList": token-list-update-details
    }

    ; Specifies the details of a list-update operation.
    token-list-update-details = {
        ; The account to add or remove from the list.
        "target": tagged-account-address
    }

The list-update operations add or remove a specified account to or from the allow or deny list.
The list-update operations are considered token-governance operations, and thus are not available to all accounts.

A list-update operation MUST fail if any of the following conditions holds:

- the token has a governance account, and the sender account is not the governance account;
- the token does not implement the relevant list; or
- the target account does not exist.

Adding an account to a list that it already belongs to, or removing it from a list that it does not belong to, SHOULD NOT be considered grounds for failure.
Such updates MUST NOT affect the list.

If the token is paused, that SHOULD NOT be considered grounds for failure of a list-update operation, since the list-update operations do not involve balance changes.

The reject reason SHOULD indicate which condition caused the failure.
If multiple conditions apply, the reject reason can indicate any of them.

``pause`` and ``unpause``
^^^^^^^^^^^^^^^^^^^^^^^^^
::

    ; Suspend any current or future token operations involving
    ; balance changes. If any transaction submitted includes any such operation
    ; while the token is in its paused state, the transaction will fail. The
    ; suspension lasts until the token is unpaused with the corresponding
    ; `token-unpause` operation.
    token-pause = {
        "pause": {}
    }

    ; Unpause the token operations described in the `token-pause` operation,
    ; thus acting as an inverse of `token-pause`.
    token-unpause = {
        "unpause": {}
    }

The pause operation sets the token into a global pause state in which no balance-changing operations can be carried out.
The unpause operations ends the global pause.

The pause and unpause operations MUST fail if any of the following conditions holds:

- the token has a governance account, and the sender account is not the governance account.

Pausing a token that is already paused, or unpausing a token that is not paused, SHOULD NOT be considered grounds for failure.
Such updates MUST NOT affect whether the token is paused.

The reject reason SHOULD indicate which condition caused the failure.
If multiple conditions apply, the reject reason can indicate any of them.

Forward Compatibility
^^^^^^^^^^^^^^^^^^^^^

Token Modules MAY implement additional token operations that are not defined in this specification.
In order for tools such as hardware wallets to be able to handle such operations, these operations SHOULD conform to the following schema:
::

    generic-token-operation = {
        short-text => generic-token-operation-details
    }

    short-text = text .size (1..24)

    generic-token-operation-details = {
        * simple-key => details-value
    }

    simple-key = short-text / uint

    details-value = value-1

    value-1 = value-0
        / list-0
        / map-0

    list-0 = [ * value-0 ]

    map-0 = { * simple-key => value-0 }

    value-0 =
        tagged-account-address      ; An account address
        / tagged-contract-address   ; A smart contract address
        / int                       ; An integer
        / bigint                    ; A big integer
        / decfrac                   ; A decimal fraction
        / text                      ; A text string
        / bytes                     ; A byte string
        / epoch-time                ; An epoch time
        / encoded-cbor              ; Encoded CBOR data
        / base16-data               ; Data to be represented in base16
        / base64-data               ; Data to be represented in base64
        / bool                      ; A boolean value
        / null                      ; The null value
        / undefined                 ; The undefined value

    epoch-time = #6.1(uint)
    base16-data = #6.23(bytes)
    base64-data = #6.22(bytes)

A ``generic-token-operation`` consists of a short text key (1-24 characters) that identifies the operation, and a map of simple keys to values that represent the details of the operation.
Simple keys are either short text strings (1-24 characters) or unsigned integers.

The values can be of various types:

- ``tagged-account-address``: An account address.
- ``tagged-contract-address``: A smart contract address.
- ``int``: An integer value.
- ``bigint``: A big integer value.
- ``decfrac``: A decimal fraction.
- ``text``: A text string.
- ``bytes``: A byte string.
- ``epoch-time``: An time represented as a number of seconds since the Unix epoch (1970-01-01T00:00:00Z).
- ``encoded-cbor``: Encoded CBOR data. (Tooling may decode this data and display it in a human-readable format where appropriate.)
- ``base16-data``: Data to be represented in base16 (hexadecimal) format.
- ``base64-data``: Data to be represented in base64 format.
- ``bool``: A boolean value (true or false).
- ``null``: The null value.
- ``undefined``: The undefined value.
- ``list-0``: A list of values the above simple values.
- ``map-0``: A map of simple keys to simple values.


Token Kernel Events
-------------------

Token Kernel events are emitted by the Token Kernel as a consequence of transaction execution.
These events are emitted whenever the Token Module invokes the Token Kernel to perform a balance-changing update, or when the PLT is first initialized.

TokenTransfer
^^^^^^^^^^^^^

The TokenTransfer event occurs whenever an amount is transferred from one account to another.
It indicates the Token ID involved, the sender and recipient accounts, the amount transferred, and any memo associated with the transfer.

TokenMint
^^^^^^^^^

The TokenMint event occurs whenever an amount of tokens is minted (i.e. introduced into circulation).
It indicates the Token ID involved, the account that received the minted tokens, and the amount minted.

TokenBurn
^^^^^^^^^

The TokenBurn event occurs whenever an amount of tokens is burned (i.e. removed from circulation).
It indicates the Token ID involved, the account that burned the tokens, and the amount burned.

TokenCreated
^^^^^^^^^^^^

The TokenCreated event occurs when a new PLT is created.
It indicates the :ref:`CIS-7-TokenID`, the :ref:`CIS-7-TokenModuleHash`, the :ref:`CIS-7-TokenDecimals`, and the `initialization parameters <CIS-7-InitializationParameters>`_.

.. _CIS-7-TokenModuleEvents:

Token Module Events
-------------------

The Token Module may emit Token Module Events as a consequence of transaction execution.
These events are in addition to the `Token Kernel Events`, and the semantics is dependent on the Token Module implementation.

Each Token Module Event type is designated by a ``TokenEventType``, which is a UTF-8 encoded string of at most 255 bytes.
Each Token Module Event has a CBOR-encoded event details.
The ``TokenEventType`` determines the semantics of the event details, and in particular the schema to which it should conform.

``addAllowList``
^^^^^^^^^^^^^^^^
::

    ; The details of a token "addAllowList" event.
    ; Indicates that the account was added to the allow list.
    token-add-allow-list-event = token-list-update-details

``removeAllowList``
^^^^^^^^^^^^^^^^^^^
::

    ; The details of a token "removeAllowList" event.
    ; Indicates that the account was removed from the allow list.
    token-remove-allow-list-event = token-list-update-details

``addDenyList``
^^^^^^^^^^^^^^^
::

    ; The details of a token "addDenyList" event.
    ; Indicates that the account was added to the deny list.
    token-add-deny-list-event = token-list-update-details

``removeDenyList``
^^^^^^^^^^^^^^^^^^
::

    ; The details of a token "removeDenyList" event.
    ; Indicates that the account was removed from the deny list.
    token-remove-deny-list-event = token-list-update-details

``pause``
^^^^^^^^^
::

    ; The details of a token "pause" event.
    ; Indicates that the token operations involving balance changes are suspended.
    token-pause-event = {}

``unpause``
^^^^^^^^^^^
::

    ; The details of a token "unpause" event.
    ; Indicates that the token operations involving balance changes are resumed.
    token-unpause-event = {}

.. _CIS-7-RejectReasons:

Reject Reasons
--------------

The Token Module may reject a transaction for various reasons.
When a transaction is rejected, the reject reason identifies the PLT, the type of the reject reason (a UTF-8 encoded string of at most 255 bytes), and, optionally, the details of the reject reason (encoded as CBOR).

Then the Token Module rejects a transaction, it produces a "token update transaction failed" reject reason that includes the Token ID, the reject reason type, and (optionally) CBOR-encoded reject reason details.
A TokenUpdate transaction may also be rejected for a reason outside of the control of the Token Module.
In particular "non existent Token ID" and "out of energy" reject reasons are possible.

As with Token Module Events, the reject reason type determines the semantics of the reject reason details, and in particular the schema to which it should conform.
The following reject reason types are defined by TokenModuleV0:

``addressNotFound``
^^^^^^^^^^^^^^^^^^^
::

    ; "addressNotFound": an account address was not valid.
    reject-details-address-not-found = {
        ; The index in the list of operations of the failing operation.
        "index": uint,
        ; The address that could not be resolved.
        "address": tagged-account-address
    }

``tokenBalanceInsufficient``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
::

    ; "tokenBalanceInsufficient": the balance of tokens on the sender account is insufficient
    ; to perform the operation.
    reject-details-token-balance-insufficient = {
        ; The index in the list of operations of the failing operation.
        "index": uint,
        ; The available balance of the sender.
        "availableBalance": token-amount,
        ; The minimum required balance to perform the operation.
        "requiredBalance": token-amount
    }

``deserializationFailure``
^^^^^^^^^^^^^^^^^^^^^^^^^^
::

    ; "deserializationFailure": the transaction could not be deserialized.
    reject-details-deserialization-failure = {
        ; Text description of the failure mode.
        ? "cause": text
    }

Note that it is considered a deserialization failure if the transaction contains a :ref:`CIS-7-TokenAmount` that does not conform to the :ref:`CIS-7-TokenDecimals`.

``unsupportedOperation``
^^^^^^^^^^^^^^^^^^^^^^^^
::

    ; "unsupportedOperation": the operation is not supported by the token module.
    ; This may be because the operation is not implemented by the module, or because the
    ; token is not configured to support the operation. If the operation is not authorized
    ; (i.e. the particular participants do not have the authority to perform the operation)
    ; then the reject reason is "operationNotPermitted" instead.
    reject-details-unsupported-operation = {
        ; The index in the list of operations of the failing operation.
        "index": uint,
        ; The type of operation that was not supported.
        "operationType": text,
        ; The reason why the operation was not supported.
        ? "reason": text
    }

``operationNotPermitted``
^^^^^^^^^^^^^^^^^^^^^^^^^^
::

    ; "operationNotPermitted": the operation requires that a participating account has a certain
    ; permission, but the account does not have that permission.
    reject-details-operation-not-permitted = {
        ; The index in the list of operations of the failing operation.
        "index": uint,
        ; (Optionally) the address that does not have the necessary permissions to perform the
        ; operation.
        ? "address": tagged-account-address,
        ; The reason why the operation is not permitted.
        ? "reason": text
    }

``mintWouldOverflow``
^^^^^^^^^^^^^^^^^^^^^
::

    ; "mintWouldOverflow": minting the requested amount would overflow the representable token amount.
    reject-details-mint-would-overflow = {
        ; The index in the list of operations of the failing operation.
        "index": uint,
        ; The requested amount to mint.
        "requestedAmount": token-amount,
        ; The current supply of the token.
        "currentSupply": token-amount,
        ; The maximum representable token amount.
        "maxRepresentableAmount": token-amount,
    }


.. _CIS-7-TokenModuleState:

Token Module State
------------------

The Token Module state is a representation of the global state of a PLT, which is maintained by the Token Module.
It is returned as part of a `GetTokenInfo` query.
The Token Module state does not include state that is managed by the Token Kernel, such as the Token ID and global supply.
It also does not (typically) include account-specific state, which is returned as part of `GetAccountInfo` instead.
The Token Module state is represented as a CBOR map conforming to the following schema:

::

    token-module-state = {
        ; The name of the token
        ? "name": text,
        ; A URL pointing to the token metadata
        ? "metadata": metadata-url,
        ; The governance account of the token
        ? "governanceAccount": tagged-account-address
        ; Whether the token enforces an allow list.
        ? "allowList": bool,
        ; Whether the token enforces a deny list.
        ? "denyList": bool,
        ; Whether the token is mintable.
        ? "mintable": bool,
        ; Whether the token is burnable.
        ? "burnable": bool,
        ; Whether the token is paused, i.e. operations involving balance changes are suspended.
        ? "paused": bool,
        ; Additional state information may be provided under further text keys, the meaning
        ; of which are not defined in the present specification.
        * text => any
    }

All fields are optional.
It is RECOMMENDED that Token Modules provide the ``name`` and ``metadata`` fields.
The structure supports additional fields for future extensibility.

A Token Module MAY include non-standard fields (i.e. any fields that are not defined by a standard, and are specific to the module implementation).
These non-standard fields SHOULD be prefixed with an underscore ("_") to distinguish them as such.
For example, a Token Module may include a field ``"_customField"`` with a value that is specific to the module implementation.
The semantics of such non-standard fields are not defined by this specification, and are specific to the module implementation.

The fields :ref:`name<CIS-7-name>`, :ref:`metadata<CIS-7-metadata>`, :ref:`governanceAccount<CIS-7-governanceAccount>`, :ref:`allowList<CIS-7-allowList>`, :ref:`denyList<CIS-7-denyList>`, :ref:`mintable<CIS-7-mintable>`, and :ref:`burnable<CIS-7-burnable>` have the same semantics as in ``token-initialization-parameters``.

``pause``
^^^^^^^^^

Whether the token is currently paused.
When the token is paused, any transaction that includes a balance-changing operation (such as a transfer, mint, or burn) MUST fail.
Other operations (such as adding or removing accounts from the allow or deny list) SHOULD NOT be affected by the pause state.

.. _CIS-7-AccountState:

Account State
-------------

The account state represents account-specific information that is maintained by the Token Module.
It is returned as part of a `GetAccountInfo` query.
The account state does not include state that is managed by the Token Kernel, such as the Token ID and account balance.
It is represented as a CBOR map conforming to the following schema:

::

    token-module-account-state = {
        ; Whether the account is on the allow list.
        ; This is only present if the token supports an allow list; that is accounts can only
        ; send or receive tokens if they are on the allow list.
        ? "allowList": bool,
        ; Whether the account is on the deny list.
        ; This is only present if the token supports a deny list; that is accounts can only
        ; send or receive tokens if they are not on the deny list.
        ? "denyList": bool,
        ; Additional state information may be provided under further text keys, the meaning
        ; of which are not defined in the present specification.
        * text => any
    }

All fields are optional.
The structure supports additional fields for future extensibility.

A Token Module MAY include non-standard fields (i.e. any fields that are not defined by a standard, and are specific to the module implementation).
These non-standard fields SHOULD be prefixed with an underscore ("_") to distinguish them as such.


Token Metadata Format
---------------------

While some token metadata (such as the Token ID, name and number of decimals) is stored on-chain, additional metadata is stored off-chain and referenced by the ``metadata`` field in the Token Module state.
This token metadata MUST be a JSON (:rfc:`8259`) file.

All of the fields in the JSON file are optional, and this specification reserves a number of field names, shown in the table below.

.. list-table:: Token metadata JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``name`` (optional)
    - string
    - The name of the token (used for localization).
  * - ``description`` (optional)
    - string
    - A description for this token type.
  * - ``thumbnail`` (optional)
    - URL JSON object
    - An image URL to a small image for displaying the asset.
  * - ``display`` (optional)
    - URL JSON object
    - An image URL to a large image for displaying the asset.
  * - ``localization`` (optional)
    - JSON object with locales as field names (:rfc:`5646`) and field values are URL JSON objects linking to JSON files.
    - URLs to JSON files with localized token metadata.

To enforce integrity of the metadata, the SHA256 hash of the JSON file MAY be included as part of the :ref:`CIS-7-MetadataURL`.
Since the metadata JSON file can itself contain URLs, a SHA256 hash MAY be associated with each URL.
To associate a hash with a URL, the JSON value is an object:

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
