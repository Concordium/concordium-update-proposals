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
- Token state
- Account state

Introduction
============

Protocol-level tokens (PLTs) provide chain-native support for tokens (other than CCD) without depending on smart contracts for the implementation.
Each PLT is governed by a Token Module, which manages the token's lifecycle, including creation, transfer, and burning of tokens.
There may be a single Token Module that manages all PLTs, or there may be multiple Token Modules, each managing one or more PLTs.

The Token Module defines a number of interfaces for interaction with PLTs.
At each interface, data is exchanged in Concise Binary Object Representation (CBOR) format.
The CBOR format is defined in :rfc:`8949`.
This document defines schemas for the data exchanged at each interface, including initialization parameters, transactions, events, reject reasons, token state, and account state.

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

.. _CIS-7-token-amount:

``token-amount``
^^^^^^^^^^^^^^^^
::

  token-amount = decfrac

A token amount is represented as a CBOR decimal fraction.
A decimal fraction is a tagged pair of a base-10 exponent and significand.
From the CDDL prelude (:rfc:`8610`)::

  decfrac = #6.4([e10: int, m: integer])

For ``token-amount``, as of protocol version 9, the exponent (``e10``) must be between 0 and -255 (inclusive).
The significand (``m``) must be between 0 and 18446744073709551615 (inclusive).


Each PLT defines the number of decimal places in its representation.
A ``token-amount`` for a given PLT MUST be expressed with the exponent ``e10`` being the negation of the number of decimals.
Thus, the following ``token-amount``\s are not equivalent::

  4([2, 100])      -- 1.00
  4([6, 1000000])  -- 1.000000

``memo``
^^^^^^^^
::

    memo = raw-memo / cbor-memo

    raw-memo = bytes .size (0..256)

    cbor-memo = #6.24(raw-memo)

A ``memo`` is a byte string of up to 256 bytes.
The ``memo`` can be represented either directly as a byte string (``raw-memo``), or as a byte string tagged as CBOR-encoded data (``cbor-memo``).
The tag 24 is defined in :rfc:`8949#section-3.4.5.1` to denote that the enclosed byte string represents CBOR-encoded data.

The tagged ``cbor-memo`` format SHOULD NOT be used unless the memo data itself is valid CBOR.
The purpose of the tagged ``cbor-memo`` is to hint to a decoder that the contents is interpreted as CBOR, and allow it to displayed in decoded form, where appropriate.

``tagged-account-address``
^^^^^^^^^^^^^^^^^^^^^^^^^^
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
The data field (key ``3``) is required and must be the 32-byte representation of the Concordium account address.

When present, the info field should hold the value ``40305({1: 919})``.
The tag 40305 denotes a coin info type as defined in `BCR-2020-007 <https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-007-hdkey.md>`_.
The info field MAY be omitted.
Decoders SHOULD assume that a tagged cryptocurrency address with no info field represents a Concordium address.

The coin info structure consists of the type field (key ``1``) which holds value 919 for Concordium, which is the code assigned under `SLIP44 <https://github.com/satoshilabs/slips/blob/master/slip-0044.md>`_.
The network field (key ``2``) is not supported and therefore omitted.

When rendering a ``tagged-account-address`` in a human-readable format, it SHOULD be displayed in the standard base58 check encoding.

``metadata-url``

::

    metadata-url = {
        ; A string field representing the URL
        "url": text,
        ; An optional sha256 checksum value tied to the content of the URL
        ? "checksumSha256": sha256-hash
        ; Additional fields may be included for future extensibility, e.g. another hash algorithm.
        * text => any
    }

    sha256-hash = bytes .size(32)

A ``metadata-url`` encodes a URL that identifies metadata, together with an optional sha256 checksum of the contents of the metadata.
When the ``checksumSha256`` field is present, tools SHOULD confirm that the computed sha256 hash of the data retrieved from the URL specified by the ``url`` field matches the contents of the ``checksumSha256`` field.


Initialization Parameters
-------------------------

The initialization parameters are used when creating a new PLT instance.
They are included as part of the CreatePLT chain update transaction.
They are passed to the Token Module to initialize the state.
Note that the CreatePLT chain update includes additional parameters that are separate from the initialization parameters: the Token ID, the Token Module Reference, and the number of decimal places in the token's representation.

The format and semantics of the initialization parameters may differ between Token Module implementations.
The format presented here is that used by the TokenModuleV0 implementation.
::

    token-initialization-parameters = { 
        ; The name of the token
        "name": text,
        ; A URL pointing to the token metadata
        "metadata": metadata-url,
        ; The governance account of the token
        "governanceAccount": tagged-account-address
        ; Whether the token supports an allow list
        ? "allowList": bool .default false,
        ; Whether the token supports a deny list
        ? "denyList": bool .default false,
        ; The initial supply of the token. If not present, no tokens are minted initially.
        ? "initialSupply": token-amount,
        ; Whether the token is mintable
        ? "mintable": bool .default false,
        ; Whether the token is burnable
        ? "burnable": bool .default false
    }

Token Modules that use a different format for initialization parameters SHOULD represent the parameters in a key-value map.
Where keys that are the same as those above are used in initialization parameters, their semantics SHOULD be the same or substantially similar.


Transactions
------------

A Token Update transaction identifies a PLT by its Token ID and carries a CBOR-encoded payload that consists of a list of token operations (``token-update-transaction``).
::

    token-update-transaction = [ * token-operation ]

    token-operation = token-transfer
        / token-mint
        / token-burn
        / token-update-list

The token operations presented here are those implemented by TokenModuleV0.
Different Token Module implementations may implement a different set of operations.
However, the payload MUST always consist of a CBOR list of token operations.
Each token operation MUST consist of a map with a single key that identifies the operation type.

The semantics of each token operation SHOULD be the same across all Token Modules which implements it.
In particular, implementations MUST conform to the schema for the token operations defined in this document.
Implementation MUST NOT use the operation types ``transfer``, ``mint``, ``burn``, ``addAllowList``, ``removeAllowList``, ``addDenyList``, or ``removeDenyList`` for any other operation than those defined below.

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

    ; Specifies the details of a list update operation.
    token-list-update-details = {
        ; The account to add or remove from the list.
        "target": tagged-account-address
    }

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

    value-0 =
        tagged-account-address  ; An account address
        / int                   ; An integer
        / bigint                ; A big integer
        / decfrac               ; A decimal fraction
        / text                  ; A text string
        / bytes                 ; A byte string
        / epoch-time            ; An epoch time
        / encoded-cbor          ; Encoded CBOR data
        / base16-data           ; Data to be represented in base16
        / base64-data           ; Data to be represented in base64
        / bool                  ; A boolean value
        / null                  ; The null value
        / undefined             ; The undefined value
    
    epoch-time = #6.1(uint)
    base16-data = #6.23(bytes)
    base64-data = #6.22(bytes)

    list-0 = [ * value-0 ]
    map-0 = { * simple-key => value-0 }

    value-1 = value-0
        / list-0
        / map-0

    details-value = value-1

A `generic-token-operation` consists of a short text key (1-24 characters) that identifies the operation, and a map of simple keys to values that represent the details of the operation.
Simple keys are either short text strings (1-24 characters) or unsigned integers.

The values can be of various types:

- `tagged-account-address`: An account address.
- `int`: An integer value.
- `bigint`: A big integer value.
- `decfrac`: A decimal fraction.
- `text`: A text string.
- `bytes`: A byte string.
- `epoch-time`: An time represented as a number of seconds since the Unix epoch (1970-01-01T00:00:00Z).
- `encoded-cbor`: Encoded CBOR data. (Tooling may decode this data and display it in a human-readable format where appropriate.)
- `base16-data`: Data to be represented in base16 (hexadecimal) format.
- `base64-data`: Data to be represented in base64 format.
- `bool`: A boolean value (true or false).
- `null`: The null value.
- `undefined`: The undefined value.
- `list-0`: A list of values the above simple values.
- `map-0`: A map of simple keys to simple values.


Events
------

The Token Module may emit Token Module Events as a consequence of transaction execution.
These events are in addition to the ``TokenTransfer``, ``TokenMint``, ``TokenBurn`` and ``TokenCreated`` events, and the semanitcs is dependent on the Token Module implementation.

Each Token Module Event type is designated by a ``TokenEventType``, which is a UTF-8 enocded string of at most 255 bytes.
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
^^^^^^^^^^^^^^^^
::

    ; The details of a token "addDenyList" event.
    ; Indicates that the account was added to the deny list.
    token-add-deny-list-event = token-list-update-details

``removeDenyList``
^^^^^^^^^^^^^^^^^^^
::

    ; The details of a token "removeDenyList" event.
    ; Indicates that the account was removed from the deny list.
    token-remove-deny-list-event = token-list-update-details


Reject Reasons
--------------

The Token Module may reject a transaction for various reasons.
When a transaction is rejected, the reject reason identifies the PLT, the type of the reject reason (a UTF-8 encoded string of at most 255 bytes), and, optionally, the details of the reject reason (encoded as CBOR).

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
^^^^^^^^^^^^^^^^^^^^^^^^^^^
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

Token State
-----------

The Token Module state is a representation of the global state of a PLT, which is maintained by the Token Module.
It is returned as part of a `GetTokenInfo` query.
The Token Module state does not include state that is managed by the Token Kernel, such as the token identifier and global supply.
It also does not (typically) include account-specific state, which is returned as part of `GetAccountInfo` instead.
The Token Module state is represented as a CBOR map conforming to the following schema:

::

    token-module-state = {
        ; The name of the token
        "name": text,
        ; A URL pointing to the token metadata
        "metadata": metadata-url,
        ; The governance account of the token
        ? "governanceAccount": tagged-account-address
        ; Whether the token supports an allow list.
        ? "allowList": bool,
        ; Whether the token supports a deny list.
        ? "denyList": bool,
        ; Whether the token is mintable.
        ? "mintable": bool,
        ; Whether the token is burnable.
        ? "burnable": bool,
        ; Additional state information may be provided under further text keys, the meaning
        ; of which are not defined in the present specification.
        * text => any
    }

The ``name``, ``metadata``, and ``governanceAccount`` fields are required.
Other fields are optional, and can be omitted if the module implementation does not support them.
The structure supports additional fields for future extensibility.

A Token Module MAY include non-standard fields (i.e. any fields that are not defined by a standard, and are specific to the module implementation).
These non-standard fields SHOULD be prefixed with an underscore ("_") to distinguish them as such.
For example, a Token Module may include a field ``"_customField"`` with a value that is specific to the module implementation.
The semantics of such non-standard fields are not defined by this specification, and are specific to the module implementation.

Account State
-------------

The account state represents account-specific information that is maintained by the Token Module.
It is returned as part of a `GetAccountInfo` query.
The account state does not include state that is managed by the Token Kernel, such as the token identifier and account balance.
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

All fields are optional, and can be omitted if the module implementation does not support them.
The structure supports additional fields for future extensibility.

A Token Module MAY include non-standard fields (i.e. any fields that are not defined by a standard, and are specific to the module implementation).
These non-standard fields SHOULD be prefixed with an underscore ("_") to distinguish them as such.
