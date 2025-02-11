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

### SwapParams

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

`SwapParams` 结构体用于定义交易参数，包括：

- `zeroForOne`：是否将 `token0` 换成 `token1`，或者反之
- `amountSpecified`：交易的输入或输出数量，负数表示 `exactIn`，即输入代币数量；正数表示 `exactOut`，即输出代币数量
  * 如果 `zeroForOne` 为 `true`，则：
    * `exactIn` 表示希望提供精确数量的 `token0`（从而获得尽可能多的 `token1`）
    * `exactOut` 表示希望获得精确数量的 `token1`（而提供尽可能少的 `token0`）
  * 如果 `zeroForOne` 为 `false`，则：
    * `exactIn` 表示希望提供精确数量的 `token1`（从而获得尽可能多的 `token0`）
    * `exactOut` 表示希望获得精确数量的 `token0`（而提供尽可能少的 `token1`）
- `sqrtPriceLimitX96`：交易的价格上限，如果达到该价格，则停止交易
  * 如果 `zeroForOne` 为 `true`，即从 `token0` 到 `token1` 的交易，交易之后 `token0`（ $x$ ） 变多，`token1`（ $y$ ） 变少，即 $ \sqrt{P} = \sqrt{\frac{y}{x}} $ 变小，因此目标价格 `sqrtPriceLimitX96` 应该小于当前价格
  * 反之，目标价格 `sqrtPriceLimitX96` 应该大于当前价格

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

* 调用 Hooks Library 的 [beforeInitialize](./HooksLibrary.md#beforeinitialize) 函数。

> 注意：这里不是直接调用 Hooks 合约的 `beforeInitialize` 方法，而是调用 Hooks Library 的 `beforeInitialize` 函数。

* 通过 [initialize](./PoolLibrary.md#initialize) 函数初始化池子。

* 调用 Hooks Library 的 [afterInitialize](./HooksLibrary.md#afterinitialize) 函数。

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

调用 Hooks Library 的 [beforeModifyLiquidity](./HooksLibrary.md#beforemodifyliquidity) 函数。

调用池子的 [pool.modifyLiquidity](./PoolLibrary.md#modifyliquidity) 函数修改流动性。
返回的 `principalDelta` 为调用方的流动性变化量，负数表示需要调用方存入代币，正数表示允许调用方取回代币；`feesAccrued` 为待调用方领取的手续费。

`callerDelta = principalDelta + feesAccrued;` 表示调用方的闪电记账余额变化量。

调用 Hooks Library 的 [afterModifyLiquidity](./HooksLibrary.md#aftermodifyliquidity) 函数，允许 Hooks 合约将旧 `callerDelta` 拆成新 `callerDelta` 和 `hookDelta`。

* 如果 `hookDelta` 不为 0，则调用 [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 函数，为 Hooks 合约记录闪电记账余额变化量。
* 调用 [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 函数，为调用方记录闪电记账余额变化量。

注意：`hookDelta` 和 `callerDelta` 分别为需要 Hooks 合约和调用方结算的闪电记账余额变化量。
* 如果 `delta` 为正，则允许 Hooks 合约或调用方取回代币；
* 如果 `delta` 为负，则需要 Hooks 合约或调用方存入代币。
* 高 128 位表示 `token0` 的余额变化量，低 128 位表示 `token1` 的余额变化量。

### swap

交易。该方法会调用闪电记账操作，因此需要 `onlyWhenUnlocked` 修饰符。

输入参数如下：

- [PoolKey](./PoolIdLibrary.md#poolkey) memory key：池子的 key
- [SwapParams](#swapparams) memory params：交易的参数
- bytes calldata hookData：Hooks 函数的数据

返回：

- `BalanceDelta` swapDelta：交易的闪电记账余额变化量

```solidity
/// @inheritdoc IPoolManager
function swap(PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
{
    if (params.amountSpecified == 0) SwapAmountCannotBeZero.selector.revertWith();
    PoolId id = key.toId();
    Pool.State storage pool = _getPool(id);
    pool.checkPoolInitialized();

    BeforeSwapDelta beforeSwapDelta;
    {
        int256 amountToSwap;
        uint24 lpFeeOverride;
        (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

        // execute swap, account protocol fees, and emit swap event
        // _swap is needed to avoid stack too deep error
        swapDelta = _swap(
            pool,
            id,
            Pool.SwapParams({
                tickSpacing: key.tickSpacing,
                zeroForOne: params.zeroForOne,
                amountSpecified: amountToSwap,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96,
                lpFeeOverride: lpFeeOverride
            }),
            params.zeroForOne ? key.currency0 : key.currency1 // input token
        );
    }

    BalanceDelta hookDelta;
    (swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);

    // if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
    if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

    _accountPoolBalanceDelta(key, swapDelta, msg.sender);
}
```

首先检查输入参数，`swap` 方法必须提供 `amountSpecified`，即交易的数量不能为 0。

根据 PoolId 获取池子当前状态，检查池子是否已初始化。

调用 Hooks Library 的 [beforeSwap](./HooksLibrary.md#beforeswap) 函数，获取本次交易的数量、交易前的余额变化量和 LP 手续费。

调用 `_swap` 函数执行交易，计算交易的闪电记账余额变化量 `swapDelta`，该值为需要调用方结算的代币数量。

  * 在 `_swap` 函数中，会调用 [pool.swap](./PoolLibrary.md#swap) 函数执行交易，计算交易的闪电记账余额变化量 `swapDelta`。

调用 Hooks Library 的 [afterSwap](./HooksLibrary.md#afterswap) 函数，允许 Hooks 合约将旧 `swapDelta` 拆成新 `swapDelta` 和 `hookDelta`。

* 如果 `hookDelta` 不为 0，则调用 [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 函数，为 Hooks 合约记录闪电记账余额变化量。
* 调用 [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 函数，为调用方记录闪电记账余额变化量 `swapDelta`。
  * 这两个余额分别需要 Hooks 合约和调用方结算。
  * 如果为正，则允许 Hooks 合约或调用方取回代币；如果为负，则需要 Hooks 合约或调用方存入代币。
  * 高 128 位表示 `token0` 的余额变化量，低 128 位表示 `token1` 的余额变化量。

### _accountPoolBalanceDelta

基于池子的两种代币，为某个地址记录闪电记账余额变化量。

```solidity
/// @notice Accounts the deltas of 2 currencies to a target address
function _accountPoolBalanceDelta(PoolKey memory key, BalanceDelta delta, address target) internal {
    _accountDelta(key.currency0, delta.amount0(), target);
    _accountDelta(key.currency1, delta.amount1(), target);
}
```

### _accountDelta

为某个地址记录某种代币的闪电记账余额变化量。

```solidity
/// @notice Adds a balance delta in a currency for a target address
function _accountDelta(Currency currency, int128 delta, address target) internal {
    if (delta == 0) return;

    (int256 previous, int256 next) = currency.applyDelta(target, delta);

    if (next == 0) {
        NonzeroDeltaCount.decrement();
    } else if (previous == 0) {
        NonzeroDeltaCount.increment();
    }
}
```

通过 `currency.applyDelta` 函数，为某个地址记录某种代币的闪电记账余额变化量。
返回修改前后的余额，由于 `delta != 0`，因为，`previous` 和 `next` 至多有一个为 `0`。
如果修改后的余额为 0，则调用 `NonzeroDeltaCount.decrement()` 减少闪电记账余额不为 0 的地址数量，否则增加。
注意：这里 `NonzeroDeltaCount` 基于任何 `target` 地址和 `currency` 进行统计，因此是全局的。

回顾前面介绍的 [unlock](#unlock) 函数，`NonzeroDeltaCount.read()` 用于检查调用结束后闪电记账余额是否为 0。

### donate

向池子捐赠代币。该方法会调用闪电记账操作，因此需要 `onlyWhenUnlocked` 修饰符。

```solidity
/// @inheritdoc IPoolManager
function donate(PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta delta)
{
    PoolId poolId = key.toId();
    Pool.State storage pool = _getPool(poolId);
    pool.checkPoolInitialized();

    key.hooks.beforeDonate(key, amount0, amount1, hookData);

    delta = pool.donate(amount0, amount1);

    _accountPoolBalanceDelta(key, delta, msg.sender);

    // event is emitted before the afterDonate call to ensure events are always emitted in order
    emit Donate(poolId, msg.sender, amount0, amount1);

    key.hooks.afterDonate(key, amount0, amount1, hookData);
}
```

根据 PoolId 获取池子当前状态，检查池子是否已初始化。

调用 Hooks Library 的 [beforeDonate](./HooksLibrary.md#beforedonate) 函数。

调用池子的 [pool.donate](./PoolLibrary.md#donate) 函数，捐赠代币，返回闪电记账余额变化量 `delta`。

注意：这里 `amount0` 和 `amount1` 都是 `uint256` 类型，在 `pool.donate` 函数中，会将 `amount0` 和 `amount1` 取反，确保 `delta` 中的 `amount0` 和 `amount1` 为负数，表示需要调用方存入的代币数量。

调用 [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 函数，为调用方记录闪电记账余额变化量。

调用 Hooks Library 的 [afterDonate](./HooksLibrary.md#afterdonate) 函数。

### sync

同步`PoolManager` 的指定 ERC20 代币的当前余额到 `transient storage`。该方法需要在任何 ERC20 代币发送到 `PoolManager` 合约之前调用，以便确认转移的代币数量。

> 注意：如果希望结算原生 ETH 的余额，也可以先调用该方法，此时 `currency` 为 `address(0)`。

```solidity
/// @inheritdoc IPoolManager
function sync(Currency currency) external {
    // address(0) is used for the native currency
    if (currency.isAddressZero()) {
        // The reserves balance is not used for native settling, so we only need to reset the currency.
        CurrencyReserves.resetCurrency();
    } else {
        uint256 balance = currency.balanceOfSelf();
        CurrencyReserves.syncCurrencyAndReserves(currency, balance);
    }
}
```

如果 `currency` 为原生 ETH，则重置 `currency` 地址。

否则，获取当前合约的 `currency` 代币的余额，同步 `currency` 代币地址和当前余额到 `transient storage`。

### take

从 `PoolManager` 取回指定数量的代币，并发送到指定地址。

可以使用该方法完成闪电贷 `flash loan` 操作。

```solidity
/// @inheritdoc IPoolManager
function take(Currency currency, address to, uint256 amount) external onlyWhenUnlocked {
    unchecked {
        // negation must be safe as amount is not negative
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        currency.transfer(to, amount);
    }
}
```

### settle

`PoolManager` 提供 `settle` 和 `settleFor` 两个方法为调用方 `msg.sender` 和指定地址 `target` 结算闪电记账余额。这两个方法都调用 `_settle` 函数完成。

在调用该方法前，需要先调用 `sync` 方法同步代币地址和余额。

正常的结算流程为：

1. 调用 `sync` 方法同步代币地址和余额。
2. transfer token
3. 调用 `settle` 或 `settleFor` 方法结算闪电记账余额。

```solidity
// if settling native, integrators should still call `sync` first to avoid DoS attack vectors
function _settle(address recipient) internal returns (uint256 paid) {
    Currency currency = CurrencyReserves.getSyncedCurrency();

    // if not previously synced, or the syncedCurrency slot has been reset, expects native currency to be settled
    if (currency.isAddressZero()) {
        paid = msg.value;
    } else {
        if (msg.value > 0) NonzeroNativeValue.selector.revertWith();
        // Reserves are guaranteed to be set because currency and reserves are always set together
        uint256 reservesBefore = CurrencyReserves.getSyncedReserves();
        uint256 reservesNow = currency.balanceOfSelf();
        paid = reservesNow - reservesBefore;
        CurrencyReserves.resetCurrency();
    }

    _accountDelta(currency, paid.toInt128(), recipient);
}
```

通过 `CurrencyReserves.getSyncedCurrency()` 获取需要结算的 `currency` 代币地址。该地址先通过 `sync` 方法同步。

如果 `currency` 为原生 ETH，则直接从 `msg.value` 中获取需要结算的金额。

否则，获取 `PoolManager` 的 `currency` 代币的当前余额，比较结算前 `sync` 的余额，计算出结算金额（即 transfer token 的数量）。这里要求结算的金额 `paid` 为正数或 0。

调用 `_accountDelta` 函数，为 `recipient` 地址记录结算的闪电记账余额变化量。

### clear

`clear` 方法用于调用方放弃在 `PoolManager` 中应取回的代币，并将其余额清零。

该方法一般用于调用方放弃余额很少的粉尘代币，因为结算这些代币的手续费（如 `transfer token` 消耗的 gas），可能比代币本身的价值更高。

输入参数：

- `Currency` currency：需要清零的代币地址
- `uint256` amount：需要清零的代币数量

```solidity
/// @inheritdoc IPoolManager
function clear(Currency currency, uint256 amount) external onlyWhenUnlocked {
    int256 current = currency.getDelta(msg.sender);
    // Because input is `uint256`, only positive amounts can be cleared.
    int128 amountDelta = amount.toInt128();
    if (amountDelta != current) MustClearExactPositiveDelta.selector.revertWith();
    // negation must be safe as amountDelta is positive
    unchecked {
        _accountDelta(currency, -(amountDelta), msg.sender);
    }
}
```

通过 `currency.getDelta` 获取调用方 `msg.sender` 的闪电记账余额。

如果需要清零的代币数量 `amount` 与当前余额不一致，则回滚。

由于 `amount` 为非负数，因此调用方只能放弃应取回的代币权益。

调用 `_accountDelta` 函数，为调用方记录清零的闪电记账余额。

### mint

通过 ERC6909 `mint token` 的形式向调用方 `msg.sender` 发放代币，以余额形式记录，从而避免了执行 `transfer token` 操作，对于需要频繁与 `PoolManager` 合约交互的调用方，可以极大节省 gas 消耗。

输入参数：

- `address` to：接收代币的地址。
- `uint256` id：代币的 ID，实际上就是代币地址 `address` 的 uint160 值。
- `uint256` amount：代币的数量，确保为非负数。

```solidity
    /// @inheritdoc IPoolManager
    function mint(address to, uint256 id, uint256 amount) external onlyWhenUnlocked {
        unchecked {
            Currency currency = CurrencyLibrary.fromId(id);
            // negation must be safe as amount is not negative
            _accountDelta(currency, -(amount.toInt128()), msg.sender);
            _mint(to, currency.toId(), amount);
        }
    }
```

通过 `CurrencyLibrary.fromId` 获取代币地址 `address`。

由于 `amount` 为非负数，因此从闪电记账余额中增加调用方应付的 `amount` 数量的代币，然后调用 `_mint` 函数向指定地址发放 ERC6909 代币（只记录余额变化，没有 `transfer token` 操作）。

### burn

与 [mint](#mint) 类似，`burn` 通过 ERC6909 `burn token` 的形式销毁应取回的代币，同时在闪电记账余额中增加对应数量的代币，从而避免了执行 `transfer token` 操作。

输入参数：

- `address` from：销毁代币的地址。
- `uint256` id：代币的 ID，实际上就是代币地址 `address` 的 uint160 值。
- `uint256` amount：代币的数量，确保为非负数。


```solidity
/// @inheritdoc IPoolManager
function burn(address from, uint256 id, uint256 amount) external onlyWhenUnlocked {
    Currency currency = CurrencyLibrary.fromId(id);
    _accountDelta(currency, amount.toInt128(), msg.sender);
    _burnFrom(from, currency.toId(), amount);
}
```

通过 `CurrencyLibrary.fromId` 获取代币地址 `address`。

由于 `amount` 为非负数，因此从闪电记账余额中增加调用方应取回的 `amount` 数量的代币，然后调用 `_burnFrom` 函数销毁对应数量的 ERC6909 代币（只记录余额变化，没有 `transfer token` 操作）。

### updateDynamicLPFee

更新动态 LP 手续费。只有动态手续费的池子才能调用该方法，且只有 Hooks 合约才能调用该方法。

```solidity
/// @inheritdoc IPoolManager
function updateDynamicLPFee(PoolKey memory key, uint24 newDynamicLPFee) external {
    if (!key.fee.isDynamicFee() || msg.sender != address(key.hooks)) {
        UnauthorizedDynamicLPFeeUpdate.selector.revertWith();
    }
    newDynamicLPFee.validate();
    PoolId id = key.toId();
    _pools[id].setLPFee(newDynamicLPFee);
}
```

检查手续费是否为动态手续费，且调用方是否为 Hooks 合约。

检查新的动态 LP 手续费是否合法。

调用池子的 [setLPFee](./PoolLibrary.md#setlpfee) 函数更新动态 LP 手续费。
