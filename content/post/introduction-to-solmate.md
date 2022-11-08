+++
author = "merkleplant"
title = "Introduction to solmate"
date = "2022-11-08"
+++

The `solmate` contracts from [t11s](https://twitter.com/transmissions11) are
"_not designed with user safety in mind_". Implicit invariants are expected
to be followed, and it's easy to shoot yourself in the foot.

Therefore, I thought it's a good idea to introduce some of the contracts,
their footguns, and cross-check them with the OpenZeppelin library.
<!--more-->


## EVM Expeditions at [byterocket](https://byterocket.com)

Recently, we at [byterocket](https://byterocket.com) started to share internal
learning sessions under the term [_EVM Expeditions_](https://www.youtube.com/playlist?list=PLxO4n4c2l1TIukYArtqectoOoCv3ZF0A7).

If you ever wanted to konw how to create an overflow in `solmate`'s ERC20 token,
or finally want to understand the assembly in the `SafeTransferLib`, this
presentation is for you.

- [Video](https://youtu.be/IXzY6yUo1t4?list=PLxO4n4c2l1TIukYArtqectoOoCv3ZF0A7)
- [GitHub Repo](https://github.com/byterocket/about-solmate)

### Approved by the Author ✅

![t11s-tweet](/post/introduction-to-solmate/t11s-tweet.png)
