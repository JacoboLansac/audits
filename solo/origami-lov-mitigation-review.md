# Origami Lov tokens mitigation review

As I ranked #1 in the contest, the protocol asked me for an extension of the audit period after the contest, as well as a mitigation review.
The following issues were found:

- [Low - Mint overflow in debtToken](./origami-after-hats/low_debtToken-mint-overflow.md)
- [Low - Rounding direction in inverseSubtractBps function](./origami-after-hats/low_inverseSubtractBps-rounds-up-in-maxExit.md)
- [Low - Swapper contract balance can be withdrawn by an attacker](./origami-after-hats/low_swapper-tokens-can-be-withdrawn.md)
- [Info - multiple informational issues](./origami-after-hats/informational-issues.md)
