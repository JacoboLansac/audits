# Hashlock findings review

By jacopod

## Overview:

High:
- H-01: **Invalid**: negligible effect or no economic incentives to do so.
- H-02: **Invalid**: all for-loops are bounded. See detailed description in next section

Low:
- L-01: Acknowledged:
- L-02: **Invalid**: no valid path in which the rentee is not in the array.
- L-03: Will fix. 
- L-04: **Invalid**: the contracts are only deployed in mainnet
- L-05: **Invalid**: even thought the naming is confusing, the function selectors are clear. It is not possible to call the wrong function
- L-06: Acknowledged: as a first proof-of-concept, this contract is accepted to have a high level of centralization
- L-07: Will fix. 

Gas:
- G-01: Acknowledged. Proof of concept, not needed to be gas efficient (there are room for much bigger improvements actually).
- G-02: Will fix.
- G-03: Will fix. 


--------------

## Detailed answers

### [H-01] PinToken#_transfer - Risk for rounding errors when calculating tax amount, potentially leading to incorrect tax amounts
- contract: PinToken
- commit hash: `3c0b0b31adc097e78e6d8c2cb3eeecdddb081e95`
- `PinToken._transfer()` doesn't exist in tht commit hash. 

Impact of the vulnerability: 

> These rounding errors may cause the contract to collect less tax than intended, affecting the distribution of staking rewards and the treasury. 
>> **Invalidity reason:** negligible effect. Collecting 1 wei less tax from every transaction is hardly going to affect the distribution of staking rewards. imagine 1.000.000 transactions per day, in which 1 wei of taxes are lost everytime. It would take 2,739,726,027 years to accumulate to 1 pin token. 

> This finding specifically affects the transfer of small token amounts.
>> **Invalidity reason:** no economic incentive: An attacker transfering 1 wei of pintoken to not pay tax is an irrelevant example, as the gas cost of the transaction exceeds significantly the tax saved. 

```js
    function _update(address sender, address recipient, uint256 amount) internal override {
        if (taxExempt[sender] || taxExempt[recipient]) {
            // Tax free transactiom
            super._update(sender, recipient, amount);
        }
        else if (dexWhitelist[sender] || dexWhitelist[recipient]) {
            // Buy or sell detected
@>          uint256 buySellTaxAmount = (amount * buySellTax) / TAX_DIVISOR;
            uint256 amountAfterTax = amount - buySellTaxAmount;

            super._update(sender, recipient, amountAfterTax);
            super._update(sender, treasury, buySellTaxAmount);
        } else {
```

Moreover, in the comit hash `3c0b0b31adc097e78e6d8c2cb3eeecdddb081e95` there is no `PinToken._transfer()`, so I'm assuming that you are talking about the `_update()` function. 


### [H-02] FractionalToken, Marketplace - Unbounded Loop Over Dynamic Arrays May Cause Out-of-Gas Errors
- commit hash: `3c0b0b31adc097e78e6d8c2cb3eeecdddb081e95`

Why it is invalid:
- FractionalToken.tokensOfOwner(): **INVALID**, as it is a `view` external function for frontend purposes
- FractionalToken._preUpdate(): **INVALID**, as the array of `ids` that is iterated, is passed as an argument to the function by the caller. 
- Marketplace.delist(): **INVALID**, because the addition of rentees to the array is bounded by `MAX_RENTEES_PER_TOKEN`
- Marketplace.removeRentee(): **INVALID**, because the addition of rentees to the array is bounded by `MAX_RENTEES_PER_TOKEN`
- Marketplace.adjustAllPauseRewards(): **INVALID**, because the addition of rentees to the array is bounded by `MAX_RENTEES_PER_TOKEN`

The for-loops in the Marketplace iterate over the array of rentees in the mapping `tokenRentees`. The audit claims that the array is unbounded but that claim is incorrect. 
The array is bounded because the addition of rentees can never surpass `MAX_RENTEES_PER_TOKEN`. 

```js
    function takeOnRent(uint256 tokenId, uint256 tokenAmount) external {
        require(tokenAmount > MIN_TOKEN_AMOUNT, "invalid tokenAmount"); // H1
        Rentable memory rentable = rentables[tokenId];
        require(rentableToken.isApprovedForAll(msg.sender, address(this)), "rentable token should be approved to marketplace");
        require(rentable.pauseTime == 0, "renting paused");
        require(rentable.amount - rentable.rentedAmount >= tokenAmount, "not enough fractions available");
        IERC20(rentable.collateral).safeTransferFrom(msg.sender, address(this), tokenAmount * rentable.collateralPerUnit); // G1
        if (rentals[tokenId][msg.sender].amount > 0) { // already rented
            uint256 rewardAmount = calculateReward(msg.sender, tokenId, tokenAmount);
            rewardToken.safeTransfer(msg.sender, rewardAmount);
        }
        else {
            // this avoids DOS due to running out of gas
@>          require(tokenRentees[tokenId].length <= MAX_RENTEES_PER_TOKEN, "Number of rentees exceeded");
            tokenRentees[tokenId].push(msg.sender);
        }
        rentals[tokenId][msg.sender].amount += tokenAmount;
        rentals[tokenId][msg.sender].collateralAmount += tokenAmount * rentable.collateralPerUnit; // H3
        /// 
    }
```
        
### [L-01] FractionalToken, Marketplace, PinToken - Missing Zero Checks

Acknowledged. Known issue. The deployer takes on the responsability of not inputting zero addresses. 

### [L-02] Marketplace#removeRentee() - Missing Error handling

Invalid: `removeRentee()` is an internal function only called by `returnOnRent()`. 

- There is no such case in which the rentee is not in the array. Otherwise, please provide proof of concept, because that would be a bigger issue. 
- Even in that case, adding a revert there when the rentee is not there could cause the function `returnOnRent()` to revert, which would be even worse. 
