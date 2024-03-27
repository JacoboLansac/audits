# `OrigamiAbstractLovTokenManager::investQuote()` ignores the lovToken max supply cap. 

`investQuote()` would not reflect if `fromTokenAmount` was to hit the supply cap. 


```javascript
    function investQuote(
        uint256 fromTokenAmount, 
        address fromToken,
        uint256 maxSlippageBps,
        uint256 deadline
    ) external virtual override view returns (
        IOrigamiInvestment.InvestQuoteData memory quoteData, 
        uint256[] memory investFeeBps
    ) {
        if (fromTokenAmount == 0) revert CommonEventsAndErrors.ExpectedNonZero();
        Cache memory cache = populateCache(IOrigamiOracle.PriceType.SPOT_PRICE);
        uint256 _newReservesAmount = _previewDepositIntoReserves(fromToken, fromTokenAmount);
        // The number of shares is calculated based off this `_newReservesAmount`
        // However not all of these shares are minted and given to the user -- the deposit fee is removed
        uint256 _investmentAmount = _reservesToShares(cache, _newReservesAmount);
        uint256 _depositFeeRate = _dynamicDepositFeeBps();
        _investmentAmount = _investmentAmount.subtractBps(_depositFeeRate, OrigamiMath.Rounding.ROUND_DOWN);
        quoteData.fromToken = fromToken;
        quoteData.fromTokenAmount = fromTokenAmount;
        quoteData.maxSlippageBps = maxSlippageBps;
        quoteData.deadline = deadline;
        quoteData.expectedInvestmentAmount = _investmentAmount;
        quoteData.minInvestmentAmount = _investmentAmount.subtractBps(maxSlippageBps, OrigamiMath.Rounding.ROUND_UP);
        // quoteData.underlyingInvestmentQuoteData remains as bytes(0)
        investFeeBps = new uint256[](1);
        investFeeBps[0] = _depositFeeRate;
    }
```

**Impact: low**

`investQuote()` will return quotes that will revert when calling `investWithToken()`, if such investment hits the max supply cap. 

**Recommendation**

Perhaps reuse the function `_reservesCapacityFromTotalSupply()` to make sure the limit is not hit, or to reflect it in the output from `investQuote()`

Alternatively, `_previewDepositIntoReserves()` (which is already used in `investQuote()`) could take into account the supply limit, and then `investQuote()` could revert if slippage limits were not respected. But then I guess `_maxDepositIntoReserves()` should also reflect that supply limit, which I think is not how you intended to use this function (because `borrowLend.availableToSupply()` returns `type(unit).max`. 

## Invalidation reason

Invalid because `investQuote` and `exitQuote` doesn't take the max into consideration at all, but it is OK. 
