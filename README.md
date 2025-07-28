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
| Sirius (P4) | Delegation and new contract state | Jun 13, 2022 | Jun 23, 2022 | [P4.txt](../main/updates/P4.txt) | [P4-commentary.txt](../main/commentary/P4-commentary.txt) | `20c6f246713e573fb5bfdf1e59c0a6f1a37cded34ff68fda4a60aa2ed9b151aa` | `4c135293c661cc57890aab1889e8263db70e3a857983e6aa7ffa5ea6806b9338` | `476093e74014d9c735de0173a653f094f15ee1a4d2693f24bddd184672723d98` |
| P5 | Smart contract upgradability, performance improvements | Nov 22, 2022 | Dec 13, 2022 | [P5.txt](../main/updates/P5.txt) | [P5-commentary.txt](../main/commentary/P5-commentary.txt) | `af5684e70c1438e442066d017e4410af6da2b53bfa651a07d81efa2aa668db20` | `6ca196c7fa2f3e0e64b7fa9b6c7299c5196ff38122768b799fab612db31b2114` | `5443ee296c0a87af8631998e5e7edd80ac4edec5c64255bdf8415af6e4bd0f43` |
| P6 | Concordium BFT consensus protocol, changes in Wasm validation and execution | Aug 21, 2023 | Sep 25, 2023 | [P6.txt](../main/updates/P6.txt) | [P6-commentary.txt](../main/commentary/P6-commentary.txt) | `ede9cf0b2185e9e8657f5c3fd8b6f30cef2f1ef4d9692aa4f6ef6a9fb4a762cd` | `358633697957ac1c3f91adc40f75d1ff951a11bc89826567a4118ce0371c17bf` | `5406c159c36841803d7eecba4aa5c21c6a72693a854ea88851218cfe8b31465c` |
| P7 | Smart contract costs and new queries, stake cooldown changes, disabling shielded transfers, revised block hashing | Sep 30, 2024 | Oct 30, 2024 | [P7.txt](../main/updates/P7.txt) | [P7-commentary.txt](../main/commentary/P7-commentary.txt) | `e68ea0b16bbadfa5e5da768ed9afe0880bd572e29337fe6fb584f293ed7699d6` | `8a3495a1b24d30f239ed9ab52ed6ba26f26f52039d7b740c968882732f3d0baa` | `11bb339c85b02d16c07984deeadb1c338871a49dd1527e129561b8c5b2fb1fb3` |
| P8 | Suspension of inactive validators | Feb 24, 2025 | Mar 17, 2025 | [P8.txt](../main/updates/P8.txt) | [P8-commentary.txt](../main/commentary/P8-commentary.txt) | `f12e20b6936a6b1b736e95715e1654b92adb4226ef7601b4183895bee563f9da` | `3acad056779aaa3d55aa43e92ff2cba0addbb08a8add827ea82da5de83f47ca6` | `c5e029410af1ae257a06e95894cc5277bb9a19d12f5730ed24b2f95c922ecd6e` |
| P9 | Protocol-Level Tokens | - | - | [P9.txt](../main/updates/P9.txt) | [P9-commentary.txt](../main/commentary/P9-commentary.txt) | `1524c737e9d81fe79e4fccfa0b74d11d1cd7d3217b069b4b92f0cb3490d79ba6` | - | -

## Hosted front-end

[Concordium proposals](https://proposals.concordium.software/)

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

#### macOS, Linux and Windows
```
sphinx-autobuild source build
```
and navigate to [localhost:8000](http://localhost:8000).


Before committing, make sure to run the linter and fix all the errors reported:
```
doc8 source
```
