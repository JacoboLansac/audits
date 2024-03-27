# LovToken's `maxTotalSupply` cap is not checked in all instances of `_mint()`

A new feature was added to `OrigamiLovToken`, to cap the `totalSupply()` of the `lovToken` so that it doesn't exceed `maxTotalSupply`. 

The function `OrigamiLovToken::investWithToken()` reverts if exceeding that cap:

*OrigamiLovToken:*
```javascript
    function investWithToken(
        InvestQuoteData calldata quoteData
    ) external virtual override nonReentrant returns (uint256 investmentAmount) {
        if (quoteData.fromTokenAmount == 0) revert CommonEventsAndErrors.ExpectedNonZero();

        // Send the investment token to the manager
        IOrigamiLovTokenManager _manager = lovManager;
        IERC20(quoteData.fromToken).safeTransferFrom(msg.sender, address(_manager), quoteData.fromTokenAmount);
        investmentAmount = _manager.investWithToken(msg.sender, quoteData);

        emit Invested(msg.sender, quoteData.fromTokenAmount, quoteData.fromToken, investmentAmount);

        // Mint the lovToken for the user
        if (investmentAmount != 0) {
@>          _mint(msg.sender, investmentAmount);
@>          if (totalSupply() > maxTotalSupply) {
                revert CommonEventsAndErrors.BreachedMaxTotalSupply(totalSupply(), maxTotalSupply);
            }
        }
    }

```

However, `OrigamiLovToken` inherits `OrigamiInvestment`, which inherits `MintableToken`:

```javascript
@>  contract OrigamiLovToken is IOrigamiLovToken, OrigamiInvestment {
    	using SafeERC20 for IERC20;
	
    	// ...

```

```javascript

@>  abstract contract OrigamiInvestment is IOrigamiInvestment, MintableToken, ReentrancyGuard {
        string public constant API_VERSION = "0.2.0";
   
   		// ... 
```

And `MintableToken` also has a `mint()` function, in which the `totalSupply` is not capped:

```javascript
/// @notice An ERC20 token which can be minted/burnt by approved accounts
abstract contract MintableToken is IMintableToken, ERC20Permit, OrigamiElevatedAccess {
    using SafeERC20 for IERC20;

    // ...


    function mint(address _to, uint256 _amount) external override {
        if (!_minters[msg.sender]) revert CannotMintOrBurn(msg.sender);
        _mint(_to, _amount); // @audit-issue `maxTotalSupply` cap is not taken into account here
    }

    // ... 
``` 

## Impact: low

Low because only _minters_ can mint, and only _owners_ can add _minters_. 
It could potentially be a higher severity issue, but only if other minters were added, and those minters logic had no checks on the `maxSupply`.

## Recommendation:

Even though this cap is specific to Morpho, perhaps other future lovTokens (or `MintableTokens`) also require a `maxSupply` cap. It would make sense to place that `maxSupply` cap on the `MintableToken` contract as an immutable variable passed to the constructor, which would be set to `type(uint256).max` if no cap is needed.
