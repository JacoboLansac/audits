# Blendr - Staking contract security review

A time-boxed security review of the [**Blendr Staking contract**](https://twitter.com/BlendrNetwork) protocol, with a focus on smart contract security.

Author: [**Jacopod**](https://twitter.com/jacolansac), independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits?tab=readme-ov-file).

## Findings Overview


| Finding | Description                                                                                                      | Severity | Status |
| ------- | ---------------------------------------------------------------------------------------------------------------- | -------- | ------ |
| \[H-1\] | Updating the `flexibleApy` will break the protocol accounting as it will modify the size of pending rewards for all users since their last interaction with flexible staking | High  |   |
| \[H-2\] | No explicit handling of `stakingToken` funds might lead to protocol insolvency as there will be insufficient funds to pay rewards and withdrawals | High  |   |
| \[H-3\] | Setting a new stakingToken can result in users being unable to recover their funds | High  |   |
| \[M-1\] | [Centralization risk] The function `emergencyWithdraw()` allows the contract owner to drain all the contract funds | Medium  |   |
| \[M-2\] | Missing input validation in `setPenaltyRate()` can lead to transactions reverted when calling `withdrawFixedStake()` | Medium |   |
| \[L-1\] | The event `StakedFixed()` inisde `createFixedStake()` emits a wrong `stakeId` | Low  |   |
| \[L-2\] | Removing items from the array of `fixedStakes` mixes up `stakeIndexes` between different stakes | Low  |   |
| \[L-3\] | The event `RewardClaimedFixedStake()` will always emit a wrong value of `rewards` corresponding to the `stakeIndex + 1` instead of `stakeIndex`. | Low  |   |
| \[L-4\] | Misleading output of the view function `calculateAPY()`  | Low  |   |
| \[L-5\] | Creating fixed stakes with a duration longer than `maxFixedDurationDays` should revert in `createFixedStake()`  | Low  |   |
| \[L-6\] | Unnecessary decimal loss due to inefficient placement of terms in divisions | Low  |   |
| \[L-7\] | APYs are in base 100, which gives no granularity when interpolating in the calculation of APYs. | Low  | Acknowledged  |
| \[L-8\] | Move external calls to the end of the function (follow CEI pattern) | Low  |   |
| \[L-9\] | Penalty is not proportional to the remaining time left | Low  | Acknowledged |
| \[L-10\] |  Missing events in state-changing functions | Low  |   |

----------

# Introduction

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
| Likelihood: High   |     High     |      High      |   Medium    |
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

- Delivery date: `2024-04-21`
- Duration of the audit: 3 days
- Audited commit
  hash: [96c985536ac448ba1385ae687bac36281bb9e0f6](https://github.com/blendr-network/staking-contract/commit/96c985536ac448ba1385ae687bac36281bb9e0f6)
- Review Commit hash: [ ... ]

- The following contracts are in the scope of the audit:

  | File                          | nSLOC |
  | ----------------------------- | ----- |
  | `lib/src/StakingPlatform.sol` | 195   |

# Protocol Overview

From the protocol readme:

*The BlendrStakingPlatform contract offers a versatile and upgradeable solution for staking ERC20 tokens, providing users with flexible and fixed-term staking options. Developed with the Solidity programming language and leveraging OpenZeppelin's suite of secure, standard contracts, the BlendrStakingPlatform is designed to be both secure and user-friendly, ensuring that stakeholders can earn rewards on their token investments in a transparent and efficient manner.*

## Architecture high level review


- The architecture of the staking contract mixes flexible and fixed staking, making a bit more complicated to keep track of the accounting of stakes,rewards etc. Splitting the logic into two different contracts (one for fixed-duration stakes and another one for variable-duration) is recommended.
- The contract does not keep track of basic accountability (how many tokens have been staked, how many rewards have been allocated to fixed-stakings, etc), which can lead so some serious consequences that are described more in depth in the findings of next section. Splitting the staking contract in two will allow tracking of these accounting information much easier. 
- The staking contract is upgradeable. This is not recommended. A staking contract holding user funds with a rather simple logic shoud not need upgrades. Upgradeability offers deverlopers the posibility of rug pulling all user funds, so it requires a degree of trust from the users towards the protocol that is not aligned with the ethereum principles of decentralization. Deploying an new immutable audited staking contract is suggested, and let users move the funds to that new staking contract. 

# Findings

## High risk

### [H-1] Updating the `flexibleApy` will break the protocol accounting as it will modify the size of pending rewards for all users since their last interaction with flexible staking

The rewards for flexible stakes are accrued every time an account interacts with `stake()`, `unstake()`, and `claimReward()`, which update the `lastUpdated` parameter for the user. The amount of rewards is calculated proportionally to the time duration since `lastUpdated` and to the `flexibleApy`. 

Whenever `flexibleApy` is updated, it does not only affect future rewards, but it also changes the rewards of all users since the last time they were accrued (since `lastUpdated`). 

- If the new `flexibleApy` is lower than before, all users see their rewards reduced even those rewards would be considered already "earned" 
- If the new `flexibleApy` is higher than before, all users have suddenly more rewards since the last time they were accrued. Thus the protocol will face a higher debt to the users in terms of rewards.

##### Impact: high

- Users can lose rewards unexpectedly if the new `flexibleApy` is lower than before. 
- The protocol will face a sudden step in the tokens owed to the contract in the form of rewards.

#### Proof of concept

Let's imagine the following scenario.

- Bob stakes 1000 tokens, and the current APY is `10%`.
- Bob waits for 3 months, without claiming rewards, he should have accrued 25 okens (10% APY for 4 months). He is entitled now to those 25 tokens if he claimed them now. However, he doesn't claim.
- Now the contract owner calls `setAPY()` and sets `flexibleApy` to `5%` (half of before).
- Suddenly, the rewards accrued by Bob decrease from 25 tokens to 12.5. He has been unexpectedly penalized even though he did nothing wrong

#### Recommendation

One of the following:
- Don't allow changing the `flexibleApy` parameter once it has been initialized.
- Remove all flexible-stakes-related logic and move them to a separate contract

#### Team response:

- The team will remove the `flexibleApy` staking logic and move it to a separate contract. The team has been made aware of not changing the storage layout to avoid storage collisions in contract upgrades (except renaming variables). 
- Renaming unused variables with the "_deprecated" suffix is recommended, for readability purposes. 

### [H-2] No explicit handling of `stakingToken` funds might lead to protocol insolvency as there will be insufficient funds to pay rewards and withdrawals

The contract does not keep track explicitly of where the `stakingToken` balance is allocated within the contract. The staked tokens blend together with the tokens for rewards, and also with the penalties applied for early withdraws. Besides the lack of accounting and obscurity of where the funds are, this can lead to more problematic situations

If the contract owner fails to deposit enough rewards for the fixed stakes, users withdrawing their stake including their reward will remove tokens that would correspond to the stakes of other users that can unstake until a later time. The users that unstake the latest may face a situation where there is not enough tokens in the contract balance to cover their initial stake, let alone the rewards. 

#### Impact: high

If the contract owner fails to deposit rewards, the protocol becomes insolvent, and users cannot withdraw their tokens because they have been paid out to other users as rewards

Users should always be able to withdraw **at least** their staked tokens, even if the contract runs out of funds to pay rewards. This is not the case in the current implementation, since rewards will be payed as rewards to the early unstakers, leaving the later ones without their staked tokens.

#### Recommendation

The general recommendation is to keep track of stakes and rewards. Flexible stakes complicate things for this, so it is recommended to remove the entire logic related to flexible stakes. This will allow keeping track of three main stake variables for accounting:
- `totalStaked`: the sum of all tokens staked by different users. The contract balance should never have less than this amount
- `tokensReservedForRewards`: the sum of all tokens reserved for rewards. These can be calculated at the moment of creating fixed stakes. Ideally, the function to create fixed stake should revert if there are not enough idle tokens in the contract to face these rewards
- `tokensFromPenalties`: sum of all penalties applied. These tokens can be withdrawn by the owner. 

Complementing this solution, I suggest to include specific functions to handle those balances:
- `depositTokensForRewards()`: function for the owner to transfer tokens to the contract, but adding them to the specific balance used for paying rewards. In this way they will not be mixed with other accounting. 
- `withdrawTokensFromPenalties()`: This function can remove Blendr tokens corresponding from penalties. No more, no less. This will allow modifying `emergencyWithdraw()` so that the `stakeToken` cannot be withdrawn with this function, to minimize the chance of rug pull

#### Team response:

- TBD

### [H-3] Setting a new stakingToken can result in users being unable to recover their funds

The function `setStakingToken()` allows the contract owner to change the token that is used for staking. All current stakings are related to the previous token, so changing the stakeToken will simply mess the accounting completely. Moreover, the contract does not provide a way for users to recover their funds if the token is changed. This leads to a situation where users are unable to recover their original funds if the token is changed.

#### Proof of concept
- Bob stakes 1000 tokens of `stakeToken`, which is currently `tokenA`
- Contract owner changes `stakeToken` from `tokenA` to `tokenB`
- Bob attempts withdrawing his stake of 1000 tokens. The contract attempts to send 1000 `tokenB` (as it is the new `stakeToken`), but the function reverts, as there are not enough tokens of `tokenB` in the contract balance. User funds are locked.

#### Impact: high
- Protocol will become insolvent if the token is changed as users will not be able to unstake their tokens.
- Contract owner can steal all user funds

#### Recommendation

Remove the `setStakingToken()` function and simply deploy a new staking contract if there is ever a new `stakeToken`.

#### Team response:

The team will remove the `setStakingToken()` function to make the `stakeToken` immutable (except for the fact that the contract is upgradeable, obviously).

---------------------

## Medium risk

### [M-1] [Centralization risk] The function `emergencyWithdraw()` allows the contract owner to drain all the contract funds

The function `emergencyWithdraw()` allows the contract owner to withdraw all staked funds from the contract. 

#### Impact: medium

Users can lose all their funds staked in the contract.

- Probability: low, because it would require a malicious contract owner or a compromised private-key or multisig to happen.
- Impact: high, as users can lose all their funds.

#### Recommendation

The team intended to have a function to recover lost ERC20 tokens sent by mistake to the contract. If this is the case, the function should not allow to withdraw `stakeToken`. Or at least, it should keep track of the accounting and only allow withdrawing tokens that are not part of staked funds, or rewards or penalties.  

#### Team response:
- TBD

### [M-2] Missing input validation in `setPenaltyRate()` can lead to transactions reverted when calling `withdrawFixedStake()`

The function `withdrawFixedStake()` applies a penalty on the staked amount. This penalty is expected to be a percentage, measured in base 100. However, if the `penaltyRate` parameter is set to a value greater than 100%, it will lead to reverted transactions when attempting to withdraw a fixed stake, because the `penalty` value subtracted will be higher than `userStake.amount` in:

 unexpected and incorrect behavior due to the way the penalty is calculated and applied in

```javascript
    function withdrawFixedStake(uint256 stakeIndex) external nonReentrant {
        // ...
        uint256 penalty = (userStake.amount * penaltyRate) / 100;
@>      uint256 amountToTransfer = userStake.amount - penalty;
        // ...
    }
```
#### Impact: medium
- Probability: low, as it requires either a mistake by the owner, or a malicious owner / compromised private key.
- Impact: high, as `withdrawFixedStake()` will revert until changed back. So it could be a sofisticated way of rug pulling.

#### Recommendation

Add a range for the `penaltyRate` parameter in the `setPenaltyRate()` function to prevent the contract owner from setting a penalty rate that is too high (over 100%).

```diff
    function setPenaltyRate(uint256 rate) external onlyOwner {
+       if (rate > 100) revert("rate can't exceed 100");
        penaltyRate = rate;
    }

```

#### Team response:
- TBD
 
------------------

## Low risk

### [L-1] The event `StakedFixed()` inisde `createFixedStake()` emits a wrong `stakeId`

The event emmits an `stakeId` equal to the length of the stakes, but should be the `length-1`. Index in solidity start from `0`, so when the user creates the first fixed stake, the `stakeId` emitted should be `0`. However, the event emits the lenght, which is `1`, as the event is emitted after pushing the new `FixedStake` into the array, and therefore the array has `length=1` when the event is emitted..

```javascript
    function createFixedStake(uint256 amount, uint256 durationDays) external nonReentrant {
        // ...

@>      fixedStakes[msg.sender].push(
            FixedStake({
                amount: amount,
                startDate: block.timestamp,
                endDate: endTime,
                duration: durationDays * SECONDS_IN_A_DAY,
                apy: apy,
                reward: reward
            })
        );

        // ...

@>      emit StakedFixed(msg.sender, fixedStakes[msg.sender].length, amount, durationDays);
    }
```

#### Impact: low

The event `StakedFixed()` will always emit a wrong `stakeId` which will be increamented by `1`.

#### Recommendation

```diff
    function createFixedStake(uint256 amount, uint256 durationDays) external nonReentrant {
        // ...

-       emit StakedFixed(msg.sender, fixedStakes[msg.sender].length, amount, durationDays);
+       emit StakedFixed(msg.sender, fixedStakes[msg.sender].length - 1, amount, durationDays);
    }
```

### [L-2] Removing items from the array of `fixedStakes` mixes up `stakeIndexes` between different stakes

When a user creates a stake with `createFixedStake()`, an event is emitted with the index of that stake.

However, if the user calls `claimFixedStakeReward()` or `withdrawFixedStake()` the stake corresponding to `stakeIndex` is removed from the array, by moving the last element to `stakeIndex`, and then poping the last element.

This mixes the stake indexes, as the `stakeIndex` emitted originally when staking index `i`, can be later replaced by other stake. As there is no iterations over the stakes, and claims are done per-stake, it is simply unnecessary to pop the elments changing them from one index to another. It would be simpler and more gas efficient to simply **delete** the `stakeIndex`. Therefore the array length never changes, but some elements will be uninitialized as they are being claimed/withdrawn.

#### Impact: low

The `stakeIndex` emitted when staking becomes unreliable, as the stakes are being moved around in the array.

#### Recommendation

Don't move stakes around in the array. Simply delete the struct to reset all fields to 0.

```diff
    function claimFixedStakeReward(uint256 stakeIndex) external nonReentrant {
        require(stakeIndex < fixedStakes[msg.sender].length, "Invalid stake index");
        FixedStake storage userStake = fixedStakes[msg.sender][stakeIndex];

        require(block.timestamp >= userStake.endDate, "Stake duration not yet completed");

        stakingToken.safeTransfer(msg.sender, userStake.amount + userStake.reward);

        uint256 reward = userStake.reward;

+       delete fixedStakes[msg.sender][stakeIndex];
-       uint256 lastIndex = fixedStakes[msg.sender].length - 1;
-       if (stakeIndex != lastIndex) {
-           fixedStakes[msg.sender][stakeIndex] = fixedStakes[msg.sender][lastIndex];
-       }
-       fixedStakes[msg.sender].pop();

        emit RewardClaimedFixedStake(msg.sender, stakeIndex, userStake.reward);
    }
```

### [L-3] The event `RewardClaimedFixedStake()` will always emit a wrong value of `rewards` corresponding to the `stakeIndex + 1` instead of `stakeIndex`.

When a fixed stake is claimed, the `FixedStake` struct is popped from the array of `fixedStakes`.
However, an event is emitted after the removal of the element, in which one of the fields attempts to read the `rewards` field form the storage pointer of the recently deleted index:

```javascript
    function claimFixedStakeReward(uint256 stakeIndex) external nonReentrant {
        require(stakeIndex < fixedStakes[msg.sender].length, "Invalid stake index");
        FixedStake storage userStake = fixedStakes[msg.sender][stakeIndex];

        require(block.timestamp >= userStake.endDate, "Stake duration not yet completed");

        stakingToken.safeTransfer(msg.sender, userStake.amount + userStake.reward);

        uint256 lastIndex = fixedStakes[msg.sender].length - 1;
        if (stakeIndex != lastIndex) {
            fixedStakes[msg.sender][stakeIndex] = fixedStakes[msg.sender][lastIndex];
        }
@>      fixedStakes[msg.sender].pop();

@>      emit RewardClaimedFixedStake(msg.sender, stakeIndex, userStake.reward);
    }
```

This means that when the event is emitted, the storage pointer is still pointing to `stakeIndex`, which has just been removed. Usually this will mean that the `rewards` field will be read from `stakeIndex + 1`. 

A particularly interesting case is when claiming the last `stakeIndex`. When this one is deleted, the storage pointer will be pointing to a non-existing element in the array, so the rewards emitted in the event will be `0`.Luckly, this doesn't revert, because otherwise a user would never be able to claim the last `FixedStake`.

#### Impact: low

The `rewards` field in the event `RewardClaimedFixedStake()` will always be wrong, emitting the rewards from `stakeIndex+1`, or even emitting `rewards=0` in the case of claiming the last stake. 

#### Recommendation

- Don't remove elements from the array. 
- Alternatively, cache the `userStake.rewards` in a memory variable for the event.

### [L-4] Misleading output of the view function `calculateAPY()` 

The function `calculateAPY()` returns `minFixedAPY` when `durationDays <= minFixedDurationDays`. 
This is misleading for two reasons:
- Users are not allowed to stake for less than `durationDays`
- Users would see the same apy for 0 days all the way up to `minFixedDurationDays`, instead of being increasing values as it happens from `minFixedDurationDays` to `maxFixedDurationDays`

#### Impact: low

Just a misleading value for the frontend.

#### Recommendation

The output should be 0 if `durationDays <= minFixedDurationDays`.

```diff
    function calculateAPY(uint256 durationDays) public view returns (uint256) {
-        if (durationDays <= minFixedDurationDays) {
-           return minFixedAPY;
+        if (durationDays < minFixedDurationDays) {
+           return 0;
        } else if (durationDays >= maxFixedDurationDays) {
            return maxFixedAPY;
        } else {
            // Linear interpolation formula
            uint256 durationRange = maxFixedDurationDays - minFixedDurationDays;
            uint256 apyRange = maxFixedAPY - minFixedAPY;
            uint256 durationFromMin = durationDays - minFixedDurationDays;
            // ...
        }
    }
```

### [L-5] Creating fixed stakes with a duration longer than `maxFixedDurationDays` should revert in `createFixedStake()` 

Users are able to create fixed stakes that last longer than the `maxFixedDurationDays` defined in the contract. 
However, they get the same APY as staking for `maxFixedDurationDays`, percieving less effective APYs if they stake for longer than the maximum allowed.

#### Recommendation

The function `createFixedStake()` should revert when `durationDays` is higher than `maxFixedDurationDays`.

```diff
    function createFixedStake(uint256 amount, uint256 durationDays) external nonReentrant {
        // ...
        require(amount > 0, "Amount must be greater than 0");
        require(durationDays >= minFixedDurationDays, "Have to stake more than min duration days");
+       require(durationDays <= maxFixedDurationDays, "Have to stake less than max duration days");
        // ...
    }
```


### [L-6] Unnecessary decimal loss due to inefficient placement of terms in divisions

To minimize the impact of decimal loss in divisions, don't perform any division before all numerator terms are multiplied. Example in `createFixedStake()`:

```diff
    function createFixedStake(uint256 amount, uint256 durationDays) external nonReentrant {
        // ...

        uint256 apy = calculateAPY(durationDays);

        uint256 endTime = block.timestamp + (durationDays * SECONDS_IN_A_DAY);

-       uint256 reward = (((amount * apy) / 100) * durationDays * SECONDS_IN_A_DAY) / SECONDS_IN_A_YEAR;
+       uint256 reward = (amount * apy * durationDays * SECONDS_IN_A_DAY) / (100 * SECONDS_IN_A_YEAR);

        // ...
```

### [L-7] APYs are in base 100, which gives no granularity when interpolating in the calculation of APYs.

  Using basis points (i.e. base 10,000) is more common and permits more precision

For both flexible and fixed stakings, related APYs are in base 100 leading to loss of precission and few discrete outcomes. `flexibleAPY`, `minFixedAPY` and `maxFixedAPY` are all in base 100.

#### Impact: low

Users cannot have APY with decimal values. They can only have round values 10%, 11%, ... 18%. 

#### Recommendation

Use basis points to allow for decimals and more granular APYs. Rewards should then be divided by 10,000 instead of 100:

```diff
    function createFixedStake(uint256 durationDays) public view returns (uint256) {
        // ...

-       uint256 reward = (((amount * apy) / 10000) *
-           durationDays *
-           SECONDS_IN_A_DAY) / SECONDS_IN_A_YEAR;
+       uint256 reward = (((amount * apy) / 10000) *
+           durationDays *
+           SECONDS_IN_A_DAY) / SECONDS_IN_A_YEAR;

```

Same in `_calculateRewards()`:

```diff
    function _calculateRewards(address account) internal view returns (uint256) {
        // ...
-       uint256 ratePerSecond = (flexibleApy * PRECISION_FACTOR) /
-           SECONDS_IN_A_YEAR /
-           100;
+       uint256 ratePerSecond = (flexibleApy * PRECISION_FACTOR) /
+           SECONDS_IN_A_YEAR /
+           10000;
```

### [L-8] Move external calls to the end of the function (follow CEI pattern)

In `claimFixedStakeReward()` and `withdrawFixedStake()` the external call to `stakingToken.safeTransfer()` is done before the state changes. This is a violation of the Checks-Effects-Interactions pattern, as the external call should be done at the end of the function.

This is not a concern here as the tokens to transfer is a trusted ERC20, but it is one of the most important patterns to follow in terms of security, so it has to be mentioned. 

#### Recommendation

Move the external calls to the end of the function. Cache values in memory if needed (for `claimFixedStakeReward()`):

```diff
    function withdrawFixedStake(uint256 stakeIndex) external nonReentrant {
        // ...
-       stakingToken.safeTransfer(msg.sender, amountToTransfer);

        uint256 lastIndex = fixedStakes[msg.sender].length - 1;
        if (stakeIndex != lastIndex) {
            fixedStakes[msg.sender][stakeIndex] = fixedStakes[msg.sender][
                lastIndex
            ];
        }
        fixedStakes[msg.sender].pop();
+       stakingToken.safeTransfer(msg.sender, amountToTransfer);

        // ...
    }

    function claimFixedStakeReward(uint256 stakeIndex) external nonReentrant {
        // ...
-       stakingToken.safeTransfer(msg.sender, userStake.amount + userStake.reward);
+       uint256 stakeAmount = userStake.amount;
+       uint256 stakeReward = userStake.reward;

        uint256 lastIndex = fixedStakes[msg.sender].length - 1;
        if (stakeIndex != lastIndex) {
            fixedStakes[msg.sender][stakeIndex] = fixedStakes[msg.sender][lastIndex];
        }
        fixedStakes[msg.sender].pop();

-       emit RewardClaimedFixedStake(msg.sender, stakeIndex, userStake.reward);
+       emit RewardClaimedFixedStake(msg.sender, stakeIndex, stakeReward);
+       stakingToken.safeTransfer(msg.sender, stakeAmount + stakeReward);
    }

```

### [L-9] Penalty is not proportional to the remaining time left

Users with different staking time will percieve the same penalty when withdrawing their fixed stakes. A user with a few seconds left will be penalized the same as a user with 1 year left


```javascript
    uint256 penalty = (userStake.amount * penaltyRate) / 100;
```

#### Recommendation

Make the penalty proportional to the remaining time left. The penalty should be calculated as the proportional amount of the rewards that the user would have earned if the stake had been completed.

```diff
    uint256 penalty = (userStake.amount * penaltyRate * (userStake.endDate - block.timestamp)) / (100 * userStake.duration);
```

#### Team response: acknowledged

It was a concious design decission. 

### [L-10] Missing events in state changing functions

All the following functions perform important state changes but emit no events:
- `setAPY()`
- `setPenaltyRate()`
- `setFixedDurationDays()`
- `setFixedAPYRange()`
- `setFlexibleStakingStatus()`
- `setFixedStakingStatus()`
- `setStakingToken()`

Note that some of the above functions might be removed if some of the above recommendations are followed.

#### Recommendation: 

Emit events in all state changing functions for accountability reasons.


## Informational findings / gas savings

These findings do not bring any risk to the protocol, they are just informational and best practices.

### I-1 Unnecessary Reentrancy guards

Reentrancy Guards are unnecessary as there are no untrusted external calls. Lot of gas could be saved.

### I-2 Efficient storage packing to save gas

The two structs `FlexibleStake` and `FixedStake` could have been more gas efficient if using storage packing. 

However, as this is an upgradeable contract I **DO NOT** recommend changing this, as it would mess the storage layout of the proxy contract potentially creating high vulnerabilities.

### I-3 Redundant struct fields

In the `FixedStake` struct, one of the three following fields is redundant: `startDate`, `endDate` and `duration` as any of them can be calculated from the two others. Save gas by removing one redundant field.

### I-4 Use explicit prgamas

Set explicit pragma unless it is a library. 

```diff
- pragma solidity ^0.8.19;
+ pragma solidity 0.8.19;
```

### I-5 Use modifiers for accruing rewards

Consider including the logic for accruing rewards of flexible stakes in a common modifier like `accrueRewards()` instead of duplicating the logic everywhere:

```diff
+   modifier accrueRewards(address account) {
+       if (flexibleStakes[account].totalStaked > 0) {
+           flexibleStakes[account].totalRewards += _calculateRewards(account);
+       }
+   }
+   _;
```

And update the functions `stake()`, `unstake()` and `claimReward()`, although the last one requires slight modifications.

```diff
-   function stake(uint256 amount) external nonReentrant {
+   function stake(uint256 amount) external nonReentrant accrueRewards(msg.sender) {
```

### I-6 Reduce storage reads by passing storage pointers


Consider passing `userStake` as a parameter to `_calculateRewards()` instead of reading it from storage, to save gas. Otherwise the struct is read and loaded into memory twice.

```diff
-    function _calculateRewards(address user) internal view returns (uint256) {
+    function _calculateRewards(FixedStake storage userStake) internal view returns (uint256) {
```

### I-7 Cache results from repeated operations

In `createFixedStake()` consider caching `durationDays * SECONDS_IN_A_DAY` to avoid repeating the operation and save gas.

```diff
    function createFixedStake(uint256 amount, uint256 durationDays) external nonReentrant {
        // ...
+       uint256 durationInSeconds = durationDays * SECONDS_IN_A_DAY; // Cache the repeated calculation
```

### I-8 Unecessary initialization of storage variables

In `initialize`, `isFlexibleStakingEnabled` is set to `false`. This is unnecessary as the default value for `bool` datatypes is `false` already. Removing this saves gas.

### I-9 Use custom erros instead of string literals to save gas

It is a recommended best practice to use solidity custom errors instead of string literals. However, they cannot be used together with `require` statements, so I don't recommend changing this now. 

Nevertheless, I do recommend generally to use less than 32 bytes in string literals as they use less gas than using more than 32 bytes. 

