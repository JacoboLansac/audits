# PinLink security review

***Preliminar version, 2024-07-15***

A time-boxed security review of the [**Soar(no-link)**](nolink) protocol, with a focus on smart contract security and gas optimizations.

Author: [**Jacopod**](https://twitter.com/jacolansac), independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits/blob/main/README.md).

## Findings Summary

During the security review, 2 critical, 4 high, and 8 medium risk issues were found. Some low-risk findings and gas optimizations were also identified.






| Finding                                                                                                                                                                  | Severity | Description                                                                                                                                                    | Status       |
| :----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----------- |
| [C-1](<#c-1-users-can-claim-from-the-minter-multiple-times-draining-the-contract-of-soar-tokens>)                                                                        | Critical | Users can claim from the Minter multiple times, draining the contract of SOAR tokens                                                                           | ✅ Resolved   |
| [C-2](<#c-2-wrong-decimals-scaling-of-oracle-price-allows-users-to-purchase-soar-from-in-mintersol-almost-for-free>)                                                     | Critical | Wrong decimals scaling of oracle price allows users to purchase SOAR from in `Minter.sol` almost for free                                                      | ✅ Resolved   |
| [H-1](<#h-1-the-staking-contract-will-become-insolvent-if-recovernonlockedrewardtokens-is-called>)                                                                       | High     | The staking contract will become insolvent if `recoverNonLockedRewardTokens()` is called                                                                       | ✅ Resolved   |
| [H-2](<#h-2-lack-of-slippage-protection-in-liquidityswap-can-result-in-significant-loss-of-value>)                                                                       | High     | Lack of slippage protection in `Liquidity.swap()` can result in significant loss of value                                                                      | ✅ Resolved   |
| [H-3](<#h-3-the-minter-contract-cannot-guarantee-solvency-because-purchases-are-allowed-regardless-of-the-soar-tokens-in-the-contracts-balance>)                         | High     | The Minter contract cannot guarantee solvency because purchases are allowed regardless of the SOAR tokens in the contract's balance                            | ✅ Resolved   |
| [H-4](<#h-4-prices-in-minter-depend-on-a-single-oracle-which-makes-it-vulnerable-to-price-manipulation-attacks-incomplete-finding-until-oracle-integration-is-finished>) | High     | Prices in Minter depend on a single oracle, which makes it vulnerable to price-manipulation attacks* (incomplete finding until oracle integration is finished) | -            |
| [M-1](<#m-1-pending-claims-in-the-minter-will-become-blocked-again-if-new-purchases-are-made-before-claiming>)                                                           | Medium   | Pending claims in the Minter will become blocked again if new purchases are made before claiming                                                               | ✅ Resolved   |
| [M-2](<#m-2-the-soarsol-and-liquiditysol-contracts-can-receive-eth-but-cannot-send-it-so-it-will-get-stuck-in-the-contract>)                                             | Medium   | The `Soar.sol` and `Liquidity.sol` contracts can receive ETH, but cannot send it so it will get stuck in the contract                                          | ✅ Resolved   |
| [M-3](<#m-3-soarstakingsetrewards-can-revert-under-certain-circumstances-blocking-the-admin-from-setting-new-reward-periods-due-to-a-wrong-order-of-operands>)           | Medium   | `SoarStaking.setRewards()` can revert under certain circumstances, blocking the admin from setting new reward periods due to a wrong order of operands         | ✅ Resolved   |
| [M-4](<#m-4-the-deadline-of-liquiditymintnewposition-has-no-effect>)                                                                                                     | Medium   | The deadline of `Liquidity.mintNewPosition()` has no effect                                                                                                    | ✅ Resolved   |
| [M-5](<#m-5-lack-of-input-validation-in-tax-setter-functions-can-halt-soar-token-transfers>)                                                                             | Medium   | Lack of input validation in tax setter functions can halt SOAR token transfers                                                                                 | ✅ Resolved   |
| [M-6](<#m-6-lack-of-input-validation-in-tax-setter-functions-allows-the-contract-owner-set-as-high-tax-as-100>)                                                          | Medium   | Lack of input validation in tax setter functions allows the contract owner set as high tax as 100%                                                             | ✅ Resolved   |
| [M-7](<#m-7-lack-of-slippage-protection-in-liquidity-management-functions-can-result-in-lost-value-for-the-protocol-when-addingremoving-liquidity>)                      | Medium   | Lack of slippage protection in liquidity management functions can result in lost value for the protocol when adding/removing liquidity                         | ✅ Resolved   |
| [L-1](<#l-1-soaropentrading-could-be-frontrun-to-alter-the-launching-price>)                                                                                             | Low      | Soar.OpenTrading could be frontrun to alter the launching price                                                                                                | ✅ Resolved   |
| [L-2](<#l-2-liquidity-addition-can-be-frontrun-by-another-soar-holder-setting-the-launch-price>)                                                                         | Low      | Liquidity addition can be frontrun by another SOAR holder setting the launch price                                                                             | ✅ Resolved   |
| [L-3](<#l-3-lack-of-input-validation-in-soarstakingsetrewards-allows-configuring-an-already-finished-period>)                                                            | Low      | Lack of input validation in `SoarStaking.setRewards()` allows configuring an already finished period                                                           | Acknowledged |
| [L-4](<#l-4-some-dex-could-evade-the-trading-tax-if-the-pool-contract-does-not-expose-token0-and-token1-functions>)                                                      | Low      | Some DEX could evade the trading tax if the pool contract does not expose `token0()` and `token1()` functions                                                  | Acknowledged |
| [L-5](<#l-5-lack-of-deadlines-allows-liquidity-management-operations-to-be-postponed-indefinitely>)                                                                      | Low      | Lack of deadlines allows Liquidity management operations to be postponed indefinitely                                                                          | ✅ Resolved   |
| [L-6](<#l-6-in-the-minter-contract-totalprice-can-be-0-even-if-tokenprice-is-not-0>)                                                                                     | Low      | In the Minter contract, `totalPrice` can be 0 even if `tokenPrice` is not 0                                                                                    | ✅ Resolved   |
| [L-7](<#l-7-some-state-changing-functions-do-not-emit-events>)                                                                                                           | Low      | Some state-changing functions do not emit events                                                                                                               | ✅ Resolved   |
| [L-8](<#l-8-soaropentrading-can-be-called-before-the-taxreceiver-is-set-sending-fees-to-the-zero-address>)                                                               | Low      | Soar.openTrading() can be called before the `taxReceiver` is set, sending fees to the zero address                                                             | ✅ Resolved   |
| [L-9](<#l-9-potential-reentrancy-attack-as-the-state-is-modified-after-an-external-call-in-soarstakinggetrewards>)                                                       | Low      | Potential reentrancy attack as the state is modified after an external call in `SoarStaking.getRewards()`                                                      | ✅ Resolved   |
| [L-10](<#l-10-soar-token-might-have-integration-issues-with-other-protocols-because-contract-can-make-incomingoutgoing-transfers-revert>)                                | Low      | Soar token might have integration issues with other protocols because contract can make incoming/outgoing transfers revert                                     | Acknowledged |
| [G-1](<#g-1-the-updatereward-modifier-reads-rewardpertokenstored-storage-variable-multiple-times->)                                                                      | Gas      | The `updateReward()` modifier reads `rewardPerTokenStored` storage variable multiple times                                                                     | ✅ Resolved   |
| [G-2](<#g-2-soar_shouldtaketax-makes-multiple-external-calls-and-storage-reads-making-the-token-transfers-unnecessarily-expensive>)                                      | Gas      | Soar._shouldTakeTax() makes multiple external calls and storage reads, making the token transfers unnecessarily expensive                                      | ✅ Resolved   |
| [G-3](<#g-3-liquiditycreatenewposition-makes-unnecessary-external-calls-to-read-values-that-are-already-in-memory>)                                                      | Gas      | `Liquidity.createNewPosition()` makes unnecessary external calls to read values that are already in memory                                                     | ✅ Resolved   |
| [G-4](<#g-4-unused-variable-in-liquiditydecreaseliquidity-wastes-gas>)                                                                                                   | Gas      | Unused variable in `Liquidity.decreaseLiquidity()` wastes gas                                                                                                  | ✅ Resolved   |
| [G-5](<#g-5-soarstakingsetreward-makes-an-unnecessary-storage-read-of-an-already-in-memory-variable>)                                                                    | Gas      | `SoarStaking.setReward()` makes an unnecessary storage read of an already in-memory variable                                                                   | ✅ Resolved   |






## Introduction

## Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time and
resource-bound effort to find as many vulnerabilities as possible, but there is no guarantee that all issues will be
found.
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

- Draft delivery date: `2024-07-17`
- Duration of the audit: 7 days
- Commit hashes in scope:
  - [d0b28c1d2bf20cd2ca1e3493f6ebada0c3fde4e1](https://github.com/meegalaxy/SoarContract/commit/d0b28c1d2bf20cd2ca1e3493f6ebada0c3fde4e1) (initial commit).
  - [e976d0d69a7dafdf516f4fdc49225538020e4b06](https://github.com/meegalaxy/SoarContract/commit/e976d0d69a7dafdf516f4fdc49225538020e4b06) (updates in `Minter.sol`).

### Mitigation review
- [2827fef0d30b1c44dac82f22d233ca8b0fe4a73f](https://github.com/meegalaxy/SoarContract/commit/2827fef0d30b1c44dac82f22d233ca8b0fe4a73f)
- [e96ee5400ed69ebe73ec76fe899e81cfd0ad56a6](https://github.com/meegalaxy/SoarContract/commit/e96ee5400ed69ebe73ec76fe899e81cfd0ad56a6)

### Files in original scope

| File                        | nSLOC    | Notes                                                    |
| --------------------------- | -------- | -------------------------------------------------------- |
| `contracts/Minter.sol`      | 38 -> 57 | Upgradeable, Ownable.                                    |
| `contracts/Soar.sol`        | 138      | Taxable ERC20, Ownable, reads price from oracle          |
| `contracts/SoarStaking.sol` | 182      | Upgradeable, Ownable. Handles native eth.                |
| `contracts/Liquidity.sol`   | 178      | Integrates with UniswapV3 liquidity positions and swaps. |
| **Total**                   | **555**  |                                                          |


### Libraries / standards
| Dependency / Import Path                                                 | Count |
| ------------------------------------------------------------------------ | ----- |
| @openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol        | 2     |
| @openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol     | 2     |
| @openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol        | 1     |
| @openzeppelin/contracts-upgradeable/utils/ReentrancyGuardUpgradeable.sol | 1     |
| @openzeppelin/contracts/access/Ownable.sol                               | 2     |
| @openzeppelin/contracts/token/ERC20/ERC20.sol                            | 2     |
| @openzeppelin/contracts/token/ERC20/IERC20.sol                           | 1     |
| @openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol                  | 1     |
| @openzeppelin/contracts/token/ERC721/IERC721.sol                         | 1     |
| @openzeppelin/contracts/token/ERC721/IERC721Receiver.sol                 | 1     |
| @uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol               | 1     |
| @uniswap/v3-periphery/contracts/libraries/TransferHelper.sol             | 1     |


## Protocol Overview

Team description of each contract:

- Soar.sol: It's a token contract with max tx limit and tax.
- SoarStaking.sol: It's Soar Staking and ETH reward contract.
- Minter.sol: Users can purchase soar tokens with USDC based on Soar/ETH price. We transfer tokens to the minter contract and if users purchase, they get from the contract. 
- Liquidity.sol: If users purchase tokens, we will have USDC. We swap some USDC to ETH and other tokens and add liquidity to the Uniswap v3 to get fees. We add these fees to the staking contract as a rewards.

### Minter oracle

- The Minter contract relies on an `oracle` contract, providing the SOAR price denominated in USD. 
- This oracle bases the price in the ETH/USD price feed and the price from the liquidity pair SOAR/ETH

### General info:

- Deploy chain: Ethereum mainnet
- which addresses will be excluded from the transfer fees of Soar.sol?
    - owner
    - pair address (Uniswap V2)
    - Soar staking
    - Minter contract
- Which tokens can be purchaseToken in Minter?
  - only USDC


### Architecture high level review

- The architecture is well organized and generally gas-efficient
- Contracts have an acceptable level of inline comments, although some function doc strings are missing
- The `Minter.sol` and `SoarStaknig.sol` are upgradeable contracts. However, **I do not endorse upgradeability on a staking contract holding user funds**, as it is an important centralization risk: rugpull of all staked tokens in case of malicious owner or compromised private keys. 


# Findings

## Critical risk

### [C-1] Users can claim from the Minter multiple times, draining the contract of SOAR tokens

When a user claims the purchased SOAR tokens in the Minter contract, there is no check or registry that the user has claimed them. 
This allows any user to call the `claim()` function for as long as SOAR tokens are in the Minter's balance.

```javascript
    function claim() external {
        Recipient memory _recipient = recipients[_msgSender()];
        require(
            block.timestamp >= _recipient.claimTime,
            "Claim: Locked for now"
        );
        require(_recipient.amount > 0, "Claim: Nothing to claim");
        
        SafeERC20.safeTransfer(soarToken, _msgSender(), _recipient.amount);
    }
```

#### Impact

Any user that has purchased an amount of SOAR tokens, can drain the entire SOAR balance by calling `claim()` multiple times. 

#### Proof of concept
- Admin transfers 100.000 SOAR tokens to the MInter contract 
- User purchases 1000 SOAR tokens
- User waits for the locking period
- User calls `claim()` 100 times, stealing the 100.000 SOAR tokens from other buyers

#### Mitigation

When a user claims, the entry of `msg.sender` in the `recipients` mapping should be deleted. Note that `_recipients.amount` is not deleted because it has been buffered into memory.

```diff
    function claim() external {
        Recipient memory _recipient = recipients[_msgSender()];
        require(
            block.timestamp >= _recipient.claimTime,
            "Claim: Locked for now"
        );
        require(_recipient.amount > 0, "Claim: Nothing to claim");

+       delete recipients[_msgSender()];
        
        SafeERC20.safeTransfer(soarToken, _msgSender(), _recipient.amount);
    }
```


#### Team Response: Resolved

--------------


### [C-2] Wrong decimals scaling of oracle price allows users to purchase SOAR from in `Minter.sol` almost for free

The Minter contract exposes a `mint()` function where users can purchase SOAR tokens, paying the corresponding value in USDC. 
To calculate the amount of USDC that should be charged, the Minter contract reads the price of soar in USD from `oracle.viewPriceInUSD()`, and then scales it up with the purchase token decimals. 

However, this scaling is wrong, as it multiplies times `decimals`, instead of `10 ** decimals`:

```javascript
function mint(uint256 _amountSOAR) external {
        require(_amountSOAR > 0, "Buying: cant buy 0 tokens");

        uint256 tokenPrice = oracle.viewPriceInUSD();
        require(tokenPrice > 0, "Buying: invalid price");

        uint256 totalPrice = (_amountSOAR *
            tokenPrice *
@>          ERC20Upgradeable(purchaseToken).decimals()) /
            1e18 /
            1e6;

        // ...
    }
```

#### Impact

The `totalPrice` charged to the buyer will be wrong by a factor of `x166666`.
So if the actual price charged to the buyer should be 1000 USDC (1000 * 1e6), users will only be charged 0.006 USDC (1000 * 6 USDC)
which is about x166666 times less than the actual price.

The first user to realize this would take the chance and purchase all available tokens in the contract's balance. 

#### Mitigation

```diff
    function mint(uint256 _amountSOAR) external {
        require(_amountSOAR > 0, "Buying: cant buy 0 tokens");

        uint256 tokenPrice = oracle.viewPriceInUSD();
        require(tokenPrice > 0, "Buying: invalid price");

        uint256 totalPrice = (_amountSOAR *
            tokenPrice *
-           ERC20Upgradeable(purchaseToken).decimals()) /
+           10 ** ERC20Upgradeable(purchaseToken).decimals()) /
            1e18 /
            1e6;

        // ...
    }
```

#### Team Response: Resolved

--------------

## High risk


### [H-1] The staking contract will become insolvent if `recoverNonLockedRewardTokens()` is called

The function `recoverNonLockedRewardTokens()` allows the owner to recover the reward tokens in balance that are not locked for rewards. 
If the `rewardsToken` is an ERC20, the accounting is handled properly, by excluding the `rewardTokensLocked` and the `totalStaked` from the amount that can be recovered. 

However, if the rewards token is ether (represented by `rewardsToken == address(0)`), the `rewardTokensLocked` is ignored, as the entire ether balance can be recovered:

```javascript
    function recoverNonLockedRewardTokens() external onlyOwner {
        uint256 nonLockedTokens;
        if (address(rewardsToken) != address(0)) {
            nonLockedTokens = address(stakingToken) != address(rewardsToken)
                ? rewardsToken.balanceOf(address(this)) - rewardTokensLocked
                : rewardsToken.balanceOf(address(this)) -
                    rewardTokensLocked -
                    totalStaked;

            rewardsToken.transfer(owner(), nonLockedTokens);
        } else {
@>          nonLockedTokens = address(this).balance;

            (bool sent, ) = payable(owner()).call{value: nonLockedTokens}("");
            require(sent, "Failed to send Ether");
        }
        emit RewardTokensRecovered(nonLockedTokens);
    }
```

This means that when recovering "non-locked-reward-tokens", the function will recover **all** reward tokens, leaving the staking contract insolvent and incapable of paying all users. 

#### Impact

If the `recoverNonLockedRewardTokens()` when ether is the reward token, the staking contract becomes insolvent, and all `claim()` transactions will revert with *Not enough ether in balance* error message. 

#### Proof of concept

TODO

#### Mitigation

Subtract the `rewardTokensLocked` before sending non-locked reward tokens:

```diff
    function recoverNonLockedRewardTokens() external onlyOwner {
        uint256 nonLockedTokens;
        if (address(rewardsToken) != address(0)) {
            nonLockedTokens = address(stakingToken) != address(rewardsToken)
                ? rewardsToken.balanceOf(address(this)) - rewardTokensLocked
                : rewardsToken.balanceOf(address(this)) -
                    rewardTokensLocked -
                    totalStaked;

            rewardsToken.transfer(owner(), nonLockedTokens);
        } else {
-           nonLockedTokens = address(this).balance;
+           nonLockedTokens = address(this).balance - rewardTokensLocked;

            (bool sent, ) = payable(owner()).call{value: nonLockedTokens}("");
            require(sent, "Failed to send Ether");
        }
        emit RewardTokensRecovered(nonLockedTokens);
    }
```

#### Team Response: Resolved

--------------











### [H-2] Lack of slippage protection in `Liquidity.swap()` can result in significant loss of value

The Liquidity contract can swap between the assets  `_tokenIn` and `_tokenOut`. However, some of the parameters of the swap are hardcoded: 
- `amountOutMinimum`: this protects against slippage, by declaring the minimum acceptable amount of `_tokenOut`. If set to 0, there is no slippage protection and a sandwich attack can steal 100% of the traded value, as we accept 0 as the amount out. 
- `sqrtPriceLimitX96`: another protection mechanism, this one to regulate the accepted price impact. 

```javascript
    function swap(
        address _tokenIn,
        address _tokenOut,
        uint256 amountIn,
        uint24 _fee
    ) external onlyOwner returns (uint256) {
        TransferHelper.safeApprove(_tokenIn, address(swapRouter), amountIn);

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _tokenIn,
                tokenOut: _tokenOut,
                fee: _fee,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amountIn,
@>              amountOutMinimum: 0, // no slippage protection
@>              sqrtPriceLimitX96: 0  // no price-impact protection
            });

        uint256 amountOut = ISwapRouter(swapRouter).exactInputSingle(params);
        return amountOut;
    }
```

By setting `amountOutMinimum = 0`, the slippage tolerance is hardcoded to 100%, meaning there is no slippage protection at all. 

Note that sandwich attacks usually affect DEX users who set a relaxed slippage tolerance (1%, 2%, etc). In those cases, the value that the MEV actor can extract is not so large: only a small percentage of the traded volume. However, in this case, the slippage tolerance is virtually 100%, so the frontrunner could steal close to 100% of the traded value if he had enough wealth to push the price enough in the sandwich. 

#### Impact

Due to a complete lack of slippage protection, the trades done with `Liquidity.swap()` can be sandwiched and a very significant portion of the traded value can be stolen. The magnitude of the loss will be determined by how wealthy the frontrunner is, as the slippage tolerance is hardcoded to 100%. 

#### Proof of concept

- The contract operator initiates a swap of 10.000 USDC for 100.000 SOAR using the `Liquidity.swap()` function
- A wealthy front-runner sandwiches the transaction, by purchasing first with 100.000 USDC. This increases the price of SOAR significantly
- The Liquidity.swap() transaction goes through, but the swap happens at a much higher price
- The front-runner back runs the transaction, selling all the purchased soar, receiving a net profit 
- The `Liquidity.swap()` suffers by receiving much less SOAR than he should. 

#### Mitigation

Enable extra input arguments that enable slippage and price impact protection. The caller of `swap()` is now responsible for configuring those parameters by calculating max slippage and price impact off-chain. 

```diff
    function swap(
        address _tokenIn,
        address _tokenOut,
        uint256 amountIn,
+       uint256 _amountOutMinimum,
+       uint160 _sqrtPriceLimitX96
        uint24 _fee
    ) external onlyOwner returns (uint256) {
        TransferHelper.safeApprove(_tokenIn, address(swapRouter), amountIn);

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _tokenIn,
                tokenOut: _tokenOut,
                fee: _fee,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amountIn,
-               amountOutMinimum: 0,
+               amountOutMinimum: _amountOutMinimum,
-               sqrtPriceLimitX96: 0
+               sqrtPriceLimitX96: _sqrtPriceLimitX96
            });

        uint256 amountOut = ISwapRouter(swapRouter).exactInputSingle(params);
        return amountOut;
    }
```

#### Team Response: Resolved

--------------









### [H-3] The Minter contract cannot guarantee solvency because purchases are allowed regardless of the SOAR tokens in the contract's balance

When users purchase tokens in the Minter contract, they spend USDC, hoping that they will be able to claim the same value in SOAR tokens, with a certain delay. However, the `Minter.mint()` function does not verify that there are enough tokens in balance to be purchased. 

If the `claim()` function is called when there are not enough SOAR tokens in balance, the transaction will revert:

```javascript
    function claim() external {
        Recipient memory _recipient = recipients[_msgSender()];
        require(
            block.timestamp >= _recipient.claimTime,
            "Claim: Locked for now"
        );
        require(_recipient.amount > 0, "Claim: Nothing to claim");

@>      SafeERC20.safeTransfer(soarToken, _msgSender(), _recipient.amount);
    }
```

The root cause is that there is no mechanism to control how many tokens a user can purchase. These controls should be based on the amount of tokens in balance (subtracting the ones that have been already purchased). 

Note that the `Minter.recoverERC20()` is also affected by this issue, as the admins can withdraw already purchased SOAR tokens. 

#### Impact

Inability for the protocol to control how many tokens are being purchased. 

Users are unable to claim their SOAR tokens if they are not in the contract's balance at the moment of calling `claim()`. 
This can happen in the following situations:
- The team forgets to fund the Minter contract with SOAR tokens
- There are no more SOAR tokens in circulation, and therefore the Minter contract cannot be funded (but users can still purchase)
- A user purchases more tokens than the remaining circulating supply

#### Proof of concept

- User purchases $10000 worth of SOAR
- The Minter contract only has 9000 worth of SOAR in the balance
- The locking period passes
- The user attempts to claim his $10000 worth of SOAR, but the transaction reverts

#### Mitigation

- A state variable should keep track of the amount of SOAR purchased. `reservedSoar`
- Any action that changes the balance of SOAR or the `reservedSoar`, (`mint()`, `claim()`, `recoverERC20()`), should check that the balance is enough to cover the `reservedSoar` after modifying the balance. 

```diff
+   // keeps track of how much SOAR has been purchased and already has an owner
+   uint256 reservedSoar;

    // ...

    function mint(uint256 _amountSOAR) external {
        require(_amountSOAR > 0, "Buying: cant buy 0 tokens");
+       require(reservedSoar + _amountSOAR<= soarToken.balanceOf(address(this)), "Minter: Insufficient SOAR balance");
        
        //...
    }

    function claim() external {
        Recipient memory _recipient = recipients[_msgSender()];
        require(
            block.timestamp >= _recipient.claimTime,
            "Claim: Locked for now"
        );
        
+       reservedSoar -= _recipient.amount;

        // ...

    }

    function recoverERC20(
        ERC20Upgradeable token,
        address _strategy
    ) external onlyOwner {
+       uint256 amount = token == soarToken ? 
+           token.balanceOf(address(this)) - reservedSoar : 
+           token.balanceOf(address(this));

        SafeERC20.safeTransfer(
            token,
            _strategy, 
-           token.balanceOf(address(this))
+           amount
        );
    }

```

#### Team Response: Resolved

-------------







### [H-4] Prices in Minter depend on a single oracle, which makes it vulnerable to price-manipulation attacks* (incomplete finding until oracle integration is finished)

The `Minter.mint()` function reads the SOAR price in USD from the `oracle`, which is a single instance of `UniswapV2Oracle.sol`. This constitutes a risk, as a single liquidity pool to be used as an oracle is vulnerable to price manipulation attacks. 

Luckily, the oracle uses a 1h-average price which makes the manipulation less likely, but it is still a single source of truth. 

EDIT: This issue has been paused because the `UniswapV2Oracle.sol` is undergoing some updates. This issue will be reviewed in the mitigation phase.

#### Impact
#### Proof of concept
#### Mitigation
#### Team Response
TBC

--------------









## Medium risk

### [M-1] Pending claims in the Minter will become blocked again if new purchases are made before claiming

In the Minter, there is a `lockingPeriod` which is a delay between the purchase time and the moment when the SOAR tokens become available. 
Each account has a single `lockingPeriod` which is overwritten every time a new purchase is made. 

Therefore, if a has purchased SOAR tokens but hasn't claimed them yet, and makes a new purchase before claiming, the originally purchased tokens will be subject to the new `lockingPeriod` again. 

```javascript
    function mint(uint256 _amountSOAR) external {
        require(_amountSOAR > 0, "Buying: cant buy 0 tokens");

        // ...

        recipients[_msgSender()].amount += _amountSOAR;
@>      recipients[_msgSender()].claimTime = block.timestamp + lockingPeriod;
    }

```

#### Impact

Users will experience unfair delays in claiming tokens if they make new purchases before claiming the already purchased tokens. 

#### Proof of concept

- Let's assume `lockingPeriod = 7 days`
- Bob purchases $10000 worth of SOAR
- Bob waits 10 days. He is entitled to claim his $10000 worth of SOAR now
- However, Bob makes a new purchase of $1000 before claiming
- Now Bob has to wait another 7 days before claiming the full $11000 worth of SOAR tokens

#### Mitigation

There are different approaches to mitigate this:
- Store each purchase in a different struct like `Purchase(amount, lockingPeriod)`, and allow each purchase to be claimable independently. Each user would have an array of Purchases instead of a single `amount` and `lockingPeriod`. This brings a bit more complexity to the UI
- Don't allow new purchases if previously purchased amounts are available for claim. This will fail to protect users in the scenario of purchasing before the current `lockingPeriod` is over. 
- Be very explicit in the front-end about the consequences of a given purchase (and warn them to claim before making new purchases)

#### Team Response: Resolved (in frontend)

No changes were applied to the smart contracts, but the team will show warnings in the frontend so that the user is very aware of the implications of purchasing again. 

--------------

### [M-2] The `Soar.sol` and `Liquidity.sol` contracts can receive ETH, but cannot send it so it will get stuck in the contract

The `Soar` and `Liquidity` contracts implement the payable `receive()` function, which allows ether transfers into the contract:

```javascript
    receive() external payable {
```

However, once the ether arrives at both contracts, there is no way for the ether to leave them. Therefore, any ether arriving will be forever lost. 

Note: the `Soar.openTrading()` function transfers ether to the Uniswap pool. However, this ether doesn't come from the Soar token contract, but from the `msg.value` of the caller of `openTrading()`. So the ether in the contract balance is still stuck. 

#### Impact

Any ether received by the `Soar.sol` or `Liquidity.sol` contracts will be forever lost, stuck in the contract's balance.

#### Mitigation

I would remove the `receive()` function from both contracts as there is no need to handle ether, besides the `Soar.openTrading()` function, which is already payable.

```diff
contract Soar is ERC20, Ownable {
    // ...
-   receive() external payable {
```

```diff
contract Liquidity is IERC721Receiver, Ownable {
    // ...
-   receive() external payable {
```

#### Team Response: Resolved

--------------





### [M-3] `SoarStaking.setRewards()` can revert under certain circumstances, blocking the admin from setting new reward periods due to a wrong order of operands

In the `setRewards()` function, there is an operation to update the tokens that are reserved for rewards, `rewardTokensLocked`:

```javascript
    function setRewards(
        uint256 _rewardPerBlock,
        uint256 _startingBlock,
        uint256 _blocksAmount
    ) external onlyOwner updateReward(address(0)) {
        uint256 unlockedTokens = _getFutureRewardTokens();

        rewardPerBlock = _rewardPerBlock;
        firstBlockWithReward = _startingBlock;
        lastBlockWithReward = firstBlockWithReward + _blocksAmount - 1;

        uint256 lockedTokens = _getFutureRewardTokens();
        uint256 rewardBalance;
        if (address(rewardsToken) != address(0)) {
            rewardBalance = address(stakingToken) != address(rewardsToken)
                ? rewardsToken.balanceOf(address(this))
                : stakingToken.balanceOf(address(this)) - totalStaked;
        } else {
            rewardBalance = address(this).balance;
        }

        // this subtraction can revert if rewardTokensLocked < unlockedTokens
@>      rewardTokensLocked = rewardTokensLocked - unlockedTokens + lockedTokens;

        // ...
    }
```

However, the subtraction can revert due to underflow if the current `rewardTokensLocked` is smaller than the `unlockedTokens`. This is possible because the unlocked tokens are proportional to the remaining blocks left in the current rewards period:

```javascript
    function _getFutureRewardTokens() internal view returns (uint256) {
@>      return _calculateBlocksLeft() * rewardPerBlock;
    }
```

#### Impact

When the current `rewardTokensLocked` is greater than the `unlockedTokens`, the `setRewards()` function will revert, stopping the contract admin from setting a new reward period.  

Even though the necessary condition `rewardTokensLocked < unlockedTokens`  is an edge case and will not normally occur, the solution is trivial, so it is an easy win.

#### Mitigation

Invert the order of operands so that the subtraction is only done at the end:

```diff
    function setRewards(
        uint256 _rewardPerBlock,
        uint256 _startingBlock,
        uint256 _blocksAmount
    ) external onlyOwner updateReward(address(0)) {
        uint256 unlockedTokens = _getFutureRewardTokens();

        rewardPerBlock = _rewardPerBlock;
        firstBlockWithReward = _startingBlock;
        lastBlockWithReward = firstBlockWithReward + _blocksAmount - 1;

        uint256 lockedTokens = _getFutureRewardTokens();
        uint256 rewardBalance;
        if (address(rewardsToken) != address(0)) {
            rewardBalance = address(stakingToken) != address(rewardsToken)
                ? rewardsToken.balanceOf(address(this))
                : stakingToken.balanceOf(address(this)) - totalStaked;
        } else {
            rewardBalance = address(this).balance;
        }

        // this subtraction can revert if rewardTokensLocked < unlockedTokens
-       rewardTokensLocked = rewardTokensLocked - unlockedTokens + lockedTokens;
+       rewardTokensLocked = rewardTokensLocked + lockedTokens - unlockedTokens;

        // ...
    }
```

#### Team Response: Resolved

--------------










### [M-4] The deadline of `Liquidity.mintNewPosition()` has no effect

When minting a new position via the `Liquidity` contract, the deadline is hardcoded to `block.timestamp + 100`:

```javascript
    function mintNewPosition(
        address token0,
        address token1,
        uint24 poolFee,
        int24 tickLower,
        int24 tickUpper,
        uint256 amount0ToMint,
        uint256 amount1ToMint
    )
        external
        onlyOwner
        returns (
            uint256 tokenId,
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        )
    {
        // ...

        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams({
                token0: token0,
                token1: token1,
                fee: poolFee,
                tickLower: tickLower,
                tickUpper: tickUpper,
                amount0Desired: amount0ToMint,
                amount1Desired: amount1ToMint,
                amount0Min: 0,
                amount1Min: 0,
                recipient: address(this),
@>              deadline: block.timestamp + 100
            });

        // ...

    }
```

That deadline has no effect, because the `block.timestamp` is set by the miner in whatever block this transaction is mined, so regardless of when the transaction is minted, the deadline will always be met. 

The developer intends to establish a max time in which this position can be created, and therefore, the deadline should be *the timestamp when the transaction is **broadcasted** + 100 seconds*, instead of *timestamp when the transaction is **mined** + 100 seconds*. The way to do so is to pass an extra argument to the function, with the deadline calculated off-chain. 

#### Impact

The hardcoded deadline will have no effect, and the position can be minted at any later time with no restrictions. 

#### Proof of concept

- The contract admin calls `Liquidity.mintNewPosition()` at 16:00. The intention is that the tx is only valid if executed before 16:00 + 100 seconds. 
- 2 h pass, `block.timestamp` is now 18:00. The transaction is executed, and the configured to `block.timestamp` + 100 seconds, which is 18:00+100 instead of 16:00+100 (as the dev intended). Therefore, the transaction went through, and the deadline was pretty much useless. 

#### Mitigation

```diff
    function mintNewPosition(
        address token0,
        address token1,
        uint24 poolFee,
        int24 tickLower,
        int24 tickUpper,
        uint256 amount0ToMint,
        uint256 amount1ToMint
+       uint256 deadline
    )
        external
        onlyOwner
        returns (
            uint256 tokenId,
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        )
    {
        // ...

        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams({
                token0: token0,
                token1: token1,
                fee: poolFee,
                tickLower: tickLower,
                tickUpper: tickUpper,
                amount0Desired: amount0ToMint,
                amount1Desired: amount1ToMint,
                amount0Min: 0,
                amount1Min: 0,
                recipient: address(this),
-               deadline: block.timestamp + 100
+               deadline: deadline
            });

        // ...

    }
```

#### Team Response: Resolved

------------------





### [M-5] Lack of input validation in tax setter functions can halt SOAR token transfers

In the Soar contract, the `taxFee` state variable is used to subtract the fee from the `amount` to be transferred. The `taxFee` is expected to be a percentage, expressed with `MULTIPLIER` as a base (so if `taxFee == MULTIPLIER`, then there is 100% tax). 

If the `taxFee > MULTIPLIER`, then the `amount -= _tax` will revert with an underflow:

```javascript
    function _transferWithTax(
        address from,
        address to,
        uint256 amount
    ) internal {
        if (isExcludedFromFee[from] || isExcludedFromFee[to]) {
            _transfer(from, to, amount);
            return;
        }

        if (_shouldTakeTax(to) || _shouldTakeTax(from)) {
@>          uint256 _tax = (amount * taxFee) / MULTIPLIER;
@>          amount -= _tax;
            _transfer(from, taxReceiver, _tax);
        }
        // ...
```

However, there is no input validation in `setTaxFee()`, which means that the `taxFee` can potentially be set higher than `MULTIPLIER`.
The probability of this happening is low, as it requires a mistake from the contract owner or compromised private keys, but the implications are that all taxed transactions (buys and sells) will revert. This will effectively halt all trading activity of the SOAR token. 

```javascript
    function setTaxFee(uint256 _tax) external onlyOwner {
        taxFee = _tax;
    }
```

#### Impact

If the `taxFee` is set to a higher value than `MULTIPLIER`, trading operations will be halted. 

#### Mitigation

```diff
    function setTaxFee(uint256 _tax) external onlyOwner {
+       require(_tax < MULTIPLIER, "_tax is too high");
        taxFee = _tax;
    }
```

#### Team Response: Resolved

Resolved by with the same fix as for M-6. 

--------------










### [M-6] Lack of input validation in tax setter functions allows the contract owner set as high tax as 100%

In the Soar contract, the `taxFee` state variable is used to subtract the fee from the `amount` to be transferred. The `taxFee` is a percentage of `MULTIPLIER` (if `taxFee == MULTIPLIER`, then there is 100% tax). 

The `setTaxFee()` function has no limits, and the owner is allowed to set a tax as high as 100%:
```javascript
    function setTaxFee(uint256 _tax) external onlyOwner {
        taxFee = _tax;
    }
```

#### Impact

The owner can set the trading tax to as high as 100%. 
This is not likely, as it requires either a malicious owner or compromised private keys, but it is worth mitigating it.

#### Mitigation

Establish a maximum tax and don't allow a higher `taxFee` than that:

```diff
+   uint256 public constant MAX_TAX = MULTIPLIER / 10; // MAX TAX = 10%

    function setTaxFee(uint256 _tax) external onlyOwner {
+       require(_tax < MAX_TAX, "_tax is too high");
        taxFee = _tax;
    }
```
#### Team Response: Resolved

--------------













### [M-7] Lack of slippage protection in liquidity management functions can result in lost value for the protocol when adding/removing liquidity

The functions `increaseLiquidityCurrentRange()` and `decreaseLiquidity()` add and remove liquidity **in the current range**. However, they are performed without any slippage protection, i.e., without any control over how much of each token is deposited/removed. 

```javascript

    function increaseLiquidityCurrentRange(
        uint256 tokenId,
        uint256 amountAdd0,
        uint256 amountAdd1
    )
        external
        onlyOwner
        returns (uint128 liquidity, uint256 amount0, uint256 amount1)
    {
        TransferHelper.safeApprove(
            deposits[tokenId].token0,
            address(nonfungiblePositionManager),
            amountAdd0
        );
        TransferHelper.safeApprove(
            deposits[tokenId].token1,
            address(nonfungiblePositionManager),
            amountAdd1
        );

        INonfungiblePositionManager.IncreaseLiquidityParams
            memory params = INonfungiblePositionManager
                .IncreaseLiquidityParams({
                    tokenId: tokenId,
                    amount0Desired: amountAdd0,
                    amount1Desired: amountAdd1,
@>                  amount0Min: 0, // no slippage protection
@>                  amount1Min: 0, // no slippage protection
                    deadline: block.timestamp
                });

        (liquidity, amount0, amount1) = nonfungiblePositionManager
            .increaseLiquidity(params);
    }
```

```javascript
    function decreaseLiquidity(
        uint256 tokenId,
        uint128 _amount
    ) external onlyOwner returns (uint256 amount0, uint256 amount1) {
        // get liquidity data for tokenId
        uint128 liquidity = deposits[tokenId].liquidity;

        // amount0Min and amount1Min are price slippage checks
        // if the amount received after burning is not greater than these minimums, transaction will fail
        INonfungiblePositionManager.DecreaseLiquidityParams
            memory params = INonfungiblePositionManager
                .DecreaseLiquidityParams({
                    tokenId: tokenId,
                    liquidity: _amount,
@>                  amount0Min: 0,  // no slippage protection 
@>                  amount1Min: 0,  // no slippage protection 
                    deadline: block.timestamp
                });

        (amount0, amount1) = nonfungiblePositionManager.decreaseLiquidity(
            params
        );
    }

```

Therefore, the operations for adding/removing liquidity can be frontrun, changing the *current range*. 
- In liquidity additions, the liquidity will be added to a different range than the current one, which in terms of strategy might not be ideal
- In liquidity removals, the position might not have liquidity in the new *current range* after the frontrun, leading to a revert, and therefore enabling DOS attack to this functionality. 

#### Impact

Due to the lack of slippage protection in liquidity additions and removals, MEV actors can force liquidity to be added to the wrong range, or deny liquidity removals. 

The likelihood is low, because it won't usually be a profitable attack, but solving it is trivial, so it is recommended. The team can still pass 0 to the newly added arguments, but still have the possibility of using the protection later on. 

#### Mitigation

Allow input arguments to regulate slippage in the liquidity operations. The accepted values should be calculated by off-chain components

```diff
    function decreaseLiquidity(
        uint256 tokenId,
        uint128 _amount,
+       uint256 _amount0Min,
+       uint256 _amount1Min
    ) external onlyOwner returns (uint256 amount0, uint256 amount1) {
        // get liquidity data for tokenId
        uint128 liquidity = deposits[tokenId].liquidity;

        // amount0Min and amount1Min are price slippage checks
        // if the amount received after burning is not greater than these minimums, transaction will fail
        INonfungiblePositionManager.DecreaseLiquidityParams
            memory params = INonfungiblePositionManager
                .DecreaseLiquidityParams({
                    tokenId: tokenId,
                    liquidity: _amount,
-                   amount0Min: 0,
-                   amount1Min: 0,
+                   amount0Min: _amount0Min,
+                   amount1Min: _amount1Min,
                    deadline: block.timestamp
                });

        (amount0, amount1) = nonfungiblePositionManager.decreaseLiquidity(
            params
        );
    }
```

```diff
    function increaseLiquidityCurrentRange(
        uint256 tokenId,
        uint256 amountAdd0,
        uint256 amountAdd1,
+       uint256 _amount0Min,
+       uint256 _amount1Min
    )
        external
        onlyOwner
        returns (uint128 liquidity, uint256 amount0, uint256 amount1)
    {
        TransferHelper.safeApprove(
            deposits[tokenId].token0,
            address(nonfungiblePositionManager),
            amountAdd0
        );
        TransferHelper.safeApprove(
            deposits[tokenId].token1,
            address(nonfungiblePositionManager),
            amountAdd1
        );

        INonfungiblePositionManager.IncreaseLiquidityParams
            memory params = INonfungiblePositionManager
                .IncreaseLiquidityParams({
                    tokenId: tokenId,
                    amount0Desired: amountAdd0,
                    amount1Desired: amountAdd1,
-                   amount0Min: 0,
-                   amount1Min: 0,
+                   amount0Min: _amount0Min,
+                   amount1Min: _amount1Min,
                    deadline: block.timestamp
                });

        (liquidity, amount0, amount1) = nonfungiblePositionManager
            .increaseLiquidity(params);
    }

```

#### Team Response: Resolved

--------------












## Low risk

### [L-1] Soar.OpenTrading could be frontrun to alter the launching price

The starting price in the UniswapV2 pool is determined by the ratio SOAR:ETH when the liquidity is added in `openTrading()`. 
This initial ratio is determined by the amount of ETHER and SOAR in the `addLiquidityETH()` function. These amounts are:
- ETHER: the `msg.value` when executing to `openTrading()`
- SOAR: the balance of SOAR tokens in the SOAR contract when calling `openTrading()`

```javascript
    function openTrading() external payable onlyOwner {
        // add liquidity
        ROUTER.addLiquidityETH{value: msg.value}(
            address(this),
            balanceOf(address(this)),
            0,
            0,
            msg.sender,
            block.timestamp
        );

        IUniswapV2Pair pair = IUniswapV2Pair(
            IUniswapV2Factory(ROUTER.factory()).getPair(
                address(this),
                ROUTER.WETH()
            )
        );

        setMaxWallet(INITIAL_SUPPLY / 100);
        excludeFromMaxTx(address(pair), true);
    }
```

The issue is that potential donations to the SOAR contract right before liquidity is added would decrease the ratio ETH:SOAR, diminishing the starting price in the pool. However, such a scenario is unlikely due to the following reasons:
- It is probably a non-profitable attack because the donation would need to be large to significantly affect the price
- Presumably, nobody besides the `owner` and seed investors will have SOAR tokens before adding the liquidity

However, if there is such a wallet, with a large amount of tokens, and incentives to have a smaller starting price, performing the attack is rather trivial: frontrun the `openTrading()` function by donating a large amount of SOAR tokens. 

#### Impact

The launch price in the liquidity pool can be diminished by an interested frontrunner that has enough SOAR tokens to be donated for such a purpose. 

#### Proof of concept

- SOAR token is distributed among seed investors
- SOAR contract has 0 SOAR tokens in the balance
- admin transfers 100.000 SOAR tokens to the SOAR contract
- admin calls `openTrading()` with msg.value = 100 ether, expecting that the starting price is `1 ETH : 1000 SOAR` so, `1 SOAR = 0.001 ETHER`. 
- however, a malicious seed investor frontruns `openTrading()` donating 10.000 SOAR tokens, changing the ratio to `1 ETH = 1100 SOAR`, `1 SOAR = 0.000909091 ETHER`

#### Mitigation

Instead of relying on the `balanceOf(address(this))` to decide how much SOAR is deployed as liquidity, enable an input argument, and transfer the amount from the `msg.sender` in the same call as `openTrading()`. This will give much more control of the exact liquidity that is added.

```diff
-   function openTrading() external payable onlyOwner {
+   function openTrading(uint256 amountSoar) external payable onlyOwner {

+       transferFrom(msg.sender, address(this), amountSoar);

        ROUTER.addLiquidityETH{value: msg.value}(
            address(this), 
-           balanceOf(address(this)),
+           amountSoar,
            0,
            0,
            msg.sender,
            block.timestamp
        );

        // ...

    }
```

#### Team response: Resolved







### [L-2] Liquidity addition can be frontrun by another SOAR holder setting the launch price

#### Impact

A variant of the previous attack is that another holder of SOAR tokens can frontrun `openTrading()` by adding liquidity at a different ratio, also affecting the launch price. A particularly annoying attack would be if the liquidity is added with such a high price that the aspect of the graph right after the `openTrading()` is a massive price dump in the first block. 

#### Mitigation

The best mitigation is to ensure nobody has SOAR tokens before adding liquidity. Therefore the Minter contract should not be launched until that happens, and the TGE should not distribute SOAR tokens until liquidity is added. 

#### Team Response: Resolved

--------------








### [L-3] Lack of input validation in `SoarStaking.setRewards()` allows configuring an already finished period

When a user stakes, a requirement is that the rewards period is not over yet, to save users from staking for no rewards:

```javascript
    function stake(
        uint256 _amount
    ) external whenNotPaused nonReentrant updateReward(msg.sender) {
        require(_amount > 0, "Stake: can't stake 0");
        require(
@>          block.number < lastBlockWithReward,
            "Stake: staking  period is over"
        );
        // ...
```

However, when the rewards period is set, there is no validation that the resulting `lastBlockWithReward` is valid:

#### Impact

If the rewards are wrongly configured, users won't be able to stake, because the transaction will revert. Moreover, all configured rewards would be delivered all at once, without vesting.

The issue has a low likelihood as it requires a mistake by the contract operator, but it is an easy win to fix it.

#### Mitigation

```diff

    function setRewards(
        uint256 _rewardPerBlock,
        uint256 _startingBlock,
        uint256 _blocksAmount
    ) external onlyOwner updateReward(address(0)) {
        uint256 unlockedTokens = _getFutureRewardTokens();

        rewardPerBlock = _rewardPerBlock;
        firstBlockWithReward = _startingBlock;
        lastBlockWithReward = firstBlockWithReward + _blocksAmount - 1;

+       require(block.number < firstBlockWithReward + _blocksAmount - 1, "reward period already finished");        
```

#### Team Response: Acknowledged

--------------











### [L-4] Some DEX could evade the trading tax if the pool contract does not expose `token0()` and `token1()` functions

On the SOAR token, normal transfers are not taxed, only trading transactions (buys or sells). The way the SOAR contract distinguishes between a trade and a normal transfer is by checking if the sender or receiver is a DEX pool. This is done with function `_shouldTakeTax()`, which checks if the `token0()` or `token1()` is the SOAR token. For normal transfers where none of them have such function, no tax will be applied:

```javascript
    function _shouldTakeTax(address to) internal returns (bool) {
        address token0 = _getPoolToken(
            to,
            "token0()",
            IUniswapV2Pair(to).token0
        );
        address token1 = _getPoolToken(
            to,
            "token1()",
            IUniswapV2Pair(to).token1
        );

        return token0 == address(this) || token1 == address(this);
    }
```

#### Impact

This mechanism will only work for DEX pools that expose `token0()` and `token1()` view functions. As an example, UniswapV2, UniswapV3 and Sushiswap do expose such functions, and therefore this methodology will work. However, I'm unaware if all DEX pools do that as well. 

#### Mitigation

A simpler alternative is to keep track of the DEX pool addresses in a mapping, and blacklist DEX pools as they become relevant based on trading volume with an onlyOwner function. However, I don't think it is such an issue, because most of the trading volume will be where the team adds liquidity, and that is UniswapV2. 

Note that this approach consumes way less gas as it only requires a single storage read, instead of multiple external calls with multiple storage reads. 

#### Team Response: Acknowledged

--------










### [L-5] Lack of deadlines allows Liquidity management operations to be postponed indefinitely

Some liquidity management operations have `deadline = block.tiemstamp`. This is equivalent to having no deadline. 

```javascript

    function increaseLiquidityCurrentRange(
        uint256 tokenId,
        uint256 amountAdd0,
        uint256 amountAdd1
    )
        external
        onlyOwner
        returns (uint128 liquidity, uint256 amount0, uint256 amount1)
    {

        // ...

        INonfungiblePositionManager.IncreaseLiquidityParams
            memory params = INonfungiblePositionManager
                .IncreaseLiquidityParams({
                    tokenId: tokenId,
                    amount0Desired: amountAdd0,
                    amount1Desired: amountAdd1,
                    amount0Min: 0,
                    amount1Min: 0,
@>                  deadline: block.timestamp
                });

        (liquidity, amount0, amount1) = nonfungiblePositionManager
            .increaseLiquidity(params);
    }
```

#### Impact

Without a deadline, transactions can be executed at a much later time, perhaps not aligning with the strategy of the contract operator. 

#### Mitigation

Pass a deadline as an input argument if the time of execution is important. The issue is present in the following functions:
- `increaseLiquidityCurrentRange()` 
- `decreaseLiquidity()`
- `swap()`

Below is an example change for one of the functions above:

```diff
    function increaseLiquidityCurrentRange(
        uint256 tokenId,
        uint256 amountAdd0,
        uint256 amountAdd1,
        uint256 deadline
    )
        external
        onlyOwner
        returns (uint128 liquidity, uint256 amount0, uint256 amount1)
    {

        // ...

        INonfungiblePositionManager.IncreaseLiquidityParams
            memory params = INonfungiblePositionManager
                .IncreaseLiquidityParams({
                    tokenId: tokenId,
                    amount0Desired: amountAdd0,
                    amount1Desired: amountAdd1,
                    amount0Min: 0,
                    amount1Min: 0,
-                   deadline: block.timestamp
+                   deadline: deadline
                });

        (liquidity, amount0, amount1) = nonfungiblePositionManager
            .increaseLiquidity(params);
    }

```

#### Team Response: Resolved

The suggested mitigation was applied to the two mentioned liquidity management functions. 
However, the auditor missed to mention the same issue in the `mintNewPosition()` function.

Edit: Fully resolved, including the `mintNewPosition()` function. 

--------------







### [L-6] In the Minter contract, `totalPrice` can be 0 even if `tokenPrice` is not 0

The Minter contract requires that `tokenPrice > 0` when reading from the oracle. However, due to rounding issues, the `totalPrice` could still result in being zero, especially for tokens with few decimal points like USDC. 

The likelihood is low, and it is most likely not profitable accounting for gas, but the solution is trivial, so it is an easy win.

```javascript
    function mint(uint256 _amountSOAR) external {
        require(_amountSOAR > 0, "Buying: cant buy 0 tokens");

        uint256 tokenPrice = oracle.viewPriceInUSD();
@>      require(tokenPrice > 0, "Buying: invalid price");

        // ...
    }
```

#### Impact

The `totalPrice` asked could be zero when purchasing SOAR tokens in the Minter contract.

#### Mitigation

Apply the requirement to `totalPrice` instead of `tokenPrice`:

```diff
    function mint(uint256 _amountSOAR) external {
        require(_amountSOAR > 0, "Buying: cant buy 0 tokens");

        uint256 tokenPrice = oracle.viewPriceInUSD();
-       require(tokenPrice > 0, "Buying: invalid price");

        uint256 totalPrice = (_amountSOAR *
            tokenPrice *
            ERC20Upgradeable(purchaseToken).decimals()) /
            1e18 /
            1e6;

+       require(totalPrice > 0, "Buying: invalid totalPrice");

        // ...
    }
```

#### Team Response: Resolved

--------------

### [L-7] Some state-changing functions do not emit events

It is a best practice to emit events in all external/public state-changing functions. 

Affected functions:
- `Soar.setTaxReceiver()`
- `Soar.setTaxFee()`
- `Soar.setMaxWallet()`
- `Soar.excludeFromMaxTx()`

#### Team Response: Resolved

--------------










### [L-8] Soar.openTrading() can be called before the `taxReceiver` is set, sending fees to the zero address

When `openTrading()` is called, there is no requirement so that the `taxReceiver` has already been set. All fees from trades until `taxReceiver` is set will end in the zero address. 

#### Impact

Potential loss of trading fees if `taxReceiver` is not properly configured. Requires a mistake of the contract operator.
It is an easy fix, so an easy win. 

#### Mitigation

```diff
    function openTrading() external payable onlyOwner {
+       require(taxReceiver != address(0), "taxReceiver not set");

        // ...
    }
```
#### Team Response: Resolved

--------------




### [L-9] Potential reentrancy attack as the state is modified after an external call in `SoarStaking.getRewards()`

The `getReward()` function updates the storage variable `rewardTokensLocked` after making an external call:

```javascript
    function getReward() public nonReentrant updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;

            if (address(rewardsToken) != address(0)) {
                rewardsToken.transfer(msg.sender, reward);
            } else {
@>              (bool sent, ) = payable(msg.sender).call{value: reward}("");
                require(sent, "Failed to send Ether");
            }

@>          rewardTokensLocked = rewardTokensLocked - reward;

            emit RewardPaid(msg.sender, reward);
        }
    }
```

This is generally bad practice and heavily non-recommended as it is an open door for reentrancy attacks. The `nonReentrant` modifier only acts at a contract level, but if other functions read the state from this contract, it would be possible to perform a *cross-contract reentrancy* or a *read-only reentrancy*. 

Luckily, there is no such risk in this case, because the `rewardTokensLocked` is not used outside this contract. However, imagine a scenario in which other contracts from the protocol read this variable for a critical calculation. It would be possible for the attacker to call `getReward()`, and in the external call made to `msg.sender`, he could make another call where `rewardTokensLocked` was read, **before the state was updated** in the last line of `getReward()`. 

#### Impact

A valid attack path has not been found. So low risk. However, following the [CEI pattern](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html) is the true protection against re-entrancy attacks. 

#### Mitigation

Never make external calls before all state variables of the current contract have been updated. In this particular case, update `rewardTokensLocked` before the external call. 
```diff
    function getReward() public nonReentrant updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;

+           rewardTokensLocked = rewardTokensLocked - reward;

            if (address(rewardsToken) != address(0)) {
                rewardsToken.transfer(msg.sender, reward);
            } else {
                (bool sent, ) = payable(msg.sender).call{value: reward}("");
                require(sent, "Failed to send Ether");
            }

-           rewardTokensLocked = rewardTokensLocked - reward;

            emit RewardPaid(msg.sender, reward);
        }
    }
```
--------------







### [L-10] Soar token might have integration issues with other protocols because contract can make incoming/outgoing transfers revert

Every token transfer that is not excluded from fees calls the internal function `_shouldTakeTax()`:

```javascript
    function _transferWithTax(
        address from,
        address to,
        uint256 amount
    ) internal {
        if (isExcludedFromFee[from] || isExcludedFromFee[to]) {
            _transfer(from, to, amount);
            return;
        }

@>      if (_shouldTakeTax(to) || _shouldTakeTax(from)) {
            uint256 _tax = (amount * taxFee) / MULTIPLIER;
            amount -= _tax;
            _transfer(from, taxReceiver, _tax);
        }
        // ...
```

which performs an external call to both sender and receiver (`from` and `to`) addresses, which are arbitrary addresses that could act maliciously:

```javascript
    function _shouldTakeTax(address to) internal returns (bool) {
        address token0 = _getPoolToken(
            to,
            "token0()",
            IUniswapV2Pair(to).token0
        );
        address token1 = _getPoolToken(
            to,
            "token1()",
            IUniswapV2Pair(to).token1
        );

        return token0 == address(this) || token1 == address(this);
    }
```

Note that the third input argument of `_getPoolToken()` is a `view` function. This means that if a state change is made when calling `token0()`, the whole transaction will revert with a message like *state change not allowed during static calls*:

```javascript
    function _getPoolToken(
        address pool,
        string memory signature,
@>      function() external view returns (address) getter
    ) internal returns (address) {
        // ...
    }
```

This means, that if `from` or `to` expose a malicious function called `token0()` or `token1()` that attempts a state change (of its own state for instance), the whole token transfer will revert when attempting to transfer to or from that address. Note that this is true regardless if the transaction is a trade or a transfer because the external call is precisely made to discern if it is a trade or a transfer.

#### Impact

When integrating in other protocols that attempt to move tokens for a certain account, this account can block their transfers if it is beneficial for them. This becomes especially relevant in the case of liquidations, which a malicious account can block.

Note that this could also affect other future SOAR contracts that integrate with this SOAR token contract, not only lending/borrowing. But the existing contracts do not have this issue.  

#### Mitigation

A mapping of blacklisted pool addresses to decide if trading tax should be applicable or not would not have this issue. However, if the team has no intentions of integrating with lending/borrowing platforms, this issue is less concerning. 

#### Team Response: Acknowledged

--------------











## Gas optimizations




### [G-1] The `updateReward()` modifier reads `rewardPerTokenStored` storage variable multiple times 

In the modifier, the `rewardPerTokenStored` is updated with the output from `rewardPerToken()`. Then, the `earned()` function will call again `rewardPerToken()` even though the value of `rewardPerTokenStored` has just been updated, incurring extra unnecessary gas costs. 

```javascript
    modifier updateReward(address account) {
@>      rewardPerTokenStored = rewardPerToken();
        lastUpdateBlock = block.number;
        if (account != address(0)) {
@>          rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

    function earned(address _account) public view returns (uint256) {
@>      uint256 rewardsDifference = rewardPerToken() -
            userRewardPerTokenPaid[_account];
        uint256 newlyAccumulated = (staked[_account] * (rewardsDifference)) /
            1e18;
        return rewards[_account] + newlyAccumulated;
    }

```

#### Optimization

Cache the output from `rewardPerToken()` in memory, and pass it as an argument to `earned()`:

```diff
    modifier updateReward(address account) {
-        rewardPerTokenStored = rewardPerToken();
+       _rewardPerToken = rewardPerToken();
+       rewardPerTokenStored = _rewardPerToken;
        lastUpdateBlock = block.number;
        if (account != address(0)) {
-           rewards[account] = earned(account);
+           rewards[account] = earned(account, _rewardPerToken);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

-   function earned(address _account) public view returns (uint256) {
+   function earned(address _account, uint256 _rewardPerToken) public view returns (uint256) {
-       uint256 rewardsDifference = rewardPerToken() -
+       uint256 rewardsDifference = _rewardPerToken_ -
            userRewardPerTokenPaid[_account];
        uint256 newlyAccumulated = (staked[_account] * (rewardsDifference)) /
            1e18;
        return rewards[_account] + newlyAccumulated;
    }

```
--------------






### [G-2] Soar._shouldTakeTax() makes multiple external calls and storage reads, making the token transfers unnecessarily expensive

```javascript
    function _shouldTakeTax(address to) internal returns (bool) {
        address token0 = _getPoolToken(
            to,
            "token0()",
            IUniswapV2Pair(to).token0
        );
        address token1 = _getPoolToken(
            to,
            "token1()",
            IUniswapV2Pair(to).token1
        );

        return token0 == address(this) || token1 == address(this);
    }
```

#### Optimization

Having a blacklist of pool addresses, that is configured via `onlyOwner` functions will require a single storage read for every token trade, making the trades significantly cheaper in terms of gas. 

##### Team response: Resolved

A hybrid solution was implemented. As soon as `_shouldTakeTax()` function returns true, that is stored in a mapping. This will save gas for future external calls in all future trades with that DEX. Normal transfers will suffer the extra gas costs of the external calls, but trades are expected to be the majority of transfer transactions, so it is a good compromise. 

--------------










### [G-3] `Liquidity.createNewPosition()` makes unnecessary external calls to read values that are already in memory

There is an internal function called `_createDeposit()` which performs an external call to the `nonfungiblePositionManager` to read certain information about a particular NFT position (`liquidity`,`token0`,`token1`). These parameters are then stored in the `deposits` mapping:

```javascript
    function _createDeposit(uint256 tokenId) internal {
        (
            ,
            ,
            address token0,
            address token1,
            ,
            ,
            ,
            uint128 liquidity,
            ,
            ,
            ,

@>      ) = nonfungiblePositionManager.positions(tokenId);

        // set the owner and data for position
        // operator is msg.sender
@>      deposits[tokenId] = Deposit({
@>          liquidity: liquidity,
@>          token0: token0,
@>          token1: token1
        });
    }

```

The internal function `_createDeposit()` is only called in one place: when a new position is minted:

```javascript
    function mintNewPosition(
        address token0,
        address token1,
        uint24 poolFee,
        int24 tickLower,
        int24 tickUpper,
        uint256 amount0ToMint,
        uint256 amount1ToMint
    )
        external
        onlyOwner
        returns (
            uint256 tokenId,
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        )
    {

        // ...

        // Note that the pool defined by DAI/USDC and fee tier 0.3% must already be created and initialized in order to mint
@>      (tokenId, liquidity, amount0, amount1) = nonfungiblePositionManager
            .mint(params);

        // Create a deposit
@>      _createDeposit(tokenId);

        // ...

    }
```

The redundancy here is that the information read in the external call to the `nonfungiblePositionManager` is already in memory, so the external call is completely unnecessary. 

#### Optimization

The `deposits` mapping can be filled without the need for an extra external call:

```diff
    function mintNewPosition(
        address token0,
        address token1,
        uint24 poolFee,
        int24 tickLower,
        int24 tickUpper,
        uint256 amount0ToMint,
        uint256 amount1ToMint
    )
        external
        onlyOwner
        returns (
            uint256 tokenId,
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        )
    {

        // ...

        // Note that the pool defined by DAI/USDC and fee tier 0.3% must already be created and initialized in order to mint
        (tokenId, liquidity, amount0, amount1) = nonfungiblePositionManager
            .mint(params);

        // Create a deposit
-       _createDeposit(tokenId);
+       deposits[tokenId] = Deposit({
+           liquidity: liquidity,
+           token0: token0,
+           token1: token1
+       });

        // ...

    }
```

Then, the `_createDeposit()` can be completely removed as it is only used there:

```diff
-   function _createDeposit(uint256 tokenId) internal {
-       (
-           ,
-           ,
-           address token0,
-           address token1,
-           ,
-           ,
-           ,
-           uint128 liquidity,
-           ,
-           ,
-           ,
-
-       ) = nonfungiblePositionManager.positions(tokenId);
-
-       // set the owner and data for position
-       // operator is msg.sender
-       deposits[tokenId] = Deposit({
-           liquidity: liquidity,
-           token0: token0,
-           token1: token1
-       });
-   }

```

### [G-4] Unused variable in `Liquidity.decreaseLiquidity()` wastes gas

The `liquidity` field is read from the `deposits` mapping (storage) but it is not used. 

#### Optimization

Remove it to save gas.

```diff
    function decreaseLiquidity(
        uint256 tokenId,
        uint128 _amount
    ) external onlyOwner returns (uint256 amount0, uint256 amount1) {

        // unused variable. Save gas by removing this line
-       uint128 liquidity = deposits[tokenId].liquidity;

        // ...

    }

```
--------------




### [G-5] `SoarStaking.setReward()` makes an unnecessary storage read of an already in-memory variable

In `setRewards()`, when `lastBlockWithReward` is set, it reads `firstBlockWithReward` from storage. Reading from storage is a costly operation, and the value is already in one of the calldata arguments, `_startingBlock`. 

Also, the storage variables are read to emit the event, but the values can be deducted from in-memory variables.

```javascript
    function setRewards(
        uint256 _rewardPerBlock,
        uint256 _startingBlock,
        uint256 _blocksAmount
    ) external onlyOwner updateReward(address(0)) {
        uint256 unlockedTokens = _getFutureRewardTokens();

        rewardPerBlock = _rewardPerBlock;
        firstBlockWithReward = _startingBlock;
@>      lastBlockWithReward = firstBlockWithReward + _blocksAmount - 1;

        // ...

        emit RewardsSet(
            _rewardPerBlock,
@>          firstBlockWithReward,
@>          lastBlockWithReward
        );
    }
```

#### Optimization

```diff
    function setRewards(
        uint256 _rewardPerBlock,
        uint256 _startingBlock,
        uint256 _blocksAmount
    ) external onlyOwner updateReward(address(0)) {
        uint256 unlockedTokens = _getFutureRewardTokens();

        rewardPerBlock = _rewardPerBlock;
        firstBlockWithReward = _startingBlock;
-       lastBlockWithReward = firstBlockWithReward + _blocksAmount - 1;
+       lastBlockWithReward = _startingBlock + _blocksAmount - 1;

        // ...

        emit RewardsSet(
            _rewardPerBlock,
-           firstBlockWithReward,
+           _startingBlock,
-           lastBlockWithReward
+           _startingBlock + _blocksAmount - 1
        );

    }
```
--------------

