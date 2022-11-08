+++
author = "merkleplant"
title = "Ampleforth is Hayek Money"
date = "2022-05-30"
+++

This article introduces Ferdinando M. Ametrano’s concept of _Hayek Money_ as
defined in his paper [_Hayek Money: The Cryptocurrency Price Stability Solution_](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2425270)
and a follow-up argument that, following this definition, Ampleforth is
Hayek Money.
<!--more-->


## Introduction

The paper starts with the observation that _money is a social relation instrument […] to increase cooperation_.

The author argues that money has to fulfill three interdependent functions:

1. **Medium of exchange** - It can be reliably swapped for something else
2. **Unit of account** - Other goods, services, and assets are priced in terms
   of money
3. **Store of value** - It can be reliably saved, stored, and retrieved while
   retaining its usefulness over time

Despite money’s special role as being the unit the value of other goods are
measured against, money itself **is a good**.
As such the **value of money is governed by supply and demand.**

This observation becomes obvious when the inverse of a typical question such as
“_how many apples can I buy with a unit of currency?_” is asked, i.e.
“_how many units of currency can I buy with an apple?_”.

Following, the author conducts that “_good money should provide stable prices to best perform its role as unit of account_”.
It is argued, that such a money would enable well-informed economic decisions
by households and firms, following the confidence that the value of one
currency unit will be stable over time.

In a price system, the value of goods is measured relative to the value of
money.
An increase in the price level of goods and services signals a decrease in the
value of money - _inflation_.
The opposite - _deflation_ - would be a decrease in price levels and,
therefore, an increase in the value of money.

However, any such change in the value of a unit of money leads to
**injustice to debtors or lenders** as the money’s value per unit changes
during the lifetime of their contracts.

As the demand for money cannot be controlled, the only possibility to
ensure price stability is to manage its supply - the _monetary policy_.


## Hayek’s Theory of Concurrent Private Currencies

The Nobel Prize-winning economist Friedrich Hayek analyzed the theory of
concurrent currencies in his book [Denationalisation of Money](https://nakamotoinstitute.org/static/docs/denationalisation.pdf).

After stating that competing currencies have never been seriously examined and
the governmental monopoly of the provision of money not being questioned enough,
Hayek states that an open competition of private concerns supplying different
currencies would enable the possibility to control the quantity of money so that
its value will behave in a desired manner, and
“_that it will, for this reason, retain its acceptability and its value_".

Hayek imagined a world in which people choose freely which of the concurrently
circulating currencies they use.

The success of private currencies sees Hayek in that “_there is no reason to doubt that private enterprise, whose business depended on succeeding in the attempt, could keep stable the value of money it issued_”.


## Hayek Money

Ametrano defines Hayek Money as an elastic supply money that keeps the
**purchasing power of the currency unit constant to a price index**.
Furthermore, the supply adjustments need to be performed in a
**fully automatic and non-discretionary** way with no need for a central
authority.

However, adjusting the monetary base can lead to a Cantillon Effect, i.e. the
supply adjustment could lead to unfair wealth redistributions.
This leads Ametrano to the idea of adjusting the monetary base by distributing
the supply delta “_pro-quote to every digital wallet_”.

Such an adjustment would have a “_neutral impact on the overall wallet wealth, as it does not introduce any arbitrary distortion into the intrinsic value dynamics of the wallet_”.

Currency volatility will then not be discovered by price volatility anymore but
by supply volatility.

It should be noted, again, that “_supply and demand dictates the value of money relative to other goods: nothing can be done to escape the unavoidable debasement associated with decreasing demand for money_”.

Hayek Money does not eliminate volatility, it shifts volatility from price to supply.


## Technicalities of Hayek Monies

A reader might ask now how often the monetary base adjustment should be
performed. After all, supply and demand are constantly changing and, therefore,
to achieve zero price volatility would mean a constant adjustment to the
monetary base too.

Ametrano suggests _rebasing_ the currency amount at least daily.
Furthermore, he suggests experimenting with some rebasing reaction lag.
Such a lag could be 30 days, implying that each day only 1/30 of the needed
money stock adjustment is performed.


## Ampleforth - A Hayek Money

The Ampleforth [official documentation](https://docs.ampleforth.org/learn/about-the-ampleforth-protocol)
states that
“_the Ampleforth protocol is a set of instructions […] that produces a decentralized unit of account called AMPL_”.

The price index Ampleforth uses to measure the purchasing power is the
CPI-adjusted 2019 US dollar.
Once every 24 hours the monetary base of the AMPL token can be adjusted by
executing a public callable function in Ampleforth’s monetary policy contract.

This rebase operation calculates the difference between AMPL’s market price and
price target and performs the resulting supply adjustment atomically,
and non-dilutive, to every wallet.

As demanded by Ametrano, the supply adjustments are fully automatic and non-discretionary.
A supply adjustment has neutral impact on the overall wallet wealth as the
intrinsic value dynamics of the wallets do not change.

It is, again, important to mention that the Ampleforth protocol does not
eliminate volatility. AMPL is **not a stablecoin**.

However, the AMPL token is quite successful in **keeping the purchasing power of one AMPL stable**,
as illustrated in AMPL’s market price since inception:

![price](/post/ampleforth-is-hayek-money/price.avif)

The Ampleforth protocol achieves this by shifting the price volatility to the
supply volatility.

Here is a visualization of AMPL’s supply since inception:

![supply](/post/ampleforth-is-hayek-money/supply.avif)


## Conclusion

This post summarized Ametrano’s idea of Hayek Money and Hayek’s view of why
concurrent private currencies are favorable.

Furthermore, this post illustrated that the Ampleforth protocol successfully
operates a Hayek Money, AMPL. To my knowledge, AMPL is the first Hayek Money
in existence.


### Appendix A: AMPL’s Monetary Policy Visualized

The Ampleforth protocol is a set of smart contracts on Ethereum.
Smart contracts are programs, mostly written in higher-level programming
languages, that encode rules that are then uploaded to a blockchain.

At some point, every program that gets executed is represented in so-called
_bytecode_. This bytecode is what the “machine really sees”.

Anyway, here is a visualization of AMPL’s monetary policy, showing each byte
contained in the contract:

![bytecode](/post/ampleforth-is-hayek-money/bytecode.avif)

The bytecode was visualized using [DanielVF](https://twitter.com/danielvf)'s
[evm-contract-draw tool](https://github.com/DanielVF/evm-contract-draw).


### Appendix B: Some Important Monetary Terms

- _debasement_ - The practice of lowering the intrinsic value while maintaining
  the face value
- _representative money_ - Money that represents a claim on a commodity money
- _fractional receipt money_ - Money in which the amount of issued
  representative money is greater than the commodity money reserve backing it
- _seigniorage_ - Being the profit made by issuing money, especially the
  difference between its face value and its production cost
- _currency_ - An instance of money that’s actually used as a medium of exchange
- _fiat money_ - Money deriving its value only from government regulation or
  law by defining it’s valid for meeting financial obligations
