# An account can have a lower debt than `minBorrowAmount` because the necessary requirement is only present in `TempleLineOfCredit::borrow()` but not in `repay()` 

When a liquidation occurs in `TempleLineOfCredit`, the collateral (Temple tokens) goes to the `treasuryReservesVault`. This offers no incentive to the caller of the `batchLiquidate()` function (used to liquidate unhealthy accounts) which leads me to believe that the protocol admins are the intended liquidators (even though any account can liquidate any position).

If the collateral of a liquidated position was smaller than the gas cost of executing the liquidation transaction, the difference would be paid by the protocol. To prevent such scenario, there is a `minBorrowAmount` parameter that is used in the `borrow()` so that borrowing small amounts is disallowed. Small borrows would allow small collaterals, which could lead to situations in which the gas cost of liquidation is larger than the seized collateral:
```javascript
    /**
     * @notice The minimum borrow amount per transaction
@>   * @dev It costs gas to liquidate users, so we don't want dust amounts.
     */
    uint128 public override minBorrowAmount = 1000e18;  // units: DAI
```

```javascript
   function borrow(uint128 amount, address recipient) external override notInRescueMode {
@>      if (amount < minBorrowAmount) revert InsufficientAmount(minBorrowAmount, amount);

        // ...

    }
``` 

We can translate the intention of the above logic into a protocol invariant:

> Contract invariant: An account's debt should never be below `minBorrowAmount`.

However, an account's debt is not checked in the `repay()` function. This makes it trivial for any account to break the invariant above by borrowing the `minBorrowAmount` and then repaying part of it The remaining debt would be obviously smaller than `minBorrowAmount`.

This simple action invalidates the `minBorrowAmount` parameter, as an account could call sequentially `borrow()` and `repay()` in the same block to reach any desired amount of debt below the `minBorrowAmount`.

Furthermore, besides lacking minimum-debt requirements in the `repay()` function, there isn't either any debt requirements in `removeCollateral()`. Therefore, reaching a state with a lower collateral value than the liquidation gas cost is trivial. Note that `removeCollateral()` does require a healthy LTV ratio, but that has nothing to do with the size of the debt and collateral, only the ratio between them. 

## Aditional context

The parameter `minBorrowAmount` was introduced as a response to [this issue reported by yAudits](https://reports.yaudit.dev/reports/07-2023-TempleDAO-Lending/#5-low---weak-liquidation-incentives-can-result-in-unliquidated-bad-debt), which includes a great detailed description of the impact. 

## Attack scenario

This `batchLiquidate()` [transaction](https://etherscan.io/tx/0x5f77af23cf44869a547798e4b8ae7bef1f99be426995b913a2de61b3d3cfe855) is an example of `batchLiquidate()`, which can be used to quantify the impact in different scenarios. The liquidation transaction fee is 0.0011265713955 ETH, which (assuming an ETH price of 3000$) is:
- $67 if the gas price was 100 gwei
- $100 if the gas price was 150 gwei
- $135 if the gas price was 200 gwei

For instance, if the collateral of a position is smaller than $60 (of Temple tokens), liquidating that position with any gas price above 100 gwei would cost more gas than the seized collateral. It is worth mentioning that gas spikes often come with volatility, which could also be a trigger for liquidatable positions. However, we should compare the cost of setting up a small collateral position, vs the profit the attacker could make if his position was not liquidated and he could get away with the debt:

- `borrow()` gas costs: 0.0072915604630806 ETH
- `repay()` gas costs: 0.00582253802259511 ETH
- approximate cost of setting up the "attack" = `borrow() + repay()` = 0.013114098 ETH approx
- approximate cost of liquidation: 0.0011265713955 ETH

The cost of borrowing + repaying is higher than the cost of liquidation, and this one needs to be higher than the collateral, which has to be higher than the debt. Put in other words, the following is a condition for the attack setup:

> cost of liquidation > collateral > debt 

However, as the cost of setting up the attack is already higher than the cost of liquidation, the attack would not be profitable, and it can only be taken into consideration as a grieving attack. 

## Impact: low

The extent of the impact is greatly described in the above mentioned [report by yAudits](https://reports.yaudit.dev/reports/07-2023-TempleDAO-Lending/#5-low---weak-liquidation-incentives-can-result-in-unliquidated-bad-debt), so I will not add much more details. But the core impact is:

> Any account can reduce debt and collateral so that the value of the collateral is smaller than the liquidation gas costs, leading to a cost for the protocol when liquidating that position, and therefore impacting the Temple target price index, and ultimately the holders. 

However, as shown before, the attack is not profitable, so the probability of occurrence is rather low, as damaging the protocol would be at the expense of the attacker.

However, as the `minBorrowAmount` is easily bypassed by the simple strategy of `borrow()` + `repay()`, it would classify as a low finding according to:

> Contract does not function as expected, with no loss of funds. 

One could argue there is a loss of funds from the protocol in the shape of gas costs, but it is very low. 

## Proof of code

Here is a proof of code demonstrating how a position can have less debt than `minBorrowAmount`, and a collateral value below liquidation costs if gas prices are high. The proof of code is written in Foundry, using a mainnet fork with the deployed contracts taken from Etherscan. 

```javascript
pragma solidity 0.8.19;
// SPDX-License-Identifier: AGPL-3.0-or-later

import {Test, console} from "lib/forge-std/src/Test.sol";  // foundry stuff
import {IERC20} from "lib/forge-std/src/interfaces/IERC20.sol";  // foundry stuff
import {TempleLineOfCredit} from "contracts/v2/templeLineOfCredit/TempleLineOfCredit.sol";  // downloaded from etherscan
import {ITlcDataTypes} from "contracts/interfaces/v2/templeLineOfCredit/ITlcDataTypes.sol";  // downloaded from etherscan

contract BaseTestTempleLineOfCredit is Test {
    IERC20 templeToken;
    IERC20 daiToken;
    TempleLineOfCredit templeLineOfCredit;

    address user = makeAddr("user");

    function setUp() public virtual {
        templeToken = IERC20(0x470EBf5f030Ed85Fc1ed4C2d36B9DD02e77CF1b7); // mainnet address
        daiToken = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F); // mainnet address
        templeLineOfCredit = TempleLineOfCredit(0xcbc0A8d5C7352Fe3625614ea343019e6d6b89031); // mainnet address

        // give some tokens to the user and approve the TempleLineOfCredit to use them
        deal(address(daiToken), user, 10000 ether);
        deal(address(templeToken), user, 10000 ether);

        vm.startPrank(user);
        templeToken.approve(address(templeLineOfCredit), type(uint256).max);
        daiToken.approve(address(templeLineOfCredit), type(uint256).max);
        vm.stopPrank();
    }
}

contract TestMinimumDebtInvariant is BaseTestTempleLineOfCredit {
    function setUp() public override {
        super.setUp();
    }

    function test_accountWithLessDebtThanMinBorrowAmountShouldRevert() public {
        // invariant to test: "an account cannot have less debt than the minBorrowAmount"

        uint128 collateralAmount = 2500e18;
        vm.startPrank(user);
        // add some collateral
        templeLineOfCredit.addCollateral(collateralAmount, user);
        // borrow the minimum amount, 1000 DAI
        templeLineOfCredit.borrow(1000e18, user);

        // repay most of the debt, leaving a very small remaining debt (10 DAI)
        // NOTE: this repayment should revert, as it leaves the account's debt below `minBorrowAmount`
        // vm.expectRevert();
        templeLineOfCredit.repay(950e18, user);

        // now the account removesCollateral. 
        // 50 DAI left of debt. maxLtv = 80%. 50/coll = 0.8 => collateralLeft = 62.5 $, [@ 1.20 DAI/Temple] = 52.08 Temple at least
        uint128 collateralLeft = 55e18;  // Temple tokens (value of approx 66 DAI)
        templeLineOfCredit.removeCollateral(collateralAmount - collateralLeft, user);

        vm.stopPrank();

        ITlcDataTypes.AccountPosition memory accountPosition = templeLineOfCredit.accountPosition(user);

        // the account managed to have a debt below the minBorrowAmount, and a collateral below the cost of liquidation if gas prices are high
        assertEq(accountPosition.currentDebt, 50e18);  // only 50 DAI of debt left
        assertEq(accountPosition.collateral, collateralLeft); // only 55 Temple of collateral (about 66 DAI of collateral value)
        assertLt(accountPosition.currentDebt, templeLineOfCredit.minBorrowAmount()); // invariant broken
    }
}
```

I recommend running the test caching the fork number to avoid bombarding your node provider with queries:

    forge test -vvv --fork-url <alchemy/infura-url> --fork-block-number 19792040 

## Recommendation

My recommendation is to include a similar requirement in the `repay()` function to enforce that an account's debt never drops below the `minBorrowAmount`, unless when it is fully repaid. 

```diff
    function repay(uint128 repayAmount, address onBehalfOf) external override notInRescueMode {
        if (repayAmount == 0) revert CommonEventsAndErrors.ExpectedNonZero();

        AccountData storage _accountData = allAccountsData[onBehalfOf];
        DebtTokenCache memory _cache = _debtTokenCache();

        // Update the account's latest debt
        uint128 _newDebt = _currentAccountDebt(
            _cache,
            _accountData.debtCheckpoint,
            _accountData.interestAccumulator,
            true
        );

        // They cannot repay more than this debt
        if (repayAmount > _newDebt) {
            revert ExceededBorrowedAmount(_newDebt, repayAmount);
        }
        unchecked {
            _newDebt -= repayAmount;
        }
+       if (_newDebt < minBorrowAmount) revert InsufficientAmount(minBorrowAmount, _newDebt);

        // Update storage
        _accountData.debtCheckpoint = _newDebt;
        _accountData.interestAccumulator = _cache.interestAccumulator;

        _repayToken(_cache, repayAmount, msg.sender, onBehalfOf);
    }

```

It is important to note, that the global debt accumulator is updated when calling `_debtTokenCache()`, which means that the outstanding debt of an account could be slightly higher than expected when calling `repay()` with the exact `repayAmount` of outstanding debt. This new requirement causes a revert if the recalculated debt is slightly higher than the input `repayAmount`. 

However, I don't see this as an issue, because there is a dedicated function to repay all outstanding debt: `repayAll()`, and therefore the `repay()` function should only be used for partial repayments.
