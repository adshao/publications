# PositionManager

PositionManager 用于头寸管理，包括头寸的创建、修改流动性、删除等操作。

PositionManager 合约主要包括以下接口：

- [initializePool](#initializePool)：初始化池子
- [modifyLiquidities](#modifyliquidities)：修改流动性
- [modifyLiquiditiesWithoutUnlock](#modifyLiquiditiesWithoutUnlock)：修改流动性（不解锁）

根据 [Uniswap v4 workflow](../../assets/uniswap-v4-workflow.png) 图示，`modifyLiquidities` 和 `modifyLiquiditiesWithoutUnlock` 都将通过 [BaseActionsRouter._executeActionsWithoutUnlock](./BaseActionsRouter.md#_executeActionsWithoutUnlock) 来执行 [_handleAction](#_handleAction) 方法，该方法根据用户传入的不同操作类型，来执行具体的操作。

PositionManager 主要包括以下两类操作：

* 修改流动性操作
  - [IncreaseLiquidity](#increase_liquidity)：增加流动性
  - [IncreaseLiquidityFromDelta](#increase_liquidity_from_deltas)：使用闪电记账余额增加流动性
  - [DecreaseLiquidity](#decrease_liquidity)：减少流动性
  - [MintPosition](#mint_position)：创建头寸
  - [MintPositionFromDelta](#mint_position_from_deltas)：使用闪电记账余额创建头寸
  - [BurnPosition](#burn_position)：销毁头寸

* 结算余额操作
  - [SettlePair](#settle_pair)：结算交易对的欠款，调用方向 `PoolManager` 合约支付代币
  - [TakePair](#take_pair)：提取交易对的余额
  - [Settle](#settle)：结算单个代币的欠款，向 `PoolManager` 合约支付代币
  - [Take](#take)：提取单个代币的余额
  - [CloseCurrency](#close_currency)：结算或提取单个代币的余额
  - [ClearOrTake](#clear_or_take)：放弃或提取单个代币的余额
  - [Sweep](#sweep)：从 PositionManager 提取单个代币的余额

## 方法定义

### modifyLiquidities

PositionManager 合约的设计思想是基于命令行模式，它本身并不提供具体的操作入口，而是让用户组合不同的操作，并调用 [modifyLiquidities](#modifyLiquidities)，在一次调用中按顺序执行所有的操作。

该方法是 PositionManager 的标准入口。

```solidity
/// @inheritdoc IPositionManager
function modifyLiquidities(bytes calldata unlockData, uint256 deadline)
    external
    payable
    isNotLocked
    checkDeadline(deadline)
{
    _executeActions(unlockData);
}
```

`PositionManager` 继承 `BaseActionsRouter`，`modifyLiquidities` 仅检查 `deadline`，并调用 [_executeActions](./BaseActionsRouter.md#_executeActions) 方法。

### modifyLiquiditiesWithoutUnlock

`modifyLiquiditiesWithoutUnlock` 方法与 `modifyLiquidities` 方法类似，但不执行解锁操作。

调用该方法的合约需要确保已调用 [PoolManager.unlock](../../v4-core/zh/PoolManager.md#unlock) 方法解锁。

```solidity
/// @inheritdoc IPositionManager
function modifyLiquiditiesWithoutUnlock(bytes calldata actions, bytes[] calldata params)
    external
    payable
    isNotLocked
{
    _executeActionsWithoutUnlock(actions, params);
}
```

### _handleAction

所有继承 `BaseActionsRouter` 的合约都需要实现 `_handleAction` 方法，该方法根据用户传入的不同操作类型，来执行具体的操作。

```solidity
function _handleAction(uint256 action, bytes calldata params) internal virtual override {
    if (action < Actions.SETTLE) {
        if (action == Actions.INCREASE_LIQUIDITY) {
            (uint256 tokenId, uint256 liquidity, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
                params.decodeModifyLiquidityParams();
            _increase(tokenId, liquidity, amount0Max, amount1Max, hookData);
            return;
        } else if (action == Actions.INCREASE_LIQUIDITY_FROM_DELTAS) {
            (uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
                params.decodeIncreaseLiquidityFromDeltasParams();
            _increaseFromDeltas(tokenId, amount0Max, amount1Max, hookData);
            return;
        } else if (action == Actions.DECREASE_LIQUIDITY) {
            (uint256 tokenId, uint256 liquidity, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
                params.decodeModifyLiquidityParams();
            _decrease(tokenId, liquidity, amount0Min, amount1Min, hookData);
            return;
        } else if (action == Actions.MINT_POSITION) {
            (
                PoolKey calldata poolKey,
                int24 tickLower,
                int24 tickUpper,
                uint256 liquidity,
                uint128 amount0Max,
                uint128 amount1Max,
                address owner,
                bytes calldata hookData
            ) = params.decodeMintParams();
            _mint(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, _mapRecipient(owner), hookData);
            return;
        } else if (action == Actions.MINT_POSITION_FROM_DELTAS) {
            (
                PoolKey calldata poolKey,
                int24 tickLower,
                int24 tickUpper,
                uint128 amount0Max,
                uint128 amount1Max,
                address owner,
                bytes calldata hookData
            ) = params.decodeMintFromDeltasParams();
            _mintFromDeltas(poolKey, tickLower, tickUpper, amount0Max, amount1Max, _mapRecipient(owner), hookData);
            return;
        } else if (action == Actions.BURN_POSITION) {
            // Will automatically decrease liquidity to 0 if the position is not already empty.
            (uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
                params.decodeBurnParams();
            _burn(tokenId, amount0Min, amount1Min, hookData);
            return;
        }
    } else {
        if (action == Actions.SETTLE_PAIR) {
            (Currency currency0, Currency currency1) = params.decodeCurrencyPair();
            _settlePair(currency0, currency1);
            return;
        } else if (action == Actions.TAKE_PAIR) {
            (Currency currency0, Currency currency1, address recipient) = params.decodeCurrencyPairAndAddress();
            _takePair(currency0, currency1, _mapRecipient(recipient));
            return;
        } else if (action == Actions.SETTLE) {
            (Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
            _settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE) {
            (Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
            return;
        } else if (action == Actions.CLOSE_CURRENCY) {
            Currency currency = params.decodeCurrency();
            _close(currency);
            return;
        } else if (action == Actions.CLEAR_OR_TAKE) {
            (Currency currency, uint256 amountMax) = params.decodeCurrencyAndUint256();
            _clearOrTake(currency, amountMax);
            return;
        } else if (action == Actions.SWEEP) {
            (Currency currency, address to) = params.decodeCurrencyAndAddress();
            _sweep(currency, _mapRecipient(to));
            return;
        } else if (action == Actions.WRAP) {
            uint256 amount = params.decodeUint256();
            _wrap(_mapWrapUnwrapAmount(CurrencyLibrary.ADDRESS_ZERO, amount, Currency.wrap(address(WETH9))));
            return;
        } else if (action == Actions.UNWRAP) {
            uint256 amount = params.decodeUint256();
            _unwrap(_mapWrapUnwrapAmount(Currency.wrap(address(WETH9)), amount, CurrencyLibrary.ADDRESS_ZERO));
            return;
        }
    }
    revert UnsupportedAction(action);
}
```

由于 [ActionsLibrary](./ActionsLibrary.md) 中按照数值顺序定义了所有的 `action`：

* 小于 `Actions.SETTLE` 的操作是修改流动性操作
* 大于等于 `Actions.SETTLE` 的操作是结算余额操作

以下介绍每个 Action 的逻辑：

#### INCREASE_LIQUIDITY

增加流动性。

```solidity
(uint256 tokenId, uint256 liquidity, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
    params.decodeModifyLiquidityParams();
_increase(tokenId, liquidity, amount0Max, amount1Max, hookData);
return;
```

解码参数，调用 [_increase](#_increase) 方法。

#### INCREASE_LIQUIDITY_FROM_DELTAS

使用闪电记账余额增加流动性。

```solidity
(uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
    params.decodeIncreaseLiquidityFromDeltasParams();
_increaseFromDeltas(tokenId, amount0Max, amount1Max, hookData);
return;
```

解码参数，调用 [_increaseFromDeltas](#_increasefromdeltas) 方法。

#### DECREASE_LIQUIDITY

减少流动性。

```solidity
(uint256 tokenId, uint256 liquidity, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
    params.decodeModifyLiquidityParams();
_decrease(tokenId, liquidity, amount0Min, amount1Min, hookData);
return;
```

解码参数，调用 [_decrease](#_decrease) 方法。

#### MINT_POSITION

创建头寸。

```solidity
(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint256 liquidity,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) = params.decodeMintParams();
_mint(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, _mapRecipient(owner), hookData);
return;
```

解码参数，调用 [_mint](#_mint) 方法。

##### _mapRecipient

其中，`_mapRecipient` 方法用于将 `owner` 转换为 `recipient`：
* 如果 `owner` 为 `address(1)`，则表示 `msgSender()`
* 如果 `owner` 为 `address(2）`，则表示 `PositionManager` 合约地址
* 否则，使用 `owner` 地址本身

#### MINT_POSITION_FROM_DELTAS

使用闪电记账余额创建头寸。

```solidity
(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) = params.decodeMintFromDeltasParams();
_mintFromDeltas(poolKey, tickLower, tickUpper, amount0Max, amount1Max, _mapRecipient(owner), hookData);
return;
```

解码参数，调用 [_mintFromDeltas](#_mintfromdeltas) 方法。

这里同样适用 [_mapRecipient](#_maprecipient) 方法计算 `owner` 地址。

#### BURN_POSITION

销毁头寸。

```solidity
// Will automatically decrease liquidity to 0 if the position is not already empty.
(uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
    params.decodeBurnParams();
_burn(tokenId, amount0Min, amount1Min, hookData);
return;
```

解码参数，调用 [_burn](#_burn) 方法。

#### SETTLE_PAIR

结算交易对的余额，调用方向 `PoolManager` 合约支付代币。

```solidity
(Currency currency0, Currency currency1) = params.decodeCurrencyPair();
_settlePair(currency0, currency1);
return;
```

解码参数，调用 [_settlePair](#_settlepair) 方法。

#### TAKE_PAIR

提取交易对的余额。

```solidity
(Currency currency0, Currency currency1, address recipient) = params.decodeCurrencyPairAndAddress();
_takePair(currency0, currency1, _mapRecipient(recipient));
return;
```

解码参数，调用 [_takePair](#_takepair) 方法。

使用 [_mapRecipient](#_maprecipient) 方法计算接收代币的 `recipient` 地址。

#### SETTLE

结算单个代币的余额，调用方向 `PoolManager` 合约支付代币。

```solidity
(Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
_settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
return;
```

解码参数，调用 [_settle](#_settle) 方法。

##### _mapPayer

其中，`_mapPayer` 方法根据 `payerIsUser` 来确定支付方：
* 如果 `payerIsUser` 为 `true`，则支付方为 `msgSender()`，即调用方
* 否则，支付方为 `PositionManager` 合约地址

##### _mapSettleAmount

`_mapSettleAmount` 方法根据 `amount` 和 `currency` 计算结算的代币数量：
* 如果 `amount` 为 `0`，则表示结算所有欠款
* 否则，表示结算指定 `amount` 数量的代币

#### TAKE

提取单个代币的余额。

```solidity
(Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
return;
```

解码参数，调用 [_take](#_take) 方法。

其中，使用 [_mapRecipient](#_maprecipient) 方法计算接收代币的 `recipient` 地址。
使用 [_mapTakeAmount](#_maptakeamount) 方法计算提取的代币数量。

##### _mapTakeAmount

`_mapTakeAmount` 方法根据 `amount` 和 `currency` 计算提取的代币数量：
* 如果 `amount` 为 `0`，则表示提取所有余额
* 否则，表示提取指定 `amount` 数量的代币

#### CLOSE_CURRENCY

结算或提取单个代币的余额。

```solidity
Currency currency = params.decodeCurrency();
_close(currency);
return;
```

解码参数，调用 [_close](#_close) 方法。

#### CLEAR_OR_TAKE

放弃或提取单个代币的余额。

```solidity
(Currency currency, uint256 amountMax) = params.decodeCurrencyAndUint256();
_clearOrTake(currency, amountMax);
return;
```

解码参数，调用 [_clearOrTake](#_clearortake) 方法。

#### SWEEP

从 `PositionManager` 提取单个代币的余额。

```solidity
(Currency currency, address to) = params.decodeCurrencyAndAddress();
_sweep(currency, _mapRecipient(to));
return;
```

解码参数，调用 [_sweep](#_sweep) 方法。

其中，使用 [_mapRecipient](#_maprecipient) 方法计算接收代币的 `to` 地址。

#### WRAP

包装代币，将 `PositionManager` 合约的原生 `ETH` 包装成 `WETH`。

```solidity
uint256 amount = params.decodeUint256();
_wrap(_mapWrapUnwrapAmount(CurrencyLibrary.ADDRESS_ZERO, amount, Currency.wrap(address(WETH9))));
return;
```

解码参数，调用 [_wrap](#_wrap) 方法。

使用 [_mapWrapUnwrapAmount](#_mapwrapunwrapamount) 方法计算包装的代币数量。

##### _mapWrapUnwrapAmount

计算包装/解包的代币数量。

输入参数：

- `inputCurrency`：输入的代币，可以是原生代币或包装代币
- `amount`：包装/解包的数量，可以是 `CONTRACT_BALANCE`、`OPEN_DELTA` 或具体数量
- `outputCurrency`：包装/解包后的代币，用户在 `PoolManager` 上可能欠费的代币

```solidity
/// @notice Calculates the sanitized amount before wrapping/unwrapping.
/// @param inputCurrency The currency, either native or wrapped native, that this contract holds
/// @param amount The amount to wrap or unwrap. Can be CONTRACT_BALANCE, OPEN_DELTA or a specific amount
/// @param outputCurrency The currency after the wrap/unwrap that the user may owe a balance in on the poolManager
function _mapWrapUnwrapAmount(Currency inputCurrency, uint256 amount, Currency outputCurrency)
    internal
    view
    returns (uint256)
{
    // if wrapping, the balance in this contract is in ETH
    // if unwrapping, the balance in this contract is in WETH
    uint256 balance = inputCurrency.balanceOf(address(this));
    if (amount == ActionConstants.CONTRACT_BALANCE) {
        // return early to avoid unnecessary balance check
        return balance;
    }
    if (amount == ActionConstants.OPEN_DELTA) {
        // if wrapping, the open currency on the PoolManager is WETH.
        // if unwrapping, the open currency on the PoolManager is ETH.
        // note that we use the DEBT amount. Positive deltas can be taken and then wrapped.
        amount = _getFullDebt(outputCurrency);
    }
    if (amount > balance) revert InsufficientBalance();
    return amount;
}
```

获取 `PositionManager` 合约的 `inputCurrency` 代币余额。

如果 `amount` 为 `1 << 255`，则表示使用代币余额作为包装/解包的数量。

如果 `amount` 为 `0`，则表示使用 `PositionManager` 在 `PoolManager` 上 `outputCurrency` 的欠款作为包装/解包的数量。
确保 `inputCurrency` 代币余额大于欠款数量。

#### UNWRAP

解包代币，将 `PositionManager` 合约的 `WETH` 解包成原生 `ETH`。

```solidity
uint256 amount = params.decodeUint256();
_unwrap(_mapWrapUnwrapAmount(Currency.wrap(address(WETH9)), amount, CurrencyLibrary.ADDRESS_ZERO));
return;
```

解码参数，调用 [_unwrap](#_unwrap) 方法。

使用 [_mapWrapUnwrapAmount](#_mapwrapunwrapamount) 方法计算解包的代币数量。

### _increase

增加头寸的流动性。

输入参数：

- `tokenId`：头寸 ID，即 PositionManager 分配的 ERC721 代币 ID
- `liquidity`：增加的流动性
- `amount0Max`：最大提供的 `token0` 数量
- `amount1Max`：最大提供的 `token1` 数量
- `hookData`：Hook 数据，用于传入 `beforeModifyLiquidity` 以及 `afterModifyLiquidity` Hooks 函数

```solidity
/// @dev Calling increase with 0 liquidity will credit the caller with any underlying fees of the position
function _increase(
    uint256 tokenId,
    uint256 liquidity,
    uint128 amount0Max,
    uint128 amount1Max,
    bytes calldata hookData
) internal onlyIfApproved(msgSender(), tokenId) {
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    // Note: The tokenId is used as the salt for this position, so every minted position has unique storage in the pool manager.
    (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
        _modifyLiquidity(info, poolKey, liquidity.toInt256(), bytes32(tokenId), hookData);
    // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
    (liquidityDelta - feesAccrued).validateMaxIn(amount0Max, amount1Max);
}
```

通过 `getPoolAndPositionInfo` 获取池子信息和头寸信息。

调用 [_modifyLiquidity](#_modifyLiquidity) 方法，完成流动性修改。返回 `liquidityDelta` 和 `feesAccrued`。其中：

* `liquidityDelta` 是需要调用方结算的流动性变化值，均为非正数；
* `feesAccrued` 是截止到当前，头寸可领取的手续费，均为非负数。

由于 `liquidityDelta` 已经包含了 `feeAccrued`，而 `amount0Max` 和 `amount1Max` 是不包含手续费的（仅计算流动性本身），因此需要将 `liquidityDelta - feesAccrued` 作为输入参数进行最大值检查。

> 注：`liquidityDelta` 是调用方需要支付的最终代币数量 `amount0` 和 `amount1`，用负数表示

### _increaseFromDeltas

该方法与 [_increase](#_increase) 方法类似，不同之处在于 `_increaseFromDeltas` 无需指定具体的流动性数量，而是通过查询当前 `PositionManager` 合约在 `PoolManager` 中关于池子两种代币的闪电记账余额，然后计算出需要增加的流动性数量。

最后再调用 [_modifyLiquidity](#_modifyLiquidity) 方法，完成流动性修改。

```solidity
/// @dev The liquidity delta is derived from open deltas in the pool manager.
function _increaseFromDeltas(uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData)
    internal
    onlyIfApproved(msgSender(), tokenId)
{
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    uint256 liquidity;
    {
        (uint160 sqrtPriceX96,,,) = poolManager.getSlot0(poolKey.toId());

        // Use the credit on the pool manager as the amounts for the mint.
        liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96,
            TickMath.getSqrtPriceAtTick(info.tickLower()),
            TickMath.getSqrtPriceAtTick(info.tickUpper()),
            _getFullCredit(poolKey.currency0),
            _getFullCredit(poolKey.currency1)
        );
    }

    // Note: The tokenId is used as the salt for this position, so every minted position has unique storage in the pool manager.
    (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
        _modifyLiquidity(info, poolKey, liquidity.toInt256(), bytes32(tokenId), hookData);
    // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
    (liquidityDelta - feesAccrued).validateMaxIn(amount0Max, amount1Max);
}
```

`_getFullCredit` 方法确保 `PositionManager` 在 `PoolManager` 中的闪电记账余额一定是正数，即拥有可取回的代币，否则将回滚交易。

### _decrease

减少头寸的流动性。

输入参数：

- `tokenId`：头寸 ID，即 PositionManager 分配的 ERC721 代币 ID
- `liquidity`：减少的流动性
- `amount0Min`：最小获得的 `token0` 数量
- `amount1Min`：最小获得的 `token1` 数量
- `hookData`：Hook 数据，用于传入 `beforeModifyLiquidity` 以及 `afterModifyLiquidity` Hooks 函数

```solidity
/// @dev Calling decrease with 0 liquidity will credit the caller with any underlying fees of the position
function _decrease(
    uint256 tokenId,
    uint256 liquidity,
    uint128 amount0Min,
    uint128 amount1Min,
    bytes calldata hookData
) internal onlyIfApproved(msgSender(), tokenId) {
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    // Note: the tokenId is used as the salt.
    (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
        _modifyLiquidity(info, poolKey, -(liquidity.toInt256()), bytes32(tokenId), hookData);
    // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
    (liquidityDelta - feesAccrued).validateMinOut(amount0Min, amount1Min);
}
```

通过 `getPoolAndPositionInfo` 获取池子信息和头寸信息。

调用 [_modifyLiquidity](#_modifyLiquidity) 方法，完成流动性修改。返回 `liquidityDelta` 和 `feesAccrued`。其中：

* `liquidityDelta` 是需要调用方提取的流动性变化值，均为非负数；
* `feesAccrued` 是截止到目前，头寸可领取的手续费，均为非负数。

由于 `liquidityDelta` 已经包含了 `feeAccrued`，而 `amount0Min` 和 `amount1Min` 是不包含手续费的（仅计算流动性本身），因此需要将 `liquidityDelta - feesAccrued` 作为输入参数进行最小值检查。

> 注：`liquidityDelta` 是调用方可提取的最终代币数量，用正数或 0 表示

由于 Uniswap v4 没有提供直接提取手续费的方法，因此可以通过调用 `DECRESAE_LIQUIDITY` 操作来提取手续费，即将 `liquidity`、`amount0Min` 和 `amount1Min` 设置为 0。

### _mint

创建头寸。

输入参数：

- `poolKey`：池子的 Key
- `tickLower`：头寸的下限 Tick
- `tickUpper`：头寸的上限 Tick
- `liquidity`：创建的流动性
- `amount0Max`：最大提供的 `token0` 数量
- `amount1Max`：最大提供的 `token1` 数量
- `owner`：头寸的所有者
- `hookData`：Hook 数据，用于传入 `beforeModifyLiquidity` 以及 `afterModifyLiquidity` Hooks 函数

```solidity
function _mint(
        PoolKey calldata poolKey,
        int24 tickLower,
        int24 tickUpper,
        uint256 liquidity,
        uint128 amount0Max,
        uint128 amount1Max,
        address owner,
        bytes calldata hookData
    ) internal {
        // mint receipt token
        uint256 tokenId;
        // tokenId is assigned to current nextTokenId before incrementing it
        unchecked {
            tokenId = nextTokenId++;
        }
        _mint(owner, tokenId);

        // Initialize the position info
        PositionInfo info = PositionInfoLibrary.initialize(poolKey, tickLower, tickUpper);
        positionInfo[tokenId] = info;

        // Store the poolKey if it is not already stored.
        // On UniswapV4, the minimum tick spacing is 1, which means that if the tick spacing is 0, the pool key has not been set.
        bytes25 poolId = info.poolId();
        if (poolKeys[poolId].tickSpacing == 0) {
            poolKeys[poolId] = poolKey;
        }

        // fee delta can be ignored as this is a new position
        (BalanceDelta liquidityDelta,) =
            _modifyLiquidity(info, poolKey, liquidity.toInt256(), bytes32(tokenId), hookData);
        liquidityDelta.validateMaxIn(amount0Max, amount1Max);
    }
```

分配 ERC721 代币 ID `tokenId`，调用 `_mint` 方法，为 ``owner` 铸造 ERC721 代币，代表头寸。

初始化头寸信息。`positionInfo` 保存 `tokenId` 到 头寸信息的映射。
如果 `poolKey` 未存储，存储 `poolKey`。
因此，后续可以使用 `tokenId` 获取池子信息 `poolKey` 和 头寸信息 `positionInfo`。

调用 [_modifyLiquidity](#_modifyLiquidity) 方法，完成流动性修改。返回 `liquidityDelta` 和 `feesAccrued`。由于此时 `freeAccrued` 为 0，因此可以忽略。检查 `liquidityDelta` 是否超过 `amount0Max` 和 `amount1Max`。

### _mintFromDeltas

使用闪电记账余额创建头寸。

输入参数：

- `poolKey`：池子的 Key
- `tickLower`：头寸的下限 Tick
- `tickUpper`：头寸的上限 Tick
- `amount0Max`：最大提供的 `token0` 数量
- `amount1Max`：最大提供的 `token1` 数量
- `owner`：头寸的所有者
- `hookData`：Hook 数据，用于传入 `beforeModifyLiquidity` 以及 `afterModifyLiquidity` Hooks 函数

与 `_mint` 方法类似，不同之处在于 `_mintFromDeltas` 无需指定具体的流动性数量，而是通过查询当前 `PositionManager` 合约在 `PoolManager` 中关于池子两种代币的闪电记账余额，然后计算出需要增加的流动性数量。

最后再调用 [_mint](#_mint) 方法，创建头寸。

```solidity
function _mintFromDeltas(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) internal {
    (uint160 sqrtPriceX96,,,) = poolManager.getSlot0(poolKey.toId());

    // Use the credit on the pool manager as the amounts for the mint.
    uint256 liquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtPriceX96,
        TickMath.getSqrtPriceAtTick(tickLower),
        TickMath.getSqrtPriceAtTick(tickUpper),
        _getFullCredit(poolKey.currency0),
        _getFullCredit(poolKey.currency1)
    );

    _mint(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, owner, hookData);
}
```

### _burn

销毁头寸。

输入参数：

- `tokenId`：头寸 ID，即 PositionManager 分配的 ERC721 代币 ID
- `amount0Min`：最小获得的 `token0` 数量
- `amount1Min`：最小获得的 `token1` 数量
- `hookData`：Hook 数据，用于传入 `beforeModifyLiquidity` 以及 `afterModifyLiquidity` Hooks 函数

```solidity
/// @dev this is overloaded with ERC721Permit_v4._burn
function _burn(uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData)
    internal
    onlyIfApproved(msgSender(), tokenId)
{
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    uint256 liquidity = uint256(_getLiquidity(tokenId, poolKey, info.tickLower(), info.tickUpper()));

    address owner = ownerOf(tokenId);

    // Clear the position info.
    positionInfo[tokenId] = PositionInfoLibrary.EMPTY_POSITION_INFO;
    // Burn the token.
    _burn(tokenId);

    // Can only call modify if there is non zero liquidity.
    BalanceDelta feesAccrued;
    if (liquidity > 0) {
        BalanceDelta liquidityDelta;
        // do not use _modifyLiquidity as we do not need to notify on modification for burns.
        IPoolManager.ModifyLiquidityParams memory params = IPoolManager.ModifyLiquidityParams({
            tickLower: info.tickLower(),
            tickUpper: info.tickUpper(),
            liquidityDelta: -(liquidity.toInt256()),
            salt: bytes32(tokenId)
        });
        (liquidityDelta, feesAccrued) = poolManager.modifyLiquidity(poolKey, params, hookData);
        // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
        (liquidityDelta - feesAccrued).validateMinOut(amount0Min, amount1Min);
    }

    // deletes then notifies the subscriber
    if (info.hasSubscriber()) _removeSubscriberAndNotifyBurn(tokenId, owner, info, liquidity, feesAccrued);
}
```

通过 `getPoolAndPositionInfo` 获取池子信息和头寸信息。

通过 `PoolManager` 查询当前头寸的流动性 `liquidity`。

将头寸信息清空，销毁 ERC721 代币。

如果流动性大于 0，调用 [poolManager.modifyLiquidity](../../v4-core/zh/PoolManager.md#modifyliquidity) 方法，移除该头寸的所有流动性。返回 `liquidityDelta` 和 `feesAccrued`。

由于 `liquidityDelta` 已经包含了 `feeAccrued`，而 `amount0Min` 和 `amount1Min` 是不包含手续费的（仅计算流动性本身），因此需要将 `liquidityDelta - feesAccrued` 作为输入参数进行最小值检查。

如果头寸有订阅者，调用 [_removeSubscriberAndNotifyBurn](#_removeSubscriberAndNotifyBurn) 方法，通知订阅者。

注意，在完成头寸销毁后，调用方并没有收到任何代币，这些代币记录在 `PoolManager` 的闪电记账余额中；调用方需要结合 [TAKE_PAIR](#TAKE_PAIR) 等操作，将这些代币提取到自己的账户中。

### _settlePair

结算交易对的欠款。需要调用方向 `PoolManager` 支付代币。

输入参数：

- `currency0`：代币 0
- `currency1`：代币 1

```solidity
function _settlePair(Currency currency0, Currency currency1) internal {
    // the locker is the payer when settling
    address caller = msgSender();
    _settle(currency0, caller, _getFullDebt(currency0));
    _settle(currency1, caller, _getFullDebt(currency1));
}
```

调用 [_settle](#_settle) 方法，分别结算代币 0 和代币 1 的余额。此时 `PoolManager` 中的闪电记账余额一定是非负数，意味着需要调用方支付代币。代币的支付者是调用方，而非 `PositionManager`。

### _takePair

提取交易对的余额。

```solidity
function _takePair(Currency currency0, Currency currency1, address recipient) internal {
    _take(currency0, recipient, _getFullCredit(currency0));
    _take(currency1, recipient, _getFullCredit(currency1));
}
```

调用 [_take](#_take) 方法，分别提取代币 0 和代币 1 的余额。此时 `PoolManager` 中的闪电记账余额一定是非负数，意味着允许调用方提取代币。

### _close

结算或提取单个代币的余额。

```solidity
function _close(Currency currency) internal {
    // this address has applied all deltas on behalf of the user/owner
    // it is safe to close this entire delta because of slippage checks throughout the batched calls.
    int256 currencyDelta = poolManager.currencyDelta(address(this), currency);

    // the locker is the payer or receiver
    address caller = msgSender();
    if (currencyDelta < 0) {
        // Casting is safe due to limits on the total supply of a pool
        _settle(currency, caller, uint256(-currencyDelta));
    } else {
        _take(currency, caller, uint256(currencyDelta));
    }
}
```

调用 [poolManager.currencyDelta](../../v4-core/zh/PoolManager.md#currencydelta) 方法，获取 `PositionManager` 在 `PoolManager` 中的闪电记账余额。

如果 `currencyDelta` 小于 0，即代表 `PositionManager` 欠款，调用 [_settle](#_settle) 方法，向 `PoolManager` 支付代币。

否则，调用 [_take](#_take) 方法，从 `PoolManager` 提取代币。

### _clearOrTake

放弃或提取单个代币的余额。

输入参数：

- `currency`：代币
- `amountMax`：最大提取金额。如果小于等于该值，放弃代币；否则提取代币

```solidity
/// @dev integrators may elect to forfeit positive deltas with clear
/// if the forfeit amount exceeds the user-specified max, the amount is taken instead
/// if there is no credit, no call is made.
function _clearOrTake(Currency currency, uint256 amountMax) internal {
    uint256 delta = _getFullCredit(currency);
    if (delta == 0) return;

    // forfeit the delta if its less than or equal to the user-specified limit
    if (delta <= amountMax) {
        poolManager.clear(currency, delta);
    } else {
        _take(currency, msgSender(), delta);
    }
}
```

获取 `PositionManager` 在 `PoolManager` 中的闪电记账余额。该值一定是非负数。

如果 `delta` 为 0，直接返回。

如果 `delta` 小于等于 `amountMax`，则调用 [poolManager.clear](../../v4-core/zh/PoolManager.md#clear) 方法，放弃代币。

否则，调用 [_take](#_take) 方法，从 `PoolManager` 提取代币到调用方。

### _sweep

从 `PositionManager` （注意，不是 `PoolManager`）提取单个代币的余额。

```solidity
/// @notice Sweeps the entire contract balance of specified currency to the recipient
function _sweep(Currency currency, address to) internal {
    uint256 balance = currency.balanceOfSelf();
    if (balance > 0) currency.transfer(to, balance);
}
```

调用 `currency.balanceOfSelf` 方法，获取 `PositionManager` 合约中的代币余额。

如果余额大于 0，调用 `currency.transfer` 方法，将代币转入 `to` 地址。

### _settle

结算单个代币的欠款。需要向 `PoolManager` 支付代币。

输入参数：

- `currency`：代币
- `payer`：支付者
- `amount`：支付金额

```solidity
/// @notice Pay and settle a currency to the PoolManager
/// @dev The implementing contract must ensure that the `payer` is a secure address
/// @param currency Currency to settle
/// @param payer Address of the payer
/// @param amount Amount to send
/// @dev Returns early if the amount is 0
function _settle(Currency currency, address payer, uint256 amount) internal {
    if (amount == 0) return;

    poolManager.sync(currency);
    if (currency.isAddressZero()) {
        poolManager.settle{value: amount}();
    } else {
        _pay(currency, payer, amount);
        poolManager.settle();
    }
}
```

如果 `amount` 为 0，直接返回。

调用 [poolManager.sync](../../v4-core/zh/PoolManager.md#sync) 方法，同步 `poolManager` 中的代币余额。

如果 `currency` 为 `ADDRESS_ZERO`，即原生 ETH，通过 `{value: amount}` 向 `poolManager` 转入 ETH，调用 [poolManager.settle](../../v4-core/zh/PoolManager.md#settle) 方法，结算记账余额。

否则，调用 [_pay](#_pay) 方法，向 `poolManager` 支付 ERC20 代币，然后调用 [poolManager.settle](../../v4-core/zh/PoolManager.md#settle) 方法，结算记账余额。

### _take

提取单个代币的余额。

输入参数：

- `currency`：代币
- `recipient`：接收者
- `amount`：提取金额

```solidity
/// @notice Take an amount of currency out of the PoolManager
/// @param currency Currency to take
/// @param recipient Address to receive the currency
/// @param amount Amount to take
/// @dev Returns early if the amount is 0
function _take(Currency currency, address recipient, uint256 amount) internal {
    if (amount == 0) return;
    poolManager.take(currency, recipient, amount);
}
```

如果 `amount` 为 0，直接返回。

调用 [poolManager.take](../../v4-core/zh/PoolManager.md#take) 方法，从 `poolManager` 提取代币。

### _modifyLiquidity

```solidity
/// @dev if there is a subscriber attached to the position, this function will notify the subscriber
function _modifyLiquidity(
    PositionInfo info,
    PoolKey memory poolKey,
    int256 liquidityChange,
    bytes32 salt,
    bytes calldata hookData
) internal returns (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) {
    (liquidityDelta, feesAccrued) = poolManager.modifyLiquidity(
        poolKey,
        IPoolManager.ModifyLiquidityParams({
            tickLower: info.tickLower(),
            tickUpper: info.tickUpper(),
            liquidityDelta: liquidityChange,
            salt: salt
        }),
        hookData
    );

    if (info.hasSubscriber()) {
        _notifyModifyLiquidity(uint256(salt), liquidityChange, feesAccrued);
    }
}
```

调用 [poolManager.modifyLiquidity](../../v4-core/zh/PoolManager.md#modifyliquidity) 方法，完成流动性修改。返回 `liquidityDelta` 和 `feesAccrued`。

如果头寸有订阅者，调用 [_notifyModifyLiquidity](#_notifyModifyLiquidity) 方法，通知订阅者。

### _pay

由 `payer` 向 `poolManager` 支付代币。

```solidity
// implementation of abstract function DeltaResolver._pay
function _pay(Currency currency, address payer, uint256 amount) internal override {
    if (payer == address(this)) {
        currency.transfer(address(poolManager), amount);
    } else {
        // Casting from uint256 to uint160 is safe due to limits on the total supply of a pool
        permit2.transferFrom(payer, address(poolManager), uint160(amount), Currency.unwrap(currency));
    }
}
```

如果 `payer` 为 `PositionManager` 合约地址，直接调用 `currency.transfer` 方法，将代币转入 `poolManager`。

否则，调用 `permit2.transferFrom` 方法，从 `payer` 账户中转出代币，转入 `poolManager`。这里需要 `payer` 提前调用 `permit` 方法授权 `PositionManager` 从 `payer` 账户中转出代币。

### _wrap

包装代币，将 `PositionManager` 合约的原生 `ETH` 包装成 `WETH`。

```solidity
/// @dev The amount should already be <= the current balance in this contract.
function _wrap(uint256 amount) internal {
    if (amount > 0) WETH9.deposit{value: amount}();
}
```

调用 `WETH9.deposit` 方法，将 `amount` 数量的 `ETH` 包装成 `WETH`。

### _unwrap

解包代币，将 `PositionManager` 合约的 `WETH` 解包成原生 `ETH`。

```solidity
/// @dev The amount should already be <= the current balance in this contract.
function _unwrap(uint256 amount) internal {
    if (amount > 0) WETH9.withdraw(amount);
}
```

调用 `WETH9.withdraw` 方法，将 `amount` 数量的 `WETH` 解包成 `ETH`。
