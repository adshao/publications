# PositionManager

PositionManager is used for position management, including creating positions, modifying liquidity, deleting positions, and other operations.

The PositionManager contract mainly includes the following interfaces:

- [initializePool](#initializepool): Initialize the pool
- [modifyLiquidities](#modifyliquidities): Modify liquidity
- [modifyLiquiditiesWithoutUnlock](#modifyliquiditieswithoutunlock): Modify liquidity (without unlocking)

According to the [Uniswap v4 workflow](../../assets/uniswap-v4-workflow.png) diagram, both `modifyLiquidities` and `modifyLiquiditiesWithoutUnlock` execute the [_handleAction](#_handleaction) method through [BaseActionsRouter._executeActionsWithoutUnlock](./BaseActionsRouter.md#_executeactionswithoutunlock), which executes different operations specified by the user.

PositionManager mainly includes the following two types of operations:

* Liquidity modification operations
  - [INCREASE_LIQUIDITY](#increase_liquidity): Increase liquidity
  - [INCREASE_LIQUIDITY_FROM_DELTAS](#increase_liquidity_from_deltas): Increase liquidity using flash accounting balance
  - [DECREASE_LIQUIDITY](#decrease_liquidity): Decrease liquidity
  - [MINT_POSITION](#mint_position): Create a position
  - [MINT_POSITION_FROM_DELTAS](#mint_position_from_deltas): Create a position using flash accounting balance
  - [BURN_POSITION](#burn_position): Burn a position

* Balance settlement operations
  - [SETTLE_PAIR](#settle_pair): Settle the debt of a token pair, the caller pays tokens to the `PoolManager` contract
  - [TAKE_PAIR](#take_pair): Withdraw the balance of a token pair
  - [SETTLE](#settle): Settle the debt of a single token, pay tokens to the `PoolManager` contract
  - [TAKE](#take): Withdraw the balance of a single token
  - [CLOSE_CURRENCY](#close_currency): Settle or withdraw the balance of a single token
  - [CLEAR_OR_TAKE](#clear_or_take): Abandon or withdraw the balance of a single token
  - [SWEEP](#sweep): Withdraw the balance of a single token from the PositionManager

## Method Definitions

### initializePool
Initialize the pool.

```solidity
/// @inheritdoc IPoolInitializer_v4
function initializePool(PoolKey calldata key, uint160 sqrtPriceX96) external payable returns (int24) {
    try poolManager.initialize(key, sqrtPriceX96) returns (int24 tick) {
        return tick;
    } catch {
        return type(int24).max;
    }
}
```

Call the [PoolManager.initialize](../../v4-core/en/PoolManager.md#initialize) method to initialize the pool.

### modifyLiquidities

The PositionManager contract adopts the design philosophy of a command line. It does not provide interfaces for each operation separately but allows users to combine different operation commands through [modifyLiquidities](#modifyLiquidities) and execute them sequentially.

This method is the standard entry point for PositionManager.

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

`PositionManager` inherits `BaseActionsRouter`, and `modifyLiquidities` only checks the `deadline` and calls the [_executeActions](./BaseActionsRouter.md#_executeactions) method.

### modifyLiquiditiesWithoutUnlock

The `modifyLiquiditiesWithoutUnlock` method is similar to the `modifyLiquidities` method but does not perform the unlock operation.

External contracts need to ensure that the [PoolManager.unlock](../../v4-core/en/PoolManager.md#unlock) method has been called to unlock.

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

All contracts inheriting [BaseActionsRouter](./BaseActionsRouter.md) need to implement the `_handleAction` method to execute specific operations based on the Action and parameters passed by the user.

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

[ActionsLibrary](./ActionsLibrary.md) defines all `actions` in numerical order:

* Operations less than `Actions.SETTLE` are liquidity modification operations
* Operations greater than or equal to `Actions.SETTLE` are balance settlement (accounting) operations

The logic for each Action is introduced below:

#### INCREASE_LIQUIDITY

Increase liquidity.

```solidity
(uint256 tokenId, uint256 liquidity, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
    params.decodeModifyLiquidityParams();
_increase(tokenId, liquidity, amount0Max, amount1Max, hookData);
return;
```

Decode the parameters and call the [_increase](#_increase) method.

#### INCREASE_LIQUIDITY_FROM_DELTAS

Increase liquidity using flash accounting balance.

```solidity
(uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
    params.decodeIncreaseLiquidityFromDeltasParams();
_increaseFromDeltas(tokenId, amount0Max, amount1Max, hookData);
return;
```

Decode the parameters and call the [_increaseFromDeltas](#_increasefromdeltas) method.

#### DECREASE_LIQUIDITY

Decrease liquidity.

```solidity
(uint256 tokenId, uint256 liquidity, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
    params.decodeModifyLiquidityParams();
_decrease(tokenId, liquidity, amount0Min, amount1Min, hookData);
return;
```

Decode the parameters and call the [_decrease](#_decrease) method.

#### MINT_POSITION

Create a position.

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

Decode the parameters and call the [_mint](#_mint) method.

#### MINT_POSITION_FROM_DELTAS

Create a position using flash accounting balance.

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

Decode the parameters and call the [_mintFromDeltas](#_mintfromdeltas) method.

Here, the [_mapRecipient](./BaseActionsRouter.md#_maprecipient) method is used to calculate the `owner` address.

#### BURN_POSITION

Burn a position.

```solidity
// Will automatically decrease liquidity to 0 if the position is not already empty.
(uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
    params.decodeBurnParams();
_burn(tokenId, amount0Min, amount1Min, hookData);
return;
```

Decode the parameters and call the [_burn](#_burn) method.

#### SETTLE_PAIR

Settle the debt of a token pair, the caller pays tokens to the `PoolManager` contract.

```solidity
(Currency currency0, Currency currency1) = params.decodeCurrencyPair();
_settlePair(currency0, currency1);
return;
```

Decode the parameters and call the [_settlePair](#_settlepair) method.

#### TAKE_PAIR

Withdraw the balance of a token pair, the `PoolManager` pays tokens to the `recipient`.

```solidity
(Currency currency0, Currency currency1, address recipient) = params.decodeCurrencyPairAndAddress();
_takePair(currency0, currency1, _mapRecipient(recipient));
return;
```

Decode the parameters and call the [_takePair](#_takepair) method.

Use the [_mapRecipient](./BaseActionsRouter.md#_maprecipient) method to calculate the `recipient` address.

#### SETTLE

Settle the debt of a single token, the caller pays tokens to the `PoolManager` contract.

```solidity
(Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
_settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
return;
```

Decode the parameters, use the [_mapPayer](./BaseActionsRouter.md#_mappayer) method to determine the payer, use the [_mapSettleAmount](./DeltaResolver.md#_mapsettleamount) method to calculate the amount of tokens to be settled, and finally call the [_settle](./DeltaResolver.md#_settle) method.

#### TAKE

Withdraw the balance of a single token, the `PoolManager` pays tokens to the `recipient`.

```solidity
(Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
return;
```

Decode the parameters and call the [_take](./DeltaResolver.md#_take) method.

Use the [_mapRecipient](./BaseActionsRouter.md#_maprecipient) method to calculate the `recipient` address.
Use the [_mapTakeAmount](./DeltaResolver.md#_maptakeamount) method to calculate the amount of tokens to be withdrawn.

#### CLOSE_CURRENCY

Settle or withdraw the balance of a single token.

When it is uncertain whether to settle or withdraw, you can use `CLOSE_CURRENCY` to close the position of the specified token.

```solidity
Currency currency = params.decodeCurrency();
_close(currency);
return;
```

Decode the parameters and call the [_close](#_close) method.

#### CLEAR_OR_TAKE

Abandon or withdraw the balance of a single token.

When the balance is less than or equal to `amountMax`, abandon the token; otherwise, withdraw the token.

```solidity
(Currency currency, uint256 amountMax) = params.decodeCurrencyAndUint256();
_clearOrTake(currency, amountMax);
return;
```

Decode the parameters and call the [_clearOrTake](#_clearortake) method.

#### SWEEP

Withdraw the balance of a single token from the `PositionManager`, the `PositionManager` pays tokens to the `recipient`.

```solidity
(Currency currency, address to) = params.decodeCurrencyAndAddress();
_sweep(currency, _mapRecipient(to));
return;
```

Decode the parameters and call the [_sweep](#_sweep) method.

Use the [_mapRecipient](./BaseActionsRouter.md#_maprecipient) method to calculate the `to` address.

#### WRAP

Wrap tokens, converting the native `ETH` of the `PositionManager` contract into `WETH`.

```solidity
uint256 amount = params.decodeUint256();
_wrap(_mapWrapUnwrapAmount(CurrencyLibrary.ADDRESS_ZERO, amount, Currency.wrap(address(WETH9))));
return;
```

Decode the parameters and call the [_wrap](#_wrap) method.

Use the [_mapWrapUnwrapAmount](./DeltaResolver.md#_mapwrapunwrapamount) method to calculate the amount of tokens to be wrapped.

#### UNWRAP

Unwrap tokens, converting the `WETH` of the `PositionManager` contract into native `ETH`.

```solidity
uint256 amount = params.decodeUint256();
_unwrap(_mapWrapUnwrapAmount(Currency.wrap(address(WETH9)), amount, CurrencyLibrary.ADDRESS_ZERO));
return;
```

Decode the parameters and call the [_unwrap](#_unwrap) method.

Use the [_mapWrapUnwrapAmount](./DeltaResolver.md#_mapwrapunwrapamount) method to calculate the amount of tokens to be unwrapped.

### _increase

Increase the liquidity of a position.

Input parameters:

- `tokenId`: Position ID, i.e., the ERC721 token ID assigned by PositionManager
- `liquidity`: The amount of liquidity to be increased
- `amount0Max`: The maximum amount of `token0` to be provided
- `amount1Max`: The maximum amount of `token1` to be provided
- `hookData`: Hook data, used to pass to `beforeModifyLiquidity` and `afterModifyLiquidity` Hooks functions

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

Get the pool information and position information through `getPoolAndPositionInfo`.

Call the [_modifyLiquidity](#_modifyliquidity) method to complete the liquidity modification. Return `liquidityDelta` and `feesAccrued`. Among them:

* `liquidityDelta` is the liquidity change value that the caller needs to settle, all non-positive;
* `feesAccrued` is the fees that the position can receive up to the current point, all non-negative.

Since `liquidityDelta` already includes `feeAccrued`, and `amount0Max` and `amount1Max` do not include fees (only calculate the liquidity itself), `liquidityDelta - feesAccrued` needs to be used as the input parameter for the maximum value check.

> Note: `liquidityDelta` is the final amount of tokens `amount0` and `amount1` that the caller needs to pay, represented by negative numbers.

### _increaseFromDeltas

This method is similar to the [_increase](#_increase) method. The difference is that `_increaseFromDeltas` does not need to specify the specific amount of liquidity but calculates the amount of liquidity to be increased by querying the current flash accounting balance of the `PositionManager` contract in the `PoolManager` for the two tokens of the pool.

Finally, call the [_modifyLiquidity](#_modifyliquidity) method to complete the liquidity modification.

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

The [_getFullCredit](./DeltaResolver.md#_getfullcredit) method ensures that the flash accounting balance of the `PositionManager` in the `PoolManager` is always non-negative, meaning that there are tokens that can be withdrawn; otherwise, the transaction will be rolled back.

### _decrease

Decrease the liquidity of a position.

Input parameters:

- `tokenId`: Position ID, i.e., the ERC721 token ID assigned by PositionManager
- `liquidity`: The amount of liquidity to be decreased
- `amount0Min`: The minimum amount of `token0` to be received
- `amount1Min`: The minimum amount of `token1` to be received
- `hookData`: Hook data, used to pass to `beforeModifyLiquidity` and `afterModifyLiquidity` Hooks functions

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

Get the pool information and position information through `getPoolAndPositionInfo`.

Call the [_modifyLiquidity](#_modifyliquidity) method to complete the liquidity modification. Return `liquidityDelta` and `feesAccrued`. Among them:

* `liquidityDelta` is the liquidity change value that the caller needs to withdraw, all non-negative;
* `feesAccrued` is the fees that the position can receive up to the current point, all non-negative.

Since `liquidityDelta` already includes `feeAccrued`, and `amount0Min` and `amount1Min` do not include fees (only calculate the liquidity itself), `liquidityDelta - feesAccrued` needs to be used as the input parameter for the minimum value check.

> Note: `liquidityDelta` is the final amount of tokens that the caller can withdraw, represented by positive numbers or 0.

Since Uniswap v4 does not provide a direct method to withdraw fees, you can use the `DECRESAE_LIQUIDITY` operation to withdraw fees by setting `liquidity`, `amount0Min`, and `amount1Min` to 0.

### _mint

Create a position.

Input parameters:

- `poolKey`: The key of the pool
- `tickLower`: The lower tick of the position
- `tickUpper`: The upper tick of the position
- `liquidity`: The amount of liquidity to be created
- `amount0Max`: The maximum amount of `token0` to be provided
- `amount1Max`: The maximum amount of `token1` to be provided
- `owner`: The owner of the position
- `hookData`: Hook data, used to pass to `beforeModifyLiquidity` and `afterModifyLiquidity` Hooks functions

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

Assign the ERC721 token ID `tokenId`, call the `_mint` method to mint the ERC721 token for the `owner`, representing the position.

Initialize the position information. `positionInfo` represents the mapping of `tokenId` to position information.
If the `poolKey` is not stored, store the `poolKey`.
Therefore, the pool information `poolKey` and position information `positionInfo` can be returned based on the `tokenId` later.

Call the [_modifyLiquidity](#_modifyliquidity) method to complete the liquidity modification. Return `liquidityDelta` and `feesAccrued`. Since `freeAccrued` is 0 at this time, it can be ignored. Check whether `liquidityDelta` exceeds `amount0Max` and `amount1Max`.

### _mintFromDeltas

Create a position using flash accounting balance.

Input parameters:

- `poolKey`: The key of the pool
- `tickLower`: The lower tick of the position
- `tickUpper`: The upper tick of the position
- `amount0Max`: The maximum amount of `token0` to be provided
- `amount1Max`: The maximum amount of `token1` to be provided
- `owner`: The owner of the position
- `hookData`: Hook data, used to pass to `beforeModifyLiquidity` and `afterModifyLiquidity` Hooks functions

Similar to the `_mint` method, the difference is that `_mintFromDeltas` does not need to specify the specific amount of liquidity but calculates the amount of liquidity to be increased by querying the current flash accounting balance of the `PositionManager` contract in the `PoolManager` for the two tokens of the pool.

Finally, call the [_mint](#_mint) method to create the position.

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

Burn a position.

Input parameters:

- `tokenId`: Position ID, i.e., the ERC721 token ID assigned by PositionManager
- `amount0Min`: The minimum amount of `token0` to be received
- `amount1Min`: The minimum amount of `token1` to be received
- `hookData`: Hook data, used to pass to `beforeModifyLiquidity` and `afterModifyLiquidity` Hooks functions

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

Get the pool information and position information through `getPoolAndPositionInfo`.

Query the current liquidity of the position through the `PoolManager`.

Clear the position information and burn the ERC721 token.

If the liquidity is greater than 0, call the [poolManager.modifyLiquidity](../../v4-core/en/PoolManager.md#modifyliquidity) method to remove all liquidity of the position. Return `liquidityDelta` and `feesAccrued`.

Since `liquidityDelta` already includes `feeAccrued`, and `amount0Min` and `amount1Min` do not include fees (only calculate the liquidity itself), `liquidityDelta - feesAccrued` needs to be used as the input parameter for the minimum value check.

If the position has a subscriber, call the `_removeSubscriberAndNotifyBurn` method to notify the subscriber.

Note that after the position is burned, the caller does not receive any tokens. These tokens are recorded in the flash accounting balance of the `PoolManager`; the caller needs to combine operations such as [TAKE_PAIR](#take_pair) to withdraw the tokens to their account.

### _settlePair

Settle the debt of a token pair. The caller needs to pay tokens to the `PoolManager`.

Input parameters:

- `currency0`: Token 0
- `currency1`: Token 1

```solidity
function _settlePair(Currency currency0, Currency currency1) internal {
    // the locker is the payer when settling
    address caller = msgSender();
    _settle(currency0, caller, _getFullDebt(currency0));
    _settle(currency1, caller, _getFullDebt(currency1));
}
```

Call the [_settle](./DeltaResolver.md#_settle) method to settle the balances of token 0 and token 1. At this time, the flash accounting balance in the `PoolManager` must be non-positive, meaning that the caller needs to pay tokens.

Use the [_getFullDebt](./DeltaResolver.md#_getfulldebt) method to get the debt of the `PositionManager` in the `PoolManager`.

### _takePair

Withdraw the balance of a token pair, the `PoolManager` pays tokens to the `recipient`.

```solidity
function _takePair(Currency currency0, Currency currency1, address recipient) internal {
    _take(currency0, recipient, _getFullCredit(currency0));
    _take(currency1, recipient, _getFullCredit(currency1));
}
```

Call the [_take](./DeltaResolver.md#_take) method to withdraw the balances of token 0 and token 1. According to the [_getFullCredit](./DeltaResolver.md#_getfullcredit) method, the flash accounting balance of the `PoolManager` must be non-negative, meaning that the caller is allowed to withdraw tokens.

### _close

Settle or withdraw the balance of a single token. Depending on the sign of `currencyDelta`, decide whether to settle the debt or withdraw the balance.

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

Call the [poolManager.currencyDelta](../../v4-core/en/PoolManager.md#currencydelta) method to get the flash accounting balance of the `PositionManager` in the `PoolManager`.

If `currencyDelta` is less than 0, it means that the `PositionManager` owes tokens, call the [_settle](./DeltaResolver.md#_settle) method to pay tokens to the `PoolManager`, and the `payer` is the caller.

Otherwise, call the [_take](./DeltaResolver.md#_take) method to withdraw tokens from the `PoolManager`, and the token recipient is the caller.

### _clearOrTake

Abandon or withdraw the balance of a single token.

Input parameters:

- `currency`: Token
- `amountMax`: Maximum amount. If less than or equal to this value, abandon the token; otherwise, withdraw the token.

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

Use the [_getFullCredit](./DeltaResolver.md#_getfullcredit) method to get the flash accounting balance of the `PositionManager` in the `PoolManager`. This value must be non-negative.

If `delta` is 0, return directly.

If `delta` is less than or equal to `amountMax`, call the [poolManager.clear](../../v4-core/en/PoolManager.md#clear) method to abandon the token.

Otherwise, call the [_take](./DeltaResolver.md#_take) method to withdraw the token from the `PoolManager` to the caller.

### _sweep

Withdraw the balance of a single token from the `PositionManager` (note, not the `PoolManager`).

```solidity
/// @notice Sweeps the entire contract balance of specified currency to the recipient
function _sweep(Currency currency, address to) internal {
    uint256 balance = currency.balanceOfSelf();
    if (balance > 0) currency.transfer(to, balance);
}
```

Call the `currency.balanceOfSelf` method to get the balance of the specified token in the `PositionManager` contract.

If the balance is greater than 0, call the `currency.transfer` method to transfer the token to the `to` address.

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

Call the [poolManager.modifyLiquidity](../../v4-core/en/PoolManager.md#modifyliquidity) method to complete the liquidity modification. Return `liquidityDelta` and `feesAccrued`.

If the position has a subscriber, call the `_notifyModifyLiquidity` method to notify the subscriber.

### _pay

The `payer` pays tokens to the `poolManager`. Implement the [_pay](./DeltaResolver.md#_pay) method of `DeltaResolver`.

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

If the `payer` is the `PositionManager` contract address, directly call the `currency.transfer` method to transfer the token to the `poolManager`.

Otherwise, call the `permit2.transferFrom` method to transfer the token from the `payer` account to the `poolManager`.
> Note: The `payer` needs to call the `permit` method in advance to authorize the `PositionManager` to transfer tokens from the `payer` account.

### _wrap

Wrap tokens, converting the native `ETH` of the `PositionManager` contract into `WETH`.

```solidity
/// @dev The amount should already be <= the current balance in this contract.
function _wrap(uint256 amount) internal {
    if (amount > 0) WETH9.deposit{value: amount}();
}
```

Call the `WETH9.deposit` method to convert `amount` of `ETH` into `WETH`.

### _unwrap

Unwrap tokens, converting the `WETH` of the `PositionManager` contract into native `ETH`.

```solidity
/// @dev The amount should already be <= the current balance in this contract.
function _unwrap(uint256 amount) internal {
    if (amount > 0) WETH9.withdraw(amount);
}
```

Call the `WETH9.withdraw` method to convert `amount` of `WETH` into `ETH`.
