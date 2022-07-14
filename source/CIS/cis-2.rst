.. _CIS-2:

==================================
CIS-2: Concordium Token Standard 2
==================================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 16, 2021
   * - Draft version
     - 3
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-2``
   * - Requires
     - :ref:`CIS-0<CIS-0>`
   * - Deprecates
     - :ref:`CIS-1<CIS-1>`

.. warning::

   This standard is still a draft and significant changes might still be made.

Abstract
========

A standard interface for both fungible and non-fungible tokens implemented in a smart contract.
The interface provides functions for transferring token ownership, allowing other addresses to transfer tokens and for querying token balances, operators and token metadata.
It allows for off-chain applications to track token balances and the location of token metadata using logged events.

This standard is a modification of :ref:`CIS-1<CIS-1>` that uses features introduced in smart contract version 1.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

General types and serialization
-------------------------------

.. _CIS-2-TokenID:

``TokenID``
^^^^^^^^^^^

Token Identifier, which combined with the address of the smart contract instance implementing CIS2, forms the globally unique identifier of a token type.

- A token ID for a token type SHALL NOT change after a token type has been minted.
- A token ID for a token type SHALL NOT be reused for another token type within the same smart contract.

A token ID is serialized as 1 byte for the size (``n``) of the identifier, followed by this number of bytes for the token id (``id``)::

  TokenID ::= (n: Byte) (id: Byteⁿ)

.. note::

  Token IDs can be as small as a single byte (by setting the first byte to the value 0) or as big as 256 bytes leaving more than 10^614 possible token IDs.
  The token ID could be an encoding of a small text string or some checksum hash, but to save energy it is still recommended to use small token IDs if possible.

.. _CIS-2-TokenAmount:

``TokenAmount``
^^^^^^^^^^^^^^^

An amount of a token type is an unsigned integer up to 2^256 - 1.
It is serialized using the LEB128_ variable-length unsigned integer encoding, with the additional constraint of the total number of bytes of the encoding MUST not exceed 37 bytes::

  TokenAmount ::= (x: Byte)                   =>  x                     if x < 2^7
                | (x: Byte) (m: TokenAmount)  =>  (x - 2^7) + 2^7 * m   if x >= 2^7

.. _LEB128: https://en.wikipedia.org/wiki/LEB128

.. _CIS-2-ReceiveHookName:

``ReceiveHookName``
^^^^^^^^^^^^^^^^^^^

A smart contract entrypoint name.
An entrypoint name MUST consist only of ASCII alphanumeric or punctuation characters.

It is serialized as: the function name byte length (``n``) is represented by the first 2 bytes, followed by this many bytes for the function name (``name``).
The entrypoint name MUST be 100 bytes or less::

  ReceiveHookName ::= (n: Byte²) (name: Byteⁿ)

.. note::

  This type is passed in a parameter for smart contract function calls. Be aware of the parameter size limit of 1024 bytes.

.. _CIS-2-AccountAddress:

``AccountAddress``
^^^^^^^^^^^^^^^^^^^

An address of an account.

It is serialized as 32 bytes::

  AccountAddress ::= (address: Byte³²)

.. _CIS-2-ContractAddress:

``ContractAddress``
^^^^^^^^^^^^^^^^^^^

An address of a contract instance.
It consists of an index and a subindex both unsigned 64 bit integers.

It is serialized as: first 8 bytes for the index (``index``) followed by 8 bytes for the subindex (``subindex``) both little endian::

  ContractAddress ::= (index: Byte⁸) (subindex: Byte⁸)

.. _CIS-2-Address:

``Address``
^^^^^^^^^^^

Is either an account address or a contract address.

It is serialized as: First byte indicates whether it is an account address or a contract address.
In case the first byte is 0 then an ``AccountAddress`` (``address``) follows.
In case the first byte is 1 then a ``ContractAddress`` (``address``) follows::

  Address ::= (0: Byte) (address: AccountAddress)
            | (1: Byte) (address: ContractAddress)


.. _CIS-2-Receiver:

``Receiver``
^^^^^^^^^^^^

The receiving address of a transfer, which is either an account address or a contract address.
In the case of a contract address: a name of the hook receive function to invoke is also needed.

It is serialized as: First byte indicates whether it is an account address or a contract address.
In case the first byte is 0 then an ``AccountAddress`` (``address``) follows.
In case the first byte is 1 then a ``ContractAddress`` (``address``) and bytes for :ref:`CIS-2-ReceiveHookName` (``hook``) follows::

    Receiver ::= (0: Byte) (address: AccountAddress)
               | (1: Byte) (address: ContractAddress) (hook: ReceiveHookName)

.. _CIS-2-AdditionalData:

``AdditionalData``
^^^^^^^^^^^^^^^^^^^

Additional bytes to include in a transfer, which can be used to add additional parameters for the transfer function call.

It is serialized as: the first 2 bytes encode the length (``n``) of the data, followed by this many bytes for the data (``data``)::

  AdditionalData ::= (n: Byte²) (data: Byteⁿ)

.. note::

  This type is passed as part of a parameter for smart contract function calls.
  Be aware of the parameter size limit of 1024 bytes.

.. _CIS-2-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL and optional checksum for metadata stored outside of this contract.

It is serialized as: 2 bytes for the length of the metadata url (``n``) and then this many bytes for the url to the metadata (``url``) followed by an optional checksum.
The checksum is serialized by 1 byte to indicate whether a hash of the metadata is included, if its value is 0, then no content hash, if the value is 1 then 32 bytes for a SHA256 hash (``hash``) follows::

  MetadataChecksum ::= (0: Byte)
                     | (1: Byte) (hash: Byte³²)

  MetadataUrl ::= (n: Byte²) (url: Byteⁿ) (checksum: MetadataChecksum)

.. _CIS-2-functions:

Contract functions
------------------

A smart contract implementing CIS2 MUST export the following functions :ref:`CIS-2-functions-transfer`, :ref:`CIS-2-functions-updateOperator`, :ref:`CIS-2-functions-balanceOf`, :ref:`CIS-2-functions-operatorOf` and :ref:`CIS-2-functions-tokenMetadata` according to the following description:

.. _CIS-2-functions-transfer:

``transfer``
^^^^^^^^^^^^

Executes a list of token transfers.
A transfer is a token ID, an amount of tokens to be transferred, and the ``from`` address and ``to`` address.

When transferring tokens to a contract address, additional information for a receive function hook to trigger is required.

Parameter
~~~~~~~~~

The parameter is a list of transfers.

It is serialized as: 2 bytes representing the number of transfers (``n``) followed by the bytes for this number of transfers (``transfers``).
Each transfer is serialized as: a :ref:`CIS-2-TokenID` (``id``), a :ref:`CIS-2-TokenAmount` (``amount``), the token owner address :ref:`CIS-2-Address` (``from``), the receiving address :ref:`CIS-2-Receiver` (``to``) and some additional data (``data``)::

  Transfer ::= (id: TokenID) (amount: TokenAmount) (from: Address) (to: Receiver) (data: AdditionalData)

  TransferParameter ::= (n: Byte²) (transfers: Transferⁿ)

.. note::

  Be aware of the smart contract parameter size limit of 1024 bytes.
  Since the byte size of a single transfer can vary in size, this will limit the number of transfers that can be included in the same function call.
  Currently, with the smallest possible transfers, the parameter can contain 21 transfers and with the biggest possible transfer, it will take the whole parameter.

.. _CIS-2-functions-transfer-receive-hook-parameter:

Receive hook parameter
~~~~~~~~~~~~~~~~~~~~~~

The parameter for the receive function hook contains information about the transfer and some additional data bytes.

It is serialized as: a :ref:`CIS-2-TokenID` (``id``), a :ref:`CIS-2-TokenAmount` (``amount``), the token owner address :ref:`CIS-2-Address` (``from``) and :ref:`CIS-2-AdditionalData` (``data``)::

  ReceiveHookParameter ::= (id: TokenID) (amount: TokenAmount) (from: Address) (data: AdditionalData)

Requirements
~~~~~~~~~~~~

- The list of transfers MUST be executed in order.
- The contract function MUST reject if any of the transfers fail to be executed.
- A transfer MUST fail if:

  - The token balance of the ``from`` address is insufficient to do the transfer.
  - The TokenID is not known by the contract.

- A transfer MUST non-strictly decrease the balance of the ``from`` address and non-strictly increase the balance of the ``to`` address or fail.
- A transfer with the same address as ``from`` and ``to`` MUST be executed as a normal transfer.
- A transfer of a token amount zero MUST be executed as a normal transfer.
- A transfer of some amount of a token type MUST only transfer the exact amount of the given token type between balances.
- A transfer of any amount of a token type to a contract address MUST call receive hook function on the receiving smart contract with a :ref:`receive hook parameter<CIS-2-functions-transfer-receive-hook-parameter>`.
- Let ``operator`` be an operator of the address ``owner``. A transfer of any amount of a token type from an address ``owner`` sent by an address ``operator`` MUST be executed as if the transfer was sent by ``owner``.
- The contract function MUST reject if a receive hook function called on the contract receiving tokens rejects.
- The balance of an address not owning any amount of a token type SHOULD be treated as having a balance of zero.

.. warning::

  Be aware of transferring tokens to a non-existing account address.
  This specification by itself does not include a mechanism to recover these tokens.
  Checking the existence of an account address would ideally be done off-chain before the message is even sent to the token smart contract.

.. _CIS-2-functions-updateOperator:

``updateOperator``
^^^^^^^^^^^^^^^^^^

Add or remove a number of addresses as operators of the address sending this message.

Parameter
~~~~~~~~~

The parameter contains a list of operator updates. An operator update includes information on whether to add or remove an operator and the address to add/remove as operator.
It does not contain the address which is adding/removing the operator as this will be the sender of the message invoking this function.

The parameter is serialized as: first 2 bytes (``n``) for the number of updates followed by this number of operator updates (``updates``).
An operator update is serialized as: 1 byte (``update``) indicating whether to remove or add an operator, where if the byte value is 0 the sender is removing an operator, if the byte value is 1 the sender is adding an operator.
The update is followed by the operator address (``operator``) :ref:`CIS-2-Address` to add or remove as operator for the sender::

  OperatorUpdate ::= (0: Byte) // Remove operator
                   | (1: Byte) // Add operator

  UpdateOperator ::= (update: OperatorUpdate) (operator: Address)

  UpdateOperatorParameter ::= (n: Byte²) (updates: UpdateOperatorⁿ)

Requirements
~~~~~~~~~~~~

- The list of updates MUST be executed in order.
- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The balance of an address not owning any amount of a token type SHOULD be treated as having a balance of zero.
- The contract function MUST reject if any of the updates fails to be executed.

.. _CIS-2-functions-balanceOf:

``balanceOf``
^^^^^^^^^^^^^

Query balances of a list of addresses and token IDs.

Parameter
~~~~~~~~~

The parameter consists of a list of token ID and address pairs.

It is serialized as: 2 bytes for the number of queries (``n``) and then this number of queries (``queries``).
A query is serialized as :ref:`CIS-2-TokenID` (``id``) followed by :ref:`CIS-2-Address` (``address``)::

  BalanceOfQuery ::= (id: TokenID) (address: Address)

  BalanceOfParameter ::= (n: Byte²) (queries: BalanceOfQueryⁿ)

.. note::

  Be aware of the size limit on contract function parameters which currently is 1024 bytes, which puts a limit on the number of queries depending on the byte size of the Token ID.

Response
~~~~~~~~

The function output response is a list of token amounts.

It is serialized as: 2 bytes for the number of token amounts (``n``) and then this number of :ref:`CIS-2-TokenAmount` (``results``)::

  BalanceOfResponse ::= (n: Byte²) (results: TokenAmountⁿ)

Requirements
~~~~~~~~~~~~

- The balance of an address not owning any amount of a token type SHOULD be treated as having a balance of zero.
- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The contract function MUST NOT add or remove any operator for any address.
- The contract function MUST reject if any of the queries fail:

  - A query MUST fail if the token ID is unknown.

.. _CIS-2-functions-operatorOf:

``operatorOf``
^^^^^^^^^^^^^^

Query operators with a list of pairs, an owner address and a potential operator address, to check whether the potential operator address is an operator for the owner address.

Parameter
~~~~~~~~~

The parameter consists of a list of address pairs.

It is serialized as: 2 bytes for the number of queries (``n``) and then this number of queries (``queries``).
A query is serialized as :ref:`CIS-2-Address` (``owner``) followed by :ref:`CIS-2-Address` (``address``)::

  OperatorOfQuery ::= (owner: Address) (address: Address)

  OperatorOfParameter ::= (n: Byte²) (queries: OperatorOfQueryⁿ)

.. note::

  Be aware of the size limit on contract function parameters which currently is 1024 bytes, which puts a limit on the number of queries.

Response
~~~~~~~~

The function output is a list of booleans, where a value is ``True`` if and only if the ``address`` is an operator of the ``owner`` address from the corresponding query.

It is serialized as: 2 bytes for the number of results (``n``) and then this number of results (``results``).
A boolean is serialized as a byte with value 0 for false and 1 for true (``isOperator``)::

  Bool ::= (0: Byte) // False
         | (1: Byte) // True

  OperatorOfQueryResult ::= (isOperator: Bool)

  OperatorOfResultParameter ::= (n: Byte²) (results: OperatorOfQueryResultⁿ)

Requirements
~~~~~~~~~~~~

- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The contract function MUST NOT add or remove any operator for any address.
- The contract function MUST reject if any of the queries fail.

.. _CIS-2-functions-tokenMetadata:

``tokenMetadata``
^^^^^^^^^^^^^^^^^

Query the current token metadata URLs for a list of token IDs.

Parameter
~~~~~~~~~

The parameter consists of a list of token IDs.

It is serialized as: 2 bytes for the number of queries (``n``) and then this number of :ref:`CIS-2-TokenID` (``ids``)::

  TokenMetadataParameter ::= (n: Byte²) (ids: TokenIDⁿ)

.. note::

  Be aware of the size limit on contract function parameters which is currently 1024 bytes, which puts a limit on the number of queries depending on the byte size of the Token ID.


Response
~~~~~~~~

The function output is a list of token metadata URLs.

It is serialized as: 2 bytes for the number of queries (``n``) and then this number of :ref:`CIS-2-MetadataUrl` (``results``)::

  TokenMetadataResultParameter ::= (n: Byte²) (results: MetadataUrlⁿ)

Requirements
~~~~~~~~~~~~

- The number of results in the response MUST correspond to the number of the queries in the parameter.
- The order of results in the response MUST correspond to the order of queries in the parameter.
- The contract function MUST NOT increase or decrease the balance of any address for any token type.
- The contract function MUST NOT add or remove any operator for any address.
- The contract function MUST reject if any of the queries fail:

  - A query MUST fail if the token ID is unknown.

Logged events
-------------

The idea of the logged events for this specification is for off-chain applications to be able to track balances and operators without knowledge of the contract-specific implementation details.
For this reason, it is important for the token contract to log the appropriate event, any time modifications of balances or operators are made.

- It MUST be possible to derive the balance of an address for a token type from the logged :ref:`CIS-2-event-transfer`, :ref:`CIS-2-event-mint` and :ref:`CIS-2-event-burn` events.
- It MUST be safe to assume that with no events logged, every address has zero tokens and no operators enabled.

The events defined by this specification are serialized using one byte to the discriminate the different events.
A custom event SHOULD NOT have a first byte colliding with any of the events defined by this specification.

.. _CIS-2-event-transfer:

``TransferEvent``
^^^^^^^^^^^^^^^^^

A ``TransferEvent`` event MUST be logged for every amount of a token type changing ownership from one address to another.

The ``TransferEvent`` event is serialized as: first a byte with the value of 255, followed by the token ID :ref:`CIS-2-TokenID` (``id``), an amount of tokens :ref:`CIS-2-TokenAmount` (``amount``), from address :ref:`CIS-2-Address` (``from``) and to address :ref:`CIS-2-Address` (``to``)::

  TransferEvent ::= (255: Byte) (id: TokenID) (amount: TokenAmount) (from: Address) (to: Address)

.. _CIS-2-event-mint:

``MintEvent``
^^^^^^^^^^^^^

A ``MintEvent`` event MUST be logged every time a new token is minted. This also applies when introducing new token types and the initial token types and amounts in a contract.
Minting a token with a zero amount can be used to indicate the existence of a token type without minting any amount of tokens.

The ``MintEvent`` event is serialized as: first a byte with the value of 254, followed by the token ID :ref:`CIS-2-TokenID` (``id``), an amount of tokens being minted :ref:`CIS-2-TokenAmount` (``amount``) and the owner address of the tokens :ref:`CIS-2-Address` (``to``)::

  MintEvent ::= (254: Byte) (id: TokenID) (amount: TokenAmount) (to: Address)

.. note::

  Be aware of the :ref:`limit on the number of logs<CIS-2-smart-contract-limitations>`.
  A token smart contract function which needs to mint a large number of token types with token metadata might hit this limit.

.. _CIS-2-event-burn:

``BurnEvent``
^^^^^^^^^^^^^

A ``BurnEvent`` event MUST be logged every time an amount of a token type is burned.

Summing all of the minted amounts from ``MintEvent`` events and subtracting all of the burned amounts from ``BurnEvent`` events for a token type MUST sum up to the total supply for the token type.
The total supply of a token type MUST be in the inclusive range of [0, 2^256 - 1].

The ``BurnEvent`` event is serialized as: first a byte with the value of 253, followed by the token ID :ref:`CIS-2-TokenID` (``id``), an amount of tokens being burned :ref:`CIS-2-TokenAmount` (``amount``), and the owner address of the tokens :ref:`CIS-2-Address` (``from``)::

  BurnEvent ::= (253: Byte) (id: TokenID) (amount: TokenAmount) (from: Address)

.. _CIS-2-event-updateOperator:

``UpdateOperatorEvent``
^^^^^^^^^^^^^^^^^^^^^^^

The event to log when updating an operator of some address.

The ``UpdateOperatorEvent`` event is serialized as: first a byte with the value of 252, followed by a ``OperatorUpdate`` (``update``), then the owner address updating an operator :ref:`CIS-2-Address` (``owner``), and an operator address :ref:`CIS-2-Address` (``operator``) being added or removed::

  UpdateOperatorEvent ::= (252: Byte) (update: OperatorUpdate) (owner: Address) (operator: Address)

.. _CIS-2-event-tokenMetadata:

``TokenMetadataEvent``
^^^^^^^^^^^^^^^^^^^^^^

The event to log when setting the metadata url for a token type.

It consists of a token ID and a URL (:rfc:`3986`) for the location of the metadata for this token type with an optional SHA256 checksum of the content.
Logging the ``TokenMetadataEvent`` event again with the same token ID, is used to update the metadata location and only the most recently logged token metadata event for a certain token id should be used to get the token metadata.

The ``TokenMetadataEvent`` event is serialized as: first a byte with the value of 251, followed by the token ID :ref:`CIS-2-TokenID` (``id``), and then a :ref:`CIS-2-MetadataUrl` (``metadata``)::

  TokenMetadataEvent ::= (251: Byte) (id: TokenID) (metadata: MetadataUrl)

.. note::

  Be aware of the :ref:`limit on the number of logs<CIS-2-smart-contract-limitations>`, and also the byte size limit on each logged event.
  This will limit the length of the metadata URL depending on the size of the token ID and whether a content hash is included.

.. _CIS-2-rejection-errors:

Rejection errors
----------------

A smart contract following this specification MAY reject using the following error codes:

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

Rejecting using an error code from the table above MUST only occur in a situation as described in the corresponding error description.

The smart contract implementing this specification MAY introduce custom error codes other than the ones specified in the table above.


Token metadata JSON
-------------------

The token metadata is stored off-chain and MUST be a JSON (:rfc:`8259`) file.

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
    - JSON object with locales as field names (:rfc:`5646`) and field values are URL JSON objects linking to JSON files.
    - URLs to JSON files with localized token metadata.

Optionally a SHA256 hash of the JSON file can be logged with the TokenMetadata event for checking integrity.
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

An example of token metadata for a CIS2 implementation wrapping the CCD could be:

.. code-block:: json

  {
    "name": "Wrapped CCD Token",
    "symbol": "wCCD",
    "decimals": 6,
    "description": "A CIS2 token wrapping the Concordium native token (CCD)",
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
    "description": "CIS2 indpakket CCD"
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
    "description": "Ryan katte er ensomme væsner, som rejser rundt i galaxen søgende efter deres forfædre og sande arv"
  }

.. _CIS-2-smart-contract-limitations:

Smart contract limitations
==========================

A number of limitations are important to be aware of:

- Smart contract function parameters are limited to 1 KiB.
- Each logged event is limited to 0.5 KiB.
- The number of logged events is limited to 64 without invoking another contract.
  However invoking another contract function during a contract call resets the count and allows another 64 events to be logged.
- The total size of the smart contract module is limited to 512 KiB.

Decisions and rationale
=======================

In this section we point out some of the differences from other popular token standards found on other blockchains, and provide reasons for deviating from them in CIS2.

Token ID bytes instead of integers
----------------------------------

Token standards such as ERC721 and ERC1155 both use a 256-bit unsigned integer (32 bytes) for the token ID, to support using something like a SHA256 hash for the token ID.
But in the case where the token ID have no significance other than a simple identifier, smaller sized token IDs can reduce energy costs.
This is why we chose to let the first byte indicate the size of the token ID, meaning a token ID can vary between 1 byte and 256 bytes. The latter allows more than 10^614 possible token IDs.

Variable-length encoding of token amount
----------------------------------------

Similar to ERC721 and ERC1155, the token amount is limited to a 256-bit unsigned integer.
However, using 32 bytes for encoding the token amount is wasteful for token contracts with a total supply fitting into fewer bytes. This is especially the case for non-fungible tokens.
Additionally 256-bit integers are not natively supported by WebAssembly meaning arithmetics are expensive compared to a 32-bit or 64-bit integer.
This specification uses a variable-length encoding of the token amount, allowing a token smart contract to restrict the token amount and internally represent the token amount using fewer bytes.

Only batched transfers
----------------------

The specification only has a :ref:`CIS-2-functions-transfer` smart contract function that takes a list of transfers and no function for a single transfer.
This will result in lower energy cost compared to multiple contract calls and only introduces a small overhead for single transfers.
The reason for not also including a single transfer function is to have smaller smart contract modules, which in turn leads to saving cost on every function call.

.. note::

   Notice that :ref:`CIS-2-functions-transfer` is more general than both ``safeTransferFrom`` and ``safeBatchTransferFrom`` found in ERC721 and ERC1155 as these standards only take a single sender and receiver for a batch of transfers.

No explicit authorization
-------------------------

The specification does not mandate any authorization scheme and one might expect a requirement for the owner and operators being authorized to transfer tokens.
This is intentional and the reason for this is to keep the specification focused on the interface for transferring token ownership with as few restrictions as possible.

Having a requirement that only owners and operators can transfer would prevent introducing any other authorization scheme on top of this specification.

Adding a requirement for owners and operators being authorized to transfer tokens would prevent introducing custom contract logic rejecting transfers, such as limiting the daily transfers, temporary token lockups or non-transferrable tokens.

Instead, this specification includes a requirement to ensure transfers by operators are executed as if they are sent by the owner, meaning whenever a token owner is authorized, so is an operator of the owner.

Most smart contracts implementing this specification will add an authorization scheme to restrict when tokens can be transferred. But, as stated, requirements for such a scheme are beyond the scope of this standard.

No token-level approval/allowance like in ERC20 and ERC721
----------------------------------------------------------

This standard only specifies address-level operators and not token-level operators.
The main argument is simplicity and to save energy cost on common cases, but other reasons are:

- Token-level operators require the token smart contract to track more state, which increases the overall energy cost.
- For token smart contracts with a lot of token types, such as a smart contract with a large collection of NFTs, token-level operators could become very expensive.
- For fungible tokens; `approval/allowance introduces an attack vector <https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit>`_.

.. note::

  The specification does not prevent adding more fine-grained authorization, such as token-level operators.

Receive hook function
---------------------

The specification requires a token receive hook to be invoked on a smart contract receiving tokens, this will in some cases prevent mistakes such as sending tokens to smart contracts which do not define behavior for receiving tokens.
These token could then be lost forever.

The reason for this not being optional is to allow other smart contracts, which integrate with a token smart contract, to rely on this for functionality.

.. warning::

  The smart contract receive hook function can be called by any account or smart contract. It is up to the integrating contract whether it should trust the caller or not.

Receive hook function callback argument
---------------------------------------

The name of the receive hook function called on a smart contract receiving tokens is supplied as part of the parameter.
This allows for a smart contract integrating with a token smart contract to have multiple hooks and leave it to the caller to know which hook they want to trigger.

No sender hook function
-----------------------

The FA2 token standard found on Tezos allows for a hook function to be called on a smart contract sending tokens, such that the contract can reject the transfer on some criteria.
This seems to only make sense if some operator is transferring tokens from a contract, in which case the sender smart contract might as well contain the logic to transfer the tokens and trigger this directly.

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

Differences from CIS1
---------------------

The query functions :ref:`CIS-2-functions-balanceOf`, :ref:`CIS-2-functions-operatorOf`, and :ref:`CIS-2-functions-tokenMetadata` differ from CIS1.
The query functions in CIS1 use a callback pattern to output the result of a query. However, starting from Concordium smart contract version 1, a smart contract receive function can return values back to the invoker.
CIS2 uses this output instead of a callback pattern to return the query result.
Using output instead of callbacks requires less energy and will reduce the contract code needed for querying.

In CIS1 the callback result includes the corresponding query to ease the use of the callback pattern. The query information is not needed in the output result of CIS2 query functions.
Instead, the results are required to be the same length and order as the queries.

In CIS2 smart contract functions are not required to fail with a specific error code as in CIS1. This is to allow receive functions to fail early for reason specific to the implementation such as authorization or serialization.

Prior to smart contract version 1 invoking another smart contract required knowing the contract name as well as the contract address and endpoint.
Smart contract version 1 removes the need for the contract name, which is why :ref:`CIS-2-functions-transfer-receive-hook-parameter` does not included the token contract name as seen in CIS1.

In :ref:`CIS1 the token amount<CIS-1-TokenAmount>` is fixed to u64 which was deemed sufficient for most token smart contracts.
However, to improve the interoperability with decentralized applications with support for other blockchains using 256-bit integers, the variable-length encoding was introduced making :ref:`CIS2 token amount<CIS-2-TokenAmount>` more flexible.
