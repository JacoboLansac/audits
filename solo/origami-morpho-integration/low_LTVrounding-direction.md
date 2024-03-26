In `OrigamiMorphoBorrowAndLend::debtAccountData()`, the LTV ratio is calculated rounding down:

```javascript
    function debtAccountData() external override view returns (

        // ...

        maxBorrow = _collateralInBorrowTerms.mulDiv(
            morphoMarketLltv, // loan-to-value == 1/AL // 1e18 units
            1e18,
            OrigamiMath.Rounding.ROUND_DOWN
        );
        
        if (borrowed == 0) {
            healthFactor = type(uint256).max;
        } else {
            currentLtv = borrowed.mulDiv(
                1e18,
                _collateralInBorrowTerms,
@>              OrigamiMath.Rounding.ROUND_DOWN // @audit-issue round up to be conservative (higher LTV -> higher risk)
            );
            healthFactor = (borrowed == 0) 
                ? type(uint256).max 
                : maxBorrow.mulDiv(1e18, borrowed, OrigamiMath.Rounding.ROUND_DOWN);
        }
    }
```

The LTV (Loan-to-Value) is calculated as (Liabilities / assets). The higher the LTV, the higher the exposure (and the risk). 

Therefore it would be more conservative to round the LTV ratio **up**, to avoid a situation in which the `debtAccountData()` shows a rounded-down-healthy LTV, while in reality it is hitting the LTV limit. 

**Recommendation**

```diff
            currentLtv = borrowed.mulDiv(
                1e18,
                _collateralInBorrowTerms,
-               OrigamiMath.Rounding.ROUND_DOWN
+               OrigamiMath.Rounding.ROUND_UP
            );
```
