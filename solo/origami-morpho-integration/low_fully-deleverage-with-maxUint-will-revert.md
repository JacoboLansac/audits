# Attempting to fully deleverage by providing `withdrawCollateralAmount=type(uint256).max` will revert when swapping the collateral for debt token 

When calling `decreaseLeverage()`, the `onMorphoRepay()` function is called, which withdraws collateral and swaps it for `_borrowToken`. However, the output from `_withdraw()` is ignored. 

```javascript
    function decreaseLeverage(
        uint256 repayAmount,
        uint256 withdrawCollateralAmount,
        bytes memory swapData,
        uint256 repaySurplusThreshold
    ) external override onlyPositionOwnerOrElevated returns (uint256 debtRepaidAmount) {
        return _repay(
            repayAmount, 
            getMarketParams(),
            abi.encode(DecreaseLeverageData(
                withdrawCollateralAmount,
                swapData,
                repaySurplusThreshold
            ))
        );
    }

    /**
     * @notice Callback called when a repayment occurs.
     * @dev The callback is called only if data is not empty.
     * @param repayAmount The amount of repaid assets.
     * @param data Arbitrary data passed to the `repay` function.
     */
    function onMorphoRepay(uint256 repayAmount, bytes calldata data) external {
        if (msg.sender != address(morpho)) revert CommonEventsAndErrors.InvalidAccess();
        DecreaseLeverageData memory decoded = abi.decode(data, (DecreaseLeverageData));

        MorphoMarketParams memory marketParams = getMarketParams();

        // Withdraw collateral
@>      _withdraw(decoded.withdrawCollateralAmount, address(this), marketParams);
        
        // Swap from [supplyToken] to [borrowToken]
        // The expected amount of [borrowToken] received after swapping from [supplyToken]
        // needs to at least cover the repayAmount
        uint256 borrowTokenReceived = swapper.execute(_supplyToken, decoded.withdrawCollateralAmount, _borrowToken, decoded.swapData);
        if (borrowTokenReceived < repayAmount) {
            revert CommonEventsAndErrors.Slippage(repayAmount, borrowTokenReceived);
        }

        // There may be a suplus of `borrowToken` in this contract after the swap
        // If over the threshold, repay any surplus back to morpho
        {
            uint256 surplusAfterSwap = _borrowToken.balanceOf(address(this)) - repayAmount;
            if (surplusAfterSwap > decoded.repaySurplusThreshold) {
                _repay(surplusAfterSwap, marketParams, "");
            }
        }
    }

```

The output from `_withdraw()` would generally be exactly `withdrawCollateralAmount` except in the case when `withdrawCollateralAmount = type(uint256).max`, which could only happen when the admins want to fully deleverage and withdraw all collateral at once. In this circumstance, the `swapper.execute()` will revert, as it try to swap `type(unit256).max` instead of the actual withdrawn collateral. 

## Impact: low

Fully decreasing leverage and withdrawing all collateral by calling `decreaseLeverage()` with `withdrawCollateralAmount = type(uint256).max` will revert with a `not enough balance error`, which might not be very informative. If the team wants to fully deleverage due to a hack or a depeg or a black swan event, having clear error messages can improve the developer's mental health haha. 

## Recommendation

- Either include a check in `decreaseLeverage()` that disallows `withdrawCollateralAmount = type(uint256).max`
- Read the output from `_withdraw()` in `onMorphoRepay`, and only swap that:

```diff
    function onMorphoRepay(uint256 repayAmount, bytes calldata data) external {
        if (msg.sender != address(morpho)) revert CommonEventsAndErrors.InvalidAccess();
        DecreaseLeverageData memory decoded = abi.decode(data, (DecreaseLeverageData));

        MorphoMarketParams memory marketParams = getMarketParams();

        // Withdraw collateral
-       _withdraw(decoded.withdrawCollateralAmount, address(this), marketParams);
+       uint256 withdrawn = _withdraw(decoded.withdrawCollateralAmount, address(this), marketParams);
        
        // Swap from [supplyToken] to [borrowToken]
        // The expected amount of [borrowToken] received after swapping from [supplyToken]
        // needs to at least cover the repayAmount
-       uint256 borrowTokenReceived = swapper.execute(_supplyToken, decoded.withdrawCollateralAmount, _borrowToken, decoded.swapData);
+       uint256 borrowTokenReceived = swapper.execute(_supplyToken, withdrawn, _borrowToken, decoded.swapData);
        if (borrowTokenReceived < repayAmount) {
            revert CommonEventsAndErrors.Slippage(repayAmount, borrowTokenReceived);
        }

        // There may be a suplus of `borrowToken` in this contract after the swap
        // If over the threshold, repay any surplus back to morpho
        {
            uint256 surplusAfterSwap = _borrowToken.balanceOf(address(this)) - repayAmount;
            if (surplusAfterSwap > decoded.repaySurplusThreshold) {
                _repay(surplusAfterSwap, marketParams, "");
            }
        }
    }

```
