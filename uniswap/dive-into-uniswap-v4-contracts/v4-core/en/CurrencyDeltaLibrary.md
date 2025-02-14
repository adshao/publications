# CurrencyDelta Library

The CurrencyDelta Library implements flash accounting by recording the token balance changes of a specific address to update the token balance. Since all operations are performed in `transient storage`, it can greatly save gas.

[EIP-1153](https://eips.ethereum.org/EIPS/eip-1153) introduces how to use transient storage in contracts. By introducing `tload` and `tstore` operations, read and write operations on temporary storage can be implemented in contracts.

### _computeSlot

Calculate the storage slot address based on `target` and `currency` to store the balance change value.

```solidity
/// @notice calculates which storage slot a delta should be stored in for a given account and currency
function _computeSlot(address target, Currency currency) internal pure returns (bytes32 hashSlot) {
    assembly ("memory-safe") {
        mstore(0, and(target, 0xffffffffffffffffffffffffffffffffffffffff))
        mstore(32, and(currency, 0xffffffffffffffffffffffffffffffffffffffff))
        hashSlot := keccak256(0, 64)
    }
}
```

Since both `target` and `currency` are of `address` type, only the lower 20 bytes of data need to be retained.

In the EVM memory layout, the first 64 bytes are used as a temporary area for hash calculation, so the `mstore` instruction can be used to write `target` and `currency` to addresses 0 and 32 (32 bytes each).

Finally, `keccak256(0, 64)` is used to hash the first 64 bytes of data to get the slot address.

### getDelta

Get the accounting balance of a specific address and token.

```solidity
function getDelta(Currency currency, address target) internal view returns (int256 delta) {
    bytes32 hashSlot = _computeSlot(target, currency);
    assembly ("memory-safe") {
        delta := tload(hashSlot)
    }
}
```

Calculate the slot address through `_computeSlot`, and then read the value in the slot through `tload`.

### applyDelta

Apply a new balance change value to update the balance of the specified address and token.

Input parameters:

- `currency` Token type
- `target` Address
- `delta` Balance change value

Returns:

- `previous` Balance before update
- `next` Balance after update

```solidity
/// @notice applies a new currency delta for a given account and currency
/// @return previous The prior value
/// @return next The modified result
function applyDelta(Currency currency, address target, int128 delta)
    internal
    returns (int256 previous, int256 next)
{
    bytes32 hashSlot = _computeSlot(target, currency);

    assembly ("memory-safe") {
        previous := tload(hashSlot)
    }
    next = previous + delta;
    assembly ("memory-safe") {
        tstore(hashSlot, next)
    }
}
```

Calculate the slot address through `_computeSlot`, and then read the value in the slot through `tload` to get the balance before the update.

Next, calculate the new balance and save it through `tstore`.
