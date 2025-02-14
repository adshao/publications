# DeltaResolver

DeltaResolver 是一个抽象合约（Abstract Contract），用于向 `PoolManager` 合约同步（sync）、转移代币（send）以及结算资金（settle）。

## 方法定义

### _pay

DeltaResolver 定义了一个抽象方法 `_pay`，实现将指定数量的代币支付给 `poolManager`。所有继承 DeltaResolver 的合约都需要实现 `_pay` 方法。

如 [PositionManager](./PositionManager.md) 合约继承了 DeltaResolver 合约，并实现了 [_pay](./PositionManager.md#_pay) 方法。

输入参数：

- `Currency token`：需要支付的代币
- `address payer`：支付者地址
- `uint256 amount`：支付数量

```solidity
/// @notice Abstract function for contracts to implement paying tokens to the poolManager
/// @dev The recipient of the payment should be the poolManager
/// @param token The token to settle. This is known not to be the native currency
/// @param payer The address who should pay tokens
/// @param amount The number of tokens to send
function _pay(Currency token, address payer, uint256 amount) internal virtual;
```

### _getFullDebt

获取当前合约在 `PoolManager` 的全部欠款（负 delta）。取反后，返回值为 `uint256` 类型。

输入参数：

- `Currency currency`：代币地址

```solidity
/// @notice Obtain the full amount owed by this contract (negative delta)
/// @param currency Currency to get the delta for
/// @return amount The amount owed by this contract as a uint256
function _getFullDebt(Currency currency) internal view returns (uint256 amount) {
    int256 _amount = poolManager.currencyDelta(address(this), currency);
    // If the amount is positive, it should be taken not settled.
    if (_amount > 0) revert DeltaNotNegative(currency);
    // Casting is safe due to limits on the total supply of a pool
    amount = uint256(-_amount);
}
```

### _getFullCredit

获取当前合约在 `PoolManager` 全部可取回金额（正 delta）。返回值为 `uint256` 类型。

输入参数：

- `Currency currency`：代币地址

```solidity
/// @notice Obtain the full credit owed to this contract (positive delta)
/// @param currency Currency to get the delta for
/// @return amount The amount owed to this contract as a uint256
function _getFullCredit(Currency currency) internal view returns (uint256 amount) {
    int256 _amount = poolManager.currencyDelta(address(this), currency);
    // If the amount is negative, it should be settled not taken.
    if (_amount < 0) revert DeltaNotPositive(currency);
    amount = uint256(_amount);
}
```

### _take

提取单个代币的余额。由 `PoolManager` 合约向 `recipient` 转移代币。

输入参数：

- `currency`：代币
- `recipient`：接收者
- `amount`：提取金额

```solidity
/// @notice Take an amount of currency out of the PoolManager
/// @param currency Currency to take
/// @param recipient Address to receive the currency
/// @param amount Amount to take
/// @dev Returns early if the amount is 0
function _take(Currency currency, address recipient, uint256 amount) internal {
    if (amount == 0) return;
    poolManager.take(currency, recipient, amount);
}
```

如果 `amount` 为 0，直接返回。

调用 [PoolManager.take](../../v4-core/zh/PoolManager.md#take) 方法，从 `poolManager` 提取指定数量的代币。

### _settle

结算单个代币的欠款。需要向 `PoolManager` 支付代币。

在 [PoolManager.settle](../../v4-core/zh/PoolManager.md#settle) 方法中，我们介绍了结算欠款的流程：

1. 调用 [PoolManager.sync](../../v4-core/zh/PoolManager.md#sync) 方法，同步代币余额；
2. 向 `PoolManager` 转移代币；
3. 调用 [PoolManager.settle](../../v4-core/zh/PoolManager.md#settle) 方法，结算记账余额。

输入参数：

- `currency`：代币
- `payer`：支付者
- `amount`：支付金额

```solidity
/// @notice Pay and settle a currency to the PoolManager
/// @dev The implementing contract must ensure that the `payer` is a secure address
/// @param currency Currency to settle
/// @param payer Address of the payer
/// @param amount Amount to send
/// @dev Returns early if the amount is 0
function _settle(Currency currency, address payer, uint256 amount) internal {
    if (amount == 0) return;

    poolManager.sync(currency);
    if (currency.isAddressZero()) {
        poolManager.settle{value: amount}();
    } else {
        _pay(currency, payer, amount);
        poolManager.settle();
    }
}
```

如果 `amount` 为 0，直接返回。

调用 [poolManager.sync](../../v4-core/zh/PoolManager.md#sync) 方法，同步 `poolManager` 中的代币余额。

如果 `currency` 为 `ADDRESS_ZERO`，即原生 ETH，通过 `{value: amount}` 向 `poolManager` 转入 ETH，调用 [PoolManager.settle](../../v4-core/zh/PoolManager.md#settle) 方法，结算记账余额。

否则，对于 ERC20 代币，调用 [_pay](#_pay) 方法，向 `poolManager` 转移代币，然后调用 [PoolManager.settle](../../v4-core/zh/PoolManager.md#settle) 方法，结算记账余额。

### _mapSettleAmount

`_mapSettleAmount` 方法根据 `amount` 和 `currency` 计算待结算的代币数量：
* 如果 `amount` 为 `1 << 255`，即 `ActionConstants.CONTRACT_BALANCE`，则表示使用当前合约的代币余额进行结算
* 否则，如果 `amount` 为 `0`，即 `ActionConstants.OPEN_DELTA`，则表示结算所有欠款
* 否则，表示结算指定 `amount` 数量的代币

```solidity
/// @notice Calculates the amount for a settle action
function _mapSettleAmount(uint256 amount, Currency currency) internal view returns (uint256) {
    if (amount == ActionConstants.CONTRACT_BALANCE) {
        return currency.balanceOfSelf();
    } else if (amount == ActionConstants.OPEN_DELTA) {
        return _getFullDebt(currency);
    } else {
        return amount;
    }
}
```

### _mapTakeAmount

`_mapTakeAmount` 方法根据 `amount` 和 `currency` 计算提取的代币数量：
* 如果 `amount` 为 `0`，则表示提取当前合约在 `PoolManager` 中的所有可提取余额
* 否则，表示提取指定 `amount` 数量的代币

```solidity
/// @notice Calculates the amount for a take action
function _mapTakeAmount(uint256 amount, Currency currency) internal view returns (uint256) {
    if (amount == ActionConstants.OPEN_DELTA) {
        return _getFullCredit(currency);
    } else {
        return amount;
    }
}
```

### _mapWrapUnwrapAmount

计算包装/解包的代币数量。

输入参数：

- `inputCurrency`：输入的代币，可以是原生代币或包装代币
- `amount`：包装/解包的数量，可以是 `CONTRACT_BALANCE`、`OPEN_DELTA` 或具体数量
- `outputCurrency`：包装/解包后的代币，用户在 `PoolManager` 上可能欠费的代币

```solidity
/// @notice Calculates the sanitized amount before wrapping/unwrapping.
/// @param inputCurrency The currency, either native or wrapped native, that this contract holds
/// @param amount The amount to wrap or unwrap. Can be CONTRACT_BALANCE, OPEN_DELTA or a specific amount
/// @param outputCurrency The currency after the wrap/unwrap that the user may owe a balance in on the poolManager
function _mapWrapUnwrapAmount(Currency inputCurrency, uint256 amount, Currency outputCurrency)
    internal
    view
    returns (uint256)
{
    // if wrapping, the balance in this contract is in ETH
    // if unwrapping, the balance in this contract is in WETH
    uint256 balance = inputCurrency.balanceOf(address(this));
    if (amount == ActionConstants.CONTRACT_BALANCE) {
        // return early to avoid unnecessary balance check
        return balance;
    }
    if (amount == ActionConstants.OPEN_DELTA) {
        // if wrapping, the open currency on the PoolManager is WETH.
        // if unwrapping, the open currency on the PoolManager is ETH.
        // note that we use the DEBT amount. Positive deltas can be taken and then wrapped.
        amount = _getFullDebt(outputCurrency);
    }
    if (amount > balance) revert InsufficientBalance();
    return amount;
}
```

获取当前合约的 `inputCurrency` 代币余额。

如果 `amount` 为 `1 << 255`，则表示使用当前合约的代币余额作为包装/解包的数量。

如果 `amount` 为 `0`，则表示使用当前合约在 `PoolManager` 上 `outputCurrency` 的欠款作为包装/解包的数量。

确保欠款数量小于 `inputCurrency` 代币余额。
