# PoolId Library

The `PoolId` type is defined in the PoolId.sol contract, which is essentially `bytes32`:

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

The contract has only one method `toId`, which converts the `PoolKey` struct to `PoolId`. It hashes all fields of the `PoolKey` struct using `keccak256`, and the result is the `PoolId`.

### PoolKey

The `PoolKey` struct is defined as follows:

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

It can be seen that Uniswap v4 uniquely identifies a pool through the following fields:

- `currency0`: The first token in the pool
- `currency1`: The second token in the pool
- `fee`: The pool's fee
- `tickSpacing`: The pool's tick spacing
- `hooks`: The pool's hooks

By hashing this struct using `keccak256`, the result is the unique ID of the pool.
