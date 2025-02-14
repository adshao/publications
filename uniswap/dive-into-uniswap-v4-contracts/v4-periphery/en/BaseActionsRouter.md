# BaseActionsRouter

## Method Definitions

### _executeActions

Batch execute actions.

```solidity
/// @notice internal function that triggers the execution of a set of actions on v4
/// @dev inheriting contracts should call this function to trigger execution
function _executeActions(bytes calldata unlockData) internal {
    poolManager.unlock(unlockData);
}
```

Call the [PoolManager.unlock](../../v4-core/en/PoolManager.md#unlock) method, which performs the unlock operation and calls back the [_unlockCallback](#_unlockcallback) method.

### _unlockCallback

Callback method, called by the `PoolManager.unlock` method to execute specific actions.

The input parameter `data` is passed when calling the `PoolManager.unlock` method.

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

Decode `data` through `data.decodeActionsRouterParams()` to get `actions` and `params`.

The format of `data` is `abi.encode(bytes actions, bytes[] params)` calculated `bytes`, where `params[i]` is the parameter of `actions[i]`.

* The format of `actions` is `abi.encodePacked(uint8 action1, uint8 action2, ...)` calculated `bytes`, each byte represents an `action`, and all `actions` are defined in [ActionsLibrary](./ActionsLibrary.md).
* `params` is an array, where the format of `params[i]` is `abi.encode(v1, v2, ...)` calculated `bytes`.

Call the [_executeActionsWithoutUnlock](#_executeactionswithoutunlock) method to execute specific actions.

### _executeActionsWithoutUnlock

Iterate through all actions and corresponding parameters, and execute each action in order.

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

Ensure that the lengths of `actions` and `params` are consistent, then execute each action in order through `_handleAction`. The `_handleAction` method is implemented by contracts that inherit `BaseActionsRouter`, such as [PositionManager._handleAction](./PositionManager.md#_handleaction) and [V4Router._handleAction](./V4Router.md#_handleaction).

### _mapRecipient

The `_mapRecipient` method is used to calculate the `recipient` address:

* If `recipient` is `address(1)`, i.e., `ActionConstants.MSG_SENDER`, it represents the `msgSender()` caller address.
* Otherwise, if `recipient` is `address(2)`, i.e., `ActionConstants.ADDRESS_THIS`, it represents the current contract address, such as `PositionManager`.
* Otherwise, use the `recipient` address itself.

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

The `_mapPayer` method determines the payer based on `payerIsUser`:

* If `payerIsUser` is `true`, the payer is `msgSender()`, i.e., the caller.
* Otherwise, the payer is the current contract address, such as `PositionManager`.

```solidity
/// @notice Calculates the payer for an action
function _mapPayer(bool payerIsUser) internal view returns (address) {
    return payerIsUser ? msgSender() : address(this);
}
```
