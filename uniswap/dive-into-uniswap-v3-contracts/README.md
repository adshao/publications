[English](./README.md) | [中文](./README_zh.md)

# Deep Dive into Uniswap v3 Smart Contracts (Part 1)

###### tags: `uniswap` `uniswap-v3` `smart contract` `solidity`

## Overview

Similar to Uniswap v2, Uniswap v3 contracts are divided into two categories:

* [Uniswap-v3-core](#Uniswap-v3-core)
    - The core code of Uniswap v3, implementing all functionalities defined by the protocol, external contracts can interact directly with the core contracts.
* [Uniswap-v3-periphery](../dive-into-uniswap-v3-contracts-2/README.md)
    - Interfaces encapsulated based on usage scenarios, such as Position management, multi-path token swaps, etc. The Uniswap interface interacts with the periphery contracts.

If you wish to read from the perspective of user scenarios, start from [Uniswap-v3-periphery](../dive-into-uniswap-v3-contracts-2/README.md), which includes common functionalities like creating positions, modifying position liquidity, swapping tokens, etc.

If you prefer starting from the core modules at the bottom, begin with [Uniswap-v3-core](#Uniswap-v3-core).

## Uniswap-v3-core

### UniswapV3Factory.sol

The factory contract mainly contains three functionalities:

* [createPool](#createPool): Create a trading pair pool.
* [setOwner](#setOwner): Set the owner of the factory contract.
* [enableFeeAmount](#enableFeeAmount): Add a fee tier.

#### createPool

Creates a Uniswap v3 trading pair pool. Note, due to Uniswap v3 supporting different fee tiers, such as 0.05%, 0.30%, 1.00%, etc., a trading pair contract is uniquely identified by `tokenA`, `tokenB`, and `fee`.

> Calculating the trading pair contract also requires: factory contract address, hash of the contract initialization code.

```solidity
/// @inheritdoc IUniswapV3Factory
function createPool(
    address tokenA,
    address tokenB,
    uint24 fee
) external override noDelegateCall returns (address pool) {
    require(tokenA != tokenB);
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    require(token0 != address(0));
    int24 tickSpacing = feeAmountTickSpacing[fee];
    require(tickSpacing != 0);
    require(getPool[token0][token1][fee] == address(0));
    pool = deploy(address(this), token0, token1, fee, tickSpacing);
    getPool[token0][token1][fee] = pool;
    // populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
    getPool[token1][token0][fee] = pool;
    emit PoolCreated(token0, token1, fee, tickSpacing, pool);
}
```

Since `tokenA` and `tokenB` are unordered, first sort `tokenA`, `tokenB` to ensure `tokenA < tokenB`.

Obtain corresponding `tickSpacing` based on the fee tier:

```solidity
int24 tickSpacing = feeAmountTickSpacing[fee];
```

As introduced in "Deep Dive into Uniswap v3 Whitepaper", `tickSpacing` serves a purpose. Each fee tier corresponds to a `tickSpacing`. Only ticks divisible by `tickSpacing` are allowed to initialize, the larger the `tickSpacing`, the more liquidity per tick and the larger the slippage between ticks, but it saves gas for operations crossing ticks. Here, it is saved as a parameter of `Pool`.

Ensure the trading pair for the corresponding fee tier has not been created:

```solidity
require(getPool[token0][token1][fee] == address(0));
```

Create (deploy) the trading pair contract:

```solidity
pool = deploy(address(this), token0, token1, fee, tickSpacing);
```

Deploy code as follows:

```solidity
/// @dev Deploys a pool with the given parameters by transiently setting the parameters storage slot and then
/// clearing it after deploying the pool.
/// @param factory The contract address of the Uniswap V3 factory
/// @param token0 The first token of the pool by address sort order
/// @param token1 The second token of the pool by address sort order
/// @param fee The fee collected upon every swap in the pool, denominated in hundredths of a bip
/// @param tickSpacing The spacing between usable ticks
function deploy(
    address factory,
    address token0,
    address token1,
    uint24 fee,
    int24 tickSpacing
) internal returns (address pool) {
    parameters = Parameters({factory: factory, token0: token0, token1: token1, fee: fee, tickSpacing: tickSpacing});
    pool = address(new UniswapV3Pool{salt: keccak256(abi.encode(token0, token1, fee))}());
    delete parameters;
}
```

As mentioned in Uniswap v2, to ensure the calculability and uniqueness of the trading pair contract address, Uniswap v2 uses the `CREATE2` opcode to create trading pair contracts; starting from Solidity 0.6.2 version ([Github PR](https://github.com/ethereum/solidity/pull/8177)), passing the `salt` parameter in the `new` method to implement `CREATE2` functionality; the `salt` parameter ensures the uniqueness and calculability of the contract address. From the code, it is known that Uniswap v3 trading pair contract uses token0, token1, fee to uniquely determine a trading pair contract, e.g., based on ETH-USDC 0.05% fee (and the factory contract address, initialization code hash), etc., the trading pair contract address can be calculated.

Lastly, save the trading pair contract address to the `getPool` variable:

```solidity
getPool[token0][token1][fee] = pool;
// populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
getPool[token1][token0][fee] = pool;
```

#### setOwner

Sets the owner of the factory contract. The owner has the following permissions:

* [setOwner](#setOwner): Modify the owner.
* [enableFeeAmount](#enableFeeAmount): Add a fee tier.
* [setFeeProtocol](#setFeeProtocol): Modify the protocol fee ratio for a specific trading pair.
* [collectProtocol](#collectProtocol): Collect protocol fees for a specific trading pair.

First, verify the request is initiated by the current owner, then modify the owner:

```solidity
/// @inheritdoc IUniswapV3Factory
function setOwner(address _owner) external override {
    require(msg.sender == owner);
    emit OwnerChanged(owner, _owner);
    owner = _owner;
}
```

#### enableFeeAmount

Uniswap v3 supports three default fee tiers: 0.05%, 0.30%, and 1.00%, corresponding to fees of 500, 3000, and 10000, respectively; the base unit of fee is one-hundredth of a basis point, i.e., 0.01 bp = $10^{-6}$.

Fee percentage calculation formula:

$$
f_{ratio} = \frac{fee}{1,000,000}
$$

```solidity
/// @inheritdoc IUniswapV3Factory
function enableFeeAmount(uint24 fee, int24 tickSpacing) public override {
    require(msg.sender == owner);
    require(fee < 1000000);
    // tick spacing is capped at 16384 to prevent the situation where tickSpacing is so large that
    // TickBitmap#nextInitializedTickWithinOneWord overflows int24 container from a valid tick
    // 16384 ticks represents a >5x price change with ticks of 1 bips
    require(tickSpacing > 0 && tickSpacing < 16384);
    require(feeAmountTickSpacing[fee] == 0);

    feeAmountTickSpacing[fee] = tickSpacing;
    emit FeeAmountEnabled(fee, tickSpacing);
}
```

### UniswapV3Pool.sol

This is the main code of Uniswap v3, defining the functionalities of the trading pair pool:

* [initialize](#initialize): Initializes the trading pair.
* [mint](#mint): Adds liquidity.
* [burn](#burn): Removes liquidity.
* [swap](#swap): Swaps tokens.
* [flash](#flash): Executes a flash loan.
* [collect](#collect): Withdraws tokens.
* [increaseObservationCardinalityNext](#increaseObservationCardinalityNext): Expands the space for the oracle.
* [observe](#observe): Obtains oracle data.

Additionally, the factory owner can call the following two methods:

* [setFeeProtocol](#setFeeProtocol): Modifies the protocol fee ratio for a specific trading pair.
* [collectProtocol](#collectProtocol): Collects protocol fees for a specific trading pair.

![](./assets/uniswapV3Pool.png)

#### initialize

After creating the trading pair, the `initialize` method must be called to initialize the contract before it can be used normally.

This method initializes the `slot0` variable:

```solidity
/// @inheritdoc IUniswapV3PoolActions
/// @dev not locked because it initializes unlocked
function initialize(uint160 sqrtPriceX96) external override {
    require(slot0.sqrtPriceX96 == 0, 'AI');

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());

    slot0 = Slot0({
        sqrtPriceX96: sqrtPriceX96,
        tick: tick,
        observationIndex: 0,
        observationCardinality: cardinality,
        observationCardinalityNext: cardinalityNext,
        feeProtocol: 0,
        unlocked: true
    });

    emit Initialize(sqrtPriceX96, tick);
}
```

`slot0` is defined as follows:

* `sqrtPriceX96`: The current square root price $\sqrt{P}$ of the trading pair.
* `tick`: The current tick corresponding to $\sqrt{P}$, calculated using [getTickAtSqrtRatio](#gettickatsqrtratio).
* `observationIndex`: The most recently updated (oracle) observation array index.
* `observationCardinality`: The capacity of the (oracle) observation array, maximum 65536, initially set to 1.
* `observationCardinalityNext`: The next (oracle) observation array capacity, if manually expanded, this value will be updated, initially set to 1.
* `feeProtocol`: The protocol fee ratio, can set the transaction fee ratio for `token0` and `token1` separately given to the protocol.
* `unlocked`: Indicates whether the current trading pair contract is unlocked.

#### mint

This method implements the functionality of adding liquidity. In fact, both the first-time addition of liquidity and subsequent increases in liquidity use this method.

Parameters of the `mint` method:

* `recipient`: The recipient (owner) of the position.
* `tickLower`: The lower bound of the liquidity range.
* `tickUpper`: The upper bound of the liquidity range.
* `amount`: The amount of liquidity.
* `data`: Callback parameters.

```solidity
/// @inheritdoc IUniswapV3PoolActions
/// @dev noDelegateCall is applied indirectly via _modifyPosition
function mint(
    address recipient,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount,
    bytes calldata data
) external override lock returns (uint256 amount0, uint256 amount1) {
    require(amount > 0);
    (, int256 amount0Int, int256 amount1Int) =
        _modifyPosition(
            ModifyPositionParams({
                owner: recipient,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: int256(amount).toInt128()
            })
        );

    amount0 = uint256(amount0Int);
    amount1 = uint256(amount1Int);

    uint256 balance0Before;
    uint256 balance1Before;
    if (amount0 > 0) balance0Before = balance0();
    if (amount1 > 0) balance1Before = balance1();
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
    if (amount0 > 0) require(balance0Before.add(amount0) <= balance0(), 'M0');
    if (amount1 > 0) require(balance1Before.add(amount1) <= balance1(), 'M1');

    emit Mint(msg.sender, recipient, tickLower, tickUpper, amount, amount0, amount1);
}
```

The main logic of `mint` is in [_modifyPosition](#_modifyPosition), which returns `amount0Int` and `amount1Int` indicating the amounts of `token0` and `token1` tokens that need to be transferred into the trading pair contract if adding `amount` liquidity.

The caller must transfer the tokens in `uniswapV3MintCallback`; the contract calling `mint` must implement the `IUniswapV3MintCallback` interface, which is implemented in the `NonfungiblePositionManager.sol` of the Uniswap v3 periphery contracts.

> Since the `mint` caller needs to implement an interface method, individual ETH accounts (EOA) cannot call this method.

```solidity
IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
```

##### _modifyPosition

Let's look at `_modifyPosition`:

```solidity
/// @dev Effect some changes to a position
/// @param params the position details and the change to the position's liquidity to effect
/// @return position a storage pointer referencing the position with the given owner and tick range
/// @return amount0 the amount of token0 owed to the pool, negative if the pool should pay the recipient
/// @return amount1 the amount of token1 owed to the pool, negative if the pool should pay the recipient
function _modifyPosition(ModifyPositionParams memory params)
    private
    noDelegateCall
    returns (
        Position.Info storage position,
        int256 amount0,
        int256 amount1
    )
{
    checkTicks(params.tickLower, params.tickUpper);

    Slot0 memory _slot0 = slot0; // SLOAD for gas optimization

    position = _updatePosition(
        params.owner,
        params.tickLower,
        params.tickUpper,
        params.liquidityDelta,
        _slot0.tick
    );
```

First, update position information through [_updatePosition](#_updatePosition), which will be detailed in the next section.

```solidity
    if (params.liquidityDelta != 0) {
        if (_slot0.tick < params.tickLower) {
            // current tick is below the passed range; liquidity can only become in range by crossing from left to
            // right, when we'll need _more_ token0 (it's becoming more valuable) so user must provide it
            amount0 = SqrtPriceMath.getAmount0Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        } else if (_slot0.tick < params.tickUpper) {
            // current tick is inside the passed range
            uint128 liquidityBefore = liquidity; // SLOAD for gas optimization

            // write an oracle entry
            (slot0.observationIndex, slot0.observationCardinality) = observations.write(
                _slot0.observationIndex,
                _blockTimestamp(),
                _slot0.tick,
                liquidityBefore,
                _slot0.observationCardinality,
                _slot0.observationCardinalityNext
            );

            amount0 = SqrtPriceMath.getAmount0Delta(
                _slot0.sqrtPriceX96,
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                _slot0.sqrtPriceX96,
                params.liquidityDelta
            );

            liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);
        } else {
            // current tick is above the passed range; liquidity can only become in range by crossing from right to
            // left, when we'll need _more_ token1 (it's becoming more valuable) so user must provide it
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        }
    }
}
```

The latter half of the code mainly calculates the amounts of `token0` and `token1` needed for this liquidity, `amount0` and `amount1`, through [getAmount0Delta](#getAmount0Delta) and [getAmount1Delta](#getAmount1Delta).

Specifically, when the liquidity range you provide is greater than the current `tick` $i_c$, because the size of `tick` is proportional to $\sqrt{P}$ (i.e., $\sqrt{\frac{y}{x}}$), it means in the range greater than $i_c$, $x$ is more valuable (requires less $x$), so you need to provide $x$ token, i.e., `amount0` amount of `token0`; otherwise, provide $y$ token, i.e., `amount1` of `token1`.

As shown below:

$$
\begin{cases}i_c, ..., \overbrace{i_l, ..., i_u}^{amount0} & \text{if $i_c < i_l$}\\
\overbrace{i_l, ...}^{amount1}, i_c, \overbrace{..., i_u}^{amount0} & \text{if $i_l \leq i_c < i_u$}\\
\overbrace{i_l, ..., i_u}^{amount1}, ..., i_c & \text{if $i_u \leq i_c$}\end{cases}
$$

Where, $i_l$, $i_u$ are the boundaries of the liquidity price range, $i_c$ is the current price corresponding `tick`.

If the current price is within the range, i.e., $i_l \leq i_c < i_u$, `_modifyPosition` will record an (oracle) observation point data, because the liquidity in the range has changed, and it is necessary to record the duration of each liquidity `secondsPerLiquidityCumulativeX128`:

```solidity
// write an oracle entry
(slot0.observationIndex, slot0.observationCardinality) = observations.write(
    _slot0.observationIndex,
    _blockTimestamp(),
    _slot0.tick,
    liquidityBefore,
    _slot0.observationCardinality,
    _slot0.observationCardinalityNext
);
```

After calculating `amount0` and `amount1`, update the global active liquidity `liquidity` of the current trading pair:

```solidity
liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);
```

This global liquidity will be used in [swap](#swap).

##### _updatePosition

The code of `_updatePosition` in `_modifyPosition` is as follows:

```solidity
/// @dev Gets and updates a position with the given liquidity delta
/// @param owner the owner of the position
/// @param tickLower the lower tick of the position's tick range
/// @param tickUpper the upper tick of the position's tick range
/// @param tick the current tick, passed to avoid sloads
function _updatePosition(
    address owner,
    int24 tickLower,
    int24 tickUpper,
    int128 liquidityDelta,
    int24 tick
) private returns (Position.Info storage position) {
    position = positions.get(owner, tickLower, tickUpper);

    uint256 _feeGrowthGlobal0X128 = feeGrowthGlobal0X128; // SLOAD for gas optimization
    uint256 _feeGrowthGlobal1X128 = feeGrowthGlobal1X128; // SLOAD for gas optimization

    // if we need to update the ticks, do it
    bool flippedLower;
    bool flippedUpper;
    if (liquidityDelta != 0) {
        uint32 time = _blockTimestamp();
        (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
            observations.observeSingle(
                time,
                0,
                slot0.tick,
                slot0.observationIndex,
                liquidity,
                slot0.observationCardinality
            );
```

`observations.observeSingle` calculates the cumulative tick `tickCumulative` and the cumulative duration per unit of liquidity `secondsPerLiquidityCumulativeX128` from the last observation point to now.

```solidity
        flippedLower = ticks.update(
            tickLower,
            tick,
            liquidityDelta,
            _feeGrowthGlobal0X128,
            _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            false,
            maxLiquidityPerTick
        );
        flippedUpper = ticks.update(
            tickUpper,
            tick,
            liquidityDelta,
            _feeGrowthGlobal0X128,
            _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            true,
            maxLiquidityPerTick
        );
        if (flippedLower) {
            tickBitmap.flipTick(tickLower, tickSpacing);
        }
        if (flippedUpper) {
            tickBitmap.flipTick(tickUpper, tickSpacing);
        }
    }
```

Then, using `ticks.update` to update the states of `tickLower` (the lower bound of the price range) and `tickUpper` (the upper bound of the price range) respectively, please refer to [Tick.update](#update).

If the corresponding `tick`'s liquidity changes from zero to non-zero or vice versa, it indicates that the `tick` needs to be flipped. If that `tick` is not marked as initialized, then it is marked as initialized; otherwise, it is unmarked; here, the `tickBitmap.flipTick` method is used, please refer to [TickBitmap.flipTick](#flipTick).

```solidity
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
        ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);
```

Then, calculate the cumulative fee per liquidity for that price range.

```solidity
    position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);
```

Update Position information, mainly updating the position's receivable fees `tokensOwed0` and `tokensOwed1`, as well as position liquidity `liquidity`, please refer to [Position.update](#update1).

```solidity
    // clear any tick data that is no longer needed
    if (liquidityDelta < 0) {
        if (flippedLower) {
            ticks.clear(tickLower);
        }
        if (flippedUpper) {
            ticks.clear(tickUpper);
        }
    }
}
```

If it is removing liquidity, and the `tick` is flipped, then call [clear](#Tick.clear) to clear `tick` state.

Lastly, back to the `mint` method, the caller needs to ensure that the tokens calculated here as `amount0` and `amount1` of `token0` and `token1` are transferred to the trading pair contract in `uniswapV3MintCallback`.

In summary, the main tasks of the `mint` method are as follows:

1. Update information about the end points (lower, upper) of the price range: `ticks.update`.
2. If Tick state is flipped, update the bitmap indicator to "initialized" or "uninitialized": `tickBitmap.flipTick`.
3. Update Position information: `positions.update`.
4. If the current tick is within the price range, then:
    - Write an oracle observation point: `observations.write`.
    - Update the global active liquidity: `liquidity`.
5. The caller transfers to the trading pair contract: `uniswapV3MintCallback`.

#### burn

The logic of burning liquidity (`burn`) is almost identical to adding liquidity (`mint`), the only difference is that `liquidityDelta` is negative.

```solidity
/// @inheritdoc IUniswapV3PoolActions
/// @dev noDelegateCall is applied indirectly via _modifyPosition
function burn(
    int24 tickLower,
    int24 tickUpper,
    uint128 amount
) external override lock returns (uint256 amount0, uint256 amount1) {
    (Position.Info storage position, int256 amount0Int, int256 amount1Int) =
        _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: -int256(amount).toInt128()
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, tickLower, tickUpper, amount, amount0, amount1);
}
```

Here, the same [_modifyPosition](#_modifyPosition) method is used as in `mint`.

Note, after burning (part of) the liquidity, the tokens are not transferred back to the caller, but are recorded as unclaimed tokens in the position (Position).

#### swap

The `swap` method is the core of Uniswap v3 code, implementing the exchange of two tokens, from `token0` to `token1`, or vice versa.

Compared to the homogeneous liquidity in Uniswap v2, we focus on how the price changes during the `swap` process, and how it affects liquidity.

First, let's look at the parameters of the `swap` method:

* `recipient`: The recipient of the tokens after the trade.
* `zeroForOne`: If true, swap from `token0` to `token1`, otherwise from `token1` to `token0`.
* `amountSpecified`: The specified amount of tokens, positive for the amount of tokens to be input; negative for the amount of tokens to be output.
* `sqrtPriceLimitX96`: The maximum price (or minimum price) that can be tolerated, in `Q64.96` format; if swapping from `token0` to `token1`, it represents the lower limit of the price during the swap; if swapping from `token1` to `token0`, it represents the upper limit of the price; the swap fails if the price exceeds this value.
* `data`: Callback parameters.

```solidity
/// @inheritdoc IUniswapV3PoolActions
function swap(
    address recipient,
    bool zeroForOne,
    int256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) external override noDelegateCall returns (int256 amount0, int256 amount1) {
    require(amountSpecified != 0, 'AS');

    Slot0 memory slot0Start = slot0;

    require(slot0Start.unlocked, 'LOK');
    require(
        zeroForOne
            ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
            : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
        'SPL'
    );

    slot0.unlocked = false;

    SwapCache memory cache =
        SwapCache({
            liquidityStart: liquidity,
            blockTimestamp: _blockTimestamp(),
            feeProtocol: zeroForOne ? (slot0Start.feeProtocol % 16) : (slot0Start.feeProtocol >> 4),
            secondsPerLiquidityCumulativeX128: 0,
            tickCumulative: 0,
            computedLatestObservation: false
        });

    bool exactInput = amountSpecified > 0;

    SwapState memory state =
        SwapState({
            amountSpecifiedRemaining: amountSpecified,
            amountCalculated: 0,
            sqrtPriceX96: slot0Start.sqrtPriceX96,
            tick: slot0Start.tick,
            feeGrowthGlobalX128: zeroForOne ? feeGrowthGlobal0X128 : feeGrowthGlobal1X128,
            protocolFee: 0,
            liquidity: cache.liquidityStart
        });
```

The code above mainly initializes the states.

Because $\sqrt{P} = \sqrt{\frac{y}{x}}$, when `zeroForOne = true`, i.e., swapping from `token0` to `token1`, during the swap process $x$ in the pool increases, $y$ decreases, thus $\sqrt{P}$ gradually decreases. Therefore, the specified price limit `sqrtPriceLimitX96` needs to be less than the current market price `sqrtPriceX96`.

Also, several key data should be noted:

* Initial trade price `state.sqrtPriceX96` is: `slot0.sqrtPriceX96`
    - Here, `slot0.tick` is not used to calculate the initial price because the value calculated from `slot0.tick` might not match `slot0.sqrtPriceX96`. We will see in the later code that `slot0.tick` cannot serve as the current price.
* Initial available liquidity `state.liquidity` is: `liquidity`, which is the global available liquidity we update in [mint](#mint) or [burn](#burn).

Based on `zeroForOne` and `exactInput`, there can be four combinations of `swap`:

|zeroForOne|exactInput|swap|
|---|---|---|
|true|true|Input a fixed amount of `token0`, output the maximum amount of `token1`|
|true|false|Input the minimum amount of `token0`, output a fixed amount of `token1`|
|false|true|Input a fixed amount of `token1`, output the maximum amount of `token0`|
|false|false|Input the minimum amount of `token1`, output a fixed amount of `token0`|

A complete `swap` can consist of multiple `steps`, the code is as follows:

```solidity
// continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
    StepComputations memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        tickSpacing,
        zeroForOne
    );

    // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
    if (step.tickNext < TickMath.MIN_TICK) {
        step.tickNext = TickMath.MIN_TICK;
    } else if (step.tickNext > TickMath.MAX_TICK) {
        step.tickNext = TickMath.MAX_TICK;
    }

    // get the price for the next tick
    step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);

    // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
    (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
        state.sqrtPriceX96,
        (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
            ? sqrtPriceLimitX96
            : step.sqrtPriceNextX96,
        state.liquidity,
        state.amountSpecifiedRemaining,
        fee
    );

    if (exactInput) {
        state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
        state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
    } else {
        state.amountSpecifiedRemaining += step.amountOut.toInt256();
        state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
    }

    // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }

    // update global fee tracker
    if (state.liquidity > 0)
        state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);

    // shift tick if we reached the next price
    if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
        // if the tick is initialized, run the tick transition
        if (step.initialized) {
            // check for the placeholder value, which we replace with the actual value the first time the swap
            // crosses an initialized tick
            if (!cache.computedLatestObservation) {
                (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                    cache.blockTimestamp,
                    0,
                    slot0Start.tick,
                    slot0Start.observationIndex,
                    cache.liquidityStart,
                    slot0Start.observationCardinality
                );
                cache.computedLatestObservation = true;
            }
            int128 liquidityNet =
                ticks.cross(
                    step.tickNext,
                    (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                    (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                    cache.secondsPerLiquidityCumulativeX128,
                    cache.tickCumulative,
                    cache.blockTimestamp
                );
            // if we're moving leftward, we interpret liquidityNet as the opposite sign
            // safe because liquidityNet cannot be type(int128).min
            if (zeroForOne) liquidityNet = -liquidityNet;

            state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
        }

        state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
    } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
        // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
        state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
    }
}
```

Translated into pseudo code (pseudo code):

```python
loop if remaining tokens != 0 and current price != minimum (or maximum) price:
    // step
    initial price := the price of the previous step
    next tick := find the nearest initialized tick based on the current tick, or the last uninitialized tick in the group
    target price := the price calculated based on the next tick
    post-swap price, consumed input token amount, received output token amount, transaction fee := complete one swap step(initial price, target price, available liquidity, remaining tokens)

    update remaining tokens
    update protocol fee

    if post-swap price == target price:
        if tick is initialized：
            cross the tick, update tick related fields
            update available liquidity with tick net liquidity

        current tick := next tick - 1
    else if post-swap price != initial price:
        current tick := calculate tick based on post-swap price
```

First, find the next `tick`, i.e., `tickNext`, based on the current `tick`, the specific logic can refer to: [tickBitmap.nextInitializedTickWithinOneWord](#nextInitializedTickWithinOneWord).

Calculate the target price for this step:

```solidity
// get the price for the next tick
step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);
```

Calculate this step's input, output, and fees (i.e., execute one swap):

```solidity
// compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
(state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
    state.sqrtPriceX96,
    (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
        ? sqrtPriceLimitX96
        : step.sqrtPriceNextX96,
    state.liquidity,
    state.amountSpecifiedRemaining,
    fee
);
```

`SwapMath.computeSwapStep` will calculate the most input token amount (`amountIn`), output token amount (`amountOut`), transaction fee (`feeAmount`), and post-swap price (`sqrtRatioNextX96`) that can be traded in this step based on the current price, target price, available liquidity, available input tokens, etc. Please refer to [computeSwapStep](#computeSwapStep).

Save the amount of `amountIn` and `amountOut` for this trade:

```solidity
if (exactInput) {
    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
    state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
} else {
    state.amountSpecifiedRemaining += step.amountOut.toInt256();
    state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
}
```

* If specifying input token amount (`token0` or `token1`)
    - `amountSpecifiedRemaining` represents the remaining usable input token amount (after deducting fees)
    - `amountCalculated` represents the output token amount (note, this is a negative value)
* If specifying output token amount (`token0` or `token1`)
    - `amountSpecifiedRemaining` represents the remaining required output token amount (initially negative, so each swap needs to `+= step.amountOut` until it reaches 0)
    - `amountCalculated` represents the (including fee) used input token amount

Calculate the protocol fee:

```solidity
// if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
if (cache.feeProtocol > 0) {
    uint256 delta = step.feeAmount / cache.feeProtocol;
    step.feeAmount -= delta;
    state.protocolFee += uint128(delta);
}
```

If the protocol fee is enabled, then the protocol fee is taken from the transaction fee. Note, the value of `feeProtocol` represents $\frac{1}{n}$ of the transaction fee.

```solidity
// update global fee tracker
if (state.liquidity > 0)
    state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
```

Calculate the cumulative fee per liquidity globally.

```solidity
// shift tick if we reached the next price
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    // if the tick is initialized, run the tick transition
    if (step.initialized) {
        // check for the placeholder value, which we replace with the actual value the first time the swap
        // crosses an initialized tick
        if (!cache.computedLatestObservation) {
            (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                cache.blockTimestamp,
                0,
                slot0Start.tick,
                slot0Start.observationIndex,
                cache.liquidityStart,
                slot0Start.observationCardinality
            );
            cache.computedLatestObservation = true;
        }
        int128 liquidityNet =
            ticks.cross(
                step.tickNext,
                (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                cache.secondsPerLiquidityCumulativeX128,
                cache.tickCumulative,
                cache.blockTimestamp
            );
        // if we're moving leftward, we interpret liquidityNet as the opposite sign
        // safe because liquidityNet cannot be type(int128).min
        if (zeroForOne) liquidityNet = -liquidityNet;

        state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
    }

    state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
} else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
    // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

* If the post-swap price reaches the target price (i.e., the price calculated based on the next `tick`):
    * If that `tick` is initialized, then:
        - Through `ticks.cross` to cross the `tick`, set related `Outside` variables in reverse.
        - Use `tick` net liquidity `liquidityNet` to update the available liquidity `state.liquidity`.
            - During the initialization of `tick` in [mint](#mint), the `liquidityNet` of `tickLower` is positive (equal to `liquidityDelta`), while the `liquidityNet` of `tickUpper` is negative (equal to `-liquidityDelta`). Therefore, the sign of `liquidityNet` needs to be adjusted based on the value of `zeroForOne`.
            - When `zeroForOne = true`, as trading progresses, the amount of $x$ in the pool increases, $y$ decreases, and the price $\sqrt{P}$ gradually decreases. The `tick` moves in the lower direction. If it crosses `tickLower`, it means leaving the range, so liquidity needs to be reduced. Conversely, if it crosses `tickUpper`, it means entering the range, so liquidity needs to be increased. In both cases, `-liquidityNet` is used.
            - When `zeroForOne = false`, as trading progresses, the amount of $y$ in the pool increases, $x$ decreases, and the price $\sqrt{P}$ gradually increases. The `tick` moves in the upper direction. If it crosses `tickUpper`, it means leaving the range, so liquidity needs to be reduced. Conversely, if it crosses `tickLower`, it means entering the range, so liquidity needs to be increased. In both cases, `liquidityNet` is used.
    * Move the current `tick` to the next `tick`.
* If the post-swap price did not reach this step's target price but is not equal to the initial price, i.e., indicates the trade is finished:
    * Calculate the latest `tick` value based on the post-swap price.

Repeat the above steps until the swap is completely finished.

After completing the swap, update the global state:

```solidity
// update tick and write an oracle entry if the tick change
if (state.tick != slot0Start.tick) {
    (uint16 observationIndex, uint16 observationCardinality) =
        observations.write(
            slot0Start.observationIndex,
            cache.blockTimestamp,
            slot0Start.tick,
            cache.liquidityStart,
            slot0Start.observationCardinality,
            slot0Start.observationCardinalityNext
        );
    (slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
        state.sqrtPriceX96,
        state.tick,
        observationIndex,
        observationCardinality
    );
} else {
    // otherwise just update the price
    slot0.sqrtPriceX96 = state.sqrtPriceX96;
}
```

* If the post-swap `tick` differs from the pre-swap `tick`:
    * Record an (oracle) observation point data because `tickCumulative` has changed.
    * Update `slot0.sqrtPriceX96`, `slot0.tick`, etc., note that at this point `sqrtPriceX96` and `tick` may not correspond, only `sqrtPriceX96` can accurately reflect the current price.
* If the pre and post-swap `tick` values are the same, then only modify the price:
    * Update `slot0.sqrtPriceX96`.

Likewise, if the global liquidity has changed, update `liquidity`:

```solidity
// update liquidity if it changed
if (cache.liquidityStart != state.liquidity) liquidity = state.liquidity;
```

Update cumulative fees and protocol fees:

```solidity
// update fee growth global and, if necessary, protocol fees
// overflow is acceptable, protocol has to withdraw before it hits type(uint128).max fees
if (zeroForOne) {
    feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
    if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
} else {
    feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
    if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
}
```

Note, if swapping from `token0` to `token1`, then `token0` can only be collected as a fee; conversely, only `token1` can be collected as a fee.

```solidity
(amount0, amount1) = zeroForOne == exactInput
    ? (amountSpecified - state.amountSpecifiedRemaining, state.amountCalculated)
    : (state.amountCalculated, amountSpecified - state.amountSpecifiedRemaining);
```

Calculate the specific `amount0` and `amount1` needed for this swap.

```solidity
// do the transfers and collect payment
if (zeroForOne) {
    if (amount1 < 0) TransferHelper.safeTransfer(token1, recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
    require(balance0Before.add(uint256(amount0)) <= balance0(), 'IIA');
} else {
    if (amount0 < 0) TransferHelper.safeTransfer(token0, recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
    require(balance1Before.add(uint256(amount1)) <= balance1(), 'IIA');
}

emit Swap(msg.sender, recipient, amount0, amount1, state.sqrtPriceX96, state.liquidity, state.tick);
slot0.unlocked = true;
```

The contract transfers the output tokens to `recipient`, and at the same time, the caller must transfer the input tokens to the trading pair contract in `uniswapV3SwapCallback`.

Thus, the entire `swap` process is concluded.

#### flash

This method implements the flash loan functionality of Uniswap v3.

Method parameters:

* `recipient`: The recipient of the flash loan.
* `amount0`: The amount of `token0` loaned.
* `amount1`: The amount of `token1` loaned.
* `data`: Callback method parameters.

```solidity
/// @inheritdoc IUniswapV3PoolActions
function flash(
    address recipient,
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) external override lock noDelegateCall {
    uint128 _liquidity = liquidity;
    require(_liquidity > 0, 'L');

    uint256 fee0 = FullMath.mulDivRoundingUp(amount0, fee, 1e6);
    uint256 fee1 = FullMath.mulDivRoundingUp(amount1, fee, 1e6);
```

The flash loan fee is the same as the `swap` fee, which is $\frac{fee}{10^6}$.

```solidity
    uint256 balance0Before = balance0();
    uint256 balance1Before = balance1();

    if (amount0 > 0) TransferHelper.safeTransfer(token0, recipient, amount0);
    if (amount1 > 0) TransferHelper.safeTransfer(token1, recipient, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(fee0, fee1, data);

    uint256 balance0After = balance0();
    uint256 balance1After = balance1();

    require(balance0Before.add(fee0) <= balance0After, 'F0');
    require(balance1Before.add(fee1) <= balance1After, 'F1');
```

Transfer the loaned token amounts to the recipient, the caller of the `flash` method must implement the `IUniswapV3FlashCallback.uniswapV3FlashCallback` interface method and return the tokens, including the fee, in that method.

```solidity
    // sub is safe because we know balanceAfter is gt balanceBefore by at least fee
    uint256 paid0 = balance0After - balance0Before;
    uint256 paid1 = balance1After - balance1Before;

    if (paid0 > 0) {
        uint8 feeProtocol0 = slot0.feeProtocol % 16;
        uint256 fees0 = feeProtocol0 == 0 ? 0 : paid0 / feeProtocol0;
        if (uint128(fees0) > 0) protocolFees.token0 += uint128(fees0);
        feeGrowthGlobal0X128 += FullMath.mulDiv(paid0 - fees0, FixedPoint128.Q128, _liquidity);
    }
    if (paid1 > 0) {
        uint8 feeProtocol1 = slot0.feeProtocol >> 4;
        uint256 fees1 = feeProtocol1 == 0 ? 0 : paid1 / feeProtocol1;
        if (uint128(fees1) > 0) protocolFees.token1 += uint128(fees1);
        feeGrowthGlobal1X128 += FullMath.mulDiv(paid1 - fees1, FixedPoint128.Q128, _liquidity);
    }

    emit Flash(msg.sender, recipient, amount0, amount1, paid0, paid1);
}
```

Based on the collected fees, calculate the protocol fees (note, `token0` and `token1`'s protocol fees are set separately, see: [setFeeProtocol](#setFeeProtocol)), and finally update `protocolFees` and `feeGrowthGlobal1X128`.

#### collect

This method implements the function to retrieve tokens, including tokens from burning liquidity and fee tokens.

Parameters are as follows:

- `recipient`: Token recipient
- `tickLower`: Position low point
- `tickUpper`: Position high point
- `amount0Requested`: Amount of `token0` requested to retrieve
- `amount1Requested`: Amount of `token1` requested to retrieve

```solidity
/// @inheritdoc IUniswapV3PoolActions
function collect(
    address recipient,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock returns (uint128 amount0, uint128 amount1) {
    // we don't need to checkTicks here, because invalid positions will never have non-zero tokensOwed{0,1}
    Position.Info storage position = positions.get(msg.sender, tickLower, tickUpper);

    amount0 = amount0Requested > position.tokensOwed0 ? position.tokensOwed0 : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1 ? position.tokensOwed1 : amount1Requested;

    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit Collect(msg.sender, recipient, tickLower, tickUpper, amount0, amount1);
}
```

The code above is relatively simple and is not expanded here. Note that if you wish to retrieve all tokens, you need to specify a number larger than `tokensOwned`, such as using `type(uint128).max`.

#### increaseObservationCardinalityNext

Expands the writable space for oracle observation points. This method calls the [grow](#grow) method in `Oracle.sol` to achieve expansion.

```solidity
/// @inheritdoc IUniswapV3PoolActions
function increaseObservationCardinalityNext(uint16 observationCardinalityNext)
    external
    override
    lock
    noDelegateCall
{
    uint16 observationCardinalityNextOld = slot0.observationCardinalityNext; // for the event
    uint16 observationCardinalityNextNew =
        observations.grow(observationCardinalityNextOld, observationCardinalityNext);
    slot0.observationCardinalityNext = observationCardinalityNextNew;
    if (observationCardinalityNextOld != observationCardinalityNextNew)
        emit IncreaseObservationCardinalityNext(observationCardinalityNextOld, observationCardinalityNextNew);
}
```

#### observe

Retrieves observation point data for specified times in bulk. This method implements it by calling [observe](#observe2) in `Oracle.sol`.

```solidity
/// @inheritdoc IUniswapV3PoolDerivedState
function observe(uint32[] calldata secondsAgos)
    external
    view
    override
    noDelegateCall
    returns (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s)
{
    return
        observations.observe(
            _blockTimestamp(),
            secondsAgos,
            slot0.tick,
            slot0.observationIndex,
            liquidity,
            slot0.observationCardinality
        );
}
```

#### setFeeProtocol

Sets the ratio of protocol fees. This method is only allowed to be executed by the factory contract's owner.

Note that the protocol fee ratios for `token0` and `token1` must be set separately. The ratio is a proportion of the transaction fee, with valid values being 0 (protocol fee disabled) or $4 \leq n \leq 10$, i.e., the protocol fee can be set to $\frac{1}{n}$ of the transaction fee.

```solidity
/// @inheritdoc IUniswapV3PoolOwnerActions
function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
    require(
        (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
            (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
    );
    uint8 feeProtocolOld = slot0.feeProtocol;
    slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
    emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
}
```

The type of `slot0.feeProtocol` is `uint8`, storing the protocol fee ratios for both the two tokens, with the high 4 bits for `token1` and the low 4 bits for `token0`:

$$
slot0.feeProtocol = \overbrace{0000}^{fee1}\overbrace{0000}^{fee0}
$$

Therefore, `feeProtocolOld % 16` represents the protocol fee ratio `fee0` for `token0`, and `feeProtocolOld >> 4` represents the protocol fee ratio `fee1` for `token1`.

#### collectProtocol

Retrieves protocol fees. This method is only allowed to be executed by the factory contract's owner.

Protocol fees come from two sources:

- Transaction fees generated by `swap`
- Flash loan fees generated by `flash`

```solidity
/// @inheritdoc IUniswapV3PoolOwnerActions
function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
    amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
    amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

    if (amount0 > 0) {
        if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
        protocolFees.token0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
        protocolFees.token1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit CollectProtocol(msg.sender, recipient, amount0, amount1);
}
```

The code above is relatively simple and is not expanded further.

### Tick.sol

Tick.sol manages the internal state of Tick.

#### tickSpacingToMaxLiquidityPerTick

Calculates the maximum liquidity per `tick` based on `tickSpacing`, only `tick` that can be divided by `tickSpacing` can store liquidity:

```solidity
/// @notice Derives max liquidity per tick from given tick spacing
/// @dev Executed within the pool constructor
/// @param tickSpacing The amount of required tick separation, realized in multiples of `tickSpacing`
///     e.g., a tickSpacing of 3 requires ticks to be initialized every 3rd tick i.e., ..., -6, -3, 0, 3, 6, ...
/// @return The max liquidity per tick
function tickSpacingToMaxLiquidityPerTick(int24 tickSpacing) internal pure returns (uint128) {
    int24 minTick = (TickMath.MIN_TICK / tickSpacing) * tickSpacing;
    int24 maxTick = (TickMath.MAX_TICK / tickSpacing) * tickSpacing;
    uint24 numTicks = uint24((maxTick - minTick) / tickSpacing) + 1;
    return type(uint128).max / numTicks;
}
```

#### getFeeGrowthInside

Calculates the accumulated fee per liquidity inside two `tick` ranges, implementing formulas 6.17-6.19 from the white paper:

$$
f_a(i) = \begin{cases} f_g - f_o(i) & \text{if $i_c \geq i$}\\
f_o(i) & \text{$i_c < i$} \end{cases} \quad \text{(6.17)}
$$

$$
f_b(i) = \begin{cases} f_o(i) & \text{if $i_c \geq i$}\\
f_g - f_o(i) & \text{$i_c < i$}\end{cases} \quad \text{(6.18)}
$$

$$
f_r = f_g - f_b(i_l) - f_a(i_u) \quad \text{(6.19)}
$$

Code as follows:

```solidity
/// @notice Retrieves fee growth data
/// @param self The mapping containing all tick information for initialized ticks
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @param tickCurrent The current tick
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @return feeGrowthInside0X128 The all-time fee growth in token0, per unit of liquidity, inside the position's tick boundaries
/// @return feeGrowthInside1X128 The all-time fee growth in token1, per unit of liquidity, inside the position's tick boundaries
function getFeeGrowthInside(
    mapping(int24 => Tick.Info) storage self,
    int24 tickLower,
    int24 tickUpper,
    int24 tickCurrent,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
) internal view returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) {
    Info storage lower = self[tickLower];
    Info storage upper = self[tickUpper];

    // calculate fee growth below
    uint256 feeGrowthBelow0X128;
    uint256 feeGrowthBelow1X128;
    if (tickCurrent >= tickLower) {
        feeGrowthBelow0X128 = lower.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = lower.feeGrowthOutside1X128;
    } else {
        feeGrowthBelow0X128 = feeGrowthGlobal0X128 - lower.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = feeGrowthGlobal1X128 - lower.feeGrowthOutside1X128;
    }

    // calculate fee growth above
    uint256 feeGrowthAbove0X128;
    uint256 feeGrowthAbove1X128;
    if (tickCurrent < tickUpper) {
        feeGrowthAbove0X128 = upper.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = upper.feeGrowthOutside1X128;
    } else {
        feeGrowthAbove0X128 = feeGrowthGlobal0X128 - upper.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = feeGrowthGlobal1X128 - upper.feeGrowthOutside1X128;
    }

    feeGrowthInside0X128 = feeGrowthGlobal0X128 - feeGrowthBelow0X128 - feeGrowthAbove0X128;
    feeGrowthInside1X128 = feeGrowthGlobal1X128 - feeGrowthBelow1X128 - feeGrowthAbove1X128;
}
```

First, based on the current `tickCurrent`, calculate $f_a$ and $f_b$ for `tickLower` and `tickUpper`, and finally calculate the in-range fee $f_r$:

$$
\underbrace{\overbrace{..., i_l - 1}^{f_b(i_l)}, \overbrace{i_l, i_l + 1, ..., i_u - 1, i_u}^{f_r}, \overbrace{i_u + 1, ...}^{f_a(i_u)}}_{f_g}
$$

Why calculate the cumulative fee per liquidity inside the range? Because each position (`Position`) calculates its own receivable fee upon `mint`/`burn` based on this value:

$$
liquidityDelta \cdot (feeGrowthInside - feeGrowthInsideLast)
$$

#### update

Updates the `tick` state and returns whether the `tick` was flipped from initialized to uninitialized, or vice versa:

```solidity
/// @notice Updates a tick and returns true if the tick was flipped from initialized to uninitialized, or vice versa
/// @param self The mapping containing all tick information for initialized ticks
/// @param tick The tick that will be updated
/// @param tickCurrent The current tick
/// @param liquidityDelta A new amount of liquidity to be added (subtracted) when tick is crossed from left to right (right to left)
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @param secondsPerLiquidityCumulativeX128 The all-time seconds per max(1, liquidity) of the pool
/// @param tickCumulative The tick * time elapsed since the pool was first initialized
/// @param time The current block timestamp cast to a uint32
/// @param upper true for updating a position's upper tick, or false for updating a position's lower tick
/// @param maxLiquidity The maximum liquidity allocation for a single tick
/// @return flipped Whether the tick was flipped from initialized to uninitialized, or vice versa
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int24 tickCurrent,
    int128 liquidityDelta,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    uint160 secondsPerLiquidityCumulativeX128,
    int56 tickCumulative,
    uint32 time,
    bool upper,
    uint128 maxLiquidity
) internal returns (bool flipped) {
    Tick.Info storage info = self[tick];

    uint128 liquidityGrossBefore = info.liquidityGross;
    uint128 liquidityGrossAfter = LiquidityMath.addDelta(liquidityGrossBefore, liquidityDelta);

    require(liquidityGrossAfter <= maxLiquidity, 'LO');

    flipped = (liquidityGrossAfter == 0) != (liquidityGrossBefore == 0);
```

If a `tick` transitions from no liquidity to having liquidity, or from having liquidity to no liquidity, it indicates that the `tick` needs to be flipped (`flipped`).

```solidity
    if (liquidityGrossBefore == 0) {
        // by convention, we assume that all growth before a tick was initialized happened _below_ the tick
        if (tick <= tickCurrent) {
            info.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            info.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
            info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128;
            info.tickCumulativeOutside = tickCumulative;
            info.secondsOutside = time;
        }
        info.initialized = true;
    }
```

If a `tick` had no liquidity before, it is initialized; for ticks less than the current `tickCurrent`, set the `Outside` variables.

```solidity
    info.liquidityGross = liquidityGrossAfter;
```

`liquidityGross` represents the total liquidity and is used to determine whether a `tick` requires initialization:

- For `mint` operations, liquidity is increased; for burn operations, liquidity is decreased.
- This variable is unrelated to whether the `tick` serves as a boundary low point (`tickLower`) or high point (`tickUpper`) in different positions. It is solely related to the `mint` or `burn` operations.
- If a `tick` is simultaneously used as both `tickLower` and `tickUpper`, its `liquidityNet` might be 0, but `liquidityGross` will still be greater than 0. As a result, no reinitialization is required.

```solidity
    // when the lower (upper) tick is crossed left to right (right to left), liquidity must be added (removed)
    info.liquidityNet = upper
        ? int256(info.liquidityNet).sub(liquidityDelta).toInt128()
        : int256(info.liquidityNet).add(liquidityDelta).toInt128();
}
```

`liquidityNet` represents net liquidity, when `swap` crosses a `tick`, it is used to update the global available liquidity `liquidity`:

- If it acts as `tickLower`, i.e., the lower boundary (left boundary), then increase `liquidityDelta` (`mint` is positive, `burn` is negative)
- If it acts as `tickUpper`, i.e., the upper boundary (right boundary), then decrease `liquidityDelta` (`mint` is positive, `burn` is negative)

#### clear

When a `tick` is flipped, if no liquidity is associated with that `tick`, i.e., `liquidityGross = 0`, then clear the `tick` state:

```solidity
/// @notice Clears tick data
/// @param self The mapping containing all initialized tick information for initialized ticks
/// @param tick The tick that will be cleared
function clear(mapping(int24 => Tick.Info) storage self, int24 tick) internal {
    delete self[tick];
}
```

#### cross

When a `tick` is crossed, it is necessary to flip the direction of the `Outside` variables, as per formula 6.20 in the white paper:

$$
f_o(i) := f_g - f_o(i) \quad \text{(6.20)}
$$

These variables are used in methods like [getFeeGrowthInside](#getFeeGrowthInside).

```solidity
/// @notice Transitions to next tick as needed by price movement
/// @param self The mapping containing all tick information for initialized ticks
/// @param tick The destination tick of the transition
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @param secondsPerLiquidityCumulativeX128 The current seconds per liquidity
/// @param tickCumulative The tick * time elapsed since the pool was first initialized
/// @param time The current block.timestamp
/// @return liquidityNet The amount of liquidity added (subtracted) when tick is crossed from left to right (right to left)
function cross(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    uint160 secondsPerLiquidityCumulativeX128,
    int56 tickCumulative,
    uint32 time
) internal returns (int128 liquidityNet) {
    Tick.Info storage info = self[tick];
    info.feeGrowthOutside0X128 = feeGrowthGlobal0X128 - info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 = feeGrowthGlobal1X128 - info.feeGrowthOutside1X128;
    info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128 - info.secondsPerLiquidityOutsideX128;
    info.tickCumulativeOutside = tickCumulative - info.tickCumulativeOutside;
    info.secondsOutside = time - info.secondsOutside;
    liquidityNet = info.liquidityNet;
}
```

### TickMath.sol

TickMath primarily contains two methods:

- [getSqrtRatioAtTick](#getSqrtRatioAtTick): Calculates the square root price $\sqrt{P}$ based on a tick
- [getTickAtSqrtRatio](#getTickAtSqrtRatio): Calculates the tick based on the square root price $\sqrt{P}$

#### getSqrtRatioAtTick

This method corresponds to formula 6.2 in the white paper:

$$
\sqrt{p}(i) = \sqrt{1.0001}^i = 1.0001^{\frac{i}{2}}
$$

where $i$ is the `tick`.

Since Uniswap v3 supports a price range ($\frac{token1}{token0}$) of $[2^{-128}, 2^{128}]$, according to formula 6.1 in the white paper:

$$
p(i) = 1.0001^i
$$

The corresponding maximum tick (MAX_TICK) is:

$$
i = \lfloor log_{1.0001}{p(i)} \rfloor = \lfloor log_{1.0001}{2^{128}} \rfloor = \lfloor 887272.7517970635 \rfloor = 887272
$$

The minimum tick (MIN_TICK) is:

$$
i = \lceil log_{1.0001}{2^{-128}} \rceil = \lceil -887272.7517970635 \rceil = -887272
$$

Assuming $i \geq 0$, for a given tick $i$, it can always be represented in binary, thus the following formula always holds:

$$
\begin{cases} i = \sum_{n=0}^{19}{(x_n \cdot 2^n)} = x_0 \cdot 1 + x_1 \cdot 2 + x_2 \cdot 4 + ... + x_{19}\cdot 524288 \\
\forall x_n \in \{0, 1\} \end{cases} \quad \text{(1.1)}
$$

where $x_n$ are the binary digits of $i$. For example, if $i=6$, its binary representation is `000000000000000000000110`, then $x_1 = 1, x_2 = 1$, and the rest of $x_n$ are 0.

Similarly, it can be deduced that $i < 0$ can also be represented by a similar formula.

Let's first consider the case when $i < 0$:

If $i < 0$, then:

$$
\sqrt{p}(i) = 1.0001^{\frac{i}{2}} = 1.0001^{-\frac{|i|}{2}} = \frac{1}{1.0001^{\frac{|i|}{2}}} = \frac{1}{1.0001^{\frac{1}{2}(\sum_{n=0}^{19}{(x_n \cdot 2^n)})}} \\
= \frac{1}{1.0001^{\frac{1}{2} \cdot x_0}} \cdot \frac{1}{1.0001^{\frac{2}{2} \cdot x_1}} \cdot \frac{1}{1.0001^{\frac{4}{2} \cdot x_2}} \cdot ... \cdot \frac{1}{1.0001^{\frac{524288}{2} \cdot x_{19}}}
$$

Based on the value of binary digits $x_n$, we can summarize as follows:

$$
\frac{1}{1.0001^{\frac{x_n \cdot 2^n}{2}}} \begin{cases} = 1 & \text{if $x_n = 0, n \geq 0, i < 0$}\\
< 1 & \text{if $x_n = 1, n \geq 0, i < 0$}
\end{cases}
$$

To minimize precision errors during computation, `Q128.128` (128-bit fixed-point number) is used to represent intermediate prices, and for each price $p$, it needs to be left-shifted by 128 bits. Since $i < 0, x_n = 1$ implies $\frac{1}{1.0001^{\frac{x_n \cdot 2^n}{2}}} < 1$, there won't be an overflow issue during the continuous multiplication process.

The method to calculate $\sqrt{p}(i)$ can be summarized as follows:

- Start with a value of 1, begin from the 0th bit, and iterate through the binary bits of $i$ from low to high (from right to left)
- If the bit is not 0, then multiply by $\frac{2^{128}}{1.0001^{\frac{2^n}{2}}}$, where $2^{128}$ represents a 128-bit left shift
- If the bit is 0, then multiply by 1, which can be omitted

```solidity
/// @notice Calculates sqrt(1.0001^tick) * 2^96
/// @dev Throws if |tick| > max tick
/// @param tick The input tick for the above formula
/// @return sqrtPriceX96 A Fixed point Q64.96 number representing the sqrt of the ratio of the two assets (token1/token0)
/// at the given tick
function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) {
    uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));
    require(absTick <= uint256(MAX_TICK), 'T');

    // If the 0th bit is not 0, then ratio = 0xfffcb933bd6fad37aa2d162d1a594001, which is 2^128 / 1.0001^0.5
    uint256 ratio = absTick & 0x1 != 0 ? 0xfffcb933bd6fad37aa2d162d1a594001 : 0x100000000000000000000000000000000;
    // If the 1st bit is not 0, then multiply by 0xfff97272373d413259a46990580e213a, which is 2^128 / 1.0001^1, since both multipliers are Q128.128, the final result is multiplied by 2^128, so it needs to be right-shifted by 128
    if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
    // If the 2nd bit is not 0, then multiply by 0xfff2e50f5f656932ef12357cf3c7fdcc, which is 2^128 / 1.0001^2,
    if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
    // And so on
    if (absTick & 0x8 != 0) ratio = (ratio * 0xffe5caca7e10e4e61c3624eaa0941cd0) >> 128;
    if (absTick & 0x10 != 0) ratio = (ratio * 0xffcb9843d60f6159c9db58835c926644) >> 128;
    if (absTick & 0x20 != 0) ratio = (ratio * 0xff973b41fa98c081472e6896dfb254c0) >> 128;
    if (absTick & 0x40 != 0) ratio = (ratio * 0xff2ea16466c96a3843ec78b326b52861) >> 128;
    if (absTick & 0x80 != 0) ratio = (ratio * 0xfe5dee046a99a2a811c461f1969c3053) >> 128;
    if (absTick & 0x100 != 0) ratio = (ratio * 0xfcbe86c7900a88aedcffc83b479aa3a4) >> 128;
    if (absTick & 0x200 != 0) ratio = (ratio * 0xf987a7253ac413176f2b074cf7815e54) >> 128;
    if (absTick & 0x400 != 0) ratio = (ratio * 0xf3392b0822b70005940c7a398e4b70f3) >> 128;
    if (absTick & 0x800 != 0) ratio = (ratio * 0xe7159475a2c29b7443b29c7fa6e889d9) >> 128;
    if (absTick & 0x1000 != 0) ratio = (ratio * 0xd097f3bdfd2022b8845ad8f792aa5825) >> 128;
    if (absTick & 0x2000 != 0) ratio = (ratio * 0xa9f746462d870fdf8a65dc1f90e061e5) >> 128;
    if (absTick & 0x4000 != 0) ratio = (ratio * 0x70d869a156d2a1b890bb3df62baf32f7) >> 128;
    if (absTick & 0x8000 != 0) ratio = (ratio * 0x31be135f97d08fd981231505542fcfa6) >> 128;
    if (absTick & 0x10000 != 0) ratio = (ratio * 0x9aa508b5b7a84e1c677de54f3e99bc9) >> 128;
    if (absTick & 0x20000 != 0) ratio = (ratio * 0x5d6af8dedb81196699c329225ee604) >> 128;
    if (absTick & 0x40000 != 0) ratio = (ratio * 0x2216e584f5fa1ea926041bedfe98) >> 128;
    // If the 19th bit is not 0, since (2^19 = 0x80000=524288), then multiply by 0x2216e584f5fa1ea926041bedfe98, which is 2^128 / 1.0001^(524288/2)
    // The maximum value of tick is 887272, so its binary representation only needs up to 20 bits, starting from 0, the last bit is the 19th bit.
    if (absTick & 0x80000 != 0) ratio = (ratio * 0x48a170391f7dc42444e8fa2) >> 128;

    if (tick > 0) ratio = type(uint256).max / ratio;

    // this divides by 1<<32 rounding up to go from a Q128.128 to a Q128.96.
    // we then downcast because we know the result always fits within 160 bits due to our tick input constraint
    // we round up in the division so getTickAtSqrtRatio of the output price is always consistent
    sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
}
```

Assuming $i > 0$:

$$
\sqrt{p_{Q128128}(i)} = 2^{128} \cdot \sqrt{p(i)} = 2^{128} \cdot 1.0001^{\frac{i}{2}} \\
= \frac{2^{128}}{1.0001^{-\frac{i}{2}}} = \frac{2^{256}}{2^{128} \cdot \sqrt{p(-i)}} = \frac{2^{256}}{\sqrt{p_{Q128128}(-i)}}
$$

Therefore, just calculate the ratio value for $i < 0$, and use $2^{256}$ divided by ratio to get the ratio value for $i > 0$ in `Q128.128` representation:
```solidity
if (tick > 0) ratio = type(uint256).max / ratio;
```

The last line of code right-shifts ratio by 32 bits, converting it into `Q128.96` format:

```solidity
sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
```

This is calculating the square root price $\sqrt{p}$, and since the maximum price $p$ is $2^{128}$, the maximum of $\sqrt{p}$ is $2^{64}$, meaning the integer part only needs up to 64 bits, so the final sqrtPriceX96 can definitely be represented by 160 bits (64+96, i.e., `Q64.96` format).

#### getTickAtSqrtRatio

This method corresponds to formula 6.8 in the white paper:

$$
i_c = \lfloor \log_{\sqrt{1.0001}} \sqrt{P} \rfloor
$$

This method involves calculating logarithms in Solidity, and based on the logarithm formula, it can be deduced:

$$
\log_{\sqrt{1.0001}} \sqrt{P} = \frac{log_2{\sqrt{P}}}{log_2{\sqrt{1.0001}}} = log_2{\sqrt{P}}  \cdot log_{\sqrt{1.0001}}{2}
$$

Since $log_{\sqrt{1.0001}}{2}$ is a constant, we only need to calculate $log_2{\sqrt{P}}$.

Consider the input parameter $\sqrt{P}$ as $x$, and the problem becomes calculating $log_2{x}$.

Divide the result into an integer part $n$ and a fractional part $m$, then:

$$
n \leq log_2{x} = n + m < n + 1
$$

##### Integer Part

For $n$, since:

$$
2^n \leq x < 2^{n+1}
$$

$n$ can be found using binary search:

- For a 256-bit number, $0 \leq n < 256$, which can be represented by 8 binary digits
- Starting from the 8th binary digit (k=7) to the 1st digit (k=0) (from highest to lowest), compare whether $x$ is greater than $2^{2^k} - 1$, if it is, then mark that digit as 1 and right-shift by $2^k$ bits; otherwise, mark it as 0
- The marked 8 binary digits is the value of $n$

A Python code description is as follows:

```python
def find_msb(x):
    msb = 0
    for k in reversed(range(8)): # k = 7, 6, 5. 4, 3, 2, 1, 0
        if x > 2 ** (2 ** k) - 1:
            msb += 2 ** k # Mark this digit as 1, i.e., add 2 ** k
            x /= 2 ** (2 ** k) # Right-shift by 2 ** k bits
    return msb
```

The Solidity code in Uniswap v3 is as follows (please refer to the comments in the code):

```solidity
/// @notice Calculates the greatest tick value such that getRatioAtTick(tick) <= ratio
/// @dev Throws in case sqrtPriceX96 < MIN_SQRT_RATIO, as MIN_SQRT_RATIO is the lowest value getRatioAtTick may
/// ever return.
/// @param sqrtPriceX96 The sqrt ratio for which to compute the tick as a Q64.96
/// @return tick The greatest tick for which the ratio is less than or equal to the input ratio
function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24 tick) {
    // second inequality must be < because the price can never reach the price at the max tick
    require(sqrtPriceX96 >= MIN_SQRT_RATIO && sqrtPriceX96 < MAX_SQRT_RATIO, 'R');
    uint256 ratio = uint256(sqrtPriceX96) << 32; // Left-shift by 32 bits, converting to Q128.128 format

    uint256 r = ratio;
    uint256 msb = 0;

    assembly {
        // If greater than 2 ** (2 ** 7) - 1, save the temporary variable: 2 ** 7
        let f := shl(7, gt(r, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF))
        // msb += 2 ** 7
        msb := or(msb, f)
        // r /= (2 ** (2 ** 7)), i.e., right-shift by 2 ** 7
        r := shr(f, r)
    }
    assembly {
        let f := shl(6, gt(r, 0xFFFFFFFFFFFFFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(5, gt(r, 0xFFFFFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(4, gt(r, 0xFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(3, gt(r, 0xFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(2, gt(r, 0xF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(1, gt(r, 0x3))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := gt(r, 0x1)
        msb := or(msb, f)
    }
```

#### Fractional Part

For the fractional part $m$:

$$
0 \leq m = log_2{x} - n = log_2{\frac{x}{2^n}} < 1 \quad \text{(1.2)}
$$

where $n$ is the msb calculated earlier, i.e., the integer part.

We first consider $\frac{x}{2^n}$ as a whole $r$, thus:

$$
0 \leq log_2{r} < 1
$$

$$
1 \leq r = \frac{x}{2^n} < 2
$$

Here we aim to find $log_2{r}$. If we can express $log_2{r}$ as a converging series, with sufficient decimal places, we can approximate the value of $log_2{r}$.

Using logarithm properties, we can derive the following two equations:

$$
log_2{r} = \frac{2 \cdot log_2{r}}{2} = \frac{log_2{r^2}}{2} \quad \text{(1.3)}
$$

$$
log_2{r} = log_2{2 \cdot \frac{r}{2}} = 1 + log_2{\frac{r}{2}} \quad \text{(1.4)}
$$

By iteratively applying the above two formulas, we can organize the following method:

1. Since initially $log_2{r} < 1$, first apply formula 1.3 to transform the problem into finding $log_2{r^2}$, noting that the base is $\frac{1}{2}$ at this time.
    - In fact, each entry into step 1 adjusts the base to half of its previous value, e.g., the base would be $\frac{1}{4}$ on the second entry, and so on.
2. If $r^2 \geq 2$, then apply formula 1.4, separate out 1, and transform the problem into finding $log_2{\frac{r^2}{2}}$;
    - Since this decision follows after formula 1.3, the separated 1 must be multiplied by the base from the previous step 1. If it is the first time, then record $\frac{1}{2}$, the second time $\frac{1}{4}$, and so on;
    - Since $1 \leq r < 2$ and $2 \leq r^2 < 4$, thus $1 \leq \frac{r^2}{2} < 2$, considering $\frac{r^2}{2}$ as a whole $r$, we return to step 1 to solve for $log_2{r}$, and $1 \leq r < 2$.
3. If $r^2 < 2$, then return to step 1 and continue.

These steps can be summarized in the following formula:

$$
log_2{r} = m_1 \cdot \frac{1}{2} + m_2 \cdot \frac{1}{4} + ... + m_n \cdot \frac{1}{2^n} = \sum^{\infty}_{i=1}(m_i \cdot \frac{1}{2^i}) \quad \text{(1.5)}
$$

where, $\forall m_i \in \{0, 1\}$.

This is actually the binary representation of decimals, where the first binary place represents $2^{-1}$, the second $2^{-2}$, and so forth. In our steps for calculating $log_2{r}$, entering step 2 is equivalent to marking that place as 1; entering step 3 is equivalent to marking it as 0.

Repeating the above process is the procedure for determining the value of each binary bit of the fractional part from high to low (left to right), with each cycle determining one bit. The more cycles performed, the higher the precision of the calculated $log_2{r}$.

Let's continue with the code for calculating the fractional part in Uniswap v3:

```solidity
        if (msb >= 128) r = ratio >> (msb - 127);
        else r = ratio << (127 - msb);
```

Here msb is the integer part $n$. Since ratio is `Q128.128`, if `msb >= 128` then `ratio >= 1`, thus it needs to be shifted right by the integer digits to get the fractional part `ratio >> msb`; `-127` means shifting left by 127 places, using `Q129.127` to represent the fractional part; similarly, if `msb < 128`, then `ratio < 1`, which only has a fractional part, thus by shifting left by `127 - msb` places, the fractional part is made up to 127 places, also represented using `Q129.127`.

Actually, `ratio >> msb` is $\frac{x}{2^n}$ from formula 1.2, which is $r$ in step 1, used in the subsequent iterative algorithm (steps 1-3).

```solidity
    int256 log_2 = (int256(msb) - 128) << 64;
```
Since msb is calculated based on `Q128.128` ratio, `int256(msb) - 128` represents the real value of $n$. `<< 64` uses `Q192.64` to represent $n$.
This line of code is actually using `Q192.64` to save the value of the integer part.

The following code loops to calculate the first 14 decimal places of the binary representation of the fractional part:

```solidity
    assembly {
        // According to step 1, calculate r^2, shifting right by 127 places because both rs are Q129.127
        r := shr(127, mul(r, r))
        // Since 1 <= r^2 < 4, only 2 places are needed to represent the integer part of r^2,
        // therefore, counting from the right, the 129th and 128th places represent the integer part of r^2,
        // shifting right by 128 places, only leaving the 129th place,
        // if this value is 1, then it indicates r >= 2; if 0, then r < 2
        let f := shr(128, r)
        // If f == 1, then log_2 += Q192.64's 1/2
        log_2 := or(log_2, shl(63, f))
        // According to step 2 (i.e., formula 1.4), if r >= 2 (i.e., f == 1), then r /= 2; otherwise, no action, i.e., step 3
        r := shr(f, r)
    }
    // Repeat the above process
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(62, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(61, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(60, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(59, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(58, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(57, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(56, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(55, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(54, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(53, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(52, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(51, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(50, f))
    }
```

The calculated log_2 is $log_2{\sqrt{P}}$ represented in `Q192.64`, with a precision of $2^{-14}$.

```solidity
    int256 log_sqrt10001 = log_2 * 255738958999603826347141; // 128.128 number
```
Because:

$$
\log_{\sqrt{1.0001}} \sqrt{P} = \frac{log_2{\sqrt{P}}}{log_2{\sqrt{1.0001}}} = log_2{\sqrt{P}}  \cdot log_{\sqrt{1.0001}}{2}
$$

Here `255738958999603826347141` is $log_{\sqrt{1.0001}}{2} \cdot 2^{64}$, multiplying two `Q192.64` results in `Q128.64` (without overflow).

Since the precision of $log_2{\sqrt{P}}$ is $2^{-14}$, multiplying by `255738958999603826347141` further amplifies the error, hence it needs to be corrected to ensure the result is the tick closest to the given price.

```solidity
    int24 tickLow = int24((log_sqrt10001 - 3402992956809132418596140100660247210) >> 128);
    int24 tickHi = int24((log_sqrt10001 + 291339464771989622907027621153398088495) >> 128);

    tick = tickLow == tickHi ? tickLow : getSqrtRatioAtTick(tickHi) <= sqrtPriceX96 ? tickHi : tickLow;
```

Where, `3402992956809132418596140100660247210` represents `0.01000049749154292 << 128`, `291339464771989622907027621153398088495` represents `0.8561697375276566 << 128`.

Referencing [this article by abdk](https://hackmd.io/@abdk/SkVJeHK9v), with a precision of $2^{-14}$, the smallest tick error is $−
0.85617$, and the largest error is $0.0100005$. The article also theoretically proves that only when the precision is equal to (or greater than) $2^{-14}$, can the required tick value be calculated in one go.

Our goal is to find the largest tick that satisfies the current conditions, such that the tick's corresponding $\sqrt{P}$ is less than or equal to the given value. Therefore, if the compensated tickHi meets the requirements, it is used preferentially; otherwise, tickLow is used.

Below are the reference articles for this section, for those interested in further reading:

- [Logarithmic Calculations in Solidity](https://liaoph.com/logarithm-in-solidity/)
- [Math in Solidity](https://medium.com/coinmonks/math-in-solidity-part-5-exponent-and-logarithm-9aef8515136e)
- [Logarithm Approximation Precision](https://hackmd.io/@abdk/SkVJeHK9v)

### TickBitmap.sol

TickBitmap uses Bitmap to save the initialization state of Ticks, offering the following methods:

- [position](#position): Returns bitmap index data based on tick
- [flipTick](#flipTick): Flips the tick state
- [nextInitializedTickWithinOneWord](#nextInitializedTickWithinOneWord): Finds the next initialized tick within the same group

#### position

```solidity
/// @notice Computes the position in the mapping where the initialized bit for a tick lives
/// @param tick The tick for which to compute the position
/// @return wordPos The key in the mapping containing the word in which the bit is stored
/// @return bitPos The bit position in the word where the flag is stored
function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
    wordPos = int16(tick >> 8);
    bitPos = uint8(tick % 256);
}
```

Only ticks that are divisible by `tickSpacing` can be recorded in the bitmap, so here the parameter: $tick = \frac{tick}{tickSpacing}$.

The type of `tick` is `int24`, in its binary form from right to left, from low to high, the first 8 bits represent `bitPos`, and the next 16 bits represent `wordPos`, as shown in the diagram:

$$
\overbrace{23,...,8}^{wordPos},\overbrace{7,...,0}^{bitPos}
$$

The bitmap representation of `tick` is: `self[wordPos] ^= 1 << bitPos`.

#### flipTick

When a `tick` flips its initialization status, if its bitmap value is 0, it needs to be changed to 1; otherwise, change it to 0; i.e., "toggle" that bit.

```solidity
/// @notice Flips the initialized state for a given tick from false to true, or vice versa
/// @param self The mapping in which to flip the tick
/// @param tick The tick to flip
/// @param tickSpacing The spacing between usable ticks
function flipTick(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing
) internal {
    require(tick % tickSpacing == 0); // ensure that the tick is spaced
    (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
    uint256 mask = 1 << bitPos;
    self[wordPos] ^= mask;
}
```

First, get the `wordPos` and `bitPos` corresponding to `tick`. Since the final operation on `bitPos` is a bitwise "xor":

> Because 1 xor with any value b (0 or 1) equals ~b; 0 xor with any value b (0 or 1) equals b.

* For the bit corresponding to `tick`, mask is 1
    - If the old value is 1, `1^1=0`, thus the `tick` status changes from "initialized" to "uninitialized";
    - If the old value is 0, `0^1=1`, thus the `tick` status changes from "uninitialized" to "initialized"
* For bits not corresponding to `tick`, mask is 0
    - If the old value is 1, `1^0=1`, thus the status remains unchanged
    - If the old value is 0, `0^0=0`, thus the status remains unchanged

Thus, the above code implements the effect of toggling the `tick` bit.

#### nextInitializedTickWithinOneWord

Based on the `tick` parameter, it finds the nearest initialized `tick` on the bitmap. If not found, it returns the last uninitialized tick in the group.

```solidity
/// @notice Returns the next initialized tick contained in the same word (or adjacent word) as the tick that is either
/// to the left (less than or equal to) or right (greater than) of the given tick
/// @param self The mapping in which to compute the next initialized tick
/// @param tick The starting tick
/// @param tickSpacing The spacing between usable ticks
/// @param lte Whether to search for the next initialized tick to the left (less than or equal to the starting tick)
/// @return next The next initialized or uninitialized tick up to 256 ticks away from the current tick
/// @return initialized Whether the next tick is initialized, as the function only searches within up to 256 ticks
function nextInitializedTickWithinOneWord(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing,
    bool lte
) internal view returns (int24 next, bool initialized) {
    int24 compressed = tick / tickSpacing;
    if (tick < 0 && tick % tickSpacing != 0) compressed--; // round towards negative infinity

    if (lte) {
        (int16 wordPos, uint8 bitPos) = position(compressed);
        // all the 1s at or to the right of the current bitPos
        uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
        uint256 masked = self[wordPos] & mask;

        // if there are no initialized ticks to the right of or at the current tick, return rightmost in the word
        initialized = masked != 0;
        // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
        next = initialized
            ? (compressed - int24(bitPos - BitMath.mostSignificantBit(masked))) * tickSpacing
            : (compressed - int24(bitPos)) * tickSpacing;
    } else {
        // start from the word of the next tick, since the current tick state doesn't matter
        (int16 wordPos, uint8 bitPos) = position(compressed + 1);
        // all the 1s at or to the left of the bitPos
        uint256 mask = ~((1 << bitPos) - 1);
        uint256 masked = self[wordPos] & mask;

        // if there are no initialized ticks to the left of the current tick, return leftmost in the word
        initialized = masked != 0;
        // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
        next = initialized
            ? (compressed + 1 + int24(BitMath.leastSignificantBit(masked) - bitPos)) * tickSpacing
            : (compressed + 1 + int24(type(uint8).max - bitPos)) * tickSpacing;
    }
}
```

If `lte == true`, i.e., looking for a value less than or equal to the current `tick`, the problem transforms into: finding if there are `1`s in the lower bits.

```solidity
if (lte) {
    (int16 wordPos, uint8 bitPos) = position(compressed);
    // all the 1s at or to the right of the current bitPos
    uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
    uint256 masked = self[wordPos] & mask;
}
```

Where, mask's value sets all bits less than or equal to `bitPos` to 1, e.g., if `bitPos = 7`, then `mask in binary = 1111111`; masked retains the bitmap values less than or equal to `bitPos`, e.g., if `self[wordPos] = 110101011`, then `masked = 110101011 & 1111111 = 000101011`.

If `masked != 0`, it indicates that on the current direction (`lte`) there are initialized `ticks` on that `wordPos`; otherwise, it means all ticks in that direction are uninitialized.

```solidity
// if there are no initialized ticks to the right of or at the current tick, return rightmost in the word
initialized = masked != 0;
```

If there are initialized `ticks`, locate the highest bit of 1 in `masked`; if not, return the last `tick` in the current direction.

```solidity
// overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
next = initialized
    ? (compressed - int24(bitPos - BitMath.mostSignificantBit(masked))) * tickSpacing
    : (compressed - int24(bitPos)) * tickSpacing;
```

`BitMath.mostSignificantBit(masked)` finds the highest bit of 1 in `masked` using a binary search method. For a detailed explanation of this algorithm, refer to the section on [TickMath logarithm calculations](#Integer Part). In short, `mostSignificantBit(masked)` returns a number `n` such that:

$$
2^n \leq masked < 2^{n+1}
$$

For example, if `masked = 000101011`, `mostSignificantBit` will return 5 because the highest bit of 1 is in the 5th position (starting from 0): 000`1`01011.

`compressed - int24(bitPos)` represents the first `tick` of the current `wordPos`, `compressed - int24(bitPos - BitMath.mostSignificantBit(masked))` represents the `tick` corresponding to that highest `bitPos`; `* tickSpacing` restores to the original `tick` value because `tick` is divided by `tickSpacing` when saving in the bitmap.

Similarly, when searching for the first initialized `tick` in the direction greater than the current `tick`, the method is similar, but `mostSignificantBit` needs to be replaced with `leastSignificantBit`, i.e., starting from the `tick` bit (excluding), searching upwards for the first `bitPos` bit that is 1.

```solidity
else {
    // start from the word of the next tick, since the current tick state doesn't matter
    (int16 wordPos, uint8 bitPos) = position(compressed + 1);
    // all the 1s at or to the left of the bitPos
    uint256 mask = ~((1 << bitPos) - 1);
    uint256 masked = self[wordPos] & mask;

    // if there are no initialized ticks to the left of the current tick, return leftmost in the word
    initialized = masked != 0;
    // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
    next = initialized
        ? (compressed + 1 + int24(BitMath.leastSignificantBit(masked) - bitPos)) * tickSpacing
        : (compressed + 1 + int24(type(uint8).max - bitPos)) * tickSpacing;
}
```

### Position.sol

`Position.sol` manages position-related information, including the following methods:

- [get](#get): Retrieves a position object
- [update](#update1): Updates a position

A position can be uniquely determined by "owner" `owner`, "lower tick" `tickLower`, and "upper tick" `tickUpper`. For the same trading pair pool, each user can create multiple positions with different price ranges, but can only create one position with the same price range; different users can create positions with the same price range.

#### get

Returns a position object based on `owner`, `tickLower`, and `tickUpper`:

```solidity
/// @notice Returns the Info struct of a position, given an owner and position boundaries
/// @param self The mapping containing all user positions
/// @param owner The address of the position owner
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @return position The position info struct of the given owners' position
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 tickLower,
    int24 tickUpper
) internal view returns (Position.Info storage position) {
    position = self[keccak256(abi.encodePacked(owner, tickLower, tickUpper))];
}
```

#### update

Updates the liquidity and withdrawable tokens of a position. Note, this method is only triggered during `mint` and `burn`; `swap` does not update position information.

```solidity
/// @notice Credits accumulated fees to a user's position
/// @param self The individual position to update
/// @param liquidityDelta The change in pool liquidity as a result of the position update
/// @param feeGrowthInside0X128 The all-time fee growth in token0, per unit of liquidity, inside the position's tick boundaries
/// @param feeGrowthInside1X128 The all-time fee growth in token1, per unit of liquidity, inside the position's tick boundaries
function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal {
    Info memory _self = self;

    uint128 liquidityNext;
    if (liquidityDelta == 0) {
        require(_self.liquidity > 0, 'NP'); // disallow pokes for 0 liquidity positions
        liquidityNext = _self.liquidity;
    } else {
        liquidityNext = LiquidityMath.addDelta(_self.liquidity, liquidityDelta);
    }
```

Updating liquidity:

- If it's `mint`, then `liquidityDelta > 0`
- If it's `burn`, then `liquidityDelta < 0`

```solidity
    // calculate accumulated fees
    uint128 tokensOwed0 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0X128 - _self.feeGrowthInside0LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );
    uint128 tokensOwed1 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1X128 - _self.feeGrowthInside1LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );
```

Calculates the receivable fees for `token0` and `token1` based on the fee growth per liquidity inside the position's tick boundaries since the last update.

```solidity
    // update the position
    if (liquidityDelta != 0) self.liquidity = liquidityNext;
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;
    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        // overflow is acceptable, have to withdraw before you hit type(uint128).max fees
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

Updates the position liquidity, this period's fee per liquidity, and withdrawable token amounts.

### SwapMath.sol

#### computeSwapStep

SwapMath has only one method, `computeSwapStep`, which calculates the input and output for a single swap step.

This method essentially implements the concept described in Section 6.2.3 of the white paper: conducting trades within a single tick. In this scenario, the liquidity $L$ remains constant, while only the price $\sqrt{P}$ changes.

Parameters are as follows:

- `sqrtRatioCurrentX96`: Current price
- `sqrtRatioTargetX96`: Target price
- `liquidity`: Available liquidity
- `amountRemaining`: Remaining input tokens
- `feePips`: Transaction fee

Return values:

- `sqrtRatioNextX96`: Price after swap
- `amountIn`: Input tokens consumed in this swap
- `amountOut`: Output tokens produced in this swap
- `feeAmount`: Transaction fee (including protocol fee)

```solidity
function computeSwapStep(
    uint160 sqrtRatioCurrentX96,
    uint160 sqrtRatioTargetX96,
    uint128 liquidity,
    int256 amountRemaining,
    uint24 feePips
)
    internal
    pure
    returns (
        uint160 sqrtRatioNextX96,
        uint256 amountIn,
        uint256 amountOut,
        uint256 feeAmount
    )
{
    bool zeroForOne = sqrtRatioCurrentX96 >= sqrtRatioTargetX96;
    bool exactIn = amountRemaining >= 0;
```

We mentioned in the [swap](#swap) section that for a swap operation, based on the values of `zeroForOne` and `exactIn`, there are four combinations:

| zeroForOne | exactInput | swap |
| --- | --- | --- |
| true | true | Input a fixed amount of `token0`, output the maximum amount of `token1` |
| true | false | Input the minimum amount of `token0`, output a fixed amount of `token1` |
| false | true | Input a fixed amount of `token1`, output the maximum amount of `token0` |
| false | false | Input the minimum amount of `token1`, output a fixed amount of `token0` |

```solidity
if (exactIn) {
    uint256 amountRemainingLessFee = FullMath.mulDiv(uint256(amountRemaining), 1e6 - feePips, 1e6);
    amountIn = zeroForOne
        ? SqrtPriceMath.getAmount0Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, true)
        : SqrtPriceMath.getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, true);
    if (amountRemainingLessFee >= amountIn) sqrtRatioNextX96 = sqrtRatioTargetX96;
    else
        sqrtRatioNextX96 = SqrtPriceMath.getNextSqrtPriceFromInput(
            sqrtRatioCurrentX96,
            liquidity,
            amountRemainingLessFee,
            zeroForOne
        );
}
```

If the input amount is fixed, then the transaction fee needs to be deducted from the input tokens.

Since the unit of `feePips` is one-hundredth of a basis point, or $\frac{1}{10^6}$, deduct the fee according to the following formula:

$$
amountRemaining \cdot (1 - \frac{feePips}{10^6})
$$

Using [getAmount0Delta](#getAmount0Delta) or [getAmount1Delta](#getAmount1Delta), calculate the required input token amount `amountIn` based on the current price, target price, and available liquidity:

- If swapping from `token0` to `token1`, then the input token is `token0`, hence calculate `amount0`
- If swapping from `token1` to `token0`, then the input token is `token1`, hence calculate `amount1`

Note, the last parameter `roundUp`, when calculating `amountIn` set `roundUp = true`, meaning the contract can only charge more and not less, otherwise, the contract would lose funds; similarly, when calculating `amountOut` set `roundUp = false`, meaning the contract can pay less, but not more.

- If the available token amount is greater than `amountIn`, it indicates that the step transaction can be completed, therefore the price after the swap equals the target price
- Otherwise, calculate the price after the swap based on the current price, available liquidity, and available tokens `amountRemainingLessFee`, we will specifically introduce the calculation method in [SqrtPriceMath.getNextSqrtPriceFromInput](#getNextSqrtPriceFromInput)

```solidity
else {
    amountOut = zeroForOne
        ? SqrtPriceMath.getAmount1Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, false)
        : SqrtPriceMath.getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, false);
    if (uint256(-amountRemaining) >= amountOut) sqrtRatioNextX96 = sqrtRatioTargetX96;
    else
        sqrtRatioNextX96 = SqrtPriceMath.getNextSqrtPriceFromOutput(
            sqrtRatioCurrentX96,
            liquidity,
            uint256(-amountRemaining),
            zeroForOne
        );
}
```

If the input amount is not fixed, meaning the output amount is fixed, `amountRemaining` is used as the output token amount:

- If swapping from `token0` to `token1`, then the output token is `token1`, hence calculate `amount1`
- If swapping from `token1` to `token0`, then the output token is `token0`, hence calculate `amount0`

First, calculate the producible output token amount `amountOut` based on the current price, target price, and available liquidity. Note, here, opposite to the above, if swapping from `token0` to `token1`, the `token1` amount needs to be calculated using the `SqrtPriceMath.getAmount1Delta` method.

When specifying a fixed output token, the passed `amountRemaining` is a negative value:

- If the absolute value of `amountOut` is less than the required output token amount, it means the entire swap can be executed, hence the price after the swap equals the target price
- Otherwise, calculate the price after the swap based on the available output `amountRemaining`

```solidity
    bool max = sqrtRatioTargetX96 == sqrtRatioNextX96;
```

If the price after the swap equals the target price, it means the swap is complete.

```solidity
// get the input/output amounts
if (zeroForOne) {
    amountIn = max && exactIn
        ? amountIn
        : SqrtPriceMath.getAmount0Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, true);
    amountOut = max && !exactIn
        ? amountOut
        : SqrtPriceMath.getAmount1Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, false);
} else {
    amountIn = max && exactIn
        ? amountIn
        : SqrtPriceMath.getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, true);
    amountOut = max && !exactIn
        ? amountOut
        : SqrtPriceMath.getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, false);
}
```

Calculates the required input `amountIn` and produced output `amountOut` for this swap:

- If swapping from `token0` to `token1`
    - Required input `amountIn`
        - If the swap is complete and the input is fixed, then the required input is `amountIn`
        - Otherwise (incomplete swap, or fixed output), use `getAmount0Delta` to calculate the required input based on the price range and liquidity
    - Produced output `amountOut`
        - If the swap is complete and the output is fixed, then the produced output is `amountOut`
        - Otherwise (incomplete swap, or fixed input), use `getAmount1Delta` to calculate the produced output based on the price range and liquidity
- If swapping from `token1` to `token0`
    - Required input `amountIn`
        - If the swap is complete and the input is fixed, then the required input is `amountIn`
        - Otherwise (incomplete swap, or fixed output), use `getAmount1Delta` to calculate the required input based on the price range and liquidity
    - Produced output `amountOut`
        - If the swap is complete and the output is fixed, then the produced output is `amountOut`
        - Otherwise (incomplete swap, or fixed input), use `getAmount0Delta` to calculate the produced output based on the price range and liquidity

```solidity
// cap the output amount to not exceed the remaining output amount
if (!exactIn && amountOut > uint256(-amountRemaining)) {
    amountOut = uint256(-amountRemaining);
}
```

Ensures the produced output does not exceed the specified output.

```solidity
if (exactIn && sqrtRatioNextX96 != sqrtRatioTargetX96) {
    // we didn't reach the target, so take the remainder of the maximum input as fee
    feeAmount = uint256(amountRemaining) - amountIn;
} else {
    feeAmount = FullMath.mulDivRoundingUp(amountIn, feePips, 1e6 - feePips);
}
```

Calculates the transaction fee (including the protocol fee):

- If the input is fixed and the target price is not reached, equivalent to the last swap, then the original input token `amountRemaining` minus the input required for this swap `amountIn`, is the fee;
- Otherwise, the fee needs to be calculated based on `amountIn`; note, besides the last swap mentioned above, here `amountIn` and `amountOut` do not include the fee, hence, calculating the fee needs to be divided by `1e6 - feePips`, not `1e6`.


### SqrtPriceMath.sol

#### getNextSqrtPriceFromAmount0RoundingUp

Calculates the target price based on the current price, `liquidity`, and $\Delta{x}$.

According to whitepaper formula 6.16:

$$
\Delta{x} = \Delta{\frac{1}{\sqrt{P}}} \cdot L
$$

Assuming $\sqrt{P_a} > \sqrt{P_b}$, then $x_a < x_b$:

$$
\Delta{x} = x_b - x_a
$$

If calculating $\sqrt{P_b}$ given $\sqrt{P_a}$:

$$
\frac{1}{\sqrt{P_b}} = \frac{\Delta{x}}{L} + \frac{1}{\sqrt{P_a}} = \frac{1}{L} \cdot (\Delta{x} + \frac{L}{\sqrt{P_a}}) \quad \text{(1.1)}
$$

$$
\sqrt{P_b} = \frac{L}{\Delta{x} + \frac{L}{\sqrt{P_a}}} \quad \text{(1.2)}
$$

$$
{\sqrt{P_b}} = \frac{L \cdot \sqrt{P_a}}{L + \Delta{x} \cdot \sqrt{P_a}}  \quad \text{(1.3)}
$$

If calculating $\sqrt{P_a}$ given $\sqrt{P_b}$:

$$
\frac{1}{\sqrt{P_a}} = \frac{1}{\sqrt{P_b}} - \frac{\Delta{x}}{L}  \quad \text{(1.4)}
$$

$$
{\sqrt{P_a}} = \frac{L \cdot \sqrt{P_b}}{L - \Delta{x} \cdot \sqrt{P_b}}  \quad \text{(1.5)}
$$

```solidity
/// @notice Gets the next sqrt price given a delta of token0
/// @dev Always rounds up, because in the exact output case (increasing price) we need to move the price at least
/// far enough to get the desired output amount, and in the exact input case (decreasing price) we need to move the
/// price less in order to not send too much output.
/// The most precise formula for this is liquidity * sqrtPX96 / (liquidity +- amount * sqrtPX96),
/// if this is impossible because of overflow, we calculate liquidity / (liquidity / sqrtPX96 +- amount).
/// @param sqrtPX96 The starting price, i.e. before accounting for the token0 delta
/// @param liquidity The amount of usable liquidity
/// @param amount How much of token0 to add or remove from virtual reserves
/// @param add Whether to add or remove the amount of token0
/// @return The price after adding or removing amount, depending on add
function getNextSqrtPriceFromAmount0RoundingUp(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amount,
    bool add
) internal pure returns (uint160) {
    // we short circuit amount == 0 because the result is otherwise not guaranteed to equal the input price
    if (amount == 0) return sqrtPX96;
    uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION;

    if (add) {
        uint256 product;
        if ((product = amount * sqrtPX96) / amount == sqrtPX96) {
            uint256 denominator = numerator1 + product;
            if (denominator >= numerator1)
                // always fits in 160 bits
                return uint160(FullMath.mulDivRoundingUp(numerator1, sqrtPX96, denominator));
        }

        return uint160(UnsafeMath.divRoundingUp(numerator1, (numerator1 / sqrtPX96).add(amount)));
    } else {
        uint256 product;
        // if the product overflows, we know the denominator underflows
        // in addition, we must check that the denominator does not underflow
        require((product = amount * sqrtPX96) / amount == sqrtPX96 && numerator1 > product);
        uint256 denominator = numerator1 - product;
        return FullMath.mulDivRoundingUp(numerator1, sqrtPX96, denominator).toUint160();
    }
}
```

- When `add = true`, i.e., calculating $\sqrt{P_b}$ given $\sqrt{P_a}$
    - If $\Delta{x} \cdot \sqrt{P_a}$ does not overflow, then use formula 1.3 to calculate $\sqrt{P_b}$, because this method has the highest precision
    - Otherwise, use formula 1.2 to calculate $\sqrt{P_b}$
- When `add = false`, i.e., calculating $\sqrt{P_a}$ given $\sqrt{P_b}$, use the formula 1.5

#### getNextSqrtPriceFromAmount1RoundingDown

Calculate the target price based on the current price, `liquidity`, and $\Delta{y}$.

According to the whitepaper formula 6.13:

$$
\Delta{\sqrt{P}} = \frac{\Delta{y}}{L} \quad \text{(6.13)}
$$

Assuming $\sqrt{P_a} > \sqrt{P_b}$, then $y_a > y_b$:

$$
\Delta{y} = y_a - y_b
$$

If $\sqrt{P_b}$ is known to calculate $\sqrt{P_a}$, then:

$$
\sqrt{P_a} = \sqrt{P_b} + \frac{\Delta{y}}{L} \quad \text{(1.6)}
$$

If $\sqrt{P_a}$ is known to calculate $\sqrt{P_b}$, then:

$$
\sqrt{P_b} = \sqrt{P_a} - \frac{\Delta{y}}{L} \quad \text{(1.7)}
$$

```solidity
/// @notice Gets the next sqrt price given a delta of token1
/// @dev Always rounds down, because in the exact output case (decreasing price) we need to move the price at least
/// far enough to get the desired output amount, and in the exact input case (increasing price) we need to move the
/// price less in order to not send too much output.
/// The formula we compute is within <1 wei of the lossless version: sqrtPX96 +- amount / liquidity
/// @param sqrtPX96 The starting price, i.e., before accounting for the token1 delta
/// @param liquidity The amount of usable liquidity
/// @param amount How much of token1 to add, or remove, from virtual reserves
/// @param add Whether to add, or remove, the amount of token1
/// @return The price after adding or removing `amount`
function getNextSqrtPriceFromAmount1RoundingDown(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amount,
    bool add
) internal pure returns (uint160) {
    // if we're adding (subtracting), rounding down requires rounding the quotient down (up)
    // in both cases, avoid a mulDiv for most inputs
    if (add) {
        uint256 quotient =
            (
                amount <= type(uint160).max
                    ? (amount << FixedPoint96.RESOLUTION) / liquidity
                    : FullMath.mulDiv(amount, FixedPoint96.Q96, liquidity)
            );

        return uint256(sqrtPX96).add(quotient).toUint160();
    } else {
        uint256 quotient =
            (
                amount <= type(uint160).max
                    ? UnsafeMath.divRoundingUp(amount << FixedPoint96.RESOLUTION, liquidity)
                    : FullMath.mulDivRoundingUp(amount, FixedPoint96.Q96, liquidity)
            );

        require(sqrtPX96 > quotient);
        // always fits 160 bits
        return uint160(sqrtPX96 - quotient);
    }
}
```

* When `add = true`, that is, according to formula 1.6, calculate $\sqrt{P_a}$ based on $\sqrt{P_b}$
* When `add = false`, according to formula 1.7, calculate $\sqrt{P_b}$ based on $\sqrt{P_a}$

#### getNextSqrtPriceFromInput

Calculate the next price based on the input token, that is, the price after adding an `amountIn` of the token:

```solidity
/// @notice Gets the next sqrt price given an input amount of token0 or token1
/// @dev Throws if price or liquidity are 0, or if the next price is out of bounds
/// @param sqrtPX96 The starting price, i.e., before accounting for the input amount
/// @param liquidity The amount of usable liquidity
/// @param amountIn How much of token0, or token1, is being swapped in
/// @param zeroForOne Whether the amount in is token0 or token1
/// @return sqrtQX96 The price after adding the input amount to token0 or token1
function getNextSqrtPriceFromInput(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amountIn,
    bool zeroForOne
) internal pure returns (uint160 sqrtQX96) {
    require(sqrtPX96 > 0);
    require(liquidity > 0);

    // round to make sure that we don't pass the target price
    return
        zeroForOne
            ? getNextSqrtPriceFromAmount0RoundingUp(sqrtPX96, liquidity, amountIn, true)
            : getNextSqrtPriceFromAmount1RoundingDown(sqrtPX96, liquidity, amountIn, true);
}
```

#### getNextSqrtPriceFromOutput

Calculate the next price based on the output token, that is, the price after removing an `amountOut` of the token:

```solidity
/// @notice Gets the next sqrt price given an output amount of token0 or token1
/// @dev Throws if price or liquidity are 0 or the next price is out of bounds
/// @param sqrtPX96 The starting price before accounting for the output amount
/// @param liquidity The amount of usable liquidity
/// @param amountOut How much of token0, or token1, is being swapped out
/// @param zeroForOne Whether the amount out is token0 or token1
/// @return sqrtQX96 The price after removing the output amount of token0 or token1
function getNextSqrtPriceFromOutput(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amountOut,
    bool zeroForOne
) internal pure returns (uint160 sqrtQX96) {
    require(sqrtPX96 > 0);
    require(liquidity > 0);

    // round to make sure that we pass the target price
    return
        zeroForOne
            ? getNextSqrtPriceFromAmount1RoundingDown(sqrtPX96, liquidity, amountOut, false)
            : getNextSqrtPriceFromAmount0RoundingUp(sqrtPX96, liquidity, amountOut, false);
}
```

#### getAmount0Delta

This method calculates formula 6.16 from the whitepaper:

$$
\Delta{x} = \Delta{\frac{1}{\sqrt{P}}} \cdot L
$$

Expanded as:

$$
amount0 = x_b - x_a = L \cdot (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}}) = L \cdot (\frac{\sqrt{P_a} - \sqrt{P_b}}{\sqrt{P_a} \cdot \sqrt{P_b}})
$$

```solidity
/// @notice Gets the amount0 delta between two prices
/// @dev Calculates liquidity / sqrt(lower) - liquidity / sqrt(upper),
/// i.e. liquidity * (sqrt(upper) - sqrt(lower)) / (sqrt(upper) * sqrt(lower))
/// @param sqrtRatioAX96 A sqrt price
/// @param sqrtRatioBX96 Another sqrt price
/// @param liquidity The amount of usable liquidity
/// @param roundUp Whether to round the amount up or down
/// @return amount0 Amount of token0 required to cover a position of size liquidity between the two passed prices
function getAmount0Delta(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint128 liquidity,
    bool roundUp
) internal pure returns (uint256 amount0) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 numerator2 = sqrtRatioBX96 - sqrtRatioAX96;

    require(sqrtRatioAX96 > 0);

    return
        roundUp
            ? UnsafeMath.divRoundingUp(
                FullMath.mulDivRoundingUp(numerator1, numerator2, sqrtRatioBX96),
                sqrtRatioAX96
            )
            : FullMath.mulDiv(numerator1, numerator2, sqrtRatioBX96) / sqrtRatioAX96;
}
```

#### getAmount1Delta

Similarly, this method calculates formula 6.7 from the whitepaper:

$$
L = \frac{\Delta{Y}}{\Delta{\sqrt{P}}}
$$

Expanded as:

$$
amount1 = y_b - y_a = L \cdot \Delta{\sqrt{P}} = L \cdot (\sqrt{P_b} - \sqrt{P_a})
$$

```solidity
/// @notice Gets the amount1 delta between two prices
/// @dev Calculates liquidity * (sqrt(upper) - sqrt(lower))
/// @param sqrtRatioAX96 A sqrt price
/// @param sqrtRatioBX96 Another sqrt price
/// @param liquidity The amount of usable liquidity
/// @param roundUp Whether to round the amount up, or down
/// @return amount1 Amount of token1 required to cover a position of size liquidity between the two passed prices
function getAmount1Delta(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint128 liquidity,
    bool roundUp
) internal pure returns (uint256 amount1) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    return
        roundUp
            ? FullMath.mulDivRoundingUp(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96)
            : FullMath.mulDiv(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96);
```

### Oracle.sol

This contract defines methods related to the oracle.

At the time of creating a pair contract, an array of length 1 is initialized by default for storing observation data; any user can call the [increaseObservationCardinalityNext](#increaseObservationCardinalityNext) method of `UniswapV3Pool.sol` to expand the oracle's observation space, but must bear the costs associated with expanding the space.

An oracle observation includes the following:

* `blockTimestamp`: The time when the observation was recorded
* `tickCumulative`: The cumulative `tick` as of the observation time
* `secondsPerLiquidityCumulativeX128`: The cumulative seconds per liquidity up to the observation time
* `initialized`: Whether the observation is initialized

Observation structure is defined as follows:

```solidity
struct Observation {
    // the block timestamp of the observation
    uint32 blockTimestamp;
    // the tick accumulator, i.e. tick * time elapsed since the pool was first initialized
    int56 tickCumulative;
    // the seconds per liquidity, i.e. seconds elapsed / max(1, liquidity) since the pool was first initialized
    uint160 secondsPerLiquidityCumulativeX128;
    // whether or not the observation is initialized
    bool initialized;
}
```

This contract includes the following methods:

* [transform](#transform): Returns a new observation object based on the previous one (but does not write the observation)
* [initialize](#initialize2): Initializes the observation array
* [write](#write): Writes an observation
* [grow](#grow): Expands the oracle observation space
* [lte](#lte): Compares two timestamps for size
* [binarySearch](#binarySearch): Binary search for the observation boundary at a specified time
* [getSurroundingObservations](#getSurroundingObservations): Gets the observation boundary at a specified time
* [observeSingle](#observeSingle): Retrieves observation data at a specified time
* [observe](#observe): Batch retrieves observation data at specified times

#### transform

Returns a temporary observation object based on the previous observation, but does not write the observation.

```solidity
function transform(
    Observation memory last,
    uint32 blockTimestamp,
    int24 tick,
    uint128 liquidity
) private pure returns (Observation memory) {
    uint32 delta = blockTimestamp - last.blockTimestamp;
    return
        Observation({
            blockTimestamp: blockTimestamp,
            tickCumulative: last.tickCumulative + int56(tick) * delta,
            secondsPerLiquidityCumulativeX128: last.secondsPerLiquidityCumulativeX128 +
                ((uint160(delta) << 128) / (liquidity > 0 ? liquidity : 1)),
            initialized: true
        });
}
```

First, calculate the time delta between the current time and the last observation: `delta = blockTimestamp - last.blockTimestamp`.

Then, mainly calculate `tickCumulative` and `secondsPerLiquidityCumulativeX128`,

Where: `tickCumulative: last.tickCumulative + int56(tick) * delta`.

According to the whitepaper formulas 5.3-5.5:

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{\sum_{i=t_1}^{t_2} \log_{1.0001}(P_i)}{t_2 - t_1} \quad \text{(5.3)}
$$

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1} \quad \text{(5.4)}
$$

$$
P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2 - t_1}} \quad \text{(5.5)}
$$

Here, the saved `tickCumulative` is $a_{t_n}$, corresponding to the formula:

$$
tickCumulative = \sum_{i=0}^{t_n} \log_{1.0001}(P_i)
$$

Similarly, the cumulative seconds per liquidity `secondsPerLiquidityCumulative` is:

$$
secondsPerLiquidityCumulative = \sum_{i=0}^{n} \frac{t_i}{L_i}
$$

#### initialize

Initialize the oracle storage space, setting the first `slot`.

```solidity
/// @notice Initialize the oracle array by writing the first slot. Called once for the lifecycle of the observations array
/// @param self The stored oracle array
/// @param time The time of the oracle initialization, via block.timestamp truncated to uint32
/// @return cardinality The number of populated elements in the oracle array
/// @return cardinalityNext The new length of the oracle array, independent of population
function initialize(Observation[65535] storage self, uint32 time)
    internal
    returns (uint16 cardinality, uint16 cardinalityNext)
{
    self[0] = Observation({
        blockTimestamp: time,
        tickCumulative: 0,
        secondsPerLiquidityCumulativeX128: 0,
        initialized: true
    });
    return (1, 1);
}
```

The default return values for `cardinality` and `cardinalityNext` are both 1, meaning that only one observation point data can be stored.

This method can only be called once, and is actually called in the [initialize](#initialize) method of `UniswapV3Pool.sol`:

```solidity
(uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());
```

#### write

Writes a single observation point data. According to the previous description, a write operation can only be triggered when `mint`, `burn`, and `swap` operations occur in `UniswapV3Pool`.

The oracle observation point array can be seen as a circular list, with the writable space determined by `cardinality` and `cardinalityNext`; when the array space is full, it will continue to overwrite from the 0th position.

Parameters of this method are as follows:

* `self`: The oracle observation point array
* `index`: The index of the last written observation point, starting from 0
* `blockTimestamp`: The time of the observation point to be added
* `tick`: The `tick` of the observation point to be added
* `liquidity`: The global available liquidity at the time of the observation point
* `cardinality`: The current length of the observation point array (writable space)
* `cardinalityNext`: The latest length of the observation point array (extended)

```solidity
function write(
    Observation[65535] storage self,
    uint16 index,
    uint32 blockTimestamp,
    int24 tick,
    uint128 liquidity,
    uint16 cardinality,
    uint16 cardinalityNext
) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    Observation memory last = self[index];

    // early return if we've already written an observation this block
    if (last.blockTimestamp == blockTimestamp) return (index, cardinality);
```

If the time of the new observation point is the same as the last observation point time, it will return directly, ensuring that no more than one observation point can be written per block.

```solidity
    // if the conditions are right, we can bump the cardinality
    if (cardinalityNext > cardinality && index == (cardinality - 1)) {
        cardinalityUpdated = cardinalityNext;
    } else {
        cardinalityUpdated = cardinality;
    }
```

* If `cardinalityNext > cardinality`, it means the oracle array has been expanded; if `index == (cardinality - 1)`, which means the last write position is the last observation point, then this time it needs to continue writing into the expanded space, therefore `cardinalityUpdated` uses the expanded array length `cardinalityNext`;
* Otherwise, continue using the old array length `cardinality`, because it has not yet been filled.

```solidity
    indexUpdated = (index + 1) % cardinalityUpdated;
```

Update the index `indexUpdated` of this write operation into the observation point array, `% cardinalityUpdated` is to calculate the index for circular writing.

```solidity
    self[indexUpdated] = transform(last, blockTimestamp, tick, liquidity);
```

Call the [transform](#transform) method to calculate the latest observation point data, and write it at the `indexUpdated` position in the observation point array.

#### grow

Expands the observation point array, increasing the number of writable observation points. Since the contract can only save 1 observation point by default, to support more observation points, any user can manually call the contract to expand the observation point array.

> Note, the `grow` method is marked as `internal`, and users actually expand the observation point array through the [increaseObservationCardinalityNext](#increaseObservationCardinalityNext) method of `UniswapV3Pool.sol`.

The expansion method will trigger `SSTORE` operations, so the user calling this method needs to pay the gas cost associated with it.

```solidity
/// @notice Prepares the oracle array to store up to `next` observations
/// @param self The stored oracle array
/// @param current The current next cardinality of the oracle array
/// @param next The proposed next cardinality which will be populated in the oracle array
/// @return next The next cardinality which will be populated in the oracle array
function grow(
    Observation[65535] storage self,
    uint16 current,
    uint16 next
) internal returns (uint16) {
    require(current > 0, 'I');
    // no-op if the passed next value isn't greater than the current next value
    if (next <= current) return current;
    // store in each slot to prevent fresh SSTOREs in swaps
    // this data will not be used because the initialized boolean is still false
    for (uint16 i = current; i < next; i++) self[i].blockTimestamp = 1;
    return next;
}
```

#### lte

Compares two timestamps.

Since the contract uses `uint32` type to represent timestamps, its maximum value $2^{32}-1 = 4294967295$, corresponding to the time `February 7, 2106, 6:28:15 AM GMT+00:00`, if two times `a` and `b` ($a < b$) are just on both sides of `uint32`'s maximum value, `b` will overflow, therefore $a > b$, directly comparing the values will lead to incorrect results, thus a unified time is needed.

> Note: In reality, there will be an overflow issue every approximately 136 years.

The method accepts 3 parameters: `time`, `a`, and `b`; where `time` is the reference time, `a` and `b` are logically less than or equal to `time`; the method returns a `bool`, indicating whether `a` is logically less than or equal to `b`.

```solidity
/// @notice comparator for 32-bit timestamps
/// @dev safe for 0 or 1 overflows, a and b _must_ be chronologically before or equal to time
/// @param time A timestamp truncated to 32 bits
/// @param a A comparison timestamp from which to determine the relative position of `time`
/// @param b From which to determine the relative position of `time`
/// @return bool Whether `a` is chronologically <= `b`
function lte(
    uint32 time,
    uint32 a,
    uint32 b
) private pure returns (bool) {
    // if there hasn't been overflow, no need to adjust
    if (a <= time && b <= time) return a <= b;

    uint256 aAdjusted = a > time ? a : a + 2**32;
    uint256 bAdjusted = b > time ? b : b + 2**32;

    return aAdjusted <= bAdjusted;
}
```

* If `a` and `b` are both less than or equal to `time`, it means there has been no overflow, so it directly returns `a <= b`
* Since `a` and `b` are both logically less than or equal to `time`
  * If $a > time$, it means overflow occurred, `aAdjusted` is the time `a` plus $2^{32}$
  * Similarly, `bAdjusted` is the time `b` plus $2^{32}$
  * Finally, compare the two adjusted times `aAdjusted` and `bAdjusted`

Thus, this method is overflow safe when `a`, `b`, and `time` have 0 to \(2^{32}\) time delta.

> It cannot cross two \(2^{32} - 1\).

#### binarySearch

Performs a binary search for the observation point of a specified target time.

Parameters are as follows:

* `self`: The observation point array
* `time`: The current block time
* `target`: The target time
* `index`: The index of the last written observation point
* `cardinality`: The current length of the observation point array (writable space)

```solidity
/// @notice Fetches the observations beforeOrAt and atOrAfter a target, i.e. where [beforeOrAt, atOrAfter] is satisfied.
/// The result may be the same observation, or adjacent observations.
/// @dev The answer must be contained in the array, used when the target is located within the stored observation
/// boundaries: older than the most recent observation and younger, or the same age as, the oldest observation
/// @param self The stored oracle array
/// @param time The current block.timestamp
/// @param target The timestamp at which the reserved observation should be for
/// @param index The index of the observation that was most recently written to the observations array
/// @param cardinality The number of populated elements in the oracle array
/// @return beforeOrAt The observation recorded before, or at, the target
/// @return atOrAfter The observation recorded at, or after, the target
function binarySearch(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    uint16 index,
    uint16 cardinality
) private view returns (Observation memory beforeOrAt, Observation memory atOrAfter) {
```

For the binary search algorithm, it is first necessary to confirm the left and right boundary points.

```solidity
    uint256 l = (index + 1) % cardinality; // oldest observation
    uint256 r = l + cardinality - 1; // newest observation
```

Since the last index is `index`, moving one position to the right (and taking modulo) is the left boundary point `l` (the oldest index, if a full round has been written); the right boundary point is `l + cardinality - 1`. Note that the right boundary point `r` does not take modulo, because in a binary search, the right node must not be smaller than the left node.

```solidity
    uint256 i;
    while (true) {
        i = (l + r) / 2;

        beforeOrAt = self[i % cardinality];

        // we've landed on an uninitialized tick, keep searching higher (more recently)
        if (!beforeOrAt.initialized) {
            l = i + 1;
            continue;
        }

        atOrAfter = self[(i + 1) % cardinality];
```

If the calculated midpoint is uninitialized (i.e., the array space is not full), then use the right half of the interval (towards the more recent direction) to continue the binary search.

`atOrAfter` is the observation point immediately to the right.

```solidity
        bool targetAtOrAfter = lte(time, beforeOrAt.blockTimestamp, target);

        // check if we've found the answer!
        if (targetAtOrAfter && lte(time, target, atOrAfter.blockTimestamp)) break;

        if (!targetAtOrAfter) r = i - 1;
        else l = i + 1;
    }
```

* If the target time `target` is between `beforeOrAt` and `atOrAfter` times, exit the binary search and return the two observation points `beforeOrAt` and `atOrAfter`
* Otherwise:
  * If the time of `beforeOrAt` is greater than the target time `target`, then continue the binary search on the left half (towards the smaller time)
  * If the target time `target` is greater than the time of `atOrAfter`, then continue the search on the right half (towards the larger time)

Assuming `cardinality` is 10, `index` is 5, we can draw the logical relationship of several variables as follows:

![](./assets/binarySearch.jpg)

#### getSurroundingObservations

Obtains the observation point data `beforeOrAt` and `atOrAfter` for the target time `target`, satisfying that `target` is between `[beforeOrAt, atOrAfter]`.

```solidity
/// @notice Fetches the observations beforeOrAt and atOrAfter a given target, i.e. where [beforeOrAt, atOrAfter] is satisfied
/// @dev Assumes there is at least 1 initialized observation.
/// Used by observeSingle() to compute the counterfactual accumulator values as of a given block timestamp.
/// @param self The stored oracle array
/// @param time The current block.timestamp
/// @param target The timestamp at which the reserved observation should be for
/// @param tick The active tick at the time of the returned or simulated observation
/// @param index The index of the observation that was most recently written to the observations array
/// @param liquidity The total pool liquidity at the time of the call
/// @param cardinality The number of populated elements in the oracle array
/// @return beforeOrAt The observation which occurred at, or before, the given timestamp
/// @return atOrAfter The observation which occurred at, or after, the given timestamp
function getSurroundingObservations(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) private view returns (Observation memory beforeOrAt, Observation memory atOrAfter) {
    // optimistically set before to the newest observation
    beforeOrAt = self[index];

    // if the target is chronologically at or after the newest observation, we can early return
    if (lte(time, beforeOrAt.blockTimestamp, target)) {
        if (beforeOrAt.blockTimestamp == target) {
            // if newest observation equals target, we're in the same block, so we can ignore atOrAfter
            return (beforeOrAt, atOrAfter);
        } else {
            // otherwise, we need to transform
            return (beforeOrAt, transform(beforeOrAt, target, tick, liquidity));
        }
    }
```

Firstly, `beforeOrAt` is set to the most recent observation point:

If `beforeOrAt` time is less than or equal to the target time `target`:
  * If equal to the target time, then directly return `beforeOrAt`, ignoring `atOrAfter`
  * If less than the target time, then use [transform](#transform) to temporarily generate an observation point as `atOrAfter` based on `beforeOrAt` and `target`

```solidity
    // now, set before to the oldest observation
    beforeOrAt = self[(index + 1) % cardinality];
    if (!beforeOrAt.initialized) beforeOrAt = self[0];
```

Otherwise, it can be confirmed that the last (latest) observation point time is greater than `target`.

Set `beforeOrAt` to the observation point one position to the right in the cycle, i.e., the oldest observation point; if this observation point is not initialized, it means the array has not been filled, so the earliest must be the 0th index observation point.

```solidity
    // ensure that the target is chronologically at or after the oldest observation
    require(lte(time, beforeOrAt.blockTimestamp, target), 'OLD');
```

Confirm that `target` time is greater than or equal to the earliest observation point time, thus `target` is definitely within the time interval of the earliest and latest observation points, binary search can be used.

```solidity
    // if we've reached this point, we have to binary search
    return binarySearch(self, time, target, index, cardinality);
```

#### observeSingle

Obtains the observation point data for a specified time.

Parameters of the method are as follows:

* `self`: The oracle observation point array
* `time`: The current block time
* `secondsAgo`: The specified time seconds ago from the current time, for which to find the observation point
* `tick`: The `tick` (price) at the target time
* `index`: The index of the last written observation point
* `liquidity`: The current global available liquidity
* `cardinality`: The current length of the oracle array (writable space)

Returns:

* `tickCumulative`: The cumulative `tick` up to `secondsAgo` since the creation of the pool pair
* `secondsPerLiquidityCumulativeX128`: The cumulative duration per liquidity up to `secondsAgo` since the creation of the pool pair

```solidity
/// @dev Reverts if an observation at or before the desired observation timestamp does not exist.
/// 0 may be passed as `secondsAgo' to return the current cumulative values.
/// If called with a timestamp falling between two observations, returns the counterfactual accumulator values
/// at exactly the timestamp between the two observations.
/// @param self The stored oracle array
/// @param time The current block timestamp
/// @param secondsAgo The amount of time to look back, in seconds, at which point to return an observation
/// @param tick The current tick
/// @param index The index of the observation that was most recently written to the observations array
/// @param liquidity The current in-range pool liquidity
/// @param cardinality The number of populated elements in the oracle array
/// @return tickCumulative The tick * time elapsed since the pool was first initialized, as of `secondsAgo`
/// @return secondsPerLiquidityCumulativeX128 The time elapsed / max(1, liquidity) since the pool was first initialized, as of `secondsAgo`
function observeSingle(
    Observation[65535] storage self,
    uint32 time,
    uint32 secondsAgo,
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) internal view returns (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) {
    if (secondsAgo == 0) {
        Observation memory last = self[index];
        if (last.blockTimestamp != time) last = transform(last, time, tick, liquidity);
        return (last.tickCumulative, last.secondsPerLiquidityCumulativeX128);
    }
```

If `secondsAgo == 0`, it means taking the observation point at the current time. If the current time is not equal to the last written observation point time, use [transform](#transform) to generate a temporary observation point (note, this observation point is not written) and return the related data.

```solidity
    uint32 target = time - secondsAgo;

    (Observation memory beforeOrAt, Observation memory atOrAfter) =
        getSurroundingObservations(self, time, target, tick, index, liquidity, cardinality);
```

Calculate the target time `target` based on `secondsAgo`, use [getSurroundingObservations](#getSurroundingObservations) method to find the nearest observation point boundaries `beforeOrAt` and `atOrAfter` to the target time.

```solidity
    if (target == beforeOrAt.blockTimestamp) {
        // we're at the left boundary
        return (beforeOrAt.tickCumulative, beforeOrAt.secondsPerLiquidityCumulativeX128);
    } else if (target == atOrAfter.blockTimestamp) {
        // we're at the right boundary
        return (atOrAfter.tickCumulative, atOrAfter.secondsPerLiquidityCumulativeX128);
    } else {
        // we're in the middle
        uint32 observationTimeDelta = atOrAfter.blockTimestamp - beforeOrAt.blockTimestamp;
        uint32 targetDelta = target - beforeOrAt.blockTimestamp;
        return (
            beforeOrAt.tickCumulative +
                ((atOrAfter.tickCumulative - beforeOrAt.tickCumulative) / observationTimeDelta) *
                targetDelta,
            beforeOrAt.secondsPerLiquidityCumulativeX128 +
                uint160(
                    (uint256(
                        atOrAfter.secondsPerLiquidityCumulativeX128 - beforeOrAt.secondsPerLiquidityCumulativeX128
                    ) * targetDelta) / observationTimeDelta
                )
        );
    }
```

According to [getSurroundingObservations](#getSurroundingObservations) method, it is preferred to use the left boundary point `beforeOrAt`:

* If the target time equals `beforeOrAt` time, then directly return the related data of that observation point
* If the target time equals `atOrAfter` time, then also return the related data
* If the target time is between `beforeOrAt` and `atOrAfter`, it is necessary to calculate related values based on the time proportion:
  * `observationTimeDelta` is the time difference between `beforeOrAt` and `atOrAfter` ( $\Delta{t}$ below), `targetDelta` is the time difference between `beforeOrAt` and `target`
  * Because $\Delta{tickCumulative} = tick \cdot \Delta{t}$, the value up to `target` should be: $\frac{\Delta{tickCumulative}}{\Delta{t}} \cdot targetDelta$
  * Similarly, $\Delta{secondsPerLiquidityCumulativeX128} = \frac{\Delta{t}}{liquidity}$, the value up to `target` should be: $\frac{\Delta{secondsPerLiquidityCumulativeX128}}{\Delta{t}} \cdot targetDelta$

#### observe

Batch obtains the observation point data for specified times.

This method mainly calls [observeSingle](#observeSingle) to get the observation point data for a single specified time, and then returns in batch.

```solidity
/// @notice Returns the accumulator values as of each time seconds ago from the given time in the array of `secondsAgos`
/// @dev Reverts if `secondsAgos` > oldest observation
/// @param self The stored oracle array
/// @param time The current block.timestamp
/// @param secondsAgos Each amount of time to look back, in seconds, at which point to return an observation
/// @param tick The current tick
/// @param index The index of the observation that was most recently written to the observations array
/// @param liquidity The current in-range pool liquidity
/// @param cardinality The number of populated elements in the oracle array
/// @return tickCumulatives The tick * time elapsed since the pool was first initialized, as of each `secondsAgo`
/// @return secondsPerLiquidityCumulativeX128s The cumulative seconds / max(1, liquidity) since the pool was first initialized, as of each `secondsAgo`
function observe(
    Observation[65535] storage self,
    uint32 time,
    uint32[] memory secondsAgos,
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) internal view returns (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s) {
    require(cardinality > 0, 'I');

    tickCumulatives = new int56[](secondsAgos.length);
    secondsPerLiquidityCumulativeX128s = new uint160[](secondsAgos.length);
    for (uint256 i = 0; i < secondsAgos.length; i++) {
        (tickCumulatives[i], secondsPerLiquidityCumulativeX128s[i]) = observeSingle(
            self,
            time,
            secondsAgos[i],
            tick,
            index,
            liquidity,
            cardinality
        );
    }
```
