# CurrencyDelta Library

CurrencyDelta Library 实现了闪电记账功能，通过记录某个地址的代币余额变化，来实现对代币余额的更新。由于所有操作在 `transient storage` 中进行，因此可以极大节省 gas。

[EIP-1153](https://eips.ethereum.org/EIPS/eip-1153) 介绍了如何在合约中使用 transient storage，通过引入 `tload` 和 `tstore` 操作，可以在合约中实现对临时存储的读写操作。

### _computeSlot

根据 `target` 和 `currency` 计算出存储的 slot 地址，用于存储余额变化值。

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

因为 `target` 和 `currency` 都是 `address` 类型，因此只需要对保留前 20 字节的数据。

同时，在 EVM 的内存布局中，前 64 字节是用于 hash 计算的临时区域，因此可以直接使用 `mstore` 指令将 `target` 和 `currency` 分别写入 0 和 32 的位置（各 32 字节）。

最后，通过 `keccak256(0, 64)` 计算出 slot 地址。

### getDelta

获取某个地址的代币余额变化值。

```solidity
function getDelta(Currency currency, address target) internal view returns (int256 delta) {
    bytes32 hashSlot = _computeSlot(target, currency);
    assembly ("memory-safe") {
        delta := tload(hashSlot)
    }
}
```

通过 `_computeSlot` 计算出 slot 地址，然后通过 `tload` 读取 slot 中的值。

### applyDelta

应用新的余额变化值，更新某个地址的代币余额。

输入参数：

- `currency` 代币类型
- `target` 地址
- `delta` 余额变化值

返回：

- `previous` 之前的余额
- `next` 更新后的余额

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

通过 `_computeSlot` 计算出 slot 地址，然后通过 `tload` 读取 slot 中的值，得到之前的余额。

接着，计算出新的余额，并通过 `tstore` 更新 slot 中的值。
