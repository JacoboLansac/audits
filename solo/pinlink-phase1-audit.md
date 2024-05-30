# PinLink security review

***Preliminar version, 2024-05-29.***

A time-boxed security review of the [**PinkLink**](https://pinlink.gitbook.io/pinlink) protocol, with a focus on smart contract security and gas optimizations.

Author: [**Jacopod**](https://twitter.com/jacolansac), independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits/blob/main/README.md).

## Findings summary

| Finding                                                                                                                                                                        | Severity   | Description                                                                                                                                                         | Status   |
|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------|
| [C-1](<#c-1-an-attacker-can-drain-the-pinlink_staking-contract-by-claiming-rewards-multiple-times-as-there-is-no-registry-of-the-last-claimed-timestamp>)                      | Critical   | An attacker can drain the `PinLink_staking` contract by claiming rewards multiple times as there is no registry of the last claimed timestamp                       | -        |
| [C-2](<#c-2-any-rentee-can-terminate-any-other-rentals-of-the-same-tokenid-stealing-the-collateral-and-rewards-from-all-other-rentees-while-keeping-its-own-rental-untouched>) | Critical   | Any rentee can terminate any other rentals of the same `tokenId`, stealing the collateral and rewards from all other rentees while keeping its own rental untouched | -        |
| [C-3](<#c-3-the-pinlink_staking-contract-is-insolvent-by-default-because-the-pin-token-is-a-_fee-on-transfer_-token>)                                                          | Critical   | The `PinLink_staking` contract is insolvent by default because the PIN token is a _fee-on-transfer_ token.                                                          | -        |
| [C-4](<#c-4-the-rentabletoken-is-not-returned-when-a-rental-is-terminated>)                                                                                                    | Critical   | The `rentableToken` is not returned when a rental is terminated                                                                                                     | -        |
| [H-1](<#h-1-dos-attack-on-the-marketplace-contract-can-make-some-critical-functions-forever-unusable-delist-returnonrent-and-resume>)                                          | High       | DOS attack on the Marketplace contract can make some critical functions forever unusable: `delist()`, `returnOnRent()`, and `resume()`                              | -        |
| [H-2](<#h-2-the-wrong-tokenidis-minted-when-minting-more-supply-of-an-existing-tokenid-of-fractionaltoken-currenttokenid-instead-of-id>)                                       | High       | The wrong `tokenId`is minted when minting more supply of an existing `tokenId` of FractionalToken (`currentTokenId` instead of `id`)                                | -        |
| [H-3](<#h-3-the-protocol-can-become-insolvent-when-collateralperunit-is-updated-if-rentees-add-to-existing-rentals-->)                                                         | High       | The protocol can become insolvent when `collateralPerUnit` is updated if rentees add to existing rentals                                                            | -        |
| [H-4](<#h-4-users-will-lose-all-unclaimed-rewards-when-unstaking-from-pinlink_staking>)                                                                                        | High       | Users will lose all unclaimed rewards when unstaking from `PinLink_Staking`                                                                                         | -        |
| [H-5](<#h-5-starting-and-ending-rentals-will-revert-for-a-period-of-time-for-rentees-that-have-returned-partial-amounts-of-rentabletoken-to-the-marketplace>)                  | High       | Starting and ending rentals will revert for a period of time for rentees that have returned partial amounts of `rentableToken` to the Marketplace                   | -        |
| [H-6](<#h-6-users-will-never-get-any-staking-rewards-from-pinlink_staking-because-the-percentage-variable-is-declared-with-a-wrong-data-type>)                                 | High       | Users will never get any staking rewards from `PinLink_Staking` because the `percentage` variable is declared with a wrong data type                                | -        |
| [H-7](<#h-7-rewards-will-get-locked-in-pinlink_staking-for-users-with-too-many-stakes-because-claimreward-runs-out-of-gas-iterating-the-array-of-stakes>)                      | High       | Rewards will get locked in `PinLink_Staking` for users with too many stakes, because `claimReward()` runs out of gas iterating the array of stakes                  | -        |
| [H-8](<#h-8-rewards-calculation-pinlink_staking-are-inflated-by-a-factor-of-x86400-because-the-divisor-is-30-seconds-instead-of-30-days>)                                      | High       | Rewards calculation `PinLink_Staking` are inflated by a factor of `x86400` because the divisor is `30 seconds` instead of `30 days`                                 | -        |
| [H-9](<#h-9-the-pinlink_staking-contract-can-become-insolvent-because-user-staked-tokens-are-also-distributed-as-rewards>)                                                     | High       | The `PinLink_Staking` contract can become insolvent because user-staked tokens are also distributed as rewards                                                      | -        |
| [H-10](<#h-10-any-account-with-the-renter_role-can-empty-the-marketplace-contract-from-rewards-tokens-by-listing-a-tokenid-with-a-very-high-rewardrateperhour>)                | High       | Any account with the `RENTER_ROLE` can empty the Marketplace contract from rewards tokens by listing a `tokenId` with a very high `rewardRatePerHour`               | -        |
| [M-1](<#m-1-starting-and-ending-rentals-in-the-marketplace-will-revert-temporarily-if-a-scheduled-pause-starts-shortly-after-the-rental-is-started>)                           | Medium     | Starting and ending rentals in the Marketplace will revert temporarily if a scheduled pause starts shortly after the rental is started                              | -        |
| [M-2](<#m-2-any-account-with-the-renter_role-can-steal-the-renter-status-from-another-renter-by-listing-with-tokenamount0-before-the-rightful-owner-does-it>)                  | Medium       | Any account with the `RENTER_ROLE` can steal the `renter` status from another renter by listing with `tokenAmount=0` before the rightful owner does it              | -        |
| [M-3](<#m-3-any-account-can-start-a-new-rent-on-delisted-tokens-when-they-are-delisted-without-being-withdrawn>)                                                               | Medium     | Any account can start a new rent on delisted tokens when they are delisted without being withdrawn                                                                  | -        |
| [M-4](<#m-4-renters-can-overwrite-by-mistake-an-existing-pausetime-by-shortening-the-pauseduration-and-thus-making-rentees-earn-more-rewards>)                                 | Medium     | Renters can overwrite by mistake an existing `pauseTime` by shortening the `pauseDuration` and thus making rentees earn more rewards                                | -        |
| [M-5](<#m-5-collateral-and-rentable-tokens-will-get-stuck-in-the-marketplace-contract-if-not-enough-rewards-tokens-in-the-balance>)                                            | Medium     | Collateral and rentable tokens will get stuck in the Marketplace contract if not enough rewards tokens in the balance                                               | -        |
| [M-6](<#m-6-lack-of-input-validation-in-pintokensettransactiontax-can-cause-reverts-in-all-token-transfers-if-provided-wrong-inputs>)                                          | Medium     | Lack of input validation in `PinToken.setTransactionTax()` can cause reverts in all token transfers if provided wrong inputs                                        | -        |
| [L-1](<#l-1-it-is-possible-to-mint-tokens-of-tokenid0-even-though-the-intention-is-to-keep-it-as-a-reserved-tokenid>)                                                          | Low        | It is possible to mint tokens of `tokenId==0` even though the intention is to keep it as a reserved `tokenId`                                                       | -        |
| [L-2](<#l-2-different-collateral-token-in-marketplacelist-has-no-effect-after-first-call>)                                                                                     | Low        | Different collateral token in `Marketplace.list()` has no effect after first call                                                                                   | -        |
| [L-3](<#l-3-in-fractionaltokenmint-consider-moving-the-uri-change-to-a-separate-function-instead-of-mint>)                                                                     | Low        | In `FractionalToken.mint()`, consider moving the URI change to a separate function instead of `mint`                                                                | -        |
| [L-4](<#l-4-follow-cei-pattern-in-pinlink_staking-functions-stake-and-unstake>)                                                                                                | Low        | Follow `CEI` pattern in `PinLink_Staking`, functions `stake()` and `unstake()`                                                                                      | -        |
| [L-5](<#l-5-in-pinlink_stakingstake-consider-using-safetransferfrom>)                                                                                                          | Low        | In `PinLink_Staking.stake()`, consider using `safeTransferFrom`                                                                                                     | -        |
| [L-6](<#l-6-in-pinlink_stakingstake-consider-using-stakingperiod-instead-of-hardcoding-7-days>)                                                                                | Low        | In `PinLink_Staking.stake()`, consider using `stakingPeriod` instead of hardcoding 7 days                                                                           | -        |
| [L-7](<#l-7-in-pinlink_stakingstake-and-unstake-consider-adding-the-stake-index-to-the-event>)                                                                                 | Low        | In `PinLink_Staking.stake()` and `unstake()`, consider adding the stake index to the event                                                                          | -        |
| [L-8](<#l-8-incorrect-visibility-modifier-in-pinlink_stakingcalculatetotalrewardamount>)                                                                                       | Low        | Incorrect visibility modifier in `PinLink_Staking.calculateTotalRewardAmount()`                                                                                     | -        |
| [L-9](<#l-9-in-pinlink_staking-missing-events-for-updatestakingperiod-and-updatepercentage-and-claimreward>)                                                                   | Low        | In `PinLink_Staking`, missing events for `updateStakingPeriod()` and `updatePercentage()` and `claimReward()`                                                       | -        |
| [GAS-1](<#gas-1-use-rentable-from-memory-instead-of-accessing-the-rentablestokenid-array>)                                                                                     | Gas        | Use rentable from memory instead of accessing the `rentables[tokenId]` array                                                                                        | -        |
| [GAS-2](<#gas-2-in-marketplace-store-pauseduration-in-rentable-instead-of-each-rental-to-avoid-a-for-loop>)                                                                    | Gas        | In `Marketplace` store `pauseDuration` in `rentable` instead of each rental to avoid a for loop                                                                     | -        |
| [GAS-3](<#gas-3-in-marketplacelist-apply-caching-in-rentablerenter>)                                                                                                           | Gas        | In `Marketplace.list()`, apply caching in `rentable.renter`                                                                                                         | -        |
| [GAS-4](<#gas-4-in-marketplacelist-apply-conditional-checks-before-writing-collateralperunit-and-rewardrateperhour>)                                                           | Gas        | In `Marketplace.list()`, apply conditional checks before writing `collateralPerUnit` and `rewardRatePerHour`                                                        | -        |
| [GAS-5](<#gas-5-redundant-renter-address-check-in-marketplacedelist-isvalidrenter>)                                                                                            | Gas        | Redundant renter address check in `Marketplace.delist()` `isValidRenter`                                                                                            | -        |
| [GAS-6](<#gas-6-add-indexes-to-the-functions-to-avoid-looping-through-the-same-elements>)                                                                                      | Gas        | Add indexes to the functions to avoid looping through the same elements                                                                                             | -        |


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

- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

## Scope

- Delivery date: `2024-05-30`
- Duration of the audit: 12 days
- Commit hashes in scope:
  - Staking repo: [6a410ebf0f3577248ea98b7be408fb3811bbbf8c](https://github.com/PinLinkNetwork/PinLink_SmartContract_Staking/commit/6a410ebf0f3577248ea98b7be408fb3811bbbf8c)
  - Token repo: [7ac1f68c418c5a62d446566daeb4ee8ad05f3844](https://github.com/PinLinkNetwork/PinLink_SmartContract_Token/commit/7ac1f68c418c5a62d446566daeb4ee8ad05f3844)
- Review Commit hash: [ ... ]

### Files in scope

| File                                                             | nSLOC   |
| ---------------------------------------------------------------- | ------- |
| `PinLink_SmartContract_Staking/contracts/PinLink_Staking.sol`    | 153     |
| `PinLink_SmartContract_Token/src/token/PinToken.sol`             | 76      |
| `PinLink_SmartContract_Token/src/marketplace/Marketplace.sol`    | 174     |
| `PinLink_SmartContract_Token/src/fractional/FractionalToken.sol` | 31      |
| **Total**                                                        | **434** |

### Libraries / standards

| Dependency / Import Path                                               | Count |
| ---------------------------------------------------------------------- | ----- |
| @openzeppelin/contracts/access/AccessControl.sol                       | 2     |
| @openzeppelin/contracts/token/ERC1155/IERC1155.sol                     | 1     |
| @openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol     | 1     |
| @openzeppelin/contracts/token/ERC1155/extensions/ERC1155URIStorage.sol | 1     |
| @openzeppelin/contracts/token/ERC1155/utils/ERC1155Holder.sol          | 1     |
| @openzeppelin/contracts/token/ERC20/IERC20.sol                         | 1     |
| @openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol                | 1     |
| openzeppelin-contracts/contracts/access/Ownable2Step.sol               | 1     |
| openzeppelin-contracts/contracts/token/ERC20/ERC20.sol                 | 1     |

## Protocol Overview

Delivering The First RWA-Tokenized DePIN Protocol, Driving Down Cost For AI Developers & Creating New Revenue Streams For DePIN Asset Owners.

- PinLink is an ERC20 token
- PinLink_Staking is a staking contract to stake the PinLink token
- FractionalToken is an ERC1155 that is meant to represent fractionalized GPUs
- Marketplace is a contract for renting FractionalTokens and getting rewards from it

### Architecture high level review

- The architecture of Marketplace and Staking contract are not efficient from a gas point of view, as they rely heavily on for-loops. Alternative implementations using "defi-math" are recommended. Feel free to reach out for further advice in this regard
- It is a common practice to request another audit even after the mitigation review, if there is more than 1 high per 100 lines of code. Here we have 15 crit/high for 434 lines of code, so it would be strongly recommended to get the code reviewed again. Especially if some architectural changes are done following the proposal of the previous point.

# Findings

Note: During the audit, some of the critical/high issues were communicated to the team.
The team has decided to rewrite the `PinLink_staking` contract from scratch, so the recommendations to mitigate issues on that contract will be minimal, to not waste much time.

## Critical risk

### [C-1] An attacker can drain the `PinLink_staking` contract by claiming rewards multiple times as there is no registry of the last claimed timestamp

When rewards are claimed, the only requirements are:

- stake hasn't been withdrawn
- the minimum staking period has passed (7 days)
- the `claimRewardDate` has passed

However, there are no requirements on the time since the last claim. Moreover, the `claimRewardDate` is updated but the new value is `block.timestamp`, which has no effect in any of the above requirements.

```javascript
    function claimReward() external {
        Stake[] storage userStakes = stakes[msg.sender];

        uint allUsersTotalStakedAmount = totalStakedAmount;
        uint totalBalance = token.balanceOf(address(this));

        for (uint i = 0; i < userStakes.length; i++) {
            if (
@>              !userStakes[i].withdrawFlag &&
@>              userStakes[i].stakedDate + stakingPeriod <= block.timestamp &&
@>              userStakes[i].claimRewardDate <= block.timestamp
            ) {
                uint userReward = (userStakes[i].amount /
                    allUsersTotalStakedAmount) *
                    ((totalBalance *
                        percentage *
                        (block.timestamp - userStakes[i].stakedDate)) / 30);

                require(
                    token.transfer(msg.sender, userReward),
                    "Token transfer failed"
                );

@>              userStakes[i].claimRewardDate = block.timestamp; // Reset claim reward date
            }
        }
    }
```

Therefore, as soon as the `stakingPeriod` passes, a staker can call the `claimReward()` function infinite times and drain all the funds in the contract.

#### Impact: Critical

A staker can drain all funds in the `PinLink_staking` contract after 7 days of staking by claiming rewards infinite times.

#### Recommendation

Instead of keeping track of the `claimRewardDate`, keep track of the last time rewards were claimed, and only accrue rewards for that period and update the
time when the claim happened.

---

### [C-2] Any rentee can terminate any other rentals of the same `tokenId`, stealing the collateral and rewards from all other rentees while keeping its own rental untouched

<a id="any-rentee-terminates-other-rental"></a>

In `Marketplace.returnOnRent()` the `msg.sender` is not checked against `account`.
Moreover, the collateral and rewards are sent to `msg.sender` and not to `account`.
Therefore, a malicious rentee can call `returnOnRent()` providing a different `account` as input and stealing all collateral and rewards from `account`,
while its own rental remains untouched. The only requirement is that the attacker has an active rental with the same `tokenId`.

```javascript
    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {
        Rentable storage rentable = rentables[tokenId];
        Rental storage rental = rentals[tokenId][account];
@>      require(msg.sender == rentable.renter || rentals[tokenId][msg.sender].amount >= tokenAmount, "invalid request");
        require(rental.amount >= tokenAmount, "no active rental");
        if (rentable.pauseTime > 0 && rentable.pauseTime < block.timestamp) {
            adjustPauseRewards(account, tokenId, rentable.pauseTime, block.timestamp);
        }
        uint256 rewardAmount = calculateReward(account, tokenId, tokenAmount);
        uint256 collateralAmount = tokenAmount * rental.collateralPerUnit;

        rentable.rentedAmount -= tokenAmount;
        if (rental.amount == tokenAmount) {
            delete rentals[tokenId][msg.sender];
            removeRentee(tokenId, msg.sender);
        }
        else {
            rental.amount -= tokenAmount;
            rental.collateralPerUnit = rentable.collateralPerUnit;
            rental.rewardRate = rentable.rewardRate;
            rental.startTime = block.timestamp;
        }
@>      IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
@>      rewardToken.safeTransfer(msg.sender, rewardAmount);

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }
```

#### Impact: Critical

A rentee could steal all collateral tokens from all existing rentals as long as he has an active rental with the same `tokenId`.

#### Recommendation

If the `account` input argument is passed, the `msg.sender` should be either the renter or the `tokenId`, or the `account`.
All operations in `takeOnRent()` should be applied to `account`, not `msg.sender.

Furthermore, for better readability and organization: it is recommended to split this function into two external functions,
which would call an internal function `_finishRental(account)`. The new flow would be:

```diff

+    function returnOnRent(uint256 tokenId, uint256 tokenAmount) external {
+        _returnOnRent(msg.sender, tokenId, tokenAmount);
+    }

+    function forceReturnOnRent(address account, uint256 tokenId, uint256 tokenAmount) external {
+        require(msg.sender == rentables[tokenId].renter, "Invalid caller");
+        _returnOnRent(account, tokenId, tokenAmount);
+    }

-   function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {
+   function _returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) internal {
        Rentable storage rentable = rentables[tokenId];
        Rental storage rental = rentals[tokenId][account];
-       require(msg.sender == rentable.renter || rentals[tokenId][msg.sender].amount >= tokenAmount, "invalid request");
+       require(rentals[tokenId][account].amount >= tokenAmount, "Invalid amount");

        require(rental.amount >= tokenAmount, "no active rental");
        if (rentable.pauseTime > 0 && rentable.pauseTime < block.timestamp) {
            adjustPauseRewards(account, tokenId, rentable.pauseTime, block.timestamp);
        }
        uint256 rewardAmount = calculateReward(account, tokenId, tokenAmount);
        uint256 collateralAmount = tokenAmount * rental.collateralPerUnit;

        rentable.rentedAmount -= tokenAmount;
        if (rental.amount == tokenAmount) {
-           delete rentals[tokenId][msg.sender];
-           removeRentee(tokenId, msg.sender);
+           delete rentals[tokenId][account];
+           removeRentee(tokenId, account);
        }
        else {
            rental.amount -= tokenAmount;
            rental.collateralPerUnit = rentable.collateralPerUnit;
            rental.rewardRate = rentable.rewardRate;
            rental.startTime = block.timestamp;
        }
-       IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
-       rewardToken.safeTransfer(msg.sender, rewardAmount);
+       IERC20(rentable.collateral).safeTransfer(account, collateralAmount);
+       rewardToken.safeTransfer(account, rewardAmount);

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }

```

- `returnOnRent()`, which calls `_finishRental(msg.sender)`.
- `forceReturn(address rentee)` (that can only be called by renters), which would call `_finishRental(rentee)`.

---

### [C-3] The `PinLink_staking` contract is insolvent by default because the PIN token is a _fee-on-transfer_ token.

The PIN token is a _fee-on-transfer_ token. Normal transfers are affected by a certain fee, so the amount received by the destination is always lower than the amount transferred:

```javascript
    function _update(address sender, address recipient, uint256 amount) internal override {
        if (dexWhitelist[sender] || dexWhitelist[recipient]) {
            // Buy or sell detected
            uint256 buySellTax = (amount * buySellTax) / TAX_DIVISOR;
            uint256 amountAfterTax = amount - buySellTax;

            super._update(sender, recipient, amountAfterTax);
            super._update(sender, treasury, buySellTax);
        } else {
            // Regular transfer
@>          uint256 feeAmount = (amount * transactionTax) / TAX_DIVISOR;
@>          uint256 amountAfterFee = amount - feeAmount;

            uint256 toStaking = (feeAmount * stakingTax) / TAX_DIVISOR;
            uint256 toTreasury = feeAmount - toStaking;

@>          super._update(sender, recipient, amountAfterFee);  // amountAfterFee < amount
            super._update(sender, staking, toStaking);
            super._update(sender, treasury, toTreasury);
        }
    }
```

This is ignored in the `PinLink_staking.stake()` function. The `transferFrom` requests `_amount`, which is also the amount registered as a stake.
However, the contract balance will always receive a lower amount as the transfer fees are deducted.
This makes the contract insolvent by default, as the PIN tokens in the contract balance will always be lower than the sum of user stakes.

```javascript
    function stake(uint _amount) external {
        require(_amount > 0, "Amount must be greater than 0");
        require(
@>          token.transferFrom(msg.sender, address(this), _amount),
            "Token transfer failed"
        );

        stakes[msg.sender].push(
            Stake({
@>              amount: _amount,
                stakedDate: block.timestamp,
                withdrawFlag: false,
                withdrawDate: 0,
                claimRewardDate: block.timestamp + 7 days
            })
        );

        totalStakedAmount += _amount;

        emit EventStaked(msg.sender, _amount);
    }
```

#### Impact: Critical

The contract is insolvent by default. If all stakers wanted to unstake their tokens, the last ones would be left with nothing to unstake, losing their funds.

#### Recommendation

Only register in the stake struct the amount that actually reaches the contract balance.

```diff
    function stake(uint _amount) external {
        require(_amount > 0, "Amount must be greater than 0");

+       // calculate how many tokens actually arrive to the contract's balance
+       uint256 balanceBefore = token.balanceOf(address(this));
        require(
            token.transferFrom(msg.sender, address(this), _amount),
            "Token transfer failed"
        );
+       uint256 actualAmount = token.balanceOf(address(this)) - balanceBefore;

        stakes[msg.sender].push(
            Stake({
-               amount: _amount,
+               amount: actualAmount,
                stakedDate: block.timestamp,
                withdrawFlag: false,
                withdrawDate: 0,
                claimRewardDate: block.timestamp + 7 days
            })
        );

-       totalStakedAmount += _amount;
+       totalStakedAmount += actualAmount;

-       emit EventStaked(msg.sender, _amount);
+       emit EventStaked(msg.sender, actualAmount);
    }
```

---

### [C-4] The `rentableToken` is not returned when a rental is terminated

The transfer of the `rentableToken` back to the contract or the owner in `Marketplace.returnOnRent()` is missing.
Therefore, when the rental is terminated, the `rentableToken` is kept in the rentee's wallet.

Moreover, the `rentableToken` should freeze transfers while being rented, otherwise, the rentee can transfer it to a different account so that the rental is never terminated.
Besides freezing transfers, the Marketplace contract should always be an approved handler by default, to always be able to get the rentable tokens back.

#### Impact: Critical

The rentee can keep the rentableToken, and the renter can't delist the token to get it back, thus loosing the token.

#### Recommendation

Retrieve the rentable token from the renter in `Marketplace.returnOnRent()`:

```diff
    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {

        // ...

        IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
        rewardToken.safeTransfer(msg.sender, rewardAmount);
+       rentableToken.safeTransferFrom(account, address(this), tokenId, tokenAmount, "");

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }
```

Also, the FractionalToken.sol needs the following changes:

- Ensure the tokens cannot be transferred while being rented. For this override `ERC1155._beforeTokenTransfer()`.
- By default, the Marketplace contract should always be an approved handler of the rentableToken. Implement overriding `isApprovedForAll()`.

Please reach out if more guidance is needed.

---

## High risk

### [H-1] DOS attack on the Marketplace contract can make some critical functions forever unusable: `delist()`, `returnOnRent()`, and `resume()`

It is possible to call `takeOnRent()` with `tokenAmount=0` (as `ERC1155.safeTransferFrom` does not revert) with no more expenditure than the gas cost and push `msg.sender` to the `tokenRentees` array multiple times.
This is possible because it assumes that if the current rental amount is 0, the `msg.sender` is not in the array. This is not true, as it is possible to call the function with `tokenAmount=0`. 

```javascript
    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {
        Rentable memory rentable = rentables[tokenId];
        require(rentable.pauseTime == 0, "renting paused");
@>      require(rentable.amount - rentable.rentedAmount >= tokenAmount, "not enough fractions available");
        IERC20(rentables[tokenId].collateral).safeTransferFrom(msg.sender, address(this), tokenAmount * rentable.collateralPerUnit);
        if (rentals[tokenId][msg.sender].amount > 0) { // already rented
            uint256 rewardAmount = calculateReward(msg.sender, tokenId, tokenAmount);
            rewardToken.safeTransfer(msg.sender, rewardAmount);
        }
        else {
@>          tokenRentees[tokenId].push(msg.sender);
        }
        rentals[tokenId][msg.sender].amount += tokenAmount;
        rentals[tokenId][msg.sender].collateralPerUnit = rentable.collateralPerUnit;
        rentals[tokenId][msg.sender].rewardRate = rentables[tokenId].rewardRate;
        rentals[tokenId][msg.sender].startTime = block.timestamp;
        rentables[tokenId].rentedAmount += tokenAmount;
        rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, tokenAmount, "");

        emit Rent(msg.sender, tokenId, tokenAmount);
    }
```

The gas cost of iterating over a storage array like `tokenRentees` scales linearly with the size of the array, so when the array is long enough, there is simply not enough gas in a block to execute the function. The functions iterating the `tokenRentees` are: 
- `delist()`
- `returnOnRent()` on when returning the full amount, as it calls `removeRentee()`
- `resume(), which calls`adjustAllPauseRewards()` which iterates the array

Therefore, an attacker can make the three functions above unusable by calling the function enough times to make the array long enough.

#### Impact: High

A malicious actor can perform a cheap Denial of Service attack, locking the rentable tokens in the contract by making the `delist()` function always revert due to running out of gas.
Functions affected that iterate over the rentees array:

- `delist()`: rentableTokens of that `tokenId` are locked forever in the contract
- `returnOnRent()`: rentals cannot be finished: collateral and rewards are locked forever in the contract
- `resume()`: once a pause is scheduled, a rentableToken will remain paused. 

#### Recommendation

Require that a minimum amount of tokens have to be rented. This has two effects:
- The requirement `rentals[tokenId][msg.sender].amount > 0` cannot bypassed anymore, and therefore it is not possible to push the same `msg.sender` multiple times to the same array
- Even if attempting the attack from multiple wallets, the cost of the attack increases significantly. 

Also, limit the size of the array: `require(tokenRentees[tokenId].length < MAX_RENTEES_PER_TOKEN)`. (configurable per rentable token, or hardcoded for all of them).

```diff
+   uint256 public constant MIN_TOKEN_AMOUNT = 10; // or whatever number
+   uint256 public constant MAX_RENTEES_PER_TOKEN = 50; // or whatever number

    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {
+       require(tokenAmount > MIN_TOKEN_AMOUNT, "Invalid tokenAmount");
        Rentable memory rentable = rentables[tokenId];
        require(rentable.pauseTime == 0, "renting paused");
        require(rentable.amount - rentable.rentedAmount >= tokenAmount, "not enough fractions available");
        IERC20(rentables[tokenId].collateral).safeTransferFrom(msg.sender, address(this), tokenAmount * rentable.collateralPerUnit);
        if (rentals[tokenId][msg.sender].amount > 0) { // already rented
            uint256 rewardAmount = calculateReward(msg.sender, tokenId, tokenAmount);
            rewardToken.safeTransfer(msg.sender, rewardAmount);
        }
        else {
+           require(tokenRentees[tokenId].length < MAX_RENTEES_PER_TOKEN, "Max rentees per tokenId hit");
            tokenRentees[tokenId].push(msg.sender);
        }
        rentals[tokenId][msg.sender].amount += tokenAmount;
        rentals[tokenId][msg.sender].collateralPerUnit = rentable.collateralPerUnit;
        rentals[tokenId][msg.sender].rewardRate = rentables[tokenId].rewardRate;
        rentals[tokenId][msg.sender].startTime = block.timestamp;
        rentables[tokenId].rentedAmount += tokenAmount;
        rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, tokenAmount, "");

        emit Rent(msg.sender, tokenId, tokenAmount);
    }
```

---

### [H-2] The wrong `tokenId`is minted when minting more supply of an existing `tokenId` of FractionalToken (`currentTokenId` instead of `id`)

There are two `mint()` functions in `FractionalToken`. One of them allows passing an `id` parameter to mint more of an existing `tokenId`. The issue is that the `_mint()` function does not mint tokens of `id`, but of `currentTokenId`:

```javascript
    function mint(uint256 id, uint256 amount, string memory tokenURI) public onlyRole(MINTER_ROLE) {
        require(id <= currentTokenId && balanceOf(msg.sender, id) == totalSupply(id), "Missing complete ownership");
@>      _mint(msg.sender, currentTokenId, amount, "");
        _setURI(tokenURI);
    }
```

#### Impact: High

- Impact 1: The minted token is not the one expected.
- Impact 2: If the minter of `id` is different than the one for `currentId`, tokens are minted to the wrong address.

#### Recommendation

Change `currentTokenId` to `id` in the `_mint()` call.

```diff
    function mint(uint256 id, uint256 amount, string memory tokenURI) public onlyRole(MINTER_ROLE) {
        require(id <= currentTokenId && balanceOf(msg.sender, id) == totalSupply(id), "Missing complete ownership");
-       _mint(msg.sender, currentTokenId, amount, "");
+       _mint(msg.sender, id, amount, "");
        _setURI(tokenURI);
    }
```


### [H-3] The protocol can become insolvent when `collateralPerUnit` is updated if rentees add to existing rentals  

Every time the `takeOnRent()` function is called, the `collateralPerUnit` is overwriten.

```javascript
    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {

        // ...

        rentals[tokenId][msg.sender].amount += tokenAmount;
@>      rentals[tokenId][msg.sender].collateralPerUnit = rentable.collateralPerUnit;
        rentals[tokenId][msg.sender].rewardRate = rentables[tokenId].rewardRate;
        rentals[tokenId][msg.sender].startTime = block.timestamp;
        rentables[tokenId].rentedAmount += tokenAmount;
        rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, tokenAmount, "");

        emit Rent(msg.sender, tokenId, tokenAmount);
    }
```

The `collateralPerUnit` is used in `returnOnRent()` when a rental is terminated to calculate the collateral to be returned to the rentee. However, the collateral to return is the multiplication of the amount of tokens times the current value of `collateralPerUnit`, which can be updated.

```javascript
    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {

        // ...
        uint256 rewardAmount = calculateReward(account, tokenId, tokenAmount);
@>      uint256 collateralAmount = tokenAmount * rental.collateralPerUnit;

        // ...

@>      IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
        rewardToken.safeTransfer(msg.sender, rewardAmount);

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }
```

When overwriting `collateralPerUnit` in `takeOnRent()` with a new `collateralPerUnit` amount, this amount is applied to all previous rentals of the same token at the moment of returning the rental token and calculating the collateral to be returned.

Therefore, when a rentee calls `takeOnRent()` on an existing rental, the calculation of total collateral to return is wrong (higher or lower depending on whether the `collateralPerUnit` was increased or decreased). This can make the protocol insolvent or lead to stuck collateral in the contract.

#### Proof of concept

- Renter lists 50 tokens with `collateralPerUnit=1`, rentee takes 50 tokens. paying 50 of collateral
- Renter lists 50 more tokens with `collateralPerUnit=2`, same rentee takes 50 tokens, paying 100 collateral
- Up to this point, the Marketplace contract has 150 collateral tokens in balance (50 + 100).
- Rentee returns 100 tokens, but as the latest `collateralPerUnit` is 2, he receives 200 collateral tokens instead of 150. This leads to protocol insolvency, as those tokens would be taken from some other rentee.

#### Impact: High

- If the `collateralPerUnit` is increased, the protocol becomes insolvent, because `returnOnRent()` will return more collateral than it should, leaving other rentees without collateral funds left, loosing their collateral.
- If the `collateralPerUnit` is decreased, collateral tokens will get stuck in the contract, because `returnOnRent()` will return fewer tokens than were deposited.

#### Recommendation
- fix 1: Don't allow updates on the `collateralPerUnit`. Once set for a rentableToken, this value should not change to not break collateral accounting.
- fix 2 (more complexity): Instead of keeping track of `collateralPerUnit` in the rental struct, store the full amount of collateral deposited from all calls to `takeOnRent()`. This makes the `returnOnRent()` more complicated, as it is not trivial to calculate how much collateral to return if not all tokens are returned. 

Fix 1 is recommended.

---

### [H-4] Users will lose all unclaimed rewards when unstaking from `PinLink_Staking`

In `PinLink_Staking` the `claimReward()` function skips rewards for indexes that have been already withdrawn:

```javascript
    function claimReward() external {
        Stake[] storage userStakes = stakes[msg.sender];
        // ...

        for (uint i = 0; i < userStakes.length; i++) {
            if (
@>              !userStakes[i].withdrawFlag &&
                userStakes[i].stakedDate + stakingPeriod <= block.timestamp &&
                userStakes[i].claimRewardDate <= block.timestamp
            ) {
                uint userReward = (userStakes[i].amount /
                    allUsersTotalStakedAmount) *
                    ((totalBalance *
                        percentage *
                        (block.timestamp - userStakes[i].stakedDate)) / 30);

                require(
                    token.transfer(msg.sender, userReward),
                    "Token transfer failed"
                );

                userStakes[i].claimRewardDate = block.timestamp; // Reset claim reward date
            }
        }
    }
```

However, when calling `unstake()` there is no guarantee that the rewards have been claimed for the index:

```javascript
    function unstake(uint index) external {
        require(index < stakes[msg.sender].length, "Invalid index");

        Stake storage userStake = stakes[msg.sender][index];
        require(!userStake.withdrawFlag, "Already withdrawn");

        require(
            token.transfer(msg.sender, userStake.amount),
            "Token transfer failed"
        );

@>      userStake.withdrawFlag = true;
        userStake.withdrawDate = block.timestamp;
        totalStakedAmount -= userStake.amount;

        emit EventUnstaked(msg.sender, userStake.amount);
    }
```

Therefore, if a user unstakes a stake before claiming its rewards, will forever lose those rewards.

#### Impact: high

Users who withdraw before claiming rewards will lose their rewards.

#### Recommendation

- Force-claim the rewards before unstaking an index. 
- If there are no reward funds, the function should not revert but simply emit an event for traceability.


### [H-5] Starting and ending rentals will revert for a period of time for rentees that have returned partial amounts of `rentableToken` to the Marketplace

The calculation of rewards is done as follows:

```javascript
    function calculateReward(address account, uint256 tokenId, uint256 tokenAmount) private view returns (uint256) {
        Rental memory rental = rentals[tokenId][account];
@>      uint256 duration = (block.timestamp - (rental.startTime + rental.pauseDuration)) / 1 hours;
        uint256 reward = tokenAmount * rental.rewardRate * duration;
        return reward;
    }
```

However, when calling calling `returnOnRent()` with `tokenAmount < rental.amount`, the `startTime` is reset but not the `pauseDuration`. Note also that the `periodDuration` has already been included in `adjustPauseRewards()`:

```javascript
    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {

        // ...

        if (rentable.pauseTime > 0 && rentable.pauseTime < block.timestamp) {
@>          adjustPauseRewards(account, tokenId, rentable.pauseTime, block.timestamp);
        }

        //

        else {
            rental.amount -= tokenAmount;
            rental.collateralPerUnit = rentable.collateralPerUnit;
            rental.rewardRate = rentable.rewardRate;
@>          rental.startTime = block.timestamp;  // rental.periodDuration should be udpated here
        }
        IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
        rewardToken.safeTransfer(msg.sender, rewardAmount);

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }
```

This means that if `pauseDuration > 0`, as soon as `returnOnRent()` is called with `tokenAmount < rental.amount`, the value of `rental.startTime + rental.pauseDuration`
will always be higher than `block.timestamp`, making the `calculateReward()` revert with an underflow.

#### Impact: High

Both functions `takeOnRent()` and `returnOnRent()` will revert for a rentee that has returned a partial amount of the rented tokens (`returnOnRent()` with `tokenAmount < rental.amount`).

#### Recommendation

Reset `rental.pauseDuration = 0`; here when `startTime` is set to `block.timestamp`, because this period duration has just been included in the rewards adjustments

```diff
    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {

        // ...

        else {
            rental.amount -= tokenAmount;
            rental.collateralPerUnit = rentable.collateralPerUnit;
            rental.rewardRate = rentable.rewardRate;
            rental.startTime = block.timestamp;
+           rental.periodDuration = 0;  // This is fine as the pause time has already been included in the rewards adjustment above
        }
        IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
        rewardToken.safeTransfer(msg.sender, rewardAmount);

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }
```

---


### [H-6] Users will never get any staking rewards from `PinLink_Staking` because the `percentage` variable is declared with a wrong data type

The `percentage` variable is a `uint` (unsigned integer), but the value assigned is `0.83`. Unsigned integers round float numbers to integers, and the `0.83` becomes `0`. This `percentage` variable is used to calculate rewards (as a percentage of the staked amount) and therefore, the calculated rewards will always be `0`, regardless of the amount staked and the period of time.

```javascript
contract PinLinkStaking {
    // ...

    address public owner;
    uint public totalStakedAmount;
@>  uint public percentage = 0.83; // reward percentage for each month
    uint public stakingPeriod = 7 days;
    
    // ...
}
```

This issue is so bad, that some smart contract frameworks don't even compile due to this bug.

#### Impact: High

Some frameworks won't even compile this contract. 

If compiled, the rewards will be zero for all stakers regardless of the staking period and amount staked. No rewards means breaking the only purpose of a staking contract.

#### Recommendation

Use a `PRECISION` constant to multiply the percentage to achieve the effect of floats. Divide by `PRECISION` in whatever quantity multiplied by `percentage`.

```diff
    address public owner;
    uint public totalStakedAmount;
-   uint public percentage = 0.83; // reward percentage for each month
+   uint public constant PRECISION = 1e18;
+   uint public percentage = 0.83 * PRECISION; // reward percentage for each month
    uint public stakingPeriod = 7 days;


    function claimReward() external {
        // ...
                uint userReward = (userStakes[i].amount /
                    allUsersTotalStakedAmount) *
                    ((totalBalance *
                        percentage *
-                       (block.timestamp - userStakes[i].stakedDate)) / 30);
+                       (block.timestamp - userStakes[i].stakedDate)) / (30 * PRECISION));
        // ...

```

---

### [H-7] Rewards will get locked in `PinLink_Staking` for users with too many stakes, because `claimReward()` runs out of gas iterating the array of stakes

The `stake()` function does not limit the number of stakes an account can have. With every new stake, a new struct is pushed to the user array:

```javascript
    function stake(uint _amount) external {
        require(_amount > 0, "Amount must be greater than 0");
        require(
            token.transferFrom(msg.sender, address(this), _amount),
            "Token transfer failed"
        );

@>      stakes[msg.sender].push(
            Stake({
                amount: _amount,
                stakedDate: block.timestamp,
                withdrawFlag: false,
                withdrawDate: 0,
                claimRewardDate: block.timestamp + 7 days
            })
        );

        totalStakedAmount += _amount;

        emit EventStaked(msg.sender, _amount);
    }
```

This array is iterated in `unstakeAll()` and `claimReward()`. For users that stake too many times, both functions will revert due to gas exhaustion.

#### Impact: High

If the number of stakes in the array is large enough, both `unstakeAll()` and `claimReward()` will revert with out-of-gas error.

- In the case of `unstakeAll()`, this is not a big deal (besides the function being unusable), because users can stake each index individually by calling `unstake()`
- However, if `claimReward()` reverts, this means that the user cannot claim staking rewards ever again.

#### Recommendations

- Limit the size of `userStakes` to avoid gas exhaustion 
- Add start and end indexes to the functions that need to iterate an array so that all user stakes can be processed in batches.

---

### [H-8] Rewards calculation `PinLink_Staking` are inflated by a factor of `x86400` because the divisor is `30 seconds` instead of `30 days`

According to the comments in the code, the intention of the dev is to reward `percentage` every 30 days. Let's assume now that `percentage` was properly defined. Duration in solidity is expressed as seconds. Therefore, when calculating monthly APR, the divisor should be `30 days`. Instead, only `30` is used.

This leads to an increased reward factor of x86400 (seconds in one day).

#### Impact: high

Rewards are scaled up by a factor of 864000.

#### Recommendation

Divide by `30 days` instead of `30` in the rewards calculations.

```diff
    function claimReward() external {
        // ...

                uint userReward = (userStakes[i].amount /
                    allUsersTotalStakedAmount) *
                    ((totalBalance *
                        percentage *
-                       (block.timestamp - userStakes[i].stakedDate)) / 30);
+                       (block.timestamp - userStakes[i].stakedDate)) / 30 days);
        // ...
    }
```

---

### [H-9] The `PinLink_Staking` contract can become insolvent because user-staked tokens are also distributed as rewards

In `PinLink_Staking.claimReward()`, the reward calculation assumes the full contract balance can be used for distributing rewards. This is incorrect because the stakes are also part of the contract balance.

#### Impact: high

The staked funds from users are distributed as rewards, leading to protocol insolvency.

#### Recommendation:

In the reward calculation, exclude the user stakes from the balance to distribute as rewards:

```diff
    function claimReward() external {
        // ...

                uint userReward = (userStakes[i].amount /
                    allUsersTotalStakedAmount) *
-                   ((totalBalance *
+                   (((totalBalance - allUsersTotalStakedAmount) *
                        percentage *
                        (block.timestamp - userStakes[i].stakedDate)) / 30);
        // ...
    }
```

### [H-10] Any account with the `RENTER_ROLE` can empty the Marketplace contract from rewards tokens by listing a `tokenId` with a very high `rewardRatePerHour`

The reward rate is set by the renter, while the reward token is expected to be funded by the contract owner. The fact that the renter and the funder are not the same entity, incentives the renter to set a high reward rate and rent it to itself to steal all the reward tokens.

Since `rewardRatePerHour` is not capped, a renter can list with a very high `rewardRatePerHour` and call `takeOnRent()` to empty the contract from reward tokens.

#### Impact: Critical

The contract can be emptied from reward tokens, halting the execution of further `takeOnRent()` calls.

#### Recommendation

The fix is non-trivial. As long as the _funder_ is a different agent than the _renter_, the renter will always have a strong incentive to set the rewards as high as possible profit from it. An alternative architecture would require the renter to fund the rewards himself, but this might not be aligned with the protocol's intention.

#### Team response

The current version is only a proof of concept in which the only renter is also the funder of the contract (contract admins). The issue will become relevant in the next version of the Marketplace when the `RENTER_ROLE` is given to arbitrary untrusted addresses.

---



## Medium

### [M-1] Starting and ending rentals in the Marketplace will revert temporarily if a scheduled pause starts shortly after the rental is started

The reward calculation reverts if the `rental.pauseDuration` is longer than the difference `block.timestamp - rental.startTime`:

```javascript
    function calculateReward(address account, uint256 tokenId, uint256 tokenAmount) private view returns (uint256) {
        Rental memory rental = rentals[tokenId][account];
@>      uint256 duration = (block.timestamp - (rental.startTime + rental.pauseDuration)) / 1 hours;
        uint256 reward = tokenAmount * rental.rewardRate * duration;
        return reward;
    }
```

This can easily happen in the following scenario:

- The rental starts now.
- 2 h later, renter schedules 24h of downtime starting in 1h.
- The `rental.pauseDuration` is now 24h, and `block.timestamp - rental.start = 3 hours`
- Therefore, `calculateReward()` will revert with underflow, as the `startime + pauseDuration` is larger than the `block.timestamp`

#### Impact: Medium

Temporal halt of important contract functions such as `takeOnRent()` and `returnOnRent()`, until the time from `rental.startTime` is larger than the pause duration. Once this period passes, this bug has no effect.

#### Recommendation

The `calculateReward()` function should calculate zero rewards in the above scenario:

```diff
    function calculateReward(address account, uint256 tokenId, uint256 tokenAmount) private view returns (uint256) {
        Rental memory rental = rentals[tokenId][account];
+       if (block.timestamp < rental.startTime + rental.pauseDuration) return 0;
        uint256 duration = (block.timestamp - (rental.startTime + rental.pauseDuration)) / 1 hours;
        uint256 reward = tokenAmount * rental.rewardRate * duration;
        return reward;
    }
```

---


### [M-2] Any account with the `RENTER_ROLE` can steal the `renter` status from another renter by listing with `tokenAmount=0` before the rightful owner does it

When a `tokenId` is listed for the first time in the Marketplace contract, the `msg.sender` gets assigned the `rentable.renter` status. In order to do so, the `msg.sender` must own at least `tokenAmount` of `tokenId` in his account:

```javascript
    function list(uint256 tokenId, uint256 tokenAmount, address collateralToken, uint256 collateralPerUnit, uint256 rewardRatePerHour) external onlyRole(RENTER_ROLE) {
        Rentable storage rentable = rentables[tokenId];
        require(rentable.renter == address(0) || rentable.renter == msg.sender, "already rented");
@>      require(rentableToken.balanceOf(msg.sender, tokenId) >= tokenAmount, "renter must own the token");
        rentableToken.safeTransferFrom(msg.sender, address(this), tokenId, tokenAmount, "");

        if (rentable.renter == address(0)) {
@>          rentable.renter = msg.sender;  // the first caller gets the renter rights
            rentable.collateral = collateralToken;
        }
        rentable.amount += tokenAmount;
        rentable.collateralPerUnit = collateralPerUnit;
        rentable.rewardRate = rewardRatePerHour;

        emit List(msg.sender, tokenId, tokenAmount, rentable.collateral, collateralPerUnit, rewardRatePerHour);
    }
```

However, there is no requirement to list a positive amount such as `require(tokenAmount > 0)`. When this happens, neither `ERC1155` nor (`ERC20` for collateral transfer) reverts, and therefore the caller will be the assigned  `renter` of that `tokenId`, even if not have any of them in his wallet. Any account with `RENTER_ROLE` can call `list()` any `tokenId` passing `tokenAmount=0`, as long as they haven't been listed yet in the Marketplace. This allows renters to "steal" the renter status for any `tokenIds` from their rightful owners.

Moreover, there isn't either a requirement so that such `tokenId` has `totalSupply > 0`, so nothing stops the attacker from claiming renter ownership from future tokenIds that have never been minted, although its impact is less important.

#### Impact: Medium

- Any account with the `RENTER_ROLE` can steal the right to rent any `tokenId` that hasn't been rented yet, even without owning any of them
- Any account with the `RENTER_ROLE` can steal the right to rent future `tokenIds` that have not been minted, although its impact is less relevant.

#### Recommendation

Require that only positive `tokenAmount` can be listed. This also ensures that the supply for that `tokenId` is non zero, and therefore already exists.

```diff
    function list(uint256 tokenId, uint256 tokenAmount, address collateralToken, uint256 collateralPerUnit, uint256 rewardRatePerHour) external onlyRole(RENTER_ROLE) {
+       require(tokenAmount > 0, "Invalid tokenAmount");
        Rentable storage rentable = rentables[tokenId];
        require(rentable.renter == address(0) || rentable.renter == msg.sender, "already rented");
        require(rentableToken.balanceOf(msg.sender, tokenId) >= tokenAmount, "renter must own the token");
        rentableToken.safeTransferFrom(msg.sender, address(this), tokenId, tokenAmount, "");

        if (rentable.renter == address(0)) {
            rentable.renter = msg.sender;
            rentable.collateral = collateralToken;
        }
        rentable.amount += tokenAmount;
        rentable.collateralPerUnit = collateralPerUnit;
        rentable.rewardRate = rewardRatePerHour;

        emit List(msg.sender, tokenId, tokenAmount, rentable.collateral, collateralPerUnit, rewardRatePerHour);
    }
```

---


### [M-3] Any account can start a new rent on delisted tokens when they are delisted without being withdrawn

When a rentable is delisted, the `withdraw` boolean allows to withdraw the rentable tokens from the contract if `true`. Otherwise they are "delisted", but they stay in the contract balance:

```javascript
    function delist(uint256 tokenId, bool withdraw) external isValidRenter(tokenId) {
        Rentable memory rentable = rentables[tokenId];
        require(rentable.renter == msg.sender, "invalid renter");
        if (rentable.rentedAmount > 0) {
            address[] memory rentees = tokenRentees[tokenId];
            for (uint256 i = 0; i < rentees.length; i++) {
                returnOnRent(rentees[i], tokenId, rentals[tokenId][rentees[i]].amount);
            }
        }
@>      if (withdraw) {
@>          rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, rentable.amount, "");
            delete rentables[tokenId];
        }

        emit Delist(tokenId, withdraw);
    }
```

However, if `withdraw=false`, there is no registry that the tokens are actually delisted. Therefore, it is still possible for any account to call `takeOnRent()` on delisted tokens that have not been withdrawn.

```javascript
    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {
        Rentable memory rentable = rentables[tokenId];
        require(rentable.pauseTime == 0, "renting paused");
        require(rentable.amount - rentable.rentedAmount >= tokenAmount, "not enough fractions available");
        IERC20(rentables[tokenId].collateral).safeTransferFrom(msg.sender, address(this), tokenAmount * rentable.collateralPerUnit);
        if (rentals[tokenId][msg.sender].amount > 0) { // already rented
            uint256 rewardAmount = calculateReward(msg.sender, tokenId, tokenAmount);
            rewardToken.safeTransfer(msg.sender, rewardAmount);
        }
        else {
            tokenRentees[tokenId].push(msg.sender);
        }

        // ...
    }
```

Fortunately, there is no funds lost as the renter can call `delist()` again. However, it can be considered as a temporal loss of funds, as the renter is expecting them to be in the contract (but delisted).

#### Impact: Medium

The rentable can still be rented even after being delisted.

#### Recommendation

One of the following:

- Store the delisted state in the rentable and revert `takeOnRent()` for delisted tokens.
- Remove the `witdraw` parameter from the `delist()` function, allowing for only two states: 'listed' and 'delisted'.
- When delisting with `withdraw=false`, automatically pause the rentals.

#### Proof of concept

```javascript
    function testDelistAllowsForRentingAgain() public {
        uint256 tokenId = 1;
        uint256 amount = 5;


        vm.startPrank(renter);
        rentableToken.setApprovalForAll(address(marketplace), true);
        marketplace.list(tokenId, amount, address(collateralToken), 1 ether, 0.01 ether);
        marketplace.delist(tokenId, false);

        // At this point renter thinks it is not possible to rent the token anymore
        vm.stopPrank();

        vm.startPrank(rentee);
        collateralToken.approve(address(marketplace), type(uint256).max);
        marketplace.takeOnRent(tokenId, amount);
        vm.stopPrank();

        Marketplace.Rental memory rental = marketplace.getRental(rentee, tokenId);
        assertEq(rental.amount, amount);

    }
```

---

### [M-4] Renters can overwrite by mistake an existing `pauseTime` by shortening the `pauseDuration` and thus making rentees earn more rewards

There is no requirement in `pause()` so that the pause time hasn't been already set. A renter can overwrite their existing `pauseTime` shortening the `pauseDuration` and thus affecting the rewards calculations.

```javascript
    function pause(uint256 tokenId, uint256 pauseTime) external isValidRenter(tokenId) {
        require(pauseTime >= block.timestamp, "invalid pause time");
        rentables[tokenId].pauseTime = pauseTime;

        emit Pause(tokenId, pauseTime);
    }
```

#### Impact: Medium

All the rentees will receive different amounts of rewards.

#### Recommendation

Not trivial. `require(rental.pauseTime == 0)` can help, but then we need a way to reset them back to 0 after resuming. Bad UX for the renter.
A better approach would be to store the pauseTime in the rental, and only adjust the individual rentals

#### Team response:

The team said this was only a proof of concept and a new version would be deployed in a near future. A different architecture to handle pauses can be discussed.

---

### [M-5] Collateral and rentable tokens will get stuck in the Marketplace contract if not enough rewards tokens in the balance

The `rewardToken` balance is not explicitly managed or guaranteed, thus `returnOnRent()` can revert when the protocol does not have enough reward tokens in balance. The contract relies on admins depositing reward tokens, but if the admins discontinue this activity and there are no rewards in the contract, the rentable tokens and the collateral will get stuck in the contract.

```javascript
    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {

        // ...

        IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);
@>      rewardToken.safeTransfer(msg.sender, rewardAmount);

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }

```

#### Impact: Medium

Collateral and rentableTokens can get stuck in the contract if the contract admins discontinue providing reward tokens to the contract's balance because `returnOnRent()` will revert.

#### Recommendation

Before transferring `rewardToken`, check if there is enough balance in the contract. If not, instead of reverting, an event should be emitted. Regardless if there are rewards or not, the `collateral` and `rentableToken` should be returned to the rentee.

```diff
+   event MissedRewards(uint256 tokenId, address account, uint256 rewardAmount);

    function returnOnRent(address account, uint256 tokenId, uint256 tokenAmount) public {
        Rentable storage rentable = rentables[tokenId];
        Rental storage rental = rentals[tokenId][account];
        require(msg.sender == rentable.renter || rentals[tokenId][msg.sender].amount >= tokenAmount, "invalid request");
        require(rental.amount >= tokenAmount, "no active rental");
        if (rentable.pauseTime > 0 && rentable.pauseTime < block.timestamp) {
            adjustPauseRewards(account, tokenId, rentable.pauseTime, block.timestamp);
        }
        uint256 rewardAmount = calculateReward(account, tokenId, tokenAmount);
        uint256 collateralAmount = tokenAmount * rental.collateralPerUnit;

        rentable.rentedAmount -= tokenAmount;
        if (rental.amount == tokenAmount) {
            delete rentals[tokenId][msg.sender];
            removeRentee(tokenId, msg.sender);
        }
        else {
            rental.amount -= tokenAmount;
            rental.collateralPerUnit = rentable.collateralPerUnit;
            rental.rewardRate = rentable.rewardRate;
            rental.startTime = block.timestamp;
        }
        IERC20(rentable.collateral).safeTransfer(msg.sender, collateralAmount);

+        uint256 rewardsBalance = rewardToken.balanceOf(address(this));
+        if (rewardsBalance < rewardAmount) {
+            emit MissedRewards(tokenId, account, rewardAmount - rewardsBalance);
+            rewardAmount = rewardsBalance;
+        }

        rewardToken.safeTransfer(account, rewardAmount);  // note that this originally was sent to msg.sender

        emit Return(account, tokenId, tokenAmount, rewardAmount, collateralAmount);
    }

```

### [M-6] Lack of input validation in `PinToken.setTransactionTax()` can cause reverts in all token transfers if provided wrong inputs

The `PinToken.setTransactionTax()` lacks any input validation:

```javascript
    function setTransactionTax(uint256 transactionTax_) external onlyOwner {
        transactionTax = transactionTax_;

        emit SetTransactionTax(transactionTax_);
    }
```

This variable is later used in the `_update()` function, which handles the transfer fees:

```javascript

    function _update(address sender, address recipient, uint256 amount) internal override {
        if (dexWhitelist[sender] || dexWhitelist[recipient]) {
            // Buy or sell detected
            uint256 buySellTax = (amount * buySellTax) / TAX_DIVISOR;
            uint256 amountAfterTax = amount - buySellTax;

            super._update(sender, recipient, amountAfterTax);
            super._update(sender, treasury, buySellTax);
        } else {
            // Regular transfer
@>          uint256 feeAmount = (amount * transactionTax) / TAX_DIVISOR;
@>          uint256 amountAfterFee = amount - feeAmount;

            uint256 toStaking = (feeAmount * stakingTax) / TAX_DIVISOR;
            uint256 toTreasury = feeAmount - toStaking;

            super._update(sender, recipient, amountAfterFee);
            super._update(sender, staking, toStaking);
            super._update(sender, treasury, toTreasury);
        }
    }
```

The parameter `transactionTax` is meant to be a percentage, which uses `TAX_DIVISOR` as a scaling divisor. If the `transactionTax` is wrongly set to a higher value than `TAX_DIVISOR`, the `_update()` function will revert with an underflow as `amount < feeAmount`.

#### Other instances of the issue

Same issue affects the `buySellTax` and `stakingTax` which are set in the `setBuySellTax()` and `setStakingTax()` functions.

#### Impact: medium

If the `transactionTax` is wrongly configured, all token transfers will revert.

- Probability: low, as it requires a mistake or compromised keys
- Impact: critical, as it will halt all token transfers

#### Recommendation

None of the tax parameters should be higher than the `TAX_DIVISOR`:

```diff
    function setTransactionTax(uint256 transactionTax_) external onlyOwner {
+       require(transactionTax_ <= TAX_DIVISOR, "Invalid tax");
        transactionTax = transactionTax_;
        emit SetTransactionTax(transactionTax_);
    }

    function setBuySellTax(uint256 buySellTax_) external onlyOwner {
+       require(buySellTax_ <= TAX_DIVISOR, "Invalid tax");
        buySellTax = buySellTax_;
        emit SetBuySellTax(buySellTax_);
    }

    function setStakingTax(uint256 stakingTax_) external onlyOwner {
+       require(stakingTax_ <= TAX_DIVISOR, "Invalid tax");
        stakingTax = stakingTax_;
        emit SetStakingTax(stakingTax_);
    }
```

---

### [M-7][centralization] Admins can rug all PIN tokens by setting 100% transfer fees

The tax setter functions have no limit to the value they are set to. The contract admin can set the value to 100%, and cash out all tokens as fees.

#### Impact: medium

Contract admins can rug all the tokens transferred.

- Probability: low, as it requires malicious owner or compromised keys
- Impact: critical, as all transferred tokens are "stolen"

#### Recommendation

None of the tax parameters should be higher than the `TAX_DIVISOR`:

```diff

+   uint256 public constant MAX_TAX_BASIS_POINTS = 1000; // 1000 = 10%

    function setTransactionTax(uint256 transactionTax_) external onlyOwner {
+       require(transactionTax_ <= MAX_TAX_BASIS_POINTS, "Invalid tax");
        transactionTax = transactionTax_;
        emit SetTransactionTax(transactionTax_);
    }

    function setBuySellTax(uint256 buySellTax_) external onlyOwner {
+       require(buySellTax_ <= MAX_TAX_BASIS_POINTS, "Invalid tax");
        buySellTax = buySellTax_;
        emit SetBuySellTax(buySellTax_);
    }

    function setStakingTax(uint256 stakingTax_) external onlyOwner {
+       require(stakingTax_ <= MAX_TAX_BASIS_POINTS, "Invalid tax");
        stakingTax = stakingTax_;
        emit SetStakingTax(stakingTax_);
    }
```

## Low risk / Informational

### [L-1] It is possible to mint tokens of `tokenId==0` even though the intention is to keep it as a reserved `tokenId`

In `FractionalToken.mint(uint256 amount, string memory tokenURI)`, `tokenId=0` is never minted because `currentTokenId` is incremented before minting, thus starting from 1. The team said this was intentional, to reserve `tokenId=0` for future checks.

There is another `mint()` function that allows minting more tokens of an existing `tokenId` if the `msg.sender` owns the full existing supply of that token:

```javascript
    function mint(uint256 amount, string memory tokenURI) public onlyRole(MINTER_ROLE) {
@>      currentTokenId++; // tokenId=0 is never minted by this function
        _mint(msg.sender, currentTokenId, amount, "");
        _setURI(tokenURI);
    }

    // used to mint more supply for an already minted
    function mint(uint256 id, uint256 amount, string memory tokenURI) public onlyRole(MINTER_ROLE) {
@>      require(id <= currentTokenId && balanceOf(msg.sender, id) == totalSupply(id), "Missing complete ownership");
        _mint(msg.sender, currentTokenId, amount, "");
        _setURI(tokenURI);
    }
```

However, since the token with `tokenId=0` is never minted through the function above, the supply is 0. This allows anyone with the `MINTER_ROLE` minting tokens for `tokenId=0`, making it _not reserved anymore_, against the contract design.

#### Impact: Medium

Even though `tokenId=0` is supposed to be reserved, it can be minted by anyone with the `MINTER_ROLE`, breaking the purpose of this _reserved_ `tokenId`.

#### Recommendation

Consider using `ERC1155Supply.exists(id)` to check if the token exists, instead of `(id <= currentTokenId)` in the first part of the requirement. If the supply is 0 for a given `id`, the `exists(id)` returns `false.

```diff
    function mint(uint256 id, uint256 amount, string memory tokenURI) public onlyRole(MINTER_ROLE) {
-        require(id <= currentTokenId && balanceOf(msg.sender, id) == totalSupply(id), "Missing complete ownership");
+        require(ERC1155Supply.exists(id) && balanceOf(msg.sender, id) == totalSupply(id), "Missing complete ownership");
        _mint(msg.sender, id, amount, "");
        _setURI(id, tokenURI);
    }
```

### [L-2] Different collateral token in `Marketplace.list()` has no effect after first call

When listing a token for the second and subsequent times, the caller might think they can change the collateral token, by specifying a new one, which has no effect.

#### Impact: Low

The collateral token is not changed, and the caller might think it is.

#### Recommendation

Add a condition to check that the collateral token is the same as the previous one, and revert if it is not.

```diff
function list(uint256 tokenId, uint256 tokenAmount, address collateralToken, uint256 collateralPerUnit, uint256 rewardRatePerHour) external onlyRole(RENTER_ROLE) {

        Rentable storage rentable = rentables[tokenId];

        require(rentable.renter == address(0) || rentable.renter == msg.sender, "already rented");
        require(rentableToken.balanceOf(msg.sender, tokenId) >= tokenAmount, "renter must own the token");
        rentableToken.safeTransferFrom(msg.sender, address(this), tokenId, tokenAmount, "");

        if (rentable.renter == address(0)) {
            rentable.renter = msg.sender;
            rentable.collateral = collateralToken;
-        }
+        } else {
+            require(rentable.collateral == collateralToken, "collateral token cannot be changed");
        }

        rentable.amount += tokenAmount;

        rentable.collateralPerUnit = collateralPerUnit;
        rentable.rewardRate = rewardRatePerHour;

        emit List(msg.sender, tokenId, tokenAmount, rentable.collateral, collateralPerUnit, rewardRatePerHour);
    }
```

### [L-3] In `FractionalToken.mint()`, consider moving the URI change to a separate function instead of `mint`

Calling `FractionalToken.mint()` also sets the URI. It is considered best practice to move separate functionalities to separate functions. In this case, a caller might want to mint a token without changing the URI, or change the URI without minting a token.

#### Impact: Low

Single call used for `mint()` and setting `URI`.

#### Recommendation

Consider moving the URI change to a separate function instead of `mint`.

```diff
    function mint(uint256 id, uint256 amount, string memory tokenURI) public onlyRole(MINTER_ROLE) {
        require(id <= currentTokenId && balanceOf(msg.sender, id) == totalSupply(id), "Missing complete ownership");
        _mint(msg.sender, currentTokenId, amount, "");
-        _setURI(id, tokenURI);
    }

+    function setURI(uint256 id, string memory tokenURI) public onlyRole(MINTER_ROLE) {
+        _setURI(id, tokenURI);
+    }
```

### [L-4] Follow `CEI` pattern in `PinLink_Staking`, functions `stake()` and `unstake()`

In `PinLink_Staking`, follow the `CEI` pattern in functions `stake()` and `unstake()`.

#### Recommendation

It is always recommended to functions that interact with external contracts at the end of the function, to avoid reentrancy attacks.

```diff
    function stake(uint256 amount) external {
        require(_amount > 0, "Amount must be greater than 0");

-        require(
-            token.transferFrom(msg.sender, address(this), _amount),
-            "Token transfer failed"
-        );

        // ...

        // External call at the end
+        require(
+            token.transferFrom(msg.sender, address(this), _amount),
+            "Token transfer failed"
+        );
    }

    function unstake(uint256 index) external {
        // ...
-        require(
-            token.transfer(msg.sender, _amount),
-            "Token transfer failed"
-        );

        // External call at the end
+        require(
+            token.transfer(msg.sender, _amount),
+            "Token transfer failed"
+        );
    }
```

### [L-5] In `PinLink_Staking.stake()`, consider using `safeTransferFrom`

It is considered best practice to use `safeTransferFrom` instead of `transferFrom` to avoid reentrancy attacks. Under certain conditions, reentrancy attacks are possible for certain tokens.

#### Recommendation

Use `safeTransferFrom` to avoid reentrancy attacks.

```diff
    function stake(uint256 amount) external {
        require(_amount > 0, "Amount must be greater than 0");

        require(
-            token.transferFrom(msg.sender, address(this), _amount),
+            token.safeTransferFrom(msg.sender, address(this), _amount),
            "Token transfer failed"
        );

        // ...
    }
```

### [L-6] In `PinLink_Staking.stake()`, consider using `stakingPeriod` instead of hardcoding 7 days

An already defined variable called `stakingPeriod` should be used instead of hardcoding `7 days`.

#### Recommendation

```diff
    function stake(uint256 amount) external {
        // ...
        stakes[msg.sender].push(
            Stake({
                amount: _amount,
                stakedDate: block.timestamp,
                withdrawFlag: false,
                withdrawDate: 0,
-                claimRewardDate: block.timestamp + 7 days
+                claimRewardDate: block.timestamp + stakingPeriod
            })
        );
        // ...
    }
```

### [L-7] In `PinLink_Staking.stake()` and `unstake()`, consider adding the stake index to the event

Missing the stake index in the event in `PinLink_Staking.stake()` and `unstake()`.

#### Recommendation

Add the stake index to the events.

```diff
     // ...
-    event EventStaked(address by, uint256 amount);
+    event EventStaked(address by, uint256 amount, uint256 index);
-    event EventUnstaked(address by, uint256 amount);
+    event EventUnstaked(address by, uint256 amount, uint256 index);

     // ...

    function stake(uint256 amount) external {
        // ...
        emit EventStaked(msg.sender, amount, userStakes.length - 1);
    }

    function unstake(uint256 index) external {
        // ...
        emit EventUnstaked(msg.sender, amount, index);
    }
```

### [L-8] Incorrect visibility modifier in `PinLink_Staking.calculateTotalRewardAmount()`

If the function `PinLink_Staking.calculateTotalRewardAmount()` is thought to be public, the modifier should be changed to `public`, as it is currently declared as `internal`.

#### Recommendation

Make `calculateTotalRewardAmount()` public.

```diff
    function calculateTotalRewardAmount(
        address user
-    ) internal view returns (uint) {
+    ) public view returns (uint) {
      // ...
    }
```

### [L-9] In `PinLink_Staking`, missing events for `updateStakingPeriod()` and `updatePercentage()` and `claimReward()`

Missing events for `updateStakingPeriod()` and `updatePercentage()` and `claimReward()`. Consider adding events for these functions to track the changes.

#### Recommendation

Add events for `updateStakingPeriod()` and `updatePercentage()` and `claimReward()`.

```diff
    // ...
+    event StakingPeriodUpdated(uint256 stakingPeriod);
+    event PercentageUpdated(uint256 percentage);
+    event RewardClaimed(address account, uint256 amount);

    // ...

    function updateStakingPeriod(uint _stakingPeriod) external onlyOwner {
        stakingPeriod = _stakingPeriod;
+       emit StakingPeriodUpdated(_stakingPeriod);
    }

    function updatePercentage(uint _percentage) external onlyOwner {
        percentage = _percentage;
+       emit PercentageUpdated(_percentage);
    }

    function claimReward() external {
        // ...

        for (uint i = 0; i < userStakes.length; i++) {
          // ...
        }

+       emit RewardClaimed(msg.sender, rewardAmount);
    }
```

## Gas savings

### [GAS-1] Use rentable from memory instead of accessing the `rentables[tokenId]` array

In `Marketplace.takeOnRent()`, use the already defined `rentable` from memory instead of accessing the `rentables[tokenId]` array. In Solidity, reading from and writing to storage is significantly more expensive in terms of gas cost compared to reading from memory. Therefore, when accessing a storage variable multiple times within a function, it is more efficient to read it once and store its value in a local memory variable

#### Impact: Gas/optimization

Gas savings when accessing memory variables instead of storage.

#### Recommendation

Change `rentables[tokenId]` to the previously defined `rentable`.

```diff
    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {
        Rentable memory rentable = rentables[tokenId];
        require(rentable.pauseTime == 0, "renting paused");
        require(rentable.amount - rentable.rentedAmount >= tokenAmount, "not enough fractions available");
-        IERC20(rentables[tokenId].collateral).safeTransferFrom(msg.sender, address(this), tokenAmount * rentable.collateralPerUnit);
+        IERC20(rentable.collateral).safeTransferFrom(msg.sender, address(this), tokenAmount * rentable.collateralPerUnit);
        if (rentals[tokenId][msg.sender].amount > 0) { // already rented
            uint256 rewardAmount = calculateReward(msg.sender, tokenId, tokenAmount);
            rewardToken.safeTransfer(msg.sender, rewardAmount);
        }
        else {
            tokenRentees[tokenId].push(msg.sender);
        }
        rentals[tokenId][msg.sender].amount += tokenAmount;
        rentals[tokenId][msg.sender].collateralPerUnit = rentable.collateralPerUnit;
        rentals[tokenId][msg.sender].rewardRate = rentable.rewardRate;
        rentals[tokenId][msg.sender].startTime = block.timestamp;
-        rentables[tokenId].rentedAmount += tokenAmount;
+        rentable.rentedAmount += tokenAmount;
        rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, tokenAmount, "");

        emit Rent(msg.sender, tokenId, tokenAmount);
    }
```

### [GAS-2] In `Marketplace` store `pauseDuration` in `rentable` instead of each rental to avoid a for loop

In `Marketplace.adjustPauseRewards()`, a much more efficient approach would be to store the `pauseDuration` in the rentable itself, and only adjust it there as "accumulatedPausedDuration".

#### Impact: Gas/optimization

Gas savings when removing a for loop.

#### Recommendation

Store `pauseDuration` in `rentable` instead of each rental.

```diff
    struct Rentable {
        address renter;
        uint256 amount;
        address collateral;
        uint256 collateralPerUnit;
        uint256 rewardRate;
        uint256 rentedAmount;
        uint256 pauseTime;
+        uint256 accummulatedPauseDuration;
    }

    struct Rental {
        uint256 amount;
        uint256 collateralPerUnit;
        uint256 rewardRate;
        uint256 startTime;
-        uint256 pauseDuration;
+       uint256 currentAccumulatedPauseDuration;
    }

    // ...

    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {

        // ...

        rentals[tokenId][msg.sender].collateralPerUnit = rentable.collateralPerUnit;
        rentals[tokenId][msg.sender].rewardRate = rentables[tokenId].rewardRate;
        rentals[tokenId][msg.sender].startTime = block.timestamp;
+        rentals[tokenId][msg.sender].currentAccumulatedPauseDuration = rentables[tokenId].accumulatedPauseDuration;
        rentables[tokenId].rentedAmount += tokenAmount;
        rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, tokenAmount, "");

        emit Rent(msg.sender, tokenId, tokenAmount);
    }

    // ...

    function calculateReward(address account, uint256 tokenId, uint256 tokenAmount) private view returns (uint256) {
        Rental memory rental = rentals[tokenId][account];
-        uint256 duration = (block.timestamp - (rental.startTime + rental.pauseDuration)) / 1 hours;
+        uint256 rentalPauseDuration = rentables[tokenId].accumulatedPauseDuration - rental.currentAccumulatedPauseDuration;
+        uint256 duration = (block.timestamp - (rental.startTime + rentalPauseDuration)) / 1 hours;
        uint256 reward = tokenAmount * rental.rewardRate * duration;
        return reward;
    }

    // ...

    function adjustAllPauseRewards(uint256 tokenId, uint256 pauseTime, uint256 resumeTime) internal {
-       address[] memory rentees = tokenRentees[tokenId];
-       for (uint256 i = 0; i < rentees.length; i++) {
-           adjustPauseRewards(rentees[i], tokenId, pauseTime, resumeTime);
-        }

+        Rentable storage rentable = rentables[tokenId];
+        rentable.accumulatedPauseDuration += pauseEndTime - pauseStartTime;
    }


-    function adjustPauseRewards(address account, uint256 tokenId, uint256 pauseTime, uint256 resumeTime) internal {
-        Rental storage rental = rentals[tokenId][account];
-        if (rental.startTime < pauseTime) {
-            rental.pauseDuration += (resumeTime - pauseTime);
-        }
-    }

```

### [GAS-3] In `Marketplace.list()`, apply caching in `rentable.renter`

In `Marketplace.list()`, apply caching in `rentable.renter` to avoid multiple storage reads.

#### Impact: Gas/optimization

Gas savings when avoiding multiple storage reads.

#### Recommendation

Store `rentable.renter` in a local variable.

```diff
    function list(uint256 tokenId, uint256 tokenAmount, address collateralToken, uint256 collateralPerUnit, uint256 rewardRatePerHour) external onlyRole(RENTER_ROLE) {

        Rentable storage rentable = rentables[tokenId];
+        address renter = rentable.renter;
-        require(rentable.renter == address(0) || rentable.renter == msg.sender, "already rented");
+        require(renter == address(0) || renter == msg.sender, "already rented");
        require(rentableToken.balanceOf(msg.sender, tokenId) >= tokenAmount, "renter must own the token");
        rentableToken.safeTransferFrom(msg.sender, address(this), tokenId, tokenAmount, "");

-        if (rentable.renter == address(0)) {
+        if (renter == address(0)) {
            rentable.renter = msg.sender;
            rentable.collateral = collateralToken;
        }

        rentable.amount += tokenAmount;

        rentable.collateralPerUnit = collateralPerUnit;
        rentable.rewardRate = rewardRatePerHour;

        emit List(msg.sender, tokenId, tokenAmount, rentable.collateral, collateralPerUnit, rewardRatePerHour);
    }
```

### [GAS-4] In `Marketplace.list()`, apply conditional checks before writing `collateralPerUnit` and `rewardRatePerHour`

In `Marketplace.list()`, apply conditional checks before writing `collateralPerUnit` and `rewardRatePerHour` to storage when the values don't change to save gas. It is likely that in re-listing a token, the collateral and reward rate will not change, in which case writing to storage is unnecessary.

#### Impact: Gas/optimization

Gas savings when avoiding unnecessary writes to storage.

#### Recommendation

Add a check before writing to storage.

```diff
    function list(uint256 tokenId, uint256 tokenAmount, address collateralToken, uint256 collateralPerUnit, uint256 rewardRatePerHour) external onlyRole(RENTER_ROLE) {

        Rentable storage rentable = rentables[tokenId];
        address renter = rentable.renter;
        require(renter == address(0) || renter == msg.sender, "already rented");
        require(rentableToken.balanceOf(msg.sender, tokenId) >= tokenAmount, "renter must own the token");
        rentableToken.safeTransferFrom(msg.sender, address(this), tokenId, tokenAmount, "");

        if (renter == address(0)) {
            rentable.renter = msg.sender;
            rentable.collateral = collateralToken;
        }

        rentable.amount += tokenAmount;

+        if (rentable.collateralPerUnit != collateralPerUnit) {
            rentable.collateralPerUnit = collateralPerUnit;
+        }

+        if (rentable.rewardRate != rewardRatePerHour) {
            rentable.rewardRate = rewardRatePerHour;
+        }

        emit List(msg.sender, tokenId, tokenAmount, rentable.collateral, collateralPerUnit, rewardRatePerHour);
    }
```

### [GAS-5] Redundant renter address check in `Marketplace.delist()` `isValidRenter`

In `Marketplace.delist()`, the first `require` is redundant since it is already verified through the modifier `isValidRenter`.

#### Impact: Gas/optimization

Gas savings when removing a redundant requirement.

#### Recommendation

Remove the first `require`:

```diff
    function delist(uint256 tokenId) external isValidRenter(tokenId)  {
        Rentable memory rentable = rentables[tokenId];

-        require(rentable.renter == msg.sender, "invalid renter");

        if (rentable.rentedAmount > 0) {
            address[] memory rentees = tokenRentees[tokenId];
            for (uint256 i = 0; i < rentees.length; i++) {
                returnOnRent(rentees[i], tokenId, rentals[tokenId][rentees[i]].amount);
            }
        }

        if (withdraw) {
            rentableToken.safeTransferFrom(address(this), msg.sender, tokenId, rentable.amount, "");
            delete rentables[tokenId];
        }

        emit Delist(tokenId, withdraw);
    }
```

### [GAS-6] Add indexes to the functions to avoid looping through the same elements

In `PinLink_Staking.unstakeAll()` and `claimReward()`, add indexes to the function to avoid looping through the same elements in every call.
Batch calling the function will significantly reduce the gas cost by targeting specific elements.

Moreover, the indexes will avoid a possible gas exhaustion in the functions when `userStakes` is not limited.

#### Impact: Gas/optimization

Gas savings when adding indexes to the functions.

#### Recommendation

Add indexes to the functions.

```diff
-    function unstakeAll() external {
+    function unstakeAll(uint startIndex, uint endIndex) external {

+       require(startIndex < endIndex, "Invalid indexes order");
+       require(endIndex <= userStakes[msg.sender].length, "Invalid end index");

        Stake[] storage userStakes = stakes[msg.sender];

        for (uint i = startIndex; i < endIndex; i++) {
            if (!userStakes[i].withdrawFlag) {
                require(
                    token.transfer(msg.sender, userStakes[i].amount),
                    "Token transfer failed"
                );
                userStakes[i].withdrawFlag = true;
                userStakes[i].withdrawDate = block.timestamp;
                totalStakedAmount -= userStakes[i].amount;
            }
        }
    }

    // ...

-    function claimReward() external {
+    function claimReward(uint startIndex, uint endIndex) external {

+       require(startIndex < endIndex, "Invalid indexes");
+       require(endIndex <= userStakes[msg.sender].length, "Invalid end index");

        Stake[] storage userStakes = stakes[msg.sender];

        uint allUsersTotalStakedAmount = totalStakedAmount;
        uint totalBalance = token.balanceOf(address(this));

        for (uint i = startIndex; i < endIndex; i++) {
            if (
                !userStakes[i].withdrawFlag &&
                userStakes[i].stakedDate + stakingPeriod <= block.timestamp &&
                userStakes[i].claimRewardDate <= block.timestamp
            ) {

                uint userReward = (userStakes[i].amount /
                    allUsersTotalStakedAmount) *
                    ((totalBalance *
                        percentage *
                        (block.timestamp - userStakes[i].stakedDate)) / 30);

                require(
                    token.transfer(msg.sender, userReward),
                    "Token transfer failed"
                );

                userStakes[i].claimRewardDate = block.timestamp; // Reset claim reward date
            }
        }
    }

```
