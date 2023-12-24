# Introduction

A time-boxed security review of the **NextGen** protocol, with a focus on smart contract security.

Author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time and resource-bound effort to find as many vulnerabilities as possible, but there is no guarantee that all issues will be found. 
A security researcher holds no
responsibility for the findings provided in this document. A security review is not an endorsement of the underlying
business or product and can never be used as a guarantee that the protocol is bug-free. This security review is focused solely on the security aspects of the Solidity implementation of the contracts. 

# Protocol Summary

NextGen is a series of contracts whose purpose is to explore more experimental directions in generative art and other non-art use cases of 100% on-chain NFTs.
At a high-level, is a classic on-chain generative contract with extended functionality: phase-based, allowlist-based, delegation-based 
minting philosophy of The Memes, the ability to pass arbitrary data to the contract for specific addresses to customize the outputs, 
a wide range of minting models, etc. 

More info can be found in NextGen's [gitbook documentation](https://seize-io.gitbook.io/nextgen/).

# Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
| :----------------- | :----------: | :------------: | :---------: |
| Likelihood: High   |     High     |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

## Likelihood

- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no incentive.

## Impact

- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- **Low** - can lead to unexpected behavior with some of the protocol's functionalities that are not so critical.

## Actions required by severity level

- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

# Scope

- Number of Solidity Lines Of Code (nSLOC): 1265
- Audited commit hash: [a6f2397b68ef2865374c1bf7629349f25e71a44d](https://github.com/code-423n4/2023-10-nextgen/commit/a6f2397b68ef2865374c1bf7629349f25e71a44d)
- Audited as a warden from [Code4rena](https://code4rena.com/audits/2023-10-nextgen)
- Submission date: `2023-11-12`

## Inside scope

Files and contracts in scope for this audit in the table below:

| Contract |	SLOC | 	Purpose..................................................................................                                                                                                                                                                                                               | 	Libraries and Interfaces used                                                                     |
|---|----|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| smart-contracts/NextGenCore.sol |	366 | 	Core is the contract where the ERC721 tokens are minted and includes all the core functions of the ERC721 standard as well as additional setter & getter functions.                                                                                                                                     | 	ERC721Enumerable, Ownable, Strings, Base64, ERC2981, IRandomizer, INextGenAdmins, IMinterContract |
| smart-contracts/MinterContract.sol |	475 | 	The Minter contract is used to mint an ERC721 token for a collection on the Core contract based on certain requirements that are set prior to the minting process.                                                                                                                                      | 	INextGenCore, Ownable, IDelegationManagementContract, MerkleProof, INextGenAdmins, IERC721        |
| smart-contracts/NextGenAdmins.sol |	61 | 	The Admin contract is responsible for adding or removing global or function-based admins who are allowed to call certain functions in both the Core and Minter contracts.	                                                                                                                              | Ownable                                                                                            |
| smart-contracts/RandomizerNXT.sol |	51 | 	The RandomizerNXT contract is responsible for generating a random hash for each token during the minting process using the NextGen's proposed approach.	                                                                                                                                                | IXRandoms, INextGenAdmins, Ownable, INextGenCore                                                   |
| smart-contracts/RandomizerVRF.sol |	87 | 	The RandomizerVRF contract is responsible for generating a random hash for each token during the minting process using the Chainlink's VRF service.	                                                                                                                                                    | VRFCoordinatorV2Interface, VRFConsumerBaseV2, Ownable, INextGenCore, INextGenAdmins                |
| smart-contracts/RandomizerRNG.sol |	72 | 	The RandomizerRNG contract is responsible for generating a random hash for each token during the minting process using the ARRng.io service.	                                                                                                                                                           | ArrngConsumer, Ownable, INextGenCore, INextGenAdmins                                               |
| smart-contracts/XRandoms.sol |	39 | 	The randomPool smart contract is used by the RandomizerNXT contract, once it's called from the RandomizerNXT smart contract it returns a random word from the current word pool as well as a random number back to the RandomizerNXT smart contract which uses those values to generate a random hash.	 | Ownable                                                                                            |
| smart-contracts/AuctionDemo.sol |	114 | 	The auctionDemo smart contract holds the current auctions after the mintAndAuction functionality is called. Users can bid on a token and the highest bidder can claim the token after an auction finishes.	                                                                                             | IMinterContract, IERC721, INextGenAdmins, Ownable                                                  |


# Executive Summary

## Main Findings Summary

| Finding ID | Description | Severity |
|--|--|--|
| \[H-1\] | Arbitrary external calls inside a loop in `AuctionDemo::claimAuction()` make auctions vulnerable to DDOS attack | High |
| \[H-2\] | An auction can become unclaimable if the token is transfered to another address during the auction, locking all the ether from all bids in the contract | High |
| \[H-3\] | The value from the highest bid is not transferred to the owner of the auctioned token | High |
| \[M-1\] | Earnings from auctioned tokens are not shared with the teams and the artists | Medium |
| \[L-1\] | Tokens can be minted after collection is frozen if certain conditions are met | Low |

# Detailed findings

## High

### [H-1] Arbitrary external calls inside a loop in `AuctionDemo::claimAuction()` make auctions vulnerable to DDOS attack

#### Description

The `claimAuction()` performs external calls to arbitrary addresses inside a the for-loop that returns ether from non-winning bids. 
A malicious contract can make a bid, and then have a `receive()` function that reverts, making the `claimAuction()` 
revert every time. The auction could never be claimed, and the bids would be locked in the contract.

#### Proof of concept

The malicious contract could look something like this:

```javascript
contract Attacker {
    Auction auction;

    constructor(Auction _auction) {
        auction = _auction;
    }

    function participate(uint256 tokenId) external payable {
        auction.participateToAuction{value: msg.value}(tokenId);
    }

    receive() external payable {
@>      revert();
    }
}
```

So, if the attacker contract does not win the auction, when the `claimAuction()` is called and it attempts to return ether to the attacker contract, 
this one will revert, locking all the ether from all other bids in the contract.

#### Severity classification

- Likelihood: High, as it is a very easy attack vector to exploit.
- Impact: High, as it would make the auction unfinishable, and the ether from losing bids would be locked in the contract.
- Overall severity: **High**

#### Recommendation

The `claimAuction()` function should be refactored to use a pull-over-push pattern, where the winner can claim the token
and the bidders can claim their loosing bids, instead of the `claimAuction()` making all the transfers.

## High

### [H-2] An auction can become unclaimable if the token is transfered to another address during the auction, locking all the ether from all bids in the contract

#### Description

The auctioned token is not locked in the auction contract when an auction is started. When the auction is finished,
the `claimAuction()` expects it be owned in an address that has approved the auction contract. If the token is transferred
to another wallet,the call to `transferFrom()` will revert with `ERC721: caller is not token owner or approved`. This revert makes the auction unclaimable, 
and all the ether from participating bids will be locked in the contract.

This will also happen if the `Auction` contract is not approved, but in the specifications it was explicitly said that it is assumed that it is approved.

#### Severity classification

- Likelihood: High, as it is a very easy attack vector to exploit.
- Impact: High, as it would make the auction unfinishable, and the ether from losing bids would be locked in the contract.
- Overall severity: **High**

#### Recommendation

The auctioned token should be locked in the auction contract.

## High

### [H-3] The value from the highest bid is not transferred to the owner of the auctioned token 

#### Description

The `claimAuction()` is supposed to send the ether from the highest bid to the owner of the token 
(`ownerOfToken` in the snippet below). Instead, it is sent to the owner of the auction contract 
(`owner()` in the snippet below). The same mistake is made when emitting the `ClaimAuction` event.

```javascript
    function claimAuction(uint256 _tokenid) public WinnerOrAdminRequired(_tokenid, this.claimAuction.selector) {
        ...
        for (uint256 i = 0; i < auctionInfoData[_tokenid].length; i++) {
            if (
                auctionInfoData[_tokenid][i].bidder == highestBidder && auctionInfoData[_tokenid][i].bid == highestBid
                    && auctionInfoData[_tokenid][i].status == true
            ) {
                IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);
@>              (bool success,) = payable(owner()).call{value: highestBid}("");
@>                emit ClaimAuction(owner(), _tokenid, success, highestBid);
```

#### Severity classification

- Likelihood: High, as it will happen every time an auction is claimed.
- Impact: High, as the owner of the token will not receive the highest bid.
- Overall severity: **High**

#### Recommendation

Send the auctioned token to `ownerOfToken` instead of `owner()`.

```diff
      IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);
-     (bool success,) = payable(owner()).call{value: highestBid}("");
-     emit ClaimAuction(owner(), _tokenid, success, highestBid);
+     (bool success,) = payable(ownerOfToken).call{value: highestBid}("");
+     emit ClaimAuction(ownerOfToken, _tokenid, success, highestBid);
```

## Medium

### [M-1] Earnings from auctioned tokens are not shared with the teams and the artists

#### Description

When items are minted using `MinterContract::mint()`, the value sent by message sender to pay for the mint is stored in `collectionTotalAmount`, which is later shared among the team and the collection aritsts. 

```javascript
    function mint(

        // ...

        collectionTotalAmount[col] = collectionTotalAmount[col] + msg.value;

        // ...

```

However, when an item is auctioned, it is minted using the `MinterContract::mintAndAuction()` function, which does not store any `msg.value` to the `collectionTotalAmount`. Instead, it is designed so that the full value from the highest bid goes to the owner of the token.

This means that the artist and the team will not receive any earnings from this auctioned tokens. 

#### Severity classification

- Likelihood: high, as it will happen for every auctioned token
- Impact: medium, as only the team and the artists are impacted, and it is not clear from the docs if this is an intended feature
- Overall severity: **medium**

#### Recommendation

The value from the highest bid should also be shared with the artists and NextGen team members.


## Low

### [L-1] Tokens can be minted after collection is frozen if certain conditions are met

#### Description

One of the specifications of the protocol is that frozen collections cannot have the metadata changed, and more tokens
cannot be minted. Collections are frozen using the `NextGenCore.freezeCollection()` which sets the `collectionFreeze`
to true.

```javascript
    function freezeCollection(uint256 _collectionID) public FunctionAdminRequired(this.freezeCollection.selector) {
        require(isCollectionCreated[_collectionID] == true, "No Col");
@>      collectionFreeze[_collectionID] = true;
    }

```

When new tokens are minted, the only requirements are that the total supply is not exceeded and that the 
`publicEndTime` hasn't been reached yet. However, there is no check to see if the collection has been frozen or not. 

If a collection is frozen before the `publicEndTime` is reached, and the total supply hasn't reached, the frozing 
mechanism would not impede the mint of more tokens. 

#### Severity classification

- Likelihood: Low, as the collections will tipically be frozen only after the `publicEndTime` has been reached. 
- Impact: Medium, as it breaks a core premise of the protocol which is that frozen collections don't change.
- Overall severity: **Medium**

#### Recommendation

When a collection is frozen, the total supply should be overwriten by calling `setFinalSupply()`. 

```diff
    function freezeCollection(uint256 _collectionID) public FunctionAdminRequired(this.freezeCollection.selector) {
        require(isCollectionCreated[_collectionID] == true, "No Col");
++      setFinalSupply(_collectionID);
        collectionFreeze[_collectionID] = true;
    }
```
