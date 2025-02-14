# PathKey Library

## Struct Definitions

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

- `intermediateCurrency`: Intermediate token
- `fee`: Fee
- `tickSpacing`: Pool tick spacing
- `hooks`: Hooks contract
- `hookData`: Hooks data

## Method Definitions

### getPoolAndSwapDirection

Calculate the trading pool and trading direction based on the input token and intermediate token information.

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

In [PoolKey](../../v4-core/en/PoolIdLibrary.md#poolkey), the `PoolKey` struct is introduced. Uniswap v4 uniquely identifies a pool through `currency0`, `currency1`, `fee`, `tickSpacing`, and `hooks`, where the `currency0` address is less than the `currency1` address.

Therefore, when determining the trading pool in `getPoolAndSwapDirection`, first sort `currency0` and `currency1`, and combine with `fee`, `tickSpacing`, and `hooks` to get `poolKey`.

Based on whether `currencyIn` is equal to `currency0`, determine the trading direction `zeroForOne`, as `currency0` is `token0`.
