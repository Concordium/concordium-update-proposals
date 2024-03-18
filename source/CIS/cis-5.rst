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

The goal is to simplify the account creation onboarding flow on Concordium 
allowing for CIS5 smart contract wallets to be supported as first-classs citizens on Concordium wallets and tooling.

Specification
=============

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in :rfc:`2119`.

