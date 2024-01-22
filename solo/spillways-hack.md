# Report description

A research study on a "hack" experienced by the **Spillways** protocol. The research was published on twitter [here](https://twitter.com/jacolansac/status/1731137430676242640)

The protocol described the hack as follows:
- An address had been selling large amounts of tokens without purchasing them first
- The address was not part of the team wallets

Author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time and
resource-bound effort to find as many vulnerabilities as possible, but there is no guarantee that all issues will be
found.
A security researcher holds no
responsibility for the findings provided in this document. A security review is not an endorsement of the underlying
business or product and can never be used as a guarantee that the protocol is bug-free. This security review is focused
solely on the security aspects of the Solidity implementation of the contracts.

# Protocol Summary

A simple staking contract

# Scope of the Audit

The scope was an **unverified** staking contract, which had been presumably exploited.
This made the assessment more difficult obviously

# The research

## The synthoms

- `0x0899d57e77d6d0eca2d31cb9840abd0228fe0387` had been selling large amounts of spillway tokens
- The addres had not purchased those tokens before. Where are they comming from?
- the Spillways staking contract has become insolvent, people cannot unstake their staking balances, because the tokens are not in the contract's balance

## The staking contract

The unverfied contract can be read here:
- https://etherscan.io/address/0xd19d6ad86f6c4d94184c19b99cc64432974faccf#code

The decompiled version using dedaub tool:
- https://app.dedaub.com/ethereum/address/0xd19d6ad86f6c4d94184c19b99cc64432974faccf/decompiled

## The hack

The hack was pretty simple. The deployer of the staking contract did so maliciously, by introducing some
nasty logic on the `stake()` function. This logic would increase artificially the staking balance of `0x0899d57e77d6d0eca2d31cb9840abd0228fe0387`
which would allow him to withdraw tokens that didn't belong to him. More specifically, for every `amount` a user staked in the contract,
- the users staking balance was incremented by `amount`
- the malicious address (hardcoded in the contract) was increased with 25% of `amount`

This created protocol insolvency, as for every 100 tokens a user staked, the malicious deployer could 
withdraw 25, which would lead to the contract with only 75 real tokes in its balance. 

In practical terms, 25% of the staked tokens were stolen by the attacker. 

## Conclusion

There was no "exploit" in the sense that there wasn't any bug that an arbitrary attacker could exploit. 
The deployer was basically malicious, and deployed a contract that would allow him to steal tokens from stakers.

# Updates

The spillways team asked me to perform a security review of the new staking contract. 
The audit of the new contract can be found here: [Spillways staking](./spillways-staking.md)