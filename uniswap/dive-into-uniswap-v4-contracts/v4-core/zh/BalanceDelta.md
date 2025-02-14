# BalanceDelta

## 类型说明

BalanceDelta 类型用一个 `int256` 类型同时表示 `token0` 和 `token1` 的余额变化值。

```solidity
/// @dev Two `int128` values packed into a single `int256` where the upper 128 bits represent the amount0
/// and the lower 128 bits represent the amount1.
type BalanceDelta is int256;
```

其中，高 128 位表示 `token0` 的余额变化值 `amount0`，低 128 位表示 `token1` 的余额变化值 `amount1`。

## 运算符重载

通过以下代码声明了 `BalanceDelta` 类型的运算符重载：

```solidity
using {add as +, sub as -, eq as ==, neq as !=} for BalanceDelta global;
```

当对 `BalanceDelta` 类型的变量使用 `+`、`-`、`==`、`!=` 运算符时，会调用对应 [add](#add)、[sub](#sub)、[eq](#eq)、[neq](#neq) 方法。

## 方法定义

### toBalanceDelta

将两个 `int128` 类型的 `amount0` 和 `amount1` 拼凑成 `BalanceDelta` 类型。

```solidity
function toBalanceDelta(int128 _amount0, int128 _amount1) pure returns (BalanceDelta balanceDelta) {
    assembly ("memory-safe") {
        balanceDelta := or(shl(128, _amount0), and(sub(shl(128, 1), 1), _amount1))
    }
}
```

`shl(128, _amount0)` 将 `_amount0` 左移 128 位，即将 `_amount0` 放到高 128 位，低 128 位设置为 `0`。

`sub(shl(128, 1), 1)` 将 `1` 左移 128 位，然后减去 `1`，将低 128 位设置为 `1`。
`and(sub(shl(128, 1), 1), _amount1)` 将 `_amount1` 与低 128 位 `1` 进行与操作，即只保留 `_amount1` 的低 128 位，高 128 位为 `0`。

最后，通过 `or` 将 `_amount0` 和 `_amount1` 拼凑成 `BalanceDelta` 类型。

### add

将两个 `BalanceDelta` 类型的变量相加，得到 `BalanceDelta` 类型的结果。

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

首先，通过 `sar(128, a)` 将 `a` 算数右移 128 位，即取出 `a` 的高 128 位 `a0`。

然后，通过 `signextend(15, a)` 将 `a` 的低 128 位符号扩展为 256 位，即取出 `a` 的低 128 位 `a1`，并保持符号位不变。

同理，取出 `b` 的高 128 位 `b0` 和低 128 位 `b1`。

接着，将 `a0` 和 `b0` 相加得到 `res0`，将 `a1` 和 `b1` 相加得到 `res1`。

最后，通过 [toBalanceDelta](#toBalanceDelta) 方法将 `res0` 和 `res1` 拼凑成 `BalanceDelta` 类型。

### sub

将两个 `BalanceDelta` 类型的变量相减，得到 `BalanceDelta` 类型的结果。

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

与 [add](#add) 方法类似，只是将 `a0` 和 `b0` 相减得到 `res0`，将 `a1` 和 `b1` 相减得到 `res1`。

### eq

判断两个 `BalanceDelta` 类型的变量是否相等。

```solidity
function eq(BalanceDelta a, BalanceDelta b) pure returns (bool) {
    return BalanceDelta.unwrap(a) == BalanceDelta.unwrap(b);
}
```

通过 `unwrap` 方法将 `a` 和 `b` 解包成 `int256` 类型，然后比较是否相等。

### neq

判断两个 `BalanceDelta` 类型的变量是否不相等。

```solidity
function neq(BalanceDelta a, BalanceDelta b) pure returns (bool) {
    return BalanceDelta.unwrap(a) != BalanceDelta.unwrap(b);
}
```

通过 `unwrap` 方法将 `a` 和 `b` 解包成 `int256` 类型，然后比较是否不相等。

## BalanceDeltaLibrary

BalanceDeltaLibrary 提供了 `amount0` 和 `amount1` 方法，用于获取 `BalanceDelta` 类型的变量的 `amount0` 和 `amount1` 值。

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

获取 `BalanceDelta` 类型的变量的 `amount0` 值。

通过 `sar(128, balanceDelta)` 将 `balanceDelta` 算数右移 128 位，即取出 `balanceDelta` 的高 128 位。

### amount1

获取 `BalanceDelta` 类型的变量的 `amount1` 值。

通过 `signextend(15, balanceDelta)` 将 `balanceDelta` 的低 128 位符号扩展为 256 位，即取出 `balanceDelta` 的低 128 位，并保持符号位不变。
