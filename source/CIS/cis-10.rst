.. _CIS-10:

==============================================
CIS-10: Asserted External Identifier Registry
==============================================

.. list-table::
   :stub-columns: 1

   * - Created
     - May 26, 2026
   * - Draft
     - May 26, 2026
   * - Supported versions
     - | Smart contract version 1 (concordium-std v10+)
   * - Standard identifier
     - ``CIS-10``


Abstract
========

A standard interface for a smart contract that records verifier-anchored
bindings between Concordium accounts and external identifiers that do *not*
have an associated external cryptographic key — typically platform handles
(Moltbook, Discord, X), HTTPS endpoint domains, or any other off-chain
identifier expressible as bytes.

CIS-10 is the verifier-anchored companion to :ref:`CIS-8`. Where CIS-8 records
bindings proven cryptographically by the holder of the external key, CIS-10
records bindings *attested* by a third-party verifier whose endorsement is
itself signed by their Concordium account key and anchored by an on-chain
transaction. :ref:`CIS-8004` agents MAY point to a CIS-10 entry via their
``external_reference`` field's ``Cis10(_)`` variant.


Introduction
============

CIS-10 records a single fact per entry: a Concordium account is bound to a
specific external identifier, with the binding vouched for by a third-party
verifier. Each external identifier is uniquely identified by the triple
``(namespace, identifier_type, identifier)``. The verifier signs the hash of
an on-chain *anchor transaction* whose contents contain the substantive
verification evidence; the CIS-10 contract verifies the signature against the
verifier's on-chain credentials and records the binding.

The contract maintains only the current state of each assertion. Full
history is recoverable from the emitted events.

Terminology and Conventions
---------------------------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in :rfc:`2119`.

Rust types are expressed using ``concordium-std`` conventions (``Serialize``,
``SchemaType``, ``Reject``). Schemas embedded in deployed modules are
serialised in the Concordium binary schema format.

Relationship to Other CIS Standards
-----------------------------------

- :ref:`CIS-0` — CIS-10 contracts MUST implement standard detection and
  advertise support for ``CIS-0`` and ``CIS-10``.
- :ref:`CIS-8004` — Agent Registry contracts MAY consume CIS-10 entries via
  cross-contract ``assertionFor`` calls when an agent declares a
  verifier-anchored external reference.

CIS-10 does not depend on any other standard.


Specification
=============

Common Types
------------

.. _CIS-10-ExternalIdentifier:

ExternalIdentifier
^^^^^^^^^^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType, Clone, PartialEq, Eq, PartialOrd, Ord, Debug)]
    pub struct ExternalIdentifier {
        /// Platform-style or URI-scheme namespace (e.g., "platform:moltbook",
        /// "platform:discord", "https").
        pub namespace:       String,
        /// How to interpret the identifier bytes (e.g., "handle", "username",
        /// "domain").
        pub identifier_type: String,
        /// Raw identifier bytes — typically UTF-8.
        pub identifier:      Vec<u8>,
    }

The ``namespace`` MUST be non-empty and MUST NOT exceed 128 bytes.

The ``identifier_type`` MUST be non-empty and MUST NOT exceed 64 bytes.

The ``identifier`` MUST be non-empty and MUST NOT exceed 512 bytes.

.. note::

   The field is named ``identifier_type``. An earlier (pre-2026-05-25) draft
   used ``id_type``; schema-driven serialisers reject payloads with the old
   field name. Tooling that targeted the earlier draft MUST be updated.

.. _CIS-10-MetadataEntry:

MetadataEntry
^^^^^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType, Clone, Debug)]
    pub struct MetadataEntry {
        pub key:   String,
        pub value: String,
    }

A registration MAY carry up to 32 ``MetadataEntry`` items. Each ``key`` MUST
be non-empty and MUST NOT exceed 64 bytes. Each ``value`` MUST NOT exceed
512 bytes.

.. _CIS-10-AssertionProof:

AssertionProof
^^^^^^^^^^^^^^

The verifier's authorisation for the binding, expressed as a signature over
the anchor transaction hash with a domain-separation prefix.

.. code-block:: rust

    #[derive(Serialize, SchemaType, Clone, Debug)]
    pub struct AssertionProof {
        /// Hash of the on-chain anchor transaction containing verification
        /// evidence. MUST reference an existing Concordium transaction.
        pub anchor_tx_hash:   HashSha2256,
        /// Concordium account that performed the verification and signed the
        /// anchor hash.
        pub verifier_account: AccountAddress,
        /// Signature by verifier_account's signing key over the signing
        /// bytes specified in §CIS-10-SigningBytes.
        pub signature:        Vec<u8>,
    }

.. _CIS-10-AssertionStatus:

AssertionStatus
^^^^^^^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType, Clone, Copy, PartialEq, Eq, Debug)]
    pub enum AssertionStatus {
        Active,
        Revoked,
    }

.. _CIS-10-Assertion:

Assertion
^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType, Clone, Debug)]
    pub struct Assertion {
        pub bound_account:       AccountAddress,
        pub external_identifier: ExternalIdentifier,
        pub proof:               AssertionProof,
        pub metadata:            Vec<MetadataEntry>,
        pub status:              AssertionStatus,
        pub last_updated:        Timestamp,
    }


Contract State
--------------

A CIS-10 contract MUST maintain at least the following state:

.. code-block:: rust

    #[derive(Serial, DeserialWithState)]
    #[concordium(state_parameter = "S")]
    pub struct State<S: HasStateApi> {
        /// Current assertion keyed by ExternalIdentifier.
        pub assertions: StateMap<ExternalIdentifier, Assertion, S>,
        /// Single-account admin model (see Admin section).
        pub admin: Option<AccountAddress>,
    }

Only the current state is stored. Historical transitions MUST be recoverable
from emitted events.


.. _CIS-10-SigningBytes:

Signing Bytes
-------------

The verifier signs the anchor transaction hash, prefixed with a 16-byte
domain-separation tag. The contract reconstructs these bytes verbatim and
verifies the signature against them.

.. code-block:: rust

    fn signing_bytes(anchor_tx_hash: &HashSha2256) -> Vec<u8> {
        let mut bytes = Vec::with_capacity(16 + 32);
        bytes.extend_from_slice(b"CIS-10/v1/anchor");      // 16 bytes UTF-8
        bytes.extend_from_slice(anchor_tx_hash.as_ref());  // 32 bytes hash
        bytes
    }

Total payload: 48 bytes. The 16-byte prefix ``CIS-10/v1/anchor`` provides
domain separation; signatures produced for any other purpose cannot be
replayed against CIS-10.


Entrypoints
-----------

.. _CIS-10-init:

init
^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct InitParams {
        pub admin: AccountAddress,
    }

Initialises ``state.admin = Some(params.admin)`` and an empty
``assertions`` map.

.. _CIS-10-supports:

supports
^^^^^^^^

Standard :ref:`CIS-0` detection. MUST return ``Support`` for ``CIS-0`` and
``CIS-10``.

.. _CIS-10-register:

register
^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct RegisterParams {
        pub bound_account:       AccountAddress,
        pub external_identifier: ExternalIdentifier,
        pub proof:               AssertionProof,
        pub metadata:            Vec<MetadataEntry>,
    }

Authorisation
    None at the sender layer. Any account or contract MAY submit a
    ``register`` transaction. Authorisation comes entirely from the
    cryptographic verification of ``proof.signature`` against
    ``proof.verifier_account``'s on-chain credentials.

Validation
    The contract MUST:

    1. Validate ``external_identifier`` per :ref:`CIS-10-ExternalIdentifier`.
    2. Validate ``metadata`` per :ref:`CIS-10-MetadataEntry`.
    3. Reconstruct the :ref:`CIS-10-SigningBytes` and verify
       ``proof.signature`` against ``proof.verifier_account``'s account
       credentials via ``host.check_account_signature``. Reject with
       :ref:`CIS-10-InvalidProof` on failure, or with
       :ref:`CIS-10-VerifierAccountInvalid` if the account does not exist or
       has no on-chain credentials.

Effect
    Determine whether ``external_identifier`` already has an active
    assertion.

    - **No active assertion exists.** Insert a new ``Assertion`` with
      ``status = Active`` and ``last_updated = ctx.metadata().slot_time()``.
      Emit :ref:`CIS-10-AssertionRegistered`.
    - **Active assertion exists with the same ``bound_account``.** Replace
      the assertion in place (refreshing ``proof``, ``metadata``,
      ``last_updated``). Emit :ref:`CIS-10-AssertionRegistered`.
    - **Active assertion exists with a different ``bound_account``.** Mark
      the previous assertion as ``Revoked``, emit
      :ref:`CIS-10-AssertionRevoked` for the previous bound account, then
      insert the new active assertion and emit
      :ref:`CIS-10-AssertionRegistered` for the new bound account. The two
      events MUST be emitted in that order.

Replacement is unilateral by the new verified binding. The verifier
supplying the new assertion need not be the same verifier as the previous
assertion's verifier — the contract trusts any valid signature from any
account that has on-chain credentials.

.. _CIS-10-updateMetadata:

updateMetadata
^^^^^^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct UpdateMetadataParams {
        pub identifier: ExternalIdentifier,
        pub metadata:   Vec<MetadataEntry>,
    }

Authorisation
    Sender MUST be ``Address::Account(addr)`` with ``addr`` equal to the
    current active assertion's ``bound_account``. Otherwise reject with
    :ref:`CIS-10-Unauthorized`.

Validation
    Reject with :ref:`CIS-10-NotRegistered` if no active assertion exists.
    Validate ``metadata`` per :ref:`CIS-10-MetadataEntry`.

Effect
    Replace ``metadata`` and update ``last_updated``. Emit
    :ref:`CIS-10-UpdateMetadata`.

.. _CIS-10-revoke:

revoke
^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct RevokeParams {
        pub identifier: ExternalIdentifier,
    }

Authorisation
    Sender MUST be ``Address::Account(addr)`` with ``addr`` equal to the
    current active assertion's ``bound_account``. The verifier who created
    the assertion has **no** revocation authority.

Validation
    Reject with :ref:`CIS-10-NotRegistered` if no active assertion exists.

Effect
    Set ``status = Revoked`` and update ``last_updated``. Emit
    :ref:`CIS-10-AssertionRevoked`.

.. _CIS-10-assertionFor:

assertionFor
^^^^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct AssertionForParams {
        pub external_identifier: ExternalIdentifier,
    }

    // returns: Option<Assertion>

Read-only. Returns the current ``Assertion`` for the supplied identifier or
``None`` if no assertion exists. Consumers MUST inspect ``status`` —
``Revoked`` assertions are returned but MUST be ignored for trust purposes.


Events
------

.. list-table::
   :header-rows: 1

   * - Event
     - Tag
   * - :ref:`CIS-10-AssertionRegistered`
     - 260
   * - :ref:`CIS-10-AssertionRevoked`
     - 261
   * - :ref:`CIS-10-UpdateMetadata`
     - 262
   * - :ref:`CIS-10-Upgraded`
     - 263
   * - :ref:`CIS-10-AdminTransferred`
     - 264

The contract MUST implement custom ``Serial`` on its event enum, writing the
tag byte before the payload.

.. _CIS-10-AssertionRegistered:

AssertionRegistered (tag 260)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: rust

    pub struct AssertionRegisteredEvent {
        pub bound_account:       AccountAddress,
        pub external_identifier: ExternalIdentifier,
        pub verifier_account:    AccountAddress,
        pub anchor_tx_hash:      HashSha2256,
        pub metadata:            Vec<MetadataEntry>,
    }

Emitted when a new assertion becomes active or a previous ``bound_account``
is replaced.

.. _CIS-10-AssertionRevoked:

AssertionRevoked (tag 261)
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: rust

    pub struct AssertionRevokedEvent {
        pub bound_account:       AccountAddress,
        pub external_identifier: ExternalIdentifier,
    }

Emitted on a successful :ref:`CIS-10-revoke` call and on replacement when a
new ``bound_account`` displaces an existing one.

.. _CIS-10-UpdateMetadata:

UpdateMetadata (tag 262)
^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: rust

    pub struct UpdateMetadataEvent {
        pub external_identifier: ExternalIdentifier,
        pub metadata:            Vec<MetadataEntry>,
    }

Emitted on a successful :ref:`CIS-10-updateMetadata` call.


Errors
------

Reject codes are mapped explicitly via ``From<Cis10Error> for Reject`` to the
numeric values listed below; implementations MUST NOT rely on derived
sequential numbering.

.. list-table::
   :header-rows: 1

   * - Code
     - Name
     - Meaning
   * - .. _CIS-10-InvalidProof:

       -7300
     - ``InvalidProof``
     - ``proof.signature`` does not verify against ``proof.verifier_account``
       over the reconstructed :ref:`CIS-10-SigningBytes`.
   * - .. _CIS-10-MalformedExternalIdentifier:

       -7301
     - ``MalformedExternalIdentifier``
     - ``external_identifier`` fails the validation rules in
       :ref:`CIS-10-ExternalIdentifier`.
   * - .. _CIS-10-Unauthorized:

       -7302
     - ``Unauthorized``
     - Caller is not the active ``bound_account`` on
       :ref:`CIS-10-updateMetadata` or :ref:`CIS-10-revoke`, or caller is a
       contract on a state-mutating entrypoint.
   * - .. _CIS-10-NotRegistered:

       -7303
     - ``NotRegistered``
     - No active assertion exists for the supplied ``ExternalIdentifier``.
   * - .. _CIS-10-InvalidMetadata:

       -7304
     - ``InvalidMetadata``
     - ``metadata`` fails the validation rules in :ref:`CIS-10-MetadataEntry`.
   * - .. _CIS-10-VerifierAccountInvalid:

       -7305
     - ``VerifierAccountInvalid``
     - ``proof.verifier_account`` does not exist on-chain or has no current
       credentials.
   * - .. _CIS-10-NotAdmin:

       -7306
     - ``NotAdmin``
     - Sender is not the contract admin.
   * - .. _CIS-10-UpgradeFailed:

       -7307
     - ``UpgradeFailed``
     - ``host.upgrade`` returned an error, or the post-upgrade migration
       invocation failed.


.. _CIS-10-Admin:

Admin and Upgradeability
------------------------

The contract supports upgradeability via Concordium's native ``host.upgrade``
primitive under a single-account admin model. Semantics are identical to
:ref:`CIS-8-Admin`: ``upgrade``, ``transferAdmin``, ``getAdmin`` entrypoints
with reject codes :ref:`CIS-10-NotAdmin` and :ref:`CIS-10-UpgradeFailed`,
events ``Upgraded`` (tag 263) and ``AdminTransferred`` (tag 264).

State
^^^^^

``state.admin: Option<AccountAddress>`` — set from ``InitParams.admin`` at
deployment.

upgrade
^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct UpgradeParams {
        pub new_module: ModuleReference,
        pub migrate:    Option<(OwnedEntrypointName, OwnedParameter)>,
    }

Sender MUST equal ``state.admin``; otherwise reject with
:ref:`CIS-10-NotAdmin`. The contract MUST call ``host.upgrade(new_module)``
and, if ``migrate`` is supplied, invoke the named entrypoint on ``self`` in
the new module. Emit :ref:`CIS-10-Upgraded`.

transferAdmin
^^^^^^^^^^^^^

.. code-block:: rust

    #[derive(Serialize, SchemaType)]
    pub struct TransferAdminParams {
        /// None permanently locks the contract (admin renounced).
        pub new_admin: Option<AccountAddress>,
    }

Sender MUST equal ``state.admin``; otherwise reject with
:ref:`CIS-10-NotAdmin`. Emit :ref:`CIS-10-AdminTransferred`.

Renouncing the admin (``new_admin = None``) is irreversible.

.. _CIS-10-Upgraded:

Upgraded (event, tag 263)
^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: rust

    pub struct UpgradedEvent {
        pub new_module:  ModuleReference,
        pub upgraded_at: Timestamp,
    }

.. _CIS-10-AdminTransferred:

AdminTransferred (event, tag 264)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: rust

    pub struct AdminTransferredEvent {
        pub previous:   Option<AccountAddress>,
        pub new:        Option<AccountAddress>,
        pub updated_at: Timestamp,
    }


Schema Embedding
----------------

A CIS-10 contract module MUST embed its Concordium schema in the compiled
``.wasm.v1`` artifact (``cargo concordium build --schema-embed``). All
parameter, return, error, and event types MUST derive ``SchemaType``. This
ensures wallets and explorers can render contract interactions as structured
JSON.


.. _CIS-10-VerificationMechanics:

Verification Mechanics
======================

This section is informative.

Anchor Transaction
------------------

The anchor transaction is any Concordium transaction whose presence on-chain
serves as off-chain-verifiable evidence of the verification event. The
standard does not prescribe the anchor transaction's form. Common patterns:

- A transfer transaction sent by the verifier with a memo containing
  structured verification details (claim, timestamp, off-chain proof URL or
  hash).
- A call to a separate anchor contract that records verification facts.
- A no-op self-transfer with verification metadata in the memo.

The anchor SHOULD be published before the CIS-10 ``register`` call so
consumers can audit the anchor's contents. The CIS-10 contract does not
inspect the anchor's contents — its only on-chain check is the verifier's
signature over the anchor transaction's hash.

Why Sign The Hash
-----------------

Two design considerations motivate signing the anchor transaction hash
directly rather than reconstructing a canonical structured message:

- The anchor hash is a unique, cryptographically-strong commitment to the
  entire verification event. Signing it endorses everything the anchor
  records.
- Verification of the signature is computationally cheap and requires no
  canonical-message reconstruction in the contract — the signed bytes are
  just the 32-byte hash plus the 16-byte domain-separation prefix.

The consequence is that consumers, not the contract, are responsible for
inspecting the anchor's actual contents to confirm the verification was
about the claimed external identifier. The on-chain CIS-10 entry guarantees
only that the verifier endorsed *some* anchor; the anchor itself contains
the substantive claim.

Signature Verification Semantics
--------------------------------

The contract verifies ``proof.signature`` against ``proof.verifier_account``'s
on-chain credentials via ``host.check_account_signature``. Concordium
accounts may have multiple credentials with multiple keys; the contract
MUST accept the signature as valid if it verifies against any current
credential key of the verifier account, per Concordium's standard
account-signature semantics.


Reference Deployments
=====================

This section is informative.

The following contract instances implement this specification on
Concordium-operated networks:

.. list-table::
   :header-rows: 1

   * - Network
     - Contract address
     - Notes
   * - Concordium DevNet P-11
     - ``<18, 0>``
     - Concordium-owned redeploy 2026-05-25
   * - Concordium TestNet
     - ``<12796, 0>``
     - Concordium-owned redeploy 2026-05-25; same module bytes as DevNet


Security Considerations
=======================

Anchor content trust
    The CIS-10 contract does not parse or validate the anchor transaction's
    contents. Consumers verifying an assertion MUST inspect the anchor
    transaction to confirm: (a) the anchor's contents describe the claimed
    ``external_identifier``; (b) the anchor was submitted by an entity the
    consumer trusts (typically, but not necessarily, the
    ``verifier_account`` itself); and (c) any application-specific
    verification policy (e.g., anchor recency, anchor format conformance)
    is satisfied. Without these off-chain checks, the verifier's signature
    only establishes that the verifier endorsed *some* anchor — not that
    the anchor described the claimed binding.

Verifier identity and accountability
    The ``verifier_account`` is itself a Concordium account, which by
    protocol invariant is ID-backed — every Concordium account is created
    from an Identity Provider's ID Object. Consumers therefore have a
    foundation of accountability for the verifier. Consumers MAY maintain
    allowlists of trusted verifier accounts and reject assertions from
    unknown verifiers.

Domain separation
    The 16-byte ``CIS-10/v1/anchor`` prefix prevents a verifier's signature
    accepted by this contract from being reused as a signature in any other
    protocol or contract instance.

Revocation
    Revocation is authorised only by the currently bound account. The
    verifier who created an assertion has no revocation authority. This
    protects against verifier misbehaviour: a verifier cannot retroactively
    withdraw an attestation after the bound account has built reputation
    against the binding. Conversely, the bound account can always revoke an
    assertion they did not consent to.

Replacement semantics
    The replacement rule in :ref:`CIS-10-register` allows a new account to
    take over an external identifier by presenting a valid verifier
    signature. Consumers tracking specific bound accounts SHOULD treat
    replacement as a meaningful state change and re-verify trust
    assumptions.

Admin powers
    The admin can replace the contract code with arbitrary logic via
    ``upgrade``. Deployers SHOULD use a multi-credential Concordium account
    so that admin actions require multi-party authorisation at the account
    layer. There is no upgrade timelock in this version.

Public linkability
    Events publicly log bound accounts, external identifiers, verifier
    accounts, and anchor hashes. This standard is not privacy-preserving by
    design.


Copyright
=========

This document is placed in the public domain.
