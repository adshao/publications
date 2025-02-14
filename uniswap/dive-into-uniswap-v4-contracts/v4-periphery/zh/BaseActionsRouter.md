# BaseActionsRouter

## 方法定义

### _executeActions

批量执行操作。

```solidity
/// @notice internal function that triggers the execution of a set of actions on v4
/// @dev inheriting contracts should call this function to trigger execution
function _executeActions(bytes calldata unlockData) internal {
    poolManager.unlock(unlockData);
}
```

调用 [PoolManager.unlock](../../v4-core/zh/PoolManager.md#unlock) 方法，在 `PoolManager.unlock` 中执行解锁操作，回调 [_unlockCallback](#_unlockcallback) 方法。

### _unlockCallback

回调方法，被 `PoolManager.unlock` 方法调用，执行具体的操作。

输入参数 `data` 是在调用 `PoolManager.unlock` 方法时传入的。

```solidity
/// @notice function that is called by the PoolManager through the SafeCallback.unlockCallback
/// @param data Abi encoding of (bytes actions, bytes[] params)
/// where params[i] is the encoded parameters for actions[i]
function _unlockCallback(bytes calldata data) internal override returns (bytes memory) {
    // abi.decode(data, (bytes, bytes[]));
    (bytes calldata actions, bytes[] calldata params) = data.decodeActionsRouterParams();
    _executeActionsWithoutUnlock(actions, params);
    return "";
}
```

通过 `data.decodeActionsRouterParams()` 解码 `data`，获取 `actions` 和 `params`。

`data` 的格式为 `abi.encode(bytes actions, bytes[] params)` 计算的 `bytes`，其中 `params[i]` 是 `actions[i]` 的参数。

* `actions` 的格式为 `abi.encodePacked(uint8 action1, uint8 action2, ...)` 计算的 `bytes`，每个字节表示一个 `action`，[ActionsLibrary](./ActionsLibrary.md) 中定义了所有的 `action`。
* `params` 是一个数组，其中 `params[i]` 的格式为 `abi.encode(v1, v2, ...)` 计算的 `bytes`。

调用 [_executeActionsWithoutUnlock](#_executeactionswithoutunlock) 方法，执行具体的操作。

### _executeActionsWithoutUnlock

遍历所有操作和对应的参数，顺序执行每个操作。

```solidity
function _executeActionsWithoutUnlock(bytes calldata actions, bytes[] calldata params) internal {
    uint256 numActions = actions.length;
    if (numActions != params.length) revert InputLengthMismatch();

    for (uint256 actionIndex = 0; actionIndex < numActions; actionIndex++) {
        uint256 action = uint8(actions[actionIndex]);

        _handleAction(action, params[actionIndex]);
    }
}
```

确认 `actions` 和 `params` 的长度一致，然后通过 `_handleAction` 顺序执行每个操作。`_handleAction` 方法由继承 `BaseActionsRouter` 的合约实现，如 [PositionManager._handleAction](./PositionManager.md#_handleaction) 和 [V4Router._handleAction](./V4Router.md#_handleaction)。

### _mapRecipient

`_mapRecipient` 方法用于计算 `recipient` 地址：

* 如果 `recipient` 为 `address(1)`，即 `ActionConstants.MSG_SENDER`，则表示 `msgSender()` 调用方地址
* 否则，如果 `recipient` 为 `address(2）`，即 `ActionConstants.ADDRESS_THIS`，则表示当前合约地址，比如 `PositionManager`
* 否则，使用 `recipient` 地址本身

```solidity
/// @notice Calculates the address for a action
function _mapRecipient(address recipient) internal view returns (address) {
    if (recipient == ActionConstants.MSG_SENDER) {
        return msgSender();
    } else if (recipient == ActionConstants.ADDRESS_THIS) {
        return address(this);
    } else {
        return recipient;
    }
}
```

### _mapPayer

`_mapPayer` 方法根据 `payerIsUser` 来确定支付方：

* 如果 `payerIsUser` 为 `true`，则支付方为 `msgSender()`，即调用方
* 否则，支付方为当前合约地址，比如 `PositionManager`

```solidity
/// @notice Calculates the payer for an action
function _mapPayer(bool payerIsUser) internal view returns (address) {
    return payerIsUser ? msgSender() : address(this);
}
```
