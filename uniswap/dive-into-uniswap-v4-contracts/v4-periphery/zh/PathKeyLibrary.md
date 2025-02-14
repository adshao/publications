# PathKey Library

## 结构体定义

### PathKey

```solidity
struct PathKey {
    Currency intermediateCurrency;
    uint24 fee;
    int24 tickSpacing;
    IHooks hooks;
    bytes hookData;
}
```

- `intermediateCurrency`：中间代币
- `fee`：手续费
- `tickSpacing`：池子 tick 间隔
- `hooks`：Hooks 合约
- `hookData`：Hooks 数据

## 方法定义

### getPoolAndSwapDirection

根据输入代币和中间代币信息，计算交易池子和交易方向。

```solidity
/// @notice Get the pool and swap direction for a given PathKey
/// @param params the given PathKey
/// @param currencyIn the input currency
/// @return poolKey the pool key of the swap
/// @return zeroForOne the direction of the swap, true if currency0 is being swapped for currency1
function getPoolAndSwapDirection(PathKey calldata params, Currency currencyIn)
    internal
    pure
    returns (PoolKey memory poolKey, bool zeroForOne)
{
    Currency currencyOut = params.intermediateCurrency;
    (Currency currency0, Currency currency1) =
        currencyIn < currencyOut ? (currencyIn, currencyOut) : (currencyOut, currencyIn);

    zeroForOne = currencyIn == currency0;
    poolKey = PoolKey(currency0, currency1, params.fee, params.tickSpacing, params.hooks);
}
```

在 [PoolKey](../../v4-core/zh/PoolIdLibrary.md#poolkey) 中介绍了 `PoolKey` 结构体，Uniswap v4 通过 `currency0`、`currency1`、`fee`、`tickSpacing`、`hooks` 来唯一标识一个池子，其中，`currency0` 地址小于 `currency1` 地址。

因此，在 `getPoolAndSwapDirection` 中确定交易池子时，首先排序 `currency0` 和 `currency1`，结合 `fee`、`tickSpacing`、`hooks`，得到 `poolKey`。

根据 `currencyIn` 是否等于 `currency0`，确定交易方向 `zeroForOne`，因为此时 `currency0` 即 `token0`。
