# Vector Reserve security review

A time-boxed security review of the [**Vector Reserve**](https://twitter.com/vectorreserve) protocol, with a focus on
smart contract security.

Author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits?tab=readme-ov-file).

## Findings Overview

| Finding  | Description                                                                                                                                                                      | Severity                 | Status       | 
|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------|--------------| 
| \[H-1\]  | An attacker can inflate the share's price of `StakedVectorETH`, and steal all the vETH tokens from other stakers                                                                 | High                     | Fixed        |
| \[H-2\]  | A malicious address can postpone indefinitely the redemption of a victim's bond, making the funds inaccessible to the victim                                                     | High                     | Fixed        | 
| \[M-1\]  | Unaware users will postpone their own redemption date if they make a new deposit before redeeming an existing bond                                                               | Medium                   | Acknowledged | 
| \[M-2\]  | When the owner of `VectorBonding` sets new Vesting terms calling `setBondTerms()`, malicious addresses can force existing bonds of other users to adopt the new vesting terms    | Medium                   | Fixed        |
| \[M-3\]  | VEC transfers can revert when the `VEC` ether balance is 0 at the end of `swapBack()` function, because `vETH.deposit()` reverts with `amount=0`                                 | Medium                   | Fixed        |
| \[M-4\]  | No slippage protection in `VectorBond.deposit()` for BondType `WETHTOLP` will give WETH depositors a less favorable deal when WETH is swapped for VEC                            | Medium                   | Acknowledged |
| \[M-5\]  | Flash arbitrageurs can leverage small deviations between the LST pegs and the exchange rate given by `vETHPerRestakedLST` and slowly drain the protocol reserves                 | Medium                   | Acknowledged |
| \[M-6\]  | The owner of `VectorETH` can withdraw all LST tokens from all users                                                                                                              | Medium  (centralization) | Acknowledged |
| \[M-7\]  | The owner of `VectorETH` can halt redemptions of any (or all) LST tokens                                                                                                         | Medium  (centralization) | Acknowledged |
| \[L-1\]  | The stepwise jump caused by updating the `vETHPerLST` in `VectorETH` can be sandwiched, providing a free-risk trade to frontrunners, without providing any value to the protocol | Low                      | Acknowledged |
| \[L-2\]  | Use OpenZeppelin's SafeERC20 library wherever the ERC20 tokens are not known in advance                                                                                          | Low                      | Fixed        |
| \[L-3\]  | `VectorETH.deposit()` and `VectorETH.redeem()` assume that all future LSTs will have 18 decimals                                                                                 | Low                      | Fixed        |
| \[L-4\]  | The event `SwapAndLiquify()` emitted by `Vector` will always log `tokensForLiquidity=0` as it is set prior to the event                                                          | Low                      | Fixed        |
| \[L-5\]  | Missing input validation in `VectorBnoding.initializeBond()` for `controlVariable` and `_minimumPrice`                                                                           | Low                      | Acknowledged |
| \[L-6\]  | Missing input validation in `VectorBnoding.setFeeAndFeeTo()`                                                                                                                     | Low                      | Fixed        |
| \[L-7\]  | Missing input validation in `VectorBnoding.setBondTerms()` when updating `maxPayout`                                                                                             | Low                      | Fixed        |
| \[L-8\]  | Missing input validation in `VectorTreasury.setRedemtionActive()`                                                                                                                | Low                      | Fixed        |
| \[L-9\]  | Adhere to the Checks-Effects-Interactions pattern in `VectorETH.addMangedRestakedLST()` to avoid potential reentrancy with future LST integrations                               | Low                      | Fixed        |
| \[L-10\] | Ether transfers for `teamWallet` and `distributor` do not check the return boolean in `Vector.swapBack()`                                                                        | Low                      | Acknowledged |
| \[L-11\] | Use Ownable2Step library instead of Ownable for extra security                                                                                                                   | Low                      | Acknowledged |
| \[L-12\] | The view function `VECStaking.secondsToNextEpoch()` will revert from the moment the epoch reaches its end                                                                        | Low                      | Fixed        |
| \[L-13\] | Unbounded gas loop on `VectorTreasury.burnAndRedeem()`, as the length of `redeemableTokens` is not limited                                                                       | Low                      | Acknowledged |
| \[L-14\] | Lack of slippage protection in `VectorETH.swapTokensForEth()` will likely be sandwiched, giving a less favorable output of the swap                                              | Low                      | Acknowledged |

------------------------------------------------------------------

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
|:-------------------|:------------:|:--------------:|:-----------:|
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

- Delivery date: `2024-01-25`
- Duration of the audit: 7 days
- Audited commit
  hash: [9bf32f455121328a774a9a1c74bcb334b69dd704](https://github.com/vectorreserve/vr/commit/9bf32f455121328a774a9a1c74bcb334b69dd704)
- Review Commit hash: [....]

- The following contracts are in the scope of the audit:

  | File                                     | nSLOC    | 
    |------------------------------------------|----------|
  | `contracts/reward/RewardDistributor.sol` | 71       |
  | `contracts/tokens/sVEC.sol`              | 150      |
  | `contracts/treasury/VectorTreasury.sol`  | 78       |
  | `contracts/bonds/VectorBonding.sol`      | 318      |
  | `contracts/tokens/VectorETH.sol`         | 135      |
  | `contracts/tokens/StakedVectorETH.sol`   | 30       |
  | `contracts/staking/VECStaking.sol`       | 66       |
  | `contracts/tokens/Vector.sol`            | 255      |
  | **Total**                                | **1103** |

# Vector Protocol Overview

Vector Reserve [@vectorreserve](https://twitter.com/vectorreserve) is DeFi’s first Liquidity Layer and issuer of the
first LPD: vETH. Powered by [@EigenLayer](https://twitter.com/EigenLayer) and Superfluid Staking.

Read more about Vector Reserve Protocol:

- [Medium article](https://medium.com/@vectorreserve/a-simple-guide-to-superfluid-staking-c7f0ece9ce03)
- [Documentattion](https://vector-reserve.gitbook.io/vector-reserve/introduction/what-is-vector-reserve)

--------------------------------------------

# Findings

## High risk

### [H-1] An attacker can inflate the share's price of `StakedVectorETH`, and steal all the vETH tokens from other stakers

The function `StakedVectorETH::stake()` calculates the amount of shares to mint to a `msg.sender`, dividing by
`vETH.balanceOf()` of the staking contract. This balance can be inflated by transferring tokens to the contract.

```javascript
    function stake(uint256 _amount) public {
@>      uint256 totalvETH = vETH.balanceOf(address(this));
        uint256 totalShares = totalSupply();

        if (totalShares == 0 || totalvETH == 0) {
@>          _mint(_msgSender(), _amount);
            emit Stake(_msgSender(), _amount);
        } else {
@>          uint256 _shares = _amount * totalShares / totalvETH;
            _mint(_msgSender(), _shares);
            emit Stake(_msgSender(), _shares);
        }
        vETH.transferFrom(_msgSender(), address(this), _amount);
    }
```

Moreover, the calculation of the shares minted to the depositor rounds down, which means that 0 shares can be minted
if `totalvETH` is higher than `_amount * totalShares`.
Lastly, when the contract is initialized, there are no restrictions for the first depositor.

Therefore, when the contract is deployed, before any shares have been minted, the calculation of the shares is highly
manipulable.
An attacker can stake with `amount=1 wei`, which would mint exactly 1 share, and after that, donate a large amount of
vETH
tokens to the contract. Any time an honest staker calls `stake()` after that, the calculation of the shares "price"
will be severely inflated by the transferred tokens, which are accounted in the balance, but not as part of the shares.
This will cause future stakers to receive 0 shares if they don't stake a large enough amount of vETH.

#### Impact

- Best-case scenario: The attacker mints one share, transfers tokens, and the team is made aware of the attack:
  contracts need to
  be redeployed.
- Middle-case scenario: The attacker front runs the first depositor, performing this inflation attack, and steals all
  the
  tokens of the first depositor.
- Worst-case scenario: The attacker mints one share and transfers tokens, without the team realizing about it. Multiple
  users stake vETH, and the attacker later withdraws all tokens from all users.

#### Proof of concept

With a freshly deployed contract, before anybody stakes:

- The attacker stakes 1 wei of `vETH`, therefore receiving 1 wei of shares. The share price is `1:1` `vETH:shares`
- The attacker transfers an amount of `vETH` to the staking contract, for instance, 10 `vETH` (10 * 1e18). This inflates
  the shares to 10 * 1e18 + 1 vETH per share.
- From now on, any user calling `stake()` with an amount lower than (10 * 1e18 + 1) `vETH`, will get the share price
  calculation
  rounded down to 0, and therefore receive 0 shares.
- The attacker can withdraw his single share at any time, receiving all the `vETH` in the contract from other stakers

#### Mitigation

Instead of relying on `vETH.balanceOf()` keep track of the staked `vETH` with a storage variable like `totalStakedvETH`.
Use this value to calculate the shares instead of the contract's balance.

Alternatively, initialize the contract with a big enough amount of shares minted to the zero address or to the deployer.

#### Team response: Fixed

The following requirement has been added: if the pool has 0 shares or 0 token balance, only the deployer is allowed
to deposit.

```diff
    constructor(IERC20 _vETH) {
        vETH = _vETH;
+       deployer = msg.sender;
    }

    function stake(uint256 _amount) public {
        uint256 totalvETH = vETH.balanceOf(address(this));
        uint256 totalShares = totalSupply();
        if (totalShares == 0 || totalvETH == 0) {
+           require(msg.sender == deployer, "Deployer to be first stake to initialize proper shares");
            _mint(_msgSender(), _amount);
            emit Stake(_msgSender(), _amount);
        } else {
            uint256 _shares = _amount * totalShares / totalvETH;
            _mint(_msgSender(), _shares);
            emit Stake(_msgSender(), _shares);
        }
        vETH.transferFrom(_msgSender(), address(this), _amount);
    }
```

The solution is valid as long as:

- The deployer acts in goodwill (because he could potentially perform the attack himself)
- The deployer deposits an amount large enough

### [H-2] A malicious address can postpone indefinitely the redemption of a victim's bond, making the funds inaccessible to the victim

The function `VectorBonding.deposit()` can be called providing a `_depositor` address.
This function will overwrite the `bond.lastBlockTimestamp`, setting it to `block.timestamp`.

```javascript
    function deposit(
        // ...

        // @audit _depositor's bond info is overwritten here
        bondInfo[_depositor] = Bond({
            payout: bondInfo[_depositor].payout.add(payout),
            vesting: terms.vestingTerm,
@>          lastBlockTimestamp: block.timestamp, 
            truePricePaid: bondPrice() 
        });
```

On the other hand, the `redeem()` function calls `percentVestedFor()` to read the percentage of the bond that is
redeemable. Every time a deposit is made, the `bond.lastBlockTimestamp` is updated to `block.timestamp`, so right
after the deposit is made, the redeemable percentage will always be 0, ignoring old bonds and ongoing bonds that have not
been redeemed yet.

```javascript
    function percentVestedFor(address _depositor) public view returns (uint256 percentVested_) {
        Bond memory bond = bondInfo[_depositor];
@>      uint256 timestampSinceLast = block.timestamp.sub(bond.lastBlockTimestamp);

        uint256 vesting = bond.vesting;

        if (vesting > 0) {
            percentVested_ = timestampSinceLast.mul(10_000).div(vesting);
        } else {
            percentVested_ = 0;
        }
    }
```

Essentially, the duration of an existing bond is ignored and overwritten when a new deposit is made.

#### Impact

Malicious addresses can `deposit()` on behalf of a victim (the `_depositor`) and reset the vesting period for the
victim,
postponing the time when the bond will be redeemable. In an extremely evil scenario, a malicious address could
front-run all redemptions from all users, postponing their vesting terms, resulting in all redemptions redeeming 0% of
the bonds. The attacker will need to deposit the minimum amount every time, so it wouldn't be a cheap attack, but
possible nonetheless.

#### Proof of concept

- Alice deposits.
- 75% of the vesting time passes
- If Alice redeems now, `percentVested` would read 75%, and therefore would get 75% of the bond redeemed.
- A malicious address calls deposit() with minimum amount, providing Alice as `_depositor`. Alice's `lastBlockTimestamp`
  is reset, the percentage redeemable is 0%, to redeem she will have to wait for the entire vesting term to pass.

#### Mitigation

Only allow deposits for the caller, substituting `_depositor` for `msg.sender` in the `deposit()` function.

```diff
    deposit(
        // ...

-       bondInfo[_depositor] = Bond({
+       bondInfo[msg.sender] = Bond({
-           payout: bondInfo[_depositor].payout.add(payout),
+           payout: bondInfo[msg.sender].payout.add(payout),
            vesting: terms.vestingTerm,
            lastBlockTimestamp: block.timestamp,
            truePricePaid: bondPrice()
        });

```

#### Team response: Fixed

The `VectorBonding.deposit()` function can now only be called to deposit on behalf of `msg.sender`. This stops malicious
addresses from postponing the victim's redemption dates.

------------------------------------------------------------------

## Medium risk

### [M-1] Unaware users will postpone their own redemption date if they make a new deposit before redeeming an existing bond

Similarly to [H-2], when a user makes a new deposit for himself, while an existing bond hasn't been redeemed, the
redemption date is also postponed.

The reason why this issue has been separated from the above one is because the solutions to solve both issues don't
necessarily overlap completely.

_>> In fact, the team decided to implement the solution that fixes H-2, but not this issue._

#### Proof of concept

- Bob deposits
- 90% of the vesting term passes. He should be able to redeem 90% now if he wanted to.
- Bob makes a new deposit. `lastBlockTimestamp` is updated to `block.timestamp`, and therefore `percentVested` becomes
  0%.
- Bob cannot redeem anything anymore, and the new redemption date is postponed a full vesting term ahead.

#### Mitigation

When a deposit is made, check if an existing bond can be redeemed (partially or fully), and if that is the case, redeem
first the existing bond before overwriting the bond terms.

```diff
    function deposit(
        // ...

        require(_depositor != address(0), "Invalid address");
        require(IERC20(principalToken).balanceOf(msg.sender) >= _amount, "Balance too low");

+       if (bondInfo[_depositor].payout > 0) {
+           redeem(_depositor);
+       }

        // ...
```

### Team response: Acknowledged

No action was taken.

### [M-2] When the owner of `VectorBonding` sets new Vesting terms calling `setBondTerms()`, malicious addresses can force existing bonds of other users to adopt the new vesting terms

When the owner sets new vesting terms, the malicious address can call deposit() on behalf of the victim with the
the smallest amount, overwriting the bond terms of the victim to the recently updated terms.

```javascript
    function deposit(
        // ...
        bondInfo[_depositor] = Bond({
            payout: bondInfo[_depositor].payout.add(payout),
@>          vesting: terms.vestingTerm,
            lastBlockTimestamp: block.timestamp, 
            truePricePaid: bondPrice() 
        });
```

Also, a user that makes a new deposit with an un-redeemed bond, will overwrite its own.

#### Impact

Particularly annoying for the user if the new vesting term set by the owner is significantly higher.

#### Proof of concept

- Alice deposits
- Some seconds before Alice's vesting terms finish, a malicious address makes the smallest possible deposit on behalf of
  Alice.
- Alice's vesting terms are overwritten, and therefore she has to wait for the full new vesting period before being able to
  redeem

#### Mitigation

- Option 1: Remove the possibility of updating vesting terms, and consider the option of deploying new contracts when
  different
  vesting terms are desired.
- Option 2: Remove `_depositor` input parameter and only update `bondInfo` for `msg.sender`.

#### Team response: Fixed

The `VectorBonding.deposit()` function can now only be called to deposit on behalf of `msg.sender`. This stops malicious
addresses from updating vesting terms of victims.

### [M-3] VEC transfers can revert when the `VEC` ether balance is 0 at the end of `swapBack()` function, because `vETH.deposit()` reverts with `amount=0`

A the end of the `swapBack()` function, the remaining ether in the contract is swapped for WETH,
and the WETH is deposited into the `vETH` contract:

```javascript
    function swapBack() internal {
        // ...
        uint256 _balance = address(this).balance;
        IWETH(WETH).deposit{value: _balance}();
@>      IvETH(vETH).deposit(WETH, treasury, _balance);
    }
```

However, the `VectorETH.deposit()` reverts if called with `amount=0`:

```javascript
    function deposit(
        address _restakedLST,
        address _to,
        uint256 _amount
    ) external {
        require(restakedLST[_restakedLST], "Not approved restaked LST");
@>      require(_amount > 0, "Can not deposit 0");
```

Therefore, if the ether contract balance was 0 right before depositing, the `vETH.deposit()` call will revert,
reverting the entire `VEC` transfer.

#### Impact

When the circumstance of 0 ether balance at the end of `swapBack()` takes place, the transfer of VEC tokens will revert.

#### Mitigation

Only perform the `vETH` deposit if the ether balance is greater than zero:

```diff
    function swapBack() internal {
        // ...
        uint256 _balance = address(this).balance;
+       if (_balance > 0) {
            IWETH(WETH).deposit{value: _balance}();
            IvETH(vETH).deposit(WETH, treasury, _balance);
+       }
    }
```

#### Team response: Fixed

The above fix was implemented.

### [M-4] No slippage protection in `VectorBond.deposit()` for BondType `WETHTOLP` will give WETH depositors a less favorable deal when WETH is swapped for VEC

When depositing WETH for BondType `WETHTOLP`, the contract swaps half of the sent WETH for VEC.
This swap is done without slippage protection as it accepts any amount of VEC back.

An attacker can perform a sandwich attack, raising the price of VEC before the deposit happens,
and selling the VEC tokens back after the deposit transaction. What makes it even more interesting
is that liquidity is added in the same transaction, which will further reduce the price impact of
the attacker in the sell transaction.

The user cannot prevent this situation as he has no way of configuring slippage protection on this swap,
because it is hardcoded to 0 (any amount of tokens accepted).

```javascript
    function swapETHForTokens(uint256 ethAmount) internal {
        address[] memory path = new address[](2);
        path[0] = uniswapV2Router.WETH();
        path[1] = address(VEC);

        uniswapV2Router.swapExactTokensForTokensSupportingFeeOnTransferTokens(
            ethAmount,
@>          0, // accept any amount of VEC 
            path,
            address(this),
            block.timestamp // ok
        );
    }
```

Moreover, the `addLiquidity()` function doesn't implement any type of slippage protection either:

```javascript
    function addLiquidity(uint256 tokenAmount, uint256 ethAmount) internal {
        uniswapV2Router.addLiquidity(
            address(VEC),
            address(principalToken), 
            tokenAmount, 
            ethAmount, 
@>          0, 
@>          0, 
            address(treasury), 
            block.timestamp 
        );
    }
```

#### Impact

The depositor will swap the WETH tokens at a much less favorable rate.
A wealthy attacker with a large amount of WETH can significantly impact the price.

#### Proof of concept

- An honest user calls `VectorBonding.deposit()`, with BondType `WETHTOLP` and a significant amount
- A frontrunner sees that transaction, and performs a sandwich attack:
    - Frontruns the deposit transaction, buying a large amount of VEC tokens with WETH
    - Let the deposit call be executed, which will swap WETH for VEC at a higher price, and then the liquidity is
      added (note that more liquidity means even less price impact in following transactions).
    - Backruns the deposit transaction, selling the VEC tokens at a much more favorable rate. A risk-free trade at the
      the expense of the user, who has gotten a worse deal in the swap.

#### Mitigation

Allow the user to choose a slippage percentage. The frontend could make a call to the Uniswap pool and get a quote for a
certain amount of WETH to VEC tokens.
That minimum amount of VEC tokens should be an input parameter to the `VectorBond.deposit()`.

#### Team response: Acknowledged

No action taken on this issue.

### [M-5] Flash arbitrage can leverage small deviations between the LST pegs and the exchange rate given by `vETHPerRestakedLST` and slowly drain the protocol reserves

The contract `vectorETH` allows deposits: WETH, LSTs, LRTs. As these tokens do not have an exact 1:1 peg to ether,
(see [LSTs peg monitoring](https://dune.com/satsbruh/lst-depeg-dashboard)), the state variable `vETHPerRestakedLST`
is in charge of keeping track of the exchange rate between the deposited tokens and the `vETH` tokens minted. The
owner of the contract can adjust the exchange rate, but it is not expected to track every small fluctuation of the LST
pegs, as it would cost a fortune on fees for the contract owner.

If an LST peg fluctuates, and the `vETHPerRestakedLST` is not updated accordingly, but the Uniswap pools are,
there is an arbitrage opportunity. The profit that these arbitrageurs can make
is extracted from the reserves of the value of the `vectorETH` contract.

This is possible because the functions `deposit()` and `redeem()` have no time-based restrictions, and it is possible
to call both functions as part of the same transaction. This opens the door for flash-loan arbitrage
which increases the impact of the arbitrage and therefore increases the losses experienced by the
contract reserves.

In [this section on the documentation](https://vector-reserve.gitbook.io/vector-reserve/introduction/what-is-vector-reserve)
it is claimed that

_"When you pair a native asset with a derivative of the native asset that is fully backed, Impermanent Loss (IL) ceases
to be an issue for yield generation, meaning the backing of vETH cannot diminish - only increase. The concept of the LPD
means that vETH can act essentially as an LST, with the same price peg to ETH, albeit with substantially higher
returns."_

However, this is not true if the reserves of LSTs backing up `vETH` are reduced over time.

#### Impact

Every time there is a peg fluctuation in any of the `approvedRestakedLSTs`, and the depeg is reflected in Uniswap
pools,
but not in the `vETHPerRestakedLST`, the arbitrageurs can drain some reserves from the `vectorETH` contract.
Over time, this can mean a significant loss for the protocol and a decrease in the value of `vETH`.

#### Proof of concept

Let's go through an example of how it would happen. The numbers are made up so that the math is easy to follow.
Let's imagine two different LST tokens supported by vectorETH, `LST1` and `LST2`. They have different pegs to WETH, so
the ratio between them is `1:1.05`. This ratio is correctly reflected in the Uniswap pool, and also in the
`VectorETH`. This simply means that if LST1/WETH is 1:1, and LST2/WETH is 1:1.05, then LST1:LST2 is 1:1.05.
Same in VectorETh, if LST1/vETH is 1:X, and LST2/vETH is 1:1.05X, then LST1:LST2 is 1:1.05. Let's assume for now that
X=1, which simply means that vETH/WETH = 1:1, for the sake of simplicity,

With that, we have the following initial state:

- Uniswap Pair LST1/LST2 ratio: `1:1.05`
- VectorETH LST1/LST2 ratio: `1:1.05`
- vector LSTs reserves: `10000 LST1 + 10000 LST2`
- vector reserves in ETH: `(10000 * 1) + (10000 * 1.05) = 20500 ETH`

Then, let's say that LST2 peg changes shortly but significantly from `1:1.05` to `1:1`, and that VectorETH does not
update the `vETHPerRestakedLST` accordingly. Then, the new state is:

- Uniswap Pair LST1/LST2 ratio: `1:1`
- VectorETH LST1/LST2 ratio: `1:1.05`

A flash-loaner can exploit this situation by following these steps:

- flash loan `1000 LST2`  (or any large amount)
- Swap in Uniswap `1000 LST2` for `1000 LST1` (because the ratio is 1:1)
- call `VectorETH.deposit()` with `amount=1000 LST1` -> receive `1000 vETH`
- call `VectorETH.redeem()` with `amount=1000 vETH` -> receive `1050 LST2`
- payback the `1000 LST2` to the flash loan provider, plus fees.
- profit `50 LST2` minus the flash loan fees

Now, LST2 peg goes back to normal, so we are back to the initial state, except the value of the reserves has decreased:

- VectorETH LSTs reserves:  `11000 LST1 + 9000 LST2`
- VectorETH reserves in ETH: `(11000 * 1) + (9000 * 1.05) = 20450 ETH`
- VectorETH Reserves lost: `20500 - 20450 = 50 ETH`

#### Mitigation

Some ideas:

- Disallow flash loans by disallowing `vETH` transfers in the same block that they are minted (transfers from address(0)
  should be allowed).
  By doing so, the `vETH` cannot be deposited again in the same transaction as they have been minted. However,
  flash loans simply allow small guys to play with big quantities, but this measure won't stop ETH whales from benefit
  from arbitrage opportunities.
- Limit the amounts that can be deposited and redeemed in a single transaction. Probably not ideal.

#### Team response: Acknowledged

No action was taken.

### [M-6] - [Centralization] The owner of `VectorETH` can withdraw all LST tokens from all users

The function `VectorETH.recoverTokens()` requires that the token to recover is not one of the approved `restakedLST`.
However,
the contract owner can also remove the token from the allow list by calling `VectorETH.removeRestakedLST()`,
which makes the mentioned requirement virtually useless:

```javascript
    function recoverTokens(address _to, address _token, uint256 _amount) external onlyOwner {
        // @audit This requirement can be bypassed because the contract owner can remove restakedLSTs
@>      require(!restakedLST[_token], "Can Not transfer restaked LST");
        IERC20(_token).transfer(_to, _amount);
    }
```

#### Impact

If the contract owner acts maliciously or is compromised, all users could get their restakedLST tokens rugged.

- **Likelihood**: low, as it is expected that the contract owner acts out of goodwill and that they use a multisig
  safe for contract owners
- **Impact**: high, as users would lose access to their LSTs

#### Mitigation

Either one of the two options could fix the issue:

- Remove the function `VectorETH.recoverTokens()` from the contract.
- Leave the function `VectorETH.recoverTokens()` untouched, but don't allow the contract owner to remove restakedLST
  tokens. If a token is added, it can never be removed, and therefore cannot be recovered by
  the `VectorETH.recoverTokens()` function.

#### Team response: Acknowledged

No action was taken.

### [M-7] - [centralization] The owner of `VectorETH` can halt redemptions of any (or all) LST tokens

The owner can remove LST tokens by calling `VectorETH.removeRestakedLST()`. When this is done,
redemptions of that LST cannot be performed anymore. Therefore a malicious / compromised contract owner
can halt all redemptions by removing all LSTs from the allowed list.

```javascript
    function redeem(address _restakedLSTToReceive, address _to, uint256 _vETHToRedeem) external {
        require(redemtionsActive, "Redemtions not active");
@>      require(restakedLST[_restakedLSTToReceive], "Not restaked LST");
```

- **Likelihood**: low, as it is expected that the contract owner acts out of goodwill and that they use a multisig
  safe for contract owners
- **Impact**: high, as users would lose access to their LSTs

#### Mitigation

Reconsider if the function `VectorETH.removeRestakedLST()` is necessary at all. My recommendation is to remove this
function from the contract, and never allow the contract owner to remove an LST from the list of approved tokens.

- If the intention is to protect users from redeeming depeeged LSTs -> it is the responsibility of the users to know
  which tokens they want to redeem. Don't take this responsibility for them.
- If the intention is to halt deposits of LSTs -> keep the function that removes allowed tokens, but its impact should
  be reduced to the `deposit()` function.

#### Team response: Acknowledged

No action was taken.

------------------------------------------------------------------

## Low risk

### [L-1] The stepwise jump caused by updating the `vETHPerLST` in `VectorETH` can be sandwiched, providing a free-risk trade to frontrunners, without providing any value to the protocol

When the owner updates the `vETHPerLST` for a given LST, this happens in a step-wise jump manner.
A frontrunner that is observing the mempool can take advantage of this situation and take a free-risk trade, profiting
at the expense of the protocol, without depositing for more than a single block.

```javascript
    function updatevETHPerLST(address _restakedLST, uint256 _vETHPerLST) external onlyOwner {
        require(restakedLST[_restakedLST], "Not restaked LST");
        vETHPerRestakedLST[_restakedLST] = _vETHPerLST;
        emit vETHPerTokenUpdated(_restakedLST, _vETHPerLST);
    }
```

#### Impact

The protocol will see the LST reserves slowly depleted in every one of these step-wise jumps.
Moreover, all the other WETH holders will also see their

- Likelihood: low, as the owner is not expected to update the ratio regularly, nor set redeemable tokens.
- Impact: medium, as the only affected is the protocol, which will get the LST reserves depleted slowly in each of these
  transactions.

#### Proof of Concept

- The contract owner broadcasts a transaction to decrease the WETH's `vETHPerLST` from 1.05->1 vETH/WETH
- A front-runner sees the transaction in the mempool
- He front runs the transaction, calling `VectorETH.deposit()` and depositing as much WETH as he has (remember that this
  is
  a free-risk trade). Let's say he deposits 10 WETH and therefore receives 10.5 vETH.
- The `VectorETH.updatevETHPerLST()` is executed right after, changing the exchange rate from 1.05 to 1 vETH/WETH
- The attacker back runs it, redeeming the 10.5 vETH for 10.5 WETH

The attacker just made a free-risk profit of 0.5 ether, at the expense of the protocol, and other WETh depositors.

Note that this attack could be reproduced the other way around, if the `vETHperLST` is increased, but only by already
holding `vETH` prior to this, and then doing the process in the inverse order. Redeem `vETH` for `WETH`, let the ratio
change, then deposit `WETH` again.

#### Mitigation

A potential way to be protected against these stepwise jumps is to enforce a minimum time between `deposit()`
and `redeem()`, for instance, a week.

#### Team response: Acknowledged

No action was taken.

### [L-2] Use OpenZeppelin's SafeERC20 library wherever the ERC20 tokens are not known in advance

Not all ERC20 tokens adhere to the ERC20 token standards, and therefore it is recommended to use
OpenZeppelin's [SafeERC20 library](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol)
whenever the token is not known in advance. This can happen if the protocol has the door open to integrate
with future tokens that are not known in advance.

For such cases it is recommended to substitute all `token.transfer()` with `token.safeTransfer()` and
all `token.transferFrom()` with `token.safeTransferFrom()`.

Unsafe ERC20 transfers are in multiple places, but in some of them, the tokens are known (VEC, vETH, and other tokens
part
of the Vector Reserve protocol).
However, there are some places where the tokens are not known in advance, and the SafeERC20 library is highly
recommended:

- `VectorETH.deposit()`, `VectorETH.redeem()`, `VectorETH.manageRestakedLST()`, `VectorETH.addMangedRestakedLST()`: as
  the owner can include new LSTs in the future.
- `VectorETH.recoverTokens()` as the token address is unknown in advance. The function will currently fail for weird
  ERC20 tokens like USDT (6 decimals), etc. This makes this function useless.

#### Team response: Fixed

SafeERC20 is used in all the mentioned instances.

### [L-3] `VectorETH.deposit()` and `VectorETH.redeem()` assume that all future LSTs will have 18 decimals

It is recommended to query the decimals of the LSTs, so that this contract can also support different amounts of
decimals.

```javascript
    function deposit(
        address _restakedLST,
        address _to,
        uint256 _amount
    ) external {
        require(restakedLST[_restakedLST], "Not approved restaked LST");
        require(_amount > 0, "Can not deposit 0");
@>      uint256 _amountToMint = (vETHPerRestakedLST[_restakedLST] * _amount) / 1e18;
        // ...
```

```javascript
    function redeem(address _restakedLSTToReceive, address _to, uint256 _vETHToRedeem) external {
        // ...
@>      uint256 _restakedLSTToSend = (1e18 * _vETHToRedeem) / vETHPerRestakedLST[_restakedLSTToReceive];
        // ...
```

#### Mitigation

A possible mitigation would be to query for the decimals of the `restakedLST` tokens:

```diff
    function deposit(
        // ... 
-       uint256 _amountToMint = (vETHPerRestakedLST[_restakedLST] * _amount) / 1e18;
+       uint256 _amountToMint = (vETHPerRestakedLST[_restakedLST] * _amount) / 10 ** IERC20Metadata(_restakedLST).decimals();
        // ...
```

... (and the same thing in `redeem()`).

However, this solution would assume that all restakedLSTs will implement the `decimals()` selector. If this is not the
case, `deposit()` and `redeem()` will revert.

A more complex/cumbersome solution would be to attempt to read the decimals, and fallback to 18 if such a selector is
not
implemented.

#### Team response: Fixed

The above fix was implemented. The team will make sure that tokens with different number of decimals than 18 are not
included as restakedLSTs.

### [L-4] The event `SwapAndLiquify()` emitted by `Vector` will always log `tokensForLiquidity=0` as it is set prior to the event

In `swapBack()`, the variable `tokensForLiquidity` is set to 0 before the event is emitted.
Therefore, the event will always log `tokensForLiquidity=0`.

```javascript
    function swapBack() internal {
        
        // ...
        
@>      tokensForLiquidity = 0;
        tokensForBacking = 0;
        tokensForTeam = 0;
        tokensForvETHRewards = 0;

        (success,) = address(teamWallet).call{value: ethForTeam}("");
        (success,) = address(distributor).call{value: ethForvETHReward}("");

        if (liquidityTokens > 0 && ethForLiquidity > 0) {
            addLiquidity(liquidityTokens, ethForLiquidity);
@>          emit SwapAndLiquify(amountToSwapForETH, ethForLiquidity, tokensForLiquidity);
        }
```

#### Impact

Only off-chain entities reading emitted events would be affected.

#### Mitigation

```diff
    function swapBack() internal {
        // ...
        if (liquidityTokens > 0 && ethForLiquidity > 0) {
            addLiquidity(liquidityTokens, ethForLiquidity);
-           emit SwapAndLiquify(amountToSwapForETH, ethForLiquidity, tokensForLiquidity);
+           emit SwapAndLiquify(amountToSwapForETH, ethForLiquidity, liquidityTokens);
        }
```

#### Team response: Acknowledged

No action was taken.

### [L-5] Missing input validation in `VectorBnoding.initializeBond()` for `controlVariable` and `_minimumPrice`.

The units of `controlVariable` and `_minimumPrice` are not straightforward units, and are therefore prone to errors.
It is recommended that some checks are performed when initializing the bond, to make sure that the variables relate
correctly to each other.

#### Team response: Acknowledged

No action was taken.

### [L-6] Missing input validation in `VectorBnoding.setFeeAndFeeTo()`

The input argument `feePercent_` should never exceed `FEE_DENOM`,
as otherwise the function `payoutFor()` will revert with an underflow when
subtracting `amount - fee`, as `fee` will be higher than `amount`.

#### Team response: Fixed

Requirements are in place.

### [L-7] Missing input validation in `VectorBnoding.setBondTerms()` when updating `maxPayout`

The new `maxPayout` should be lower than `100.000`, as that is the reference `100%` for this percentage variable.

#### Team response: Fixed

Requirements are in place.

### [L-8] Missing input validation in `VectorTreasury.setRedemtionActive()`

The input argument `_percentRedeemable` should never be higher than the reference 100% value, which is `100% == 100`.

#### Recommended mitigation

```diff
    function setRedemtionActive(uint256 _percentRedeemable) external onlyOwner {
        redeemtionActive = true;
+       require(_percentRedeemable < 100, "Invalid percentage");
        percentRedeemable = _percentRedeemable;
    }
```

#### Team response: Fixed

Requirements are in place.

### [L-9] Adhere to the Checks-Effects-Interactions pattern in `VectorETH.addMangedRestakedLST()` to avoid potential reentrancy with future LST integrations

The following function performs an external call before updating the state. For most `_restakedLST` this would not be an
issue. However, if a future
`_restakedLST` implemented `ERC-777` that opens up the door to reentrancy, this function could be exploited as the state
is updated after the external call is made.

```javascript
    function addMangedRestakedLST(address _restakedLST, uint256 _amount) external {
        require(approvedManager[msg.sender], "Not approved manager");
        require(restakedLST[_restakedLST], "Not restaked LST");
        // This performs an external call to `_restakedLST`
@>      IERC20(_restakedLST).transferFrom(msg.sender, address(this), _amount);

        // The following two lines update the state
        if (_amount > restakedLSTManaged[_restakedLST]) {
@>          restakedLSTManaged[_restakedLST] = 0;
        }
        else {
@>          restakedLSTManaged[_restakedLST] -= _amount;
        }

        emit TokenManagedReaded(_restakedLST, _amount);
    }
```

#### Mitigation

Perform the external call at the end of the function.

```diff
    function addMangedRestakedLST(address _restakedLST, uint256 _amount) external {
        require(approvedManager[msg.sender], "Not approved manager");
        require(restakedLST[_restakedLST], "Not restaked LST");
        // This performs an external call to `_restakedLST`
-       IERC20(_restakedLST).transferFrom(msg.sender, address(this), _amount);

        // The following two lines update the state
        if (_amount > restakedLSTManaged[_restakedLST]) {
            restakedLSTManaged[_restakedLST] = 0;
        }
        else {
            restakedLSTManaged[_restakedLST] -= _amount;
        }

+       IERC20(_restakedLST).transferFrom(msg.sender, address(this), _amount);
        emit TokenManagedReaded(_restakedLST, _amount);
    }
```

#### Team response: Fixed

The above fix was implemented.

### [L-10] Ether transfers for `teamWallet` and `distributor` do not check the return boolean in `Vector.swapBack()`

The function `Vector.swapBack()` performs ether transfers to the `teamWallet` and `distributor` addresses.
However, the boolean return parameter that determines if the transfer was successful is not checked.

```javascript
    function swapBack() internal {
        // ...
        
        bool success;
        
        // ...

        (success,) = address(teamWallet).call{value: ethForTeam}("");
        (success,) = address(distributor).call{value: ethForvETHReward}("");
        
        // ...

```

I presume that the team did so intentionally, to avoid that the transfers were halted when these addresses were not
correctly set. however, a different approach is suggested in the "Mitigation" section.

#### Impact

If any of the two transfers fails, the contract goes on with the execution, and the recipients do not receive their eth.
The current implementation of `RewardsDistributor()` implements the `receive()` function correctly, so the users should
not be impacted.

#### Mitigation

Favor the Pull-over-push pattern. Instead of transferring the ether as part of the `swapBack()` function,
store the amounts in variables that can only be claimable by the corresponding wallets. This way, if the transfer fails,
it is contained to the claim function, and it is known when it fails.

#### Team response: Acknowledged

No action was taken.

### [L-11] Use Ownable2Step library instead of Ownable for extra security

Multiple contracts use the `Ownable` library from OpenZeppelin.
For extra safety, it is recommended to
use [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol),
which requires that the new owner accepts the ownership transfer.
this removes the risk of setting the incorrect address as the new owner.

The following contracts inherit `Ownable`:

- Vector
- VectorETH
- Distributor
- VectorTreasury
- VECStaking

#### Team response: Acknowledged

No action was taken.

### [L-12] The view function `VECStaking.secondsToNextEpoch()` will revert from the moment the epoch reaches its end

To calculate the remaining seconds until the next epoch, the following subtraction is used:

```javascript
    function secondsToNextEpoch() external view returns (uint256 seconds_) {
        // @audit-issue This will revert when the end is passed until the rebase is called again. Return zero if timestamp > end
@>      return epoch.end - block.timestamp;
    }

```

This will revert when `epoch.end < block.timestamp`, which is when the epoch finishes. Presumably, soon after that the
new epoch will be reset by calling `rebase()`.
However, any frontend reading this variable will get reverted until `rebase()` is called

#### Mitigation

```diff
    function secondsToNextEpoch() external view returns (uint256 seconds_) {
        // @audit-issue This will revert when the end is passed until the rebase is called again. Return zero if timestamp > end
-       return epoch.end - block.timestamp;
+       return epoch.end > block.timestamp ? epoch.end - block.timestamp : 0;
    }

```

#### Team response: Fixed

The above fix was implemented.

### [L-12] Unbounded gas loop on `VectorTreasury.burnAndRedeem()` as the length of `redeemableTokens` is not limited

The contract owners can add tokens to the array of redeemable tokens of the treasury: `redeemableTokens`.
Moreover, the function `VectorTreasury.burnAndRedeem()`, performs operations in a loop. The gas cost of this
transaction will increase linearly with the amount of tokens in the array, potentially hitting the block gas limit.
This seems rather unlikely, as the list of redeemable tokens is expected to be short, and to reach the gas limit, a very
large amount of elements would be needed.

#### Mitigation

Impose a cap on the number of elements of the array when setting them in `setRedeemableTokens()`:

```diff
    function setRedeemableTokens(address[] calldata _tokens) external onlyOwner {
+       require(_tokens.length < MAX_NUMBER_OF_TOKENS, "Length of the array too long");
        redeemableTokens = _tokens;
    }

```

#### Team response: Acknowledged

No action was taken.

### [L-13] Lack of slippage protection in `VectorETH.swapTokensForEth()` will likely be sandwiched, giving a less favorable output of the swap

The `Vector._trasnsfer()` function calls `swapBack()` which calls `swapTokensForEth()` which performs a swap on Uniswap:

```javascript
    function swapTokensForEth(uint256 tokenAmount) internal {
        address[] memory path = new address[](2);
        path[0] = address(this);
        path[1] = uniswapV2Router.WETH();

        // make the swap. This receives ether into this contract
        uniswapV2Router.swapExactTokensForETHSupportingFeeOnTransferTokens(
            tokenAmount,
@>          0, // accept any amount of ETH
            path,
            address(this),
            block.timestamp
        );
    }
```

This lack of slippage makes it very easy for front-runners to sandwich-attack the transaction, by pushing and impacting
the
price
before the swap happens, and then undoing the trade.

Generally, the lack of slippage protection can be considered a high/med-risk finding. However here, the impact on the
user
is virtually none, as this swap only happens to add liquidity as for the protocol. If anything, the protocol is the one
that would get slightly affected.
Moreover, because of the direction of this trade (VEC -> WETH) a successful sandwich attack would require that the
attacker already holds VEC.

#### Team response: Acknowledged

No action was taken.

------------------------------------------------------------------

## Informational findings

These findings do not bring any risk to the protocol, they are just informational and best practices.

- Missing events in some important state-changing functions

    - (Ackn.): `VectorBonding.setFeeAndFeeTo()`
    - (Ackn.): `VectorBonding.decayDebt()`
    - (Fixed): `VectorTreasury.setRedemtionActive()`
    - (Fixed): `VectorTreasury.setRedeemableTokens()`

- (Fixed): `VectorBonding.percentVestedFor()` can return values higher than 10.000 (percentages higher than 100%). This
  doesn't
  bring any high risk to the protocol, because the necessary corrections are in place before using the output of this
  function. However, It would be a better practice to include those checks inside the function itself, to reduce the
  likelihood of errors with future integrations

- (Fixed): `VectorBonding.transferOwnership()` is marked as virtual, and it need not to.

- (Fixed): Wrong natspec comments:
    - `VectorBonding.addLiquidity()` says '/// @dev Invoked in `swapBack()`' which is not even a function of this
      contract

- (Fixed): The function `VectorETH.redeem()` will revert with an underflow if a user attempts to redeem an amount of a
  token that
  is not in the contract balance.
  Instead, consider including a previous check, and throw a more meaningful error so that the user knows the reason for
  the revert. Something
  like: `require(totalRestakedLSTDeposited[_restakedLSTToReceive] >= _restakedLSTToSend, "Not enough funds to redeem LST");`

- (Ackn.): In `VectorETH`, there are 4 boolean setter functions that could be condensed into two, by allowing a boolean
  input
  parameter. `setRedemtionActive()`, `setRedemtionUnactive()`, `addApprovedManager()` and `removeApprovedManager()`,
  condensed into `setRedemtionActive(bool _active)` and `setApprovedManager(address _manager, bool _approved)`.

- (Fixed): Missing important parameters in emitted events for accountability.
    - In `VectorETH`, the event `TokenManaged();` should include the `msg.sender` as a parameter.
    - In `VectorETH`, the event `TokenManagedReaded();` should include the `msg.sender` as a parameter.


- (Ackn.): `VECStaking::epoch.length` is never changed. Consider setting it as a constant variable which would save gas

- (Ackn.): The
  requirement `require(_amount <= VEC.balanceOf(address(this)), "Insufficient VEC balance in the contract");` performed
  in `VECStaking::unstake()` is unnecessary as the VEC transfer would revert anyways. Save some gas.

- (Fixed): The internal function `VECStaking::_send(address _to, uint256 _amount)` is not used anywhere and can be
  removed

- (Ackn.): The private variable `INDEX` in `sVEC` contract is not used for any calculations. It is set and can be read,
  but it has no impact on the rest of the contract. Consider removing it.

- (Ackn.): The immutable variable `LP` in `VectorTreasury` is unused and can be removed

- (Fixed): Floating pragmas are not recommended. Use static pragmas when not developing libraries. Instances of this
  issue:
    - `VectorTreasury.sol`


- (Ackn.): The calculation of `VectorETH.currentBalance()` could be simplified to a single line. See the change and the
  proof
  below.

```diff
        function currentBalance(address _restakedLST) external view returns (uint256 _balance) {
-           uint256 totalDepositsInContract = totalRestakedLSTDeposited[_restakedLST] - restakedLSTManaged[_restakedLST];
-           uint256 accruedLST = IERC20(_restakedLST).balanceOf(address(this)) - totalDepositsInContract;
-           return totalRestakedLSTDeposited[_restakedLST] + accruedLST;

            // @audit-info the above three lines are equivalent to:

+           // return IERC20(_restakedLST).balanceOf(address(this)) + restakedLSTManaged[_restakedLST]

            // PROOF:
            // inContract = deposited - managed   -->   deposited = inContract + managed
            // accrued = balanceOf - inContract   -->  inContract = balance - accrued
            // return = deposited + accrued
            // return = inContract + managed + accrued
            // return = (balance - accrued) + managed + accrued
            // return = balance + managed
```
