# Overview

A review of [this Pull-Request](https://github.com/TempleDAO/origami/pull/940/) was requested, with the following two contracts in scope:

- `apps/protocol/contracts/common/swappers/OrigamiDexAggregatorSwapper.sol`
- `apps/protocol/contracts/common/swappers/OrigamiErc4626AndDexAggregatorSwapper.sol`

Both contracts suffer from the same issue (lack of input validation on the `router` address) so here I'll describe the issue on the `OrigamiDexAggregatorSwapper` only.

The changes aimed at giving more flexibility on which router to use when swapping tokens. 
Until now, the swapper had the router as an immutable address, but now, the router can be passed by the caller as an input argument, allowing the bot to optimize for the best swap available. 

## Medium
- [M1] - [Lack of input validation on OrigamiDexAggregatorSwapper.execute() allows an attacker to steal the token balance of contracts that granted approvals to the swapper contract](origami/flexi-swapper-exploit-lack-of-input-validation.md)
