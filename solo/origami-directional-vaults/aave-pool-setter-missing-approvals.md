# `OrigamiAaveV3BorrowAndLend::setAavePool()` does not approve the new pool to use handle `supplyToken` and `borrowToken`. 

The constructor of `OrigamiAaveV3BorrowAndLend` grants max approval to the `aavePool` contract to handle both `supplyToken` and `borrowToken`:

```javascript
    constructor(
        address _initialOwner,
        address _supplyToken,
        address _borrowToken,
        address _aavePool,
        uint8 _defaultEMode
    ) OrigamiElevatedAccess(_initialOwner) {
        supplyToken = _supplyToken;
        borrowToken = _borrowToken;
        
        aavePool = IAavePool(_aavePool);
        aaveAToken = IAaveAToken(aavePool.getReserveData(supplyToken).aTokenAddress);
        aaveDebtToken = IERC20Metadata(aavePool.getReserveData(_borrowToken).variableDebtTokenAddress);

        // Approve the supply and borrow to the Aave/Spark pool upfront
@>      IERC20(supplyToken).forceApprove(address(aavePool), type(uint256).max);
@>      IERC20(borrowToken).forceApprove(address(aavePool), type(uint256).max);

        // Initate e-mode on the Aave/Spark pool if required
        if (_defaultEMode != 0) {
            aavePool.setUserEMode(_defaultEMode);
        }
    }
```

There is also a function to set a new address for `aavePool`, as Aave does not recommend hard-coding pool addresses. 
However, when a new pool address is set, the approvals from the previous pool are not migrated to the new one:

```javascript
    function setAavePool(address pool) external override onlyElevatedAccess {
        if (pool == address(0)) revert CommonEventsAndErrors.InvalidAddress(pool);
        emit AavePoolSet(pool);
        aavePool = IAavePool(pool);
@>      // @audit-issue missing granting approvals to the new aave pool
@>      // @audit-issue missing revoking approvals of the old aave pool
    }
```

This has two implications:
- The deployed `OrigamiAaveV3BorrowAndLend` cannot operate with the new aave pool
- The deployed `OrigamiAaveV3BorrowAndLend` will have dandling approvals for the old pool

## Impact: low/medium?

The functionality for switching to a new aave pool is broken, as the pool expects to be able to handle tokens of the contract that interacts with it. 

- There won't be funds at risk as long as the original pool is not compromised.
- Any elevated-access wallets would have been able to set the original pool back in case this issue went unnoticed, withdraw the funds from the original pool, and simply deploy a new instance of the `OrigamiAaveV3BorrowAndLend`. Not ideal, but no user funds are at risk. 

## Suggested mitigation

Approve the new pool and revoke the allowance of the old pool when a new one is set.

```diff
    function setAavePool(address pool) external override onlyElevatedAccess {
+       address oldPool = aavePool;
        if (pool == address(0)) revert CommonEventsAndErrors.InvalidAddress(pool);
        emit AavePoolSet(pool);
        aavePool = IAavePool(pool);
+       IERC20(supplyToken).forceApprove(pool, type(uint256).max);
+       IERC20(borrowToken).forceApprove(pool, type(uint256).max);
+       IERC20(supplyToken).forceApprove(oldPool, 0);
+       IERC20(borrowToken).forceApprove(oldPool, 0);
    }
```
