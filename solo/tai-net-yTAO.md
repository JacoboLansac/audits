# TaiNet - Security review

A time-boxed security review of the [**TaiNet**](https://tainet.finance/) protocol, with a focus on smart contract security and gas optimizations.

Author: [**Jacopod**](https://twitter.com/jacolansac), independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits).

## Findings Overview

TODO: TBC

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

- The contract is upgradeable, following the transparent proxy design pattern (TODO: review)
- The architecture requires a high level of activity from the protocol admins to function. This leads to the following points:
    - Inactivity from admins means that users' funds can be stuck in the contract until admins reactivate activity. 
    - The protocol owners will have very large gas costs related to operating the contract. Some of these gas costs are passed to the users as staking/unstaking/bridging fees

# Thread model

## Actors
- Users: can `wrap()`, `requestUnstake()` and `unstake()`. 
- Admin roles: can configure the contract, upgrade it, update the exchange rate, and approve unstake requests. 

## Worst-case scenarios
- Users losing all staked funds
- Users can't unstake their tokens
- Users can't request more unstakes
- Attacker realizes the same unstake request twice
- Griefing: Protocol loses value due to staking/unstaking costs or other mechanisms


# Findings

## High risk

### [H-] The exchange rate mechanism either makes the protocol vulnerable to sandwich attacks or dooms it to spend a huge amount of gas in keeping the rate updated

This is only profitable if the % delta in the exchange rate is higher than the unstaking Fee. However, note that the protocol needs to spend gas fees in every update of the exchange rate. This leads me to think that the exchange rate will most certainly deviate from the free market rate in DEXes. 

With large deviations from the free-market rate, wealthy attackers will be able to leech value from other honest depositors. 

#### Impact: Medium

- Probability: high, as the gains from the attacker are at the expense of the protocol, who will face those costs from their treasury
- Impact: high, as it only requires a certain level of volatility and unaware admins approving the unstakes.  

#### Proof of concept

- The exchange rate is measured in units of [yTAO/wTAO], so how many yTAO per 1 wTAO. 
- Let's assume that the unstaking fee is set to 10% of the unstaked value. 

- Let's assume the exchange rate in the market fluctuates rapidly, leading to a -15% with respect to the latest update of the exchange rate (so less yTAO per each wTAO).
- The automated bot in charge of keeping the exchange rate up to date will call `updateExchangeRate()` with the new price. 
- The attacker frontruns the exchange rate update by calling `wrap()`. By front-running he gets the amount of yTAO corresponding to the original exchange rate, therefore receiving more yTAO than he should.
- The attacker backruns the tx calling `requestUnstake()`, which uses the fresh exchange rate to calculate the amount of wTAO. This will be +15% wTAO - 10% fees, so a net +5% in wTAO value

#### Recommendation

It is true, that in order for it to be effective, the admin has to approve the unstake request, but this requires:
1. That the admins are aware of this leeching attack
2. They put in place the tools to filter out sandwich attackers... but... is it really fair to do this? that would lock their tokens, which is almost like stealing them. Not a very good solution either. 

Here are some alternatives, all of them with pros and cons:

1. Impose a short commitment period of a few days after wrapping. Example: users cannot request to unstake until 2 days have passed since the last time they called `wrap()`. However, users could wrap and transfer to another wallet.
2. Impose a restriction that a user cannot transfer yTAO tokens in the same block as they call `wrap()`. This can be implemented with an ERC20 hook such as `_beforeTokenTransfer()`. Although this would effectively kill the issue, it might lead to issues with protocols trying to integrate with yTAO. 
