# `Origami::inverseSubtractBps()` should round down in `OrigamiLendingSupplyManager::maxExit()` and `OrigamiAbstractLovTokenManager::maxExit()`

The function `inverseSubtractBps()` is the inverse of `subtractBps()`, and therefore estimates the amount that would result in `remainderAmount` after applying `subtractBps()`. The function accepts a rounding parameter to round up or down:

```javascript   
    /**
     * @notice Reverse the fractional amount of an input.
     * eg: For 3333 BPS (33.3%) and the remainder=400, the result is 600
     */
    function inverseSubtractBps(
        uint256 remainderAmount, 
        uint256 basisPoints,
@>      Rounding roundingMode
    ) internal pure returns (uint256 result) {
        if (basisPoints == 0) return remainderAmount; // gas shortcut for 0
        if (basisPoints >= BASIS_POINTS_DIVISOR) revert CommonEventsAndErrors.InvalidParam();

        uint256 denominatorBps;
        unchecked {
            denominatorBps = BASIS_POINTS_DIVISOR - basisPoints;
        }
        result = mulDiv(
            remainderAmount,
            BASIS_POINTS_DIVISOR, 
            denominatorBps, 
@>          roundingMode
        );
    }
}
```

The function is used in `OrigamiLendingSupplyManager::maxExit()` and `OrigamiAbstractLovTokenManager::maxExit()`. These two functions estimate the max amount of shares that they will be able to redeem 

*OrigamiLendingSupplyManager.sol*:
```javascript
    function maxExit(address toToken) external override view returns (uint256 amount) {
        if (toToken == address(asset)) {
            // Capacity is bound by the available remaining in the circuit breaker (18dp)
@>          amount = circuitBreakerProxy.available(address(oToken), address(this));

            // And also by what's available in the lendingClerk.
            // Convert from the underlying asset to the oToken decimals
            uint256 amountFromLendingClerk = lendingClerk.totalAvailableToWithdraw().scaleUp(_assetScalar);
            if (amountFromLendingClerk < amount) {
                amount = amountFromLendingClerk;
            }

            // Since exit fees are taken when exiting,
            // reverse out the fees
@>          amount = amount.inverseSubtractBps(exitFeeBps, OrigamiMath.Rounding.ROUND_UP);
        }
    }
```

*OrigamiAbstractLovTokenManager.sol*:
```javascript
    function maxExit(address toToken) external override view returns (uint256 sharesAmount) {
        // Calculate the max reserves that can be removed before the A/L floor is hit
        // Round up for the minimum reserves
        Cache memory cache = populateCache(IOrigamiOracle.PriceType.SPOT_PRICE);

        uint256 _minReserves = cache.liabilities.mulDiv(
            userALRange.floor, 
            PRECISION, 
            OrigamiMath.Rounding.ROUND_UP
        );

        // Only check the underlying implementation if there's capacity to remove reserves
        if (cache.assets > _minReserves) {
            // Calculate the max number of lovToken shares which can be exited given the A/L 
            // floor on reserves
            uint256 _amountFromAvailableCapacity;
            unchecked {
                _amountFromAvailableCapacity = cache.assets - _minReserves;
            }

            // Check the underlying implementation's max reserves that can be redeemed
            uint256 _underlyingAmount = _maxRedeemFromReserves(toToken);

            // Use the minimum of both the underlying implementation max and
            // the capacity based on the A/L floor
            if (_underlyingAmount < _amountFromAvailableCapacity) {
                _amountFromAvailableCapacity = _underlyingAmount;
            }

            // Convert reserves to lovToken shares
            sharesAmount = _reservesToShares(cache, _amountFromAvailableCapacity);

            // Since exit fees are taken when exiting (so these reserves aren't actually redeemed),
            // reverse out the fees
@>          sharesAmount = sharesAmount.inverseSubtractBps(_dynamicExitFeeBps(), OrigamiMath.Rounding.ROUND_UP);

            // Finally use the min of the derived amount and the lovToken total supply
            if (sharesAmount > cache.totalSupply) {
                sharesAmount = cache.totalSupply;
            }
        }
    }
```

*The Issue:*

When a user reads `maxExit`, the expected value would be *the max amount that could be exited for sure, without a risk of any revert*.
Rounding up sharesAmount goes against this. In the very edge case that an amount of shares X is fine, but `X+1` reverts, the upward rounding could increase the result from `X` to `X+1`. 

**Impact: low** 

Users might get reverts when exiting with `exitToToken()` if using the output read from `maxExit()`. 

It is important to acknowledge that `exitToToken()` rounds down when subtracting the fees. This would seem it would compensate the rounding UP from the `maxExit()` function. However, if the limiting factor is the preCheck, the rounding up could hit the circuit breaker limit, which is checked in `exitToToken()` before the subtraction of the fees. 

```javascript
    function exitToToken(
        address /*account*/,
        IOrigamiInvestment.ExitQuoteData calldata quoteData,
        address recipient
    ) external override onlyOToken returns (
        uint256 toTokenAmount,
        uint256 toBurnAmount
    ) {
        if (_paused.exitsPaused) revert CommonEventsAndErrors.IsPaused();
        if (quoteData.toToken != address(asset)) revert CommonEventsAndErrors.InvalidToken(quoteData.toToken);

        // Ensure that this exit doesn't break the circuit breaker limits for oToken
        circuitBreakerProxy.preCheck(
            address(oToken),
            quoteData.investmentTokenAmount
        );

        // Exit fees are taken from the sender's oToken amount.
        (uint256 nonFees, uint256 fees) = quoteData.investmentTokenAmount.splitSubtractBps(
            exitFeeBps, 
@>          OrigamiMath.Rounding.ROUND_DOWN
        );
        toBurnAmount = nonFees;

        // Collect the fees
        if (fees != 0) {
            IERC20Metadata(oToken).safeTransfer(feeCollector, fees);
        }

        if (nonFees != 0) {
            // This scaleDown intentionally rounds down (so it's not in the user's benefit)
            toTokenAmount = nonFees.scaleDown(_assetScalar, OrigamiMath.Rounding.ROUND_DOWN);
            if (toTokenAmount != 0) {
                lendingClerk.withdraw(toTokenAmount, recipient);
            }
        }
    }
```

**Recommendation**

Even though the likelihood is low, I would still suggest to round down in `maxExit()` for correctness. The max-exit value should be calculated in a "conservative" manner.

*OrigamiLendingSupplyManager.sol*:
```diff
    function maxExit(address toToken) external override view returns (uint256 amount) {
        if (toToken == address(asset)) {
            // Capacity is bound by the available remaining in the circuit breaker (18dp)
            amount = circuitBreakerProxy.available(address(oToken), address(this));

            // And also by what's available in the lendingClerk.
            // Convert from the underlying asset to the oToken decimals
            uint256 amountFromLendingClerk = lendingClerk.totalAvailableToWithdraw().scaleUp(_assetScalar);
            if (amountFromLendingClerk < amount) {
                amount = amountFromLendingClerk;
            }

            // Since exit fees are taken when exiting,
            // reverse out the fees
-           amount = amount.inverseSubtractBps(exitFeeBps, OrigamiMath.Rounding.ROUND_UP);
+           amount = amount.inverseSubtractBps(exitFeeBps, OrigamiMath.Rounding.ROUND_DOWN);
        }
    }
```

*OrigamiAbstractLovTokenManager.sol*:
```diff
    function maxExit(address toToken) external override view returns (uint256 sharesAmount) {
        // Calculate the max reserves which can be removed before the A/L floor is hit
        // Round up for the minimum reserves
        Cache memory cache = populateCache(IOrigamiOracle.PriceType.SPOT_PRICE);

        uint256 _minReserves = cache.liabilities.mulDiv(
            userALRange.floor, 
            PRECISION, 
            OrigamiMath.Rounding.ROUND_UP
        );

        // Only check the underlying implementation if there's capacity to remove reserves
        if (cache.assets > _minReserves) {
            // Calculate the max number of lovToken shares which can be exited given the A/L 
            // floor on reserves
            uint256 _amountFromAvailableCapacity;
            unchecked {
                _amountFromAvailableCapacity = cache.assets - _minReserves;
            }

            // Check the underlying implementation's max reserves that can be redeemed
            uint256 _underlyingAmount = _maxRedeemFromReserves(toToken);

            // Use the minimum of both the underlying implementation max and
            // the capacity based on the A/L floor
            if (_underlyingAmount < _amountFromAvailableCapacity) {
                _amountFromAvailableCapacity = _underlyingAmount;
            }

            // Convert reserves to lovToken shares
            sharesAmount = _reservesToShares(cache, _amountFromAvailableCapacity);

            // Since exit fees are taken when exiting (so these reserves aren't actually redeemed),
            // reverse out the fees
-           sharesAmount = sharesAmount.inverseSubtractBps(_dynamicExitFeeBps(), OrigamiMath.Rounding.ROUND_UP);
+           sharesAmount = sharesAmount.inverseSubtractBps(_dynamicExitFeeBps(), OrigamiMath.Rounding.ROUND_DOWN);

            // Finally use the min of the derived amount and the lovToken total supply
            if (sharesAmount > cache.totalSupply) {
                sharesAmount = cache.totalSupply;
            }
        }
    }
```
