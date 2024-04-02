# `maxSafeTvl` is not checked in constructor as in `setMaxSafeLtv()`

*OrigamiMorphoBorrowAndLend.sol*

```javascript

    constructor(
        address _initialOwner,
        address __supplyToken,
        address __borrowToken,
        address _morphoAddress,
        address _morphoMarketOracle, 
        address _morphoMarketIrm,
        uint96 _morphoMarketLltv,
        uint256 _maxSafeLtv
    ) OrigamiElevatedAccess(_initialOwner) {
        _supplyToken = IERC20(__supplyToken);
        _borrowToken = IERC20(__borrowToken);
        
        morpho = IMorpho(_morphoAddress);

        morphoMarketOracle = _morphoMarketOracle;
        morphoMarketIrm = _morphoMarketIrm;
        morphoMarketLltv = _morphoMarketLltv;
        maxSafeLtv = _maxSafeLtv;

        marketId = getMarketParams().id();

        // ...
```


```javascript
    /**
     * @notice Set the max LTV we will allow when borrowing or withdrawing collateral.
     * @dev The morpho LTV is the liquidation LTV only, we don't want to allow up to that limit
     * so we set a more restrictive 'safe' LTV'
     */
    function setMaxSafeLtv(uint256 _maxSafeLtv) external override onlyElevatedAccess {
        if (_maxSafeLtv >= morphoMarketLltv) revert CommonEventsAndErrors.InvalidParam();
        maxSafeLtv = _maxSafeLtv;
        emit MaxSafeLtvSet(_maxSafeLtv);
    }
    
```

## Recommendation
Either use `setMaxSafeLtv()` in the constructor (visibility needs to be changed to `public`), or add an explicit check in the constructor.
