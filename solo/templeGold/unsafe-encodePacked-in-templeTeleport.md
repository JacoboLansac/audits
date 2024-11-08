# Temple Tokens can be lost forever when teleporting for certain combinations of the amount (`uint256`) and to (`address`)

## Description

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
>>>>    bytes memory _payload = abi.encodePacked(to.addressToBytes32(), amount);
        // debit
        temple.burnFrom(msg.sender, amount);
        emit TempleTeleported(dstEid, msg.sender, to, amount);

>>>>    receipt = _lzSend(dstEid, _payload, options, MessagingFee(msg.value, 0), payable(msg.sender));
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
>>>>    (address _recipient, uint256 _amount) = abi.decode(_payload, (address, uint256));
        temple.mint(_recipient, _amount);
    }
```

The issue is that the output bytes from `abi.encodePacked()` cannot always be decoded reliably because it removes padding (to be more gas efficient) and this can lead to collisions. 
In other words, the bytes produced by some combinations of (address, unit256), can be decoded into more than one pair of those types. Illustrative example of collision with `abi.encodePacked` with simple strings:
```solidity
	abi.encodePacked("a", "bc") == abi.encodePacked("ab", "c") // Both produce "abc", which is ambiguous when decoding.
```

When this happens, the `_lzReceive()`  will revert with an EVM error, unable to mint the tokens in the destination chain or notify the origin chain. In these situations, the user teleporting the tokens effectively loses the funds, as they were burned in the origin chain. 


## Impact: medium

For certain combinations of `(address to ,uint amount)`, the teleported tokens will be burned in the origin chain but won't be minted in the destination chain, i.e., user funds are lost.
- Likelihood: low
- Impact: High

## Mitigation

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


## Proof of Code

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

