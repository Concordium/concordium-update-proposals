================================
CIS-1: Concordium Token Standard
================================

.. list-table::
   :stub-columns: 1

   * - Created
     - Sep 22, 2021
   * - Final
     - planned Nov 1, 2021
   * - Draft version
     - 3 (Oct 18, 2021)

Abstract
========

A standard interface for both fungible and non-fungible tokens implemented in a smart contract.
The interface provides functions for transferring token ownership, allowing other addresses to transfer tokens and for querying token balances and token metadata.
It allows for off-chain applications to track token balances and the location of token metadata using logged events.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

General types and serialization
-------------------------------

.. _CIS-1-TokenID:

``TokenID``
^^^^^^^^^^^

Token Identifier, which combined with the address of the smart contract instance implementing CIS1, forms the globally unique identifier of a token type.

- A token ID for a token type SHALL NOT change after a token type have been minted.
- A token ID for a token type SHALL NOT be reused for another token type within the same smart contract.

It is serialized as 1 byte for the size (``n``) of the identifier, followed by this number of bytes for the token id (``id``)::

  TokenID ::= (n: Byte) (id: Byteⁿ)

.. note::

  Token IDs can be as small as a single byte (by setting the first byte to the value 0) or as big as 256 bytes leaving more than 10^614 possible token IDs.
  The token ID could be an encoding of a small text string or some checksum hash, but to save energy it is still recommended to use small token IDs if possible.

.. _CIS-1-TokenAmount:

``TokenAmount``
^^^^^^^^^^^^^^^

An amount of a token type is an unsigned 64 bit integer.

It is serialized using 8 bytes little endian::

  TokenAmount ::= (amount: Byte⁸)

.. _CIS-1-ReceiveHookName:

``ReceiveHookName``
^^^^^^^^^^^^^^^^^^^

A smart contract receive function name.
A receive function name is prefixed with the contract name, followed by a ``.`` and a name for the function.
It MUST consist only of ASCII alphanumeric or punctuation characters.
The contract name is not allowed to contain ``.``.

It is serialized as: the function name byte length (``n``) is represented by the first 2 bytes, followed by this many bytes for the function name (``name``).
The receive function name MUST be 100 bytes or less::

  ReceiveHookName ::= (n: Byte²) (name: Byteⁿ)

.. note::

  This type is passed in a parameter for smart contract function calls, be aware of the parameter size limit of 1024 bytes.

.. _CIS-1-ContractName:

``ContractName``
^^^^^^^^^^^^^^^^

A name of a smart contract.
It must be prefixed with ``init_`` and MUST consist only of ASCII alphanumeric or punctuation characters.
The contract name is not allowed to contain ``.``.

It is serialized as: the contract name byte length (``n``) is represented by the first 2 bytes, followed by this many bytes for the contract name (``name``).
The contract name MUST be 100 bytes or less::

  ContractName ::= (n: Byte²) (name: Byteⁿ)

.. _CIS-1-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^^

An address of an account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte³²)

.. _CIS-1-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex both unsigned 64 bit integers.

It is serialized as: first 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``) both little endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)

.. _CIS-1-Address:

``Address``
^^^^^^^^^^^

Is either an account address or a contract address.

It is serialized as: First byte indicates whether it is an account address or a contract address.
In case the first byte is 0 then ``AccountAddress`` (``address``) is followed.
In case the first byte is 1 then ``ContractAddress`` (``address``) is followed::

  Address ::= (0: Byte) (address: AccountAddress)
            | (1: Byte) (address: ContractAddress)


.. _CIS-1-Receiver:

``Receiver``
^^^^^^^^^^^^

The receiving address of a transfer, which is either an account address or a contract address.
In the case of a contract address: a name of the hook receive function to invoke is also needed.

It is serialized as: First byte indicates whether it is an account address or a contract address.
In case the first byte is 0 then ``AccountAddress`` (``address``) is followed.
In case the first byte is 1 then ``ContractAddress`` (``address``), bytes for :ref:`CIS-1-ReceiveHookName` (``hook``) is followed::

    Receiver ::= (0: Byte) (address: AccountAddress)
               | (1: Byte) (address: ContractAddress) (hook: ReceiveHookName)

.. _CIS-1-AdditionalData:

``AdditionalData``
^^^^^^^^^^^^^^^^^^^

Additional bytes to include in a transfer, which can be used to add additional parameters for the transfer function call.

It is serialized as: the first 2 bytes encode the length (``n``) of the data, followed by this many bytes for the data (``data``)::

  AdditionalData ::= (n: Byte²) (data: Byteⁿ)

.. note::

  This type is passed in a parameter for smart contract function calls.
  Be aware of the parameter size limit of 1024 bytes.

.. _CIS-1-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL and optional checksum for metadata stored outside of this contract.

It is serialized as: 2 bytes for the length of the metadata url (``n``) and then this many bytes for the url to the metadata (``url``) followed by an optional checksum.
The checksum is serialized by 1 byte to indicate whether a hash of the metadata is included, if its value is 0, then no content hash, if the value is 1 then 32 bytes for a SHA256 hash (``hash``) is followed::

  MetadataChecksum ::= (0: Byte)
                     | (1: Byte) (hash: Byte³²)

  MetadataUrl ::= (n: Byte²) (url: Byteⁿ) (checksum: MetadataChecksum)

.. _CIS-1-functions:

Contract functions
------------------

A smart contract implementing CIS1 MUST export three functions :ref:`CIS-1-functions-transfer`, :ref:`CIS-1-functions-updateOperator` and :ref:`CIS-1-functions-balanceOf` according to the following description:

.. _CIS-1-functions-transfer:

``transfer``
^^^^^^^^^^^^

Executes a list of token transfers.
A transfer is a token ID, an amount of tokens to be transferred, and the ``from`` address and ``to`` address.

When transferring tokens to a contract address additional information for a receive function hook to trigger is required.

Parameter
~~~~~~~~~

The parameter is a list of transfers.

It is serialized as: 2 bytes representing the number of transfers (``n``) followed by the bytes for this number of transfers (``transfers``).
Each transfer is serialized as: a :ref:`CIS-1-TokenID` (``id``), a :ref:`CIS-1-TokenAmount` (``amount``), the token owner address :ref:`CIS-1-Address` (``from``), the receiving address :ref:`CIS-1-Receiver` (``to``) and some additional data (``data``)::

  Transfer ::= (id: TokenID) (amount: TokenAmount) (from: Address) (to: Receiver) (data: AdditionalData)

  TransferParameter ::= (n: Byte²) (transfers: Transferⁿ)

.. note::

  Be aware of the smart contract parameter size limit of 1024 bytes.
  Since the byte size of a single transfer can vary in size, this will limit the number of transfers that can be included in the same function call.
  Currently, with the smallest possible transfers, the parameter can contain 21 transfers and with the biggest possible transfer, it will take the whole parameter.

.. _CIS-1-functions-transfer-receive-hook-parameter:

Receive hook parameter
~~~~~~~~~~~~~~~~~~~~~~

The parameter for the receive function hook contains information about the transfer, the name of the token contract and some additional data bytes.

It is serialized as: a :ref:`CIS-1-TokenID` (``id``), a :ref:`CIS-1-TokenAmount` (``amount``), the token owner address :ref:`CIS-1-Address` (``from``), the name of the token contract :ref:`CIS-1-ContractName` (``contract``) and :ref:`CIS-1-AdditionalData` (``data``)::

  ReceiveHookParameter ::= (id: TokenID) (amount: TokenAmount) (from: Address) (contract: ContractName) (data: AdditionalData)

Requirements
~~~~~~~~~~~~

- The list of transfers MUST be executed in order.
- The contract function MUST reject if any of the transfers fails to be executed.
- A transfer MUST fail if:

  - The token balance of the ``from`` address is insufficient to do the transfer with error :ref:`INSUFFICIENT_FUNDS<CIS-1-rejection-errors>`.
  - TokenID is unknown with error: :ref:`INVALID_TOKEN_ID<CIS-1-rejection-errors>`.

- A transfer MUST non-strictly decrease the balance of the ``from`` address and non-strictly increase the balance of the ``to`` address or fail.
- A transfer with the same address as ``from`` and ``to`` MUST be executed as a normal transfer.
- A transfer of a token amount zero MUST be executed as a normal transfer.
- A transfer of some amount of a token type MUST only transfer the exact amount of the given token type between balances.
- A transfer of any amount of a token type to a contract address MUST call receive hook function on the receiving smart contract with a :ref:`receive hook parameter<CIS-1-functions-transfer-receive-hook-parameter>`.
- Let ``operator`` be an operator of the address ``owner``. A transfer of any amount of a token type from an address ``owner`` sent by an address ``operator`` MUST be executed as if the transfer was sent by ``owner``.
- The contract function MUST reject if a receive hook function called on the contract receiving tokens rejects.
- The balance of an address not owning any amount of a token type SHOULD be treated as having a balance of zero.

.. warning::

  Be aware of transferring tokens to a non-existing account address.
  This specification by itself does not include a mechanism to recover these tokens.
  Checking the existence of an account address would ideally be done off-chain before the message is even sent to the token smart contract.

.. _CIS-1-functions-updateOperator:

``updateOperator``
^^^^^^^^^^^^^^^^^^

Add or remove a number of addresses as operators of the address sending this message.

Parameter
~~~~~~~~~

The parameter contains a list of operator updates. An operator update contains information whether to add or remove an operator and the address to add/remove as operator.
It does not contain the address which is adding/removing the operator as this will be the sender of the message invoking this function.

The parameter is serialized as: first 2 bytes (``n``) for the number of updates followed by this number of operator updates (``updates``).
An operator update is serialized as: 1 byte (``update``) indicating whether to remove or add an operator, where if the byte value is 0 the sender is removing an operator, if the byte value is 1 the sender is adding an operator.
The is followed by the operator address (``operator``) :ref:`CIS-1-Address` to add or remove as operator for the sender::

  OperatorUpdate ::= (0: Byte) // Remove operator
                   | (1: Byte) // Add operator

  UpdateOperator ::= (update: OperatorUpdate) (operator: Address)

  UpdateOperatorParameter ::= (n: Byte²) (updates: UpdateOperatorⁿ)

Requirements
~~~~~~~~~~~~

- The list of updates MUST be executed in order.
- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The contract function MUST reject if any of the updates fails to be executed.

.. _CIS-1-functions-balanceOf:

``balanceOf``
^^^^^^^^^^^^^

Query balances of a list of addresses and token IDs, the result is then sent to a provided contract address.

Parameter
~~~~~~~~~

The parameter consists of a name of the contract address and receive function to callback with the result and a list of token ID and address pairs.

It is serialized as: a :ref:`CIS-1-ContractAddress` (``resultContract``) then a :ref:`CIS-1-ReceiveHookName` (``resultHook``) followed by 2 bytes for the number of queries (``n``) and then this number of queries (``queries``).
A query is serialized as :ref:`CIS-1-TokenID` (``id``) followed by :ref:`CIS-1-Address` (``address``)::

  BalanceOfQuery ::= (id: TokenID) (address: Address)

  BalanceOfParameter ::= (resultContract: ContractAddress) (resultHook: ReceiveHookName) (n: Byte²) (queries: BalanceOfQueryⁿ)

.. note::

  Be aware of the size limit on contract function parameters which currently is 1024 bytes, which puts a limit on the number of queries depending on the byte size of the Token ID and the name of the receive function.

Result parameter
~~~~~~~~~~~~~~~~

The parameter for the callback receive function is a list of query and token amount pairs.

It is serialized as: 2 bytes for the number of query-amount pairs (``n``) and then this number of pairs (``results``).
A query-amount pair is serialized as a query (``query``) and then a :ref:`CIS-1-TokenAmount` (``amount``)::

  BalanceOfQueryResult ::= (query: BalanceOfQuery) (balance: TokenAmount)

  BalanceOfResultParameter ::= (n: Byte²) (results: BalanceOfQueryResultⁿ)

Requirements
~~~~~~~~~~~~

- The balance of an address not owning any amount of a token type SHOULD be treated as having a balance of zero.
- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The contract function MUST NOT add or remove any operator for any address.
- The contract function MUST reject if any of the queries fail:

  - A query MUST fail if the token ID is unknown with error: :ref:`INVALID_TOKEN_ID<CIS-1-rejection-errors>`.

.. _CIS-1-functions-tokenMetadata:

``tokenMetadata``
^^^^^^^^^^^^^^^^^

Query the current token metadata URLs for a list token IDs, the result is then sent to a provided contract address.

Parameter
~~~~~~~~~

The parameter consists of a name of the contract address and receive function to callback with the result and a list of token ID.

It is serialized as: a :ref:`CIS-1-ContractAddress` (``resultContract``) then a :ref:`CIS-1-ReceiveHookName` (``resultHook``) followed by 1 byte for the number of queries (``n``) and then this number of :ref:`CIS-1-TokenID` (``ids``)::

  TokenMetadataParameter ::= (resultContract: ContractAddress) (resultHook: ReceiveHookName) (n: Byte²) (ids: TokenIDⁿ)

.. note::

  Be aware of the size limit on contract function parameters which currently is 1024 bytes, which puts a limit on the number of queries depending on the byte size of the Token ID and the name of the receive function.


Result parameter
~~~~~~~~~~~~~~~~

The parameter for the callback receive function is a list of token ID and metadata URL pairs.

It is serialized as: 2 bytes for the number of query-amount pairs (``n``) and then this number of pairs (``results``).
A pair is serialized as a :ref:`CIS-1-TokenID` (``id``) and then a :ref:`CIS-1-MetadataUrl` (``metadata``)::

  TokenMetadataResult ::= (id: TokenID) (metadata: MetadataUrl)

  TokenMetadataResultParameter ::= (n: Byte²) (results: TokenMetadataResultⁿ)

Requirements
~~~~~~~~~~~~

- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The contract function MUST NOT add or remove any operator for any address.
- The contract function MUST reject if any of the queries fail:

  - A query MUST fail if the token ID is unknown with error: :ref:`INVALID_TOKEN_ID<CIS-1-rejection-errors>`.

Logged events
-------------

The idea of the logged events for this specification is for off-chain applications to be able to track balances and operators without knowledge of the contract-specific implementation details.
For this reason it is important to log events in any functionality of the token contract which modifies balances or operators.

- It MUST be possible to derive the balance of an address for a token type from the logged :ref:`CIS-1-event-transfer`, :ref:`CIS-1-event-mint` and :ref:`CIS-1-event-burn` events.
- It MUST be safe to assume that with no events logged, every address have zero tokens and no operators enabled for any address.

The events defined by this specification are serialized using one byte to the discriminate the different events.
Any custom event SHOULD NOT have a first byte colliding with any of the events defined by this specification.

.. _CIS-1-event-transfer:

``TransferEvent``
^^^^^^^^^^^^^^^^^

A ``TransferEvent`` event MUST be logged for every amount of a token type changing ownership from one address to another.

The ``TransferEvent`` event is serialized as: first a byte with the value of 255, followed by the token ID :ref:`CIS-1-TokenID` (``id``), an amount of tokens :ref:`CIS-1-TokenAmount` (``amount``), from address :ref:`CIS-1-Address` (``from``) and to address :ref:`CIS-1-Address` (``to``)::

  TransferEvent ::= (255: Byte) (id: TokenID) (amount: TokenAmount) (from: Address) (to: Address)

.. _CIS-1-event-mint:

``MintEvent``
^^^^^^^^^^^^^

A ``MintEvent`` event MUST be logged every time a new token is minted. This also applies when introducing new token types and the initial token types and amounts in a contract.
Minting a token with a zero amount can be used to indicating the existence of a token type without minting any amount of tokens.

The ``MintEvent`` event is serialized as: first a byte with the value of 254, followed by the token ID :ref:`CIS-1-TokenID` (``id``), an amount of tokens being minted :ref:`CIS-1-TokenAmount` (``amount``) and the owner address of the tokens :ref:`CIS-1-Address` (``to``)::

  MintEvent ::= (254: Byte) (id: TokenID) (amount: TokenAmount) (to: Address)

.. note::

  Be aware of the limit on the number of logs per smart contract function call which currently is 64.
  A token smart contract function which needs to mint a large number of token types with token metadata might hit this limit.

.. _CIS-1-event-burn:

``BurnEvent``
^^^^^^^^^^^^^

A ``BurnEvent`` event MUST be logged every time an amount of a token type is burned.

Summing all of the minted amounts from ``MintEvent`` events and subtracting all of the burned amounts from ``BurnEvent`` events for a token type MUST sum up to the total supply for the token type.
The total supply of a token type MUST be in the inclusive range of [0, 2^64 - 1].

The ``BurnEvent`` event is serialized as: first a byte with the value of 253, followed by the token ID :ref:`CIS-1-TokenID` (``id``), an amount of tokens being burned :ref:`CIS-1-TokenAmount` (``amount``) and the owner address of the tokens :ref:`CIS-1-Address` (``from``)::

  BurnEvent ::= (253: Byte) (id: TokenID) (amount: TokenAmount) (from: Address)

.. _CIS-1-event-updateOperator:

``UpdateOperatorEvent``
^^^^^^^^^^^^^^^^^^^^^^^

The event to log when updating an operator of some address.

The ``UpdateOperatorEvent`` event is serialized as: first a byte with the value of 252, followed by a ``OperatorUpdate`` (``update``), then the owner address updating an operator :ref:`CIS-1-Address` (``owner``) and an operator address :ref:`CIS-1-Address` (``operator``) being added or removed::

  UpdateOperatorEvent ::= (252: Byte) (update: OperatorUpdate) (owner: Address) (operator: Address)

.. _CIS-1-event-tokenMetadata:

``TokenMetadataEvent``
^^^^^^^^^^^^^^^^^^^^^^

The event to log when setting the metadata url for a token type.
It consists of a token ID and an URL (:rfc:`3986`) for the location of the metadata for this token type with an optional SHA256 checksum of the content.
Logging the ``TokenMetadataEvent`` event again with the same token ID, is used to update the metadata location and only the most recently logged token metadata event for certain token id should be used to get the token metadata.

The ``TokenMetadataEvent`` event is serialized as: first a byte with the value of 251, followed by the token ID :ref:`CIS-1-TokenID` (``id``) and then a :ref:`CIS-1-MetadataUrl` (``metadata``)::

  TokenMetadataEvent ::= (251: Byte) (id: TokenID) (metadata: MetadataUrl)

.. note::

  Be aware of the limit on the number of logs per smart contract function call, which currently is 64, and also the byte size limit on each logged event, which currently is 512 bytes.
  This will limit the length of the metadata URL depending on the size of the token ID and whether a content hash is included.
  With the largest possible token ID and a content hash included; the URL can be up to 220 bytes.


.. _CIS-1-rejection-errors:

Rejection errors
----------------

A smart contract following this specification MUST reject the specified errors found in this specification with the following error codes:

.. list-table::
  :header-rows: 1

  * - Name
    - Error code
    - Description
  * - INVALID_TOKEN_ID
    - -42000001
    - A provided token ID it not part of this token contract.
  * - INSUFFICIENT_FUNDS
    - -42000002
    - An address balance contains insufficient amount of tokens to complete some transfer of a token.
  * - UNAUTHORIZED
    - -42000003
    - Sender is unauthorized to call this function. Note authorization is not mandated anywhere in this specification, but can still be introduced on top of the standard.

The smart contract implementing this specification MAY introduce custom error codes other than the ones specified in the table above.


Token metadata JSON
-------------------

The token metadata is stored off chain and MUST be a JSON (:rfc:`8259`) file.

All of the fields in the JSON file are optional, and this specification reserves a number of field names, shown in the table below.

.. list-table:: Token metadata JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``name`` (optional)
    - string
    - The name to display for the token type.
  * - ``symbol`` (optional)
    - string
    - Short text to display for the token type.
  * - ``decimals`` (optional)
    - number [``integer``]
    - The number of decimals, when displaying an amount of this token type in a user interface.
      If the decimal is set to ``d`` then a token amount ``a`` should be displayed as ``a * 10^(-d)``
  * - ``description`` (optional)
    - string
    - A description for this token type.
  * - ``thumbnail`` (optional)
    - URL JSON object
    - An image URL to a small image for displaying the asset.
  * - ``display`` (optional)
    - URL JSON object
    - An image URL to a large image for displaying the asset.
  * - ``artifact`` (optional)
    - URL JSON object
    - A URL to the token asset.
  * - ``assets`` (optional)
    - JSON array of Token metadata JSON objects
    - Collection of assets.
  * - ``attributes`` (optional)
    - JSON array of Attribute JSON objects
    - Assign a number of attributes to the token type.
      Attributes can be used to include extra information about the token type.
  * - ``localization`` (optional)
    - JSON object with locales as field names (:rfc:`5646`) and field values are URL JSON object to JSON files.
    - URL's to JSON files with localized token metadata.

Optionally a SHA256 hash of the JSON file can be logged with the TokenMetadata event for checking integrity.
Since the metadata json file could contain URLs, a SHA256 hash can optionally be associated with the URL.
To associate a hash with a URL the JSON value is an object:

.. list-table:: URL JSON Object
  :header-rows: 1

  * - Property
    - JSON value type [JSON-Schema]
    - Description
  * - ``url``
    - string (:rfc:`3986`) [``uri-reference``]
    - An URL.
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
    - Value of the attrbute.


Example token metadata: Fungible
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An example of token metadata for a CIS1 implementation wrapping the GTU could be:

.. code-block:: json

  {
    "name": "Wrapped GTU Token",
    "symbol": "wGTU",
    "decimals": 6,
    "description": "A CIS1 token wrapping the Global Transaction Unit",
    "thumbnail": { "url": "https://location.of/the/thumbnail.png" },
    "display": { "url": "https://location.of/the/display.png" },
    "artifact": { "url": "https://location.of/the/artifact.png" },
    "localization": {
      "da-DK": {
        "url": "https://location.of/the/danish/metadata.json",
        "hash": "624a1a7e51f7a87effbf8261426cb7d436cf597be327ebbf113e62cb7814a34b"
      }
    }
  }

The danish localization JSON file could be:

.. code-block:: json

  {
    "description": "CIS1 indpakket GTU"
  }

Example token metadata: Non-fungible
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An example of token metadata for a NFT could be:

.. code-block:: json

  {
    "name": "Bibi - The Ryan Cat",
    "description": "Ryan cats are lonely creatures travelling the galaxy in search of their ancestors and true inheritance",
    "thumbnail": { "url": "https://location.of/the/thumbnail.png" },
    "display": { "url": "https://location.of/the/display.png" },
    "attributes": [{
      "type": "date",
      "name": "Birthday",
      "value": "1629792199610"
    }, {
      "type": "string",
      "name": "Body",
      "value": "Strong"
    }, {
      "type": "string",
      "name": "Head",
      "value": "Round"
    }, {
      "type": "string",
      "name": "Tail",
      "value": "Short"
    }],
    "localization": {
      "da-DK": {
        "url": "https://location.of/the/danish/metadata.json",
        "hash": "588d7c14883231cfee522479cc66565fd9a50024603a7b8c99bd7869ca2f0ea3"
      }
    }
  }

The danish localization JSON file could be:

.. code-block:: json

  {
    "name": "Bibi - Ryan katten",
    "description": "Ryan katte er ensomme væsner, som rejser rundt i galaxen søgende efter deres forfædre og sande fortid"
  }

Smart contract limitations
==========================

A number of limitations are important to be aware of:

- Smart contract state size is limited to 16 KiB.
- Smart contract function parameters are limited to 1 KiB.
- Each logged event is limited to 0.5 KiB.
- The number of logged events is limited to 64 per contract function invocation.
- The total size of the smart contract module is limited to 64 KiB.

.. note::

  Smart contracts, where the contract state size limit is too low, can distribute the state across multiple smart contract instances.

Decisions and rationale
=======================

In this section we point out some of the differences from other popular token standards found on other blockchains, and provide reasons for deviating from them in CIS1.

Token ID bytes instead an integer
---------------------------------

Token standards such as ERC721 and ERC1155 both use a 256-bit unsigned integer (32 bytes) for the token ID, to support using something like a SHA256 hash for the token ID.
But in the case where the token ID have no significance other than a simple identifier, smaller sized token IDs can reduce energy costs.
This is why we chose to let the first byte indicate the size of the token ID, meaning a token ID can vary between 1 byte and 256 bytes, resulting in more than 10^614 possible token IDs.

Only batched transfers
----------------------

The specification only has a ``transfer`` smart contract function which takes list of transfer and no function for a single transfer.
This will result in lower energy cost compared to multiple contract calls and only introduces a small overhead for single transfers.
The reason for not also including a single transfer function is to have smaller smart contract modules, which in turn leads to saving cost on every function call.

No explicit authorization
-------------------------

The specification does not mandate any authorization scheme and one might expect a requirement for the owner and operators being authorized to transfer tokens.
This is very much intentional and the reasoning for this is to keep the specification focused on the interface for transferring token ownership with as few restrictions as possible.

Having a requirement that only owners and operators can transfer would prevent introducing any other authorization scheme on top of this specification.

Adding a requirement for owners and operators being authorized to transfer tokens would prevent introducing custom contract logic rejecting transfers, such as limiting the daily transfers, temporary token lockups or non-transferrable tokens.

Instead this specification includes a requirement to ensure transfers by operators are executed as if they are sent by the owner, meaning whenever a token owner is authorized, so is an operator of the owner.

Most contract implementing this specification should probably add some authorization and not have anyone being able to transfer any token, but this is not really relevant for the interface described in this specification.

No token level approval/allowance like in ERC20 and ERC721
----------------------------------------------------------

This standard only specifies address-level operators and is not scoped to a token type.
The main argument is simplicity and to save energy cost on common cases, but other reasons are:

- A token level operator requires the token smart contract to track more state, which increases the overall energy cost.
- For token smart contracts with a lot of token types, such as a smart contract with a large collection of NFTs, a token level operator could become very expensive.
- For fungible tokens; `approval/allowance introduces an attack vector <https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit>`_.

.. note::

  The specification does not prevent adding more fine-grained authorization, such as a token level operators.

Receive hook function
---------------------

The specification requires a token receive hook to be called on a smart contract receiving tokens, this will in some cases prevent mistakes such as sending tokens to smart contracts which do not define behavior for receiving tokens.
These token could then be lost forever.

The reason for this not being optional is to allow other smart contracts which integrate with a token smart contract to rely on this for functionality.

.. warning::

  The smart contract receive hook function could be called by any smart contract and it is up to the integrating contract whether to trust the token contract.

Receive hook function callback argument
---------------------------------------

The name of the receive hook function called on a smart contract receiving tokens is supplied as part of the parameter.
This allows for a smart contract integrating with a token smart contract to have multiple hooks and leave it to the caller to know which hook they want to trigger.

Another technical reason is that the name of the smart contract is part of the smart contract receive function name, which means the specification would include a requirement of the smart contract name for others to integrate reliably.

No sender hook function
-----------------------

The FA2 token standard found on Tezos, allows for a hook function to be called on a smart contract sending tokens, such that the contract could reject the transfer on some criteria.
This seems to only make sense, if some operator is transferring tokens from a contract, in which case the sender smart contract might as well contain the logic to transfer the tokens and trigger this directly.

Explicit events for mint and burn
---------------------------------

ERC20, ERC721 and ERC1155 use a transfer event from or to the zero address to indicate mint and burn respectively, but since there are no such thing as the zero address on the Concordium blockchain these events are separate.
Making it more explicit instead of special case transfer events.

No error code for receive hook rejecting
----------------------------------------

The specification could include an error code for the receive hook function to return if rejecting the token transferred (as seen in the `FA2 standard <https://gitlab.com/tezos/tzip/-/blob/master/proposals/tzip-12/tzip-12.md#error-handling>`_ on Tezos).
But we chose to leave this error code up to the receiving smart contract, which allows for more informative error codes.

Adding SHA256 checksum for token metadata event
-----------------------------------------------

A token can optionally include a SHA256 checksum when logging the token metadata event, this is to ensure the integrity of the token metadata.
This checksum can be updated by logging a new event.
