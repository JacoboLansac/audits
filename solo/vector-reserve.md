# Vector Reserve security review

A time-boxed security review of the [**Vector Reserve**](https://twitter.com/vectorreserve) protocol, with a focus on
smart contract security.

Author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.
Read [past security review](https://github.com/JacoboLansac/audits?tab=readme-ov-file).

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

Here is an attempt of schetching the architecture of the protocol in terms of contract calls and tokens flows:
- [Architecture diagram](https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=vectorresserve-architecture.drawio#R7R1Jl5s4%2BrfMod6rPtiPfTlWakneTJLul2S6p3PDIJfpwkAAV5X7148kJBZJYLDB4ErnkLKFBPjTt2%2B6Um%2B3r%2B8TJ958ijwQXCmS93ql3l0pim4qEvyDRvb5iKrYWj7ymPhePiaXA1%2F9vwEZJAsfd74H0trELIqCzI%2Frg24UhsDNamNOkkQv9WnrKKg%2FNXYeATfw1XUCfvQP38s2%2BailS%2BX4B%2BA%2FbuiTZYlc2Tp0MhlIN44XvVSG1Psr9TaJoiz%2FtH29BQGCHoVLvu6h4WrxYgkIsy4Lbt8%2FB%2FH3u%2B1H6Vm9%2F%2FUhlcPP%2BoJsxrMT7MgPJi%2Bb7SkEHpNoF1%2Bp79Z%2BENxGQZTgYdVzgLV24XiaJdETqFwxXAus1vAK%2F4bkpZ9BkoFX0f45K%2FrYEkQQuUC0BVmyh%2FPIKpPiB8Urk3x%2FKTdJ1i0yuKnuULEhDkGNx%2BLuJfTgBwLAHsBUDgMTwjL0ALqJBAH0svEz8DV2XHT1BVIQHNtkW%2FjQO1kIv9Y9ZIHKA48CS1tqXaElW%2FrSGAleqgBeRpAhXIvgL64Czvixi%2BiFRYq5xA2coErxKwYUvQ4%2FPaK%2Fv0NcgvioSF9ACpJngOCYRFnkRgF9Bnzl%2FDH5Cm6rIDCz%2Bn7UMT2MQsCQhQfWzg7f3An8xxCOuHDrALz0Du2OD9nKDbmw9T0PPUmIA3UsGRwNKA2pdRoqvlewQtGFJDQWBckdSCjnR6fBpD%2B%2FsZe2bWimpFuWoupqnfloPOBUU16a3IIqFBVtLChawwKx2JTjoTgTwNiNDGdF2cD9awx%2FEKQ9RYJE7IQuUj5uKjxjVTIMsjShIwt07%2FtvH%2BCf65cNCOHfgg%2Fh%2F%2FJv76LQ88NHpBWBOEr9LIUf%2F8jXRSEGNJpWGViBjROs0Tf0X5YAJ90l%2B1%2FEb5BBHhX%2Blvih68cOAsk6ibbVVyme3yib5MOyidEG1mtguEJtwDPtldSDjVkNyNYszY4iPnkAHHv6Ft%2Bvrdcvbvy6ePqR%2FK1%2FU78sKGlUAAs8qFKSr1GSbaLHKHSC%2B3KUYfjlnI9RFBOA%2FwWybE%2F0Y2eXRfXtAK9%2B9j%2ByHH3%2BE31e6uTb3Wvl0t2efgnh7%2F0fvQH6kq9SdPq9XIe%2F0YX9NcH8ClWgtd7YkEa7xAWHKTtzkkfQdj8yD%2B1IK24lIHAy%2F7luBIhwBS%2B9SRJnX5kQR36IqLq4829ooJQmjPIqq3oV7Q5Nt1QGSfPHlyhb%2FI4TsJg3DLY%2BVmUg03dC%2FD%2B6i%2BFsEWMIV2mMN7VgTks0%2FVrAoorbbBzE97IN1s2iFzwG0RpaTvmP30YJuvT7%2FS1CgF0cB3shZX10VtDorFFDd%2B0rAVCZJOILoTbZOXhz%2Fd2VfidE1FayP13VkJaSruh1BMm%2F9UPKEo3olGi9TiF1sBxuCHThNYwzML23wkNOU5kp4pWUmusP3yglpsjmOWjcDGmx1AVBaRWdqnA20oysQaqRVF2xNVlGkp9TCWTJXFoWrwioo5kyIut2DqaM0WbKGPOyZOTO%2FqmOmKWeDMW5QEYXQKawZQSek1WUeCBZuDlR3uClyfViUR3%2FRehLuY2g9ue4GTZYUjfx48yHZklpFZXCXWQnfc2c0HMSZFH9Guy38Q6Jfc%2BHPMJfQZaeLHktgr3tZ6QWgFdoLGHDDPIukITItnmId6vAd5GRswtd9Fopv%2FqOPsvHxpSfYrMKaROSh3w40L7aF5oIVDfgCz9B%2ByjnmwgvyK9v%2BrGCEYE1uTrGmsytQtEVAF834WhhHiac3N8jOZkNJ%2F4BBkdcX8ALxOGU4BJC28sTq03Mr1msKvy%2ByCKP4HhS1JypFG11CBr2zMTo0B5B82QwzgUyrS7B84rRJnEw6Fsg2bFLQZLi%2BzpPhYk94vOg%2FMfmkAt8HIlJc5sesscbOd9U%2BN8VjtBEpZcTi2PyiiO93AXfWKBhYMdvP01j8K1mUKm8QPb8mjpypBTkvomsvOiHfuZDgfm3gyjjl7lqNxfkoRaHqSWO57Fa70WpNU3C6JK8BQrvxZmHnqNbbYqOIOtiSmlOgTYUFAdw6p4fMn8Gz78%2B%2FPV5E35IQfT0%2Bfv21XUWPHoRL3vOebHoSDlYnTNwRcJOJHZFL4kjV123j%2B7%2BIc%2BvQvZi4LAQF8iBJlhrHoHEbH%2FuiiY3UU7x2rf%2B7J9G%2Bb1xXRBnpbpZxshx4MlNgJNhf1AUeikmXki7CNH20Q4v88OcZDr4rIoIPHZ3oduHUZ7ThEOF%2BE65jlt6mG7zMRxKxMGvDVSdA6zc4fmeD9WYJFecXpx9%2BtPrr4ziKa1xgkOxX2gIxxvR7UkaBA1QXpHcijIHY5baZiHSLlbb5F34tTSYy1M4G7WCS1I41bmGp2TJbpORAs%2Bapgg8kuMpmerAMSl10JjUuYExozDU2ZwcDwAsonDxLXHCdE3yVrDoGvGZlVRB8AqFYyVnkAi2NQC5ioJ1mTwP8faq8LulxSAUsO7TBasOM9dJBn03Eua7pbutPqCX2SEi%2BLHz3SfsSoNsAjMPb1Ad5qgCj3aO2VlgLmiQqVmn0UwRoxtNjVH5kGCuxlye%2FtIocC5Jf9Fm6zCT29WXeXnMtIE9ZtoleszEv%2BRnc45U83Q%2BfMKKw8pJ80IFHIYr5OGSN8G%2F0rBOHusrxCXc9o0TPiJfBqmIoBIVp%2BHgR6AbN94vTzYm6Tp%2BiFw0KejgfvkptYWyAGWWekDBbC7Vl6Hxph8KXeYqAIuBBc1g19QlKQiNTPyiFASRZSqUbAPXWx7h8TDr2q4uSLvK0f6sApDXeE9TDfSTATYtPESpaIwsOFqS88k5xOCnhp%2Fy8PHrN0Fyqyi%2Fhpj5OGSQgGyXhO3y8Zh01pOc9c1prfhHYpmHCnAUablc4s93cxVpRgNS9xdpZxVkfELg77ToVCzNhGrFvKVYE785IMVkUzcVWZct29LFYsy0DFXTVEk1bVOgf4wm0qiYqOxarkBf87lSZwzd08%2B9ak6rZaZ%2FVi821Jz2E8oHI%2F407%2FNgsZc2eGrAaaX9fL4qLbScCwIoR2KAfF4M6IoANMIxEwSwebdEUUsDmuptU2qFHLKacegAzkzzZMDLQyqmlN08VMo%2BFVuRB8cqvLRvIXphadB8PlOpYujB%2BQpRJIaqRVe%2Ff%2F7x5a%2F9983v7vdPSXwfpy%2B3gtw1EhXh9dUJ0LVkZ2YvgXZqEwWlN%2FZOhZQcFqlM46GFzCa%2B5YQ2ROKbEKMEXm%2FW3BI1XkmfhXYLGW4MtM6Fh3aVy3UWOnYzkP54fJgL612VO7MjwhetGXRNIjcfmTEbdRKR7fYWIex8Q9evGhnzOTtDiJxIOQGlMXbt8%2F7viuM7eVxdS3n4m%2F7JPd8SbgS3drZ%2BsM%2BnfgDBM0CmZuV62ShOlkmjOHIhfyi6EkbJ1gkq156dxHfgX2iwOtkuIa6N5nmuEzdNeSFWIbqokaw%2FKYCkC5IF%2FPkuUrG4lVESb5yQ3DKnDgkZ2gtiW6Phwrym13zIMELyJNoTL7%2BS4SwVeH%2F6JNy1LrdqcXfMymNeosSrv1hxL%2FhbVk8%2BvB26Z07BC2IX1%2BYhlveIGdiC2UcFoSTawuqHXypv6gE3SnDpzyLb%2BO5TiLJc8HJSGCSeW9nL1nmV16nNWweRk7HAgTp1HDh7Oj3wcZ34v%2FxtDNmxExLXFhN%2BqYoHSCw5gg8c8pH%2BoY4Lpo4J8bwpjvkPG36LiPYzs2GSD5nUOjagTiPIPVKJq%2BTp%2FwLnCce7p9PgqVXZqSzqaFt0YP2bqtWHfWuTtdIS6qqCVlrc3k%2BUE6%2B0JZVZ1sE0CYVA%2BkxZ4cd37WjfmIvPKlMH7sShvplOHCof2Xjb%2BXbiYD%2FrYhq%2FGQd5ItvI8zrdOAl2ZmHfF5oliC7QVAPcTiQvTKw00%2BJFcK5xkDy%2BPLcwn5pve733R%2FECz3mRhb9Fw9f5Mx9S7pUurNxgtBv%2Fk5nYu0Lh0nt6aHxqwFecf3ggrSOddV5HP2WgRX2SllBHKv7RO1TzOkTVJEMkcoh%2FgiCIy8Qf2I1q6QP%2F9u2T8dK3Olo06uARhZEbjKumXTMRaMitscU4u4CWVzUv0FoXjNOWXNAPtlLPXwbgGtQU9lAG7vyF2RCS1EpI%2FelhLgkSnV0D4zTq59G4KChnCOVQD5b%2BFNlOL9wCTVdGpS9hm6DDckl8IAmhwA7WAc4wumDJVSefg0dlDEw%2BtGPPBHLoJPVU59XT1S4Jb0LvC%2FAA2IoS1%2FgRtKRMZWMuVrIvkHM4bwWd4yc%2BqaLocUVa78wABevpGeZx%2BRl9T2tZWy4Qt6Z5eEDKsRBpZR4%2FhYk1ujEv9q6q9YM0TNZ91ZBoNFKGhBAUBs90m85OyUjHiYck2tLjHeaU%2BnskCg%2BfYkTN1QCsxZZlZ4SeVl%2Fpq2UoTCWZpbenDfHz25US026d35RmRKmR6epgj5f2J6Y0QfkmcpKISG0XNl6aX83FmxAdVkdKk88kO0wGW2VFUjuZBjyV2cxxTkUP%2FXMhPh91oqLkkJgRXXfiOImehbRBK%2FRvgiB6QZ5m0SQPCCZdHC0te1WvnJWUpituajB%2BNUZuyIZ9JC3JkswQk3lmKWI2GsliYdFk3%2Bxiz8nAXfNSfuQP2tTrUB0PKqO9%2FgLSrOgCtlwuJyWyMv38sHuryVk1J3FF862n8gjzhGHbBkNjHV1ZvLwy2PMHVVoKe4DGeuurxaHj9EkSe4o9u0Jm%2FGLsipMdY2KiPz6jZbCDIRWD0dYFh7Vpiro0jbwHB4pDioKQ5gBhLTGQBk5uMQdNbpkWNKLsFgY08wiFm71D4Yq8VPhK%2BaamBrJmIR1rHDBbA2fwNQKj%2F6mLB0lXVVUxwKyxkNJShoWWdTqrmxYeHXoiz6MreLFz3YlUWQpotAnEurkcjUR5J0yUl%2FMWjfD5XuzTJKRUt693%2B5FGYmhOU1H0pWHLimFbpqrJpsmnqdjG0tJlQ7dNzbBUnfYjq8kwZbyt46M4919uaUn45Nt1qlbRuDG4ITnZE9QZRtDdDB%2Bqa1qaBe1TxdAFjc4qUnL4jeG1UxQrExmVaJz6deZiDXbOGurXMuas5iCZOEE7kQY7SVLq5uBCZpMPO7tcbIkxBy3UJ6n8pzC%2BnJEdMDbPho5x4zelK7Cj10UK5cQkY9f8lAfclPDLbyDxIawRu317bpXhS4eafCFSvdhGllj9s7tbRWOcHSprCA3mVmF9Qap1yK0iK60rxnGr2Lwy2MmXOgEZnk8KHSMkT6C4rilF5zokTVZtZWmrZSZ9HZW1Y8kP3pchio7JIIN1neGdE1NlnZ6QQNoX%2F7t0e%2BmqPJ2rQZHFOFoVpRum8DgnM0FkzeiGvX05PvfGB%2FK92fkqaWTaPF9tmz9O4y%2BNd5rOWkU7NpAsm4xGJkv2AarC31gFr5MQ6JyW3aqfja924WM0ShvDZLQTHV62RiElPsJE25M1Kk%2B6YbauGIc8dEE6U0OHsqIVPa18mLjaYYqUi4E6k3UlKWqgzIikuKiuzpgfw503K0ZYviW%2FuNhN3OqcHyGdPtBhn76HaoPzwmGBZU%2FxHrhRuk8zsL0MrYtHyAELeQRa17QIarP2sWwfqXfxprYmd8P13tLC1tjkPqul9d5gpNTcV6%2Fi%2FEpRWIXB83TjxBg9MUX0DmE1mq8rS9f0VkTr7nBXmKwQU3BqFt3gqo%2BdVa6PcbC3poG3gpsfwWcPD70FR4X6%2Bwej2ExveeotECWXvB2M51jf5PBu6cdy%2BfC22f6qhiiZ6pzwpjd%2Bm%2FDmG9oW3rfJAC7KFXozAJctNuV6aoYiqHN7S%2FBWzengffc%2B%2B0%2Fy49cXNbD%2BrTtB6oTuesF7B3DdLHzcO%2BoAOJRGvsY2U%2BD%2F2Pmen01crn1E%2FjjbZ2R4j0Ej5lT9A%2BJWFlec9SWeR%2FSe81dyC7GKN%2BG%2Fvb%2BfFjEGi4sz5XHHm9THbL4Q2pMdPiR8G97mTAF2K%2BLNnBIJBimHnMF%2BK7Pab97ofQap6Jj52WeOzW6j5ak2uu2tWzUzD8RBtL9QbwZ3cJEu0M6skbQzIciV%2BbDL7ge29UwcGzhIKWrC1obOMyEu3uyhfc9JJxspJYedCxDio7MCwdWRqdQJSP2%2FSckFAj3xZcOb6%2B%2FQkaqdWy4qDfTWuwoGIptEe23SvFRy82ODB%2BMfYSMffxDycLV%2FMmteClziiqiJpczmrgwnNYbu8t10lu5R5UNNwKhlLkujSdQOpZBnLyVq38XOpUSyXoehpHTERGmA%2Bgfxb%2BB91f8N%2FfQFAlGRfnP8hFdnMOd9EEafuY2aSzlLP6ppdlVJ8tJgmLCsLrWyNLbsXlzZU9NaUglcq2uxlvh8AbrcGEth4v3jjud9pL4pnC98wJ11GT10O%2Fu2Tj0%2Bd3i9a3BX1ZEJXHI9WZH2%2BGzMFlAYnYRZcHIIv80EfJvucMVkDS57Yne48rbBLTHgViYG95sO9igGk6CinTGYKWzcy2N3VTg2dRJ6wFrQxfkhxvDmdW63O2fJJ%2FNsQGmVfNwCmVRkDSX5rJvd94%2Fh84uZ3vxhxIt1bL%2BuoLLZnOvQeBwhc5KgRo%2B64w73cYKgSB69OnCSG6%2FZ1%2FkGOYmuymTI0JAWQAP7EeBsi07PciSeIRkChnTEERfwaxIhkJcoAX%2FU5lPkIU%2FV%2Ff8B)
- [Tokens flow](https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=vectorreserve-token-flows.drawio#R7R3ZkqM48lv2wRHVD3aIGx7r3JnYmtmOrtru7X2TQbbZwuABXMd8%2FUogYQ6BAQOWa6crom10gVN5KzOZKbfb97%2BHcLf5LXCQN5OB8z5T7mayLCsywB%2Bk5SNtUWRLTVvWoeukbdKh4cn9E9FGOnG9dx0UFQbGQeDF7q7YaAe%2Bj%2By40AbDMHgrDlsFXvGuO7hGlYYnG3rV1h%2BuE2%2FSVlMDh%2FZfkLvesDtLgPZsIRtMG6INdIK3XJNyP1NuwyCI02%2Fb91vkEegxuKTzHmp6swcLkR%2B3mWBe7%2F%2Fz6L%2B%2BGdH1D303X%2B2s9%2BWcrvIKvT39wfRh4w8GgXUY7Hcz5Wblet5t4AVh0qw4EJkrG7dHcRi8oFyPbptoucI91Sdkt0NhjN55%2BweX7LYHEGHkQsEWxeEHHkdnKZZCwUoRy2DXb4ddkjSTNm7yW5TtCKS4sc6WP4APf6EQ7ABN%2BTg0MTB9B5FFAIbQ28aN0dMO2qT3DZMQbtvEW3zTO4kLwMZNLEO1Cj1GhepCbQstydQW%2BkjwUjjw0r2YIFuAf3EecPof%2B4B1zKOETVzjAQrYvSeAYv3425p8fsfIhBFSBt9QhMJXROAYBnFgBx67B37k9DbpjMpWYWDGxf0oorof%2BKhEFw5awX2yOPTctY9bbLx1CHfdkN1xMV%2B5ph1b13HInbg4UMSSwdHgvUQzDTQka1wSGouC1Lb86DSQdOY3hrWwLF01gGaasqIpBbgpahVuiiEtjMqEPBBldSwgasMCUT0ZiILARa9lN0vGBO7fd%2FgHYcqTASZh6NtE97jOcYzlgV3QqSFrmZO1759%2FwR9Xbxvk48%2BMCyX%2FpVc3ge%2B4%2FpooRWgXRG4c4a8%2F0nmBnwCaDMs1LNEGeityRf6LQwSjffjxhf8EMeZQ%2FtfQ9W13BwlIVmGwzT9Kdv9aySQdl0wlZWC1QrrNVQYcw1qCDkxMq0G2elnWi%2FaksXCMLZxDsnQ%2Fn%2Bm2LSIigY6KmiHlR3FTDjJq8C1hvYq0sFRFw7q%2BJJGNKHJLjqIGjIVpVrdJGW2XWuu9E8sZvUnO6GKJGamFutsJiKdDURTIAJ4I7qbZSlqNZnv9%2B0%2Fc%2FfzPf9z%2F3laTRQ62aellEMabYB340Ls%2FtN5g43RHelceer8mZnOFiRzmPQbBjnKp%2F6I4%2FqBGO9zHQZGHoXc3%2FjedTr7%2FJN8XskYv795zfXcfHZVdhmvBPrRR00C68wQGjcwrRB6M3dei3c9DDTr1a%2BAmO0kxdS6rRdo1tALxmkpxxRiGaxTTRQ74hoEPP3LDdmRA1HBbw%2BLd5qHtcLWE7On9D6ifAegEaqg38w6qS5Mmlqg0RA%2Fqo5GlmhWvB8UbLFnFVIMy3nqxelDVkvuG3mDoRHcuho67xCrR5alCHbaF9cpFG5tjKko8E3s81Wdg63AaE1u3BNN9eLbkKVCsU7IvT%2FcxKpD5kRrDIVrCCF19ISsQmxdTsEf4%2By1h8RvioYu%2B35OLaL%2FbeYkQ9J0828dTNtDHYrxeeowlTzj62jIIHRTO7ZSXXCc%2FObyaz%2FPtX7gaHBE%2ByS%2FlSSUKg6sMCBFKBDiMD52u78YuZot%2FYoUl8L%2BIKsP0Gpy%2BGBlmVlAZw%2F8phi%2Buv75A4dV%2BP1ivChYACG3IW4JKM81sEmeGWNKM6SiDQdE6GYyiQIbn4xhJLIy%2F8NC2VSqNVolbeQc%2Fgj2xVEnTFhudyfLU%2Bcx81jPq0T54voWUXBk9XKrkkqsmd%2BHw4fKEV4ctaSW8eHbY1MJLFvW0UwJWk%2FTi2GIq25%2BJ2PLARqw86Bnn1MDodq55buuHut2ImEh8c8oDeZg9MW%2F%2B2Lv2S2L0YKgne%2BEMKiF6xSw1I2BrdjRnXp56iaEaPLwZT0hULfVUSFygdOh%2BRim8aSNXrU8xpIMmNQsHwWybgS3EbFsu3rZhmNskNzq52kIBpc0hKEdMOVJnKV%2BKpaFUIymIkzIVIWW8iGL4kmBFYjhelICpo%2FquAuacEkXhOTO4rHDgCNQeBohR1JY0uQ7tJ%2BWYvJPsE2RJtiEDypJJ4cEzYCc%2Fx398ek6XAjMNTwaLxSL5ficoz8%2Bw6HSePymnr9rc31mkKp%2Fdc%2BWu0Gy%2Bw9YU2LxkaIYsaZJpmcWonzpDwjB1RVUVoBiWwZHX44kAnjY8UGAa3u37b8OFpJ0efuZjiCXxZwtDY9c%2F6XrJxSECLbliIWjdKT%2FtYQliWmeJmQaDNY2bKIpNUYvxYfMsOIWtkQbcnRy3VnujUUPR1HqT52T0zwsoEbCfBl%2By72nwZXPs5YFiCuSy0IQjmNZhn0cpi%2BkwRymLIi6GhsrCe7sRW1cS0fSiDixZWp5Cjo7XNa2eog6z2eMEq1WE4tKMYaiOF20%2FENWxhaId9LkL5XwG4Xp5BVLPM%2FtInQYgudsKbl3vIx36C%2FJeEVFCcv10bdLtB%2BEWerm%2BVxi6EH9ijQXG%2B5DqtvXjbLirG%2FJG9QDSqdJTS%2BBhWkfhHP9Km%2BTuVGYG4W4DfbqknLYRTWtOlSvSnOlXrM%2FFHMand2J5lGlPHOLFVnh9dqck0zHVY5KU6txt3oLQKT5Ythb%2BLcsXFy9H1kxJfk41ocK4JbRf1gnHm5e2SyY4THYq%2F%2BVL7kkdZAdhEgY1jzeu%2FeKjiD4eDZLij83tZeO43OMUxq28AMZl4DhutPPgBxvuuT4Javubu91h%2Fg39uBGBT6aE1yb5kzWndHLMlSeC%2FOqrvVWEznJlAGkSLU3VptHSJEUqsnpTb6elDcbRW3uSJnccyU2HE6Z51HvKPLLTeFDUgT1K2cZc%2FOmEOnBwhFpn2l8eZOqTy5ZcCdL%2F9OQWC4EQ2oT1OCiyQ3dHpOI5znD2EQqjWRY5NmOnOiPeMo0%2BD5GN3KR%2BRETvCLfE7%2BQvo10q9q%2BiDQyT8PQkP5uM4uRm08zwJPKdhnQvCVKlke9hITdmn2Rqp5HeNgmdS%2FPF06HptrNk8DRuPnuAVxJaAWJ3S5qv0ns%2BRJVHurC4xdEW%2FuvAsqvzWq2LlLqUA0u1Gi31lBxLHnFmR5fnza5VBhrUJ7DAOlL2j6Vk593XvCCl0fzV1oj%2Bam4KDmXEYptGLK%2F6kEv9M9d1zLdHZ%2FX0husAWpLBZSX67S04oO2Izj29pT3GctGGs8dOrDoxIjYL6n4%2BuJxHRNEqwlWR0zGdpTM8KsptcbG1n3kiXJRHdMo2uqLOiIszpX0digwZpQIm5pxR%2FZEx80gNjIwMx44ioy4YLo5YLuVT4WLZFyodwcWS7IZAk2WHh5E2VutyPSPK7tY4yvw9wiBpfbDTkKqoSDhKsY2haSOOtmB7mm4j2Sricyd0PpMaOTwm8t36uln06itlI7imZtBghXq4KQcDofjXb7%2F%2Bfvvr1%2BvHGS1e9SQixhe4stGaK59iUCFNUnNmU67nRiY2%2BBRMmWH49IpDv8iI0vGXrBl5ejg6npUlrxsv63rT%2BHFik%2FT6NL7PZhx2904OaBpWSUe0IDwVWEV8zSpzHznera6klOLozXLl5oHK0FUf2WomscoEA%2BjHJpRraBQmjEOU1ohEeRFKX1uPTUcjeQinYoRxN6bFKu9sD0aRa7PmB9erL6LfVioabaXi8H7GfjG1RommJB0Y49MIA1NjfUdyfltBa1Z1FCalqTvX%2BzBtxK%2F3sTQ1VeMbDJ2L5CtGOQxT5VTvUTmHIuWYnAHLy47IlUKUZuk9Pj0LyZ16xTN39eGVMM3QdWBCHqapinynGaewJ%2FLAX1HoYtQgx4U1dm5rltXWuyKaOxqMaHoKpv3WHN%2Fhy2Zk5iBKHYIf91G3ws0qUpfltmroJl9u35r3Nw%2F%2Fn7jMLbH6yd3ZvY6iO2PtEXRcAagALp%2B%2BuTNATQWwwdFx%2BCjgk9CRPffnVM80s6zzqtKZ1TODU1cnKZQ3k0m4IDeIsLwfaU09z%2F1j7zpu%2FHFe0m5%2FCDDUQVW9R2jpLNNz%2FDPYg2ZbR5Ep1vmqMaJuxQ2iev77vZAyyupiOzSpW13dHV0OxybAT0sSCz9HD%2FI7wm0jREJ%2FhUPYoqnb%2BjTqwnETCIWbzPM%2BGe%2F8jqJYSGzsF8l32choWmIhY33M3gGBHLTzgo%2FB1flB3%2Fdbr84baunAiuNs5b2ndjRt3pTFEQHtw8Q6WtbJVVnbmYYWWwfQmmIpLSYvc3NKpeVXmo8uZ%2B9LaS8yHuESebOeeSshwj%2BBJksSPKHHUnhx7YZU3WodncJI6%2FS0X0waQDELjIO9jLvvWfz4ZTHM%2Fhmuw9VyVUruE5XzkkmZl%2FIz3jugzYGLe2dwHiTztw4YC5D%2FNxrXafHeKTHeoWB2fpOvpBVhCOSWmMjK8QwP7aon71%2B%2BG71hIMrgK3RDjvpOWPcDt1ZhZaOETh6spZpaXiIBaaGXmLCkLGj502KuZ25PDXPBSgsWMkDNhZyrnsqOvIff47GdYzeinth0CO6uDb875tdtoUPe3JG%2F2QT%2B2%2FaqpiBhropSDHnTteagOk3VzKYJ48T%2FmPXuu09wwKSXAxUtTiniSc%2BXrHqX1CeAt2aVQ1bPHW5ltfG6XDC8S%2FFtxtnhLdfCe6BQIL7nFTrO4%2BHAVTiB3dMV1MsB2%2B7A9QRJLF%2BWJFYzpp8VEpMbJXFlgqZN8KZ16%2FQURRXUUM4zrYvz4JECkC3Jo2rhFPGL1k7MMz7aNKQlNACXVI2SVFI4ZW241ul4pVGqxulgbDISNDehwAKnif0FwIT89DwANONcqQks4%2BAoA2XB%2BedOTTBL9CNpWnM1Y84M0Gz9KCZonjEO05VA%2FzdMDefH1cphcBLnPVwTO3IliWeptIPMEVB%2FAleuJPHsihJ0xPDlHjbygp25klS1K36DPlxzXjIptGe2ngo6uWY5b8CtccVq6oId1Y2wKyOeo27Z9larvAmZ5DRi1Ix6R%2F4GUE2OJ0MyvjZ2mrRWTm6WsjcTT1QGW5JGrAkjaKZH34iUU0pktMjrGNI70bpEEUutECQkRZJGzAuNLgIf25dsOSk%2FASm2WVOzBVja4AjJMoouDiGZRstByNoXtvQpKMRJQSiXGCq%2B%2BGJ6fO1e9WfAmie90vT646vU%2BqSVIcjoJVX0sj%2FWaKcqcFYCpXdKtKzS1dnnbAD%2BI9c%2FmaQ0TRjJ%2FSGPqMDzX%2FopkBQ6qZJRn%2Ffa8Km6Q8mWT0XVRqlinlamxdZUbWjFlYzyew8Ho%2BrKwdARqjZYDXT%2BhLGoul6V%2FASH3qreIux12pou8ojHOYKe5vRLLO4Rf5bDImXlAGU5sH7O%2BN1x34gsVmawxH2j%2BV9lV1rnAacpxT0Lr9Cj7rzrjS%2Fou1dd4Sb01CL4GexKvrDTtZJAl1TV6CfRdaDJpaVGUtR1WdPKD202PxtnxhRCnTm2%2Fqobdil1w2pYRKeiYQJRt1omSb10rtSeuuUyoyirZr2LZePLMCC4fRiOtdbNb4FDcubu%2Fwc%3D)

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
