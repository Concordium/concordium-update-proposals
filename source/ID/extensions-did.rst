=================================
Extensions for the Concordium DID
=================================

.. _concordium-did-verification-method:

Threshold Verification Method
===============================

The threshold verification method usen in Concordium DID Documents is based on a `ConditionalProof verification method <https://w3c-ccg.github.io/verifiable-conditions/>`_.
This is a new type of verification method under development.
``ConditionalProof`` features several extensions such as logical operations (``and``, ``or``), threshold and weighted threshold.
Note that the method is not yet a W3C standard and currently has a *draft* status.

The example below shows the ``2-out-of-3`` signature verifcation method.
It uses the ``ConditionalProof2022`` verifcation method.
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
