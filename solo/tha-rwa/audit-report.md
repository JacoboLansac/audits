# [THA-RWA] - Stage 0 - security review

> Insert logo

A time-boxed security review of **THARWA - Stage 0 contracts** for [**Tharwa Finance**](https://tharwa.finance/), with a focus on smart contract security, conducted by [prismsec.xyz](http://prismsec.xyz/).

Lead author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits/blob/main/README.md).

## Findings Summary

| Finding                                                                                                                                             | Risk   | Description                                                                                                                          | Response |
| :-------------------------------------------------------------------------------------------------------------------------------------------------- | :----- | :----------------------------------------------------------------------------------------------------------------------------------- | :------- |
| [[M-1]](<#m-1-send-tokens-cross-chain-while-the-oft-tokens-are-paused-leads-to-users-losing-their-funds-permanently>)                               | Medium | Send tokens cross-chain while the OFT tokens are paused leads to users losing their funds permanently                                | ✅ Fixed  |
| [[L-1]](<#l-1-if-moving-stablecoins-from-the-thusdswap-to-the-treasury-reverts-for-one-of-the-stablecoins-all-three-will-be-stuck-in-the-contract>) | Low    | If moving stablecoins from the thUSDSwap to the treasury reverts for one of the stablecoins, all three will be stuck in the contract | ✅ Fixed  |


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

- **Preliminary manual review:**
  - Start date: `2025-06-03`
  - End date: `2025-06-05`
  - Commit hash in scope: [6ae61ddcf78bb2e446d188a216eac4fbf3188814](https://github.com/tharwa-finance/contracts-v0/commit/6ae61ddcf78bb2e446d188a216eac4fbf3188814)

- **Mitigation review**
  - Mitigation review delivery date: `2025-06-06`
  - Commit hash: [01e98827917ad2e065c72cec4b034bc0d3345484](https://github.com/tharwa-finance/contracts-v0/commit/01e98827917ad2e065c72cec4b034bc0d3345484)

### Files in original scope

| Files in scope                              | nSLOC |
| ------------------------------------------- | ----- |
| `contracts-v0/stableswap/src/thUSDSwap.sol` | 116   |
| `contracts-v0/thUSD/contracts/thUSD.sol`    | 54    |
| `contracts-v0/TRWA/contracts/TRWA.sol`      | 94    |
| **Total**                                   | 264   |


## Protocol Overview

In its Stage 0, Tharwa consists only of a stable coin `thUSD.sol` that can be minted in exchange for other more established stablecoins (DAI, USDC, USDT). This exchange happens in the `thUSDSwap.sol` contract. The `thUSD` token is a cross-chain token leveraging [*LayerZero*'s](https://layerzero.network/) OFT standard. Initially, `thUSD` will only be minted in Ethereum mainnet, but the team plans to deploy also in other EVM chains like [Base](https://www.base.org/) or non-EVM chains like [Solana](https://www.base.org/).

Lastly, there is a governance token (`TRWA.sol`) which is also a cross-chain token inheriting the OFT standard.


## Architecture high level review

- The contracts are well written and the logic is well structured. 
- The architecture is very simple, which is always good for security.
- Since `thUSD` and `TRWA` are OFT tokens, special attention has been paid to the non-atomic nature of cross-chain transactions. 
- Both OFT contracts inherit `Pausable` which can lead to some niche scenarios in cross-chain communication (see issue M-1).

The diagram below illustrates the different components around the `thUSD` token. The `TRWA` is excluded from the diagram as it would be basically the same:

![image](tharwa-funds-flow-diagram.drawio.png)




# Findings

## High risk

*No findings found.*

## Medium risk

### [M-1] Send tokens cross-chain while the OFT tokens are paused leads to users losing their funds permanently

Both OFT tokens (`thUSD` and `TRWA`) inherit both the OFT standard from LayerZero, and the `Pausable` from OpenZeppelin. 

In a normal scenario, when tokens are sent cross-chain using `OFT.send()`, the tokens are burned in the origin chain, and then minted in the destination chain. Minting tokens uses the `ERC20._update()` function, which is overriden in `thUSD` and `TRWA` with the `whenNotPaused` modifier:

```solidity
>>> function _update(address from, address to, uint256 value) internal override whenNotPaused {
        if (isUserBlacklisted(from)) {
            revert UserBlacklisted(from);
        }
        if (isUserBlacklisted(to)) {
            revert UserBlacklisted(to);
        }
        super._update(from, to, value);
    }
```

On the other hand, the `OFT.send()` function doesn't have such modifier, so sending tokens cross-chain is possible even if origin and destination chains are paused. 

If users try to send tokens cross-chain when the destination chain is paused, the `whenNotPaused` modifier will revert in the destination chain. However, due to the non-atomic nature of cross-chain messages, the destination chain is incapable to roll back the transaction already minted in the origin chain. This means that the tokens are successfully burned in the origin chain, but never minted in the destination chain. The user loses the funds that attempted to transfer

#### Impact: medium

*If users attempt to **send** tokens cross-chain when the destination chain is **paused**, the users will **loose** the transferred tokens.*

- **Probability**: *low*, because the contracts are not meant to be paused except in case of exploit or emergency. 
- **Severity**: *high*, as the users lose the funds that attempted to send cross chain. 

#### Suggested mitigation

- The fastest fix: override the `OFT.send()` function adding the `whenNotPaused` modifier. Note however, that the described scenario is also possible even with this modifier if one of the origin chain is not paused but the destination chain is. So it would require an orchestrated way of pausing transactions in all chains, which is not trivial
- The simplest and most secure fix is to simply remove the Pausable functionality from the OFT tokens. I suggest to use pausability feature in more targeted areas of future contracts (or even in the swap functions from the `thUSDSwap` contract). 

#### Team response: fixed

The team removed the `Pausable` feature from both OFT contracts. 




## Low risk

### [L-1] If moving stablecoins from the thUSDSwap to the treasury reverts for one of the stablecoins, all three will be stuck in the contract

When a user swaps stablecoins (USDC, USDT, DAI) for thUSD, the stablecoins are transferred to the thUSDSwap contract balance, and they stay there until an admin calls `MoveStablecoinsToTreasury()`. When this function is called, the three stablecoins are transferred at the same time to the treasury:

```solidity
    function MoveStablecoinsToTreasury() external onlyOwner {
        address[3] memory tokens = [dai, usdc, usdt];
        uint256[3] memory transferredAmounts;

        for (uint256 i = 0; i < tokens.length; i++) {
            IERC20 token = IERC20(tokens[i]);
            uint256 balance = token.balanceOf(address(this));
            if (balance > 0) {
                token.safeTransfer(treasury, balance);
                transferredAmounts[i] = balance;
            }
        }

        emit StablecoinsMovedToTreasury(
            transferredAmounts[0],
            transferredAmounts[1],
            transferredAmounts[2]
        );
    }
```

As it can be seen, the three tokens are coupled, as the admin cannot chose to transfer one of them only: the three of them have to be transferred together. 

If the transfer of any of the three tokens fails, the entire transaction reverts. Some (unlikely) scenarios that could make these transactions revert are:
- USDC / USDCT blacklisting the treasury or the thUSDSwap contracts
- Permanent **pause** or sunset of any of the three stablecoins (due to some massive depeg, or lack of backing reserves). 

In those unlikely scenarios, if one of them cannot be transferred, the funds from the other two stablecoins would also become stuck in the thUSDSWap contract.

#### Impact: low

A failing transfer of one of the stablecoins causes the other two to also stay locked in the thUSDSwap contract.

- Probability: *very low*
- Impact: *medium*, as only the funds that are currently in the thUSDCSwap are affected, but any funds that have already been transferred are not. 

#### Suggested mitigation

The ground idea is to split the transfers from the thUSDSwap to the treasury in different transactions, or to allow the admin to chose which tokens to transfer. 

However, I would even suggest a simpler architecture, which is that the funds are transferred directly to the treasury inside the swap functions. See an example below for one of the swap functions:

```diff
    function swapUSDT(uint256 usdtAmount) external nonReentrant {
        if (usdtAmount == 0) {
            revert AmountZero();
        }

        // USDT is also 6 decimals so we need to scale it up by 12
        uint256 thAmount = usdtAmount * SCALING_FACTOR;
        if (IERC20(thUSD).balanceOf(address(this)) < thAmount) {
            revert InsufficientLiquidity();
        }

-       IERC20(usdt).safeTransferFrom(msg.sender, address(this), usdtAmount);
+       IERC20(usdt).safeTransferFrom(msg.sender, treasury, usdtAmount);
        IERC20(thUSD).safeTransfer(msg.sender, thAmount);

        emit SwapUSDTForThUSD(msg.sender, usdtAmount, thAmount);
    }
```

In this way, the stablecoins don't have to be held in the `thUSDSwap` contract, and there is no need to periodically move them to the treasury. In this way, it is also possible to remove the `MoveStablecoinsToTreasury()` function completely. Instead it is recommended to have a general `rescueERC20()` instead, to be able to rescue any token sent by mistake to the contract. 

#### Team response: fixed

The team followed the suggestion and the stablecoins are transferred directly to the treasury. A generic function to rescue ERC20 tokens was added.

## Informational issues

- In `thUSDSwap.swapUSDC()`, the calculation `thAmount = usdcAmount * SCALING_FACTOR;` is done twice, and I don't see any reason why doing that. I believe the line is simply duplicated by mistake and can be removed. 

- In `TRWA`, the error TradingNotOpen() is not used. It can be removed. 


