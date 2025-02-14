# Pool Library

The Pool Library defines operations for a single pool, including modifying liquidity, trading, etc. Most of the calculation logic involving `liquidity`, `tick`, `fee`, etc., is the same as Uniswap v3. This is because Uniswap v4 does not adjust the core AMM logic but mainly optimizes the architecture in terms of Hooks management, singleton architecture, flash accounting, etc.

## Struct Definitions

### TickInfo

The `TickInfo` struct defines the information for each `tick` in the pool, including `liquidityGross`, `liquidityNet`, `feeGrowthOutside0X128`, `feeGrowthOutside1X128`, etc.

```solidity
// info stored for each initialized individual tick
struct TickInfo {
    // the total position liquidity that references this tick
    uint128 liquidityGross;
    // amount of net liquidity added (subtracted) when tick is crossed from left to right (right to left),
    int128 liquidityNet;
    // fee growth per unit of liquidity on the _other_ side of this tick (relative to the current tick)
    // only has relative meaning, not absolute — the value depends on when the tick is initialized
    uint256 feeGrowthOutside0X128;
    uint256 feeGrowthOutside1X128;
}
```

As introduced in Uniswap v3:

* `liquidityGross` represents the total liquidity and is used to determine whether the `tick` needs to be initialized:
    * If `mint`, increase liquidity; if `burn`, decrease liquidity
    * This variable is independent of whether the `tick` is used as the lower or upper boundary in different positions, only related to `mint` or `burn` operations
    * If a `tick` is used as both `tickLower` and `tickUpper` by different positions, its `liquidityNet` may be 0, but `liquidityGross` will still be greater than 0, so it does not need to be initialized again

* `liquidityNet` represents the net liquidity and is used to update the global available liquidity when `swap` crosses the `tick`:
    * If used as `tickLower`, i.e., the lower boundary (left boundary point), increase `liquidityDelta`
    * If used as `tickUpper`, i.e., the upper boundary (right boundary point), decrease `liquidityDelta`

### State

The `State` struct defines the state of the pool:

```solidity
/// @notice The state of a pool
/// @dev Note that feeGrowthGlobal can be artificially inflated
/// For pools with a single liquidity position, actors can donate to themselves to freely inflate feeGrowthGlobal
/// atomically donating and collecting fees in the same unlockCallback may make the inflated value more extreme
struct State {
    Slot0 slot0;
    uint256 feeGrowthGlobal0X128;
    uint256 feeGrowthGlobal1X128;
    uint128 liquidity;
    mapping(int24 tick => TickInfo) ticks;
    mapping(int16 wordPos => uint256) tickBitmap;
    mapping(bytes32 positionKey => Position.State) positions;
}
```

Among them:

* `slot0` defines basic information such as the pool fee (LP fee, protocol fee), current price, etc.
* `feeGrowthGlobal0X128` and `feeGrowthGlobal1X128` represent the global fee growth for `token0` and `token1`, respectively.
* `liquidity` represents the total liquidity of the current pool.
* `ticks` is a `mapping` that stores information for each `tick`.
* `tickBitmap` is a `mapping` that stores the `bitmap` information for each `tick`, used for quickly querying the next initialized `tick`.
* `positions` is a `mapping` that stores information for each position.

#### Slot0

In Uniswap v3, `Slot0` is a `struct`; in Uniswap v4, to save gas costs, `Slot0` is a `bytes32` used to store packed `slot0` information.

The layout of `Slot0` is defined as follows:

```
24 bits empty | 24 bits lpFee | 12 bits protocolFee 1->0 | 12 bits protocolFee 0->1 | 24 bits tick | 160 bits sqrtPriceX96
```

From left to right (from high bits to low bits) are:
* 24 bits empty: empty space, used for padding, currently unused
* 24 bits lpFee: liquidity provider fee
* 12 bits protocolFee 1->0: protocol fee from `token1` to `token0`
* 12 bits protocolFee 0->1: protocol fee from `token0` to `token1`
* 24 bits tick: `tick` corresponding to the current price
* 160 bits sqrtPriceX96: `sqrtPriceX96` of the current price

`Slot0Library` defines methods for reading and setting each field from `slot0` (from specified bits).

### ModifyLiquidityParams

The definition of `ModifyLiquidityParams` is as follows:

```solidity
struct ModifyLiquidityParams {
    // the address that owns the position
    address owner;
    // the lower and upper tick of the position
    int24 tickLower;
    int24 tickUpper;
    // any change in liquidity
    int128 liquidityDelta;
    // the spacing between ticks
    int24 tickSpacing;
    // used to distinguish positions of the same owner, at the same tick range
    bytes32 salt;
}
```

Among them:

* `owner`: the owner of the position
* `tickLower` and `tickUpper`: the `tick` range of the position
* `liquidityDelta`: the change in liquidity, positive for adding liquidity, negative for removing liquidity
* `tickSpacing`: the spacing between ticks
* `salt`: used to distinguish different positions

### SwapParams

The definition of `SwapParams` is as follows:

```solidity
struct SwapParams {
    int256 amountSpecified;
    int24 tickSpacing;
    bool zeroForOne;
    uint160 sqrtPriceLimitX96;
    uint24 lpFeeOverride;
}
```

Among them:

* `amountSpecified`: the specified token amount.
  > Note: Unlike Uniswap v3, in Uniswap v4, if `amountSpecified` is negative, it means the desired input token amount, i.e., `exactInput`; if positive, it means the desired output token amount, i.e., `exactOutput`
* `tickSpacing`: the spacing between ticks
* `zeroForOne`: the swap direction, `true` for `token0` to `token1`, `false` for `token1` to `token0`
* `sqrtPriceLimitX96`: the swap price limit, which can be the highest or lowest price
* `lpFeeOverride`: LP fee, used to override the pool's default LP fee

### SwapResult

The definition of `SwapResult` is as follows:

```solidity
// Tracks the state of a pool throughout a swap, and returns these values at the end of the swap
struct SwapResult {
    // the current sqrt(price)
    uint160 sqrtPriceX96;
    // the tick associated with the current price
    int24 tick;
    // the current liquidity in range
    uint128 liquidity;
}
```

Among them:

* `sqrtPriceX96`: the current price
* `tick`: the `tick` corresponding to the current price
* `liquidity`: the current liquidity in range

### StepComputations

The definition of `StepComputations` is as follows:

```solidity
struct StepComputations {
    // the price at the beginning of the step
    uint160 sqrtPriceStartX96;
    // the next tick to swap to from the current tick in the swap direction
    int24 tickNext;
    // whether tickNext is initialized or not
    bool initialized;
    // sqrt(price) for the next tick (1/0)
    uint160 sqrtPriceNextX96;
    // how much is being swapped in in this step
    uint256 amountIn;
    // how much is being swapped out
    uint256 amountOut;
    // how much fee is being paid in
    uint256 feeAmount;
    // the global fee growth of the input token. updated in storage at the end of swap
    uint256 feeGrowthGlobalX128;
}
```

Among them:

* `sqrtPriceStartX96`: the price at the beginning of the step
* `tickNext`: the next tick to swap to from the current tick in the swap direction
* `initialized`: whether `tickNext` is initialized or not
* `sqrtPriceNextX96`: the price for the next tick
* `amountIn`: the amount being swapped in in this step
* `amountOut`: the amount being swapped out
* `feeAmount`: the fee amount being paid in
* `feeGrowthGlobalX128`: the global fee growth of the input token, updated in storage at the end of the swap

## Function Definitions

### checkTicks

Check whether `tickLower` and `tickUpper` are valid, i.e., `tickLower` is less than `tickUpper`, and `tickLower` and `tickUpper` are within the valid range.

```solidity
/// @dev Common checks for valid tick inputs.
function checkTicks(int24 tickLower, int24 tickUpper) private pure {
    if (tickLower >= tickUpper) TicksMisordered.selector.revertWith(tickLower, tickUpper);
    if (tickLower < TickMath.MIN_TICK) TickLowerOutOfBounds.selector.revertWith(tickLower);
    if (tickUpper > TickMath.MAX_TICK) TickUpperOutOfBounds.selector.revertWith(tickUpper);
}
```

As for how `TickMath.MIN_TICK` and `TickMath.MAX_TICK` are calculated, we introduced it in Uniswap v3:

Since the price range supported by Uniswap v3 ( $\frac{token1}{token0}$ ) is $[2^{-128}, 2^{128}]$, according to formula 6.1 in the Uniswap v3 whitepaper:

$$
p(i) = 1.0001^i
$$

Therefore, the maximum `tick` (i.e., `MAX_TICK`) is:

$$
i = \lfloor log_{1.0001}{p(i)} \rfloor = \lfloor log_{1.0001}{2^{128}} \rfloor = \lfloor 887272.7517970635 \rfloor = 887272
$$

The minimum `tick` (i.e., `MIN_TICK`) is:

$$
i = \lceil log_{1.0001}{2^{-128}} \rceil = \lceil -887272.7517970635 \rceil = -887272
$$

### initialize

Initialize the pool, setting the pool's `sqrtPriceX96` and `lpFee` fields.

```solidity
function initialize(State storage self, uint160 sqrtPriceX96, uint24 lpFee) internal returns (int24 tick) {
    if (self.slot0.sqrtPriceX96() != 0) PoolAlreadyInitialized.selector.revertWith();

    tick = TickMath.getTickAtSqrtPrice(sqrtPriceX96);

    // the initial protocolFee is 0 so doesn't need to be set
    self.slot0 = Slot0.wrap(bytes32(0)).setSqrtPriceX96(sqrtPriceX96).setTick(tick).setLpFee(lpFee);
}
```

The current price `sqrtPriceX96` of an uninitialized pool is 0, so it can be determined whether the pool has been initialized by checking whether `sqrtPriceX96` is 0.

Calculate the `tick` corresponding to the current price based on `sqrtPriceX96`. The specific algorithm can be referred to in Uniswap v3's [TickMath.getTickAtSqrtPrice](../../../dive-into-uniswap-v3-contracts/README.md#getTickAtSqrtRatio) method.

The `initialize` method sets the `slot0` fields `sqrtPriceX96`, `tick`, and `lpFee`.

### setProtocolFee

Set the protocol fee.

```solidity
function setProtocolFee(State storage self, uint24 protocolFee) internal {
    self.checkPoolInitialized();
    self.slot0 = self.slot0.setProtocolFee(protocolFee);
}
```

Ensure the pool is initialized. As mentioned earlier, `checkPoolInitialized` determines whether the pool is initialized by checking whether the `slot.sqrtPriceX96` of the current pool state is 0.

### setLPFee

Set the LP fee. Only pools with dynamic fees can set the LP fee.

```solidity
/// @notice Only dynamic fee pools may update the lp fee.
function setLPFee(State storage self, uint24 lpFee) internal {
    self.checkPoolInitialized();
    self.slot0 = self.slot0.setLpFee(lpFee);
}
```

### modifyLiquidity

Modify the liquidity of a user's position, such as adding or removing liquidity.

```solidity
/// @notice Effect changes to a position in a pool
/// @dev PoolManager checks that the pool is initialized before calling
/// @param params the position details and the change to the position's liquidity to effect
/// @return delta the deltas of the token balances of the pool, from the liquidity change
/// @return feeDelta the fees generated by the liquidity range
function modifyLiquidity(State storage self, ModifyLiquidityParams memory params)
    internal
    returns (BalanceDelta delta, BalanceDelta feeDelta)
```

The input parameters of this method are:

* [State](#state) self: the state of the pool
* [ModifyLiquidityParams](#modifyliquidityparams) params: parameters for modifying liquidity, representing the user's position information and liquidity changes

The return values are:

* `BalanceDelta` delta: the token balance changes after the liquidity change
* `BalanceDelta` feeDelta: the fee changes after the liquidity change, which is non-negative.

[BalanceDelta](./BalanceDelta.md) is an `int256` type, with the first 128 bits representing the balance change of `token0` and the last 128 bits representing the balance change of `token1`. If positive, it means the user can withdraw `token0` or `token1`; if negative, it means the user needs to deposit `token0` or `token1`.

```solidity
{
    int128 liquidityDelta = params.liquidityDelta;
    int24 tickLower = params.tickLower;
    int24 tickUpper = params.tickUpper;
    checkTicks(tickLower, tickUpper);
```

Check the validity of `tickLower` and `tickUpper` through [checkTicks](#checkticks).

```solidity
    {
        ModifyLiquidityState memory state;

        // if we need to update the ticks, do it
        if (liquidityDelta != 0) {
            (state.flippedLower, state.liquidityGrossAfterLower) =
                updateTick(self, tickLower, liquidityDelta, false);
            (state.flippedUpper, state.liquidityGrossAfterUpper) = updateTick(self, tickUpper, liquidityDelta, true);

            // `>` and `>=` are logically equivalent here but `>=` is cheaper
            if (liquidityDelta >= 0) {
                uint128 maxLiquidityPerTick = tickSpacingToMaxLiquidityPerTick(params.tickSpacing);
                if (state.liquidityGrossAfterLower > maxLiquidityPerTick) {
                    TickLiquidityOverflow.selector.revertWith(tickLower);
                }
                if (state.liquidityGrossAfterUpper > maxLiquidityPerTick) {
                    TickLiquidityOverflow.selector.revertWith(tickUpper);
                }
            }

            if (state.flippedLower) {
                self.tickBitmap.flipTick(tickLower, params.tickSpacing);
            }
            if (state.flippedUpper) {
                self.tickBitmap.flipTick(tickUpper, params.tickSpacing);
            }
        }

        {
            (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
                getFeeGrowthInside(self, tickLower, tickUpper);

            Position.State storage position = self.positions.get(params.owner, tickLower, tickUpper, params.salt);
            (uint256 feesOwed0, uint256 feesOwed1) =
                position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

            // Fees earned from LPing are calculated, and returned
            feeDelta = toBalanceDelta(feesOwed0.toInt128(), feesOwed1.toInt128());
        }

        // clear any tick data that is no longer needed
        if (liquidityDelta < 0) {
            if (state.flippedLower) {
                clearTick(self, tickLower);
            }
            if (state.flippedUpper) {
                clearTick(self, tickUpper);
            }
        }
    }
```

Update the liquidity of `tickLower` and `tickUpper` through the [updateTick](#updatetick) method.

If `liquidityDelta >= 0`, it means adding liquidity, and it is necessary to check whether the total liquidity of `tickLower` and `tickUpper` exceeds the maximum liquidity supported by a single `tick` corresponding to `tickSpacing`.

If `tickLower` or `tickUpper` flips, update the bit information of the corresponding `tick` in `tickBitmap`, from `0` to `1` or from `1` to `0`.

Get the fee growth between `tickLower` and `tickUpper` through the [getFeeGrowthInside](#getfeegrowthinside) method.

Get the user's position through the [positions.get](./PositionLibrary.md#get) method. The position `position` is uniquely determined by `owner`, `tickLower`, `tickUpper`, and `salt`.
Update the liquidity of the position through the [position.update](./PositionLibrary.md#update) method and calculate the corresponding LP fee.

If `liquidityDelta < 0`, it means reducing liquidity. At the same time, if the corresponding `tick` flips, it means that the `tick` is not associated with any position, and the `tick` data of the pool can be deleted.

```solidity
    if (liquidityDelta != 0) {
        Slot0 _slot0 = self.slot0;
        (int24 tick, uint160 sqrtPriceX96) = (_slot0.tick(), _slot0.sqrtPriceX96());
        if (tick < tickLower) {
            // current tick is below the passed range; liquidity can only become in range by crossing from left to
            // right, when we'll need _more_ currency0 (it's becoming more valuable) so user must provide it
            delta = toBalanceDelta(
                SqrtPriceMath.getAmount0Delta(
                    TickMath.getSqrtPriceAtTick(tickLower), TickMath.getSqrtPriceAtTick(tickUpper), liquidityDelta
                ).toInt128(),
                0
            );
        } else if (tick < tickUpper) {
            delta = toBalanceDelta(
                SqrtPriceMath.getAmount0Delta(sqrtPriceX96, TickMath.getSqrtPriceAtTick(tickUpper), liquidityDelta)
                    .toInt128(),
                SqrtPriceMath.getAmount1Delta(TickMath.getSqrtPriceAtTick(tickLower), sqrtPriceX96, liquidityDelta)
                    .toInt128()
            );

            self.liquidity = LiquidityMath.addDelta(self.liquidity, liquidityDelta);
        } else {
            // current tick is above the passed range; liquidity can only become in range by crossing from right to
            // left, when we'll need _more_ currency1 (it's becoming more valuable) so user must provide it
            delta = toBalanceDelta(
                0,
                SqrtPriceMath.getAmount1Delta(
                    TickMath.getSqrtPriceAtTick(tickLower), TickMath.getSqrtPriceAtTick(tickUpper), liquidityDelta
                ).toInt128()
            );
        }
    }
}
```

If `liquidityDelta != 0`, i.e., liquidity changes, it is necessary to calculate `delta`, which represents the amount of `token0` and `token1` that the user needs to provide or can withdraw.

If the current `tick` is less than `tickLower`, since the size of `tick` is proportional to $\sqrt{P}$ (i.e., $\sqrt{\frac{y}{x}}$), it means that in the range greater than the current `tick`, the value of $x$ is higher (less $x$ is needed), so when adding liquidity, $x$ tokens need to be provided in this part, i.e., `token0` needs to be calculated; conversely, $y$ tokens need to be provided, i.e., `token1` needs to be calculated.

Refer to Uniswap v3 to understand how to calculate [getAmount0Delta](../../../dive-into-uniswap-v3-contracts/README.md#getAmount0Delta) and [getAmount1Delta](../../../dive-into-uniswap-v3-contracts/README.md#getAmount1Delta).

### swap

Execute the swap operation, completing the exchange from `token0` to `token1` or vice versa.

The entire `swap` code logic is almost identical to Uniswap v3's [swap](../../../dive-into-uniswap-v3-contracts/README.md#swap).

The following is the implementation of the `swap` method in Uniswap v4:

```solidity
/// @notice Executes a swap against the state, and returns the amount deltas of the pool
/// @dev PoolManager checks that the pool is initialized before calling
function swap(State storage self, SwapParams memory params)
    internal
    returns (BalanceDelta swapDelta, uint256 amountToProtocol, uint24 swapFee, SwapResult memory result)
{
```

The input parameters of this method are:

* [State](#state) self: the state of the pool
* [SwapParams](#swapparams) params: swap parameters, including swap amount, swap direction, price limit, etc.

The return values are:

* `BalanceDelta` swapDelta: the token balance changes after the swap
* `uint256` amountToProtocol: protocol fee
* `uint24` swapFee: swap fee
* [SwapResult](#swapresult) result: swap result, including the price, `tick`, and liquidity after the swap

The code is as follows:

```solidity
    Slot0 slot0Start = self.slot0;
    bool zeroForOne = params.zeroForOne;

    uint256 protocolFee =
        zeroForOne ? slot0Start.protocolFee().getZeroForOneFee() : slot0Start.protocolFee().getOneForZeroFee();

    // the amount remaining to be swapped in/out of the input/output asset. initially set to the amountSpecified
    int256 amountSpecifiedRemaining = params.amountSpecified;
    // the amount swapped out/in of the output/input asset. initially set to 0
    int256 amountCalculated = 0;
    // initialize to the current sqrt(price)
    result.sqrtPriceX96 = slot0Start.sqrtPriceX96();
    // initialize to the current tick
    result.tick = slot0Start.tick();
    // initialize to the current liquidity
    result.liquidity = self.liquidity;
```

Initialize related parameters.

```solidity
    // if the beforeSwap hook returned a valid fee override, use that as the LP fee, otherwise load from storage
    // lpFee, swapFee, and protocolFee are all in pips
    {
        uint24 lpFee = params.lpFeeOverride.isOverride()
            ? params.lpFeeOverride.removeOverrideFlagAndValidate()
            : slot0Start.lpFee();

        swapFee = protocolFee == 0 ? lpFee : uint16(protocolFee).calculateSwapFee(lpFee);
    }
```

If the hook returns a new `lpFee`, use the new `lpFee`; otherwise, use the `lpFee` configured when the pool was created.

If the protocol fee is 0, use `lpFee` as the swap fee `swapFee`; otherwise, calculate the swap fee. Since the unit of `fee` is one-hundredth of a BIP, i.e., ${10}^{6}$, the calculation method of the swap fee is as follows:

$$
swapFee = protocolFee + lpFee \cdot (1 - \frac{protocolFee}{{10}^{6}})
$$

`swapFee` includes `protocolFee` and `lpFee`, where `lpFee` is calculated after deducting the `protocolFee`.

```solidity
    // a swap fee totaling MAX_SWAP_FEE (100%) makes exact output swaps impossible since the input is entirely consumed by the fee
    if (swapFee >= SwapMath.MAX_SWAP_FEE) {
        // if exactOutput
        if (params.amountSpecified > 0) {
            InvalidFeeForExactOut.selector.revertWith();
        }
    }

    // swapFee is the pool's fee in pips (LP fee + protocol fee)
    // when the amount swapped is 0, there is no protocolFee applied and the fee amount paid to the protocol is set to 0
    if (params.amountSpecified == 0) return (BalanceDeltaLibrary.ZERO_DELTA, 0, swapFee, result);
```

Check the validity of `swapFee`.
If the input token amount is 0, exit.

```solidity
    if (zeroForOne) {
        if (params.sqrtPriceLimitX96 >= slot0Start.sqrtPriceX96()) {
            PriceLimitAlreadyExceeded.selector.revertWith(slot0Start.sqrtPriceX96(), params.sqrtPriceLimitX96);
        }
        // Swaps can never occur at MIN_TICK, only at MIN_TICK + 1, except at initialization of a pool
        // Under certain circumstances outlined below, the tick will preemptively reach MIN_TICK without swapping there
        if (params.sqrtPriceLimitX96 <= TickMath.MIN_SQRT_PRICE) {
            PriceLimitOutOfBounds.selector.revertWith(params.sqrtPriceLimitX96);
        }
    } else {
        if (params.sqrtPriceLimitX96 <= slot0Start.sqrtPriceX96()) {
            PriceLimitAlreadyExceeded.selector.revertWith(slot0Start.sqrtPriceX96(), params.sqrtPriceLimitX96);
        }
        if (params.sqrtPriceLimitX96 >= TickMath.MAX_SQRT_PRICE) {
            PriceLimitOutOfBounds.selector.revertWith(params.sqrtPriceLimitX96);
        }
    }
```

Check the validity of `sqrtPriceLimitX96`:

* If `zeroForOne` is `true`, i.e., a swap from `token0` to `token1`, after the swap, `token0` ( $x$ ) increases, `token1` ( $y$ ) decreases, i.e., $ \sqrt{P} = \sqrt{\frac{y}{x}} $ decreases, so the target price `sqrtPriceLimitX96` should be less than the current price but not less than `MIN_SQRT_PRICE`.
* Conversely, the target price `sqrtPriceLimitX96` should be greater than the current price but not greater than `MAX_SQRT_PRICE`.

As introduced in the [Uniswap v3 whitepaper](../../../dive-into-uniswap-v3-whitepaper/README.md#621-price-and-liquidity-价格和流动性), the core logic of the `swap` process is:

* At any time, only one of the liquidity $L$ and the price $\sqrt{P}$ will change.
* During a `swap`:
  * All liquidity in the current price range constitutes the total liquidity `liquidity`
  * Each `swap` is divided into multiple `steps`, and each `step` only performs the `swap` operation within one `tick` range, so:
    * During the `swap`, $L$ remains unchanged, and $\sqrt{P}$ changes;
    * When the price crosses the `cross tick`, modify the total liquidity $L$, keep $\sqrt{P}$ unchanged, and then continue to the next `swap step`;
* During add/burn liquidity, $L$ changes, and $\sqrt{P}$ remains unchanged.

The following `swap` code implements the above logic.

```solidity
    StepComputations memory step;
    step.feeGrowthGlobalX128 = zeroForOne ? self.feeGrowthGlobal0X128 : self.feeGrowthGlobal1X128;
    // continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
    while (!(amountSpecifiedRemaining == 0 || result.sqrtPriceX96 == params.sqrtPriceLimitX96)) {
        step.sqrtPriceStartX96 = result.sqrtPriceX96;

        (step.tickNext, step.initialized) =
            self.tickBitmap.nextInitializedTickWithinOneWord(result.tick, params.tickSpacing, zeroForOne);

        // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
        if (step.tickNext <= TickMath.MIN_TICK) {
            step.tickNext = TickMath.MIN_TICK;
        }
        if (step.tickNext >= TickMath.MAX_TICK) {
            step.tickNext = TickMath.MAX_TICK;
        }

        // get the price for the next tick
        step.sqrtPriceNextX96 = TickMath.getSqrtPriceAtTick(step.tickNext);

        // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
        (result.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
            result.sqrtPriceX96,
            SwapMath.getSqrtPriceTarget(zeroForOne, step.sqrtPriceNextX96, params.sqrtPriceLimitX96),
            result.liquidity,
            amountSpecifiedRemaining,
            swapFee
        );

        // if exactOutput
        if (params.amountSpecified > 0) {
            unchecked {
                amountSpecifiedRemaining -= step.amountOut.toInt256();
            }
            amountCalculated -= (step.amountIn + step.feeAmount).toInt256();
        } else {
            // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
            unchecked {
                amountSpecifiedRemaining += (step.amountIn + step.feeAmount).toInt256();
            }
            amountCalculated += step.amountOut.toInt256();
        }

        // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
        if (protocolFee > 0) {
            unchecked {
                // step.amountIn does not include the swap fee, as it's already been taken from it,
                // so add it back to get the total amountIn and use that to calculate the amount of fees owed to the protocol
                // cannot overflow due to limits on the size of protocolFee and params.amountSpecified
                // this rounds down to favor LPs over the protocol
                uint256 delta = (swapFee == protocolFee)
                    ? step.feeAmount // lp fee is 0, so the entire fee is owed to the protocol instead
                    : (step.amountIn + step.feeAmount) * protocolFee / ProtocolFeeLibrary.PIPS_DENOMINATOR;
                // subtract it from the total fee and add it to the protocol fee
                step.feeAmount -= delta;
                amountToProtocol += delta;
            }
        }

        // update global fee tracker
        if (result.liquidity > 0) {
            unchecked {
                // FullMath.mulDiv isn't needed as the numerator can't overflow uint256 since tokens have a max supply of type(uint128).max
                step.feeGrowthGlobalX128 +=
                    UnsafeMath.simpleMulDiv(step.feeAmount, FixedPoint128.Q128, result.liquidity);
            }
        }

        // Shift tick if we reached the next price, and preemptively decrement for zeroForOne swaps to tickNext - 1.
        // If the swap doesn't continue (if amountRemaining == 0 or sqrtPriceLimit is met), slot0.tick will be 1 less
        // than getTickAtSqrtPrice(slot0.sqrtPrice). This doesn't affect swaps, but donation calls should verify both
        // price and tick to reward the correct LPs.
        if (result.sqrtPriceX96 == step.sqrtPriceNextX96) {
            // if the tick is initialized, run the tick transition
            if (step.initialized) {
                (uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128) = zeroForOne
                    ? (step.feeGrowthGlobalX128, self.feeGrowthGlobal1X128)
                    : (self.feeGrowthGlobal0X128, step.feeGrowthGlobalX128);
                int128 liquidityNet =
                    Pool.crossTick(self, step.tickNext, feeGrowthGlobal0X128, feeGrowthGlobal1X128);
                // if we're moving leftward, we interpret liquidityNet as the opposite sign
                // safe because liquidityNet cannot be type(int128).min
                unchecked {
                    if (zeroForOne) liquidityNet = -liquidityNet;
                }

                result.liquidity = LiquidityMath.addDelta(result.liquidity, liquidityNet);
            }

            unchecked {
                result.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
            }
        } else if (result.sqrtPriceX96 != step.sqrtPriceStartX96) {
            // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
            result.tick = TickMath.getTickAtSqrtPrice(result.sqrtPriceX96);
        }
    }
```

Divide the entire `swap` operation into multiple `swap steps`.
In the `while` loop, execute the `swap step` for each `step`:

1. Get at most one initialized `tick` through `tickBitmap.nextInitializedTickWithinOneWord` and calculate the price `sqrtPriceNextX96` corresponding to the `tick`.

1. Calculate a swap within a `tick` through [SwapMath.computeSwapStep](../../../dive-into-uniswap-v3-contracts/README.md#computeSwapStep). In this step, the liquidity $L$ remains unchanged, and only the price $\sqrt{P}$ changes. Calculate the ending price `sqrtPriceX96`, input `amountIn`, output `amountOut`, and fee `feeAmount` for this `swap step`. The returned results are all non-negative.

1. Update `amountSpecifiedRemaining` and `amountCalculated`:
   * If `params.amountSpecified > 0`, i.e., `exactOutput`, `amountSpecifiedRemaining` represents the output token and needs to subtract `amountOut`, and `amountCalculated` represents the input token, using a negative number to indicate the amount of `amountIn` and `feeAmount` that the user needs to deposit;
   * Conversely, i.e., `exactInput`, `amountSpecifiedRemaining` represents the input token, and since it is negative, it increases the amount corresponding to `amountIn` and `feeAmount`, and `amountCalculated` represents the output token, using a positive number to indicate the amount of `amountOut` that the user can withdraw.

1. If `protocolFee > 0`, calculate the protocol fee `amountToProtocol` for this `step` and deduct the protocol fee from `feeAmount`. The calculation method of `protocolFee` is as follows:

    $$
    protocol fee = (amountIn + feeAmount) \cdot \frac{protocolFee}{10^6}
    $$

1. If the current global liquidity `result.liquidity` is greater than 0, use the `feeAmount` collected in this `step` (after deducting the protocol fee) to calculate the global fee growth per unit of liquidity and update the global fee growth `feeGrowthGlobalX128`.

1. If the price `result.sqrtPriceX96` after the `swap step` is equal to the price `step.sqrtPriceNextX96` of the next `tick`, it means that the current `tick` range has been fully completed. Since `nextInitializedTickWithinOneWord` may return uninitialized `ticks`, if the current `tick` is initialized, it means that the `tick` has been completed (crossed), so execute the [crossTick](#crosstick) operation. According to the `liquidityNet` owned by the `tick`, update the global liquidity `result.liquidity`:
   * Since when the `tick` is first initialized, we set the `liquidityNet` of `tickLower` to a positive number and the `liquidityNet` of `tickUpper` to a negative number by default.
   * If `zeroForOne` is `true`, i.e., a swap from `token0` to `token1`, it means that as the `swap` progresses, the number of `token0` increases, and the number of `token1` decreases, so $\sqrt{P} = \sqrt{\frac{y}{x}}$ decreases, and the `tick` moves from right to left:
      * If the `tick` is `tickLower`, i.e., crossing the left boundary, it means leaving the position range, so the global liquidity needs to be reduced;
      * If the `tick` is `tickUpper`, i.e., crossing the right boundary, it means entering the position range, so the global liquidity needs to be increased.
      * Therefore, when `zeroForOne` is `true`, the `liquidityNet` needs to be negated.
   * Similarly, if `zeroForOne` is `false`, i.e., a swap from `token1` to `token0`, the `tick` moves from left to right:
      * If the `tick` is `tickLower`, i.e., crossing the left boundary, it means entering the position range, so the global liquidity needs to be increased;
      * If the `tick` is `tickUpper`, i.e., crossing the right boundary, it means leaving the position range, so the global liquidity needs to be reduced.
      * Therefore, when `zeroForOne` is `false`, the `liquidityNet` does not need to be negated.

1. Update the `tick` to the next `tick`. If `zeroForOne` is `true`, the `tick` decreases, so the next `tick` is `step.tickNext - 1`; otherwise, it is `step.tickNext`.

1. If the price `result.sqrtPriceX96` after the `swap step` does not reach the price `step.sqrtPriceNextX96` of the next `tick`, it means that either `amountRemaining` is 0 or the price has reached `sqrtPriceLimit`. If the current price is not equal to `sqrtPriceStartX96`, update the `tick`.

```solidity
    self.slot0 = slot0Start.setTick(result.tick).setSqrtPriceX96(result.sqrtPriceX96);

    // update liquidity if it changed
    if (self.liquidity != result.liquidity) self.liquidity = result.liquidity;

    // update fee growth global
    if (!zeroForOne) {
        self.feeGrowthGlobal1X128 = step.feeGrowthGlobalX128;
    } else {
        self.feeGrowthGlobal0X128 = step.feeGrowthGlobalX128;
    }
```

At this point, the `swap` is completed, either `amountRemaining` is 0 or the price has reached `sqrtPriceLimit`. Update `slot0`, global liquidity `liquidity`, and global fee growth `feeGrowthGlobalX128`.

```solidity
    unchecked {
        // "if currency1 is specified"
        if (zeroForOne != (params.amountSpecified < 0)) {
            swapDelta = toBalanceDelta(
                amountCalculated.toInt128(), (params.amountSpecified - amountSpecifiedRemaining).toInt128()
            );
        } else {
            swapDelta = toBalanceDelta(
                (params.amountSpecified - amountSpecifiedRemaining).toInt128(), amountCalculated.toInt128()
            );
        }
    }
}
```

`params.amountSpecified < 0` indicates `exactInput`. Combined with `zeroForOne`, the following combinations are possible:

| zeroForOne | exactInput | Description | amountSpecified | amountCalculated |
| --- | --- | --- | --- | --- |
| true | true | Swap from `token0` to `token1`, input token amount is known, output token amount is unknown | Input `token0` token amount, negative | Output `token1` token amount |
| true | false | Swap from `token0` to `token1`, input token amount is unknown, output token amount is known | Output `token1` token amount, positive | Input `token0` token amount |
| false | true | Swap from `token1` to `token0`, input token amount is known, output token amount is unknown | Input `token1` token amount, negative | Output `token0` token amount |
| false | false | Swap from `token1` to `token0`, input token amount is unknown, output token amount is known | Output `token0` token amount, positive | Input `token1` token amount |

Therefore, when `zeroForOne != (params.amountSpecified < 0)`, `amountSpecified` always represents the amount of `token1`, and `params.amountSpecified - amountSpecifiedRemaining` is the actual amount of `token1` required; `amountCalculated` always represents the amount of `token0`.

The final calculated `swapDelta` is the token balance change for this `swap` operation, with the first 128 bits representing the balance change of `token0` and the last 128 bits representing the balance change of `token1`. A negative number indicates the amount of tokens the user needs to deposit, and a positive number indicates the amount of tokens the user can withdraw.

### donate

Donate `token0` and `token1` tokens to the pool.

```solidity
/// @notice Donates the given amount of currency0 and currency1 to the pool
function donate(State storage state, uint256 amount0, uint256 amount1) internal returns (BalanceDelta delta) {
    uint128 liquidity = state.liquidity;
    if (liquidity == 0) NoLiquidityToReceiveFees.selector.revertWith();
    unchecked {
        // negation safe as amount0 and amount1 are always positive
        delta = toBalanceDelta(-(amount0.toInt128()), -(amount1.toInt128()));
        // FullMath.mulDiv is unnecessary because the numerator is bounded by type(int128).max * Q128, which is less than type(uint256).max
        if (amount0 > 0) {
            state.feeGrowthGlobal0X128 += UnsafeMath.simpleMulDiv(amount0, FixedPoint128.Q128, liquidity);
        }
        if (amount1 > 0) {
            state.feeGrowthGlobal1X128 += UnsafeMath.simpleMulDiv(amount1, FixedPoint128.Q128, liquidity);
        }
    }
}
```

`delta` has the first 128 bits representing the balance change of `token0` and the last 128 bits representing the balance change of `token1`. A negative number indicates the amount of tokens the user needs to deposit, and a positive number indicates the amount of tokens the user can withdraw. Since it is a donation of tokens, the user needs to deposit the corresponding amount of tokens.

Divide `amount` by `liquidity` to get the global fee growth per unit of liquidity `feeGrowthGlobal0X128` and `feeGrowthGlobal1X128`, and update the corresponding values.

### getFeeGrowthInside

Calculate the fee growth between `tickLower` and `tickUpper`.

```solidity
/// @notice Retrieves fee growth data
/// @param self The Pool state struct
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @return feeGrowthInside0X128 The all-time fee growth in token0, per unit of liquidity, inside the position's tick boundaries
/// @return feeGrowthInside1X128 The all-time fee growth in token1, per unit of liquidity, inside the position's tick boundaries
function getFeeGrowthInside(State storage self, int24 tickLower, int24 tickUpper)
    internal
    view
    returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128)
{
    TickInfo storage lower = self.ticks[tickLower];
    TickInfo storage upper = self.ticks[tickUpper];
    int24 tickCurrent = self.slot0.tick();

    unchecked {
        if (tickCurrent < tickLower) {
            feeGrowthInside0X128 = lower.feeGrowthOutside0X128 - upper.feeGrowthOutside0X128;
            feeGrowthInside1X128 = lower.feeGrowthOutside1X128 - upper.feeGrowthOutside1X128;
        } else if (tickCurrent >= tickUpper) {
            feeGrowthInside0X128 = upper.feeGrowthOutside0X128 - lower.feeGrowthOutside0X128;
            feeGrowthInside1X128 = upper.feeGrowthOutside1X128 - lower.feeGrowthOutside1X128;
        } else {
            feeGrowthInside0X128 =
                self.feeGrowthGlobal0X128 - lower.feeGrowthOutside0X128 - upper.feeGrowthOutside0X128;
            feeGrowthInside1X128 =
                self.feeGrowthGlobal1X128 - lower.feeGrowthOutside1X128 - upper.feeGrowthOutside1X128;
        }
    }
}
```

As introduced in the Uniswap v3 whitepaper, the calculation method for fees is:

Based on whether the current price is within the range, a formula can be used to calculate the fees earned per unit of liquidity above ( $f_a$ ) and below ( $f_b$ ) the `tick` $i$ (based on whether the current `tick` number $i_c$ is greater than or equal to $i$ ):

$$
f_a(i) = \begin{cases} f_g - f_o(i) & \text{$i_c \geq i$}\\
f_o(i) & \text{$i_c < i$} \end{cases}
$$

$$
f_b(i) = \begin{cases} f_o(i) & \text{$i_c \geq i$}\\
f_g - f_o(i) & \text{$i_c < i$}\end{cases}
$$

Using the above functions, the total fees $f_r$ accumulated per unit of liquidity within any two `ticks` (lower `tick` $i_l$ and upper `tick` $i_u$) can be calculated as follows:

$$
f_r = f_g - f_b(i_l) - f_a(i_u)
$$

Combining the above formula, based on the relationship between the current pool `tick` and the boundary `tickLower` and `tickUpper`, the calculation method for fees within the range is as follows:

$$
f_r = \begin{cases} 
f_o(i_l) - f_o(i_u) & \text{$i_c < i_l < i_u$}\\
f_g - f_o(i_l) - f_o(i_u) & \text{$i_l \leq i_c < i_u$}\\
f_o(i_u) - f_o(i_l) & \text{$i_l < i_u \leq i_c$} \end{cases}
$$

The `getFeeGrowthInside` code implements the above logic.

### updateTick

Update the liquidity of a `tick`.

```solidity
/// @notice Updates a tick and returns true if the tick was flipped from initialized to uninitialized, or vice versa
/// @param self The mapping containing all tick information for initialized ticks
/// @param tick The tick that will be updated
/// @param liquidityDelta A new amount of liquidity to be added (subtracted) when tick is crossed from left to right (right to left)
/// @param upper true for updating a position's upper tick, or false for updating a position's lower tick
/// @return flipped Whether the tick was flipped from initialized to uninitialized, or vice versa
/// @return liquidityGrossAfter The total amount of liquidity for all positions that references the tick after the update
function updateTick(State storage self, int24 tick, int128 liquidityDelta, bool upper)
    internal
    returns (bool flipped, uint128 liquidityGrossAfter)
```

The input parameters of this method are:

* [State](#state) self: the state of the pool
* `tick`: the `tick` to be updated
* `liquidityDelta`: the change in liquidity, positive for adding liquidity, negative for removing liquidity
* `upper`: whether to update the upper `tick` of the position

The return values are:

* `flipped`: whether the `tick` was flipped, i.e., whether it changed from initialized to uninitialized or vice versa
* `liquidityGrossAfter`: the total liquidity of the `tick` after the update

```solidity
{
    TickInfo storage info = self.ticks[tick];

    uint128 liquidityGrossBefore = info.liquidityGross;
    int128 liquidityNetBefore = info.liquidityNet;

    liquidityGrossAfter = LiquidityMath.addDelta(liquidityGrossBefore, liquidityDelta);

    flipped = (liquidityGrossAfter == 0) != (liquidityGrossBefore == 0);

    if (liquidityGrossBefore == 0) {
        // by convention, we assume that all growth before a tick was initialized happened _below_ the tick
        if (tick <= self.slot0.tick()) {
            info.feeGrowthOutside0X128 = self.feeGrowthGlobal0X128;
            info.feeGrowthOutside1X128 = self.feeGrowthGlobal1X128;
        }
    }
```

Get the corresponding `TickInfo` based on the `tick`:

* Calculate `liquidityGross`. Among them, `LiquidityMath.addDelta` performs addition of `uint128` and `int128`, ensuring that the final result will not overflow positively or negatively. Therefore, if `liquidityDelta` is positive, increase the total liquidity; if `liquidityDelta` is negative, decrease the total liquidity. Finally, `tickInfo.liquidityGross` must be greater than or equal to 0.
* Determine whether the `tick` is flipped by checking whether `liquidityGrossAfter` and `liquidityGrossBefore` have exactly one of them equal to 0.
* Refer to formula (6.21) in the Uniswap v3 whitepaper. If the `tick` is less than or equal to the current pool price, initialize `feeGrowthOutside0X128` and `feeGrowthOutside1X128` with the global fee growth `feeGrowthGlobal0X128` and `feeGrowthGlobal1X128`, respectively.
  > $$
  > f_o := \begin{cases} f_g & \text{$i_c \geq i$}\\
  > 0 & \text{$i_c < i$} \end{cases}
  > $$

```solidity
    // when the lower (upper) tick is crossed left to right, liquidity must be added (removed)
    // when the lower (upper) tick is crossed right to left, liquidity must be removed (added)
    int128 liquidityNet = upper ? liquidityNetBefore - liquidityDelta : liquidityNetBefore + liquidityDelta;
    assembly ("memory-safe") {
        // liquidityGrossAfter and liquidityNet are packed in the first slot of `info`
        // So we can store them with a single sstore by packing them ourselves first
        sstore(
            info.slot,
            // bitwise OR to pack liquidityGrossAfter and liquidityNet
            or(
                // Put liquidityGrossAfter in the lower bits, clearing out the upper bits
                and(liquidityGrossAfter, 0xffffffffffffffffffffffffffffffff),
                // Shift liquidityNet to put it in the upper bits (no need for signextend since we're shifting left)
                shl(128, liquidityNet)
            )
        )
    }
}
```

Update `liquidityNet` based on the `upper` parameter. `liquidityNet` represents the net liquidity of the current `tick`. Among them, `tickLower` (left boundary point) is set to increase liquidity, and `tickUpper` (right boundary point) is set to decrease liquidity. Note that `liquidityNet` can be negative.

The purpose of this design is:
* When the `tick` moves from left to right, crossing `tickLower` means entering the position range, so the global liquidity needs to be increased; crossing `tickUpper` means leaving the position range, so the global liquidity needs to be reduced.
* When the `tick` moves from right to left, crossing `tickUpper` means entering the position range, so the global liquidity needs to be increased; crossing `tickLower` means leaving the position range, so the global liquidity needs to be reduced.

Finally, save the updated `liquidityGrossAfter` and `liquidityNet` to `TickInfo`.

### crossTick

Cross the `tick`.

```solidity
/// @notice Transitions to next tick as needed by price movement
/// @param self The Pool state struct
/// @param tick The destination tick of the transition
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @return liquidityNet The amount of liquidity added (subtracted) when tick is crossed from left to right (right to left)
function crossTick(State storage self, int24 tick, uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128)
    internal
    returns (int128 liquidityNet)
{
    unchecked {
        TickInfo storage info = self.ticks[tick];
        info.feeGrowthOutside0X128 = feeGrowthGlobal0X128 - info.feeGrowthOutside0X128;
        info.feeGrowthOutside1X128 = feeGrowthGlobal1X128 - info.feeGrowthOutside1X128;
        liquidityNet = info.liquidityNet;
    }
}
```

As introduced in the Uniswap v3 whitepaper, when a `tick` is crossed, the fee update is calculated according to the following formula:

$$
f_o(i) := f_g - f_o(i)
$$
