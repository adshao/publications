# Position Library

## State Definition

The `State` struct defines the position information held by the user, including:

- `liquidity`: The amount of liquidity held by the user
- `feeGrowthInside0LastX128`: The fee growth rate when the user holds liquidity
- `feeGrowthInside1LastX128`: The fee growth rate when the user holds liquidity

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

Get the user's position information. Calculate `positionKey`, then return `position`. The position is uniquely determined by `owner`, `tickLower`, `tickUpper`, and `salt`.

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

Calculate the position key.
Input `owner`, `tickLower`, `tickUpper`, and `salt`, and return `positionKey`. The `positionKey` is calculated as `keccak256(abi.encodePacked(owner, tickLower, tickUpper, salt))`.

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

`mstore(p, v)` is used to store `v` at memory address `p`, with a length of 32 bytes. Since `owner` is 20 bytes, `tickLower` and `tickUpper` are 3 bytes, and `salt` is 32 bytes, they need to be stored at different memory addresses.

`fmp` is the Free Memory Pointer, a special pointer in Solidity memory management, stored at address `0x40`, used to record the current available memory address. By default, `fmp` points to `0x80`.

Assuming `fmp` starts from `0x00`, the memory layout of several variables is as follows:

$$
\overbrace{0x00, ... 0x0b}^{skip, 12 bytes}, \underbrace{\overbrace{0x0c, ..., 0x1f}^{owner, 20 bytes}, \overbrace{0x20, ..., 0x22}^{tickLower , 3 bytes}, \overbrace{0x23, ..., 0x25}^{tickUpper, 3 bytes}, \overbrace{0x26, ..., 0x45}^{salt, 32 bytes}}_{keccak256(abi.encodePacked()), 58 bytes}
$$

Since `mstore` stores 32 bytes each time, if the variable length is less than 32 bytes, it will be automatically padded with 0. Therefore, `salt` needs to be stored first, and `owner` last.

## update

Update the position's liquidity and fees.

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

* `liquidityDelta` is the change in liquidity, positive for adding liquidity, negative for removing liquidity.
* `feeGrowthInside0X128 - self.feeGrowthInside0LastX128` calculates the fee growth of `token0` per unit of liquidity, and calculates the fees to be earned based on the liquidity held by the position.
* `feeGrowthInside1X128 - self.feeGrowthInside1LastX128` calculates the fee growth of `token1` per unit of liquidity, and calculates the fees to be earned based on the liquidity held by the position.
