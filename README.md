
![Build Status](https://github.com/yodaplus/xss/actions/workflows/.github/workflows/tests.yaml/badge.svg?branch=v1.2)

# XDC related notes

USXD - an over-collateralized stablecoin on XDC network - is based on MakerDAO's DAI stablecoin. 
This repository is a fork of [dss](https://github.com/makerdao/dss). 
For most parts, the documentation for DAI also applies to USXD


# Multi Collateral USXD

This repository contains the core smart contract code for Multi
Collateral USXD. This is a high level description of the system, assuming
familiarity with the basic economic mechanics as described in the DAI
whitepaper.

## Additional Documentation

- A more in-depth description of the `xss` core contracts can be found in [DEVELOPING.md](https://github.com/yodaplus/xss/blob/master/DEVELOPING.md)
- The smart contract architecture is also documented in MakerDAO's [dss wiki](https://github.com/makerdao/dss/wiki)

## Design Considerations

- Token agnostic
  - system doesn't care about the implementation of external tokens
  - can operate entirely independently of other systems, provided an authority assigns
    initial collateral to users in the system and provides price data.

- Verifiable
  - designed from the bottom up to be amenable to formal verification
  - the core cdp and balance database makes *no* external calls and
    contains *no* precision loss (i.e. no division)

- Modular
  - multi contract core system is made to be very adaptable to changing
    requirements.
  - allows for implementations of e.g. auctions, liquidation, CDP risk
    conditions, to be altered on a live system.
  - allows for the addition of novel collateral types (e.g. whitelisting)


## Collateral, Adapters and Wrappers

Collateral is the foundation of USXD and USXD creation is not possible
without it. There are many potential candidates for collateral, whether
native XDC, ERC20 tokens, other fungible token standards like ERC777,
non-fungible tokens, or any number of other financial instruments.

Token wrappers are one solution to the need to standardise collateral
behaviour in USXD. Inconsistent decimals and transfer semantics are
reasons for wrapping. For example, the 
[WETH token](https://github.com/makerdao/ds-weth/blob/master/src/weth9.sol) 
is an ERC20 wrapper around native XDC.

We abstract all of these different token behaviours away behind
*Adapters*. Adapters manipulate a single core system function: `slip`, which
modifies user collateral balances.

Adapters should be very small and well defined contracts. Adapters are
very powerful and should be carefully vetted by Governance Token holders. Some
examples are given in `join.sol`. Note that the adapter is the only
connection between a given collateral type and the concrete on-chain
token that it represents.

There can be a multitude of adapters for each collateral type, for
different requirements. For example, XDC collateral could have an
adapter for native XDC and *also* for WETH.

WETH stands for Wrapped ETH as is applicable for MakerDAO's implementation. 
The token could be renamed as WXDC for this project *in the future*.

## The USXD Token

The USXD Token is a modified "Dai" token (as created for MakerDAO). 
As such, only the symbol & name of the token have been customized for XDC.
The token itself continues to be in the `dai.sol` smart contract and 
is referenced as Dai / dai throughout.

The fundamental state of a USXD balance is given by the balance in the
core (`vat.dai`, sometimes referred to as `D`).

Given this, there are a number of ways to implement the USXD that is used
outside of the system, with different trade offs.

*Fundamentally, "USXD" is any token that is directly fungible with the
core.*

In the Apothem deployment, "USXD" is represented by an ERC20 DSToken.
After interacting with CDPs and auctions, users must `exit` from the
system to gain a balance of this token, which can then be used in 
Borrow portal etc.

It is possible to have multiple fungible USXD tokens, allowing for the
adoption of new token standards. This needs careful consideration from a
UX perspective, with the notion of a canonical token address becoming
increasingly restrictive. In the future, cross-chain communication and
scalable sidechains will likely lead to a proliferation of multiple USXD
tokens. Users of the core could `exit` into a sidechain, a shard, 
or a different blockchain entirely.


## Price Feeds

Price feeds are a crucial part of the USXD system. The code here assumes
that there are working price feeds and that their values are being
pushed to the contracts.

Specifically, the price that is required is the highest acceptable
quantity of CDP USXD debt per unit of collateral.


## Liquidation and Auctions

The auctions implemented here are simple and expect liquidations to
occur in *fixed size lots* (say 10,000 XDC).


## Settlement

System Debt is debt that has been taken from risky CDPs.
This is covered by dilution of an external token,
namely MKR (which is the governance token).

As in collateral liquidation, this dilution occurs by an auction
(`flop`), using a fixed-size lot.

In order to reduce the collateral intensity of large CDP liquidations,
MKR dilution is delayed by a configurable period (e.g 1 week).

Similarly, System Surplus is handled by an auction (`flap`), which sells
off USXD surplus in return for the highest bidder in MKR.


## Authentication

The contracts here use a very simple multi-owner authentication system,
where a contract totally trusts multiple other contracts to call its
functions and configure it.

It is expected that modification of this state will be via an interface
that is used by the Governance layer.

# Development Environment

## Prerequisites

- [dapp.tools](https://github.com/dapphub/dapptools)
- solidity 0.5.0

## Test

The tests can be run using the `dapp test` command which results in all the 
DSTest contracts being run on the test environment. 

The test contracts are present in `src/test`. Further details about the 
functioning of dapp test can be found in [dapp tools documentation](https://github.com/dapphub/dapptools/tree/master/src/dapp#dapp-test)

### Nix Shell config

- [MakerPackage](https://github.com/makerdao/makerpkgs/tarball/master) : This package is used by nix shell to deploy prerequisites
- DAPP_TEST_ADDRESS : This is the hevm [environment variable](https://github.com/dapphub/dapptools/tree/master/src/hevm#environment-variables) that defines the address where the test contract is deployed.

## Github Workflows

The `tests.yaml` ensures test cases are run on all push & pull requests

## Deployment

The contracts in this `xss` repository are deployed using [xss-deploy](https://github.com/yodaplus/xss-deploy) repository which contains a smart contract that deploys the core `xss` contracts and sets up the necessary authorisations between them.

`xss-deploy` itself is deployed using [xss-deploy-scripts](https://github.com/yodaplus/xss-deploy-scripts)