# Integrating LOV tokens with the [sUSDe/DAI] Morpho Blue market

This was an extension to the already reviewed LOV tokens, but integrating them with Morpho Blue markets. 
Most of the architecture was the same, and the code quality was as good as the previous codebases. Only low severity issues were found.

## Low
- [Low - Safe LTV should be strictly lower than morpho's](./origami-morpho-integration/low_maxSafeTvl-strictly-lower-than-morphos.md)
- [Low - Supply cap not checked in MintableToken](./origami-morpho-integration/low_maxSupply-not-chekced-in-MintableToken.md)
- [Low - Recover token restrictions in borro/supply tokens](./origami-morpho-integration/low_recoverToken-in-morphoBorrowLend-supplyToken-borrowToken.md)

## Info
- [Info - maxSafeLTV not checked in constructor](./origami-morpho-integration/info_maxSafeTvl-not-checked-in-constructor.md)
- [Info - extra safety in redeemFromReserves](./origami-morpho-integration/info_redeemFromReserves-dont-allow-uintMax.md)
- [Info - Reflections about userALfloor and maxRedeemFromReserves](./origami-morpho-integration/info_maxExit-maxRedeemFromReserves-borrowLend-userALfloor.md)
