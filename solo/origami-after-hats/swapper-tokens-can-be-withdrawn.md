# DexAggregatorSwapper::execute() allows an attacker to withdraw tokens that ended up in the contract

The function `OrigamiDexAggregatorSwapper:execute()` allows the caller to swap tokens, by calling an external dex (1inch router). Function characteristics:

- `OrigamiDexAggregatorSwapper:execute()` is an external function with no access control (anyone can call it).
- It accepts a `bytes calldata swapData` argument, for performing the actual swap against the 1inch router.
- It has some requirements to be met at the end of the function regarding the amounts of `sellToken` and `buyToken` coming in and out, to ensure the swap was successful.

*OrigamiDexAggregatorSwapper.sol*
```javascript
    /**
     * @notice Execute a DEX aggregator swap
     */
    function execute(
        IERC20 sellToken, 
        uint256 sellTokenAmount, 
        IERC20 buyToken, 
        bytes calldata swapData
    ) external override returns (uint256 buyTokenAmount) {
        Balances memory initial = Balances({
            sellTokenAmount: sellToken.balanceOf(address(this)),
            buyTokenAmount: buyToken.balanceOf(address(this))
        });

        sellToken.safeTransferFrom(msg.sender, address(this), sellTokenAmount);
        sellToken.forceApprove(router, sellTokenAmount);

        // Execute the swap
        (bool success, bytes memory returndata) = router.call(swapData);

        if (!success) {
            if (returndata.length != 0) {
                // Look for revert reason and bubble it up if present
                // https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Address.sol#L232
                assembly {
                    let returndata_size := mload(returndata)
                    revert(add(32, returndata), returndata_size)
                }
            }
            revert UnknownSwapError(returndata);
        }

        // Verify that we have spent the expected amount, and have received some proceeds
        Balances memory current = Balances({
            sellTokenAmount: sellToken.balanceOf(address(this)),
            buyTokenAmount: buyToken.balanceOf(address(this))
        });

        // Cannot have any remaining balance of sellToken
@>      if (current.sellTokenAmount != initial.sellTokenAmount) {  // the sellToken.balanceOf(this) needs to stay the same after the swap
            revert InvalidSwap();
        }

        // Should have a new balance of buyToken
        // slither-disable-next-line incorrect-equality
@>      if (current.buyTokenAmount == initial.buyTokenAmount) {  // the buyToken.balanceOf(this) has to change after the swap (increase/decrease)
            revert InvalidSwap();
        }

        unchecked {
            buyTokenAmount = current.buyTokenAmount - initial.buyTokenAmount;
        }

        // Transfer back to the caller
        buyToken.safeTransfer(msg.sender, buyTokenAmount);
        emit Swap(address(sellToken), sellTokenAmount, address(buyToken), buyTokenAmount);
    }

```

*The Issue:*

There is no requirement to enforce that the tokens inside `swapData` are the same as `buyToken` and `sellToken`. We can easily fool the requirements after the swap, as they only check `buyToken` and `sellToken`, but they can be different than the tokens in the actual swap. As bypassing the requirements is easy, we can pass `swapData` to swap the lost tokens in the contract's balance for something else, and send it to the attacker at the same time, using `1inch:unoswapTo()`. More details in *Attack scenario* section. The function [`unoswapTo()`](https://vscode.blockscan.com/ethereum/0x1111111254EEB25477B68fb85Ed929f73A960582) from 1inch router is shown below:

```javascript
    /// @notice Performs swap using Uniswap exchange. Wraps and unwraps ETH if required.
    /// Sending non-zero `msg.value` for anything but ETH swaps is prohibited
    /// @param recipient Address that will receive swapped funds
    /// @param srcToken Source token
    /// @param amount Amount of source tokens to swap
    /// @param minReturn Minimal allowed returnAmount to make transaction commit
    /// @param pools Pools chain used for swaps. Pools src and dst tokens should match to make swap happen
    function unoswapTo(
@>      address payable recipient,
        IERC20 srcToken,
        uint256 amount,
        uint256 minReturn,
        uint256[] calldata pools
    ) external payable returns(uint256 returnAmount) {
        return _unoswap(recipient, srcToken, amount, minReturn, pools);
    }

```

**Attack scenario**

- Let's say someone sent by mistake 2 WETH to the `OrigamiDexAggregatorSwapper` contract. 
- An attacker calls `execute()` with the following parameters:
	- `sellToken`: an ERC20 that will **not** be involved in the actual swap. For example, DOGE.
    - `buyToken`: an ERC20 deployed by the attacker called `FAKETOKEN`, that overrides `balanceOf(address)`, and returns `WETH.balanceOf(address)` instead. Also the `transfer()` function will be overridden to not do anything. 
    - `swapData`: necessary data to perform a swap with 1inch router using the `unoswapTo()` function, with the attacker address as `recipient`. The 2 WETH will be swapped for something else (7000 USDC for instance).

- The execution flow inside `execute()` is as follows:
    - reads `DOGE.balanceOf(swapper) = 0` 
    - reads `FAKETOKEN.balanceOf(swapper) -> WETH.balanceOf(swapper) = 2e18`
    - calls 1inch router, and swaps 2 WETH for 7000 USDC, which are sent to the attacker's address (`recipient`). 
    - reads `DOGE.balanceOf(swapper) = 0` (no change)
    - reads `FAKETOKEN.balanceOf(swapper) -> WETH.balanceOf(swapper) = 0` (it has been swapped)
    - requirement 1: `DOGE` balance of the swapper contract must NOT change: OK
    - requirement 2: `WETH` balance of the swapper contract MUST change: OK
    - buyTokenAmount is calculation underflows because the initial balance is higher than the current, but as it is inside an unchecked block, no reverts.
    - buyToken.safeTransfer(), this doesn't do anything, because the function was overridden with an empty shell.

The attacker successfully extracted 2 WETH from the contract bypassing the sanity checks.

**Impact: low**

If any tokens end up in the `OrigamiDexAggregatorSwapper`, an attacker can withdraw them before the owner calls `recoverToken()`.

Even though this attack would represent a loss of funds for the protocol, I classify it as low, because said contract is **not** expected to have any balance.
Tokens may only arrive at the contract's balance by mistake. However, if this contract was meant to have tokens in its balance, it would have been a high-severity vulnerability.

**Recommendation**

One of the two:
- Implement access control for `execute()` function, granting access to the contracts that need it
- Implement logic to read the buyToken/sellToken from the `swapData` argument. (clearly more tricky, as the swap data could be from different routers)

 
