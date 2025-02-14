# V4Router

与 [PositionManager](./PositionManager.md) 定位于头寸管理不同，V4Router 主要用于执行交易（swap），底层调用 [PoolManager](../../v4-core/zh/PoolManager.md) 合约完成具体的交易操作。

先来看一下 V4Router 合约的声明：

```solidity
/// @title UniswapV4Router
/// @notice Abstract contract that contains all internal logic needed for routing through Uniswap v4 pools
/// @dev the entry point to executing actions in this contract is calling `BaseActionsRouter._executeActions`
/// An inheriting contract should call _executeActions at the point that they wish actions to be executed
abstract contract V4Router is IV4Router, BaseActionsRouter, DeltaResolver {
```

与 [PositionManager](./PositionManager.md) 类似，V4Router 合约也继承了 `BaseActionsRouter` 和 `DeltaResolver` 合约，通过调用 `BaseActionsRouter._executeActions` 方法来批量执行操作。

V4Router 本身是一个抽象合约，因此不能直接部署，其它合约需要继承 V4Router 合约，并实现 `DeltaResolver` 合约中的 `pay` 方法：

```solidity
/// @notice Abstract function for contracts to implement paying tokens to the poolManager
/// @dev The recipient of the payment should be the poolManager
/// @param token The token to settle. This is known not to be the native currency
/// @param payer The address who should pay tokens
/// @param amount The number of tokens to send
function _pay(Currency token, address payer, uint256 amount) internal virtual;
```

`pay` 方法将指定数量的代币支付给 `poolManager`。

Uniswap v4 [universal-router](https://github.com/Uniswap/universal-router) 的 [V4SwapRouter.sol](https://github.com/Uniswap/universal-router/blob/8bd498a3fc9f8bc8577e626c024c4fcf0691f885/contracts/modules/uniswap/v4/V4SwapRouter.sol#L14) 合约继承了 V4Router 合约，并实现了 `pay` 方法。

## 结构体定义

在 IV4Router 接口中定义了一些 `swap` 方法常用的结构体：

### ExactInputSingleParams

指定池子的单跳交易的精确输入交换参数：

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

其中：

- `poolKey`：池子的 key
- `zeroForOne`：是否从 `token0` 交换到 `token1`
- `amountIn`：输入代币数量
- `amountOutMinimum`：最小输出代币数量
- `hookData`：Hook 数据

### ExactInputParams

多跳交易的精确输入交换参数：

```solidity
/// @notice Parameters for a multi-hop exact-input swap
struct ExactInputParams {
    Currency currencyIn;
    PathKey[] path;
    uint128 amountIn;
    uint128 amountOutMinimum;
}
```

其中：

- `currencyIn`：输入代币
- `path`：交换路径，包含中间代币的信息，参考 [PathKey](./PathKeyLibrary.md#pathkey) 结构体定义
- `amountIn`：输入代币数量
- `amountOutMinimum`：最小输出代币数量

### ExactOutputSingleParams

指定池子的单跳交易的精确输出交换参数：

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

其中：

- `poolKey`：池子的 key
- `zeroForOne`：是否从 `token0` 交换到 `token1`
- `amountOut`：输出代币数量
- `amountInMaximum`：最大输入代币数量
- `hookData`：Hook 数据

### ExactOutputParams

多跳交易的精确输出交换参数：

```solidity
/// @notice Parameters for a multi-hop exact-output swap
struct ExactOutputParams {
    Currency currencyOut;
    PathKey[] path;
    uint128 amountOut;
    uint128 amountInMaximum;
}
```

其中：

- `currencyOut`：输出代币
- `path`：交换路径
- `amountOut`：输出代币数量
- `amountInMaximum`：最大输入代币数量

## 方法定义

由于 V4Router 是一个抽象合约，它并没有提供直接的对外调用接口，而由继承它的合约来实现具体的交易入口。

### _handleAction

V4Router 合约主要实现了 `BaseActionsRouter._handleAction` 方法，用于处理不同类型的交易操作：

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

V4Router 同样使用了 [ActionsLibrary](./ActionsLibrary.md) 中定义的操作类型。

#### SWAP_EXACT_IN

指定精确的输入代币数量和交换路径，完成多跳交易，计算输出代币：

```solidity
IV4Router.ExactInputParams calldata swapParams = params.decodeSwapExactInParams();
_swapExactInput(swapParams);
return;
```

解码 [ExactInputParams](#exactinputparams) 类型参数，调用 [_swapExactInput](#_swapexactinput) 方法完成精确输入的多跳交易。

#### SWAP_EXACT_IN_SINGLE

指定精确的输入代币数量和池子，完成单跳交易，计算输出代币：

```solidity
IV4Router.ExactInputSingleParams calldata swapParams = params.decodeSwapExactInSingleParams();
_swapExactInputSingle(swapParams);
return;
```

解码 [ExactInputSingleParams](#exactinputsingleparams) 类型参数，调用 [_swapExactInputSingle](#_swapexactinputsingle) 方法完成精确输入的单跳交易。

#### SWAP_EXACT_OUT

指定精确的输出代币数量和交换路径，完成多跳交易，计算输入代币：

```solidity
IV4Router.ExactOutputParams calldata swapParams = params.decodeSwapExactOutParams();
_swapExactOutput(swapParams);
return;
```

解码 [ExactOutputParams](#exactoutputparams) 类型参数，调用 [_swapExactOutput](#_swapexactoutput) 方法完成精确输出的多跳交易。

#### SWAP_EXACT_OUT_SINGLE

指定精确的输出代币数量和池子，完成单跳交易，计算输入代币：

```solidity
IV4Router.ExactOutputSingleParams calldata swapParams = params.decodeSwapExactOutSingleParams();
_swapExactOutputSingle(swapParams);
return;
```

解码 [ExactOutputSingleParams](#exactoutputsingleparams) 类型参数，调用 [_swapExactOutputSingle](#_swapexactoutputsingle) 方法完成精确输出的单跳交易。

#### SETTLE_ALL

结算在 `poolManager` 中的指定代币的全部欠款（负 delta）。

```solidity
(Currency currency, uint256 maxAmount) = params.decodeCurrencyAndUint256();
uint256 amount = _getFullDebt(currency);
if (amount > maxAmount) revert V4TooMuchRequested(maxAmount, amount);
_settle(currency, msgSender(), amount);
return;
```

解码参数，调用 [_getFullDebt](./DeltaResolver.md#_getfulldebt) 方法获取指定代币的全部欠款（负 delta）。

如果指定的最大还款金额 `maxAmount` 小于欠款，则抛出异常。

调用 [_settle](./DeltaResolver.md#_settle) 方法结算所有欠款。`payer` 是调用方，`fullDebt` 作为还款金额。

#### TAKE_ALL

提取在 `poolManager` 中的指定代币的全部信用（正 delta）。

```solidity
(Currency currency, uint256 minAmount) = params.decodeCurrencyAndUint256();
uint256 amount = _getFullCredit(currency);
if (amount < minAmount) revert V4TooLittleReceived(minAmount, amount);
_take(currency, msgSender(), amount);
return;
```

解码参数，调用 [_getFullCredit](./DeltaResolver.md#_getfullcredit) 方法获取指定代币的全部信用（正 delta）。

如果指定的最小提取金额 `minAmount` 大于信用，则抛出异常。

调用 [_take](./DeltaResolver.md#_take) 方法提取所有信用。`recipient` 是调用方，`fullCredit` 作为提取金额。

#### SETTLE

结算在 `poolManager` 中的指定代币的部分欠款（负 delta）。

```solidity
(Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
_settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
return;
```

解码参数，调用 [_mapPayer](./BaseActionsRouter.md#_mappayer) 方法确定支付方，调用 [_mapSettleAmount](./DeltaResolver.md#_mapsettleamount) 方法计算结算金额。

调用 [_settle](./DeltaResolver.md#_settle) 方法结算指定欠款。

#### TAKE

提取在 `poolManager` 中的指定代币的部分信用（正 delta）。

```solidity
(Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
return;
```

解码参数，调用 [_mapRecipient](./BaseActionsRouter.md#_maprecipient) 方法确定接收方，调用 [_mapTakeAmount](./DeltaResolver.md#_maptakeamount) 方法计算提取金额。

调用 [_take](./DeltaResolver.md#_take) 方法提取指定信用。

#### TAKE_PORTION

提取在 `poolManager` 中的指定代币的部分信用（正 delta）。提取的比例由 `bips` 指定。上限是 `10000`，即 `100%`。

与 [TAKE](#take) 逻辑类似，只是提取的金额由比例 `bips` 计算。

```solidity
(Currency currency, address recipient, uint256 bips) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _getFullCredit(currency).calculatePortion(bips));
return;
```

解码参数，调用 [_mapRecipient](./BaseActionsRouter.md#_maprecipient) 方法确定接收方，调用 `calculatePortion` 方法计算提取金额，最高为 `10000`。

调用 [_take](./DeltaResolver.md#_take) 方法提取指定信用。

### _swapExactInput

完成精确输入的多跳交易。

指定输入代币数量和交换路径，依次完成多跳交易，计算输出代币。

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

如果输入代币数量 `amountIn` 为 `ActionConstants.OPEN_DELTA`，即 `0`，则使用当前合约在 `poolManager` 中的全部信用（闪电记账余额）作为 `amountIn`。参考 [_getFullCredit](./DeltaResolver.md#_getfullcredit) 方法。

依次遍历交换路径：

1. 对每个交易路径，根据 [getPoolAndSwapDirection](./PathKeyLibrary.md#getpoolandswapdirection) 方法，确定本次的交易池子和交易方向；
2. 调用 [_swap](#_swap) 方法，完成单步交易。`amountSpecified` 为负数，表示精确输入。返回的 `amountOut` 为输出代币数量；
3. 将本次中间交易的输出 `amountOut` 作为下一次交易的输入 `amountIn`；
4. 将本次中间交易的代币地址 `intermediateCurrency` 作为下一次交易的输入代币地址，用于确定下一个交易池子和方向。

完成所有交易后，获得的 `amountOut` 即为目标代币的数量。

判断 `amountOut` 是否小于 `params.amountOutMinimum`，如果小于，则抛出异常。

### _swapExactInputSingle

完成精确输入的单跳交易。

* 如果 `zeroForOne` 为 `true`，则表示精确输入 `token0`，输出 `token1`；
* 否则，表示精确输入 `token1`，输出 `token0`。

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

首先，计算输入代币数量 `amountIn`，如果 `amountIn` 为 `ActionConstants.OPEN_DELTA`，即 `0`，则将其设置为当前合约在 `poolManager` 中的全部信用（闪电记账余额）。根据 `params.zeroForOne` 确定查询的代币地址是 `currency0` 还是 `currency0`。

调用 [_swap](#_swap) 方法，完成单步交易。`amountSpecified` 为负数，表示精确输入。返回的 `amountOut` 为输出代币数量。
判断 `amountOut` 是否小于 `params.amountOutMinimum`，如果小于，则抛出异常。

### _swapExactOutput

完成精确输出的多跳交易。

指定输出代币数量和交换路径，依次完成多跳交易，计算输入代币。

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

如果输出代币数量 `amountOut` 为 `ActionConstants.OPEN_DELTA`，即 `0`，则将其设置为当前合约在 `poolManager` 中的全部欠款（负 delta）。参考 [_getFullDebt](./DeltaResolver.md#_getfulldebt) 方法。

由于我们需要根据输出代币计算输入代币，因此从最后一个代币开始，逆序依次遍历交换路径：

1. 对每个交易路径，根据 [getPoolAndSwapDirection](./PathKeyLibrary.md#getpoolandswapdirection) 方法，确定本次的交易池子和交易方向；
2. 调用 [_swap](#_swap) 方法，完成单步交易。`amountSpecified` 为正数，表示精确输出。返回的 `amountIn` 为输入代币数量；
   * 由于返回的 `amountIn` 为负数，表示输入代币数量，而在下一步操作中，需要将其表示为输出代币，即正数，因此需要进行取反操作
3. 将本次中间交易的输入 `amountIn` 作为下一次交易的输出 `amountOut`；
4. 将本次中间交易的代币地址 `intermediateCurrency` 作为下一次交易的输出代币地址。

完成所有交易后，获得的 `amountIn` 即为目标代币的数量。

由于 `amountIn` 和 `params.amountInMaximum` 都是正数，因此判断 `amountIn` 是否大于 `params.amountInMaximum`，如果大于，则抛出异常。

### _swapExactOutputSingle

完成精确输出的单跳交易。

* 如果 `zeroForOne` 为 `true`，则表示精确输出 `token1`，输入 `token0`；
* 否则，表示精确输出 `token0`，输入 `token1`。

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

首先，计算输出代币数量 `amountOut`，如果 `amountOut` 为 `ActionConstants.OPEN_DELTA`，即 `0`，则将其设置为当前合约在 `poolManager` 中的全部欠款（负 delta）。根据 `params.zeroForOne` 确定查询的代币地址是 `currency1` 还是 `currency0`。

调用 [_swap](#_swap) 方法，完成单步交易。`amountSpecified` 为正数，表示精确输出。返回的 `amountIn` 为负数，表示输入代币数量。对其执行取反操作，转换为正数。

判断 `amountIn` 是否大于 `params.amountInMaximum`，如果大于，则抛出异常。

### _swap

完成单步交换。

输入参数：

- `poolKey`：池子的 key
- `zeroForOne`：是否从 `token0` 交换到 `token1`
- `amountSpecified`：指定的代币数量
  - 如果小于 `0`，表示精确输入
  - 如果大于 `0`，表示精确输出
- `hookData`：Hook 数据，用于 Hooks 的 `beforeSwap` 和 `afterSwap` 回调

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

其中，`IPoolManager.SwapParams` 结构体定义如下：

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

如果 `zeroForOne` 为 `true`，表示从 `token0` 交换到 `token1`，随着交易的进行，池子中 `token0` 的数量会增加，`token1` 的数量会减少。因此，$ \sqrt{P} $ = $ \sqrt{\frac{y}{x}} $ 变小，价格限制需要比当前价格小，这里设置 `sqrtPriceLimitX96` 为 `TickMath.MIN_SQRT_PRICE + 1`，即最小价格，表示不限制交易的价格。

同理，如果 `zeroForOne` 为 `false`，设置 `sqrtPriceLimitX96` 为 `TickMath.MAX_SQRT_PRICE - 1`。

虽然这里没有设置价格上限，但是在外部调用方法中，可以判断 `reciprocalAmount` 是否超过最大值/最小值，从而确保交易的安全性。

调用 [poolManager.swap](../../v4-core/zh/PoolManager.md#swap) 方法，完成具体的交易操作。返回的 `delta` 为 `BalanceDelta` 结构体，高 128 位表示 `amount0`，低 128 位表示 `amount1`。

如果 `amountSpecified < 0`，则表示 `exactInput`，即精确输入，结合 `zeroForOne`，由以下组合：

| zeroForOne | amountSpecified < 0 | 说明 |
| --- | --- | --- |
| true | true | 精确输入 `amount0`，计算输出 `amount1` |
| true | false | 精确输出 `amount1`，计算输入 `amount0` |
| false | true | 精确输入 `amount1`，计算输出 `amount0` |
| false | false | 精确输出 `amount0`，计算输入 `amount1` |

因此，如果 `zeroForOne == amountSpecified < 0`，则总是需要返回 `delta.amount1()`，否则返回 `delta.amount0()`。
