# TempleGold security review

A time-boxed security review of **Temple Gold** for [**Temple Dao**](https://templedao.link/), with a focus on smart contract security and gas optimizations.

Author: [**Jacopod**](https://twitter.com/jacolansac), an independent security researcher.
Read [past security reviews](https://github.com/JacoboLansac/audits/blob/main/README.md).

## Findings Summary

| Finding | Severity | Description | Status |
| :--- | :--- | :--- | :--- |
| [[M-1]](<#m-1-temple-tokens-can-be-lost-forever-in-the-teleporter-for-certain-combinations-of-the-amount-and-destination-address>) | Medium | Temple Tokens can be lost forever in the Teleporter for certain combinations of the amount and destination address |  |
| [[X-1]](<#x-1-elevatedaccess-can-make-the-daigoldauction-contract-insolvent-by-calling-notifydistribution-without-transfering-the-tokens>) | Centralization | ElevatedAccess can make the `DaiGoldauction` contract insolvent by calling `notifyDistribution()` without transfering the tokens. |  |
| [[X-2]](<#x-2-the-function-templegoldstakingsetunstakecooldown-has-no-restrictions-and-elevated-access-can-lock-staked-funds-forever>) | Centralization | The function `TempleGoldStaking::setUnstakeCooldown()` has no restrictions and Elevated access can lock staked funds forever |  |
| [[X-3]](<#x-3-the-migrator-in-templegoldstaking-has-too-much-power-to-be-configured-without-any-control>) | Centralization | The migrator in `TempleGoldStaking` has too much power to be configured without any control |  |
| [[L-1]](<#l-1-any-msgvalue-sent-to-spiceauctionburnandnotify-will-be-donated-to-the-contract-and-not-returned-to-the-caller-if-invoked-in-the-minting-chain>) | Low | Any `msg.value` sent to `SpiceAuction.burnAndNotify()` will be donated to the contract and not returned to the caller if invoked in the minting chain |  |
| [[L-2]](<#l-2-it-is-possible-to-call-daigoldauctionstartauction-before-setting-the-configs>) | Low | It is possible to call `DaiGoldAuction.startAuction()` before setting the configs |  |
| [[L-3]](<#l-3-if-the-templeteleporter-is-not-set-as-a-valid-minter-of-templeerc20-token-in-all-chains-teleported-tokens-will-be-lost>) | Low | If the `TempleTeleporter` is not set as a valid minter of `TempleERC20` token in all chains, teleported tokens will be lost |  |
| [[L-4]](<#l-4-the-function-setvestingfactor-should-have-the-onlyarbitrum-modifier-to-avoid-misleading-outputs-from-other-view-functions>) | Low | The function `setVestingFactor()` should have the `onlyArbitrum` modifier to avoid misleading outputs from other view functions |  |
| [[L-5]](<#l-5-the-transfers-whitelisting-mechanism-in-templegold-does-not-work-cross-chain>) | Low | The transfers-whitelisting mechanism in TempleGold does not work cross-chain. |  |
| [[G-1]](<#g-1-variables-that-are-read-more-than-once-inside-the-same-function-should-be-cached-into-memory>) | Gas | Variables that are read more than once inside the same function should be cached into memory |  |
| [[G-2]](<#g-2-the-updaterewards-makes-a-duplicated-call-to-_rewardpertoken-while-the-output-should-not-change-in-between-calls>) | Gas | The `updateRewards()` makes a duplicated call to `_rewardPerToken()` while the output should not change in between calls |  |
| [[G-3]](<#g-3-some-logic-can-be-skipped-in-_rewardpertoken-when-lastupdatetime--periodfinish-to-save-gas>) | Gas | Some logic can be skipped in `_rewardPerToken()` when `lastUpdateTime == periodFinish` to save gas |  |
| [[I-1]](<#i-1-informational-issues--best-practices>) | Info | Informational issues / best practices |  |

## Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time and
resource-bound effort to find as many vulnerabilities as possible, but there is no guarantee that all issues will be
found.
A security researcher holds no
responsibility for the findings provided in this document. A security review is not an endorsement of the underlying
business or product and can never be taken as a guarantee that the protocol is bug-free. This security review is focused
solely on the security aspects of the solidity implementation of the contracts. Gas optimizations are not the main
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

**Main review:**
- Start date: `2024-10-13`
- End date: `2024-11-06`
- Effective total time: `48 hours`
- Commit hash in scope: [e20676d37c452f1405cbe91d39943c0322a6478e](https://github.com/TempleDAO/temple/pull/990/commits/e20676d37c452f1405cbe91d39943c0322a6478e)


**Mitigation review:**
- Mitigation review delivery date: `2024-11-07`
- Commit hash: [73c5e8bb0b3ebb622e454137b8a1302792f11457](https://github.com/TempleDAO/temple/pull/990/commits/73c5e8bb0b3ebb622e454137b8a1302792f11457)


Note: the system was thought to be deployed on the Arbitrum chain, and the commit hash for the review aligns with that. 
However, during the course of the audit the deam changed strategy and the deployment of most contracts will happen in mainnet. This is reflected in the mitigation review. 

### Files in original scope

| File                                           | nSLOC    |
| ---------------------------------------------- | -------- |
| `contracts/templegold/SpiceAuction.sol`        | 298      |
| `contracts/templegold/DaiGoldAuction.sol`      | 176      |
| `contracts/templegold/TempleGoldStaking.sol`   | 287      |
| `contracts/templegold/TempleGold.sol`          | 192      |
| `contracts/templegold/SpiceAuctionFactory.sol` | 51       |
| `contracts/templegold/TempleGoldAdmin.sol`     | 50       |
| `contracts/templegold/EpochLib.sol`            | 13       |
| `contracts/templegold/TempleTeleporter.sol`    | 38       |
| `contracts/templegold/AuctionBase.sol`         | 13       |
| **Total**                                      | **1118** |


## Protocol Overview

- Temple Gold (TGLD) is an ERC20 that can be earned either by staking into the TempleGoldStaking contract, or by bidding into the DaiGoldAuction contract. 
- TGLD is non-transferrable
- TGLD can be bridged over Layer Zero to other chains when the team deploys there
- Temple can be bridged between chains using the TempleTeleporter contract
- The Spice Auctions are meant to auction volatile tokens by bidding Temple Gold. After the auctions are complete, the TGLD bid can be burned

<br>

------------------------
------------------------




# Findings











## Medium

### [M-1] Temple Tokens can be lost forever in the TempleTeleporter for certain combinations of the amount and destination address

#### Description

The `TempleTeleporter.teleport()` function uses `abi.encodePacked()` when creating the `_payload` that will be sent via Layer Zero `_lzSend()`:

```solidity
    function teleport(
        uint32 dstEid,
        address to,
        uint256 amount,
        bytes calldata options
    ) external payable override returns(MessagingReceipt memory receipt) {
        if (amount == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }
        if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
        // Encodes the message before invoking _lzSend.
>>>     bytes memory _payload = abi.encodePacked(to.addressToBytes32(), amount);
        // debit
        temple.burnFrom(msg.sender, amount);
        emit TempleTeleported(dstEid, msg.sender, to, amount);

        receipt = _lzSend(dstEid, _payload, options, MessagingFee(msg.value, 0), payable(msg.sender));
    }

```

When the `_payload` is received on the destination chain, the `_lzReceive()` function processes the payload, decoding it with `abi.decode()`:

```solidity
    function _lzReceive(
        Origin calldata /*_origin*/,
        bytes32 /*_guid*/,
        bytes calldata _payload,
        address /*_executor,*/,  // Executor address as specified by the OApp.
        bytes calldata /*_extraData */ // Any extra data or options to trigger on receipt.
    ) internal override {
        // Decode the payload to get the message
>>>     (address _recipient, uint256 _amount) = abi.decode(_payload, (address, uint256));
        temple.mint(_recipient, _amount);
    }
```

The issue is that the output bytes from `abi.encodePacked()` cannot always be decoded reliably because it removes padding (to be more gas efficient) and this can lead to collisions.
In other words, the bytes produced by some combinations of (address, unit256), can be decoded into more than one pair of those types. Illustrative example of collision with `abi.encodePacked` with simple strings:
```solidity
abi.encodePacked("a", "bc") == abi.encodePacked("ab", "c") // Both produce "abc", which is ambiguous when decoding.
```

When this happens, the `_lzReceive()`  will revert with an EVM error, unable to mint the tokens in the destination chain or notify the origin chain. In these situations, the user teleporting the tokens effectively loses the funds, as they were burned in the origin chain.


#### Risk: medium

For certain combinations of `(address to ,uint amount)`, the teleported tokens will be burned in the origin chain but won't be minted in the destination chain, i.e., user funds are lost.
- Likelihood: low
- Impact: High

#### Mitigation

- Always use `abi.encode` by default for safety
- Only use `abi.encodePacked` if gas optimization is critical AND you're sure your inputs won't collide

```diff
    function teleport(
        uint32 dstEid,
        address to,
        uint256 amount,
        bytes calldata options
    ) external payable override returns(MessagingReceipt memory receipt) {
        if (amount == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }
        if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
        // Encodes the message before invoking _lzSend.
-       bytes memory _payload = abi.encodePacked(to.addressToBytes32(), amount);
+       bytes memory _payload = abi.encode(to.addressToBytes32(), amount);
        // debit
        temple.burnFrom(msg.sender, amount);
        emit TempleTeleported(dstEid, msg.sender, to, amount);

        receipt = _lzSend(dstEid, _payload, options, MessagingFee(msg.value, 0), payable(msg.sender));
    }

```


No changes are required in the `_lzReceive()` as both packing methods pack into the same datatype type (bytes).


#### Proof of Code

Here are two tests.
- The first one uses `abi.encodePacked()`, and will revert as soon as it finds a collision (very fast most likely).
- The second one uses `abi.encode()`, which won't fail regardless of the number of fuzz runs.

```solidity
pragma solidity 0.8.19;

import {Test, console} from "lib/forge-std/src/Test.sol";

contract EncodingTests is Test {
    /// For the sake of being as close as possible to the actual implementation,
    /// we encode the address as a bytes32 using the same function from LZ as the TempleTeleporter
    function addressToBytes32(address _addr) internal pure returns (bytes32) {
        return bytes32(uint256(uint160(_addr)));
    }

    function testEncodePacked(address inputAddress, uint256 inputValue) public {
        bytes memory data = abi.encodePacked(addressToBytes32(inputAddress), inputValue);

        (address decodedAddress, uint256 decodedValue) = abi.decode(data, (address, uint256));

        // These assertions will some times fail, 
        // For instance for the pair (0xfBaA208935a3cb0bFC2C0663F43f1e38B229DB4F, 55928329213096)
        // This can be double-checked with chisel
        assertEq(inputAddress, decodedAddress);
        assertEq(inputValue, decodedValue);
    }

    function testEncode_Strict(address inputAddress, uint256 inputValue) public {
        bytes memory data = abi.encode(addressToBytes32(inputAddress), inputValue);

        (address decodedAddress, uint256 decodedValue) = abi.decode(data, (address, uint256));

        // These assertions will never fail, because the encoding is done in an uneqivocal way
        assertEq(inputAddress, decodedAddress);
        assertEq(inputValue, decodedValue);
    }
}
```

#### Team response: Fixed
























## Centralization risks










### [Z-1] ElevatedAccess can make the `DaiGoldauction` contract insolvent by calling `notifyDistribution()` without transfering the amount

The function is handy to incorporate donated tokens to `nextAuctionGoldAmount`. 

```solidity
    function notifyDistribution(uint256 amount) external override {
        if (msg.sender != address(templeGold) && !isElevatedAccess(msg.sender, msg.sig)) { revert CommonEventsAndErrors.InvalidAccess(); }
        /// @notice Temple Gold contract mints TGLD amount to contract before calling `notifyDistribution`
>>>     nextAuctionGoldAmount += amount;
        emit GoldDistributionNotified(amount, block.timestamp);
    }
```

However, a compromised wallet with the right permissions could call this function and make the contract insolvent, because `nextAuctionGoldAmount` is what determins the `TGLD` that is being auctioned. If the amount is aucitoned, but the tokens aren't in the contract balance, the contract is insolvent.

```solidity
    function startAuction() external override {
        // ...
        _distributeGold();
>>>     uint256 totalGoldAmount = nextAuctionGoldAmount;
        nextAuctionGoldAmount = 0;
        uint256 epochId = _currentEpochId = _currentEpochId + 1;
        // ...
    }
```

#### Risk: low/med
- probability: low
- impact: high

Note however, that there is no economic incentive for `ElevatedAccess` to make the contract insolvency, besides pure evil.

#### Team response: Acknowledged




















### [Z-2] The function `TempleGoldStaking::setUnstakeCooldown()` has no restrictions and ElevatedAccess can lock staked funds forever

```solidity
    function setUnstakeCooldown(uint32 _period) external override onlyElevatedAccess {
        unstakeCooldown = _period;
        emit UnstakeCooldownSet(_period);
    }
```

This function has no constraints. ElevatedAccess can block tokens forever and never let users withdraw by setting a very long `unstakeCooldown` period, because the period is not set at the moment of staking, but at the moment of unstaking:

```solidity
    function withdraw(uint256 amount, bool claimRewards) external override whenNotPaused {
        /// @dev Check here so migrationWithdraw can skip this in emergency cases
>>>     uint256 unstakeTime = stakeTimes[msg.sender] + unstakeCooldown;
        if (unstakeTime > block.timestamp) { revert UnstakeCooldown(block.timestamp, unstakeTime); }
        _withdrawFor(msg.sender, msg.sender, amount, claimRewards, msg.sender);
    }
```

#### Risk: low

A compormised Elevated access with the right power can lock all staked funds in the contract.
- Probability: low.
- Impact: high

#### Mitigation

Consider adding a `MAX_UNSTAKE_COOLDOWN` constant variable, and a requirement such as:

```diff
    // ...
+   uint256 constant MAX_UNSTAKE_COOLDOWN = 4 weeks;

    function setUnstakeCooldown(uint32 _period) external override onlyElevatedAccess {
+       if (_period > MAX_UNSTAKE_COOLDOWN) revert("too long");    
        unstakeCooldown = _period;
        emit UnstakeCooldownSet(_period);
    }
```

#### Team response: Fixed






















### [Z-3] The migrator in `TempleGoldStaking` has too much power to be configured without any control

In `TempleGoldStaking`, the migrator has clearly a lot of power over user funds as it can withdraw stakes, and it therefore is a centralization weakpoint:

```solidity
    function migrateWithdraw(address staker) external override onlyMigrator returns (uint256) {
        if (staker == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
        uint256 stakerBalance = _balances[staker];
        _withdrawFor(staker, msg.sender, stakerBalance, true, staker);
        return stakerBalance;
    }
```

Even though the protocol has good intentions some of the following things could happen:

- The person in charge of deploying the migrator contract is malicious (very unlikely)
- The migrator has a bug, which can be exploited, affecting this staking contract as well (less unlikely)

#### Risk: low/med
- probability: low
- impact: high

#### Mitigation

A way of mitigating this would be to set access control in the `setMigrator()` function so that only the DAO can set the migrator. With this, we give some time to users/tech-geeks to review the new migrator contract before accepting.

#### Team response: fixed

























### [L-1] Any `msg.value` sent to `SpiceAuction.burnAndNotify()` will be donated to the contract and not returned to the caller if invoked in the minting chain

The function `SpiceAuction.burnAndNotify()` is payable to pay for fees of the underlying LayerZero calls. Any leftover ether not spent in the transaction is sent over to the caller at the end of the internal function `_burnAndNotify()`.

However, when the function is called in the `mintChainId`, there is no need to call LayerZero, and the `ITempleGold.burn()` function is invoked. After that, the function returns without returning unused ether to the caller. Therefore, any `msg.value` sent in this chain is donated to the contract.

```solidity
    function _burnAndNotify(uint256 amount, address from, bool useContractEth) private {
        // pull funds from bids recipient (set in config)
        IERC20(templeGold).safeTransferFrom(from, address(this), amount);
        // burn directly and call TempleGold to update circulating supply
        if (block.chainid == _mintChainId) {
            ITempleGold(templeGold).burn(amount);
>>>         return;
        }
        bytes memory options = OptionsBuilder.newOptions().addExecutorLzReceiveOption(lzReceiveExecutorGas, 0);
        SendParam memory sendParam = SendParam(
            _arbitrumOneLzEid, //<ARB_EID>,
            bytes32(uint256(uint160(address(0)))), // bytes32(address(0)) to burn
            amount,
            0,
            options,
            bytes(""), // compose message
            ""
        );
        MessagingFee memory fee = ITempleGold(templeGold).quoteSend(sendParam, false);
        if (useContractEth && address(this).balance < fee.nativeFee) {
            revert CommonEventsAndErrors.InsufficientBalance(address(0), fee.nativeFee, address(this).balance);
        } else if (!useContractEth && msg.value < fee.nativeFee) { 
            revert CommonEventsAndErrors.InsufficientBalance(address(0), fee.nativeFee, msg.value); 
        }

        if (useContractEth) {
            ITempleGold(templeGold).send{ value: fee.nativeFee }(sendParam, fee, payable(address(this)));
        } else {
            ITempleGold(templeGold).send{ value: fee.nativeFee }(sendParam, fee, payable(msg.sender));
            uint256 leftover;
            unchecked {
>>>             leftover = msg.value - fee.nativeFee;
            }
            if (leftover > 0) { 
>>>             (bool success,) = payable(msg.sender).call{ value: leftover }("");
                if (!success) { revert WithdrawFailed(leftover); }
            }
        }
    }
```
#### Risk: low
- probability: low
- impact: low

#### Mitigation

Don't allow `msg.value > 0` in the mint chain:

```diff
    function _burnAndNotify(uint256 amount, address from, bool useContractEth) private {
        // pull funds from bids recipient (set in config)
        IERC20(templeGold).safeTransferFrom(from, address(this), amount);
        // burn directly and call TempleGold to update circulating supply
        if (block.chainid == _mintChainId) {
+           if (msg.value > 0) revert("no eth needed");
            ITempleGold(templeGold).burn(amount);
            return;
        }
```

#### Team response: Fixed






















### [L-2] It is possible to call `DaiGoldAuction.startAuction()` before setting the configs

There is no requirement that the auction configs have been set:

```solidity
    function startAuction() external override {
        if (auctionStarter != address(0) && msg.sender != auctionStarter) { revert CommonEventsAndErrors.InvalidAccess(); }
        EpochInfo storage prevAuctionInfo = epochs[_currentEpochId];
        if (!prevAuctionInfo.hasEnded()) { revert CannotStartAuction(); }
       
        AuctionConfig storage config = auctionConfig;
        /// @notice last auction end time plus wait period
        if (_currentEpochId > 0 && (prevAuctionInfo.endTime + config.auctionsTimeDiff > block.timestamp)) {
            revert CannotStartAuction();
        }
        _distributeGold();
        uint256 totalGoldAmount = nextAuctionGoldAmount;
        nextAuctionGoldAmount = 0;
        uint256 epochId = _currentEpochId = _currentEpochId + 1;
        
        if (totalGoldAmount < config.auctionMinimumDistributedGold) { revert LowGoldDistributed(totalGoldAmount); }

        EpochInfo storage info = epochs[epochId];
        info.totalAuctionTokenAmount = totalGoldAmount;
        uint128 startTime = info.startTime = uint128(block.timestamp) + config.auctionStartCooldown;
        uint128 endTime = info.endTime = startTime + AUCTION_DURATION;

        emit AuctionStarted(epochId, msg.sender, startTime, endTime, totalGoldAmount);
    }
```


The auctions could be started with all the `AuctionConfig` fields set to zero, which would have the following consequences:
- auctionsTimeDiff = 0  >>> startAuction() can be called right away
- auctionStartCooldown = 0  >>> auction starts right after startAuction is called
- auctionMinimumDistributedGold = 0  >>> an auction can start even if there are no TempleGold to distribute, which makes the auction pretty useless.

The team won't start an auction without configs but if `auctionStarter == address(0)` , anyone could do it (by mistake or as a greifing attack, because I don't see any economic incentives). So either include a small `require`, or don't set `auctionStarter=address(0)` until configs have been set.

It is important to note that if this scenario occurr, it limits significantly the time period where the mistake can be fixed and new configs can be added, so it might require a new contract deployment, but I don't see user funds at risk anywhere.

#### Risk: low
- probability: low
- impact: medium

#### Proof of code

Small proof of code that the auction can actually be started without configs that can be added to `test/forge/templegold/DaiGoldAuction.t.sol`:

```solidity
contract DaiGoldAuction_POCs_Test is DaiGoldAuctionTestBase {

    function test_startAuction_canBeCalledBeforeConfigsAreSet() public {
        IDaiGoldAuction.AuctionConfig memory config = daiGoldAuction.getAuctionConfig();

        assertEq(config.auctionsTimeDiff, 0);
        assertEq(config.auctionStartCooldown, 0);
        assertEq(config.auctionMinimumDistributedGold, 0);

        vm.prank(executor);
        daiGoldAuction.startAuction();

        // we can see that the startTime is already set by reading the auction info
        IAuctionBase.EpochInfo memory info = daiGoldAuction.getEpochInfo(1);
        assertEq(info.startTime, block.timestamp);

        // as the cooldown is 0, bids can happen right away
        vm.startPrank(alice);
        bidToken.approve(address(daiGoldAuction), type(uint).max);
        daiGoldAuction.bid(100 ether);
    }
}
```

#### Mitigation

Require that any of the auction configs has a non-zero value (because they are all required to be greater than zero when calling `setAuctionConfig()`).













### [L-3] If the `TempleTeleporter` is not set as a valid minter of `TempleERC20` token in all chains, teleported tokens will be lost


Even if the TempleTeleporter contracts in both chains are wired up correctly via layer zero, for correct functioning of the system the `TempleTeleporter` needs minting rights.

If not setup correctly, the `teleport()` function will burn the tokens in the origin chain, send the message via LayerZero, but in the destination chain `tempe.mint()` will revert due to the lack of minting rights. As there is no way to communicate back to the origin chain that the transaction reverted, the caller will effectively lose his funds.

Luckily, the team has mint rights, so an affected user could notify the team and get compensated.

#### Risk: low
- probability: low
- impact: high


#### Mitigation

There isn't a solution that can be implemented at the contract level. However, it is recommended to have a proper deployment plan (or scripts) with all the steps necessary for the proper setup.

It is a nasty scenario that shouldn't ever occur, but I want to bring the attention so that it is included in the deployment plans.

#### Team response: Acknowledged

The team took note and appreciated the heads up. They will incorporate this info in their deployment plan. 




















### [L-4] The function `setVestingFactor()` should have the `onlyArbitrum` modifier to avoid misleading outputs from other view functions

If `setVestingFactor()` is called in a chain that is not the mint-chain for TGLD. then `_lastMintTimestamp` is set to a non-zero value. Having  `_lastMintTimestamp > 0`:
- `getMintAmount() > 0`
- `_canDistribute() = true`

Which is misleading, because the minting functions would actually revert.

Note: other functions could also have the  `onlyArbitrum` modifier for consistency, if they are functions related to the minting logic (`setStaking()`, `setDaiGoldAuction()`, `setTeamGnosis()`, ...), but it is not important

#### Risk: low
- probability: low
- impact: low

#### Team response: fixed

Note: the team decided during the course of the audit to change the deployment chain from arbitrum to mainnet.






















### [L-5] The transfers-whitelisting mechanism in TempleGold does not work cross-chain.

#### Description

In TempleGold, the token transfers are handled by the inherited OFT, which inherits ERC20, which uses the `_update()` method to update balances on transfers.
The `_update()` method is overridden by TempleGold by only allowing transferring tokens to whitelisted addresses in the `authorized` mapping:

```solidity
    function _update(address from, address to, uint256 value) internal override {
        /// can only transfer to or from whitelisted addreess
        /// @dev skip check on mint and burn. function `send` checks from == to
        if (from != address(0) && to != address(0)) {
>>>         if (!authorized[from] && !authorized[to]) { revert ITempleGold.NonTransferrable(from, to); }
        }
        super._update(from, to, value);
    }
```

When transfers are done cross-chain the `send()` function is used. This function calls `_debit()`, (who calls `_update()`, which is the one checking the `authorized` mapping). However, the `_debit()` function is only called **after** the requirement `msg.sender != _to`. Therefore, trying to send TGLD cross chain to a different address will revert even before reading if the `msg.sender` is `authorized` inside `_update()`.


```solidity
    function send(
        SendParam calldata _sendParam,
        MessagingFee calldata _fee,
        address _refundAddress
    ) external payable virtual override(IOFT, OFTCore) returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt) {
        if (_sendParam.composeMsg.length > 0) { revert CannotCompose(); }
        /// cast bytes32 to address
        address _to = _sendParam.to.bytes32ToAddress();
        /// @dev user can cross-chain transfer to self
        /// @dev whitelisted address like spice auctions can burn by setting `_to` to address(0)
        // only burn TGLD on source chain
        if (_to == address(0) && _sendParam.dstEid != _mintChainLzEid) { revert CommonEventsAndErrors.InvalidParam(); }
>>>     if (_to != address(0) && msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }

        /// @dev Applies the token transfers regarding this send() operation.
        // - amountSentLD is the amount in local decimals that was ACTUALLY sent/debited from the sender.
        // - amountReceivedLD is the amount in local decimals that will be received/credited to the recipient on the remote OFT instance.
>>>     (uint256 amountSentLD, uint256 amountReceivedLD) = _debit(
            msg.sender,
            _sendParam.amountLD,
            _sendParam.minAmountLD,
            _sendParam.dstEid
        );
        // ...

```

#### Risk: low

Whitelisted addresses cannot transfer TGLD tokens to a different address cross-chain.
To transfer tokens cross-chain, smart-contracts need to transfer the tokens first to an EOA (or gnosis safe), then do the `send()` to send it to the same address cross-chain, and then transfer it to the final destination.

#### Team response: Invalid

The team only intends to use the `authorized` mapping for whitelisting contracts in each chain, not cross-chain.





















## Gas optimizations

### [G-1] Variables that are read more than once inside the same function should be cached into memory

- The function `DaiGoldAuction::startAuction()` reads from storage `_currenEpochId` 3 times.
- The function `DaiGoldAuction::setAuctionConfig()` reads from storage `_currentEpochId` 2 times.
- The function `TempleGoldStaking::_notifyReward()` reads from storage `rewardDuration` 5 times.

Cache the variable in memory.

#### Team response: Fixed









### [G-2] The `updateRewards()` makes a duplicated call to `_rewardPerToken()` while the output should not change in between calls

The internal call to `_rewardPerToken()` reads multiple variables from storage. This function is called twice inside the `updateReards()` modifier, but the output of both cases will be the same as they are run in the same transaction.  (The first time is called directly in this modifier to update `rewardData.rewardsPerTokenStored`, and the second time is inside `_earned()`).

A significant amount of gas can be saved by caching the output of `_rewardPerToken()` and passing it as an argument to `_earned()`.

Given that this function will probably be one that will be executed more times over the lifetime of the contract I definitely think it is worth it.

The change would look something like this:
```diff
    modifier updateReward(address _account) {
-       rewardData.rewardPerTokenStored = uint216(_rewardPerToken());
+       uint216 rewardPerTokenCached = uint216(_rewardPerToken());
+       rewarddata.rewardPerTokenStored = rewardPerTokenCached;

        // ...

-       claimableRewards[_account] = _earned(_account);
+       claimableRewards[_account] = _earned(_account, rewardPerTokenCached);  // save a lot of gas here
-       userRewardPerTokenPaid[_account] = uint256(rewardData.rewardPerTokenStored);
+       userRewardPerTokenPaid[_account] = uint256(rewardPerTokenCached); // you also save a storage-read here
}
```

And of course you have to redefine the `_earned()` function to accept the new argument:

```diff
    function _earned(
        address _account,
+       uint256 rewardPerToken
    ) internal view returns (uint256) {
-       return _balances[_account] * (_rewardPerToken() - userRewardPerTokenPaid[_account]) / 1e18
- claimableRewards[_account];
+       return _balances[_account] * (rewardPerToken - userRewardPerTokenPaid[_account]) / 1e18
+ claimableRewards[_account];
}
```

#### Team response: Fixed
















### [G-3] Some logic can be skipped in `TempleGoldStaking._rewardPerToken()` when `lastUpdateTime == periodFinish` to save gas

```diff
    function _rewardPerToken() internal view returns (uint256) {
        if (totalSupply == 0) {
            return rewardData.rewardPerTokenStored;
        }

+       if (rewardData.lastUpdateTime == rewardData.periodFinish) {
+           return rewardData.rewardPerTokenStored;
+       }

        return
            rewardData.rewardPerTokenStored +
            (((_lastTimeRewardApplicable(rewardData.periodFinish) -
                rewardData.lastUpdateTime) *
                rewardData.rewardRate * 1e18)
                / totalSupply);
    }
```

Because when `rewardData.lastUpdateTime == rewardData.periodFinish`, then

`_lastTimeRewardApplicable(rewardData.periodFinish) - rewardData.lastUpdateTime) = 0;`.

And therefore the output becomes

`rewardData.rewardPerTokenStored + 0;`.

So you might as well return `rewardData.rewardPerTokenStored` directly and save some gas.

#### Team response: Fixed










## Informational issues

### [I-1] Informational issues / best practices

- Unused imports in `protocol/contracts/templegold/TempleGoldStaking.sol`.
- Incorrect naming of
- Unused declared state variables in `TempleGoldStaking`: `periodFinish` and `lastUpdateTime`
- Rename `_totalBurnedFromSpiceAuctions` to `_totalBurned` as not only the Spice auctions can burn tokens
- In the auction contracts, it would be very handly to expose `epochLib.isActive()` in a view function to quickly see if the current auction is active or not.
- Everytime `recoverToken()` is called, that epoch becomes a ghost auction with everything set at 0. This happens because `recoverToken()` only deletes `epoch[i]` but doesn't change `currentEpochId`, and then when `startAuction()` is called, the `epochId` of the new auction is `i+1`. So auction `i` is left as a ghost auction.
- `SpiceAuction` needed to have the `lzReceiveExecutorGas` updated to match the new gas ussage after the latest changes.
- The function `EpocLib::hasStarted()` is not used anywhere and can be removed
