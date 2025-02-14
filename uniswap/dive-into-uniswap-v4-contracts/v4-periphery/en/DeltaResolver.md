# DeltaResolver

DeltaResolver is an abstract contract used to sync, transfer tokens, and settle funds with the `PoolManager` contract.

## Method Definitions

### _pay

DeltaResolver defines an abstract method `_pay`, which implements paying a specified amount of tokens to the `poolManager`. All contracts inheriting DeltaResolver need to implement the `_pay` method.

For example, the [PositionManager](./PositionManager.md) contract inherits the DeltaResolver contract and implements the [_pay](./PositionManager.md#_pay) method.

Input parameters:

- `Currency token`: The token to be paid
- `address payer`: The address of the payer
- `uint256 amount`: The amount to be paid

```solidity
/// @notice Abstract function for contracts to implement paying tokens to the poolManager
/// @dev The recipient of the payment should be the poolManager
/// @param token The token to settle. This is known not to be the native currency
/// @param payer The address who should pay tokens
/// @param amount The number of tokens to send
function _pay(Currency token, address payer, uint256 amount) internal virtual;
```

### _getFullDebt

Get the full debt (negative delta) of the current contract in `PoolManager`. The return value is of type `uint256` after negation.

Input parameters:

- `Currency currency`: The token address

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

Get the full credit (positive delta) of the current contract in `PoolManager`. The return value is of type `uint256`.

Input parameters:

- `Currency currency`: The token address

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

Withdraw the balance of a single token. The `PoolManager` contract transfers the tokens to the `recipient`.

Input parameters:

- `currency`: The token
- `recipient`: The recipient
- `amount`: The amount to be withdrawn

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

If `amount` is 0, return directly.

Call the [PoolManager.take](../../v4-core/en/PoolManager.md#take) method to withdraw the specified amount of tokens from the `poolManager`.

### _settle

Settle the debt of a single token. Tokens need to be paid to the `PoolManager`.

In the [PoolManager.settle](../../v4-core/en/PoolManager.md#settle) method, we introduced the process of settling debts:

1. Call the [PoolManager.sync](../../v4-core/en/PoolManager.md#sync) method to sync the token balance;
2. Transfer tokens to the `PoolManager`;
3. Call the [PoolManager.settle](../../v4-core/en/PoolManager.md#settle) method to settle the accounting balance.

Input parameters:

- `currency`: The token
- `payer`: The payer
- `amount`: The amount to be paid

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

If `amount` is 0, return directly.

Call the [poolManager.sync](../../v4-core/en/PoolManager.md#sync) method to sync the token balance in the `poolManager`.

If `currency` is `ADDRESS_ZERO`, i.e., native ETH, transfer ETH to the `poolManager` through `{value: amount}` and call the [PoolManager.settle](../../v4-core/en/PoolManager.md#settle) method to settle the accounting balance.

Otherwise, for ERC20 tokens, call the [_pay](#_pay) method to transfer tokens to the `poolManager`, and then call the [PoolManager.settle](../../v4-core/en/PoolManager.md#settle) method to settle the accounting balance.

### _mapSettleAmount

The `_mapSettleAmount` method calculates the amount of tokens to be settled based on `amount` and `currency`:
* If `amount` is `1 << 255`, i.e., `ActionConstants.CONTRACT_BALANCE`, it means using the current contract's token balance for settlement.
* Otherwise, if `amount` is `0`, i.e., `ActionConstants.OPEN_DELTA`, it means settling all debts.
* Otherwise, it means settling the specified `amount` of tokens.

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

The `_mapTakeAmount` method calculates the amount of tokens to be withdrawn based on `amount` and `currency`:
* If `amount` is `0`, it means withdrawing all the current contract's withdrawable balance in `PoolManager`.
* Otherwise, it means withdrawing the specified `amount` of tokens.

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

Calculate the amount of tokens to be wrapped/unwrapped.

Input parameters:

- `inputCurrency`: The input token, which can be a native token or a wrapped token
- `amount`: The amount to be wrapped/unwrapped, which can be `CONTRACT_BALANCE`, `OPEN_DELTA`, or a specific amount
- `outputCurrency`: The token after wrapping/unwrapping, which the user may owe in `PoolManager`

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

Get the `inputCurrency` token balance of the current contract.

If `amount` is `1 << 255`, it means using the current contract's token balance as the amount for wrapping/unwrapping.

If `amount` is `0`, it means using the current contract's debt in `PoolManager` for `outputCurrency` as the amount for wrapping/unwrapping.

Ensure that the debt amount is less than the `inputCurrency` token balance.
