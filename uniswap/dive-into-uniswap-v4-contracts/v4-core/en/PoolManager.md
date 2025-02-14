# PoolManager

PoolManager is the core contract of Uniswap v4, adopting a singleton contract model, responsible for managing all Uniswap v4 pools, providing all external interfaces for pools, including creation, destruction, liquidity modification, trading, and other operations.

The main interfaces of PoolManager are as follows:

- [unlock](#unlock): Unlock the contract
- [initialize](#initialize): Initialize the pool
- [modifyLiquidity](#modifyliquidity): Modify liquidity
- [swap](#swap): Swap tokens, exchange `token0` for `token1`, or vice versa
- [donate](#donate): Donate tokens
- [sync](#sync): Sync token balances
- [take](#take): Withdraw tokens
- [settle](#settle): Settle tokens
- [clear](#clear): Abandon tokens to be withdrawn from `PoolManager`, clearing the balance to zero
- [mint](#mint): Withdraw tokens through ERC6909 token
- [burn](#burn): Deposit tokens into `PoolManager` by burning ERC6909 tokens

## Global Variables

As a singleton contract, we first focus on how PoolManager saves the state of all pools. PoolManager defines a global variable `_pools` to save the state of all pools.

```solidity
mapping(PoolId id => Pool.State) internal _pools;
```

Among them, `PoolId` is a custom type, i.e., `bytes32`, used to identify the unique ID of the pool. The definition of `PoolId` can be found in [PoolId.sol](./PoolIdLibrary.md).

## Struct Definitions

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

`ModifyLiquidityParams` is used to modify liquidity, including:

- `tickLower`: The lower boundary of the position
- `tickUpper`: The upper boundary of the position
- `liquidityDelta`: The change in liquidity, positive for adding liquidity, negative for removing liquidity
- `salt`: Used to distinguish different positions within the same range, such as using the token ID of ERC721 to distinguish different users' positions

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

`SwapParams` is used to define trading parameters, including:

- `zeroForOne`: Whether to exchange `token0` for `token1`, or vice versa
- `amountSpecified`: The input or output amount of the trade, negative for `exactIn`, i.e., the input token amount; positive for `exactOut`, i.e., the output token amount
  * If `zeroForOne` is `true`:
    * `exactIn` means providing an exact amount of `token0` (to get as much `token1` as possible)
    * `exactOut` means getting an exact amount of `token1` (while providing as little `token0` as possible)
  * If `zeroForOne` is `false`:
    * `exactIn` means providing an exact amount of `token1` (to get as much `token0` as possible)
    * `exactOut` means getting an exact amount of `token0` (while providing as little `token1` as possible)
- `sqrtPriceLimitX96`: The price limit of the trade, if this price is reached, the trade will stop
  * If `zeroForOne` is `true`, i.e., a trade from `token0` to `token1`, after the trade, `token0` ( $x$ ) increases, `token1` ( $y$ ) decreases, i.e., $ \sqrt{P} = \sqrt{\frac{y}{x}} $ decreases, so the target price `sqrtPriceLimitX96` should be less than the current price
  * Conversely, the target price `sqrtPriceLimitX96` should be greater than the current price

## Function Definitions

### onlyWhenUnlocked

The `onlyWhenUnlocked` modifier is used to check whether the current contract is in the `unlocked` state, otherwise, it will revert.

In Uniswap v4, all operations involving [flash accounting](./CurrencyDeltaLibrary.md) balance changes, such as `modifyLiquidity`, `swap`, `mint`, `burn`, etc., will use the `onlyWhenUnlocked` modifier to ensure that the contract is in the `unlocked` state before allowing accounting operations.

```solidity
/// @notice This will revert if the contract is locked
modifier onlyWhenUnlocked() {
    if (!Lock.isUnlocked()) ManagerLocked.selector.revertWith();
    _;
}
```

### unlock

The `unlock` function is used to unlock the contract. Only when the contract is in the `unlocked` state can flash accounting operations be performed.

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

The caller (such as the periphery contract) needs to implement the `unlockCallback` interface of `IUnlockCallback` and complete the token transfer operation in `unlockCallback` to ensure that the accounting balance between the caller and the `PoolManager` contract is zero at the end of the call.

The `unlock` function finally calls `NonzeroDeltaCount.read()` to check whether there are any non-zero accounting balances. If so, it will revert the transaction to ensure that the entire contract is correctly reconciled.

At the end of the `unlock` call, the contract ensures that the flash accounting balance of all accounts is zero, i.e., all accounts have completed repayment or withdrawal operations.

### initialize

Initialize the pool. Since this method does not involve flash accounting operations, it does not require the `onlyWhenUnlocked` modifier.

Parameters are as follows:

- [PoolKey](./PoolIdLibrary.md#poolkey) memory `key`: The key of the pool, used to uniquely identify a pool
- uint160 `sqrtPriceX96`: The initial price of the pool, represented as a 96-bit fixed-point number $\sqrt{P}$

Returns:

- int24 `tick`: The tick corresponding to the initial price of the pool

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

* Check the validity of `tickSpacing`, which must satisfy $ 1 \leq tickSpacing \leq 32767 $.

* Check the order of `currency0` and `currency1`, requiring `currency0 < currency1`. Among them, native ETH is represented by `address(0)`.

* Check the validity of the Hooks address through [isValidHookAddress](./HooksLibrary.md#isvalidhookaddress), requiring `key.hooks` to be a valid contract address.

* Get the initial `lpFee`. For pools with dynamic fees, the default initial `lpFee` is 0.

* Call the [beforeInitialize](./HooksLibrary.md#beforeinitialize) function of the Hooks Library.

    > Note: This is not a direct call to the `beforeInitialize` method of the Hooks contract, but a call to the `beforeInitialize` function of the Hooks Library.

* Initialize the pool through the [initialize](./PoolLibrary.md#initialize) function.

* Call the [afterInitialize](./HooksLibrary.md#afterinitialize) function of the Hooks Library.

### modifyLiquidity

Modify liquidity. This method will call flash accounting operations, so it requires the `onlyWhenUnlocked` modifier.

Input parameters are as follows:

- [PoolKey](./PoolIdLibrary.md#poolkey) memory `key`: The key of the pool
- [ModifyLiquidityParams](#modifyliquidityparams) memory `params`: Parameters for modifying liquidity
- bytes calldata `hookData`: Data for the Hooks function

Returns:

- [BalanceDelta](./BalanceDelta.md) `callerDelta`: The accounting balance change of the caller
- [BalanceDelta](./BalanceDelta.md) `feesAccrued`: The fee income of the caller

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

Get the current state of the pool based on PoolId and check whether the pool has been initialized.

Call the [beforeModifyLiquidity](./HooksLibrary.md#beforemodifyliquidity) function of the Hooks Library.

Call the [pool.modifyLiquidity](./PoolLibrary.md#modifyliquidity) function of the pool to modify liquidity.
The returned `principalDelta` is the liquidity change of the caller (excluding fees), negative for the amount of tokens the caller needs to deposit, positive for the amount of tokens the caller can withdraw; `feesAccrued` is the fee to be collected by the caller.

`callerDelta = principalDelta + feesAccrued;` represents the accounting balance change of the caller, including fees.

Call the [afterModifyLiquidity](./HooksLibrary.md#aftermodifyliquidity) function of the Hooks Library, allowing the Hooks contract to split the old `callerDelta` into a new `callerDelta` and `hookDelta`. Therefore, the Hooks contract is allowed to reallocate the amount of tokens the user should pay or receive.

* If `hookDelta` is not 0, call the [_accountPoolBalanceDelta](#_accountpoolbalancedelta) function to record the balance change for the Hooks contract and allocate the balance to the Hooks contract;
* Call the [_accountPoolBalanceDelta](#_accountpoolbalancedelta) function to record the balance change for the caller and allocate the balance to the caller.

Note: `hookDelta` and `callerDelta` are the balance changes that need to be settled/taken by the Hooks contract and the caller, respectively.
* If `delta` is positive, it allows the Hooks contract or the caller to withdraw tokens;
* If `delta` is negative, it requires the Hooks contract or the caller to deposit tokens;
* The upper 128 bits represent the balance change of `token0`, and the lower 128 bits represent the balance change of `token1`.

### swap

Swap tokens. This method will call flash accounting operations, so it requires the `onlyWhenUnlocked` modifier.

Input parameters are as follows:

- [PoolKey](./PoolIdLibrary.md#poolkey) memory `key`: The key of the pool
- [SwapParams](#swapparams) memory `params`: Parameters for the trade
- bytes calldata `hookData`: Data for the Hooks function

Returns:

- [BalanceDelta](./BalanceDelta.md) swapDelta: The balance change of the trade, representing the balance relationship between the caller and `PoolManager`

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

First, check the input parameters. The `swap` method must provide `amountSpecified`, i.e., the trade amount cannot be 0.

Get the current state of the pool based on PoolId and check whether the pool has been initialized.

Call the [beforeSwap](./HooksLibrary.md#beforeswap) function of the Hooks Library to get the trade amount, the balance change before the trade, and the LP fee.

Call the `_swap` function to execute the trade and calculate the balance change `swapDelta` of the trade, which is the amount of tokens the caller needs to settle.

  * In the `_swap` function, call the [pool.swap](./PoolLibrary.md#swap) function to execute the trade and calculate the balance change `swapDelta` of the trade.

Call the [afterSwap](./HooksLibrary.md#afterswap) function of the Hooks Library, allowing the Hooks contract to split `swapDelta` into a new `swapDelta` and `hookDelta`.

* If `hookDelta` is not 0, call the [_accountPoolBalanceDelta](#_accountpoolbalancedelta) function to record the balance change for the Hooks contract.
* Call the [_accountPoolBalanceDelta](#_accountpoolbalancedelta) function to record the balance change `swapDelta` for the caller.

These two balances need to be settled by the Hooks contract and the caller, respectively.
* If positive, it allows the Hooks contract or the caller to withdraw tokens;
* If negative, it requires the Hooks contract or the caller to deposit tokens;
* The upper 128 bits represent the balance change of `token0`, and the lower 128 bits represent the balance change of `token1`.

### _accountPoolBalanceDelta

Record the flash accounting balance change for a specific address based on the two tokens of the pool.

```solidity
/// @notice Accounts the deltas of 2 currencies to a target address
function _accountPoolBalanceDelta(PoolKey memory key, BalanceDelta delta, address target) internal {
    _accountDelta(key.currency0, delta.amount0(), target);
    _accountDelta(key.currency1, delta.amount1(), target);
}
```

### _accountDelta

Record the flash accounting balance change for a specific address `target` and token `currency`.

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

Through the [currency.applyDelta](./CurrencyDeltaLibrary.md#applydelta) function, record the balance change of the token `currency` for the address `target`. Therefore, flash accounting is based on the address and token.

Return the balance before and after the modification. Since `delta != 0`, at most one of `previous` and `next` is 0.

If the balance after the modification `next` is 0, call `NonzeroDeltaCount.decrement()` to decrease the count of non-zero balances, otherwise increase it.

> Note: Here, `NonzeroDeltaCount` is based on any `target` address and `currency`, so it is global.

Reviewing the previously introduced [unlock](#unlock) function, `NonzeroDeltaCount.read()` is used to check the count of non-zero balances at the end of the call.

### donate

Donate tokens to the pool. This method will call flash accounting operations, so it requires the `onlyWhenUnlocked` modifier.

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

Get the current state of the pool based on PoolId and check whether the pool has been initialized.

Call the [beforeDonate](./HooksLibrary.md#beforedonate) function of the Hooks Library.

Call the [pool.donate](./PoolLibrary.md#donate) function of the pool to donate tokens and return the balance change `delta`.

Note: Here, `amount0` and `amount1` are both of type `uint256`. In the `pool.donate` function, `amount0` and `amount1` will be negated to ensure that `amount0` and `amount1` in `delta` are negative, representing the amount of tokens the caller needs to deposit.

Call the [_accountPoolBalanceDelta](#_accountpoolbalancedelta) function to record the balance change for the caller.

Call the [afterDonate](./HooksLibrary.md#afterdonate) function of the Hooks Library.

### sync

Sync the current balance of the specified ERC20 token in `PoolManager` to `transient storage`. This method needs to be called before the token is sent to the `PoolManager` contract to calculate the amount of tokens transferred.

> Note: If you want to settle the balance of native ETH, you can also call this method first, in which case `currency` is `address(0)`.

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

If `currency` is native ETH, reset the `currency` address.

Otherwise, get the current balance of the `currency` token in the contract (`PoolManager`) and sync the token address and balance to `transient storage`.

### take

Withdraw a specified amount of tokens from `PoolManager` and send them to a specified address.

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

This operation first calls the [_accountDelta](#_accountdelta) function to deduct the amount of tokens withdrawn from the flash accounting balance, and then calls `currency.transfer` to transfer the tokens to the specified address.

> Note: This method can be used to complete flash loan operations.

### settle

Settle debts.

`PoolManager` provides both `settle` and `settleFor` methods to settle the amount of tokens owed to `PoolManager` for the caller `msg.sender` and a specified address `target`. Both methods call the [_settle](#_settle) method.

```solidity
/// @inheritdoc IPoolManager
function settle() external payable onlyWhenUnlocked returns (uint256) {
    return _settle(msg.sender);
}

/// @inheritdoc IPoolManager
function settleFor(address recipient) external payable onlyWhenUnlocked returns (uint256) {
    return _settle(recipient);
}
```

The normal settlement process is:

1. Call the [sync](#sync) method to sync the token address and balance.
2. Transfer tokens.
3. Call the [settle](#settle) or `settleFor` method to settle the token balance.

Therefore, before calling this method, the external contract needs to call the [sync](#sync) method to sync the token address and balance.

### _settle

Settle debts for a specified address.

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

Get the token address `currency` to be settled through `CurrencyReserves.getSyncedCurrency()`. This address has been synced through the [sync](#sync) method.

If `currency` is native ETH, get the amount to be settled directly from `msg.value`.

Otherwise, get the balance of the `currency` token in `PoolManager`, compare it with the balance before settlement (`sync`), and calculate the settlement amount (i.e., the amount of tokens transferred). Here, the settlement amount `paid` is required to be positive or 0.

Call the [_accountDelta](#_accountdelta) function to record the balance change for the `recipient` address.

### clear

The `clear` method is used for the caller to abandon the tokens to be withdrawn from `PoolManager` and clear the balance to zero.

This method is generally used for the caller to abandon small amounts of dust tokens, as the transaction fees (such as gas consumed by `transfer token`) for taking these tokens may be higher than the value of the tokens themselves.

Input parameters:

- Currency `currency`: The token address to be cleared.
- uint256 `amount`: The amount of tokens to be cleared.

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

Get the accounting balance of the caller `msg.sender` through `currency.getDelta`.

If the amount of tokens to be cleared `amount` is inconsistent with the current balance, revert.

Since `amount` is non-negative, the caller can only abandon the right to withdraw tokens.

Call the `_accountDelta` function to clear the accounting balance for the caller.

### mint

Through [ERC6909](https://eips.ethereum.org/EIPS/eip-6909) `mint token`, tokens are issued to the caller `msg.sender`, thereby avoiding the `transfer token` operation. For callers who need to frequently interact with the `PoolManager` contract, this method can greatly save gas consumption by avoiding the final token transfer operation for each transaction.

Input parameters:

- `address` to: The address to receive the tokens.
- `uint256` id: The ID of the token, which is actually the uint160 value of the token address `address`.
- `uint256` amount: The amount of tokens, ensuring it is non-negative.

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

Get the token address `address` through `CurrencyLibrary.fromId`.

Since `amount` is non-negative, deduct the amount of tokens withdrawn from the accounting balance, i.e., `-amount`, and then call the `_mint` function to issue ERC6909 tokens to the specified address (only recording the balance change, no `transfer token` operation).

### burn

Similar to [mint](#mint), `burn` uses [ERC6909](https://eips.ethereum.org/EIPS/eip-6909) `burn token` to destroy the tokens already withdrawn, while increasing the corresponding amount of tokens in the accounting balance, thereby avoiding the `transfer token` operation.

Input parameters:

- `address` from: The `owner` address holding the `ERC6909` tokens.
- `uint256` id: The ID of the token, which is actually the uint160 value of the token address `address`.
- `uint256` amount: The amount of tokens, ensuring it is non-negative.

```solidity
/// @inheritdoc IPoolManager
function burn(address from, uint256 id, uint256 amount) external onlyWhenUnlocked {
    Currency currency = CurrencyLibrary.fromId(id);
    _accountDelta(currency, amount.toInt128(), msg.sender);
    _burnFrom(from, currency.toId(), amount);
}
```

Get the token address `address` through `CurrencyLibrary.fromId`.

Since `amount` is non-negative, increase the amount of tokens the caller should withdraw in the accounting balance, and then call the `_burnFrom` function to destroy the corresponding amount of ERC6909 tokens from the `owner` address (only recording the balance change, no `transfer token` operation).

### updateDynamicLPFee

Update the dynamic LP fee. Only pools with dynamic fees and only the Hooks contract can call this method.

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

Check whether the fee is a dynamic fee and whether the caller is the Hooks contract.

Check whether the new dynamic LP fee is valid.

Call the [setLPFee](./PoolLibrary.md#setlpfee) function of the pool to update the dynamic LP fee.
