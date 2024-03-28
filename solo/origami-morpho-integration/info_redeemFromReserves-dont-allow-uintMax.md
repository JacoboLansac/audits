# Don't allow max withdraw in `_redeemFromReserves`

```javascript
    function _redeemFromReserves(uint256 reservesAmount, address toToken, address recipient) internal override returns (uint256 toTokenAmount) {
        if (toToken == address(_reserveToken)) {
            toTokenAmount = reservesAmount;
            // No need to check the return amount, the amount passed in can never be type(uint256).max, so this will
            // be the exact `amount`
            borrowLend.withdraw(reservesAmount, recipient);
        } else {
            revert CommonEventsAndErrors.InvalidToken(toToken);
        }
    }
```

As this is a user-exposed function (through `lovToken.exitToToken()`), I would propose to put a hard restriction here so that reservesAmount can't be `type(uint256).max`. It is a nearly impossible scenario as the `AbstractLovTokenManager` has multiple mechanisms that would reduce `quoteData.investmentTokenAmount` to a lower value (the fees for instance), shares conversion with rounding, etc.

However, in the very-unlikely-case that an attacker finds a way to sneak `type(uint256).max` into this function, he would withdraw from morpho the entire supplied collateral.

So, I can't count it as a finding, as I have not a valid exploit, but for the sake of security, I would include that requirement

## Team response

Revert if reservesAmount doesn't match the withdraw amount. Good
