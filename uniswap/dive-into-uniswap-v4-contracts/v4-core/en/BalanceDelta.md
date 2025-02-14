# BalanceDelta

## Type Description

The BalanceDelta type uses an `int256` type to represent the balance changes of `token0` and `token1` simultaneously.

```solidity
/// @dev Two `int128` values packed into a single `int256` where the upper 128 bits represent the amount0
/// and the lower 128 bits represent the amount1.
type BalanceDelta is int256;
```

The upper 128 bits represent the balance change value `amount0` of `token0`, and the lower 128 bits represent the balance change value `amount1` of `token1`.

## Operator Overloading

The following code declares operator overloading for the `BalanceDelta` type:

```solidity
using {add as +, sub as -, eq as ==, neq as !=} for BalanceDelta global;
```

When using the `+`, `-`, `==`, `!=` operators on `BalanceDelta` type variables, the corresponding [add](#add), [sub](#sub), [eq](#eq), [neq](#neq) methods will be called.

## Method Definitions

### toBalanceDelta

Combine two `int128` types `amount0` and `amount1` into a `BalanceDelta` type.

```solidity
function toBalanceDelta(int128 _amount0, int128 _amount1) pure returns (BalanceDelta balanceDelta) {
    assembly ("memory-safe") {
        balanceDelta := or(shl(128, _amount0), and(sub(shl(128, 1), 1), _amount1))
    }
}
```

`shl(128, _amount0)` shifts `_amount0` 128 bits to the left, placing `_amount0` in the upper 128 bits, and setting the lower 128 bits to `0`.

`sub(shl(128, 1), 1)` shifts `1` 128 bits to the left, then subtracts `1`, setting the lower 128 bits to `1`.
`and(sub(shl(128, 1), 1), _amount1)` performs a bitwise AND operation between `_amount1` and the lower 128 bits `1`, retaining only the lower 128 bits of `_amount1`, with the upper 128 bits set to `0`.

Finally, the `or` operation combines `_amount0` and `_amount1` into a `BalanceDelta` type.

### add

Add two `BalanceDelta` type variables to get a `BalanceDelta` type result.

```solidity
function add(BalanceDelta a, BalanceDelta b) pure returns (BalanceDelta) {
    int256 res0;
    int256 res1;
    assembly ("memory-safe") {
        let a0 := sar(128, a)
        let a1 := signextend(15, a)
        let b0 := sar(128, b)
        let b1 := signextend(15, b)
        res0 := add(a0, b0)
        res1 := add(a1, b1)
    }
    return toBalanceDelta(res0.toInt128(), res1.toInt128());
}
```

First, `sar(128, a)` performs an arithmetic right shift on `a` by 128 bits, extracting the upper 128 bits `a0` of `a`.

Then, `signextend(15, a)` sign-extends the lower 128 bits of `a` to 256 bits, extracting the lower 128 bits `a1` of `a` while preserving the sign bit.

Similarly, extract the upper 128 bits `b0` and lower 128 bits `b1` of `b`.

Next, add `a0` and `b0` to get `res0`, and add `a1` and `b1` to get `res1`.

Finally, use the [toBalanceDelta](#toBalanceDelta) method to combine `res0` and `res1` into a `BalanceDelta` type.

### sub

Subtract two `BalanceDelta` type variables to get a `BalanceDelta` type result.

```solidity
function sub(BalanceDelta a, BalanceDelta b) pure returns (BalanceDelta) {
    int256 res0;
    int256 res1;
    assembly ("memory-safe") {
        let a0 := sar(128, a)
        let a1 := signextend(15, a)
        let b0 := sar(128, b)
        let b1 := signextend(15, b)
        res0 := sub(a0, b0)
        res1 := sub(a1, b1)
    }
    return toBalanceDelta(res0.toInt128(), res1.toInt128());
}
```

Similar to the [add](#add) method, except that `a0` and `b0` are subtracted to get `res0`, and `a1` and `b1` are subtracted to get `res1`.

### eq

Determine whether two `BalanceDelta` type variables are equal.

```solidity
function eq(BalanceDelta a, BalanceDelta b) pure returns (bool) {
    return BalanceDelta.unwrap(a) == BalanceDelta.unwrap(b);
}
```

Use the `unwrap` method to unpack `a` and `b` into `int256` types, then compare for equality.

### neq

Determine whether two `BalanceDelta` type variables are not equal.

```solidity
function neq(BalanceDelta a, BalanceDelta b) pure returns (bool) {
    return BalanceDelta.unwrap(a) != BalanceDelta.unwrap(b);
}
```

Use the `unwrap` method to unpack `a` and `b` into `int256` types, then compare for inequality.

## BalanceDeltaLibrary

BalanceDeltaLibrary provides `amount0` and `amount1` methods to get the `amount0` and `amount1` values of `BalanceDelta` type variables.

```solidity
/// @notice Library for getting the amount0 and amount1 deltas from the BalanceDelta type
library BalanceDeltaLibrary {
    /// @notice A BalanceDelta of 0
    BalanceDelta public constant ZERO_DELTA = BalanceDelta.wrap(0);

    function amount0(BalanceDelta balanceDelta) internal pure returns (int128 _amount0) {
        assembly ("memory-safe") {
            _amount0 := sar(128, balanceDelta)
        }
    }

    function amount1(BalanceDelta balanceDelta) internal pure returns (int128 _amount1) {
        assembly ("memory-safe") {
            _amount1 := signextend(15, balanceDelta)
        }
    }
}
```

### amount0

Get the `amount0` value of a `BalanceDelta` type variable.

Use `sar(128, balanceDelta)` to perform an arithmetic right shift on `balanceDelta` by 128 bits, extracting the upper 128 bits.

### amount1

Get the `amount1` value of a `BalanceDelta` type variable.

Use `signextend(15, balanceDelta)` to sign-extend the lower 128 bits of `balanceDelta` to 256 bits, extracting the lower 128 bits while preserving the sign bit.
