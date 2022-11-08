+++
author = "merkleplant"
title = "Entering the Huff Ecosystem"
date = "2022-07-04"
+++

This article introduces the evolving [Huff](https://huff.sh/) language and
ecosystem by developing a non-trivial contract, an `Ownable` contract with a
Two-Step Transfer pattern, called `TSOwnable`.
<!--more-->

Along the way we will dive into tools such as the new [`huff-rs`](https://github.com/huff-language/huff-rs)
compiler and [`HuffDeployer`](https://github.com/huff-language/foundry-huff) library,
some better known tools such as `forge` and `cast`, and will learn how to
write low-level EVM code.

In order to follow along, please keep the fantastic [evm.codes](http://evm.codes/)
website open.

Note that this post is not an introduction to how the EVM works.
There are other great resources for that, e.g. [The EVM Handbook](https://blog.merkleplant.xyz/bb38e175cc404111a391907c4975426d).


## About Huff

Note that this paragraph is copied from the [`awesome-huff`](https://github.com/devtooligan/awesome-huff)
repo created by the one and only [devtooligan](https://twitter.com/devtooligan).

> Huff is a low-level programming language designed for developing highly
> optimized smart contracts that run on the Ethereum Virtual Machine (EVM).
> Huff does not hide the inner workings of the EVM.
> Instead, Huff exposes its programming stack and provides useful tools like
> constants and macros.

> Initially developed by the [Aztec Protocol](https://github.com/AztecProtocol)
> team, Huff was created to write [Weierstrudel](https://github.com/AztecProtocol/weierstrudel),
> an on-chain elliptical curve arithmetic library that requires incredibly
> optimized code which neither Solidity nor Yul could provide.

> Huff can be used to write highly-efficient smart contracts for use in
> production, or it can serve as a way for beginners to learn more about the EVM.

Please take some time to read the [README](https://github.com/AztecProtocol/huff)
in AztecProtocol’s Huff implementation to gain a feeling of how the language
looks like.


## The `TSOwnable` Contract

The famous `Ownable` contract has the weakness that the contract’s owner can
accidentally be set to an address that is not controlled by the project.

Using a Two-Step Transfer pattern reduces the risk of accidentally loosing
control over a contract. The newly set owner (called `pendingOwner`) first has
to accept the ownership of the contract before the owner switch is completed.
In case the `pendingOwner` is accidentally set to an address not controlled by
the project, the contract’s ownership is not lost directly and the
`pendingOwner` can be reset.

Take a look at this [TSOwnable](https://github.com/byterocket/solrocket/blob/main/src/TSOwnable.sol) implementation in Solidity.
The goal of the post is to write a functionally-equivalent contract in Huff.


## Setting up the Project

We will be using the [foundry toolkit](https://getfoundry.sh/),
a “_blazing fast, portable and modular toolkit for Ethereum application development_".

To compile Huff contracts to EVM bytecode we will use the new [huff-rs](https://github.com/huff-language/huff-rs) compiler.

After installation, create and `cd` into a new directory, and run `forge init`.

In order to compile and deploy Huff contracts from within the `forge` tool,
install Huff’s [HuffDeployer](https://github.com/huff-language/foundry-huff)
library with `forge install huff-language/foundry-huff`.

If running `forge test` only emits green colors, you are ready to start.


## Storage Slots and Constructor

First, delete the default contract in the `src/` directory and create a new
file called `TSOwnable.huff`.

Looking at the `TSOwnable` Solidity reference implementation, we see that we
need two storage variables, `address public owner` and
`address public pendingOwner`.
Huff does not support the concept of variables, but rather expects us to manage
the storage directly.

Declare two constants, each holding a reference to a storage slot:

```c
#define constant OWNER_SLOT = FREE_STORAGE_POINTER()
#define constant PENDING_OWNER_SLOT = FREE_STORAGE_POINTER()
```

The `FREE_STORAGE_POINTER()` macro is a builtin macro that returns, well, the
next free storage slot.

EVM storage slots are starting from 0, which means that `OWNER_SLOT = 0x00` and
`PENDING_OWNER_SLOT = 0x01`. Note that while one storage slot contains 32 bytes
of data, the slots itself are addressed incrementally.

Next, we need the constructor.
It should save the deployer address as `owner` and leave the `pendingOwner`
variable as zero.

```c
#define macro CONSTRUCTOR() = takes (0) returns (0) {
    caller [OWNER_SLOT] sstore // Store msg.sender as owner
}
```

The `CONSTRUCTOR` is a reserved keyword in Huff.

The `takes (0) returns (0)` signature indicates how many elements the macro
consumes from the stack and how many elements the macro puts on the stack after
execution. The signature does not indicate how many function arguments,
i.e. data read from calldata, the macro is expecting.

In this case, the constructor is neither expecting any arguments nor any
elements on the stack.

First, the constructor pushes the caller’s address (via the `caller` opcode)
and the `owner` variable’s storage slot (via the `[OWNER_SLOT]` Huff
instruction) on the stack.

Afterwards, the `sstore` opcode is executed. The `sstore` opcode consumes two
elements from the stack, the first element being the key and the second element
the value, and stores the value in the storage slot at position key.

Visualizing the stack after each instruction looks like this:

```c
#define macro CONSTRUCTOR() = takes (0) returns (0) {
                  // [] - Stack is empty
    caller        // [msg.sender] - Caller is pushed on the stack
    [OWNER_SLOT]  // [0x00, msg.sender] - The OWNER_SLOT (0x00) is pushed on the stack
    sstore        // [] - The sstore opcode consumed both elements from the stack
                  // The first element, 0x00, is interpreted as the storage key
                  // The second element, msg.sender, is the value stored
}
```

Checking the `TSOwnable` Solidity reference implementation indicates that our
constructor should now be functionally equivalent.


## Getter Functions

Now we need to create two getter functions in order to read the `owner` and
`pendingOwner` variables.

The functions should load 32 bytes (one word) from the variable’s storage slot
and return them.

Take a moment to read about the the `mstore`, `sload`, and `return` opcodes on
[evm.codes](https://evm.codes).

The `sload` opcode consumes one element from the stack, interprets that element
as a storage slot reference, and pushes the storage slot’s content on the stack.

The `mstore` opcode consumes two elements from the stack. It stores the second
element at the memory offset of the first element.

The `return` opcode expects two elements on the stack as well. The first element
is interpreted as the memory offset, the second element as the amount of bytes
to return. Furthermore, `return` signals a successful exit.

Having an understanding of the three opcodes, the getter functions can be
implemented as follows:

```c
#define macro OWNABLE_GET_OWNER() = takes (0) returns (0) {
    [OWNER_SLOT] sload   // Load owner address from storage and push onto stack
    0x00 mstore          // Store owner to memory at index 0x00
    0x20 0x00 return     // Exit context, return 0x20=32 bytes from memory, starting at memory index 0x00
}

#define macro OWNABLE_GET_PENDING_OWNER() = takes (0) returns (0) {
    [PENDING_OWNER_SLOT] sload
    0x00 mstore
    0x20 0x00 return
}
```

While this macros seem to be able to return the corresponding addresses, how can
an external contract call them? After all, a macro is not a function.


## Function Dispatching

In order to call the macro for external calls to `owner()` and `pendingOwner()`,
 respectively, we need to dispatch function calls to the corresponding macros.

The main entrypoint in Huff contracts is defined via the `MAIN()` macro.
A function signature is defined as the first 4 bytes of the `keccak256` hash of
a function signature written in Solidity.

### Computing the Function Signatures

In order to copmute the function signatures we use foundry’s `cast` tool.
It is as easy as running `cast sig "<function signature>"`.

Running `cast sig "owner()"` and `cast sig "pendingOwner()"` returns
`0x8da5cb5b` and `0xe30c3978`. The function signature is always the first
4 bytes of the calldata in a call.

Now, we only need to extract the first 4 bytes, compare them to our signatures
and invoke the corresponding macros.

The code to do that looks as follows:

```c
#define macro MAIN() = takes (0) returns (0) {
    0x00 calldataload 0xe0 shr

    // cast sig "owner()"
    dup1 0x8da5cb5b eq get_owner jumpi
    // cast sig "pendingOwner()"
    dup1 0xe30c3978 eq get_pending_owner jumpi

    get_owner:
        OWNABLE_GET_OWNER()
    get_pending_owner:
        OWNABLE_GET_PENDING_OWNER()
}
```

The first line inside the `MAIN` macro reads 32 bytes of calldata starting from
index 0 and shifts them right by `0xe0 = 224 bits`.

Remember, that 32 bytes are 256 bits and 4 bytes are 32 bits.
Computing `256 - 224 = 32`, indicates that we moved 224 bits “out” to the right,
leaving us with only the first 4 bytes, i.e. 32 bits.

Afterwards, we duplicate these 4 bytes on the stack via the `dup1` opcode, push
the function signature on the stack, and check if the two are equal via the `eq`
opcode.

The `eq` opcode consumes two elements from the stack and pushes one back. If the
two elements on the stack were equal, the resulting element on the stack is 1,
otherwise 0.

The `get_owner` and `get_pending_owner` Huff instructions are labels that get
translated from the Huff compiler into byte offsets in the deployed code.
The `jumpi` opcode jumps to a specific location inside the code, i.e. to the
labels, if the second element on the stack, i.e. the result of the `eq`
execution, is unequal to zero.

To summarize:
> If the first 4 bytes of the calldata equal our function selector, we invoke
> the corresponding macro. The macro exits the context and returns the corresponding
> address.


## Deploying the Contract and Testing the Constructor

Now let’s test whats implemented so far. Delete the default `Contract.t.sol`
contract in the `test/` directory, create a new contract called
`TSOwnable.t.sol`, and copy the following code into the file:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import "forge-std/Test.sol";

import {HuffDeployer} from "foundry-huff/HuffDeployer.sol";

interface ITSOwnable {
    function owner() external view returns (address);
    function pendingOwner() external view returns (address);

    function setPendingOwner(address pendingOwner) external;
    function acceptOwnership() external;
}

contract TSOwnableTest is Test {

    // The System under Test.
    ITSOwnable sut;

    function setUp() public {
        address impl = HuffDeployer.deploy("TSOwnable");
        sut = ITSOwnable(impl);
    }

    function testDeploymentInvariants() public {
        assertEq(sut.owner(), address(this));
        assertEq(sut.pendingOwner(), address(0));
    }
}
```

This code should look familiar to anyone using foundry already. The only new
thing is the `HuffDeployer` library used in the `setUp()` function.

`forge`, the foundry tool used i.a. for testing, is not (_yet?_) natively capable
of compiling Huff contracts. The `HuffDeployer` library internally calls the
`huff-rs` compiler, reads the returned contract’s bytecode, deploys it, and
returns the resulting address.

The test function, `testDeploymentInvariants()`, checks that our constructor and
getter functions are working as intended.

Now let’s run forge test to see where we introduced mistakes...

You should receive an output containing something like this:

```
Running 1 test for test/TSOwnable.t.sol:TSOwnableTest
[FAIL. Reason: Setup failed: FFI disabled: run again with `--ffi` if you want to allow tests to call external scripts.] setUp() (gas: 0)
Test result: FAILED. 0 passed; 1 failed; finished in 1.87ms

Failed tests:
[FAIL. Reason: Setup failed: FFI disabled: run again with `--ffi` if you want to allow tests to call external scripts.] setUp() (gas: 0)

Encountered a total of 1 failing tests, 0 tests succeeded
```


## Forge and the Dangerous `--ffi` Flag

The `forge` tool complains that FFI is disabled. FFI stands for
_Foreign Function Interface_ and enables `forge` to call external programs.

- Which program does `forge` try to call here?

  The `FoundryDeployer` library has to call the external `huff-rs` binary to
  compile the `TSOwnable` Huff contract and receive the bytecode.

- Why is `--ffi` dangerous?

  Never call `forge` with the `--ffi` flag if you don’t know which external
  programs are called!

  Via `--ffi`, harmless Solidity code is able to call any program on your
  computer with any arguments. A Solidity test could, for example, call a
  python interpreter, with some python code saved as `string memory` inside
  the Solidity file, and **DOS your local anvil node**.

However, for the moment the `HuffDeployer` library’s external calls seem
harmless and it is safe to call `forge` with `--ffi` enabled.

Calling `forge test --ffi` should now output something like this:

```
Running 1 test for test/TSOwnable.t.sol:TSOwnableTest
[PASS] testDeploymentInvariants() (gas: 10385)
Test result: ok. 1 passed; 0 failed; finished in 12.45ms
```

In the following, the rest of the test cases are omitted as they are quite
boring.
The important thing I wanted to show is how to set up the test framework.
If you are interested, you can check out the tests in the
`byterocket/TSOwnable-Huff` repo [here](https://github.com/byterocket/TSOwnable-Huff/blob/main/test/TSOwnable.t.sol).


## Removing `payable`

You may have noticed that the `MAIN()` macro is currently payable.
As there will be no functionality to withdraw ETH from the contract, let’s
revert in case someone wants to send some.

The `callvalue` opcode pushes the amount of ETH deposited for this execution
onto the stack. We can check via the `eq` opcode whether that amount is zero,
and if not jump to a revert statement.

Refactoring the entrypoint to being nonpayable leads to:

```c
#define macro MAIN() = takes (0) returns (0) {
    callvalue throw_error jumpi // Revert if ETH send

    0x00 calldataload 0xe0 shr // Get function signature

    // cast sig "owner()"
    dup1 0x8da5cb5b eq get_owner jumpi
    // cast sig "pendingOwner()"
    dup1 0xe30c3978 eq get_pending_owner jumpi

    throw_error:
        0x00 0x00 revert

    get_owner:
        OWNABLE_GET_OWNER()
    get_pending_owner:
        OWNABLE_GET_PENDING_OWNER()
}
```

Note that it is important to have the `throw_error` label before the “function”
macro calls, as a call with no fitting function signature would otherwise just
run through the dispatching and execute the `OWNABLE_GET_OWNER()` macro, or
whichever macro comes first.

You should think of the first statement following the dispatching as the
`fallback` of our contract. Whether you want to implement such a `fallback` or
not, it should be well defined.


## Implementing the `modifier` Macros

In order to access control the `acceptOwnership()` and `setPendingOwner()`
functions we need some kind of modifier. As nearly always, the Huff equivalent
is a macro.

The first macro, `ACCESS_ONLY_OWNER()`, reverts if the caller is not the
current owner, while the second macro, `ACCESS_ONLY_PENDING_OWNER()`, reverts
if the caller is not the current pending owner, respectively.

The macros can be implemented as follows:

```c
#define macro ACCESS_ONLY_OWNER() = takes(0) returns (0) {
    [OWNER_SLOT] sload caller eq is_owner jumpi
        0x00 0x00 revert
    is_owner:
}

#define macro ACCESS_ONLY_PENDING_OWNER() = takes (0) returns (0) {
    [PENDING_OWNER_SLOT] sload caller eq is_pending_owner jumpi
        0x00 0x00 revert
    is_pending_owner:
}
```

First, the corresponding storage slots are loaded from storage and pushed on
the stack. Afterwards, the caller address is pushed on the stack. The `eq`
opcode consumes both elements from the stack and pushes 0 on the stack if they
are equal, otherwise 1.

The following `jumpi` opcode jumps to the `is_owner` (or `is_pending_owner`)
label in case the first element on the stack is 0, i.e. if the word loaded
from storage equals the current caller address. If this is the case the macro
ends, enabling further execution.

If the caller is not authorized, the `jumpi` opcode does not jump and the
execution runs into the `revert` opcode.


## The State Mutating Functions

The next step is to implement the state mutating functions, `acceptOwnership()`
and `setPendingOwner()`.

Let’s start with the function dispatching.
Running `cast sig "acceptOwnership()"` returns `0x79ba5097`,
`cast sig "setPendingOwner()"` outputs `0xc42069ec`.

Adding these signatures to the `MAIN()` macro leads to:

```c
#define macro MAIN() = takes (0) returns (0) {
    callvalue throw_error jumpi // Revert if ETH send

    0x00 calldataload 0xe0 shr // Get function signature

    // cast sig "setPendingOwner(address)"
    dup1 0xc42069ec eq set_pending_owner jumpi
    // cast sig "acceptOwnership()"
    dup1 0x79ba5097 eq accept_ownership jumpi
    // cast sig "owner()"
    dup1 0x8da5cb5b eq get_owner jumpi
    // cast sig "pendingOwner()"
    dup1 0xe30c3978 eq get_pending_owner jumpi

    throw_error:
        0x00 0x00 revert

    set_pending_owner:
        OWNABLE_SET_PENDING_OWNER()
    accept_ownership:
        OWNABLE_ACCEPT_OWNERSHIP()
    get_owner:
        OWNABLE_GET_OWNER()
    get_pending_owner:
        OWNABLE_GET_PENDING_OWNER()
}
```

Heurika, the `MAIN()` macro is complete!


## Implemeting `acceptOwnership()`

The `acceptOwnership()` macro has to authorize the caller as being the current
pending owner, store the caller’s address as the new owner, clear the pending
owner by setting it to the zero address, and stop the execution.

There are no interesting new opcodes involved:

```c
#define macro OWNABLE_ACCEPT_OWNERSHIP() = takes (0) returns (0) {
    // Authorize caller via the "modifier" macro
    ACCESS_ONLY_PENDING_OWNER()

    // Store msg.sender as owner
    caller [OWNER_SLOT] sstore

    // Clear pending owner
    0x00 [PENDING_OWNER_SLOT] sstore

    stop
}
```


## Implementing `setPendingOwner()`

The `setPendingOwner()` macro is a bit more complicated.
We have to authorize the caller as being the current owner, read the address
argument from the calldata, and mask it to an address.
Additionally, the function should not accept the caller itself as pending owner,
store the new pending owner, and, lastly, stop the execution.

First, let’s see how to read an address argument:
`0x04 calldataload ADDRESS_MASK()`

As mentioned already, the first four bytes of the calldata are the function
signature. Therefore, we read one word, i.e. 32 bytes, from the calldata
starting at byte number 4 (`0x04 calldataload`).

What about the `MASK_ADDRESS()` macro?
Note that we do not have any guarantees that the caller indeed only send
20 bytes, i.e. an address, (+4 bytes of function signature) to the contract.
To make sure that we do not save any dirty bits into storage that could lead to
issues or even security vulnerabilities, we clear the remaining bytes by
setting them to zero.

The `MASK_ADDRESS()` macro looks like this:

```c
#define macro ADDRESS_MASK() = takes (1) returns (1) {
    0x000000000000000000000000ffffffffffffffffffffffffffffffffffffffff
    and
}
```

It expects one element on the stack (the data to mask), and leaves one
additional element on the stack (the data with the first 12 bytes set to zero).

Now we can be certain that no dirty bits will ever be stored to storage!

Continuing with the `setPendingOwner()` implementation, the macro has to revert
in case the owner tries to set the pending owner to itself. As the `eq` opcode
consumes the two elements it compares, we need to duplicate the argument read
from the calldata first.
(We could also read it from the calldata again, but that would be more costly)

The rest of the statements should be familiar already, leaving us with the
following implementation:

```c
#define macro OWNABLE_SET_PENDING_OWNER() = takes (0) returns (0) {
    ACCESS_ONLY_OWNER()

    // Read argument and mask to address
    0x04 calldataload ADDRESS_MASK()

    // Revert if address equals owner
    dup1 caller eq throw_error jumpi

    // Store address as pending owner
    [PENDING_OWNER_SLOT] sstore

    stop
    throw_error:
        0x00 0x00 revert
}
```

Congratulations for making it so far! The contract is nearly finished 🔥


## Defining the External Interface

Huff supports defining the external interface. There is not too much to say about
this:

```c
/// @notice Returns the current owner address.
#define function owner() view returns (address)

/// @notice Returns the current pending owner address.
#define function pendingOwner() view returns (address)

/// @notice Sets the pending owner address.
/// @dev Only callable by owner.
#define function setPendingOwner(address) nonpayable returns ()

/// @notice Accepts the ownership.
/// @dev Only callable by pending owner.
#define function acceptOwnership() nonpayable returns ()

/// @notice Emitted when new owner set.
#define event NewOwner(address,address)

/// @notice Emitted when new pending owner set.
#define event NewPendingOwner(address,address)
```


### Declaring and Emitting Events

One thing omitted so far is the emission of events. There are two events that
have to be send:

- The `event NewOwner(indexed address, indexed address)` will be emitted whenever
a new ownership is accepted
- The `event newPendingOwner(indexed address, indexed address)` indicates that a
new pending owner got set.

The first thing to do is creating the event signature, which is defined as the
`keccak256` hash of its Solidity signature.

Using the `cast` tool again:

- `cast keccak "NewOwner(address,address)"`
    ⇒ `0x70aea8d848e8a90fb7661b227dc522eb6395c3dac71b63cb59edd5c9899b2364`
- `cast keccak "NewPendingOwner(address,address)"`
    ⇒ `0xb3d55174552271a4f1aaf36b72f50381e892171636b3fb5447fe00e995e7a37b`

Next, check the `log` opcodes on [evm.codes](https://evm.codes).
Our events have have 3 topics:

1) The name of the event
2) The first address argument
3) The second address argument

Having 3 topics means we have to use the `log3` opcode. It consumes 5 elements
from the stack. The first 2 are needed to emit memory data, which our events do
not have.

The next elements are the topics, starting from 1 up to 3. Heading to the
Solidity docs indicates the first topic is the event signature, i.e. the hashes
we produced beforehand. The next two elements are the arguments.

## Putting all this together:

```c
// cast keccak "NewOwner(address,address)"
#define constant EVENT_NEW_OWNER
    = 0x70aea8d848e8a90fb7661b227dc522eb6395c3dac71b63cb59edd5c9899b2364
// cast keccak "NewPendingOwner(address,address)"
#define constant EVENT_NEW_PENDING_OWNER
    = 0xb3d55174552271a4f1aaf36b72f50381e892171636b3fb5447fe00e995e7a37b

// Inside macro OWNABLE_SET_PENDING_OWNER:
// Emit NewPendingOwner event
0x04 calldataload MASK_ADDRESS()    // [newPendingOwner]
[PENDING_OWNER_SLOT] sload          // [oldPendingOwner, newPendingOwner]
[EVENT_NEW_PENDING_OWNER] 0x00 0x00 // [0x00, 0x00, eventSignature, oldPendingOwner, newPendingOwner]
log3                                // []

// Inside macro OWNABLE_ACCEPT_OWNERSHIP:
// Emit NewOwner event
caller                              // [newOwner]
[OWNER_SLOT] sload                  // [oldOwner, newOwner]
[EVENT_NEW_OWNER] 0x00 0x00         // [0x00, 0x00, eventSignature, oldOwner, newOwner]
log3                                // []
```

## The Full Contract

🔥🔥🔥 for making it this far!

Your locally developed Huff contract should now look similar to this:

```c
/// @title TSOwnable
///
/// @dev An Ownable Implementation using Two-Step Transfer Pattern
///
/// @author merkleplant

// -----------------------------------------------------------------------------
// External Interface

/// @notice Returns the current owner address.
#define function owner() view returns (address)

/// @notice Returns the current pending owner address.
#define function pendingOwner() view returns (address)

/// @notice Sets the pending owner address.
/// @dev Only callable by owner.
#define function setPendingOwner(address) nonpayable returns ()

/// @notice Accepts the ownership.
/// @dev Only callable by pending owner.
#define function acceptOwnership() nonpayable returns ()

/// @notice Emitted when new owner set.
#define event NewOwner(address,address)

/// @notice Emitted when new pending owner set.
#define event NewPendingOwner(address,address)

// -----------------------------------------------------------------------------
// Event Signatures

// cast keccak "NewOwner(address,address)"
#define constant EVENT_NEW_OWNER
    = 0x70aea8d848e8a90fb7661b227dc522eb6395c3dac71b63cb59edd5c9899b2364
// cast keccak "NewPendingOwner(address,address)"
#define constant EVENT_NEW_PENDING_OWNER
    = 0xb3d55174552271a4f1aaf36b72f50381e892171636b3fb5447fe00e995e7a37b

// -----------------------------------------------------------------------------
// Storage

#define constant OWNER_SLOT = FREE_STORAGE_POINTER()
#define constant PENDING_OWNER_SLOT = FREE_STORAGE_POINTER()

// -----------------------------------------------------------------------------
// Constructor

#define macro CONSTRUCTOR() = takes (0) returns (0) {
    caller [OWNER_SLOT] sstore  // Store msg.sender as owner
}

// -----------------------------------------------------------------------------
// Helpers

#define macro ADDRESS_MASK() = takes (1) returns (1) {
	0x000000000000000000000000ffffffffffffffffffffffffffffffffffffffff
	and
}

// -----------------------------------------------------------------------------
// Access Handler

#define macro ACCESS_ONLY_OWNER() = takes(0) returns (0) {
    [OWNER_SLOT] sload caller eq is_owner jumpi
        0x00 0x00 revert
    is_owner:
}

#define macro ACCESS_ONLY_PENDING_OWNER() = takes (0) returns (0) {
    [PENDING_OWNER_SLOT] sload caller eq is_pending_owner jumpi
        0x00 0x00 revert
    is_pending_owner:
}

// -----------------------------------------------------------------------------
// Mutating Functions

#define macro OWNABLE_SET_PENDING_OWNER() = takes (0) returns (0) {
    ACCESS_ONLY_OWNER()

    // Read argument and mask to address
    0x04 calldataload ADDRESS_MASK()

    // Revert if address equals owner
    dup1 caller eq throw_error jumpi

    // Duplicate address on stack
    dup1

    // Emit NewPendingOwner event
    [PENDING_OWNER_SLOT] sload [EVENT_NEW_PENDING_OWNER] 0x00 0x00
    log3

    // Store address as pending owner
    [PENDING_OWNER_SLOT] sstore

    stop
    throw_error:
        0x00 0x00 revert
}

#define macro OWNABLE_ACCEPT_OWNERSHIP() = takes (0) returns (0) {
    ACCESS_ONLY_PENDING_OWNER()

    // Emit NewOwner event
    caller [OWNER_SLOT] sload [EVENT_NEW_OWNER] 0x00 0x00
    log3

    // Store msg.sender as owner
    caller [OWNER_SLOT] sstore

    // Clear pending owner
    0x00 [PENDING_OWNER_SLOT] sstore

    stop
}

// -----------------------------------------------------------------------------
// View Functions

#define macro OWNABLE_GET_OWNER() = takes (0) returns (0) {
    [OWNER_SLOT] sload
    0x00 mstore
    0x20 0x00 return
}

#define macro OWNABLE_GET_PENDING_OWNER() = takes (0) returns (0) {
    [PENDING_OWNER_SLOT] sload
    0x00 mstore
    0x20 0x00 return
}

// -----------------------------------------------------------------------------
// Function Dispatching

#define macro MAIN() = takes (0) returns (0) {
    callvalue throw_error jumpi // Revert if ETH send

    0x00 calldataload 0xe0 shr // Get function signature

    // cast sig "setPendingOwner(address)"
    dup1 0xc42069ec eq set_pending_owner jumpi
    // cast sig "acceptOwnership()"
    dup1 0x79ba5097 eq accept_ownership jumpi
    // cast sig "owner()"
    dup1 0x8da5cb5b eq get_owner jumpi
    // cast sig "pendingOwner()"
    dup1 0xe30c3978 eq get_pending_owner jumpi

    throw_error:
        0x00 0x00 revert

    set_pending_owner:
        OWNABLE_SET_PENDING_OWNER()
    accept_ownership:
        OWNABLE_ACCEPT_OWNERSHIP()
    get_owner:
        OWNABLE_GET_OWNER()
    get_pending_owner:
        OWNABLE_GET_PENDING_OWNER()
}
```
