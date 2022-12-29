+++
author = "merkleplant"
title = "Property-based Testing Ampleforth"
date = "2022-12-29"
+++

In this article, we are going to property-based test Ampleforth’s rebasing AMPL token.

Using [foundry](https://getfoundry.sh/)’s fuzzing capabilities we first apply a set of pseudo-random input to the AMPL
token, and afterwards, check whether a set of properties still hold.
<!--more-->


## Introduction

Property-based testing describes a testing technique in which not state transitions are tested,
but rather properties of states itself.

Via foundry's fuzzing capabilities we can receive pseudo-random input into our tests, enabling us to execute randomized
state transitions to the AMPL contract.

### Acknowledgment

I want to thank [a16z](https://a16zcrypto.com/) for publishing their [ERC-4626 property tests](https://github.com/a16z/erc4626-tests),
which I used as inspiration when creating this test suite.


## Ampleforth

In essence, the [Ampleforth protocol](https://www.ampleforth.org/) uses the AMPL token to offer an inflation-resistant
unit of account.

See, for instance, my article [Ampleforth is Hayek Money](https://garden.merkleplant.xyz/post/ampleforth-is-hayek-money/)
or this [collection of materials](https://merkleplant.notion.site/Ampleforth-Wiki-32d7e4b2f7474e4fb8ea48cb0ecba092)
for further information on AMPLs monetary qualities.

Going more into the technical details, AMPL is a non-dilutive rebasing token.
The implementation derives an external, elastic user balance by dividing a fixed internal user balance with a
conversion rate (also called _scalar_ or _rebase factor_).

Through the rebase operation, which updates the conversion rate, the protocol mutates all (external) user balances in
constant time. It is also important to note that this makes a rebase non-dilutive as each user balance is effected
equally through the updated conversion rate.

We will only discuss AMPL's internal operations in as much detail as is required for this post.
I do, however, urge you to review the remaining codebase which, in my humble opinion, serves as an illustration of a
simple and robust smart contract system.


## Setting up the Project

> Note that you can find the whole project [here](https://github.com/pmerkleplant/ampleforth-property-tests).

As always, first set up a new foundry project via `forge init`. Afterwards, install Ampleforth's contracts as dependency
via `forge install https://github.com/ampleforth/ampleforth-contracts`.

After deleting the default contracts, create two files inside the `test/` directory - `AMPL.p.sol` and `AMPL.t.sol`.

Copy the following contract into the `AMPL.p.sol` file:
```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity ^0.7.6; // Ampleforth is based on version 0.7.6.

import "forge-std/Test.sol";

// The token's name was originally UFragments. Let's rename it.
import {UFragments as AMPL} from "ampleforth-contracts/UFragments.sol";

abstract contract AMPLProp is Test {
    AMPL ampl; // The AMPL token instance we are going to test.

    // Constants copied from AMPL.
    uint constant MAX_SUPPLY = type(uint128).max;
}
```

Inside the `AMPLProp` contract we are going to define our properties and test whether they are true for the declared
`ampl` token instance.


## Properties

### Total Supply

The initial properties we test for is that the total supply should not be 0 or greater than `MAX_SUPPLY`.

In Solidity:
```solidity
/// @dev The total supply never exceeds the defined max supply.
function prop_TotalSupplyNeverExceedsMaxSupply() public {
    assertTrue(ampl.totalSupply() <= MAX_SUPPLY);
}

/// @dev The total supply is never zero.
function prop_TotalSupplyNeverZero() public {
    assertTrue(ampl.totalSupply() != 0);
}
```

Next, a property of balances in regard to the total supply can be formalized.
While the total supply equals the sum of all balances in the majority of ERC20 implementations, a rebase token's
balance is computed by dividing an internal balance with a rebase factor, and due to integer divisions having rounding
errors if the result is a floating number, the `AMPL` contract [states](https://github.com/ampleforth/ampleforth-contracts/blob/ab5abe27fc5b107d9acacd9199809760f35a2ac7/contracts/UFragments.sol#L35-L37) the following:
```c
> We do not guarantee that the sum of all balances equals the result of calling totalSupply().
> This is because, for any conversion function 'f()' that has non-zero rounding error,
> f(x0) + f(x1) + ... + f(xn) is not always equal to f(x0 + x1 + ... xn).
```

Therefore, our property dilutes to:
```solidity
/// @dev The sum of all balances never exceeds the total supply.
function prop_SumOfAllBalancesNeverExceedsTotalSupply(address owner, address[] memory users) public {
    // Note that the owner and the set of users are the only addresses that may have a non-zero balance.
    uint sum = ampl.balanceOf(owner);
    for (uint i; i < users.length; i++) {
        sum += ampl.balanceOf(users[i]);
    }

    assertTrue(sum <= ampl.totalSupply());
}
```


### Rebase

As mentioned already, rebases are non-dilutive operations. Therefore, the internal balances prior to and following a
rebase must be equal. (Note that the internal balance in AMPL is called `gons`)

```solidity
/// @dev A rebase operation is non-dilutive, i.e. does not change the wallet wealth distribution.
function prop_RebaseIsNonDilutive(address[] memory users, int supplyDelta) public {
    uint[] memory gonBalancesBeforeRebase = new uint[](users.length);
    for (uint i; i < users.length; i++) {
        gonBalancesBeforeRebase[i] = ampl.scaledBalanceOf(users[i]);
    }

    try ampl.rebase(1, supplyDelta) {} catch {}

    for (uint i; i < users.length; i++) {
        assertEq(ampl.scaledBalanceOf(users[i]), gonBalancesBeforeRebase[i]);
    }
}
```

We can also check that the external balances are always computed correctly by computing the conversion rate ourself
and applying it to the internal balances.

The `scaledTotalSupply() returns (uint)` function returns the total supply of the fixed, internal balances.
Dividing this supply by the current `totalSupply() returns (uint)` gives us the current rebase conversion rate.

Following, dividing a user's internal balance with our self-computed conversion rate should equal the user's external
balance.

```solidity
/// @dev The gon-AMPL conversion rate is the (fixed) scaled total supply divided by the (elastic) total supply.
function prop_ConversionRate(address[] memory users) public {
    uint gonsPerAMPL = ampl.scaledTotalSupply() / ampl.totalSupply();

    for (uint i; i < users.length; i++) {
        uint gonBalance = ampl.scaledBalanceOf(users[i]);
        uint amplBalance = ampl.balanceOf(users[i]);

        assertEq(gonBalance / gonsPerAMPL, amplBalance);
    }
}
```


### Transfer

Inside the `AMPL` contract it [states](https://github.com/ampleforth/ampleforth-contracts/blob/ab5abe27fc5b107d9acacd9199809760f35a2ac7/contracts/UFragments.sol#L30-L33):
```c
> We make the following guarantees:
>
> If address 'A' transfers x Fragments to address 'B'. A's resulting external balance will be decreased by precisely x
> Fragments, and B's external balance will be precisely increased by x Fragments.
```

In Solidity:
```solidity
/// @dev A transfer of x AMPLs from A to B results in A's external balance being decreased by precisely
///      x AMPLs and B's external balance being increased by precisely x AMPLs.
function prop_ExternalBalanceIsPreciseAfterTransfer(address owner, address[] memory users) public {
    // Note that the owner and the set of users are the only addresses that may have a non-zero balance.
    uint before = ampl.balanceOf(owner);

    for (uint i; i < users.length; i++) {
        uint wantIncrease = ampl.balanceOf(users[i]);

        vm.prank(users[i]);
        ampl.transfer(owner, wantIncrease);

        assertEq(ampl.balanceOf(users[i]), 0);
        assertEq(before + wantIncrease, ampl.balanceOf(owner));

        before += wantIncrease;
    }
}
```

The requirement that a zero transfer should always succeed is another, admittedly dull, property:

```solidity
/// @dev A transfer of zero AMPL is always possible, independent of whether the `transfer` or `transferFrom`
///      function is used.
function prop_ZeroTransferAlwaysPossible(address[] memory users) public {
    address user;
    for (uint i; i < users.length; i++) {
        user = users[i];

        vm.startPrank(user);
        {
            ampl.transfer(user, 0);
            ampl.transferFrom(user, user, 0);
        }
        vm.stopPrank();
    }
}
```

An interesting function offered by AMPL is `transferAllFrom`. We want to state that it is not possible to transfer any
tokens when the spender's allowance for the owner's tokens is zero. Note that we assume that any call transfering a
non-zero amount of internal tokens shoud fail.

```solidity
/// @dev Function `transferAllFrom` reverts if scaledBalanceOf owner is not zero while allowance from owner to
///      spender is zero.
function prop_TransferAllFromRevertsIfAllowanceIsInsufficient(address[] memory users) public {
    address spender;
    address owner;
    address receiver;
    for (uint i; i < users.length; i++) {
        // Note that users do not have allowance set for themselves.
        spender = users[i];
        owner = users[i];
        receiver = users[i];

        // Expect a revert if the scaledBalance, i.e. gon balance, of the owner is unequal to zero.
        bool expectRevert = ampl.scaledBalanceOf(owner) != 0;

        if (expectRevert) {
            vm.expectRevert();
        }
        vm.prank(spender);
        ampl.transferAllFrom(owner, receiver);
    }
}
```

## Creating the State

Having enough properties defined, lets bring the AMPL token into a pseudo-random state and test the properties.

Copy the following contract into the `AMPL.t.sol` file:
```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity ^0.7.6;
pragma abicoder v2;

import {UFragments as AMPL} from "ampleforth-contracts/UFragments.sol";

import {AMPLProp} from "test/AMPL.p.sol";

// Note that AMPLTest inherits from AMPLProp.
contract AMPLTest is AMPLProp {
    struct State {
        // Privileged addresses
        address owner;
        address monetaryPolicy;
        // Rebases
        int[] supplyDeltas;
        // List of addresses that could have a non-zero balance
        // Note that owner can also have a non-zero balance
        address[] users;
        // List of transfers tried to execute
        uint[] transfers;
    }

    // Actual setUp function is not used.
    function setUp() public virtual {}

    // This is our "real" setUp function.
    function setUpAMPL(State memory state) public {
        _assumeValidState(state);

        ampl = new AMPL();
        ampl.initialize(state.owner);

        vm.prank(state.owner);
        ampl.setMonetaryPolicy(state.monetaryPolicy);

        // Try to execute all transfers from owner to users.
        // Note that the owner has the full initial supply and holds the total internal supply.
        // The initial total supply is 50M.
        for (uint i; i < state.transfers.length; i++) {
            vm.prank(state.owner);
            try ampl.transfer(state.users[i], state.transfers[i]) {} catch {}
        }

        // Try to execute list of rebases.
        for (uint i; i < state.supplyDeltas.length; i++) {
            vm.prank(state.monetaryPolicy);
            try ampl.rebase(1, state.supplyDeltas[i]) {} catch {}
        }
    }

    // Example test function.
    function test_prop_TotalSupplyNeverExceedsMaxSupply(State memory state) public {
        // First set up the state...
        setUpAMPL(state);
        // ...than verify that this state holds the property.
        prop_TotalSupplyNeverExceedsMaxSupply();
    }

    // Test the rest of the properties here.

    /*//////////////////////////////////////////////////////////////
                                HELPERS
    //////////////////////////////////////////////////////////////*/

    mapping(address => bool) usersCache;

    // Filter out some non-conforming input. This is mostly specific to internals inside the AMPL token.
    function _assumeValidState(State memory state) internal {
        vm.assume(state.owner != address(0));
        vm.assume(state.monetaryPolicy != address(0));

        // User assumptions.
        vm.assume(state.users.length != 0);
        vm.assume(state.users.length < 10e9);
        for (uint i; i < state.users.length; i++) {
            // Make sure user is neither owner nor monetary policy.
            vm.assume(state.users[i] != state.owner);
            vm.assume(state.users[i] != state.monetaryPolicy);

            // Make sure user is valid recipient.
            vm.assume(state.users[i] != address(0));
            vm.assume(state.users[i] != address(ampl));

            // Make sure user is unique.
            vm.assume(!usersCache[state.users[i]]);
            usersCache[state.users[i]] = true;
        }

        // Transfer assumptions.
        vm.assume(state.transfers.length != 0);
        vm.assume(state.transfers.length <= state.users.length);

        // Rebase assumptions.
        vm.assume(state.supplyDeltas.length != 0);
        vm.assume(state.supplyDeltas.length < 1000);
    }
}
```

Note that the `AMPLTest` contract inherits from the `AMPLProp` contract.

We created a `function test_prop_TotalSupplyNeverExceedsMaxSupply()` that takes a fuzzed argument struct called `State`.
The `State` contains some privileged addresses for the AMPL token, a set of `supplyDelta`s that we apply to the total
supply by using AMPL's `rebase()` function, and a list of user addresses to which we transfer tokens from the owner, by
using the amounts given in the `State`'s `transfers` array.

Each `test_` function first calls the `setUpAMPL()` function to apply the `State` arguments fields to the AMPL instance.
A lot of the transfer and rebase operations will fail due to overflows, insufficient balances, etc, but we do not care
too much about the level of entropy we achieve using this simple mechanism.

(Note: If looking for tools enabling deeper state explorations, check out [echidna](https://github.com/crytic/echidna) or [Scribble](https://docs.scribble.codes/))


## Execution

Running `forge test` could now result in an output similar to this:
```c
[PASS] test_prop_ConversionRate((address,address,int256[],address[],uint256[])) (runs: 256, μ: 7942226, ~: 8305177)
[PASS] test_prop_ExternalBalanceIsPreciseAfterTransfer((address,address,int256[],address[],uint256[])) (runs: 256, μ: 9075672, ~: 9656829)
[PASS] test_prop_RebaseIsNonDilutive((address,address,int256[],address[],uint256[])) (runs: 256, μ: 7658408, ~: 7942935)
[PASS] test_prop_SumOfAllBalancesNeverExceedsMaxSupply((address,address,int256[],address[],uint256[])) (runs: 256, μ: 7635333, ~: 7876323)
[PASS] test_prop_TotalSupplyNeverExceedsMaxSupply((address,address,int256[],address[],uint256[])) (runs: 256, μ: 7049481, ~: 7347126)
[PASS] test_prop_TotalSupplyNeverZero((address,address,int256[],address[],uint256[])) (runs: 256, μ: 7114705, ~: 7254536)
[FAIL. Reason: Call did not revert as expected Counterexample: calldata=..., args=[...] test_prop_TransferAllFromRevertsIfAllowanceIsInsufficient((address,address,int256[],address[],uint256[])) (runs: 13, μ: 8019120, ~: 8272453)
```

It appears that at some point, the `transferAllFrom` function did not revert. This indicates that it was possible to
transfer some amount without the necessary permission.


## Evaluating the Bug

Here is a screenshot of the relevant section of the failing test's stack trace:

![stack_trace](/post/property-based-testing-ampleforth/transferAllFromFailureStackTrace.png)

In fact, the transfer function transferred 0 AMPLs, as seen by the stack trace. It did, however, transfer some
non-zero amount of `gons`.

Let's look at the function's implementation to gain a better understanding (comments added for clarity):

```solidity
function transferAllFrom(address from, address to) external validRecipient(to) returns (bool) {
    // Get the internal balance of `from`.
    uint256 gonValue = _gonBalances[from];
    // Calculate the external balance by dividing the internal balance with the rebase conversion rate.
    uint256 value = gonValue.div(_gonsPerFragment); // <-- !!!

    // Reverts in case the caller's allowance is not sufficient to transfer the external balance.
    _allowedFragments[from][msg.sender] = _allowedFragments[from][msg.sender].sub(value);

    // Delete the owner's total internal balance...
    delete _gonBalances[from];
    // ...and increase the `to`'s internal balance.
    _gonBalances[to] = _gonBalances[to].add(gonValue);

    emit Transfer(from, to, value);
    return true;
}
```

First, notice that the token's allowance - managed via the `_allowedFragments` mapping - is denominated in AMPL.
Therefore, the owner's `gon` balance needs to be converted to its AMPL balance, which includes a division.

What happens if the `gon` value divided by the conversion rate is some value between 0 and 1 (exclusiv)?
Well, the division rounds down to zero.

This leads to the allowance check being useless, i.e. its being checked whether the spender has at least 0 allowance.

Via the `tranferAllFrom` function it is therefore possible to steal from wallets with an AMPL balance of 0, but
non-zero `gons` balance.

However, AMPL's uses 9 decimals and its price target is the [CPI-adjusted 2019 US-Dollar](https://www.ampleforth.org/dashboard/).
Therefore, one AMPL token in the contract evaluates to ~`1.13$ * 1e-9`. **This is an exceedingly small amount of value.**

In a personal project of mine - a fork of [Buttonwood](https://button.foundation/)'s rebasing [ButtonToken](https://github.com/buttonwood-protocol/button-wrappers/blob/d9fe31c38957dc0fe8ae63b3546c370394fca7c2/contracts/ButtonToken.sol#L11) -, the [ElasticReceiptToken](https://github.com/pmerkleplant/elastic-receipt-token/blob/e088eb7fc43f671f3d97f007004be7f0025831f2/src/ElasticReceiptToken.sol#L7),
I fixed the issue by [decrementing the allowance by 1](https://github.com/pmerkleplant/elastic-receipt-token/blob/e088eb7fc43f671f3d97f007004be7f0025831f2/src/ElasticReceiptToken.sol#L279-L284).


## Disclaimer

- I reported the bug to all (as of my knowledge) affected projects in Februrary 2022
- I received from all the projects the allowance to publish the bug
- Yes, I'm a big fan of Ampleforth. It's a beautiful, simple and resilient protocol that actually offers new monetary
  ideas.
