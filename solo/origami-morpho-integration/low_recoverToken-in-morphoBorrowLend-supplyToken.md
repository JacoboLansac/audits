# `OrigamiMorphoBorrowAndLend::recoverToken()` should not allow owners to recover `surplusAfterSwap` of `_supplyToken`

Some `_supplyTokens` might stay in the `OrigamiMorphoBorrowAndLend` balance when `surplusAfterSwap < decoded.supplyCollateralSurplusThreshold` when calling `increaseLeverage`:


*OrigamiMorphoBorrowAndLend*:
```javascript
    function increaseLeverage(
        uint256 supplyAmount,
        uint256 borrowAmount,
        bytes memory swapData,
        uint256 supplyCollateralSurplusThreshold
    ) external override onlyPositionOwnerOrElevated {
        _supply(
            supplyAmount,
            getMarketParams(),
            abi.encode(IncreaseLeverageData(
                borrowAmount,
                swapData, 
                supplyCollateralSurplusThreshold
            ))
        );
    }

    /**
     * @notice Callback called when a supply of collateral occurs in Morpho.
     * @dev The callback is called only if data is not empty.
     * @param supplyAmount The amount of supplied collateral.
     * @param data Arbitrary data passed to the `supplyCollateral` function.
     */
    function onMorphoSupplyCollateral(uint256 supplyAmount, bytes calldata data) external override {
        if (msg.sender != address(morpho)) revert CommonEventsAndErrors.InvalidAccess();
        IncreaseLeverageData memory decoded = abi.decode(data, (IncreaseLeverageData));

        MorphoMarketParams memory marketParams = getMarketParams();

        // Perform the borrow
        _borrow(decoded.borrowAmount, address(this), marketParams);

        // Swap from [borrowToken] to [supplyToken]
        // The expected amount of [supplyToken] received after swapping from [borrowToken]
        // needs to at least cover the supplyAmount
        uint256 collateralReceived = swapper.execute(_borrowToken, decoded.borrowAmount, _supplyToken, decoded.swapData);
        if (collateralReceived < supplyAmount) {
            revert CommonEventsAndErrors.Slippage(supplyAmount, collateralReceived);
        }

        // There may be a suplus of `supplyToken` in this contract after the swap
        // If over the threshold, supply any surplus back in as collateral to morpho
        {
@>          uint256 surplusAfterSwap = _supplyToken.balanceOf(address(this)) - supplyAmount;
            if (surplusAfterSwap > decoded.supplyCollateralSurplusThreshold) {
                _supply(surplusAfterSwap, marketParams, "");
            }
        }
    }

```

These tokens still belong to vault depositors, so the owners should not be able to recover them:

```javascript
    function recoverToken(address token, address to, uint256 amount) external onlyElevatedAccess {
        emit CommonEventsAndErrors.TokenRecovered(to, token, amount);
        IERC20(token).safeTransfer(to, amount);
    }
```

## Impact: low

- It requires a compromised owner wallet that steals these tokens.
- Only the `surplusAfterSwap` is subject to this issue.

## Recommendation

```diff
    function recoverToken(address token, address to, uint256 amount) external onlyElevatedAccess {
+       if (token == _supplyToken) revert("Token not allowed");
        emit CommonEventsAndErrors.TokenRecovered(to, token, amount);
        IERC20(token).safeTransfer(to, amount);
    }
```
