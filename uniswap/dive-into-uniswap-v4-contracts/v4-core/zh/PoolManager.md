# PoolManager

PoolManager 是 Uniswap v4 的核心合约，采用单例合约模式，负责管理所有 Uniswap v4 池子，提供池子所有对外接口，包括创建、修改流动性、交易等操作。

## 全局变量

作为单例合约，我们首先关注 PoolManager 是如何保存所有池子状态的。PoolManager 定义了一个全局变量 `_pools`，用于保存所有池子的状态。

```solidity
mapping(PoolId id => Pool.State) internal _pools;
```

其中 `PoolId` 是一个自定义类型，实际上就是 `bytes32`，用于标识池子的唯一 ID。在 [PoolId.sol](./PoolIdLibrary.md) 中可查看 `PoolId` 的定义。

### ModifyLiquidityParams

```solidity
struct ModifyLiquidityParams {
    // the lower and upper tick of the position
    int24 tickLower;
    int24 tickUpper;
    // how to modify the liquidity
    int256 liquidityDelta;
    // a value to set if you want unique liquidity positions at the same range
    bytes32 salt;
}
```

`ModifyLiquidityParams` 结构体用于修改流动性，包括：

- `tickLower`：头寸的下界
- `tickUpper`：头寸的上界
- `liquidityDelta`：流动性变化量，正数表示增加流动性，负数表示减少流动性
- `salt`：用于区分相同范围内的不同头寸

## 函数定义

### onlyWhenUnlocked

`onlyWhenUnlocked` 修饰符用于检查当前合约处于 `unlocked` 状态，否则将回滚。

在 Uniswap v4 中，所有涉及到闪电记账余额变动的操作，如 `modifyLiquidity`、`swap`、`mint`、`burn` 等，都会使用 `onlyWhenUnlocked` 修饰符确保合约处于 `unlocked` 状态，才允许执行记账操作。

```solidity
/// @notice This will revert if the contract is locked
modifier onlyWhenUnlocked() {
    if (!Lock.isUnlocked()) ManagerLocked.selector.revertWith();
    _;
}
```

### unlock

`unlock` 函数用于解锁合约，只有合约处于 `unlocked` 状态，才能执行闪电记账操作。

调用方（如 periphery 合约）需要实现 `IUnlockCallback` 的 `unlockCallback` 接口，在 `unlockCallback` 中将完成 token transfer 操作，确保与 `PoolManager` 合约之间的闪电记账余额为 0。

`unlock` 函数最后会调用 `NonzeroDeltaCount.read()` 检查闪电记账余额是否为 0，如果不为 0，则会回滚。

```solidity
/// @inheritdoc IPoolManager
function unlock(bytes calldata data) external override returns (bytes memory result) {
    if (Lock.isUnlocked()) AlreadyUnlocked.selector.revertWith();

    Lock.unlock();

    // the caller does everything in this callback, including paying what they owe via calls to settle
    result = IUnlockCallback(msg.sender).unlockCallback(data);

    if (NonzeroDeltaCount.read() != 0) CurrencyNotSettled.selector.revertWith();
    Lock.lock();
}
```

### initialize

初始化池子。由于该方法不涉及闪电记账操作，因此不需要 `onlyWhenUnlocked` 修饰符。

参数如下：

- [PoolKey](./PoolIdLibrary.md#poolkey)：池子的 key，用于唯一确定一个池子
- `sqrtPriceX96`：池子的初始价格，即 96 位定点数表示的 $\sqrt{P}$

返回：

- `tick` 池子的初始价格对应的 tick

```solidity
/// @inheritdoc IPoolManager
function initialize(PoolKey memory key, uint160 sqrtPriceX96) external noDelegateCall returns (int24 tick) {
    // see TickBitmap.sol for overflow conditions that can arise from tick spacing being too large
    if (key.tickSpacing > MAX_TICK_SPACING) TickSpacingTooLarge.selector.revertWith(key.tickSpacing);
    if (key.tickSpacing < MIN_TICK_SPACING) TickSpacingTooSmall.selector.revertWith(key.tickSpacing);
    if (key.currency0 >= key.currency1) {
        CurrenciesOutOfOrderOrEqual.selector.revertWith(
            Currency.unwrap(key.currency0), Currency.unwrap(key.currency1)
        );
    }
    if (!key.hooks.isValidHookAddress(key.fee)) Hooks.HookAddressNotValid.selector.revertWith(address(key.hooks));

    uint24 lpFee = key.fee.getInitialLPFee();

    key.hooks.beforeInitialize(key, sqrtPriceX96);

    PoolId id = key.toId();

    tick = _pools[id].initialize(sqrtPriceX96, lpFee);

    // event is emitted before the afterInitialize call to ensure events are always emitted in order
    // emit all details of a pool key. poolkeys are not saved in storage and must always be provided by the caller
    // the key's fee may be a static fee or a sentinel to denote a dynamic fee.
    emit Initialize(id, key.currency0, key.currency1, key.fee, key.tickSpacing, key.hooks, sqrtPriceX96, tick);

    key.hooks.afterInitialize(key, sqrtPriceX96, tick);
}
```

* 检查 `tickSpacing` 的合法性，必须满足 $ 1 \leq tickSpacing \leq 32767 $。

* 检查 `currency0` 和 `currency1` 的顺序，要求 `currency0 < currency1`。其中，原生 ETH 用 `address(0)` 表示。

* 通过 [isValidHookAddress](./HooksLibrary.md#isvalidhookaddress) 检查 Hooks 地址的合法性，要求 `key.hooks` 是有效的合约地址。

* 获取初始的 `lpFee`，对于动态手续费的池子，默认初始的 `lpFee` 为 0。

* 调用 Hooks 合约的 `beforeInitialize` 函数。

* 通过 [initialize](./PoolLibrary.md#initialize) 函数初始化池子。

* 调用 Hooks 合约的 `afterInitialize` 函数。

### modifyLiquidity

修改流动性。该方法会调用闪电记账操作，因此需要 `onlyWhenUnlocked` 修饰符。

输入参数如下：

- [PoolKey](./PoolIdLibrary.md#poolkey) memory key：池子的 key
- [ModifyLiquidityParams](#modifyliquidityparams) memory params：修改流动性的参数
- bytes calldata hookData：Hooks 函数的数据

返回：

- `callerDelta`：调用方的闪电记账余额变化量
- `feesAccrued`：调用方的手续费

```solidity
/// @inheritdoc IPoolManager
function modifyLiquidity(
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    bytes calldata hookData
) external onlyWhenUnlocked noDelegateCall returns (BalanceDelta callerDelta, BalanceDelta feesAccrued) {
    PoolId id = key.toId();
    {
        Pool.State storage pool = _getPool(id);
        pool.checkPoolInitialized();

        key.hooks.beforeModifyLiquidity(key, params, hookData);

        BalanceDelta principalDelta;
        (principalDelta, feesAccrued) = pool.modifyLiquidity(
            Pool.ModifyLiquidityParams({
                owner: msg.sender,
                tickLower: params.tickLower,
                tickUpper: params.tickUpper,
                liquidityDelta: params.liquidityDelta.toInt128(),
                tickSpacing: key.tickSpacing,
                salt: params.salt
            })
        );

        // fee delta and principal delta are both accrued to the caller
        callerDelta = principalDelta + feesAccrued;
    }

    // event is emitted before the afterModifyLiquidity call to ensure events are always emitted in order
    emit ModifyLiquidity(id, msg.sender, params.tickLower, params.tickUpper, params.liquidityDelta, params.salt);

    BalanceDelta hookDelta;
    (callerDelta, hookDelta) = key.hooks.afterModifyLiquidity(key, params, callerDelta, feesAccrued, hookData);

    // if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
    if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

    _accountPoolBalanceDelta(key, callerDelta, msg.sender);
}
```

根据 PoolId 获取池子当前状态，检查池子是否已初始化。

调用 Hooks 合约的 `beforeModifyLiquidity` 函数。

调用池子的 [pool.modifyLiquidity](./PoolLibrary.md#modifyliquidity) 函数修改流动性。
返回的 `principalDelta` 为调用方的流动性变化量，负数表示需要调用方存入代币，正数表示允许调用方取回代币；`feesAccrued` 为待调用方领取的手续费。

`callerDelta = principalDelta + feesAccrued;` 表示调用方的闪电记账余额变化量。

调用 Hooks 合约的 `afterModifyLiquidity` 函数，允许重新计算 `callerDelta`。

如果 `hookDelta` 不为 0，则调用 `_accountPoolBalanceDelta` 函数，为 Hooks 合约记录闪电记账余额变化量。

调用 `_accountPoolBalanceDelta` 函数，为调用方记录闪电记账余额变化量。

注意：`hookDelta` 和 `callerDelta` 分别为需要 Hooks 合约和调用方解决的闪电记账余额变化量。
