# Position Library

## State 定义

`State` 结构体定义了用户持有的头寸信息，包括：

- `liquidity`：用户持有的流动性数量
- `feeGrowthInside0LastX128`：用户持有流动性时的手续费增长率
- `feeGrowthInside1LastX128`：用户持有流动性时的手续费增长率

```solidity
// info stored for each user's position
struct State {
    // the amount of liquidity owned by this position
    uint128 liquidity;
    // fee growth per unit of liquidity as of the last update to liquidity or fees owed
    uint256 feeGrowthInside0LastX128;
    uint256 feeGrowthInside1LastX128;
}
```

## get

获取用户的头寸信息。计算 `positionKey`，然后返回 `position`。其中头寸由 `owner`、`tickLower`、`tickUpper` 和 `salt` 唯一确定。

```solidity
/// @notice Returns the State struct of a position, given an owner and position boundaries
/// @param self The mapping containing all user positions
/// @param owner The address of the position owner
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @param salt A unique value to differentiate between multiple positions in the same range
/// @return position The position info struct of the given owners' position
function get(mapping(bytes32 => State) storage self, address owner, int24 tickLower, int24 tickUpper, bytes32 salt)
    internal
    view
    returns (State storage position)
{
    bytes32 positionKey = calculatePositionKey(owner, tickLower, tickUpper, salt);
    position = self[positionKey];
}
```

## calculatePositionKey

计算 position key。
输入 `owner`、`tickLower`、`tickUpper` 和 `salt`，返回 `positionKey`。其中，`positionKey` 的计算方式为 `keccak256(abi.encodePacked(owner, tickLower, tickUpper, salt))`。

```solidity
/// @notice A helper function to calculate the position key
/// @param owner The address of the position owner
/// @param tickLower the lower tick boundary of the position
/// @param tickUpper the upper tick boundary of the position
/// @param salt A unique value to differentiate between multiple positions in the same range, by the same owner. Passed in by the caller.
function calculatePositionKey(address owner, int24 tickLower, int24 tickUpper, bytes32 salt)
    internal
    pure
    returns (bytes32 positionKey)
{
    // positionKey = keccak256(abi.encodePacked(owner, tickLower, tickUpper, salt))
    assembly ("memory-safe") {
        let fmp := mload(0x40)
        mstore(add(fmp, 0x26), salt) // [0x26, 0x46)
        mstore(add(fmp, 0x06), tickUpper) // [0x23, 0x26)
        mstore(add(fmp, 0x03), tickLower) // [0x20, 0x23)
        mstore(fmp, owner) // [0x0c, 0x20)
        positionKey := keccak256(add(fmp, 0x0c), 0x3a) // len is 58 bytes

        // now clean the memory we used
        mstore(add(fmp, 0x40), 0) // fmp+0x40 held salt
        mstore(add(fmp, 0x20), 0) // fmp+0x20 held tickLower, tickUpper, salt
        mstore(fmp, 0) // fmp held owner
    }
}
```

`mstore(p, v)` 用于将 `v` 存储到内存地址 `p` 处，长度为 32 字节。由于 `owner` 是 20 字节，`tickLower` 和 `tickUpper` 是 3 字节，`salt` 是 32 字节，总长度为 58 字节，因此需要分别存储到不同的内存地址中。

`fmp` 即 `Free Memory Pointer`，是 Solidity 内存管理中的一个特殊指针，保存在 `0x40` 地址处，用于记录当前可用的内存地址。默认情况下，`fmp` 指向 `0x80`。

假设 `fmp` 从 `0x00` 开始，几个变量的内存布局如下：

$$
\overbrace{0x00, ... 0x0b}^{skip, 12 bytes}, \underbrace{\overbrace{0x0c, ..., 0x1f}^{owner, 20 bytes}, \overbrace{0x20, ..., 0x22}^{tickLower , 3 bytes}, \overbrace{0x23, ..., 0x25}^{tickUpper, 3 bytes}, \overbrace{0x26, ..., 0x45}^{salt, 32 bytes}}_{keccak256(abi.encodePacked()), 58 bytes}
$$

因为 `mstore` 每次存储 32 字节，如果变量长度不足 32 字节，会自动补 0。所以需要先保存 `salt`，最后保存 `owner`。

## update

更新头寸的流动性和手续费。

```solidity
/// @notice Credits accumulated fees to a user's position
/// @param self The individual position to update
/// @param liquidityDelta The change in pool liquidity as a result of the position update
/// @param feeGrowthInside0X128 The all-time fee growth in currency0, per unit of liquidity, inside the position's tick boundaries
/// @param feeGrowthInside1X128 The all-time fee growth in currency1, per unit of liquidity, inside the position's tick boundaries
/// @return feesOwed0 The amount of currency0 owed to the position owner
/// @return feesOwed1 The amount of currency1 owed to the position owner
function update(
    State storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal returns (uint256 feesOwed0, uint256 feesOwed1) {
    uint128 liquidity = self.liquidity;

    if (liquidityDelta == 0) {
        // disallow pokes for 0 liquidity positions
        if (liquidity == 0) CannotUpdateEmptyPosition.selector.revertWith();
    } else {
        self.liquidity = LiquidityMath.addDelta(liquidity, liquidityDelta);
    }

    // calculate accumulated fees. overflow in the subtraction of fee growth is expected
    unchecked {
        feesOwed0 =
            FullMath.mulDiv(feeGrowthInside0X128 - self.feeGrowthInside0LastX128, liquidity, FixedPoint128.Q128);
        feesOwed1 =
            FullMath.mulDiv(feeGrowthInside1X128 - self.feeGrowthInside1LastX128, liquidity, FixedPoint128.Q128);
    }

    // update the position
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;
}
```

* `liquidityDelta` 为流动性变化，增加时为正，减少时为负。
* `feeGrowthInside0X128 - self.feeGrowthInside0LastX128` 计算每流动性的 `token0` 手续费增长，根据头寸持有的流动性计算应获得的手续费。
* `feeGrowthInside1X128 - self.feeGrowthInside1LastX128` 计算每流动性的 `token1` 手续费增长，根据头寸持有的流动性计算应获得的手续费。
