# ETH Relay

This project contains Ethereum smart contracts that enable the verification of transactions of a "target"
blockchain on a different "verifying" blockchain in a trustless and decentralized way.

This means a user can send a request to the verifying chain asking whether or not a certain transaction
has been included in the target chain and is verified. The verifying chain then provides
a reliable and truthful answer without relying on any third party trust.

The ability to verify transactions "across" different blockchains is vital to enable applications such as
[cross-blockchain token transfers](https://dsg.tuwien.ac.at/projects/tast/pub/tast-white-paper-5.pdf).
> _Important: ETH Relay is a research prototype.
  It represents ongoing work conducted within the [TAST](https://dsg.tuwien.ac.at/projects/tast/)
  research project. Use with care._

## Get Started

ETH Relay is best enjoyed through the accompanying go-ethrelay CLI tool that can be found
[here](https://github.com/pantos-io/go-ethrelay).  
However, if you want to deploy the contracts manually, follow the steps below.

## Installation

The following guide will walk you through the deployment of ETH Relay with a local blockchain (Ganache)
as the verifying chain and the main Ethereum blockchain as the target blockchain. This means blocks from
the main Ethereum blockchain (source) are forwarded to the ETH Relay contract on the local blockchain (destination).

### Direct installation

In this section, we describe how to install ETH Relay directly based on Node.js
and Truffle.
You can find an alternative installation guide based on Docker below.

#### Direct installation: prerequisites

You need to have the following tools installed:

* [Node.js](https://nodejs.org/) (version >= 10.1)
* [Truffle](https://www.trufflesuite.com/truffle) (version >= 5.2.0)
* [Ganache](https://www.trufflesuite.com/ganache) (version >= 2.1)
* [Solidity](https://docs.soliditylang.org/en/latest/installing-solidity.html) (0.9 > version >= 0.8.4)

For simply running the tests, it is not necessarily required to use Ganache as
Truffle provides an integrated blockchain that is used for automatic testing.

#### Direct installation: deployment

1. Clone the repository: `git clone git@github.com:pantos-io/ethrelay.git`
2. Change into the project directory: `cd ethrelay/`
3. Install all dependencies: `npm install`
4. Deploy contracts: `truffle migrate --reset`

#### Direct installation: testing

Run the tests with `truffle test`.

### Docker installation

As an alternative to the direct installation method described above, you can
install ETH Relay with Docker.

#### Docker installation: prerequisites

You need to have

* [Docker](https://www.docker.com/) (version >= 19)

installed.

#### Docker installation: deployment

There are two different ways how to deploy ETH Relay with Docker.
First alternative:

1. Run Ganache: `docker run --network=host trufflesuite/ganache-cli -p 7545`
   (alternatively, you can also run the GUI variant of Ganache without Docker)
2. Deploy contracts:
   `docker run --network=host pantosio/ethrelay migrate --reset`

Second alternative:

1. Clone the repository: `git clone git@github.com:pantos-io/ethrelay.git`
2. Change into the project directory: `cd ethrelay/`
3. Run Ganache and deploy contracts: `docker-compose up`

#### Docker installation: testing

Run the tests with `docker run --network=host pantosio/ethrelay test`.

### Export contract

For the export script to work correctly,
you should set the `GOETHRELAY` environment variable to the project root of [go-ethrelay](https://github.com/pantos-io/go-ethrelay) on your machine, e.g.,
`export GOETHRELAY=~/code/.../go-ethrelay/`. By default, it exports to `${GOPATH}/src/github.com/pantos-io/go-ethrelay`.

To generate the Go contract files and export them to the [go-ethrelay](https://github.com/pantos-io/go-ethrelay) project run `./export.sh`

## How it works

Users can query the ETH Relay contract living on the verifying blockchain by sending requests like
"Is transaction _tx_ in block _b_ part of the target blockchain?"
For the contract to answer the request it has to verify two things.

1. Verify that block _b_ is part of the target blockchain.
2. Verify that transaction _tx_ is part of block _b_.

The way this is achieved is the following:

### 1. Verifying Blocks

To verify that a block _b_ is part of the target blockchain, the ETH Relay contract on the verifying
chain needs to know about the state of the target blockchain.

For that, clients continuously submit block headers of the target chain to the ETH Relay contract.
For each block header that the contract receives, it performs a kind of light validation:

   1. Verify that the block's parent already exists within the contract.
   2. Verify that the block's number is exactly one higher than its parent.
   3. Verify that the block's timestamp is correct.
   4. Verify that the block's gas limit is correct.

If these checks are successful, the contract accepts the block and stores it internally.

The contract does not verify the Proof of Work (PoW) for each block it receives,
as validating the PoW for every block becomes very expensive.
Instead, the contract assigns a dispute period to newly added blocks. Within this period, clients have the
possibility to dispute any block they think is illegal.

In case of a dispute, the full PoW verification is carried out.
If the verification fails, the block and all its successors are removed from the contract.

This way, the target chain is replicated within the ETH Relay contract on the verifying chain,
so that the contract can reliably provide an answer to the question whether or not a block _b_ is part
of the target blockchain.

### 2. Verifying Transactions

So now the contract knows whether or not a block _b_ is part of the target blockchain.
It now needs to verify that the transaction _tx_ is part of block _b_.

Whenever a client sends a transaction verification request to the contract,
it needs to generate a [Merkle Proof](https://dsg.tuwien.ac.at/projects/tast/pub/tast-white-paper-5.pdf) first.
The Merkle Proof is sent with the request and is then verified by the contract.
If the proof validation is successful, transaction _tx_ is part of block _b_. If the validation fails,
_tx_ is not part of _b_.

A more detailed explanation of the inner workings can be found [here](https://dsg.tuwien.ac.at/projects/tast/pub/tast-white-paper-6.pdf).

### Wrong Compiler Version

```text
Error: Source file requires different compiler version (current compiler is 0.6.12+commit.27d51765.Linux.g++) - note that nightly builds are considered to be strictly less than the released version
 --> contracts/Ethrelay.sol:2:1:
  |
2 | pragma solidity >=0.7.0 <0.9.0;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Make sure the recommended Solidity compiler version (0.9 > version >= 0.7) is used with `solc --version`.

#### Legacy Access Request Rate Exceeded Error

When running the tests you might run into the following error: `Returned error: legacy access request rate exceeded` or `Returned error: project ID is required`.

To fix this error, create an account with [Infura](https://infura.io/register) and create a new Infura project.
Then in file `./constants.js` change the constant `INFURA_ENDPOINT` to the mainnet URL from your Infura project (e.g. `https://mainnet.infura.io/v3/ab050ca78686478a9e9b06dfc4b2f069`).

#### Wrong Truffle Version

When running the tests you might run into the following error: `TypeError: param.substring is not a function`.

To fix this error, make sure the recommended Truffle version is used, i.e. v5.1.29.

## How to contribute

ETH Relay is a research prototype. We welcome anyone to contribute.
File a bug report or submit feature requests through the [issue tracker](https://github.com/pantos-io/ethrelay/issues).
If you want to contribute feel free to submit a pull request.

## Acknowledgements

* The development of this prototype was funded by [Pantos](https://pantos.io/) within the [TAST](https://dsg.tuwien.ac.at/projects/tast/) research project.
* The original code for the Ethash contract that is partly used in this project comes from the [smartpool project](https://github.com/smartpool).
* The code for the RLPReader contract that is partly used in this project comes from [Hamdi Allam](https://github.com/hamdiallam/Solidity-RLP) with parts
of it taken from [Andreas Olofsson](https://github.com/androlo/standard-contracts/blob/master/contracts/src/codec/RLP.sol).

## Licence

This project is licensed under the [MIT License](LICENSE).
