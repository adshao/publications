# PoolId Library

在 PoolId.sol 合约中定义了 `PoolId` 类型，其实际上就是 `bytes32`：

```solidity
import {PoolKey} from "./PoolKey.sol";

type PoolId is bytes32;

/// @notice Library for computing the ID of a pool
library PoolIdLibrary {
    /// @notice Returns value equal to keccak256(abi.encode(poolKey))
    function toId(PoolKey memory poolKey) internal pure returns (PoolId poolId) {
        assembly ("memory-safe") {
            // 0xa0 represents the total size of the poolKey struct (5 slots of 32 bytes)
            poolId := keccak256(poolKey, 0xa0)
        }
    }
}
```

合约只有一个方法 `toId`，实现了将 `PoolKey` 结构体转换为 `PoolId` 的功能。将 `PoolKey` 结构体的所有字段进行 `keccak256` 哈希，得到的结果即为 `PoolId`。

### PoolKey

`PoolKey` 结构体定义如下：

```solidity
/// @notice Returns the key for identifying a pool
struct PoolKey {
    /// @notice The lower currency of the pool, sorted numerically
    Currency currency0;
    /// @notice The higher currency of the pool, sorted numerically
    Currency currency1;
    /// @notice The pool LP fee, capped at 1_000_000. If the highest bit is 1, the pool has a dynamic fee and must be exactly equal to 0x800000
    uint24 fee;
    /// @notice Ticks that involve positions must be a multiple of tick spacing
    int24 tickSpacing;
    /// @notice The hooks of the pool
    IHooks hooks;
}
```

能够看出， Uniswap v4 通过以下字段来唯一标识一个池子：

- `currency0`：池子中的第一个代币
- `currency1`：池子中的第二个代币
- `fee`：池子的手续费
- `tickSpacing`：池子的 tick 间隔
- `hooks`：池子的钩子函数

通过将这个结构体执行 `keccak256` 哈希，得到的结果即为池子的唯一 ID。
