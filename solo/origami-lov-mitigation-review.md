# Origami Lov tokens mitigation review

As I ranked **#1** in the contest, the protocol asked me for an extension of the audit period after the contest, as well as a mitigation review.

## Scope
The scope is the same as in the [Origami contest](https://app.hats.finance/audit-competitions/origami-0x998f1b716a5022be026ca6b919c0ddf45ca31abd/scope)

Dates: `2024-03-07` to `2024-03-14`

## Findings

The codebase had been already audited by an audit firm, as well as during the 2 weeks of audit contest at hats. 
Only 3 new low-risk vulnerabilities were found.

### Low
- [Low - Mint overflow in debtToken](./origami-after-hats/low_debtToken-mint-overflow.md)
- [Low - Rounding direction in inverseSubtractBps function](./origami-after-hats/low_inverseSubtractBps-rounds-up-in-maxExit.md)
- [Low - Swapper contract balance can be withdrawn by an attacker](./origami-after-hats/low_swapper-tokens-can-be-withdrawn.md)

### Informational
- [Info - multiple informational issues](./origami-after-hats/informational-issues.md)
