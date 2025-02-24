# Origami oriBGT - Security review

A time-boxed security review of OriBGT for [**Origami Finance**](https://origami.finance/), with a focus on smart contract security and gas optimizations.

**Author**: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits/blob/main/README.md).

## Findings Summary

| Finding | Risk | Description | Response |
| :--- | :--- | :--- | :--- |
| [[H-1]](<#h-1-an-attacker-can-steal-up-to-33-of-the-vault-yield-when-rewardtoken--_asset>) | High | An attacker can steal up to 33% of the vault yield when `rewardToken == _asset` |   |
| [[M-1]](<#m-1-withdrawals-will-revert-with-underflow-when-the-amount-to-withdraw-is-smaller-than-the-managers-balance>) | Medium | Withdrawals will revert with underflow when the amount to withdraw is smaller than the manager's balance |   |
| [[M-2]](<#m-2-an-attacker-can-bypass-the-performance-fees-and-distribute-them-among-vault-shareholders-if-rewardtoken--_asset>) | Medium | An attacker can bypass the performance fees and distribute them among vault shareholders if `rewardToken == _asset` |   |

## Team Response

The team responsibly decided to apply a few architectural changes before doing a second security review. 

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

- **Main review:**
  - Start date: `2025-02-09`
  - End date: `2025-02-22`
  - Effective total time: `25 hours`
  - Commit hash in scope:
    - [edda2db8fd72d84b948d8dd205ef28e380c6af00](https://github.com/TempleDAO/origami/tree/edda2db8fd72d84b948d8dd205ef28e380c6af00)

- **Mitigation review**
  - Mitigation review delivery date: `TBD`
  - Commit hash: -

### Files in original scope

| Contract path                                                             | nSloc | core changes |
| ------------------------------------------------------------------------- | ----- | ------------ |
| `contracts/common/swappers/OrigamiDexAggregatorSwapperAutoCompounder.sol` | 42    | <<<<<<       |
| `contracts/investments/OrigamiDelegated4626Vault.sol`                     | 72    | <<<<<<       |
| `contracts/common/OrigamiErc4626.sol`                                     | 282   |
| `contracts/investments/infrared/OrigamiInfraredVaultManager.sol`          | 212   | <<<<<<       |



## Protocol Overview

Users deposit assets (iBGT) into the oriBGT vault and receive oriBGT shares in exchange. The iBGT tokens are staked into an Infrared vault in exchange for rewards, that are initially given in the form of HONEY, but can eventually be given in iBGT. 

The reward tokens are sent to the swapper, that swaps them for iBGT to be staked again in the Infrarred Vault. 


## Architecture high level review

The architecture was generally well structured and the code was clean. 

# Findings

## Critical risk

No issues were found where the users' collateral was at risk. 

## High risk

### [H-1] An attacker can steal up to 33% of the vault yield when `rewardToken == _asset`

The issue is described in depth here:

https://gist.github.com/JacoboLansac/d46ac65eda28ff423b119ffbfbe28b9f

## Medium risk

### [M-1] Withdrawals will revert with underflow when the amount to withdraw is smaller than the manager's balance

The issue is described in depth here:

https://gist.github.com/JacoboLansac/9e54de0b8bdb663af5951afa6980f6c1

### [M-2] An attacker can bypass the performance fees and distribute them among vault shareholders if `rewardToken == _asset`

The issue is described in depth here:

https://gist.github.com/JacoboLansac/b3f4432f31ff109f45a216a9100f3d8b

