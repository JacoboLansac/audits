# Morpho liquidations threshold

When setting the `maxSafeLtv`, a strict greater-than is used in the input validation allowing the Origami's `_maxSafeLtv` to be *equal* to morpho's `morphoMarketLltv`. However, the docstring suggests that the intention is to have a *safer* LTV limit than the one in morpho. This made me think that having a safer LTV would mean not allowing having the same limit, but strictly lower.  

*OrigamiMorphoBorrowAndLend.sol*
```javascript
    /**
     * @notice Set the max LTV we will allow when borrowing or withdrawing collateral.
     * @dev The morpho LTV is the liquidation LTV only, we don't want to allow up to that limit
     * so we set a more restrictive 'safe' LTV'
     */
    function setMaxSafeLtv(uint256 _maxSafeLtv) external override onlyElevatedAccess {
@>      if (_maxSafeLtv > morphoMarketLltv) revert CommonEventsAndErrors.InvalidParam();
        maxSafeLtv = _maxSafeLtv;
        emit MaxSafeLtvSet(_maxSafeLtv);
    }
```

## Impact: low

## Recommendation

*OrigamiMorphoBorrowAndLend.sol*
```diff
    /**
     * @notice Set the max LTV we will allow when borrowing or withdrawing collateral.
     * @dev The morpho LTV is the liquidation LTV only, we don't want to allow up to that limit
     * so we set a more restrictive 'safe' LTV'
     */
    function setMaxSafeLtv(uint256 _maxSafeLtv) external override onlyElevatedAccess {
-       if (_maxSafeLtv > morphoMarketLltv) revert CommonEventsAndErrors.InvalidParam();
+       if (_maxSafeLtv >= morphoMarketLltv) revert CommonEventsAndErrors.InvalidParam();
        maxSafeLtv = _maxSafeLtv;
        emit MaxSafeLtvSet(_maxSafeLtv);
    }
```

## Further thoughts

Furthermore, if we look at the liquidation logic in the morpho market:

*Morpho.sol*
```javascript
    function liquidate(
        MarketParams memory marketParams,
        address borrower,
        uint256 seizedAssets,
        uint256 repaidShares,
        bytes calldata data
    ) external returns (uint256, uint256) {
        Id id = marketParams.id();
        require(market[id].lastUpdate != 0, ErrorsLib.MARKET_NOT_CREATED);
        require(UtilsLib.exactlyOneZero(seizedAssets, repaidShares), ErrorsLib.INCONSISTENT_INPUT);

        _accrueInterest(marketParams, id);

        {
            uint256 collateralPrice = IOracle(marketParams.oracle).price();

@>          require(!_isHealthy(marketParams, id, borrower, collateralPrice), ErrorsLib.HEALTHY_POSITION);

            // The liquidation incentive factor is min(maxLiquidationIncentiveFactor, 1/(1 - cursor*(1 - lltv))).
            uint256 liquidationIncentiveFactor = UtilsLib.min(
                MAX_LIQUIDATION_INCENTIVE_FACTOR,
                WAD.wDivDown(WAD - LIQUIDATION_CURSOR.wMulDown(WAD - marketParams.lltv))
            );

            // ..
        }
    }

    // ...

    /// @dev Returns whether the position of `borrower` in the given market `marketParams` is healthy.
    /// @dev Assumes that the inputs `marketParams` and `id` match.
    function _isHealthy(MarketParams memory marketParams, Id id, address borrower) internal view returns (bool) {
        if (position[id][borrower].borrowShares == 0) return true;

        uint256 collateralPrice = IOracle(marketParams.oracle).price();

@>      return _isHealthy(marketParams, id, borrower, collateralPrice);
    }

    /// @dev Returns whether the position of `borrower` in the given market `marketParams` with the given
    /// `collateralPrice` is healthy.
    /// @dev Assumes that the inputs `marketParams` and `id` match.
@>  /// @dev Rounds in favor of the protocol, so one might not be able to borrow exactly `maxBorrow` but one unit less.
    function _isHealthy(MarketParams memory marketParams, Id id, address borrower, uint256 collateralPrice)
        internal
        view
        returns (bool)
    {
        uint256 borrowed = uint256(position[id][borrower].borrowShares).toAssetsUp(
            market[id].totalBorrowAssets, market[id].totalBorrowShares
        );
@>      uint256 maxBorrow = uint256(position[id][borrower].collateral).mulDivDown(collateralPrice, ORACLE_PRICE_SCALE)
            .wMulDown(marketParams.lltv);

        return maxBorrow >= borrowed;
    }


```

We see in the comment in the `_isHealthy()` that the function *Rounds in favor of the protocol, so one might not be able to borrow exactly `maxBorrow` but one unit less*. 
Thus `maxBorrow` is rounded down, which makes a position liquidatable at *one unit less*.

Even though this unit of borrowed collateral does not translate into one unit of lltv, it simply manifests that the `_isHealthy()` rounds in favor of the protocol, therefore strengthening the case for putting a *greater-than-equal* restriction to always be below the `morphoMarketLltv`. 
