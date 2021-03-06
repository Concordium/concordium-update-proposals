# Concordium update proposals

This repository contains proposals for updates to the Concordium blockchain.
It concerns itself only with protocol updates, not with software updates.

The repository is organized into the following subfolders
- [updates](./updates) contains on the main branch the accepted protocol
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
| Alpha Centauri 1.0 (P1) | Initial protocol at launch of mainnet | - | Jun 09, 2021 | - | - | - | - | - |
| Alpha Centauri 2.0 (P2) | Transaction Memos | Sep 22, 2021 | Oct 13, 2021 | [P2.txt](../main/updates/P2.txt) | [P2-commentary.txt](../main/commentary/P2-commentary.txt) | `9b1f206bbe230fef248c9312805460b4f1b05c1ef3964946981a8d4abb58b923` | `bfa62d327aeffb82978b4bad9e23d9776dddccd5fad100bc26d470c0ed5c322c` | `81bec0077c29616799e4ad9ccde28b5de76c218045eea036a173a2441725fdb3` |
| Alpha Centauri 3.0 (P3) | Account aliases | Dec 06, 2021 | Jan 11, 2022 | [P3.txt](../main/updates/P3.txt) | [P3-commentary.txt](../main/commentary/P3-commentary.txt) | `ec9f7733e872ed0b8f1f386d12c5c725379fc609ce246ffdce28cfb9163ea350` | `7d26c03a30d3156102d50e1c1429786191a002f0d124f82d6447ed3dab9da0d8` | `da7ad0050ac7352183b5731ec81d9b39a064248ba9c4acb8d12daa678040db43` |
| Sirius (P4) | Delegation and new contract state | Jun 13, 2022 | *planned Jun 23, 2022* | [P4.txt](../main/updates/P4.txt) | [P4-commentary.txt](../main/commentary/P4-commentary.txt) | `20c6f246713e573fb5bfdf1e59c0a6f1a37cded34ff68fda4a60aa2ed9b151aa` | TBD | TBD |


## Website

The website is build using [Sphinx](https://www.sphinx-doc.org/en/master/index.html) and reStructuredText ([Link to the basics](https://www.sphinx-doc.org/en/master/usage/restructuredtext/basics.html)).

### Installation

#### Linux

Install `python3` and the python package manager `pip3`.

To install the python dependencies run:
```
pip3 install -r requirements.txt
```

#### macOS

On macOS
```
brew install python3
pip3 install -r requirements.txt
```

#### Windows

Install [python3](https://www.python.org/downloads/windows/)
and select a python installer, e.g. [this one](https://www.python.org/ftp/python/3.9.1/python-3.9.1-amd64.exe).
Download and run the launcher. Make sure to select "Add Python to PATH" at the bottom before proceeding with the install.

After that from a terminal run
```
pip3 install -r requirements.txt
```
from the root of this repository.
### Development

To watch the doc files and automate the build run:

#### macOS and Linux
```
sphinx-autobuild source build
```
and navigate to [localhost:8000](http://localhost:8000).


Before committing, make sure to run the linter and fix all the errors reported:
```
doc8 source
```
