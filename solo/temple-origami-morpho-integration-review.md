# Integrating LOV tokens with the [sUSDe/DAI] Morpho Blue market

This was an extension to the already reviewed LOV tokens, but integrating them with Morpho Blue markets. 
Most of the architecture was the same, and the code quality was as good as the previous codebases. Only low-severity issues were found.

## Scope
Mainly three new contracts were added to the scope. Their core was similar to the existing LOV contracts, but some adaptations were required to integrate with Morpho Blue markets. The main contracts to review:

Dates: from `2024-03-28` to `2024-04-04`.

| Contract  | nSLOC |
| --- | --- |
| OrigamiLovTokenMorphoManager.sol | 220 | 
| OrigamiMorphoBorrowAndLend.sol | 272 | 
| OrigamiErc4626AndDexAggregatorSwapper.sol | 51 |
| DexAggregator.sol | 31 | 

## Findings

### Low
- [Low - Safe LTV should be strictly lower than morpho's](./origami-morpho-integration/low_maxSafeTvl-strictly-lower-than-morphos.md)
- [Low - Supply cap not checked in MintableToken](./origami-morpho-integration/low_maxSupply-not-chekced-in-MintableToken.md)
- [Low - Recover token restrictions in borro/supply tokens](./origami-morpho-integration/low_recoverToken-in-morphoBorrowLend-supplyToken-borrowToken.md)

### Info
- [Info - maxSafeLTV not checked in constructor](./origami-morpho-integration/info_maxSafeTvl-not-checked-in-constructor.md)
- [Info - extra safety in redeemFromReserves](./origami-morpho-integration/info_redeemFromReserves-dont-allow-uintMax.md)
- [Info - Reflections about userALfloor and maxRedeemFromReserves](./origami-morpho-integration/info_maxExit-maxRedeemFromReserves-borrowLend-userALfloor.md)
