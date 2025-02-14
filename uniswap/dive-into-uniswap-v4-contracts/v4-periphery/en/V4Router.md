# V4Router

Unlike [PositionManager](./PositionManager.md) which focuses on position management, V4Router is mainly used for executing swaps, calling the [PoolManager](../../v4-core/en/PoolManager.md) contract to complete specific swap operations.

Let's take a look at the declaration of the V4Router contract:

```solidity
/// @title UniswapV4Router
/// @notice Abstract contract that contains all internal logic needed for routing through Uniswap v4 pools
/// @dev the entry point to executing actions in this contract is calling `BaseActionsRouter._executeActions`
/// An inheriting contract should call _executeActions at the point that they wish actions to be executed
abstract contract V4Router is IV4Router, BaseActionsRouter, DeltaResolver {
```

Similar to [PositionManager](./PositionManager.md), the V4Router contract also inherits the `BaseActionsRouter` and `DeltaResolver` contracts, and executes operations in batches by calling the `BaseActionsRouter._executeActions` method.

V4Router itself is an abstract contract and cannot be deployed directly. Other contracts need to inherit the V4Router contract and implement the `pay` method in the `DeltaResolver` contract:

```solidity
/// @notice Abstract function for contracts to implement paying tokens to the poolManager
/// @dev The recipient of the payment should be the poolManager
/// @param token The token to settle. This is known not to be the native currency
/// @param payer The address who should pay tokens
/// @param amount The number of tokens to send
function _pay(Currency token, address payer, uint256 amount) internal virtual;
```

The `pay` method pays the specified amount of tokens to the `poolManager`.

The Uniswap v4 [universal-router](https://github.com/Uniswap/universal-router) contract [V4SwapRouter.sol](https://github.com/Uniswap/universal-router/blob/8bd498a3fc9f8bc8577e626c024c4fcf0691f885/contracts/modules/uniswap/v4/V4SwapRouter.sol#L14) inherits the V4Router contract and implements the `pay` method.

## Struct Definitions

The IV4Router interface defines some commonly used structs for `swap` methods:

### ExactInputSingleParams

Exact input swap parameters for a single-hop swap in a specified pool:

```solidity
/// @notice Parameters for a single-hop exact-input swap
struct ExactInputSingleParams {
    PoolKey poolKey;
    bool zeroForOne;
    uint128 amountIn;
    uint128 amountOutMinimum;
    bytes hookData;
}
```

Among them:

- `poolKey`: The key of the pool
- `zeroForOne`: Whether to swap from `token0` to `token1`
- `amountIn`: The input token amount
- `amountOutMinimum`: The minimum output token amount
- `hookData`: Hook data

### ExactInputParams

Exact input swap parameters for a multi-hop swap:

```solidity
/// @notice Parameters for a multi-hop exact-input swap
struct ExactInputParams {
    Currency currencyIn;
    PathKey[] path;
    uint128 amountIn;
    uint128 amountOutMinimum;
}
```

Among them:

- `currencyIn`: The input token
- `path`: The swap path, including information about intermediate tokens, refer to the [PathKey](./PathKeyLibrary.md#pathkey) struct definition
- `amountIn`: The input token amount
- `amountOutMinimum`: The minimum output token amount

### ExactOutputSingleParams

Exact output swap parameters for a single-hop swap in a specified pool:

```solidity
/// @notice Parameters for a single-hop exact-output swap
struct ExactOutputSingleParams {
    PoolKey poolKey;
    bool zeroForOne;
    uint128 amountOut;
    uint128 amountInMaximum;
    bytes hookData;
}
```

Among them:

- `poolKey`: The key of the pool
- `zeroForOne`: Whether to swap from `token0` to `token1`
- `amountOut`: The output token amount
- `amountInMaximum`: The maximum input token amount
- `hookData`: Hook data

### ExactOutputParams

Exact output swap parameters for a multi-hop swap:

```solidity
/// @notice Parameters for a multi-hop exact-output swap
struct ExactOutputParams {
    Currency currencyOut;
    PathKey[] path;
    uint128 amountOut;
    uint128 amountInMaximum;
}
```

Among them:

- `currencyOut`: The output token
- `path`: The swap path
- `amountOut`: The output token amount
- `amountInMaximum`: The maximum input token amount

## Method Definitions

Since V4Router is an abstract contract, it does not provide direct external call interfaces. Instead, the contracts inheriting it implement specific swap entry points.

### _handleAction

The V4Router contract mainly implements the `BaseActionsRouter._handleAction` method to handle different types of swap operations:

```solidity
function _handleAction(uint256 action, bytes calldata params) internal override {
    // swap actions and payment actions in different blocks for gas efficiency
    if (action < Actions.SETTLE) {
        if (action == Actions.SWAP_EXACT_IN) {
            IV4Router.ExactInputParams calldata swapParams = params.decodeSwapExactInParams();
            _swapExactInput(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_IN_SINGLE) {
            IV4Router.ExactInputSingleParams calldata swapParams = params.decodeSwapExactInSingleParams();
            _swapExactInputSingle(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_OUT) {
            IV4Router.ExactOutputParams calldata swapParams = params.decodeSwapExactOutParams();
            _swapExactOutput(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_OUT_SINGLE) {
            IV4Router.ExactOutputSingleParams calldata swapParams = params.decodeSwapExactOutSingleParams();
            _swapExactOutputSingle(swapParams);
            return;
        }
    } else {
        if (action == Actions.SETTLE_ALL) {
            (Currency currency, uint256 maxAmount) = params.decodeCurrencyAndUint256();
            uint256 amount = _getFullDebt(currency);
            if (amount > maxAmount) revert V4TooMuchRequested(maxAmount, amount);
            _settle(currency, msgSender(), amount);
            return;
        } else if (action == Actions.TAKE_ALL) {
            (Currency currency, uint256 minAmount) = params.decodeCurrencyAndUint256();
            uint256 amount = _getFullCredit(currency);
            if (amount < minAmount) revert V4TooLittleReceived(minAmount, amount);
            _take(currency, msgSender(), amount);
            return;
        } else if (action == Actions.SETTLE) {
            (Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
            _settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE) {
            (Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE_PORTION) {
            (Currency currency, address recipient, uint256 bips) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _getFullCredit(currency).calculatePortion(bips));
            return;
        }
    }
    revert UnsupportedAction(action);
}
```

V4Router also uses the operation types defined in [ActionsLibrary](./ActionsLibrary.md).

#### SWAP_EXACT_IN

Specify the exact input token amount and swap path, complete a multi-hop swap, and calculate the output token:

```solidity
IV4Router.ExactInputParams calldata swapParams = params.decodeSwapExactInParams();
_swapExactInput(swapParams);
return;
```

Decode the [ExactInputParams](#exactinputparams) type parameters and call the [_swapExactInput](#_swapexactinput) method to complete the exact input multi-hop swap.

#### SWAP_EXACT_IN_SINGLE

Specify the exact input token amount and pool, complete a single-hop swap, and calculate the output token:

```solidity
IV4Router.ExactInputSingleParams calldata swapParams = params.decodeSwapExactInSingleParams();
_swapExactInputSingle(swapParams);
return;
```

Decode the [ExactInputSingleParams](#exactinputsingleparams) type parameters and call the [_swapExactInputSingle](#_swapexactinputsingle) method to complete the exact input single-hop swap.

#### SWAP_EXACT_OUT

Specify the exact output token amount and swap path, complete a multi-hop swap, and calculate the input token:

```solidity
IV4Router.ExactOutputParams calldata swapParams = params.decodeSwapExactOutParams();
_swapExactOutput(swapParams);
return;
```

Decode the [ExactOutputParams](#exactoutputparams) type parameters and call the [_swapExactOutput](#_swapexactoutput) method to complete the exact output multi-hop swap.

#### SWAP_EXACT_OUT_SINGLE

Specify the exact output token amount and pool, complete a single-hop swap, and calculate the input token:

```solidity
IV4Router.ExactOutputSingleParams calldata swapParams = params.decodeSwapExactOutSingleParams();
_swapExactOutputSingle(swapParams);
return;
```

Decode the [ExactOutputSingleParams](#exactoutputsingleparams) type parameters and call the [_swapExactOutputSingle](#_swapexactoutputsingle) method to complete the exact output single-hop swap.

#### SETTLE_ALL

Settle all debts (negative delta) of the specified token in `poolManager`.

```solidity
(Currency currency, uint256 maxAmount) = params.decodeCurrencyAndUint256();
uint256 amount = _getFullDebt(currency);
if (amount > maxAmount) revert V4TooMuchRequested(maxAmount, amount);
_settle(currency, msgSender(), amount);
return;
```

Decode the parameters and call the [_getFullDebt](./DeltaResolver.md#_getfulldebt) method to get the full debt (negative delta) of the specified token.

If the specified maximum repayment amount `maxAmount` is less than the debt, throw an exception.

Call the [_settle](./DeltaResolver.md#_settle) method to settle all debts. The `payer` is the caller, and `fullDebt` is the repayment amount.

#### TAKE_ALL

Withdraw all credits (positive delta) of the specified token in `poolManager`.

```solidity
(Currency currency, uint256 minAmount) = params.decodeCurrencyAndUint256();
uint256 amount = _getFullCredit(currency);
if (amount < minAmount) revert V4TooLittleReceived(minAmount, amount);
_take(currency, msgSender(), amount);
return;
```

Decode the parameters and call the [_getFullCredit](./DeltaResolver.md#_getfullcredit) method to get the full credit (positive delta) of the specified token.

If the specified minimum withdrawal amount `minAmount` is greater than the credit, throw an exception.

Call the [_take](./DeltaResolver.md#_take) method to withdraw all credits. The `recipient` is the caller, and `fullCredit` is the withdrawal amount.

#### SETTLE

Settle part of the debts (negative delta) of the specified token in `poolManager`.

```solidity
(Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
_settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
return;
```

Decode the parameters, call the [_mapPayer](./BaseActionsRouter.md#_mappayer) method to determine the payer, and call the [_mapSettleAmount](./DeltaResolver.md#_mapsettleamount) method to calculate the settlement amount.

Call the [_settle](./DeltaResolver.md#_settle) method to settle the specified debts.

#### TAKE

Withdraw part of the credits (positive delta) of the specified token in `poolManager`.

```solidity
(Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
return;
```

Decode the parameters, call the [_mapRecipient](./BaseActionsRouter.md#_maprecipient) method to determine the recipient, and call the [_mapTakeAmount](./DeltaResolver.md#_maptakeamount) method to calculate the withdrawal amount.

Call the [_take](./DeltaResolver.md#_take) method to withdraw the specified credits.

#### TAKE_PORTION

Withdraw part of the credits (positive delta) of the specified token in `poolManager`. The withdrawal ratio is specified by `bips`. The upper limit is `10000`, which is `100%`.

Similar to the logic of [TAKE](#take), except that the withdrawal amount is calculated by the ratio `bips`.

```solidity
(Currency currency, address recipient, uint256 bips) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _getFullCredit(currency).calculatePortion(bips));
return;
```

Decode the parameters, call the [_mapRecipient](./BaseActionsRouter.md#_maprecipient) method to determine the recipient, and call the `calculatePortion` method to calculate the withdrawal amount, with a maximum of `10000`.

Call the [_take](./DeltaResolver.md#_take) method to withdraw the specified credits.

### _swapExactInput

Complete an exact input multi-hop swap.

Specify the input token amount and swap path, and complete the multi-hop swap sequentially, calculating the output token.

```solidity
function _swapExactInput(IV4Router.ExactInputParams calldata params) private {
    unchecked {
        // Caching for gas savings
        uint256 pathLength = params.path.length;
        uint128 amountOut;
        Currency currencyIn = params.currencyIn;
        uint128 amountIn = params.amountIn;
        if (amountIn == ActionConstants.OPEN_DELTA) amountIn = _getFullCredit(currencyIn).toUint128();
        PathKey calldata pathKey;

        for (uint256 i = 0; i < pathLength; i++) {
            pathKey = params.path[i];
            (PoolKey memory poolKey, bool zeroForOne) = pathKey.getPoolAndSwapDirection(currencyIn);
            // The output delta will always be positive, except for when interacting with certain hook pools
            amountOut = _swap(poolKey, zeroForOne, -int256(uint256(amountIn)), pathKey.hookData).toUint128();

            amountIn = amountOut;
            currencyIn = pathKey.intermediateCurrency;
        }

        if (amountOut < params.amountOutMinimum) revert V4TooLittleReceived(params.amountOutMinimum, amountOut);
    }
}
```

If the input token amount `amountIn` is `ActionConstants.OPEN_DELTA`, i.e., `0`, use the current contract's full credit (flash accounting balance) in `poolManager` as `amountIn`. Refer to the [_getFullCredit](./DeltaResolver.md#_getfullcredit) method.

Iterate through the swap path sequentially:

1. For each swap path, determine the swap pool and direction for this swap based on the [getPoolAndSwapDirection](./PathKeyLibrary.md#getpoolandswapdirection) method;
2. Call the [_swap](#_swap) method to complete the single-step swap. `amountSpecified` is negative, indicating exact input. The returned `amountOut` is the output token amount;
3. Use the output `amountOut` of this intermediate swap as the input `amountIn` for the next swap;
4. Use the intermediate token address `intermediateCurrency` of this intermediate swap as the input token address for the next swap to determine the next swap pool and direction.

After completing all swaps, the obtained `amountOut` is the amount of the target token.

Check whether `amountOut` is less than `params.amountOutMinimum`. If it is, throw an exception.

### _swapExactInputSingle

Complete an exact input single-hop swap.

* If `zeroForOne` is `true`, it means exact input of `token0` and output of `token1`;
* Otherwise, it means exact input of `token1` and output of `token0`.

```solidity
function _swapExactInputSingle(IV4Router.ExactInputSingleParams calldata params) private {
    uint128 amountIn = params.amountIn;
    if (amountIn == ActionConstants.OPEN_DELTA) {
        amountIn =
            _getFullCredit(params.zeroForOne ? params.poolKey.currency0 : params.poolKey.currency1).toUint128();
    }
    uint128 amountOut =
        _swap(params.poolKey, params.zeroForOne, -int256(uint256(amountIn)), params.hookData).toUint128();
    if (amountOut < params.amountOutMinimum) revert V4TooLittleReceived(params.amountOutMinimum, amountOut);
}
```

First, calculate the input token amount `amountIn`. If `amountIn` is `ActionConstants.OPEN_DELTA`, i.e., `0`, set it to the current contract's full credit (flash accounting balance) in `poolManager`. Determine whether to query the token address `currency0` or `currency1` based on `params.zeroForOne`.

Call the [_swap](#_swap) method to complete the single-step swap. `amountSpecified` is negative, indicating exact input. The returned `amountOut` is the output token amount.
Check whether `amountOut` is less than `params.amountOutMinimum`. If it is, throw an exception.

### _swapExactOutput

Complete an exact output multi-hop swap.

Specify the output token amount and swap path, and complete the multi-hop swap sequentially, calculating the input token.

```solidity
function _swapExactOutput(IV4Router.ExactOutputParams calldata params) private {
    unchecked {
        // Caching for gas savings
        uint256 pathLength = params.path.length;
        uint128 amountIn;
        uint128 amountOut = params.amountOut;
        Currency currencyOut = params.currencyOut;
        PathKey calldata pathKey;

        if (amountOut == ActionConstants.OPEN_DELTA) {
            amountOut = _getFullDebt(currencyOut).toUint128();
        }

        for (uint256 i = pathLength; i > 0; i--) {
            pathKey = params.path[i - 1];
            (PoolKey memory poolKey, bool oneForZero) = pathKey.getPoolAndSwapDirection(currencyOut);
            // The output delta will always be negative, except for when interacting with certain hook pools
            amountIn = (uint256(-int256(_swap(poolKey, !oneForZero, int256(uint256(amountOut)), pathKey.hookData))))
                .toUint128();

            amountOut = amountIn;
            currencyOut = pathKey.intermediateCurrency;
        }
        if (amountIn > params.amountInMaximum) revert V4TooMuchRequested(params.amountInMaximum, amountIn);
    }
}
```

If the output token amount `amountOut` is `ActionConstants.OPEN_DELTA`, i.e., `0`, set it to the current contract's full debt (negative delta) in `poolManager`. Refer to the [_getFullDebt](./DeltaResolver.md#_getfulldebt) method.

Since we need to calculate the input token based on the output token, iterate through the swap path in reverse order:

1. For each swap path, determine the swap pool and direction for this swap based on the [getPoolAndSwapDirection](./PathKeyLibrary.md#getpoolandswapdirection) method;
2. Call the [_swap](#_swap) method to complete the single-step swap. `amountSpecified` is positive, indicating exact output. The returned `amountIn` is the input token amount;
   * Since the returned `amountIn` is negative, indicating the input token amount, and in the next operation, it needs to be represented as the output token, i.e., positive, it needs to be negated
3. Use the input `amountIn` of this intermediate swap as the output `amountOut` for the next swap;
4. Use the intermediate token address `intermediateCurrency` of this intermediate swap as the output token address for the next swap.

After completing all swaps, the obtained `amountIn` is the amount of the target token.

Since `amountIn` and `params.amountInMaximum` are both positive, check whether `amountIn` is greater than `params.amountInMaximum`. If it is, throw an exception.

### _swapExactOutputSingle

Complete an exact output single-hop swap.

* If `zeroForOne` is `true`, it means exact output of `token1` and input of `token0`;
* Otherwise, it means exact output of `token0` and input of `token1`.

```solidity
function _swapExactOutputSingle(IV4Router.ExactOutputSingleParams calldata params) private {
    uint128 amountOut = params.amountOut;
    if (amountOut == ActionConstants.OPEN_DELTA) {
        amountOut =
            _getFullDebt(params.zeroForOne ? params.poolKey.currency1 : params.poolKey.currency0).toUint128();
    }
    uint128 amountIn = (
        uint256(-int256(_swap(params.poolKey, params.zeroForOne, int256(uint256(amountOut)), params.hookData)))
    ).toUint128();
    if (amountIn > params.amountInMaximum) revert V4TooMuchRequested(params.amountInMaximum, amountIn);
}
```

First, calculate the output token amount `amountOut`. If `amountOut` is `ActionConstants.OPEN_DELTA`, i.e., `0`, set it to the current contract's full debt (negative delta) in `poolManager`. Determine whether to query the token address `currency1` or `currency0` based on `params.zeroForOne`.

Call the [_swap](#_swap) method to complete the single-step swap. `amountSpecified` is positive, indicating exact output. The returned `amountIn` is negative, indicating the input token amount. Negate it to convert it to positive.

Check whether `amountIn` is greater than `params.amountInMaximum`. If it is, throw an exception.

### _swap

Complete a single-step swap.

Input parameters:

- `poolKey`: The key of the pool
- `zeroForOne`: Whether to swap from `token0` to `token1`
- `amountSpecified`: The specified token amount
  - If less than `0`, it indicates exact input
  - If greater than `0`, it indicates exact output
- `hookData`: Hook data for the `beforeSwap` and `afterSwap` callbacks of Hooks

```solidity
function _swap(PoolKey memory poolKey, bool zeroForOne, int256 amountSpecified, bytes calldata hookData)
    private
    returns (int128 reciprocalAmount)
{
    // for protection of exactOut swaps, sqrtPriceLimit is not exposed as a feature in this contract
    unchecked {
        BalanceDelta delta = poolManager.swap(
            poolKey,
            IPoolManager.SwapParams(
                zeroForOne, amountSpecified, zeroForOne ? TickMath.MIN_SQRT_PRICE + 1 : TickMath.MAX_SQRT_PRICE - 1
            ),
            hookData
        );

        reciprocalAmount = (zeroForOne == amountSpecified < 0) ? delta.amount1() : delta.amount0();
    }
}
```

Among them, the `IPoolManager.SwapParams` struct is defined as follows:

```solidity
struct SwapParams {
    /// Whether to swap token0 for token1 or vice versa
    bool zeroForOne;
    /// The desired input amount if negative (exactIn), or the desired output amount if positive (exactOut)
    int256 amountSpecified;
    /// The sqrt price at which, if reached, the swap will stop executing
    uint160 sqrtPriceLimitX96;
}
```

If `zeroForOne` is `true`, it means swapping from `token0` to `token1`. As the swap progresses, the amount of `token0` in the pool will increase, and the amount of `token1` will decrease. Therefore, $ \sqrt{P} $ = $ \sqrt{\frac{y}{x}} $ decreases, and the price limit needs to be smaller than the current price. Here, `sqrtPriceLimitX96` is set to `TickMath.MIN_SQRT_PRICE + 1`, which is the minimum price, indicating no price limit for the swap.

Similarly, if `zeroForOne` is `false`, set `sqrtPriceLimitX96` to `TickMath.MAX_SQRT_PRICE - 1`.

Although the price limit is not set here, in external call methods, it is possible to check whether `reciprocalAmount` exceeds the maximum/minimum value to ensure the safety of the swap.

Call the [poolManager.swap](../../v4-core/en/PoolManager.md#swap) method to complete the specific swap operation. The returned `delta` is a `BalanceDelta` struct, with the high 128 bits representing `amount0` and the low 128 bits representing `amount1`.

If `amountSpecified < 0`, it indicates `exactInput`, i.e., exact input. Combined with `zeroForOne`, the following combinations are possible:

| zeroForOne | amountSpecified < 0 | Description |
| --- | --- | --- |
| true | true | Exact input of `amount0`, calculate output of `amount1` |
| true | false | Exact output of `amount1`, calculate input of `amount0` |
| false | true | Exact input of `amount1`, calculate output of `amount0` |
| false | false | Exact output of `amount0`, calculate input of `amount1` |

Therefore, if `zeroForOne == amountSpecified < 0`, always return `delta.amount1()`, otherwise return `delta.amount0()`.
