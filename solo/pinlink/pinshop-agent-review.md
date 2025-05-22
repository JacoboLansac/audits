# Pinlink - Pinlinkshop Agent security review

A time-boxed security review of **Pinlink Purchasing Agent** for [**Pinlink**](https://x.com/PinLinkAi), with a focus on smart contract security.

Author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits/blob/main/README.md).

## Findings Summary



## Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time and
resource-bound effort to find as many vulnerabilities as possible, but there is no guarantee that all issues will be found.
A security researcher holds no
responsibility for the findings provided in this document. A security review is not an endorsement of the underlying
business or product and can never be taken as a guarantee that the protocol is bug-free. This security review is focused
solely on the security aspects of the Solidity implementation of the contracts. Gas optimizations are not the main
focus, but significant inefficiencies will also be reported.

## Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
| :----------------- | :----------: | :------------: | :---------: |
| Likelihood: High   |   Critical   |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

### Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the
  attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no
  incentive.

### Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the
  protocol is affected.
- **Low** - can lead to unexpected behavior with some of the protocol's functionalities that are not so critical.

### Actions required by severity level

- **High/Critical** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

## Scope

- **Main review:**
  - Start date: `2025-05-01`
  - End date: `2025-05-07`
  - Commit hash in scope:
    - [4c46efc9ab7d189fa8bab7a869ecaf06db18cea5](https://github.com/FasihHaider/PinLinkShop-Agent/blob/4c46efc9ab7d189fa8bab7a869ecaf06db18cea5/)

- **Mitigation review**
  - Mitigation review delivery date: `2025-05-14`
  - Commit hash:
    - [3d68df9ff83708305e8a606f0de6a65e6893f0b8](https://github.com/TempleDAO/origami/pull/1532/commits/3d68df9ff83708305e8a606f0de6a65e6893f0b8)

### Files in original scope

| Files in scope                      | nSLOC   |
| ----------------------------------- | ------- |
| `src/PurchaseAgent.sol`             | 131     |
| `src/interfaces/IPinlinkShop.sol`   | 11      |
| `src/interfaces/IPinlinkOracle.sol` | 3       |
| **Total**                           | **144** |


## System Overview
The purchase agent is a system to automate purchases of fractions from the PinlinkShop, a marketplace of ERC11155 RWA assets where rewards are automatically streamed for listed and non-listed assets.

The agent needs to be funded with some USDC to purchase initial fractions. From there, the USDC rewards that are accrued are compounded by purchasing more fractions. 

## Architecture high-level review
The purchase-agent is a single monolithic contract which interacts with the Pinlinkshop contract by purchasing and claiming rewards

The purchase agent is designed to manage only one tokenId at a time, which means that once all fractions from that tokenId are purchased, the contract can only keep claiming USDC, but not purchase anymore fractions. 



# Findings


## Critical risk

None.

## High risk

None.

## Medium risk

### [M-1] 

## Low risk


### [L-1] The transaction to swap rewards for PIN can be executed at any time because the configured deadline no effect 

When swapping USDC for PIN in the `_convertRewards()` function, the deadline is set to `block.timestamp + 30`:

https://github.com/FasihHaider/PinLinkShop-Agent/blob/4c46efc9ab7d189fa8bab7a869ecaf06db18cea5/src/PurchaseAgent.sol#L144

This deadline is ineffective because `block.timesatmp` will take whatever time the transaction is minted in. Assuming the intention is that the swap can only be executed within the following 30 seconds after broadcasting the transaction, this check will never work correctly. Example:

- The caller broadcast the transaction in time `t`
- The transaction waits in the mempool for `period`
- The transaction is picked up by a miner in `t + period`
- When the transaction is executed, `block.timestamp = `t + period` (when the tx is minted)
- The deadline requirement is 
  - `T + period + 30 seconds > T + period` which can be rewriten as:
  - `T + 30 seconds > T`, and this will always hold.
- So the transaction will be always minted regardless of the `period` that the tx spends in the mempool. 


### [L-2] Slippage protection can only be integers so it cannot be 0.5% for example

The slippage protection is configured as a percentage. However, it is using unsigned integers which means it can only take round numbers: 0%, 1%, 2%, ... 

https://github.com/FasihHaider/PinLinkShop-Agent/blob/4c46efc9ab7d189fa8bab7a869ecaf06db18cea5/src/PurchaseAgent.sol#L25

The divisor in the slippage calculations is 100:

https://github.com/FasihHaider/PinLinkShop-Agent/blob/4c46efc9ab7d189fa8bab7a869ecaf06db18cea5/src/PurchaseAgent.sol#L91C29-L91C30

While not a vulnerability as such, it simply means that a slippage protection of 0.5% is not possible. 

#### Suggested fix

Use basis points instead

```diff

-   uint256 purchaseSlippage = 5; // 5%
+   uint256 purchaseSlippage = 500; // 5%
-   uint256 swapSlippage = 5; // 5%
+   uint256 swapSlippage = 500; // 5%

+   uint256 constant BASIS_POINTS = 10_000; // 100%

    function performUpkeep(bytes calldata) external {
        claimRewards();
        IPinlinkShop.Listing memory listing = shop.getListing(listingId);
        if (USDC.balanceOf(address(this)) >= (listing.usdPricePerFraction / 1e12)) {
            _convertRewards();
        }
        uint256 price = shop.getQuoteInTokens(listingId, fractionsAmount);
-       uint256 maxTotalPinAmount = (price * (100 + purchaseSlippage)) / 100;
+       uint256 maxTotalPinAmount = (price * (BASIS_POINTS + purchaseSlippage)) / BASIS_POINTS;
        if (listing.amount >= fractionsAmount && PIN.balanceOf(address(this)) >= maxTotalPinAmount) {
            shop.purchase(listingId, fractionsAmount, maxTotalPinAmount);

            emit Purchased(listingId, fractionsAmount, maxTotalPinAmount);
        }
    }

    // ...

    function _convertRewards() private {
        uint256 amountIn = USDC.balanceOf(address(this));

        address[] memory path = new address[](3);
        path[0] = address(USDC);
        path[1] = WETH;
        path[2] = address(PIN);

        uint256 amountOut = IUniswapV2Router02(UniswapV2Router).getAmountsOut(amountIn, path)[1];
-       uint256 minOut = (amountOut * (100 - swapSlippage)) / 100;
+       uint256 minOut = (amountOut * (BASIS_POINTS - swapSlippage)) / BASIS_POINTS;

        IERC20(USDC).approve(address(UniswapV2Router), amountIn);

        uint256 prevPinBal = PIN.balanceOf(address(this));

        IUniswapV2Router02(UniswapV2Router).swapExactTokensForTokensSupportingFeeOnTransferTokens(
            amountIn, minOut, path, address(this), block.timestamp + 30
        );

        emit ConvertedRewards(amountIn, PIN.balanceOf(address(this)) - prevPinBal);
    }


```

