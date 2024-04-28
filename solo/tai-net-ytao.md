# TaiNet - Security review

A time-boxed security review of the [**TaiNet**](https://tainet.finance/) protocol, with a focus on smart contract security and gas optimizations.

Author: [**Jacopod**](https://twitter.com/jacolansac), independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits).

## Findings Overview


| Finding | Description                                                                                                      | Severity | Status |
| ------- | ---------------------------------------------------------------------------------------------------------------- | -------- | ------ |
| \[H-1\] | The exchange rate mechanism either makes the protocol vulnerable to sandwich attacks or dooms it to spend a huge amount of gas in keeping the rate updated  |  High |   |
| \[M-1\] | Lack of accountability of `wTAO` tokens can lead to users not being able to unstake their tokens |  Medium |   |
| \[M-2\] | Users will not be able to request unstakes if the ether transfer to the `withdrawalManager` fails.  |  Medium |   |
| \[C-1\] | CENTRALIZATION: There is no on-chain guarantee for depositors that they will receive rewards or their original stake |  Centralization |   |
| \[C-2\] | CENTRALIZATION: As `yTAO` is an upgradeable contract, user funds can be stolen if the contract is upgraded to a malicious implementation |  Centralization |   |
| \[C-3\] | CENTRALIZATION: Admin roles have the power to update the exchange rate between `yTAO` and `wTAO`   |  Centralization |   |
| \[L-1\] | In `requestUnstake()`, the `unstakingFee` is used as if it was measured in wTAO units, but it is expressed as a percentage (base 1000), making it trivial to bypass the requirement `yTAOAmt > unstakingFee` |  Low |   |
| \[L-2\] | Pulling ether from the `yTAO` contract will revert if the recipient is not an EOA |  Low |   |
| \[L-3\] | The `wrap()` function will revert when users attempt to wrap exactly `minStakingAmt` |  Low |   |
| \[L-4\] | The `wrap()` function requires that the caller's `wTAO` balance is higher than the amount to stake (`wtaoAmount`) but ignores the fees in the requirement |  Low |   |
| \[L-5\] | The function `approveMultipleUnstakes()` lacks protection against repeated unstake requests |  Low |   |
| \[L-6\] | The parameter `minStakingAmount` cannot be ever set to 0, due to a wrong requirement in `setMinStakingAmount()`  |  Low |   |
| \[L-7\] | The `maxDepositPerRequest` is not taken into account when calculating `maxTaoForWrap()` which will create reverts if users try to wrap the max amount |  Low |   |
| \[L-8\] | In future contract updates, maxSupply needs to be checked against initial supply  |  Low |   |
| \[L-9\] | The ussage of Ownable & Access control is redundant as they are both serving the same purpose. |  Low |   |
| \[L-10\] | Use SafeERC20 library for ERC20 transfers |  Low |   |
| \[G-1\] | Repeating the same check over and over for every request unstake |  Gas |   |
| \[G-2\] | Checking for the `wrappedToken` in every `UnstakeRequest` is expensive and unnecessary if the `wTAO` token is not expected to change |  Gas |   |
| \[G-3\] | The same array is iterated in three separated for-loops in the same function instead of performing all operations in the same loop |  Gas |   |
| \[G-4\] | Unnecessary repeated check every time the exchange rate is updated |  Gas |   |
| \[G-5\] | Unnecessary allowance check in `approveMultipleUnstakes()` |  Gas |   |
| \[G-6\] | Read length from cached memory in `requestUnstake` as it has already been read before |  Gas |   |
| \[G-7\] | Save gas by storing bytes instead of strings for `nativeWalletReceiver` |  Gas |   |
| \[I-1\] | The `checkPaused` modifier is used in `approveMultipleUnstakes()` |  Informational |   |
| \[I-2\] | The nonReentrant modifier should be placed first for security |  Informational |   |
| \[I-3\] | Wrong argument name in events  |  Informational |   |
| \[I-4\] | Naming inconsistencies |  Informational |   |
| \[I-5\] | Unnecessary initializations in `initialize()` |  Informational |   |
| \[I-6\] | Redundant check in `setUpperExchangeRateBound()` |  Informational |   |
| \[I-7\] | Avoid "magic" numbers and define constants instead (embedded in bytecode so they cost no gas) |  Informational |   |



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

- Delivery date: `2024-05-02`
- Duration of the audit: 7 days
- Audited commit
  hash: [dfd3cea4db30eafbd69fdb48caccc6dc043e230c](https://github.com/thevoidcoder/taiNET_contracts/commit/dfd3cea4db30eafbd69fdb48caccc6dc043e230c)
- Review Commit hash: [ ... ]

- The following contracts are in the scope of the audit:

  | File                          | nSLOC |
  | ----------------------------- | ----- |
  | `contracts/yTAO.sol` | 396   |

# Protocol Overview

- Users can deposit their `wTAO` tokens and receive `yTAO`. 
- Users can request unstaking by burning their `yTAO` tokens. However, an admin role needs to approve their request before actually being able to realize the unstake. 
- The contract is highly centralized, requiring a very high level of trust in the protocol admins. As such, there is a big risk of user funds being lost if admins act maliciously or private keys are compromised. 

## Architecture high level review

- There is a single contract, which is upgradeable, following the transparent proxy design pattern.
- The architecture requires a high level of activity from the protocol admins to function. This leads to the following points:
    - Inactivity from admins means that users' funds can be stuck in the contract until admins reactivate activity. 
    - The protocol owners will have very large gas costs related to operating the contract. Some of these gas costs are passed to the users as staking/unstaking/bridging fees

# Thread model

## Actors
- Users: can `wrap()`, `requestUnstake()` and `unstake()`. 
- Admin roles: can configure the contract, upgrade it, update the exchange rate, and approve unstake requests. 

## Worst-case scenarios: where is the value
- Users lose all staked funds
- Users can't unstake their tokens
- Users can't request more unstakes
- The attacker can unstake the same amount twice
- Griefing: Protocol loses value due to staking/unstaking gas costs or other mechanisms


# Findings

## High risk

### [H-1] The exchange rate mechanism either makes the protocol vulnerable to sandwich attacks or dooms it to spend a huge amount of gas in keeping the rate updated 

With large deviations from the free-market rate, wealthy attackers will be able to leech value from other honest depositors by depositing and requesting to unstake right before and right after an update in the exchange rate.

This is only profitable if the % delta in the exchange rate is higher than the unstaking Fee. However, note that the protocol needs to spend gas fees in every update of the exchange rate. This leads me to think that the exchange rate will most certainly deviate from the free market rate in DEXes.

#### Impact: Medium

- Probability: high, as it only requires a certain level of volatility and unaware admins approving the unstakes.  
- Impact: high, as the gains from the attacker are at the expense of the protocol, who will face those costs from their treasury

#### Proof of concept

- The exchange rate is measured in units of `[yTAO/wTAO]`, (so, how many `yTAO` per 1 wTAO). 
- Let's assume that the unstaking fee is set to 10% (of the unstaked value). 
- Let's assume the exchange rate in the market fluctuates rapidly, leaving a 15% drop with respect to the latest update of the exchange rate (so less `yTAO` per each wTAO).
- The authorized role in the protocol will call `updateExchangeRate()` to update the exchange rate to the new price. 
- The attacker frontruns the exchange rate update by calling `wrap()`. By front-running he gets the amount of `yTAO` corresponding to the original exchange rate, therefore receiving more `yTAO` than he should.
- The attacker backruns the tx calling `requestUnstake()`, which uses the fresh exchange rate to calculate the amount of wTAO. This will be +15% `wTAO` - 10% fees, so a net +5% in `wTAO` value.

#### Recommendation

It is true, that in order for it to be effective, the admin has to approve the unstake request, but this requires:
1. That the admins are aware of this leeching attack
2. They put in place the tools to filter out sandwich attackers... but... is it really fair to do this? that would lock their tokens, which is almost like stealing them. Not a very good solution either. 

Here are some alternatives, all of them with pros and cons:

1. Impose a short commitment period of a few days after wrapping. Example: users cannot request to unstake until 2 days have passed since the last time they called `wrap()`. However, users could call `wrap()` and then transfer to another wallet to bypass this protection.
2. Impose a restriction that a user cannot transfer `yTAO` tokens in the same block as they call `wrap()`. This can be implemented with an ERC20 hook such as `_beforeTokenTransfer()`. Although this would effectively kill the issue, it might lead to issues with protocols trying to integrate with yTAO. 

I personally favor option 2. 




----------------------------------------------------------

## Medium risk







### [M-1] Lack of accountability of `wTAO` tokens can lead to users not being able to unstake their tokens

The unstaking flow goes as follows:
1. User requests to unstake, by burning an amount of `yTAO` tokens. This registers the amount of `wTAO` tokens that should be given 
2. An admin with the `APPROVE_WITHDRAWAL_ROLE` approves the request and transfers `wTAO` to the contract corresponding to the principal staked + rewards - fees. 
3. The user realizes the unstake by calling `unstake()`, and the `wTAO` corresponding to principal + rewards are transfered to the user. 

> IMPORTANT:  step 3 requires that the `wTAO` tokens are in the contract's balance, otherwise `unstake()` reverts. 

On another hand, an admin with `TOKEN_SAFE_PULL_ROLE` has the power to withdraw ERC20 tokens from the contract (including the `wTAO`) using the function `safePullERC20()`.
When the admins pull `wTAO` using `safePullERC20()`, there is no on-chain restriction to not pull funds that are already allocated to unstaking. An admin can make the mistake of pulling more `wTAO` tokens than he should, leaving users without `wTAO` tokens in the balance to realize the unstakes.

#### Impact: Medium

- Probability: high, as it only takes a mistake in off-chain accounting to pull more funds than needed
- Impact: medium, as it can be considered a temporal lock of user funds. However, it would not be permanent, as the admins can fix it by transferring more `wTAO` tokens to the contract. 

#### Recommendation

Keep track of the balance allocated to untakes in a state variable. Increase it when unstakes are approved, and decrease it when unstakes are realized. When pulling `wTAO` tokens using `safePullERC20()`, never allow pulling the tokens allocated to approved unstakes.

```diff
+   /// The amount of `wTAO` in the contract balance allocated to approved unstakes.
+   uint256 public wTaoForUnstakes;

    // ...

    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // ...

+       wTaoForUnstakes += totalRequiredTaoAmt;

        // Transfer the tao from the withdrawal manager to this contract
        require(
            IERC20(commonWrappedToken).transferFrom(msg.sender, address(this), totalRequiredTaoAmt),
            "taoAmt transfer failed"
        );
    }

    // ...

    function unstake(uint256 requestIndex) public nonReentrant checkPaused {
        require(requestIndex < unstakeRequests[msg.sender].length, "Invalid request index");
        UnstakeRequest memory request = unstakeRequests[msg.sender][requestIndex];
        require(request.amount > 0, "No unstake request found");
        require(request.isReadyForUnstake, "Unstake not approved yet");

        // Transfer `wTAO` tokens back to the user
        uint256 amountToTransfer = request.taoAmt + request.rewardAmount;
        // Update state to false
        delete unstakeRequests[msg.sender][requestIndex];
        // Perform ERC20 transfer
        bool transferSuccessful = IERC20(request.wrappedToken).transfer(msg.sender, amountToTransfer);

+       wTaoForUnstakes -= amountToTransfer;

        require(transferSuccessful, "wTAO transfer failed");

        // Process the unstake event
        emit UserUnstake(msg.sender, requestIndex, block.timestamp);
    }

    // ...

    function safePullERC20(address tokenAddress, address to, uint256 amount) public hasTokenSafePullRole checkPaused {
        _requireNonZeroAddress(to, "Recipient address cannot be null address");

        require(amount > 0, "Amount must be greater than 0");

        IERC20 token = IERC20(tokenAddress);
        uint256 balance = token.balanceOf(address(this));
-       require(balance >= amount, "Not enough tokens in contract");
+       require(balance >= amount + wTaoForUnstakes, "Not enough tokens in contract");

        // "to" have been checked to be a non-zero address
        bool success = token.transfer(to, amount);
        require(success, "Token transfer failed");
        emit ERC20TokenPulled(tokenAddress, to, amount);
    }

```







### [M-2] Users will not be able to request unstakes if the ether transfer to the `withdrawalManager` fails. 

Transfering the `serviceFee` to the `withdrawalManager` is done as part of the main execution flow inside `requestUnstake()`:

```javascript
    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {

        // ...

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");

        // ..

@>      bool success = payable(withdrawalManager).send(serviceFee);
        require(success, "Service fee transfer failed");
    }
```

This is an anti-pattern, as any issues transferring the funds would impede the users to request unstakes. Such scenarios are:
- `withdrawalManager` is a contract without a `receive()` function. (This can be a mistake or intentional, by a malicious owner). 
- `withdrawalManager` returns `false` (by mistake or intentionally, being malicious).
- Future upgrades to Ethereum mainnet make the `send()` function obsolete as they only forward 2300 gas. Read more about the topic in [this article](https://blockchain-academy.hs-mittweida.de/courses/solidity-coding-beginners-to-intermediate/lessons/solidity-2-sending-ether-receiving-ether-emitting-events/topic/sending-ether-send-vs-transfer-vs-call/). The low-level `call()` is the recommended way to transfer native tokens. 

None of the above are likely, but they are very easily avoidable, or at least it is easy to not make every unstaker dependent on this call being successful. 

#### Impact: Medium

Temporal DoS, as the contract may become unusable to request unstakes. 

- Probability: low
- Impact: medium. It would be high, but it is solvable by a contract upgrade.

#### Recommendation

Favor the pullover push pattern with a state variable tracking the collected service fees. Also, use the low-level `call` instead of `send`:

```diff
+   // storage variable keeping track of total serviceFee collected
+   uint256 public serviceBalance;

    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {

        // ...

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");

        // ..

+       serviceBalance += serviceFee;
-       bool success = payable(withdrawalManager).send(serviceFee);
-       require(success, "Service fee transfer failed");
    }

+    function withdrawServiceFees() external {
+        require(msg.sender == withdrawalManager, "Not authorized");
+
+        // update state before the call to avoid reentrancy
+        uint256 amount = serviceBalance;
+        serviceBalance = 0;
+
+        bool success = payable(withdrawalManager).call{value:serviceBalance}("");
+        require(success, "Service fee transfer failed");
+    }

```


------------------------------



## Centralization risks




### [C-1] CENTRALIZATION: There is no on-chain guarantee for depositors that they will receive rewards or their original stake

- Users deposits `wTAO` tokens in the `yTAO` contract in exchange for rewards, also in `wTAO` tokens. When users deposit `wTAO` using `wrap()`, they receive `yTAO` as a receipt.
- To unstake, they need to submit a request by calling `requestUnstake`. In this same transaction, the corresponding amount of `yTAO` is burned. 
- Then the admins approve the unstake requests with `approveMultipleUnstakes()`, setting the rewards for each request (presumably calculated off-chain). 

However, the rewards distribution is entirely in the power of the contract admins, who sets the rewards to be given upon unstake approvals.
If the protocol was insolvent, or simply malicious, they could choose to:
- Accept the unstake requests, but grant 0 rewards.
- Don't approve the unstake requests, which is equivalent to not letting users unstake.

#### Impact: Medium

- Probability: low, as it requires malicious admins or compromised keys
- Impact: high, as users would lose their rewards or even their initial `wTAO` stakes

#### Recommendation

Unfortunately, the current architecture doesn't offer many alternatives, as the `wTAO` tokens are not in the `yTAO` contracts balance until the admins approve unstake requests. 
This is a risk that the users have to take, and they must therefore be aware of it.













### [C-2] CENTRALIZATION: As `yTAO` is an upgradeable contract, user funds can be stolen if the contract is upgraded to a malicious implementation

The owner of the transparent proxy has the power of upgrading the contract to a malicious implementation. 

#### Impact: Medium

Temporal DoS, as the contract may become unusable to request unstakes. 

- Probability: low, as it requires a malicious owner or compromised keys
- Impact: high, as users would lose all of their staked funds

#### Recommendation

The upgradeability feature should be protected with a multisig. 













### [C-3] CENTRALIZATION: Admin roles have the power to update the exchange rate between `yTAO` and `wTAO`  

With the power to update the `exchangeRate`, the `EXCHANGE_UPDATE_ROLE` role has the control over how many `yTAO` per `wTAO` are given in `wrap()` or `wTAO` per `yTAO` in `requestUnstake()`. 
A malicious or compromised admin address with such a role could front-run users doing several malicious actions:
- front-run `wrap()` with a very low exchange rate to decrease the amount of `yTAO` the user receives
- front-run `requestUnstake()` with a very high exchange rate to decrease the amount of `wTAO` the user gets back
- self-front-run his own `wrap()` with a high exchange rate to receive a disproportionate amount of `yTAO`

#### Impact: Medium
- Probability: low, as it requires a malicious admin or compromised keys
- Impact: high, as it could significantly decrease the value of users funds in the blink of an eye

#### Recommendation

Use all protections possible to protect the `EXCHANGE_UPDATE_ROLE`. 
Unfortunately, it seems to be a function that needs to be executed frequently, and it is time-sensitive. Therefore it is not very suitable for multisigs, as the delay time until all signatures are collected can have other serious impacts (outdated prices).











 --------------------------
 

## Low risk







### [L-1] In `requestUnstake()`, the `unstakingFee` is used as if it was measured in wTAO units, but it is expressed as a percentage (base 1000), making it trivial to bypass the requirement `yTAOAmt > unstakingFee`

In `requestUnstake()`, there is a requirement so that `yTAOAmnt` has to be higher than the `unstakingFee`. This requirement makes no sense as the `unstakingFee` is defined (and set) as a percentage (base 1000), and here is directly compared with `yTAOAmt`, which is expressed `wTAO` units (wei). Therefore, this requirement will be bypassed in the majority of cases. For example, if the unstaking fee is set to 10%, then `unstakingFee=100` (% in base 1000), and any unstake amount such that `yTAOAmt > 100 wei` will bypass the check. 

```javascript
    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {
        // Check that wrappedToken and withdrawalManager is a valid address
        _requireNonZeroAddress(address(wrappedToken), "wrappedToken address is invalid");
        _requireNonZeroAddress(address(withdrawalManager), "withdrawal address cannot be null");

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");
        
@>      require(yTAOAmt > unstakingFee, "Invalid yTAO amount");
        // Check if enough balance
        require(balanceOf(msg.sender) >= yTAOAmt, "Insufficient yTAO balance");
        uint256 outWTaoAmt = getWTAOByYTAOAfterFee(yTAOAmt);

        // ...

```

More generally, it doesn't make sense to require a minimum amount of something that is a percentage. Whatever `yTAOAmt` is, if the staking fee is a percentage, it will always be lower than that, so this check is fundamentally useless, even if correctly implemented. The requirement should be done with an absolute value of `yTAOAmnt` in wei. See more in the recommendation subsection.


Just as a reference, in the function `getWTAOByYTAOAfterFee()`, the `unstakingFee` is used (correctly) as a percentage with base 1000:

```javascript
    function getWTAOByYTAOAfterFee(uint256 yTaoAmount) public view returns (uint256) {
        uint256 feeAmount = (yTaoAmount * unstakingFee) / 1000;

        return ((yTaoAmount - feeAmount) * 1 ether) / exchangeRate;
    }
```

The issue has been probably introduced by the fact that comments are not consistent. The function `setUnstakingFee()` enforces that `unstakingFee` is never higher than 200, but it states it is measured as gwei (wrong) and in a later comment it is stated that it is expressed in percentage basis points, saying that 100% is 10000 (wrong, as the divisor is always 1000, so it is in base 1000, so 100%=1000). 

```javascript
    /*
    *
    * This function determines the unstaking fee that is charged to the user
    *
@>  * The value is in gwei so if _unstakingFee is 1 gwei, it means 1 TAO is charged as unstaking fee  // @audit-info wrong comment. It is expressed and used in base 1000. 1 == 1/1000 = 0.1%
    *
    * Value Boundary: UnstakingFee can be any value even 0 in the event that unstakingFee is not longer charged.
    *
    */
    function setUnstakingFee(uint256 _unstakingFee) public hasManageStakingConfigRole {
@>      require(_unstakingFee <= 200, "Unstaking fee cannot be more than 20%"); // 100% is 10000 in terms of percentage basis points  // @audit-info wrong comment. It is expressed in base 1000 everywhere
        unstakingFee = _unstakingFee;
        emit UpdateUnstakingFee(unstakingFee);
    }

```

#### Impact: Low

The impact is low because even though the requirement is passed almost always, the function `getWTAOByYTAOAfterFee()` will revert in the case that the requirement is not met. The only purpose I see for the -wrongly used- requirement is to throw a meaningful error when the unstaking amount is too low, instead of throwing an underflow, which is what would happen now.

Another consequence (with very low impact as well) is that the `feeAmount` calculated inside `getWTAOByYTAOAfterFee()` *can* be 0 when `(yTaoAmount * unstakingFee) < 1000`. 
For instance, if the team decided to lower the unstaking fee to 1%, they would set `unstakingFee=10`, so for every amount `yTaoAmount < 100`, no fee would be charged. This would not be very profitable due to fees, but it is worth taking it into account in case this issue could be combined with other issues to reach a bigger impact. 

#### Recommendation

Replace the following requirement with another one that checks with absolute values. A state variable `minimumUnstakeAmount` would be required.

```diff
+   uint256 public minimumUnstakeAmount = 1 * 1e9; // arbitrary value just for the example
    
    // ...

    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {
        // Check that wrappedToken and withdrawalManager is a valid address
        _requireNonZeroAddress(address(wrappedToken), "wrappedToken address is invalid");
        _requireNonZeroAddress(address(withdrawalManager), "withdrawal address cannot be null");

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");
        
-       require(yTAOAmt > unstakingFee, "Invalid yTAO amount");
+       require(yTAOAmt > minimumUnstakeAmount, "Invalid yTAO amount");

        // ...

    }
```







### [L-2] Pulling ether from the `yTAO` contract will revert if the recipient is not an EOA

The `pullNativeToken()` function uses a low-level call to withdraw ether from the contract. However, the datatype `address` does does not have the call method with value by default unless it is an EOA. Any attempts to transfer funds to contracts will fail (a Treasury contract, a smart contract wallet, a multisig).

```javascript
    function pullNativeToken(address to, uint256 amount) public hasTokenSafePullRole checkPaused {
        _requireNonZeroAddress(to, "Recipient address cannot be null address");
        require(amount > 0, "Amount must be greater than 0");

        uint256 balance = address(this).balance;
        require(balance >= amount, "Not enough native tokens in contract");


        // "to" have been checked to be a non zero address
@>      (bool success,) = to.call{value: amount}("");
        require(success, "Native token transfer failed");
        emit NativeTokenPulled(to, amount);
    }
```

#### Impact: Low

Withdrawing ether from the contract will revert if the destination `to` is a smart contract. Examples:
- Gnosis safes or other multisigs
- Protocol treasury/vault
- Smart contract wallets

Low impact because the deployer can either upgrade the contract or transfer the funds to an EOA and then transfer to the end receiver contract.

#### Recommendation

Wrap the `to` with the `payable` attribute to transfer the funds regardless if it is a contract or not:

```diff
    function pullNativeToken(address to, uint256 amount) public hasTokenSafePullRole checkPaused {
        _requireNonZeroAddress(to, "Recipient address cannot be null address");
        require(amount > 0, "Amount must be greater than 0");

        uint256 balance = address(this).balance;
        require(balance >= amount, "Not enough native tokens in contract");


        // "to" have been checked to be a non zero address
-       (bool success,) = to.call{value: amount}("");
+       (bool success,) = payable(to).call{value: amount}("");
        require(success, "Native token transfer failed");
        emit NativeTokenPulled(to, amount);
    }
```









### [L-3] The `wrap()` function will revert when users attempt to wrap exactly `minStakingAmt`

The `wrap()` function checks that the `wtaoAmount` is strictly greater than `minStakingAmount`, which means that the case `wtaoAmount == minStakingAmt` will revert.

```javascript
    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
        require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        // ...

        // Ensure that at least 0.125 TAO is being bridged
        // based on the smart contract
@>      require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");

        // ...
    }
``` 

#### Impact: Low

Users will get reverted transactions when they attempt to stake exactly `minStakingAmount`. Bad UX, but not the end of the world.

#### Recommendation

Change it to greater or equal to:

```diff
```javascript
    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
        require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        // ...

        // Ensure that at least 0.125 TAO is being bridged
        // based on the smart contract
-       require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");
+       require(wtaoAmount >= minStakingAmt, "Does not meet minimum staking amount");

        // ...
    }
```








### [L-4] The `wrap()` function requires that the caller's `wTAO` balance is higher than the amount to stake (`wtaoAmount`) but ignores the fees in the requirement

There is a requirement in `wrap()` such that the `msg.sender` has at least `wtaoAmount` tokens in its balance. However, later in the same function it attempts to retrieve a higher amount of `wTAO` from the `msg.sender`: the sum of `wtaoAmount + stakingFees + bridgingFee`. 

```javascript
    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
        require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        string memory _nativeWalletReceiver = nativeWalletReceiver;
        IERC20 _wrappedToken = IERC20(wrappedToken);
        // The two requirements below protect in case contract hasn't been initialized yet. A single validation would be less gas expensive. 
        // Check that the nativeWalletReceiver is not an empty string
        _checkValidFinneyWallet(_nativeWalletReceiver);
        _requireNonZeroAddress(address(_wrappedToken), "wrappedToken address is invalid");
@>      require(_wrappedToken.balanceOf(msg.sender) >= wtaoAmount, "Insufficient wTAO balance");

        // Check to ensure that the protocol vault address is not zero
        // OK protocolVault is used below in _transferTovault()
        _requireNonZeroAddress(address(protocolVault), "Protocol vault address cannot be 0");
        
        // Ensure that at least 0.125 TAO is being bridged
        // based on the smart contract
        require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");

        // Ensure that the wrap amount after free is more than 0. 
        //OK [wtaoAmount = wrappedTokenAmoutn + feeAmt] exactly
@>      (uint256 wrapAmountAfterFee, uint256 feeAmt) = calculateAmtAfterFee(wtaoAmount);  //OK feeAmt stakingFee[wTAO] (bridging fee not included). OK it is added afterwards

        uint256 yTAOAmount = getYTAObyWTAO(wrapAmountAfterFee);

        // Perform token transfers
        _mintWithSupplyCap(msg.sender, yTAOAmount);
        _transferToVault(feeAmt);  //OK transfered from msg.sender to vault. 
@>      uint256 amtToBridge = wrapAmountAfterFee + bridgingFee;
@>      _transferToContract(amtToBridge);
        // now amtToBridge of wTAO are in this contract

        //i this method presumably bridges amtToBridge wTAO on behalf of _nativeWalletReceiver
        bool success = wTAO(wrappedToken).bridgeBack(amtToBridge, _nativeWalletReceiver);
        require(success, "Bridge back failed");

        emit UserStake(msg.sender, block.timestamp, wtaoAmount, yTAOAmount);
    }
```

#### Impact: Low

No impact, as if the user didn't have enough `wTAO`, the contract would not be at risk of exploitation anyway. It is just an unnecessary redundant check (as the ERC20 transfer would check it anyways), and on top of that, the check is wrongly performed. 

#### Recommendation

The requirement can be removed because the `_transferToContract()` function will anyways revert if the `msg.sender` does not have enough `wTAO` in the balance (ERC20 will revert).

```diff
    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
        require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        string memory _nativeWalletReceiver = nativeWalletReceiver;
        IERC20 _wrappedToken = IERC20(wrappedToken);
        // The two requirements below protect in case contract hasn't been initialized yet. A single validation would be less gas expensive. 
        // Check that the nativeWalletReceiver is not an empty string
        _checkValidFinneyWallet(_nativeWalletReceiver);
        _requireNonZeroAddress(address(_wrappedToken), "wrappedToken address is invalid");
-       require(_wrappedToken.balanceOf(msg.sender) >= wtaoAmount, "Insufficient wTAO balance");

        // ...
    }
```










### [L-5] The function `approveMultipleUnstakes()` lacks protection against repeated unstake requests


In the function  `approveMultipleUnstakes()`, The requirement that a particular request has `isReadyForUnstake = false` is read in a different for-loop than the one that is setting `isReadyForUnstake = true`. This means that if the admins approve the same unstaking request multiple times in the same calldata array of pending requests, the contract will not revert on the repeated one. Instead, it will add twice the amount of wTAO required to be sent to the contract (`totalRequiredTaoAmt`): 

```javascript
    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // There might be cases that the underlying token might be different
        // so we need to add checks to ensure that the unstaking is the same token
        // across all indexes in the current requests UserRequest[] array
        // If there is 2 different tokens underlying in the requests, we return the
        // error as system is not designed to handle such a scenario.
        // In that scenario, the user needs to unstake the tokens separately
        // in two separate request
        address commonWrappedToken = unstakeRequests[requests[0].user][requests[0].requestIndex].wrappedToken;

        // Loop through each request to unstake and check if the request is valid
        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            require(request.requestIndex < unstakeRequests[request.user].length, "Invalid request index");
            require(unstakeRequests[request.user][request.requestIndex].amount > 0, "Request is invalid");
@>          require(!unstakeRequests[request.user][request.requestIndex].isReadyForUnstake, "Request is already approved");

            // Check if wrappedToken is the same for all requests
            require(
                unstakeRequests[request.user][request.requestIndex].wrappedToken == commonWrappedToken,
                "Wrapped token is not the same across all unstake requests"
            );
@>          totalRequiredTaoAmt += (unstakeRequests[request.user][request.requestIndex].taoAmt + request.rewardAmount);
        }

        // Check if the sender has allowed the contract to spend enough tokens
        require(
            IERC20(commonWrappedToken).allowance(msg.sender, address(this)) >= totalRequiredTaoAmt,
            "Insufficient token allowance"
        );

        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
@>          unstakeRequests[request.user][request.requestIndex].isReadyForUnstake = true;
            // storing here the rewardAmount for the user to retrieve with unstake()
            unstakeRequests[request.user][request.requestIndex].rewardAmount = request.rewardAmount;
        }
    }
```


#### Impact: Low

- Probability: low, as it requires the admins to make the mistake of sending repeated requests in the array
- Impact: low, as even though more tokens are sent to the contract, these can be recovered using `safePullERC20()`.

**Important Note:** In a previously described medium finding, it is stated that it would be better to track the funds that are allocated to pending unstakes. If this recommendation is implemented, then this issue is not trivial, as the tokens from double-approved requests would get locked in the contract, requiring a contract upgrade.

#### Recommendation

To solve the issue, get rid of the multiple loops in `approveMultipleUnstakes()` and join all the operations in a single loop. If a request is repeated, the second instance will revert as `isReadyForUnstake` was set to `true` in the first iteration. This same suggestion saves also a significant amount of gas, but that is described in the gas savings section.

```diff

The suggested implementation is:

```diff
    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // There might be cases that the underlying token might be different
        // so we need to add checks to ensure that the unstaking is the same token
        // across all indexes in the current requests UserRequest[] array
        // If there is 2 different tokens underlying in the requests, we return the
        // error as system is not designed to handle such a scenario.
        // In that scenario, the user needs to unstake the tokens separately
        // in two separate request
        address commonWrappedToken = unstakeRequests[requests[0].user][requests[0].requestIndex].wrappedToken;

        // Loop through each request to unstake and check if the request is valid
        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            require(request.requestIndex < unstakeRequests[request.user].length, "Invalid request index");
            require(unstakeRequests[request.user][request.requestIndex].amount > 0, "Request is invalid");
@>          require(!unstakeRequests[request.user][request.requestIndex].isReadyForUnstake, "Request is already approved");

            // Check if wrappedToken is the same for all requests
            require(
                unstakeRequests[request.user][request.requestIndex].wrappedToken == commonWrappedToken,
                "Wrapped token is not the same across all unstake requests"
            );

+           UserRequest calldata request = requests[i];
+           unstakeRequests[request.user][request.requestIndex].isReadyForUnstake = true;
+           // storing here the rewardAmount for the user to retrieve with unstake()
+           unstakeRequests[request.user][request.requestIndex].rewardAmount = request.rewardAmount;

+           emit AdminUnstakeApproved(request.user, request.requestIndex, block.timestamp);

            totalRequiredTaoAmt += (unstakeRequests[request.user][request.requestIndex].taoAmt + request.rewardAmount);
        }

        // Check if the sender has allowed the contract to spend enough tokens
        require(
            IERC20(commonWrappedToken).allowance(msg.sender, address(this)) >= totalRequiredTaoAmt,
            "Insufficient token allowance"
        );

-        for (uint256 i = 0; i < requests.length; i++) {
-           UserRequest calldata request = requests[i];
-           unstakeRequests[request.user][request.requestIndex].isReadyForUnstake = true;
-           // storing here the rewardAmount for the user to retrieve with unstake()
-           unstakeRequests[request.user][request.requestIndex].rewardAmount = request.rewardAmount;
-        }

        // Transfer the tao from the withdrawal manager to this contract
        require(
            IERC20(commonWrappedToken).transferFrom(msg.sender, address(this), totalRequiredTaoAmt),
            "taoAmt transfer failed"
        );

-       // Emit events after state changes and external interactions
-       for (uint256 i = 0; i < requests.length; i++) {
-           UserRequest calldata request = requests[i];
-           emit AdminUnstakeApproved(request.user, request.requestIndex, block.timestamp);
-       }
    }
```








### [L-6] The parameter `minStakingAmount` cannot be ever set to 0, due to a wrong requirement in `setMinStakingAmount()` 

The docstring of `setMinStakingAmt()` states that `minStakingAmt` can be even 0. However, this is not true when the value is attempted to be set using the function `setMinStakingAmt`. 
The reason is that the requirement uses a strictly greater than `bridgingFee`, and this means that even if the `bridgingFee == 0`, the `minStakingAmt` can't be 0, but has to be at least 1. 

```javascript
    /*
    *
    * This method sets the min staking amount required to perform staking
    *
    * Value Boundary: minStakingAmt can be any value even 0 in the event that minStakingAmt by taobridge changes in the future
    *
    */
    function setMinStakingAmt(uint256 _minStakingAmt) public hasManageStakingConfigRole {
@>      require(_minStakingAmt > bridgingFee, "Min staking amount must be more than bridging fee");
        minStakingAmt = _minStakingAmt; // UNITS is tao[wei]
        emit UpdateMinStakingAmt(minStakingAmt);
    }
```

Note that `minStakingAmt` *can* be zero if it is never "set" because the default value is 0. However, once set it will never be 0 again. 

HOWEVER, I must say that `minStakingAmt` should be higher than zero, because this can protect against other issues. 

#### Impact: Low

Low impact, as being forced to have `minStakingAmt > 0` is not such a big deal. 

This issue is only reported because it is inconsistent with the intentions of the developer as per documentation. 

#### Recommendation

One of the following:
- Update documentation so that `minStakingAmt` has to be greater than 0 (safer)
- Update the requirement to follow the existing documentation with a greater-or-equal:
```diff
    function setMinStakingAmt(uint256 _minStakingAmt) public hasManageStakingConfigRole {
-       require(_minStakingAmt > bridgingFee, "Min staking amount must be more than bridging fee");
+       require(_minStakingAmt >= bridgingFee, "Min staking amount must be more than bridging fee");
        minStakingAmt = _minStakingAmt; // UNITS is tao[wei]
        emit UpdateMinStakingAmt(minStakingAmt);
    }
```



### [L-7] The `maxDepositPerRequest` is not taken into account when calculating `maxTaoForWrap()` which will create reverts if users try to wrap the max amount

The `wrap()` function imposes a hard check so that the staked `wtaoAmount` is lower than the parameter `maxDepositPerRequest`:

```javascript
    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
@>      require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        string memory _nativeWalletReceiver = nativeWalletReceiver;
        IERC20 _wrappedToken = IERC20(wrappedToken);

        // ...
```

The function `maxTaoForWrap()` intends to return what is the max `wTAO` that a user can stake in one transaction:

```javascript
    function maxTaoForWrap() public view returns (uint256) {
        uint256 availableYTAOSupply = maxSupply - totalSupply();
        if (availableYTAOSupply == 0) {
            return 0;
        }

        uint256 equivalentTaoAmount = (availableYTAOSupply * 1 ether) / exchangeRate;
        uint256 grossTaoAmount;

        // Adjust for the staking fee and bridging fee
        if (stakingFee > 0) {
            grossTaoAmount = (equivalentTaoAmount * 1000) / (1000 - stakingFee);
        } else {
            grossTaoAmount = equivalentTaoAmount;
        }
        uint256 maxTaoToWrap = grossTaoAmount + bridgingFee;
        return maxTaoToWrap;
    }
```

However, the `maxTaoForWrap()` does not take into account if the output value `maxTaoToWrap` is higher than the storage variable `maxDepositPerRequest`. 

#### Impact: Low

Every time a user tries to wrap an amount told by `maxTaoForWrap()` that is higher than `maxDepositPerRequest` will get reverted transactions, leading to a bad UX.

- Probability: high
- Impact: low, as users will just need to stake in smaller batches. It is still a bad UX experience that the contract tells you "you can stake up to X", but reverts when you try

#### Recommendation

The `maxTaoForWrap()` function should take this cap into account:

```diff
    function maxTaoForWrap() public view returns (uint256) {
        uint256 availableYTAOSupply = maxSupply - totalSupply();
        if (availableYTAOSupply == 0) {
            return 0;
        }

        uint256 equivalentTaoAmount = (availableYTAOSupply * 1 ether) / exchangeRate;
        uint256 grossTaoAmount;

        // Adjust for the staking fee and bridging fee
        if (stakingFee > 0) {
            grossTaoAmount = (equivalentTaoAmount * 1000) / (1000 - stakingFee);
        } else {
            grossTaoAmount = equivalentTaoAmount;
        }
        uint256 maxTaoToWrap = grossTaoAmount + bridgingFee;

+       if (maxDepositPerRequest < maxTaoForWrap) return maxDepositPerRequest;
        return maxTaoToWrap;
    }
```



### [L-8] In future contract updates, maxSupply needs to be checked against initial supply 

The initialize function sets the `maxSupply` to the input argument `initialSupply`. 

```javascript
    function initialize(address initialOwner, uint256 initialSupply) public initializer {
        require(initialOwner != address(0), "Owner cannot be null");
        require(initialSupply > 0, "Initial supply must be more than 0");

        // ...

@>      maxSupply = initialSupply;
    }
```

There are no issues in the first deployment of the contract. However, for future versions, when `yTAO` is non-zero, a requirement must be set so that the new `maxSupply` is higher than the current existing supply:

#### Impact: Low

If this same contract was used as a template to deploy a new version, and the max supply argument was set to a lower value than the existing supply, could lead to a halt of the contract as users would get reverted transactions when attempting to stake. Even though this impact is quite serious, I'm classifying it as low, because it requires a certain level of incompetence from the deployer, which I'm assuming is competent. 

#### Recommendation

```diff
```javascript
    function initialize(address initialOwner, uint256 initialSupply) public initializer {
        require(initialOwner != address(0), "Owner cannot be null");
        require(initialSupply > 0, "Initial supply must be more than 0");

        // ...

        maxSupply = initialSupply;
+       require(maxSupply >= totalSupply(), "Cannot set max supply below current supply");  // better check it in the input argument to save gas, instead of reading from storage
    } 
```








### [L-9] The ussage of Ownable & Access control is redundant as they are both serving the same purpose.

The contract inherits both `Ownable2Step` and `AccessControl` which is redundant, as both are meant for access control. 
Only the actual access control modifiers are used, the ownable could be removed.

Also, calling `_setRoleAdmin()` is unnecessary. 

```javascript
    contract YTAO is
        Initializable,
        ERC20Upgradeable, 
@>      Ownable2StepUpgradeable,
        ReentrancyGuardUpgradeable,
@>      AccessControlUpgradeable 
    {

    function initialize(address initialOwner, uint256 initialSupply) public initializer {
        require(initialOwner != address(0), "Owner cannot be null");
        require(initialSupply > 0, "Initial supply must be more than 0");
        __ERC20_init("TaiNET TAO", "yTAO");
@>      __Ownable_init(initialOwner);  // OwnableUpgradeable (not 2step)
@>      __AccessControl_init(); // AccessControlUpgradeable
        __ReentrancyGuard_init(); // ReentrancyGuardUpgradeable
        _setRoleAdmin(DEFAULT_ADMIN_ROLE, DEFAULT_ADMIN_ROLE); 
@>      _transferOwnership(initialOwner);
@>      _grantRole(DEFAULT_ADMIN_ROLE, initialOwner);
        maxSupply = initialSupply;
    }
```

#### Impact: low

This architecture is prone to mistakes while upgrading, exposing critical admin functions.

#### Recommendation

Remove the ownable related functions in the initialization and `renounceOwnership()`. Instead, treat the `DEFAULT_ADMIN_ROLE` as the admin. 

```diff
    contract YTAO is
        Initializable,
        ERC20Upgradeable, 
-       Ownable2StepUpgradeable,
        ReentrancyGuardUpgradeable,
        AccessControlUpgradeable 
    {


    function initialize(address initialOwner, uint256 initialSupply) public initializer {
        require(initialOwner != address(0), "Owner cannot be null");
        require(initialSupply > 0, "Initial supply must be more than 0");
        __ERC20_init("TaiNET TAO", "yTAO");
-       __Ownable_init(initialOwner);  // OwnableUpgradeable (not 2step)
@>      __AccessControl_init(); // AccessControlUpgradeable
        __ReentrancyGuard_init(); // ReentrancyGuardUpgradeable
-       _setRoleAdmin(DEFAULT_ADMIN_ROLE, DEFAULT_ADMIN_ROLE); 
-       _transferOwnership(initialOwner);
@>      _grantRole(DEFAULT_ADMIN_ROLE, initialOwner);
        maxSupply = initialSupply;
    }

-   function renounceOwnership() public override onlyOwner {}

```







### [L-10] Use SafeERC20 library for ERC20 transfers

The SafeERC20 library is imported but never used. Unsafe ERC20 transfers found in multiple token transfers. 

#### Recommendation

- Use `token.safeTransfer()` instead of `token.transfer()`
- Use `token.safeTransferFrom()` instead of `token.transferFrom()`

Besides the extra security it provides, it already does all the necessary safety checks and requirements, reducing code. Taje the `_trasferToVault()` function as an example:

```diff
    function _transferToVault(uint256 _feeAmt) internal {
-       require(
-           IERC20(wrappedToken).transferFrom(msg.sender, address(protocolVault), _feeAmt),
-           "Transfer to protocol vault address failed"
-       );
+       IERC20(wrappedToken).safeTransferFrom(msg.sender, address(protocolVault), _feeAmt),

    }
```





---------------------





## Gas optimizations




### [G-1] Repeating the same check over and over for every request unstake

In `requestUnstake()`, there are two requirements to ensure that `wrappedToken` and `withdrawalManager` are not `address(0)`. 
These checks should be only on the setter functions so that they are only checked **once** instead of being checked over and over for every new `requestUnstake()`. As a matter of fact, the checks are already in place in the setters, so there is absolutely no need to place them here. 

The problem is made worse by the fact that error strings are passed as memory to the `_requireNonZeroAddress()` which is even more gas-costly. In-memory strings for errors are an antipattern, and custom errors are recommended.

```javascript
    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {
        // Check that wrappedToken and withdrawalManager is a valid address
@>      _requireNonZeroAddress(address(wrappedToken), "wrappedToken address is invalid");
@>      _requireNonZeroAddress(address(withdrawalManager), "withdrawal address cannot be null");

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");
        
        // ...

    }
``` 

The same thing applies to the `_checkValidFinneyWallet()` check, which is done in both the setter, and everytime a user calls `wrap()`. It is very expensive to repeat these checks

```javascript
    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
        require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        string memory _nativeWalletReceiver = nativeWalletReceiver;
        IERC20 _wrappedToken = IERC20(wrappedToken);
        // Check that the nativeWalletReceiver is not an empty string
@>      _checkValidFinneyWallet(_nativeWalletReceiver);
@>      _requireNonZeroAddress(address(_wrappedToken), "wrappedToken address is invalid");
        require(_wrappedToken.balanceOf(msg.sender) >= wtaoAmount, "Insufficient wTAO balance");

        // Check to ensure that the protocol vault address is not zero
@>      _requireNonZeroAddress(address(protocolVault), "Protocol vault address cannot be 0");
        
        // Ensure that at least 0.125 TAO is being bridged
        // based on the smart contract
        require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");

        // .. 
    }
```

The only reason for doing these checks could be to avoid calling this function *before* the addresses were set. However, there are other checks in place that would make the function revert, so I reiterate, that these checks are redundant and **very** expensive for the user.

#### Recommendation

Use custom errors instead of strings:

```diff

+   error InvalidAddress(address addr);

    function setWTAO(address _wTAO) public hasManageStakingConfigRole {
        // Check to ensure _wTAO is not null
-       _requireNonZeroAddress(_wTAO, "wTAO address cannot be null");
+       if (_wTAO == address(0)) revert InvalidAddress(_wTAO)
        wrappedToken = _wTAO;
        emit UpdateWTao(_wTAO);
    }

    function setWithdrawalManager(address _withdrawalManager) public hasManageStakingConfigRole {
-       require(_withdrawalManager != address(0), "Withdrawal manager cannot be null");
+       if (_withdrawalManager == address(0)) revert InvalidAddress(_withdrawalManager)
        withdrawalManager = _withdrawalManager;
        emit UpdateWithdrawalManager(withdrawalManager);
    }

    function setProtocolVault(address _protocolVault) public hasManageStakingConfigRole {
-       require(_protocolVault != address(0), "Protocol vault cannot be null");
+       if (_protocolVault == address(0)) revert InvalidAddress(_protocolVault)
        protocolVault = _protocolVault;
        emit UpdateProtocolVault(protocolVault);
    }
```

And remove unnecessary checks to save gas in the following places:

```diff
    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {
-       // Check that wrappedToken and withdrawalManager is a valid address
-       _requireNonZeroAddress(address(wrappedToken), "wrappedToken address is invalid");
-       _requireNonZeroAddress(address(withdrawalManager), "withdrawal address cannot be null");

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");
        
        // ...
    }


    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // Deposit cap amount
        require(maxDepositPerRequest >= wtaoAmount, "Deposit amount exceeds maximum");

        string memory _nativeWalletReceiver = nativeWalletReceiver;
        IERC20 _wrappedToken = IERC20(wrappedToken);
        // Check that the nativeWalletReceiver is not an empty string
-       _checkValidFinneyWallet(_nativeWalletReceiver);
-       _requireNonZeroAddress(address(_wrappedToken), "wrappedToken address is invalid");
        require(_wrappedToken.balanceOf(msg.sender) >= wtaoAmount, "Insufficient wTAO balance");

        // Check to ensure that the protocol vault address is not zero
-       _requireNonZeroAddress(address(protocolVault), "Protocol vault address cannot be 0");
        
        // Ensure that at least 0.125 TAO is being bridged
        // based on the smart contract
        require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");

        // .. 
    }
```







### [G-2] Checking for the `wrappedToken` in every `UnstakeRequest` is expensive and unnecessary if the `wTAO` token is not expected to change

The `wTAO` is not an upgradeable contract, so it is safe to assume that it won't change. 
The `approveMultipleUnstakes()` function checks that the `wrappedToken` (`wTAO` address) is the same in all requests. It is a reasonable check, but expensive if done for every request in history, especially if not expected to change.

#### Recommendation

Remove the check, and safely assume that all requests have the same `wrappedToken`. 
```diff
    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // There might be cases that the underlying token might be different
        // so we need to add checks to ensure that the unstaking is the same token
        // across all indexes in the current requests UserRequest[] array
        // If there is 2 different tokens underlying in the requests, we return the
        // error as system is not designed to handle such a scenario.
        // In that scenario, the user needs to unstake the tokens separately
        // in two separate request
-       address commonWrappedToken = unstakeRequests[requests[0].user][requests[0].requestIndex].wrappedToken;

        // Loop through each request to unstake and check if the request is valid
        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            require(request.requestIndex < unstakeRequests[request.user].length, "Invalid request index");
            require(unstakeRequests[request.user][request.requestIndex].amount > 0, "Request is invalid");
            require(!unstakeRequests[request.user][request.requestIndex].isReadyForUnstake, "Request is already approved");

            // Check if wrappedToken is the same for all requests
-           require(
-               unstakeRequests[request.user][request.requestIndex].wrappedToken == commonWrappedToken,
-               "Wrapped token is not the same across all unstake requests"
-           );

            // ...

        // ..
    }
```

Moreover, it could also be removed from the struct itself, and removed it everywhere else where relevant:

```diff
    struct UnstakeRequest {
        uint256 amount; // Amount of yTAO being unstaked
        uint256 taoAmt; // Amount of wTAO equivalent to be returned
        bool isReadyForUnstake; // Whether the unstake has been approved
-       address wrappedToken; // Address of the wrapped token
        uint256 timestamp; // Timestamp when the unstake request was made
        uint256 rewardAmount; // Reward in TAO to be given to the user  // UNITS wTAO
    }

```





### [G-3] The same array is iterated in three separated for-loops in the same function instead of performing all operations in the same loop

Loops are expensive in solidity and should be avoided as much as possible. 
In `approveMultipleUnstakes()` the array of `requests` is iterated **three** times instead of iterating once performing all necessary operations. 

In this particular case, it is possible to merge all the logic from the three loops into the same one because there are no untrusted external calls, so no risk of reentrancy. 

#### Recommendation

The original version is:

```javascript

    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // There might be cases that the underlying token might be different
        // so we need to add checks to ensure that the unstaking is the same token
        // across all indexes in the current requests UserRequest[] array
        // If there is 2 different tokens underlying in the requests, we return the
        // error as system is not designed to handle such a scenario.
        // In that scenario, the user needs to unstake the tokens separately
        // in two separate request
        address commonWrappedToken = unstakeRequests[requests[0].user][requests[0].requestIndex].wrappedToken;

        // Loop through each request to unstake and check if the request is valid
        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            require(request.requestIndex < unstakeRequests[request.user].length, "Invalid request index");
            require(unstakeRequests[request.user][request.requestIndex].amount > 0, "Request is invalid");
            require(!unstakeRequests[request.user][request.requestIndex].isReadyForUnstake, "Request is already approved");

            // Check if wrappedToken is the same for all requests
            require(
                unstakeRequests[request.user][request.requestIndex].wrappedToken == commonWrappedToken,
                "Wrapped token is not the same across all unstake requests"
            );
            totalRequiredTaoAmt += (unstakeRequests[request.user][request.requestIndex].taoAmt + request.rewardAmount);
        }

        // Check if the sender has allowed the contract to spend enough tokens
        require(
            IERC20(commonWrappedToken).allowance(msg.sender, address(this)) >= totalRequiredTaoAmt,
            "Insufficient token allowance"
        );

        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            unstakeRequests[request.user][request.requestIndex].isReadyForUnstake = true;
            // storing here the rewardAmount for the user to retrieve with unstake()
            unstakeRequests[request.user][request.requestIndex].rewardAmount = request.rewardAmount;
        }

        // Transfer the tao from the withdrawal manager to this contract
        require(
            IERC20(commonWrappedToken).transferFrom(msg.sender, address(this), totalRequiredTaoAmt),
            "taoAmt transfer failed"
        );

        // Emit events after state changes and external interactions
        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            emit AdminUnstakeApproved(request.user, request.requestIndex, block.timestamp);
        }
    }

```

The suggested implementation is:

```diff
    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // There might be cases that the underlying token might be different
        // so we need to add checks to ensure that the unstaking is the same token
        // across all indexes in the current requests UserRequest[] array
        // If there is 2 different tokens underlying in the requests, we return the
        // error as system is not designed to handle such a scenario.
        // In that scenario, the user needs to unstake the tokens separately
        // in two separate request
        address commonWrappedToken = unstakeRequests[requests[0].user][requests[0].requestIndex].wrappedToken;

        // Loop through each request to unstake and check if the request is valid
        for (uint256 i = 0; i < requests.length; i++) {
            UserRequest calldata request = requests[i];
            require(request.requestIndex < unstakeRequests[request.user].length, "Invalid request index");
            require(unstakeRequests[request.user][request.requestIndex].amount > 0, "Request is invalid");
            require(!unstakeRequests[request.user][request.requestIndex].isReadyForUnstake, "Request is already approved");

            // Check if wrappedToken is the same for all requests
            require(
                unstakeRequests[request.user][request.requestIndex].wrappedToken == commonWrappedToken,
                "Wrapped token is not the same across all unstake requests"
            );

+           UserRequest calldata request = requests[i];
+           unstakeRequests[request.user][request.requestIndex].isReadyForUnstake = true;
+           // storing here the rewardAmount for the user to retrieve with unstake()
+           unstakeRequests[request.user][request.requestIndex].rewardAmount = request.rewardAmount;

+           emit AdminUnstakeApproved(request.user, request.requestIndex, block.timestamp);

            totalRequiredTaoAmt += (unstakeRequests[request.user][request.requestIndex].taoAmt + request.rewardAmount);
        }

        // Check if the sender has allowed the contract to spend enough tokens
        require(
            IERC20(commonWrappedToken).allowance(msg.sender, address(this)) >= totalRequiredTaoAmt,
            "Insufficient token allowance"
        );

-        for (uint256 i = 0; i < requests.length; i++) {
-           UserRequest calldata request = requests[i];
-           unstakeRequests[request.user][request.requestIndex].isReadyForUnstake = true;
-           // storing here the rewardAmount for the user to retrieve with unstake()
-           unstakeRequests[request.user][request.requestIndex].rewardAmount = request.rewardAmount;
-        }

        // Transfer the tao from the withdrawal manager to this contract
        require(
            IERC20(commonWrappedToken).transferFrom(msg.sender, address(this), totalRequiredTaoAmt),
            "taoAmt transfer failed"
        );

-       // Emit events after state changes and external interactions
-       for (uint256 i = 0; i < requests.length; i++) {
-           UserRequest calldata request = requests[i];
-           emit AdminUnstakeApproved(request.user, request.requestIndex, block.timestamp);
-       }
    }
```





### [G-4] Unnecessary repeated check every time the exchange rate is updated

This requirement is already enforced in the setter of `lowerExchangeRateBound` and `upperExchangeRateBound`. Repeating the check every time that the exchange rate is updated is very gas-costly for the protocol in the long term.

```diff
    function updateExchangeRate(uint256 newRate) public hasExchangeUpdateRole {
        require(newRate > 0, "New rate must be more than 0");
        require(
            newRate >= lowerExchangeRateBound && newRate <= upperExchangeRateBound, "New rate must be within bounds"
        );
-       require(lowerExchangeRateBound > 0 && upperExchangeRateBound > 0, "Bounds must be more than 0");

        exchangeRate = newRate;
        emit UpdateExchangeRate(newRate);
    }
```







### [G-5] Unnecessary allowance check in `approveMultipleUnstakes()`

This function checks the allowance every time some requests need to be approved. This is unnecessary as the later `transferFrom()` already performs the checks internally. Remove the allowance requirement and save gas.

```diff

    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
        hasApproveWithdrawalRole
        nonReentrant
        checkPaused
    {
        uint256 totalRequiredTaoAmt = 0;
        require(requests.length > 0, "Requests array is empty");
        require(requests[0].requestIndex < unstakeRequests[requests[0].user].length, "First request index out of bounds");

        // ...

-       // Check if the sender has allowed the contract to spend enough tokens
-       require(
-           IERC20(commonWrappedToken).allowance(msg.sender, address(this)) >= totalRequiredTaoAmt,
-           "Insufficient token allowance"
-       );
        
        // ...

        require(
            IERC20(commonWrappedToken).transferFrom(msg.sender, address(this), totalRequiredTaoAmt),
            "taoAmt transfer failed"
        );
```



### [G-6] Read length from cached memory in `requestUnstake` as it has already been read before

```diff
    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {
        // Check that wrappedToken and withdrawalManager is a valid address
        _requireNonZeroAddress(address(wrappedToken), "wrappedToken address is invalid");
        _requireNonZeroAddress(address(withdrawalManager), "withdrawal address cannot be null"); 

        // Ensure that the fee amount is sufficient
        require(msg.value >= serviceFee, "Fee amount is not sufficient");
        
        require(yTAOAmt > unstakingFee, "Invalid yTAO amount");
        // Check if enough balance
        require(balanceOf(msg.sender) >= yTAOAmt, "Insufficient yTAO balance");
        uint256 outWTaoAmt = getWTAOByYTAOAfterFee(yTAOAmt);

        uint256 length = unstakeRequests[msg.sender].length;
        bool added = false;
        // Loop throught the list of existing unstake requests
        for (uint256 i = 0; i < length; i++) {
            uint256 currAmt = unstakeRequests[msg.sender][i].amount;
            if (currAmt > 0) {
                continue;
            } else {
                // If the curr amt is zero, it means
                // we can add the unstake request in this index
                unstakeRequests[msg.sender][i] = UnstakeRequest({
                    amount: yTAOAmt,  // ytao burned at the moment of unstaking
                    taoAmt: outWTaoAmt,  // wtao to be returned (calculated from the exchange rate minus the unstaking fee)
                    isReadyForUnstake: false,
                    timestamp: block.timestamp,
                    wrappedToken: wrappedToken,
                    rewardAmount: 0 // when unstake is requested, the rewardAmount is 0, and it is the withdrawalManager that sets it when approving the withdraw
                });
                added = true;
                emit UserUnstakeRequested(msg.sender, i, block.timestamp, yTAOAmt, outWTaoAmt, wrappedToken);
                break;
            }
        }

        // If we have not added the unstake request, it means that
        // we need to push a new unstake request into the array
        if (!added) {
-           require(unstakeRequests[msg.sender].length < maxUnstakeRequests, "Maximum unstake requests exceeded");
+           require(length < maxUnstakeRequests, "Maximum unstake requests exceeded");
            unstakeRequests[msg.sender].push(
                UnstakeRequest({
                    amount: yTAOAmt,
                    taoAmt: outWTaoAmt,
                    isReadyForUnstake: false,
                    timestamp: block.timestamp,
                    wrappedToken: wrappedToken,
                    rewardAmount: 0 
                })
            );
            emit UserUnstakeRequested(msg.sender, length, block.timestamp, yTAOAmt, outWTaoAmt, wrappedToken);
        }
```




### [G-7] Save gas by storing bytes instead of strings for `nativeWalletReceiver`

Storing strings is an antipattern, except if they are constants (string literals). Storing bytes is recommended instead, and should only be converted to strings in the output of view functions if really required. 

#### Recommendation

Replace bytes in every instance of `_nativeWalletReceiver`, except in `bridgeBack()`, which incorrectly uses `string` as well. These are only some examples of places where it needs to be changed, but not

```diff
    /*
    * finney wallet that will receive the wTAO after wrap()
    * NOTE: it must be at least 48 chars so validation shall
    * be done to ensure that it is true
    */
    // @audit-info storing strings is more expensive than storing bytes. Consider changing to bytes and operate always on bytes. Only convert to string in public view functions when necessary
-   string public nativeWalletReceiver;
+   bytes public nativeWalletReceiver;


-   function _checkValidFinneyWallet(string memory _nativeWalletReceiver) internal pure {
+   function _checkValidFinneyWallet(bytes memory _nativeWalletReceiver) internal pure {
        // ok 1 byte == 1 character
-       require(bytes(_nativeWalletReceiver).length == 48, "nativeWalletReceiver must be of length 48");
+       require(_nativeWalletReceiver.length == 48, "nativeWalletReceiver must be of length 48");
    }

    // ...


    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // ...

-       bool success = wTAO(wrappedToken).bridgeBack(amtToBridge, _nativeWalletReceiver);
+       bool success = wTAO(wrappedToken).bridgeBack(amtToBridge, string(_nativeWalletReceiver));
```


--------------------------


## Informational

### [I-1] The `checkPaused` modifier is used in `approveMultipleUnstakes()`

A valid scenario is that the owner needs to pause the contract to process unstakes. This is not possible, however, as the `checkPaused` modifier also applies to the `approveMultipleUnstakes()` function. 
so while paused, requests cannot be processed. 

#### Recommendation

I would remove the `checkPaused` from the admin functions unless there is a strong reason for keeping it. 


### [I-2] The nonReentrant modifier should be placed first for security

The order of the modifiers in the function definition defines the order in which they are verified. For safety, it is recommended to place the nonReentrant modifier the first one (if really needed).

```diff
    function approveMultipleUnstakes(UserRequest[] calldata requests)
        public
+       nonReentrant
        hasApproveWithdrawalRole
-       nonReentrant
        checkPaused
    {
```


### [I-3] Wrong argument name in events 

The event `UserUnstakeRequested()` has to amounts named as `tao`, but one of them refers as `yTAO`:

```diff
    event UserUnstakeRequested(
        address indexed user,
        uint256 idx,  // @audit this index is a bit irrelevant as the indexes are being reused. Reusing the indexes is an anti-pattern
        uint256 requestTimestamp,
-       uint256 taoAmount,  // @audit-issue LOW this should be named yTaoAmount instead
+       uint256 yTaoAmount,
        uint256 outTaoAmt,
        address wrappedToken
    );

```

Same issue with the `UserStake()` event:

```diff
-   event UserStake(address indexed user, uint256 stakeTimestamp, uint256 inTaoAmt, uint256 taoAmount);
+   event UserStake(address indexed user, uint256 stakeTimestamp, uint256 inTaoAmt, uint256 yTAOAmount);
```

As an example, these are the values when emitting the event:

```javascript
    function requestUnstake(uint256 yTAOAmt) public payable nonReentrant checkPaused {
        // ..

        for (uint256 i = 0; i < length; i++) {
            uint256 currAmt = unstakeRequests[msg.sender][i].amount;
            if (currAmt > 0) {
                continue;
            } else {

                // ...

                added = true;
@>              emit UserUnstakeRequested(msg.sender, i, block.timestamp, yTAOAmt, outWTaoAmt, wrappedToken);  // 4th item is yTAOAmt, but event name is "taoAmoun"
                break;
            }
        }


    function wrap(uint256 wtaoAmount) public nonReentrant checkPaused {
        // ...

@>      emit UserStake(msg.sender, block.timestamp, wtaoAmount, yTAOAmount);
    }

```

### [I-4] Naming inconsistencies

#### Weird supplies variable naming in `initialize()`

The `initialize()` function takes an argument called `initialSupply`, but it is set to `maxSupply`. Moreover, this amount is not minted, and therefore the initial supply is 0, which means it is weird to call that input variable `initialSupply`.
I suggest to call it `initialMaxSupply` instead.

#### The tokens TAO and wTAO are used interchangeably

Some variable naming is inconsistent, which difficults the overall readability. Here are some examples:

```javascript
    struct UnstakeRequest {
        uint256 amount;
        uint256 taoAmt; // @audit This should be called wTaoAmount, as TAO is not accepted. 
        bool isReadyForUnstake; // Whether the unstake has been approved
        // ...
```

#### Stake/wrap/deposit/bridge are used interchangeably

I suggest using deposit instead of wrap, as wTAO is already a wrapped token, and it is not being wrapped. It is being deposited in exchange for yTAO tokens.

#### Recommendation

Be consistent with naming. 



### [I-5] Unnecessary initializations in `initialize()`

The contract `YTAO` inherits `Ownable2StepUpgradeable`, which is a safer version of OwnableUpgradeable. It is safer because for transfering ownership, the destination needs to accept the transfer. This can save the day if the wrong address is passed when transferring ownership, as the new address will most likely not be accepted as it is not aware. 

```javascript
contract YTAO is
    Initializable,
    ERC20Upgradeable, 
@>  Ownable2StepUpgradeable,
    ReentrancyGuardUpgradeable,
@>  AccessControlUpgradeable 
{

    // ...

    function initialize(address initialOwner, uint256 initialSupply) public initializer {
        require(initialOwner != address(0), "Owner cannot be null");
        require(initialSupply > 0, "Initial supply must be more than 0");
        __ERC20_init("TaiNET TAO", "yTAO");
@>      __Ownable_init(initialOwner);
        __AccessControl_init();
        __ReentrancyGuard_init(); // ReentrancyGuardUpgradeable
@>      _setRoleAdmin(DEFAULT_ADMIN_ROLE, DEFAULT_ADMIN_ROLE);
@>      _transferOwnership(initialOwner);
        _grantRole(DEFAULT_ADMIN_ROLE, initialOwner);
        maxSupply = initialSupply;  // @audit why not calling the calldata variable maxSupply directly?
    }
```

In the initialize function, first the `__Ownable_init()` is called with the `initialOwner`, but then the internal function `_transferOwnership` is called again, which is unnecessary, because the owner has already be set. 
Moreover, the action `_setRoleAdmin(DEFAULT_ADMIN_ROLE, DEFAULT_ADMIN_ROLE)` is unnecessary as the `DEFAULT_ADMIN_ROLE = 0x0`, and it is already set as the role admin by default. So it can be removed. 

#### Recommendations

```diff

    function initialize(address initialOwner, uint256 initialSupply) public initializer {
        require(initialOwner != address(0), "Owner cannot be null");
        require(initialSupply > 0, "Initial supply must be more than 0");
        __ERC20_init("TaiNET TAO", "yTAO");
        __Ownable_init(initialOwner);
        __AccessControl_init();
        __ReentrancyGuard_init(); // ReentrancyGuardUpgradeable
-       _setRoleAdmin(DEFAULT_ADMIN_ROLE, DEFAULT_ADMIN_ROLE);
-       _transferOwnership(initialOwner);
        _grantRole(DEFAULT_ADMIN_ROLE, initialOwner);
        maxSupply = initialSupply;  // @audit why not call the calldata variable maxSupply directly?
    }
```

### [I-6] Redundant check in `setUpperExchangeRateBound()`

by checking `newUpper > lower`, the first check (greater than 0) is redundant.

```diff
    function setUpperExchangeRateBound(uint256 _newUpperBound) public hasExchangeUpdateRole {
-       require(_newUpperBound > 0, "New upper bound must be more than 0");
        require(_newUpperBound > lowerExchangeRateBound, "New upper bound must be greater than current lower bound");
        upperExchangeRateBound = _newUpperBound;
        emit UpperBoundUpdated(_newUpperBound);
    }
```

### [I-7] Avoid "magic" numbers and define constants instead (embedded in bytecode so they cost no gas)

```diff
    
+   // constant used as divisor for all percentage operations
+   uint256 internal constant DIVISOR_BASE = 1000;

    function getWTAOByYTAOAfterFee(uint256 yTaoAmount) public view returns (uint256) {
-       uint256 feeAmount = (yTaoAmount * unstakingFee) / 1000;
+       uint256 feeAmount = (yTaoAmount * unstakingFee) / DIVISOR_BASE;

        return ((yTaoAmount - feeAmount) * 1 ether) / exchangeRate;
    }
```






