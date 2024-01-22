# Introduction

A time-boxed security review of the **Curves** protocol, with a focus on smart contract security.

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

Summary
The Curves protocol, an extension of friend.tech, introduces several innovative features. For context on friend.tech,
consider this insightful article: Friend Tech Smart Contract Breakdown. Key enhancements in the Curves protocol include:

- Token Export to ERC20: This pivotal feature allows users to transfer their tokens from the Curves protocol to the
  ERC20 format. Such interoperability significantly expands usability across various platforms. Within Curves, tokens
  lack decimal places, but when converted to ERC20, they adopt a standard 18-decimal format. Importantly, users can
  seamlessly reintegrate their ERC20 tokens into the Curves ecosystem, albeit only as whole, integer units.
- Referral Fee Implementation: Curves empowers protocols built upon its framework by enabling them to earn a percentage
  of all user transaction fees. This incentive mechanism benefits both the base protocol and its derivative platforms.
- Presale Feature: Learning from the pitfalls of friend.tech, particularly issues with frontrunners during token
  launches, Curves incorporates a presale phase. This allows creators to manage and stabilize their tokens prior to
  public trading, ensuring a more controlled and equitable distribution.
- Token Holder Fee: To encourage long-term holding over short-term trading, Curves introduces a fee distribution model
  that rewards token holders. This fee is proportionally divided among all token holders, incentivizing sustained
  investment in the ecosystem.
- These additions by Curves not only enhance functionality but also foster a more robust and inclusive financial
  ecosystem.

# Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
|:-------------------|:------------:|:--------------:|:-----------:|
| Likelihood: High   |     High     |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

## Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the
  attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no
  incentive.

## Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the
  protocol is affected.
- **Low** - can lead to unexpected behavior with some of the protocol's functionalities that are not so critical.

## Actions required by severity level

- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

# Scope

| Contract	                         | SLOC  | 	Libraries used |
|-----------------------------------|-------|-----------------|
| contracts/Curves.sol              | 	413	 | 	openzeppelin   |
| contracts/FeeSplitter.sol	        | 95    | 		openzeppelin  |
| contracts/Security.sol	           | 23    |                 |		
| contracts/CurvesERC20.sol	        | 14    | 		openzeppelin  |
| contracts/CurvesERC20Factory.sol	 | 8     |                 |		

- Number of Solidity Lines Of Code (nSLOC): 554
- Audited commit
  hash: [0e007d84608eeeca2b26d7f2ed62f15fcf50221c](https://github.com/code-423n4/2024-01-curves/commit/0e007d84608eeeca2b26d7f2ed62f15fcf50221c)
- Audited as a warden from [Code4rena](https://code4rena.com/audits/2023-10-nextgen)

## Inside scope

Files and contracts in scope for this audit in the table below:

| Contract                           | 	SLOC | 	Purpose..................................................................................                                                                                                                                                                                                               | 	Libraries and Interfaces used                                                                     |
|------------------------------------|-------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| smart-contracts/NextGenCore.sol    | 	366  | 	Core is the contract where the ERC721 tokens are minted and includes all the core functions of the ERC721 standard as well as additional setter & getter functions.                                                                                                                                     | 	ERC721Enumerable, Ownable, Strings, Base64, ERC2981, IRandomizer, INextGenAdmins, IMinterContract |
| smart-contracts/MinterContract.sol | 	475  | 	The Minter contract is used to mint an ERC721 token for a collection on the Core contract based on certain requirements that are set prior to the minting process.                                                                                                                                      | 	INextGenCore, Ownable, IDelegationManagementContract, MerkleProof, INextGenAdmins, IERC721        |
| smart-contracts/NextGenAdmins.sol  | 	61   | 	The Admin contract is responsible for adding or removing global or function-based admins who are allowed to call certain functions in both the Core and Minter contracts.	                                                                                                                              | Ownable                                                                                            |
| smart-contracts/RandomizerNXT.sol  | 	51   | 	The RandomizerNXT contract is responsible for generating a random hash for each token during the minting process using the NextGen's proposed approach.	                                                                                                                                                | IXRandoms, INextGenAdmins, Ownable, INextGenCore                                                   |
| smart-contracts/RandomizerVRF.sol  | 	87   | 	The RandomizerVRF contract is responsible for generating a random hash for each token during the minting process using the Chainlink's VRF service.	                                                                                                                                                    | VRFCoordinatorV2Interface, VRFConsumerBaseV2, Ownable, INextGenCore, INextGenAdmins                |
| smart-contracts/RandomizerRNG.sol  | 	72   | 	The RandomizerRNG contract is responsible for generating a random hash for each token during the minting process using the ARRng.io service.	                                                                                                                                                           | ArrngConsumer, Ownable, INextGenCore, INextGenAdmins                                               |
| smart-contracts/XRandoms.sol       | 	39   | 	The randomPool smart contract is used by the RandomizerNXT contract, once it's called from the RandomizerNXT smart contract it returns a random word from the current word pool as well as a random number back to the RandomizerNXT smart contract which uses those values to generate a random hash.	 | Ownable                                                                                            |
| smart-contracts/AuctionDemo.sol    | 	114  | 	The auctionDemo smart contract holds the current auctions after the mintAndAuction functionality is called. Users can bid on a token and the highest bidder can claim the token after an auction finishes.	                                                                                             | IMinterContract, IERC721, INextGenAdmins, Ownable                                                  |

# Executive Summary

## Main Findings Summary

| Finding Id | Description                                                                                                                                                                   | Severity |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|
| \[H-1\]    | Missing Access Control in FeeSplitter::setCurves()                                                                                                                            | High     |
| \[H-2\]    | FeeSplitter insolvency due non-accounted changes in supply with Curves::deposit() and Curves::withdraw()                                                                      | High     |
| \[H-3\]    | An attacker can flashloan ERC20 curves tokens and steal rewards from other holders                                                                                            | High     |
| \[H-4\]    | Malicious tokenSubject can DOS all holders and impede them to sell the tokens by setting a malicious referralFeeDestination that does not accept ether                        | High     |
| \[M-1\]    | Holders will lose unclaimed fees when selling Curves token due to bad accounting on onBalanceChange()                                                                         | Medium   |
| \[M-2\]    | The function `onBalanceChange()` is not called in several operations that do change an accounts balance without changing totalSupply                                          | Medium   |
| \[M-3\]    | Lack of input validation on Curves::setMaxFeePercent() can make getSellPriceAfterFee() revert if fees are wrongly configured, with max above 1 ether                          | Medium   |
| \[M-4\]    | Incorrect Referral Fee Implementation                                                                                                                                         | Medium   |
| \[M-5\]    | A malicious agent can frontrun buyCurvesToken() and cause it to revert                                                                                                        | Medium   |
| \[L-2\]    | Curves::transferAllCurvesTokens() unbounded gas loop, because tokens are not removed from ownedCurvesTokenSubjects array. Consider removing them when selling or transferring | Low      |
| \[L-3\]    | Duplicatd elements inside FeeSplitter::userTokens mapping as FeeSplitter::onBalanceChange() does not check if elements are present in the array before adding                 | Low      |

# Detailed findings

## High

### [H-1] Missing Access Control in FeeSplitter::setCurves()

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L35

#### Impact

An attacker can steal the vast majority of the ether balance sitting in the  `FeeSplitter` contract from every other
holder, defeating the purpose of the `FeeSplitter` contract.

#### Proof of Concept

A malicious actor calls `FeeSplitter::setCurves(_curves)`, passing the address of a malicious implementation of the
new `curves` contract. This malicious implementation gives wrong values of `totalSupply()` and `balanceOf()` in favor of
the attacker, which will result in a manipulated value of the claimable fees of the attacker when calling `claimFees()`.

#### Recommended Mitigation Steps

Add an access control modifier to the `FeeSplitter::setCurves()` function:

```diff
-   function setCurves(Curves curves_) public {
+   function setCurves(Curves curves_) public onlyManager {
    curves = curves_;
}
```

### [H-2] FeeSplitter insolvency due non-accounted changes in supply with Curves::deposit() and Curves::withdraw()

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L465
- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L490

The function `FeeSplitter::addFees()` is meant to distribute the `msg.value` among the holders of a particular curves
token. Note that the tokens that have been withdrawn as external ERC20s are excluded from this distribution, as they are
not included
in [`FeeSplitter::totalSupply()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L43).

This fee distribution is tracked by a global storage variable `cumulativeFeePerToken` and a pre-user
variable: `userFeeOffset`. The difference between these two represents the owed "fees per token" for a user.
When `FeeSplitter::addFees()` is called, the distribution is made using the current `totalSupply`, and
updating `cumulativeFeePerToken` to increase the "fees per token".

To maintain a correct accounting, any posterior changes in `totalSupply` should be accounted for to not affect the
aggregated unclaimed fees of the users, as there are no further fees added to the system. To achieve
this,  `userFeeOffset` should be set to `cumulativeFeePerToken` before any user operation that would increase or
decrease the `totalSupply`.

Nevertheless, there is no such update on two functions that change the `totalSupply`:

- [`Curves::deposit()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L490)
- [`Curves::withdraw()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L465)

#### Impact

The following protocol invariant is broken:  _At any given time, the ether balance should match the aggregated claimable
fees from all holders, so that all of them can claim their corresponding fees._

This invariant is broken when `addFees()` is called followed by `deposit()`, as the latter will increase the total
claimable fees above the contract ether balance.

Luckily, the `FeeSplitter` contract has a `payable receive()` function, so it could become solvent again, but it would
be at the expense of the protocol owners sending extra ether to the contract. This could overall increase the expenses
for the team significantly, proportionally to the tokens that are withdrawn when the `addFees()` function is called.

Moreover, if `FeeSplitter::addFees()` is called, and `Curves::withdraw()` is called afterwards, the bad accounting goes
in the other direction, and some ether would sit in the contract without anybody being able to claim it.

#### Proof of Concept

Take the following scenario. There are 10 tokens of a particular curve, held among 2 users (5 tokens each), and there
are 5 extra tokens that had been withdrawn as an external ERC20. Now `addFees()` is called with `msg.value = 10 ether`.
The supply accounted in the distribution is 10 tokens, therefore, there is "1 ether per token",
so `tokensData[token].cumulativeFeePerToken = 1 ether`. To sum up the current state:

- Total supply sitting in `Curves` contract = 10 tokens
- Fees per token = 1 ether / token
- `FeeSpliter` ether balance = 10 ether
- Aggregated claimables of all holders = 10 tokens * 1 ether/token = 10 ether == ether balance. >> OK

Then, the external 5 ERC20 tokens are deposited. The mapping `userFeeOffset` is not updated upon deposits, so the "fees
per token" stays constant for the depositor. The updated state is:

- Total supply in `Curves` contract = 15 tokens
- Fees per token = 1 ether / token (no change)
- `FeeSplitter` ether balance = 10 ether (no change)
- Aggregated claimables of all holders = 15 tokens * 1 ether/token = 15 ether > contract balance.
  Therefore, **Boken invariant**.

#### Recommended Mitigation Steps

`FeeSplitter::onBalanceChange()` should be called every time `Curves::deposit()` or `Curves::Withdraw()` are called.

```diff Curves.sol
  function deposit(address curvesTokenSubject, uint256 amount) public {

      // ... requirements and checks here ...

+       feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);

      CurvesERC20(externalToken).burn(msg.sender, amount);
      _transfer(curvesTokenSubject, address(this), msg.sender, tokenAmount);
  }
```

```diff Curves.sol
  function withdraw(address curvesTokenSubject, uint256 amount) public {

      // ... requirements and checks here ...

+       feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);

      _transfer(curvesTokenSubject, msg.sender, address(this), amount);
      CurvesERC20(externalToken).mint(msg.sender, amount * 1 ether);  // @audit-ok when minted, the amounts are tracked with 18 decimals
  }

```

### [H-3] An attacker can flashloan ERC20 curves tokens and steal rewards from other holders

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L80

When the tokens are withdrawn as external ERC20 tokens using `Curves::withdraw()`, they can be deposited as liquidity in
DEX pools, and therefore they can be flash-loaned. No rule disallows depositing and claiming fees in the same
transaction, therefore allowing for claims using a large number of flash-loaned tokens. This would steal fees that
belonged to other users who have been holding the tokens for longer periods and are therefore more rightful to earn the
fees.

#### Impact

An attacker can steal a large portion of the fees from `FeeSplitter` contract, without holding the tokens for longer
than a single flash-loan transaction, significantly diminishing the fees earned by other holders.

#### Proof of Concept

Let's assume a scenario where some curves tokens are withdrawn and deposited in a DEX liquidity pool that supports flash
loans (Uniswap V2 for instance). An attacker could do the following steps in a single transaction:

- Flashloan the max amount of tokens
- Deposit all the flash-loaned tokens using `Curves::deposit()`
- Claim fees calling `FeeSplitter::claimFees()`, where the calculated claimable fees will be inflated with all the
  tokens from the DEX pool
- Withdraw all flash-loaned tokens using `Curves::withdraw()`
- Pay back the flash loan and extra fees.

#### Recommended Mitigation Steps

Disallow claiming fees in the same transaction or block where a deposits or withdraw has taken place

```diff Curves.sol
mapping(address => uint256) public lastDepositBlockNumber;

// ...

function deposit(address curvesTokenSubject, uint256 amount) public {

   lastDepositBlockNumber[msg.sender] = block.number;

   // ...
```

```diff FeeSplitter.sol
function claimFees(address token) external {

  require(curves.lastDepositBlockNumber(mgs.sender) < block.number);

  // ...

```

### [H-4] Malicious tokenSubject can DOS all holders and impede them to sell the tokens by setting a malicious referralFeeDestination that does not accept ether

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L240
- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L158

As seen
in [`curves::setReferralFeeDestination()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L158),
the `tokenSubject` can change the destination of referral fees at will. Moreover, if the ether transfer to the referral
address fails
in [`Curves::_transferFees()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L243),
the whole transaction reverts.

#### Impact

The tokenSubject has the power to DOS any public function that internally calls `_transferFees()`. The affected
functions are:

- [`Curves::_buyCurvesToken()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L274)
- [`Curves::sellCurvesToken()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L292)

Thus, the tokenSubject can halt buying and selling transactions of its token at any point in time. The tokenSubject
won't necessarily profit from it, as he earns fees from purchases, but it gives him a huge manipulation power.

Note1: a malicious protocol and manager could also perform such attack, but in the specifications of the contest it is
stated that we must assume these two agents act in good will. However, assuming that all tokenSubjects will act in good
will is not a correct assumption.

Note2: The tokenSubject address itself could also be a malicious contract, but as the tokenSubject address is public
knowledge, buyers could see in advance that it is a malicious implementation and therefore not buy in the first place,
as this address cannot change.

#### Proof of Concept

The tokenSubject can simply deploy a contract that reverts when receiving ether:

```javascript
contract Malicious {
receive() external payable {
   revert("You have been denied");
}
}
```

And call `Curves::setReferralFeeDestination()` with the address of that malicious contract as the receiver.

#### Recommended Mitigation Steps

The recommendation is to favor the pull-over-push pattern. The `_transferFees()` function should not perform any ether
transfers but instead, it should store the balance into a storage variable. Then, a claim function should be implemented
to allow the tokenSubjects to claim the stored fees.

differences in `Curves.sol`:

```diff

contract Curves is CurvesErrors, Security {
 mapping(address => uint256) public claimableByTokenSubject;

 // ...

 function _transferFees() {

     // ....

-       (bool success3,) = referralDefined
-           ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
-           : (true, bytes(""));
-       if (!success3) revert CannotSendFunds();

+       if (referralDefined) {
+           claimableByTokenSubject[curvesTokenSubject] += referralFee;
+       }
 }

+   function claimTokenSubjectFee() external {
+       uint256 claimable = claimableByTokenSubject[msg.sender];
+       delete claimableByTokenSubject[msg.sender];
+       (bool success,) = referralFeeDestination[msg.sender].call{value: claimable}("");
+       require(success);
+   }
}


```

## Medium

### [M-1] Holders will lose unclaimed fees when selling Curves token due to bad accounting on onBalanceChange()

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L96

If a user buys some curves tokens and holds them for a while, the claimable fees will presumably increase as fees are
being distributed. This is correctly reflected in the view function `FeeSplitter::getClaimableFees()`, which
compares `cumulativeFeePerToken` with the `userFeeOffset[account]` to keep track of the "fees per token".

```javascript
  TokenData storage data = tokensData[token];
  uint256 balance = balanceOf(token, account);
  uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;
```

If the holder sells the tokens before claiming the rewards by calling `Curves::sellCurvesToken()`, this function
calls `FeeSplitter:onBalanceChange()` which resets the user's "fees per token", matching it to the current global "fees
per token" by setting `userFeeOffset[account]=cumulativeFeePerToken`.
However, an important step is missed, which is to account for the unclaimed fees, as the `unclaimedFees[account]`
mapping is not updated. Therefore, the seller will lose any unclaimed fees that he would have earned if he had
called `FeeSplitter::claimFees()` before selling.

#### Impact

Holders that sell tokens before claiming unclaimed fees, will lose those unclaimed fees.

#### Proof of Concept

An honest user performs the following actions:

- Buys tokens with `Curves::buyCurvesToken()`
- Waits for a long time, and accumulates fees in his balance, but without claiming them.
- He can see that `FeeSplitter::getClaimableFees()` returns a non-zero value.
- He now sells the tokens with `Curves::sellCurvesToken()` without claiming fees first.
- He can see now that `FeeSplitter::getClaimableFees()` returns zero. He has lost the fees
- Attempting to call `FeeSplitter::claimFees()` reverts with `NoFeesToClaim` error.

#### Recommended Mitigation Steps

Update the `unclaimedFees` mapping with the unclaimed fees in the function `FeeSplitter::onBalanceChange()`:

```diff FeeSplitter.sol::onBalanceChange()
function onBalanceChange(address token, address account) public onlyManager {

+       tokensData[token].unclaimedFees[account] = getClaimableFees(token, account);

    TokenData storage data = tokensData[token];
    data.userFeeOffset[account] = data.cumulativeFeePerToken;
    if (balanceOf(token, account) > 0) userTokens[account].push(token);
}
```

### [M-2] The function `onBalanceChange()` is not called in several operations that do change an accounts balance without changing totalSupply

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L96

Fee accounting in the `FeeSplitter` depends on the balance of an account, because the contract tracks the "fees per
token" and multiplies by the token balance. When the account's balance changes, it is necessary to reset the "fees per
token" for that account by setting `userFeeOffset[account] = cumulativeFeePerToken`. The function in charge of doing
this update is `onBalanceChange()`.

The function `onBalanceChange()` should be triggered every time an account's balance changes. However, there are at
least two functions that alter users' balances, (without changing the total supply) that should invoke such function:

- [Curves::transferCurvesToken()](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L296C14-L296C27)
- [Curves::transferAllCurvesTokens()](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L302C14-L302C25)

#### Impact

The bad accounting results in an unfair distribution of the fees between sender and receiver accounts, as the sender who
has "earned" fees will lose unclaimed fees when transferring their tokens to a receiver, who will receive those fees.

Note that this bug does not make the protocol insolvent, as the supply does not change when the two functions above are
called, so the aggregated claimable fees of all holders do not change.

#### Proof of Concept

A simple proof of concept that illustrates the issue:

- User Alice buys tokens with `Curves::buyCurvesToken()`
- Waits for a long time, and accumulates fees in her balance, which can be seen by
  calling `FeeSplitter::getClaimableFees() > 0`.
- Without calling `FeeSplitter::claimFees()`, she transfers the tokens to another user (Bob) as part of an OTC trade
  using `Curves::transferCurvesToken()`.
- She loses all unclaimed rewards as now `FeeSplitter::getClaimableFees()` returns 0 for her.
- Bob sees that `FeeSplitter::getClaimableFees() > 0` from the right moment he receives the tokens
- Alice has lost all unclaimed rewards, which are now claimable by Bob.

#### Recommended Mitigation Steps

Include calls to `onBalanceChange()` in the `Curves.sol` token transfer functions for both the sender and receiver:

```diff
  function transferCurvesToken(address curvesTokenSubject, address to, uint256 amount) external {
      if (to == address(this)) revert ContractCannotReceiveTransfer();

+       feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
+       feeRedistributor.onBalanceChange(curvesTokenSubject, to);

      _transfer(curvesTokenSubject, msg.sender, to, amount);
  }
```

```diff
  function transferAllCurvesTokens(address to) external {
      if (to == address(this)) revert ContractCannotReceiveTransfer();
      address[] storage subjects = ownedCurvesTokenSubjects[msg.sender];
      for (uint256 i = 0; i < subjects.length; i++) {
          uint256 amount = curvesTokenBalance[subjects[i]][msg.sender];
          if (amount > 0) {

+               feeRedistributor.onBalanceChange(subjects[i], msg.sender);
+               feeRedistributor.onBalanceChange(subjects[i], to);

              _transfer(subjects[i], msg.sender, to, amount);
          }
      }
  }
```

### [M-3] Lack of input validation on Curves::setMaxFeePercent() can make getSellPriceAfterFee() revert if fees are wrongly configured, with max above 1 ether

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L117

When the protocol owner configures the fees, a check imposes that the aggregated fee percentages do not exceed
the `feesEconomics.maxFeePercent`.

However, there is no input validation when setting this cap on `Curves::setMaxFeePercent()`. The max fee should never
surpass 100% of the price, which in this case is 1 ether. This is not entirely unlikely, as the choice
of `100% == 1 ether` is completely arbitrary, and an uncareful manager could interpret `100% == 100ether` for instance.

#### Impact

If the protocol owner mistakenly configures the fees with the wrong units, and the total fees surpass `1 ether`, (
surpassing therefore 100%), then the function `getSellPriceAfterFee()` will revert, as `price < totalFee`.

```javascript
function getSellPriceAfterFee(address curvesTokenSubject, uint256 amount) public view returns (uint256) {
    uint256 price = getSellPrice(curvesTokenSubject, amount);
    (, , , , uint256 totalFee) = getFees(price);
@>      return price - totalFee;
}
```

#### Proof of Concept

- The owner misinterprets units and thinks that `100%=100ether` (for instance)
- The owner sets `maxFeePercent` to 15 ether thinking it means 15%, while the protocol understands 1500%.
- The owner sets the remaining protocol fees that add up to 15 ether.
- users will not be able to sell tokens, as `getSellPriceAfterFee()` will revert.

Note that this scenario is not entirely unlikely, as the choice of `100% == 1 ether` is completely arbitrary.

#### Recommended Mitigation Steps

Add input validation in `Curves::setMaxFeePercent()`:

```diff
function setMaxFeePercent(uint256 maxFeePercent_) external onlyManager {
+       if (maxFeePercent > 1 ether) revert InvalidFeeDefinition();
    if (
        feesEconomics.protocolFeePercent +
            feesEconomics.subjectFeePercent +
            feesEconomics.referralFeePercent +
            feesEconomics.holdersFeePercent >
        maxFeePercent_
    ) revert InvalidFeeDefinition();
    feesEconomics.maxFeePercent = maxFeePercent_;
}
```

### [M-4] Incorrect Referral Fee Implementation

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L166

There is little documentation about the referral structure and its purpose besides the following sentence from the
Code4rena contest description:

_"Curves empowers protocols built upon its framework by enabling them to earn a percentage of all user transaction
fees"_.

According to this, the transaction fees from the user's perspective should be the same regardless if a user purchases
tokens from the base protocol or the protocols built upon its framework. That fee is then split between the base
protocol and the protocols on top depending on which "percentage" has been set to.

This is not the case, however, as the referral fee is added **on top** of the base fee from the protocol. The user will
get a less favorable pricing structure when `referralFee > 0`:

- Purchasing tokens will be more expensive, as the buying price including fees is calculated as:

  ```javascript
  uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
  ```

- Selling tokens will yield less income, as the selling price including the fees is calculated as:

  ```javascript
  uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
  ```

#### Impact

The user will get a less favorable pricing structure when `referralFee > 0`. Moreover, any subject will have a strong
incentive to always set referralFee to itself, (or another address owned by the subject), as adding a referral fee does
not incur a loss on its base earnings, but it is passed to the user on top of the original price.

#### Proof of Concept

For a subject where `referralFee > 0`, the buy price is higher and the selling price is lower than for a subject
where `referralFee == 0`.

#### Recommended Mitigation Steps

Ideally, the `referralFee` should be a percentage of the `protocolFee`, only changeable by the protocol. Moreover, the
buy price should not be affected by the referral fee.

Changes in `Curves::_transferFees()`

```diff
-   uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
+   uint256 buyValue = referralDefined ? protocolFee : protocolFee;

-   uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
+   uint256 sellValue = price - protocolFee - subjectFee - holderFee;
```

Changes in `Curves::getFees()`

```diff
  function getFees(uint256 price)
      public
      view
      returns (uint256 protocolFee, uint256 subjectFee, uint256 referralFee, uint256 holdersFee, uint256 totalFee)
  {
-        protocolFee = (price * feesEconomics.protocolFeePercent) / 1 ether;
-        subjectFee = (price * feesEconomics.subjectFeePercent) / 1 ether;
-        referralFee = (price * feesEconomics.referralFeePercent) / 1 ether;
-        holdersFee = (price * feesEconomics.holdersFeePercent) / 1 ether;

+        // the referral fee should be a percentage of the protocol fee
+        baseProtocolFee = (price * feesEconomics.protocolFeePercent) / 1 ether;
+        subjectFee = (price * feesEconomics.subjectFeePercent) / 1 ether;
+        referralFee = (baseProtocolFee * feesEconomics.referralFeePercent) / 1 ether;
+        protocolFee = baseProtocolFee - referralFee;
+        holdersFee = (price * feesEconomics.holdersFeePercent) / 1 ether;

-        totalFee = protocolFee + subjectFee + referralFee + holdersFee;
+        totalFee = subjectFee + referralFee + holdersFee;
  }

```

On top of this, sharing the protocol fee and the referral fee should be done inside the `Curves::getFee()` function.

### [M-5] A malicious agent can frontrun buyCurvesToken() and cause it to revert

- https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L270

The function `Curves::_transferFees()` reverts if the `msg.value` is not enough. To estimate the value that needs to be
sent including the price + fees, users can call the view
function [`Curves::getBuyPriceAfterFee()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L197)

However, the estimated price depends on the exact supply from which the tokens are being purchased, as the price depends
on the supply and the amount,
see [`getPrice()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L180).

This means that if a user sends a transaction to buy tokens, if this transaction is frontrun, the required value will
not suffice, and the transaction will revert

#### Impact

A malicious griefer can frontrun purchase transactions from a specific user, or of a specific tokenSubject, purchase the
token, and sell it again. The cost of this griefing is the fees of buying and selling. It is not a profitable attack,
but it can be used to deny a particular user from getting a particular token.

The victim could choose to broadcast the purchase transaction again, this time increasing the `msg.value` sent (as the
contract accepts more than the actual price), however, the frontrunner can see the value sent, and frontrun with a
higher amount of tokens purchased, denying it again, just at a slightly higher cost.

#### Proof of Concept

- Victim calls `Curves::buyCurvesToken()` with the exact `msg.value` including fees
- Frontrunner sees it and frontruns by calling `Curves::buyCurvesToken()` and purchases a single token to deny the
  purchase of the victim.

#### Recommended Mitigation Steps

Not an easy solution. The victim might have to use another wallet.

## Low

### [L-1] Curves::_buyCurvesToken() excess value is not returned to buyer __

The
function [`Curves::_buyCurvesToken()`](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L263C14-L263C24)
accepts a `msg.value > price`, but the excess value is not returned to the buyer, but stays in the contract.

### [L-2] Curves::transferAllCurvesTokens() unbounded gas loop, because tokens are not removed from ownedCurvesTokenSubjects array. Consider removing them when selling or transferring

The
function [Curves::transferAllCurvesTokens()](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L302)
has an unbounded loop, which turns into an unpredictable amount of gas. Moreover, the iterated
array `ownedCurvesTokenSubjects`, does not remove elements once they are unnecessary, so it only increases with the
amount of `subjectTokens` that an account has ever held in history.

#### Mitigation

Consider removing the `tokenSubject` from the `ownedCurvesTokenSubjects` when the balance turns 0 (either
in `Curves::sellCurvesToken()` or in `transferAllCurvesTokens()` or `transferCurvesToken()`)

### [L-3] Duplicatd elements inside FeeSplitter::userTokens mapping as FeeSplitter::onBalanceChange() does not check if elements are present in the array before adding

The function `FeeSplitter::onBalanceChange()` adds the token address to the `userTokens` array if the user has positive
balance, regardless if the token has already been
added, [see here](https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L99).

#### Impact:

`FeeSplitter::getUserTokens()` will show duplicated tokens
`FeeSplitter::getUserTokensAndClaimable()` will show duplicated claimable amounts, as tokens are duplicated
