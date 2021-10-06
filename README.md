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
| Alpha Centauri 2.0 (P2) | Transaction Memos | Sep 22, 2021 | *planned  Oct 13, 2021* | [P2.txt](../main/updates/P2.txt) | [P2-commentary.txt](../main/commentary/P2-commentary.txt) | `9b1f206bbe230fef248c9312805460b4f1b05c1ef3964946981a8d4abb58b923`  |   |   |


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
