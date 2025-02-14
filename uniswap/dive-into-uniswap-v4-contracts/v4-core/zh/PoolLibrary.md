# Pool Library

Pool Library 定义了针对单个池子的操作，包括修改流动性、交易等。其中，大部分涉及到 `liquidity`、`tick`、`fee` 等的计算逻辑都与 Uniswap v3 一样。因为 Uniswap v4 并没有对核心的 AMM 逻辑进行调整，而主要是在 Hooks 管理、单例架构、闪电记账等方面进行了架构优化。

## 结构体定义

### TickInfo

`TickInfo` 结构体定义了池子中每个 `tick` 的信息，包括 `liquidityGross`, `liquidityNet`, `feeGrowthOutside0X128`, `feeGrowthOutside1X128` 等。

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

我们在 Uniswap v3 中介绍过：

* `liquidityGross` 表示总流动性，用于判断 `tick` 是否需要初始化：
    * 如果 `mint`，则增加流动性；如果 `burn`，则减少流动性
    * 该变量与 `tick` 在不同头寸中是否作为边界低点或高点无关，只与 `mint` 或 `burn` 操作有关
    * 如果一个 `tick` 被不同的头寸同时用作 `tickLower` 和 `tickUpper`，则其`liquidityNet` 可能是 0，但 `liquidityGross` 仍然会大于 0，因此不需要再次初始化

* `liquidityNet` 表示净流动性，当 `swap` 穿越 `tick` 时，用于更新全局可用流动性`liquidity`：
    * 如果作为 `tickLower`，即边界低点（左边界点），则增加 `liquidityDelta`
    * 如果作为 `tickUpper`，即边界高点（右边界点），则减少 `liquidityDelta`

### State

`State` 结构体定义了池子的状态：

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

其中：

* `slot0` 定义池子手续费（LP 手续费、协议手续费）、当前价格等基本信息。
* `feeGrowthGlobal0X128` 和 `feeGrowthGlobal1X128` 分别表示 `token0` 和 `token1` 的全局手续费增长。
* `liquidity` 表示当前池子的总流动性。
* `ticks` 是一个 `mapping`，存储了每个 `tick` 的信息。
* `tickBitmap` 是一个 `mapping`，存储了每个 `tick` 的 `bitmap` 信息，用于快速查询下一个已被初始化的 `tick`。
* `positions` 是一个 `mapping`，存储了每个头寸的信息。

#### Slot0

在 Uniswap v3 中，`Slot0` 是一个 `struct`；而在 Uniswap v4 中，为了节省 gas 费用，`Slot0` 是一个 `bytes32`，用于存储 packed 后的 `slot0` 信息。

`Slot0` 的 Layout 定义如下：

```
24 bits empty | 24 bits lpFee | 12 bits protocolFee 1->0 | 12 bits protocolFee 0->1 | 24 bits tick | 160 bits sqrtPriceX96
```

从左到右（从高位到低位）分别是：
* 24 bits empty：空位，用于填充，目前未使用
* 24 bits lpFee：流动性提供者手续费
* 12 bits protocolFee 1->0：从 `token1` 到 `token0` 的协议手续费
* 12 bits protocolFee 0->1：从 `token0` 到 `token1` 的协议手续费
* 24 bits tick：当前价格对应的 `tick`
* 160 bits sqrtPriceX96：当前价格的 `sqrtPriceX96`

`Slot0Library` 中定义了从 `slot0` 中（从指定 bits）读取和设置各个字段的方法。

### ModifyLiquidityParams

ModifyLiquidityParams 的定义如下：

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

其中：

* `owner`：头寸的拥有者
* `tickLower` 和 `tickUpper`：头寸的 `tick` 范围
* `liquidityDelta`：流动性变化，如果是增加流动性，则为正数；如果是减少流动性，则为负数
* `tickSpacing`：`tick` 之间的间隔
* `salt`：用于区分不同头寸

### SwapParams

SwapParams 的定义如下：

```solidity
struct SwapParams {
    int256 amountSpecified;
    int24 tickSpacing;
    bool zeroForOne;
    uint160 sqrtPriceLimitX96;
    uint24 lpFeeOverride;
}
```

其中：

* `amountSpecified`：指定的代币数量。
  > 注意，这里与 Uniswap v3 不同，在 Uniswap v4 中，如果 `amountSpecified` 为负，则表示希望输入的代币数量，即 `exactInput`；如果为正，表示希望输出的代币数量，即 `exactOutput`
* `tickSpacing`：`tick` 之间的间隔
* `zeroForOne`：交易方向，`true` 表示 `token0` 到 `token1`，`false` 表示 `token1` 到 `token0`
* `sqrtPriceLimitX96`：交易价格限制，可以是最高价格或最低价格
* `lpFeeOverride`：LP 手续费，用于覆盖池子的默认 LP 手续费

### SwapResult

SwapResult 的定义如下：

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

其中：

* `sqrtPriceX96`：当前价格
* `tick`：当前价格对应的 `tick`
* `liquidity`：当前处于价格区间内的流动性

### StepComputations

StepComputations 的定义如下：

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

其中：

* `sqrtPriceStartX96`：交易开始时的价格
* `tickNext`：当前交易方向的下一个用于交易的 `tick`
* `initialized`：`tickNext` 是否已经初始化
* `sqrtPriceNextX96`：下一个 `tick` 对应的价格
* `amountIn`：输入代币数量
* `amountOut`：输出代币数量
* `feeAmount`：交易手续费
* `feeGrowthGlobalX128`：输入代币的全局手续费增长

## 函数定义

### checkTicks

检查 `tickLower` 和 `tickUpper` 是否合法，即 `tickLower` 小于 `tickUpper`，以及 `tickLower` 和 `tickUpper` 是否在合法范围内。

```solidity
/// @dev Common checks for valid tick inputs.
function checkTicks(int24 tickLower, int24 tickUpper) private pure {
    if (tickLower >= tickUpper) TicksMisordered.selector.revertWith(tickLower, tickUpper);
    if (tickLower < TickMath.MIN_TICK) TickLowerOutOfBounds.selector.revertWith(tickLower);
    if (tickUpper > TickMath.MAX_TICK) TickUpperOutOfBounds.selector.revertWith(tickUpper);
}
```

关于 `TickMath.MIN_TICK` 和 `TickMath.MAX_TICK` 是如何计算的，我们在 Uniswap v3 中介绍过：

因为 Uniswap v3 支持的价格（ $\frac{token1}{token0}$ ）区间为 $[2^{-128}, 2^{128}]$ ，根据 Uniswap v3 白皮书中的公式 6.1:

$$
p(i) = 1.0001^i
$$

因此，对应的最大 `tick`（即 `MAX_TICK`）为：

$$
i = \lfloor log_{1.0001}{p(i)} \rfloor = \lfloor log_{1.0001}{2^{128}} \rfloor = \lfloor 887272.7517970635 \rfloor = 887272
$$

最小 `tick`（即 `MIN_TICK`）为：

$$
i = \lceil log_{1.0001}{2^{-128}} \rceil = \lceil -887272.7517970635 \rceil = -887272
$$

### initialize

初始化池子，设置池子的 `sqrtPriceX96` 和 `lpFee` 等字段。

```solidity
function initialize(State storage self, uint160 sqrtPriceX96, uint24 lpFee) internal returns (int24 tick) {
    if (self.slot0.sqrtPriceX96() != 0) PoolAlreadyInitialized.selector.revertWith();

    tick = TickMath.getTickAtSqrtPrice(sqrtPriceX96);

    // the initial protocolFee is 0 so doesn't need to be set
    self.slot0 = Slot0.wrap(bytes32(0)).setSqrtPriceX96(sqrtPriceX96).setTick(tick).setLpFee(lpFee);
}
```

未初始化的池子的当前价格 `sqrtPriceX96` 为 0，因此可以通过判断 `sqrtPriceX96` 是否为 0 来判断池子是否已经初始化。

根据 `sqrtPriceX96` 计算当前价格对应的 `tick`，具体的算法可参考 Uniswap v3 的 [TickMath.getTickAtSqrtPrice](../../../dive-into-uniswap-v3-contracts/README_zh.md#gettickatsqrtratio) 方法。

`initialize` 方法设置 `slot0` 的 `sqrtPriceX96`、`tick` 和 `lpFee` 字段。

### setProtocolFee

设置协议手续费。

```solidity
function setProtocolFee(State storage self, uint24 protocolFee) internal {
    self.checkPoolInitialized();
    self.slot0 = self.slot0.setProtocolFee(protocolFee);
}
```

确认 Pool 已经初始化，如前所述，`checkPoolInitialized` 通过判断当前 Pool State 的 `slot.sqrtPriceX96` 是否为 0 来判断 Pool 是否已经初始化。

### setLPFee

设置 LP 手续费，仅支持动态手续费的池子可以设置 LP 手续费。

```solidity
/// @notice Only dynamic fee pools may update the lp fee.
function setLPFee(State storage self, uint24 lpFee) internal {
    self.checkPoolInitialized();
    self.slot0 = self.slot0.setLpFee(lpFee);
}
```

### modifyLiquidity

修改某个用户头寸的流动性，如增加或减少流动性。

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

该方法的输入参数为：

* [State](#state) self：池子的状态
* [ModifyLiquidityParams](#modifyliquidityparams) params：修改流动性的参数，代表用户头寸的信息和流动性变化

返回值为：

* `BalanceDelta` delta：流动性变化后的 token 余额变化
* `BalanceDelta` feeDelta：流动性变化后的手续费变化，为非负数。

[BalanceDelta](./BalanceDelta.md) 是一个 int256 类型，前 128 位表示 `token0` 的余额变化，后 128 位表示 `token1` 的余额变化。如果为正数，则表示允许用户提取 `token0` 或 `token1`；如果为负数，则表示用户需要存入 `token0` 或 `token1`。

```solidity
{
    int128 liquidityDelta = params.liquidityDelta;
    int24 tickLower = params.tickLower;
    int24 tickUpper = params.tickUpper;
    checkTicks(tickLower, tickUpper);
```

通过 [checkTicks](#checkticks) 检查 `tickLower` 和 `tickUpper` 的合法性。

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

通过 [updateTick](#updatetick) 方法分别更新 `tickLower` 和 `tickUpper` 的流动性。

如果 `liquidityDelta >= 0`，则表示增加流动性，需要判断 `tickLower` 和 `tickUpper` 的总流动性是否超过了 `tickSpacing` 对应的单个 `tick` 支持的最大流动性。

如果 `tickLower` 或 `tickUpper` 翻转，则更新 `tickBitmap` 中对应的 `tick` 的 bit 信息，从 `0` 到 `1` 或从 `1` 到 `0`。

获取 `tickLower` 和 `tickUpper` 之间的手续费增长，通过 [getFeeGrowthInside](#getfeegrowthinside) 方法。

获取用户头寸，通过 [positions.get](./PositionLibrary.md#get) 方法。其中，头寸 `position` 由 `owner`、`tickLower`、`tickUpper` 和 `salt` 唯一确定。
通过 [position.update](./PositionLibrary.md#update) 更新头寸的流动性，计算对应的 LP 手续费。

如果 `liquidityDelta < 0`，则表示减少流动性，同时，如果对应的 `tick` 发生翻转，则表示该 `tick` 没有任何头寸与之关联，则可以删除池子对应的 `tick` 数据。

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

如果 `liquidityDelta != 0`，即流动性发生变化，则需要计算 `delta`，表示需要用户提供或者用户可取回的 `token0` 和 `token1` 的数量。

如果当前 `tick` 小于 `tickLower`，由于 `tick` 大小与 $\sqrt{P}$（即 $\sqrt{\frac{y}{x}}$ ）成正比，意味着在大于当前 `tick` 的区间， $x$ 的价值更高（需要更少的 $x$ ），因此添加流动性时需在该部分提供 $x$ 代币，即需要计算 `token0`；反之，则提供 $y$ 代币，即计算 `token1`。

可参考 Uniswap v3 了解如何计算 [getAmount0Delta](../../../dive-into-uniswap-v3-contracts/README_zh.md#getamount0delta) 和 [getAmount1Delta](../../../dive-into-uniswap-v3-contracts/README_zh.md#getamount1delta)。

### swap

执行交易操作，完成从 `token0` 交换到 `token1` 或者相反。

整个 `swap` 代码的逻辑与 Uniswap v3 的 [swap](../../../dive-into-uniswap-v3-contracts/README_zh.md#swap) 几乎完全一样。

以下为 Uniswap v4 的 `swap` 方法的实现：

```solidity
/// @notice Executes a swap against the state, and returns the amount deltas of the pool
/// @dev PoolManager checks that the pool is initialized before calling
function swap(State storage self, SwapParams memory params)
    internal
    returns (BalanceDelta swapDelta, uint256 amountToProtocol, uint24 swapFee, SwapResult memory result)
{
```

该方法的输入参数为：

* [State](#state) self：池子的状态
* [SwapParams](#swapparams) params：交易参数，包括交易数量、交易方向、价格限制等

返回值为：

* `BalanceDelta` swapDelta：交易后的 token 余额变化
* `uint256` amountToProtocol：协议手续费
* `uint24` swapFee：交易手续费
* [SwapResult](#swapresult) result：交易结果，包括交易结束后的价格、`tick`和流动性

代码如下：

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

初始化相关参数。

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

如果 hook 返回了新的 `lpFee`，则使用新的 `lpFee`；否则使用创建池子时配置的 `lpFee`。

如果协议手续费为 0，则使用 `lpFee` 作为交易手续费 `swapFee`；否则，计算交易手续费。由于 `fee` 的单位是百分之一 BIP，即 ${10}^{6}$，交易手续费的计算方式如下：

$$
swapFee = protocolFee + lpFee \cdot (1 - \frac{protocolFee}{{10}^{6}})
$$

`swapFee` 包括 `protocolFee` 和 `lpFee`，其中，`lpFee` 是扣除掉 `protocolFee` 之后进行计算的。

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

判断 `swapFee` 的合法性。
如果输入的 `token` 数量为 0，则退出。

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

检查 `sqrtPriceLimitX96` 的合法性：

* 如果 `zeroForOne` 为 `true`，即从 `token0` 到 `token1` 的交易，交易之后 `token0`（ $x$ ） 变多，`token1`（ $y$ ） 变少，即 $ \sqrt{P} = \sqrt{\frac{y}{x}} $ 变小，因此目标价格 `sqrtPriceLimitX96` 应该小于当前价格，但不能小于 `MIN_SQRT_PRICE`。
* 反之，目标价格 `sqrtPriceLimitX96` 应该大于当前价格，但不能大于 `MAX_SQRT_PRICE`。

我们在 [Uniswap v3 白皮书](../../../dive-into-uniswap-v3-whitepaper/README_zh.md#621-price-and-liquidity-价格和流动性) 中介绍过，`swap` 流程的核心逻辑为：

* 任意时刻， 流动性 $L$ 和 价格 $\sqrt{P}$ 只有其中一个值会变化。
* 在 `swap` 时：
  * 所有处于当前价格区间的流动性构成总流动性 `liquidity`
  * 每个 `swap` 将拆成多个 `step`，每个 `step` 只在一个 `tick` 区间内执行 `swap` 操作，因此：
    * 在 `swap` 时， $L$ 不变， $\sqrt{P}$ 变化；
    * 当价格穿越 `cross tick` 时，修改总流动性 $L$ ，保持 $\sqrt{P}$ 不变，再继续进行下一个 `swap step`；
* 在 add/burn liquidity 时， $L$ 变化， $\sqrt{P}$ 不变。

接下来的 `swap` 代码即实现上述逻辑。

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

将整个 `swap` 操作拆成多个 `swap step`。
在 `while` 循环中执行 `swap step`，针对每个 `step`：

1. 通过 `tickBitmap.nextInitializedTickWithinOneWord` 获取至多一个初始化的 `tick`，并计算该 `tick` 对应的价格 `sqrtPriceNextX96`。

1. 通过 [SwapMath.computeSwapStep](../../../dive-into-uniswap-v3-contracts/README_zh.md#computeswapstep) 计算一个 `tick` 内的交易，在该步骤里，流动性 $L$ 不变，仅价格 $\sqrt{P}$ 发生变化。计算该 `swap step` 的结束价格 `sqrtPriceX96`，输入 `amountIn`、输出 `amountOut` 和手续费 `feeAmount`。返回结果均为非负数。

1. 更新 `amountSpecifiedRemaining` 和 `amountCalculated`：
   * 如果 `params.amountSpecified > 0`，即 `exactOutput`，则 `amountSpecifiedRemaining` 表示输出代币，需减去 `amountOut`，`amountCalculated` 表示输入代币，用负数表示需要用户存入对应的 `amountIn` 和 `feeAmount`；
   * 反之，即 `exactInput`，`amountSpecifiedRemaining` 表示输入代币，因为此时其为负数，因此增加 `amountIn` 和 `feeAmount` 对应的数量，`amountCalculated` 表示输出代币，用正数表示允许用户取出对应的 `amountOut`。

1. 如果 `protocolFee > 0`，则计算该 `step` 中的协议手续费 `amountToProtocol`，并将 `feeAmount` 扣除协议手续费。`protocolFee` 的计算方式为：

    $$
    protocol fee = (amountIn + feeAmount) \cdot \frac{protocolFee}{10^6}
    $$

1. 如果当前全局流动性 `result.liquidity` 大于 0，则使用本次 `step` 收取的 `feeAmount`（扣除协议手续费后）计算每流动性应得的的手续费，并更新全局手续费增长 `feeGrowthGlobalX128`。

1. 如果 `swap step` 结束后的价格 `result.sqrtPriceX96` 等于下一个 `tick` 的价格 `step.sqrtPriceNextX96`，则表示当前 `tick` 区间已全部完成。因为 `nextInitializedTickWithinOneWord` 可能返回未初始化的 `tick`。如果当前 `tick` 是初始化的，意味着该 `tick` 已完成（被穿越 crossed），因此执行 [crossTick](#crosstick) 操作，根据 `tick` 拥有的 `liquidityNet`，更新全局流动性 `result.liquidity`：
   * 由于在 `tick` 首次初始化的时候，我们默认设置 `tickLower` 的 `liquidityNet` 为正数，`tickUpper` 的 `liquidityNet` 为负数。
   * 如果 `zeroForOne` 为 `true`，即 `token0` 到 `token1` 的交易，表示随着 `swap` 的进行，`token0` 的数量变多，`token1` 的数量变少，因此 $\sqrt{P} = \sqrt{\frac{y}{x}}$ 变小，`tick` 从右往左移动：
      * 如果 `tick` 为 `tickLower`，即穿越左边界，将跳出头寸区间，因此需要减少全局流动性；
      * 如果 `tick` 为 `tickUpper`，即穿越右边界，将进入头寸区间，因此需要增加全局流动性。
      * 因此当 `zeroForOne` 为 `true` 时，`liquidityNet` 需要执行取反操作。
   * 同理，如果 `zeroForOne` 为 `false`，即 `token1` 到 `token0` 的交易，`tick` 从左往右移动：
      * 如果 `tick` 为 `tickLower`，即穿越左边界，将进入头寸区间，因此需要增加全局流动性；
      * 如果 `tick` 为 `tickUpper`，即穿越右边界，将跳出头寸区间，因此需要减少全局流动性。
      * 因此当 `zeroForOne` 为 `false` 时，`liquidityNet` 不需要执行取反操作。

1. 将 `tick` 更新为下一个 `tick`，如果 `zeroForOne` 为 `true`，则 `tick` 变小， 因此下一个为 `step.tickNext - 1`，否则为 `step.tickNext`。

1. 如果 `swap step` 结束后的价格 `result.sqrtPriceX96` 未到达下一个 `tick` 的价格 `step.sqrtPriceNextX96`，则表示此时要么 `amountRemaining` 为 0，要么价格达到 `sqrtPriceLimit`。如果当前价格不等于 `sqrtPriceStartX96`，则更新 `tick`。

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

此时 `swap` 结束，要么 `amountRemaining` 为 0，要么价格达到 `sqrtPriceLimit`。更新 `slot0`、全局流动性 `liquidity` 和全局手续费增长 `feeGrowthGlobalX128`。

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

`params.amountSpecified < 0` 表示 `exactInput`，结合 `zeroForOne`，可以有以下组合：

| zeroForOne | exactInput | 说明 | amountSpecified | amountCalculated |
| --- | --- | --- | --- | --- |
| true | true | 从 `token0` 到 `token1` 的交易，输入代币数量已知，输出代币数量未知 | 输入的 `token0` 代币数量，负数 | 输出的 `token1` 代币数量 |
| true | false | 从 `token0` 到 `token1` 的交易，输入代币数量未知，输出代币数量已知 | 输出的 `token1` 代币数量，正数 | 输入的 `token0` 代币数量 |
| false | true | 从 `token1` 到 `token0` 的交易，输入代币数量已知，输出代币数量未知 | 输入的 `token1` 代币数量，负数 | 输出的 `token0` 代币数量 |
| false | false | 从 `token1` 到 `token0` 的交易，输入代币数量未知，输出代币数量已知 | 输出的 `token0` 代币数量，正数 | 输入的 `token1` 代币数量 |

因此，当 `zeroForOne != (params.amountSpecified < 0)` 时，`amountSpecified` 总是表示 `token1` 的代币数量，`params.amountSpecified - amountSpecifiedRemaining` 为实际需要的 `token1` 代币数量；`amountCalculated` 总是表示 `token0` 的代币数量。

最终计算的 `swapDelta` 即为本次 `swap` 操作的 token 余额变化，前 128 位表示 `token0` 的余额变化，后 128 位表示 `token1` 的余额变化。负数表示需要用户存入对应的代币，正数表示用户可以取出对应的代币。

### donate

向池子捐赠 `token0` 和 `token1` 代币。

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

delta 前 128 位表示 `token0` 的余额变化，后 128 位表示 `token1` 的余额变化。负数表示需要用户存入对应的代币，正数表示用户可以取出对应的代币。因为是捐赠代币，因此需要用户存入对应数量的代币。

将 `amount` 除以 `liquidity` 得到每份流动性的全局手续费增长 `feeGrowthGlobal0X128` 和 `feeGrowthGlobal1X128`，更新对应的值。

### getFeeGrowthInside

计算 `tickLower` 和 `tickUpper` 之间的手续费增长。

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

根据 Uniswap v3 白皮书介绍，手续费的计算方式为：

根据当前价格是否在区间内，可以使用一个公式计算每份流动性在 `tick` $i$ 之上（ $f_a$ ）和之下（ $f_b$ ）获取的手续费（根据当前 `tick` 序号 $i_c$ 是否大于等于 $i$ ）：

$$
f_a(i) = \begin{cases} f_g - f_o(i) & \text{$i_c \geq i$}\\
f_o(i) & \text{$i_c < i$} \end{cases}
$$

$$
f_b(i) = \begin{cases} f_o(i) & \text{$i_c \geq i$}\\
f_g - f_o(i) & \text{$i_c < i$}\end{cases}
$$

我们可以使用上述函数计算任意两个 `tick`（低点 `tick` $i_l$和高点 `tick` $i_u$）区间内，每个流动性累计的全部手续费 $f_r$：

$$
f_r = f_g - f_b(i_l) - f_a(i_u)
$$

结合上述公式，根据池子当前 `tick` 与区间边界 `tickLower`、`tickUpper`的关系，可推导区间内手续费的计算方式如下：

$$
f_r = \begin{cases} 
f_o(i_l) - f_o(i_u) & \text{$i_c < i_l < i_u$}\\
f_g - f_o(i_l) - f_o(i_u) & \text{$i_l \leq i_c < i_u$}\\
f_o(i_u) - f_o(i_l) & \text{$i_l < i_u \leq i_c$} \end{cases}
$$

`getFeeGrowthInside` 代码即实现了上述逻辑。

### updateTick

更新 `tick` 的流动性。

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

该方法的输入参数为：

* [State](#state) self：池子的状态
* `tick`：需要更新的 `tick`
* `liquidityDelta`：流动性变化，如果是增加流动性，则为正数；如果是减少流动性，则为负数
* `upper`：是否更新头寸的上界 `tick`

返回值为：

* `flipped`：`tick` 是否翻转，即是否从初始化到未初始化，或者从未初始化到初始化
* `liquidityGrossAfter`：更新后的 `tick` 的总流动性

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

根据 `tick` 获取对应的 `TickInfo`：

* 计算 `liquidityGross`。其中，`LiquidityMath.addDelta` 执行 `uint128` 和 `int128` 的加法，确保最终结果不会发生正溢出或负溢出。因此，如果 `liquidityDelta` 为正数，则增加总流动性；如果 `liquidityDelta` 为负数，则减少总流动性。最终，`tickInfo.liquidityGross` 一定大于等于 0。
* 判断 `tick` 是否翻转，根据 `liquidityGrossAfter` 和 `liquidityGrossBefore` 是否有且仅有一个为 0 来判断。
* 参考 Uniswap v3 白皮书的公式 (6.21)，如果 `tick` 小于等于当前池子的价格，则分别使用全局手续费增长 `feeGrowthGlobal0X128` 和 `feeGrowthGlobal1X128` 初始化 `feeGrowthOutside0X128` 和 `feeGrowthOutside1X128`。
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

根据 `upper` 参数，更新 `liquidityNet`。`liquidityNet` 表示当前 `tick` 的净流动性。其中，`tickLower` （左边界点）设置为增加流动性，`tickUpper` （右边界点）设置为减少流动性。注意，`liquidityNet` 可以为负。

这样设计的目的在于：
* 当 `tick` 从左往右移动，穿越 `tickLower` 时，则表示进入头寸区间，需要增加全局流动性；穿越 `tickUpper` 时，则表示离开头寸区间，需要减少全局流动性。
* 当 `tick` 从右往左移动，穿越 `tickUpper` 时，则表示进入头寸区间，需要增加全局流动性；穿越 `tickLower` 时，则表示离开头寸区间，需要减少全局流动性。

最后，将更新后的 `liquidityGrossAfter` 和 `liquidityNet` 保存到 `TickInfo` 中。

### crossTick

穿越 `tick`。

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

根据 Uniswap v3 白皮书公式，当 `tick` 被穿越时，手续费的更新按照如下公式计算：

$$
f_o(i) := f_g - f_o(i)
$$
