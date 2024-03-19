# In `OrigamiLovTokenFlashAndBorrowManager` the liabilities function rounds down instead of up, due to a miss-naming of the oracle storage variable

The `lovStEth` vault uses `OrigamiWstEthToEthOracle`, which initializes the parent `OrigamiOracleBase` with the following arguments:
- `baseAsset` = `wstETH`
- `quoteAsset` = `ethAddress` (`WETH`)

```javascript
contract OrigamiWstEthToEthOracle is OrigamiOracleBase {

	// ...

    constructor (
        string memory _description,
        address _wstEthAddress,
        uint8 _wstEthDecimals,
        address _ethAddress,
        uint8 _ethDecimals,
        address _stEth,
        address _stEthToEthOracle
    ) 
        OrigamiOracleBase(
            _description,  // _description,
@>          _wstEthAddress,  // _baseAssetAddress,
            _wstEthDecimals,  // _baseAssetDecimals,
@>          _ethAddress,  // _quoteAssetAddress,  (WETH)
            _ethDecimals // _quoteAssetDecimals
        )

    // ... 

```

The price units returned by `OrigamiWstEthToEthOracle::latestPrice()` are *wstETH to ETH* or *ETH per wstETH*, what I normally call `[ETH/wstETH]`.

```javascript
    function latestPrice(
        PriceType priceType, 
        OrigamiMath.Rounding roundingMode
    ) public override view returns (uint256 price) {
        // 1 wstETH to stETH
        price = stEth.getPooledEthByShares(precision);  //jl [1 wstETH to stETH] == [stETH per wstETH] == [stETH/wstETH]

        // Convert wstETH to ETH using the stEth/ETH oracle price
        price = price.mulDiv(
            stEthToEthOracle.latestPrice(priceType, roundingMode),  //jl stEthToEth == [1 stETH to ETH] == [ETH per stETH] == [ETH/stETH]
            precision,
            roundingMode
        );
        // units conversion:
        //  price[stETH/wstETH] * [ETH/stETH] = return[ETH/wstETH] == [1 wstETH to ETH]
    }
```

The output of `latestPrice()` (*1 wsETH to ETH*) matches the oracle naming: `OrigamiWstEthToEthOracle`. So far, so good.

However, in the `OrigamiLovTokenFlashAndBorrowManager` contract, which uses `OrigamiWstEthToEthOracle`, the storage variable for this oracle is called `debtTokenToReserveTokenOracle`. Note that in the context of `lovStEth`, the assets are:
- `debtToken` = `WETH`
- `reserveToken` = `wstETH`

Following the naming `baseTokenToQuoteToken`, the placeholder for `OrigamiWstEthToEthOracle` should be called `reserveTokenToDebtTokenOracle`. *HOWEVER*, this naming is flipped to `debtTokenToReserveTokenOracle`.

```javascript
contract OrigamiLovTokenFlashAndBorrowManager {

	function setOracles(
@>		address _debtTokenToReserveTokenOracle, // in testDeployer: wstEthToEthOracle  ... which should be called "reserveTokenToDebtToken". @audit-issue naming seems inverted
		address _dynamicFeePriceOracle  		// in testDeployer: stEthToEthOracle
	) external override onlyElevatedAccess {
@>      debtTokenToReserveTokenOracle = _validatedOracle(_debtTokenToReserveTokenOracle, address(_reserveToken), address(_debtToken));  // @audit-issue storage variable naming is also inverted
        dynamicFeePriceOracle = _validatedOracle(_dynamicFeePriceOracle, dynamicFeeOracleBaseToken, address(_debtToken));
        emit OraclesSet(_debtTokenToReserveTokenOracle, _dynamicFeePriceOracle);
    }
}
```

**The Issue**

The only place where this oracle is used is in the `liabilities()` function. This function reads the `debt` from the borrowLend contract, measured in `WETH`. Then the function invokes the oracle to convert the debt in `WETH` (debt token) to `wstETH` (reserve token). For this it uses the `debtTokenToReserveTokenOracle::convertAmount()` function. Note that the rounding method is ROUND_UP, which is a conservative (good) approach in the case of the vault's liabilities. 

```javascript
    function liabilities(IOrigamiOracle.PriceType debtPriceType) public override(OrigamiAbstractLovTokenManager,IOrigamiLovTokenManager) view returns (uint256) {
        // In [debtToken] terms.  // WETH
        uint256 debt = borrowLend.debtBalance();
        if (debt == 0) return 0;

        // Convert the [debtToken][WETH] into the [reserveToken][wstETH] terms
@>      return debtTokenToReserveTokenOracle.convertAmount(
            address(_debtToken),  			// fromAsset 		// WETH 
            debt,  							// fromAssetAmount 	// WETH units
            debtPriceType,   				// priceType
            OrigamiMath.Rounding.ROUND_UP  	// roundingMode 	// @audit-issue The intention of rounding up liabilities is correct, however, this will be flipped to DOWN.
        );
    }
```

Note in the comments of the snippet above, that the `fromAsset = WETH`. Note below that `convertAmount()` flips the rounding method if `fromAsset == quoteAsset`, and remember that in this oracle  `quoteAsset = WETH`. Therefore, the second path is invoked, where the rounding method is flipped. The liabilities will be rounded DOWN, which is less conservative and probably the opposite or intended.

```javascript
    function convertAmount(
        address fromAsset,
        uint256 fromAssetAmount,
        PriceType priceType,
        OrigamiMath.Rounding roundingMode 
    ) external override view returns (uint256 toAssetAmount) {
        if (fromAsset == baseAsset) {
            // The numerator needs to round in the same way to be conservative
            uint256 _price = latestPrice(
                priceType, 
                roundingMode
            );

            return fromAssetAmount.mulDiv(
                _price,
                assetScalingFactor,
@>              roundingMode
            );
        } else if (fromAsset == quoteAsset) {  // @audit fromAsset == WETH == quoteAsset
            // The denominator needs to round in the opposite way to be conservative
            uint256 _price = latestPrice(  // units: [quoteAsset/baseAsset] == [WETH/wstETH]
                priceType, 
@>              roundingMode == OrigamiMath.Rounding.ROUND_UP ? OrigamiMath.Rounding.ROUND_DOWN : OrigamiMath.Rounding.ROUND_UP
            );

            // fromAssetAmount[WETH] * scaling / [WETH/wstETH] --> wstETH
            return fromAssetAmount.mulDiv(
                assetScalingFactor,
                _price,
                roundingMode
            );
        }

        revert CommonEventsAndErrors.InvalidToken(fromAsset);
    }
```
**Impact: low**
The `liabilities()` in lovStEth system will be rounded DOWN, instead of the conservative approach of rounding UP.

**Recommendation**
In `OrigamiLovTokenFlashAndBorrowManager`,
- Change the naming of the oracle storage variable from `debtTokenToReserveTokenOracle` to `reserveTokenToDebtTokenOracle`, and the input args of `setOracles()`. This has no real effect, but it is good for consistency.
- Change the rounding strategy when calling `liabilities()`

```diff
    function liabilities(IOrigamiOracle.PriceType debtPriceType) public override(OrigamiAbstractLovTokenManager,IOrigamiLovTokenManager) view returns (uint256) {
        // In [debtToken] terms.  // WETH
        uint256 debt = borrowLend.debtBalance();
        if (debt == 0) return 0;

        // Convert the [debtToken][WETH] into the [reserveToken][wstETH] terms
        return debtTokenToReserveTokenOracle.convertAmount(
            address(_debtToken),
            debt,
            debtPriceType,
-           OrigamiMath.Rounding.ROUND_UP
+           OrigamiMath.Rounding.ROUND_DOWN // price is rounded down because it is expressed as base-to-quote, which is wstETH-to-WETH, and here we convert from WETH to wstETH, so rounding will be inverted.
        );
    }
```

