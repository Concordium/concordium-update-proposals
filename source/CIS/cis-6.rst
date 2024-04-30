.. _CIS-6:

===============================
CIS-6: Track-and-Trace Standard
===============================

.. list-table::
   :stub-columns: 1

   * - Created
     - Apr 25, 2024
   * - Final
     - Apr 26, 2024
   * - Supported versions
     - | Smart contract version 1 or newer
       | (Protocol version 4 or newer)
   * - Standard identifier
     - ``CIS-6``
   * - Requires
     - :ref:`CIS-0<CIS-0>`


Abstract
========

A standard interface that defines the logged events for tracking items in a smart contract.
The interface defines two events:

- *ItemCreatedEvent* logs the ``item_id``, ``metadata_url``, and the ``initial_status``;
- *ItemStatusChangedEvent* logs the ``item_id``, ``new_status``, and the ``additional_data``;

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.


General types and serialization
-------------------------------

.. _CIS-6-ItemId:

``ItemId``
^^^^^^^^^^

Each item MUST have a unique ``ItemId``.
An ``ItemId`` is a variable-length ASCII string up to 255 characters.

The identifier is serialized as: 1 byte for the length (``n``) followed by this many bytes for the ASCII encoding of the identifier::

  ItemId ::= (n: Byte) (id: Byteⁿ)

.. _CIS-6-MetadataUrl:

``MetadataUrl``
^^^^^^^^^^^^^^^

A URL and optional checksum for metadata stored outside of this contract.

It is serialized as: 2 bytes for the length (``n``) of the metadata URL in little-endian and then this many bytes for the URL to the metadata (``url``) followed by an optional checksum.
The checksum is serialized by 1 byte to indicate whether a hash of the metadata is included.
If its value is 0, then there is no hash; if the value is 1, then 32 bytes for a SHA256 hash (``hash``) follows::

  MetadataChecksum ::= (0: Byte)
                     | (1: Byte) (hash: Byte³²)

  MetadataUrl ::= (n: Byte²) (url: Byteⁿ) (checksum: MetadataChecksum)

.. _CIS-6-Status:

``Status``
^^^^^^^^^^

The status defined by this specification is serialized using one byte to discriminate the different statuses that an item can have.
The smart contract can have up to 255 different statuses defined for an item.

It is serialized as: a byte of value (``n``)::

  Status ::= (n: Byte)

.. _CIS-6-AdditionalData:

``AdditionalData``
^^^^^^^^^^^^^^^^^^

Additional bytes to include in the event ``ItemStatusChangedEvent``, which can be used to add additional use-case-specific data (e.g. temperature, longitude, latitude, ...) to the event.

It is serialized as: the first 2 bytes encode the length (``n``) of the data, followed by this many bytes for the data (``data``)::

  AdditionalData ::= (n: Byte²) (data: Byteⁿ)

Logged events
-------------

The events defined by this specification are serialized using one byte to discriminate the different events.
A custom event SHOULD NOT have a first byte colliding with any of the events defined by this specification.

.. _CIS-6-events-ItemCreatedEvent:

``ItemCreatedEvent``
^^^^^^^^^^^^^^^^^^^^

A ``ItemCreatedEvent`` event MUST be logged when a new item is created in the smart contract.

The ``ItemCreatedEvent`` event is serialized as: first a byte with the value of 237, followed by the :ref:`CIS-6-ItemId` (``id``), the :ref:`CIS-6-MetadataUrl` (``metadata``), and then the :ref:`CIS-6-Status` (``initial_status``)::

  ItemCreatedEvent ::= (237: Byte) (item_id: ItemId) (metadata: MetadataUrl) (initial_status: Status)

.. _CIS-6-events-ItemStatusChangedEvent:

``ItemStatusChangedEvent``
^^^^^^^^^^^^^^^^^^^^^^^^^^

A ``ItemStatusChangedEvent`` event MUST be logged when an item status is updated in the smart contract state.

The ``ItemStatusChangedEvent`` event is serialized as: first a byte with the value of 236, followed by the :ref:`CIS-6-ItemId` (``id``), the :ref:`CIS-6-Status` (``new_status``), and then the :ref:`CIS-6-AdditionalData` (``additional_data``)::

  ItemStatusChangedEvent ::= (236: Byte) (item_id: ItemId) (new_status: Status) (additional_data: AdditionalData)
