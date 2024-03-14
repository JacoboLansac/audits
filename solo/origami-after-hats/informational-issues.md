# Informational issues

## `Whitelisted::_isAllowed()` won't accpet investments from gnosis safes or other multisig wallets

Gnosis safes and other multisig wallets, are smart contract wallets, and therefore have `code.length > 0`.
Until the team sets `allowAll = true`, non whitelisted smart contract wallets investments will revert. 
Probably known by the team, but just in case. 

```javascript
    function _isAllowed(address account) internal view returns (bool) {
        if (allowAll) return true;

        // Note: If the account is a contract and access is checked within it's constructor
        // then this will still return true (unavoidable). This is just a deterrant for non-approved integrations, 
        // not intended as full protection.
        if (account.code.length == 0) return true;

        // Contracts need to be explicitly allowed
        return allowedAccounts[account];
    }

```

The function `_isAllowed()` is only used in one external (user-facing) function:
- `OrigamiInvestmentVault::investWithToken()`

It is also used in `OrigamiAbstractLovTokenManager` and `OrigamiLendingSupplyManager`, but those have already other modifiers in place like `onlyOToken`. 

## `OrigamiAaveV3BorrowAndLend.aavePool()` is hardcoded as an immutable address. 
It is recommended to always read the address. I refer to [this issue](https://github.com/hats-finance/Origami-0x998f1b716a5022be026ca6b919c0ddf45ca31abd/issues/58) from the contest. 
It would be medium, but as it has been reported in the contest for another contract, I just want to point out that here it should also be updated

## `OrigamiDebtToken::totalSupplyExcluding()` can underflow in certain circumstances giving extremely large values for `OrigamiLendingClerk::totalBorrowerDebt()`

Here is the function calculating the supply of the debt token, excluding a certain debtor. The return value is inside an unchecked block, where a subtraction is performed:

```javascript
    function totalSupplyExcluding(address[] calldata debtorList) external override view returns (uint256) {
        Debtor storage _debtor;
        uint256 _length = debtorList.length;
        uint256 _excludeSum;
        for (uint256 i; i < _length; ++i) {
            _debtor = debtors[debtorList[i]];
            unchecked {
                _excludeSum += _debtor.principal + _debtor.interestCheckpoint;
            }
        }
        
        unchecked {
@>          return uint256(totalPrincipal) + estimatedTotalInterest - _excludeSum;
        }
    }
```

The obvious question is: can this calculation underflow? To underflow, the following needs to happen:

	totalPrincipal + estimatedTotalInterest < _excludeSum
	totalPrincipal + estimatedTotalInterest < excludedPrincipal + excludedInterest
    totalPrincipal - excludedPrincipal < excludedInterests - estimatedTotalInterests
	includedPrincipal < excludedInterests - estimatedTotalInterests

The left side of the inequality (small difference between totalPrincipal and excludedPrincipal) would happen if there are few other debtors besides IdleStrategyManager. On the right side, the larger the second term, the more likely it is to underflow. The longer time without recalculating interests for other players than the excluded ones, the more underestimated the `estimatedTotalInterests`, and therefore, the larger the second term. 

Another way of looking at it:

	totalPrincipal - excludedPrincipal < excludedInterests - estimatedTotalInterests
	estimatedTotalInterests < excludedPrincipal + excludedInterest - totalPrincipal
	estimatedTotalInterests < (excludedPrincipal - totalPrincipal) + excludedInterest

The smaller the `estimatedTotalInterests`, the riskier. The more capital being outdated, the riskier.
The larger the `excludedPrincipal` and `excludedInterests`, the riskier

**Attack scenario**

I wrote an invariant test setup to try to exploit it, and I didn't find an underflowing case.

**Recommendation**

Unless there is a mathematical proof that this can never underflow, I would not use an unchecked box here. It saves so little gas here, that it is not worth it compared to the risk of underflowing. I would recommend to check if

	uint256(totalPrincipal) + estimatedTotalInterest > _excludeSum

before doing the subtraction, otherwise, just return 0. 

```diff
    function totalSupplyExcluding(address[] calldata debtorList) external override view returns (uint256) {
        Debtor storage _debtor;
        uint256 _length = debtorList.length;
        uint256 _excludeSum;
        for (uint256 i; i < _length; ++i) {
            _debtor = debtors[debtorList[i]];
            unchecked {
                _excludeSum += _debtor.principal + _debtor.interestCheckpoint;
            }
        }
  		
+ 		if (uint256(totalPrincipal) + estimatedTotalInterest > _excludeSum) {
	        unchecked {
	            return uint256(totalPrincipal) + estimatedTotalInterest - _excludeSum;
	        }
+	    else return 0;
    }
```
