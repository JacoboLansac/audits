# The transfer-whitelisting mechanism in TempleGold does not work cross-chain. 

## Description

In TempleGold, the token transfers are handled by the inherited OFT, which inherits ERC20, which uses the `_update()` method to update balances on transfers. 
The `_update()` method is overridden by TempleGold by only allowing transferring tokens to whitelisted addresses in the `authorized` mapping:

```javascript
    function _update(address from, address to, uint256 value) internal override {
        /// can only transfer to or from whitelisted addreess
        /// @dev skip check on mint and burn. function `send` checks from == to
        if (from != address(0) && to != address(0)) {
>>>>        if (!authorized[from] && !authorized[to]) { revert ITempleGold.NonTransferrable(from, to); }
        }
        super._update(from, to, value);
    }


```

When transfers are done cross-chain the `send()` function is used. This function calls `_debit()`, (who calls `_update()`, which is the one checking the `authorized` mapping). However, the `_debit()` function is only called **after** the requirement `msg.sender != _to`. Therefore, trying to send TGLD cross chain to a different address will revert even before reading if the `msg.sender` is `authorized` inside `_update()`. 


```javascript
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
>>>>    if (_to != address(0) && msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }

        /// @dev Applies the token transfers regarding this send() operation.
        // - amountSentLD is the amount in local decimals that was ACTUALLY sent/debited from the sender.
        // - amountReceivedLD is the amount in local decimals that will be received/credited to the recipient on the remote OFT instance.
>>>>    (uint256 amountSentLD, uint256 amountReceivedLD) = _debit(
            msg.sender,
            _sendParam.amountLD,
            _sendParam.minAmountLD,
            _sendParam.dstEid
        );

        // ...

    }
```

## Impact: ????

Whitelisted addresses cannot transfer TGLD tokens to a different address cross-chain. 
To transfer tokens cross-chain, smart-contracts need to transfer the tokens first to an EOA (or gnosis safe), then do the `send()` to send it to the same address cross-chain, and then transfer it to the final destination. 

## Proof of code

TBD if requested.
