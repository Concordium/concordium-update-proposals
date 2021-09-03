# Concordium update proposals

This repository contains proposals for updates to the Concordium blockchain.
It concerns itself only with protocol updates, not with software updates.

The repository is organized into the following subfolders
- [accepted](./accepted) contains on the main branch the accepted protocol
  updates that will be implemented. Each update is numbered sequentially and the
  file serves as the specification of the protocol update that is used by the
  node to validate the update instruction.
- [commentary](./commentary) contains additional commentary for each update that
  provides background information and outlines consequences of the update for
  users of the chain, node runners, etc.
  
## Overview

The table lists already effective as well as *planned/draft* protocol updates.

| Protocol Version | Protocol Update | Effective on Testnet | Effective on Mainnet | Protocol Specification | Commentary | Specification Hash | Transaction Hash (Mainnet) | Block Hash (Mainnet) |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| P1 | Initial protocol at launch of mainnet | - | Jun 09, 2021 | - | - | - | - | - |
| P2 | Transaction Memos | *planned  Sep 22, 2021* | *planned  Oct 25, 2021* | *draft <br/> [P2.txt](../main/accepted/P2.txt)* | *draft <br/> [P2-commentary.txt](../main/commentary/P2-commentary.txt)* | `9b1f206bbe230fef248c9312805460b4f1b05c1ef3964946981a8d4abb58b923`  |   |   |
