# Hooks Library

## Hooks Address Design

### Hooks Permissions

Uniswap v4 uses the lower 14 bits of the Hooks address to represent the following permissions:

- `BEFORE_INITIALIZE_FLAG = 1 << 13`: Execute before initializing the pool
- `AFTER_INITIALIZE_FLAG = 1 << 12`: Execute after initializing the pool

- `BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11`: Execute before adding liquidity
- `AFTER_ADD_LIQUIDITY_FLAG = 1 << 10`: Execute after adding liquidity

- `BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9`: Execute before removing liquidity
- `AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 8`: Execute after removing liquidity

- `BEFORE_SWAP_FLAG = 1 << 7`: Execute before swapping
- `AFTER_SWAP_FLAG = 1 << 6`: Execute after swapping

- `BEFORE_DONATE_FLAG = 1 << 5`: Execute before donating
- `AFTER_DONATE_FLAG = 1 << 4`: Execute after donating

- `BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3`: Execute before returning delta in swap
  - Must enable `BEFORE_SWAP_FLAG`
- `AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2`: Execute after returning delta in swap
  - Must enable `AFTER_SWAP_FLAG`
- `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1`: Execute after returning delta in add liquidity
  - Must enable `AFTER_ADD_LIQUIDITY_FLAG`
- `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0`: Execute after returning delta in remove liquidity
  - Must enable `AFTER_REMOVE_LIQUIDITY_FLAG`

### Hooks Address

The permissions of Hooks are designed to be included in their address. By checking specific bits of the Hooks address, it can be determined whether specific permissions are present.

For example, if the Hooks address is `0x0000000000000000000000000000000000002400`, the lower 14 bits are `10 0100 0000 0000`, so the Hooks have `before initialize` and `after add liquidity` permissions.

### How to Generate Hooks Address

To ensure that the Hooks address meets the preset permissions, you can try generating the Hooks address by continuously modifying the `CREATE2` `salt` value until the permission requirements are met.

The following method is an example of calculating the Hooks address:

```solidity
/// @notice Precompute a contract address deployed via CREATE2
/// @param deployer The address that will deploy the hook. In `forge test`, this will be the test contract `address(this)` or the pranking address
///                 In `forge script`, this should be `0x4e59b44847b379578588920cA78FbF26c0B4956C` (CREATE2 Deployer Proxy)
/// @param salt The salt used to deploy the hook
/// @param creationCode The creation code of a hook contract
function computeAddress(address deployer, uint256 salt, bytes memory creationCode)
    internal
    pure
    returns (address hookAddress)
{
    return address(
        uint160(uint256(keccak256(abi.encodePacked(bytes1(0xFF), deployer, salt, keccak256(creationCode)))))
    );
}
```

## Function Definitions

### isValidHookAddress

Check the validity of the Hooks address.

```solidity
/// @notice Ensures that the hook address includes at least one hook flag or dynamic fees, or is the 0 address
/// @param self The hook to verify
/// @param fee The fee of the pool the hook is used with
/// @return bool True if the hook address is valid
function isValidHookAddress(IHooks self, uint24 fee) internal pure returns (bool) {
    // The hook can only have a flag to return a hook delta on an action if it also has the corresponding action flag
    if (!self.hasPermission(BEFORE_SWAP_FLAG) && self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_SWAP_FLAG) && self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG) && self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG))
    {
        return false;
    }
    if (
        !self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)
            && self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
    ) return false;

    // If there is no hook contract set, then fee cannot be dynamic
    // If a hook contract is set, it must have at least 1 flag set, or have a dynamic fee
    return address(self) == address(0)
        ? !fee.isDynamicFee()
        : (uint160(address(self)) & ALL_HOOK_MASK > 0 || fee.isDynamicFee());
}
```

Check whether the permissions of the Hooks address are correct:
* If `BEFORE_SWAP_RETURNS_DELTA_FLAG` is enabled, `BEFORE_SWAP_FLAG` must be enabled
* If `AFTER_SWAP_RETURNS_DELTA_FLAG` is enabled, `AFTER_SWAP_FLAG` must be enabled
* If `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG` is enabled, `AFTER_ADD_LIQUIDITY_FLAG` must be enabled
* If `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG` is enabled, `AFTER_REMOVE_LIQUIDITY_FLAG` must be enabled

Check the logic of the Hooks address:
* If the Hooks is `address(0)`, that is, no Hooks address is set, then the pool fee cannot be dynamic, that is, `fee` cannot be equal to `0x800000`.
* If the Hooks address is not `address(0)`, it must meet one of the following two conditions:
  * The lower 14 bits of the Hooks address must have at least one permission flag;
  * If there is no permission flag, then the address is only used as a contract to implement dynamic fees, so `fee` must be a dynamic fee (that is, `0x800000`).

### callHook

Execute the Hooks contract method and handle errors uniformly.

Among them, `self` is the Hooks address, and `data` is the call data, such as `abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96))`.

```solidity
/// @notice performs a hook call using the given calldata on the given hook that doesn't return a delta
/// @return result The complete data returned by the hook
function callHook(IHooks self, bytes memory data) internal returns (bytes memory result) {
    bool success;
    assembly ("memory-safe") {
        success := call(gas(), self, 0, add(data, 0x20), mload(data), 0, 0)
    }
    // Revert with FailedHookCall, containing any error message to bubble up
    if (!success) CustomRevert.bubbleUpAndRevertWith(address(self), bytes4(data), HookCallFailed.selector);

    // The call was successful, fetch the returned data
    assembly ("memory-safe") {
        // allocate result byte array from the free memory pointer
        result := mload(0x40)
        // store new free memory pointer at the end of the array padded to 32 bytes
        mstore(0x40, add(result, and(add(returndatasize(), 0x3f), not(0x1f))))
        // store length in memory
        mstore(result, returndatasize())
        // copy return data to result
        returndatacopy(add(result, 0x20), 0, returndatasize())
    }

    // Length must be at least 32 to contain the selector. Check expected selector and returned selector match.
    if (result.length < 32 || result.parseSelector() != data.parseSelector()) {
        InvalidHookResponse.selector.revertWith();
    }
}
```

### callHookAndReturnDelta

Execute the Hooks contract method through [callHook](#callhook), and return delta if `parseReturn == true`.

```solidity
/// @notice performs a hook call using the given calldata on the given hook
/// @return int256 The delta returned by the hook
function callHookWithReturnDelta(IHooks self, bytes memory data, bool parseReturn) internal returns (int256) {
    bytes memory result = callHook(self, data);

    // If this hook wasn't meant to return something, default to 0 delta
    if (!parseReturn) return 0;

    // A length of 64 bytes is required to return a bytes4, and a 32 byte delta
    if (result.length != 64) InvalidHookResponse.selector.revertWith();
    return result.parseReturnDelta();
}
```

### beforeInitialize

If the Hooks have `BEFORE_INITIALIZE_FLAG` permission, execute the `beforeInitialize` method through [callHook](#callhook).

```solidity
/// @notice calls beforeInitialize hook if permissioned and validates return value
function beforeInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96) internal noSelfCall(self) {
    if (self.hasPermission(BEFORE_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96)));
    }
}
```

### afterInitialize

If the Hooks have `AFTER_INITIALIZE_FLAG` permission, execute the `afterInitialize` method through [callHook](#callhook).

```solidity
/// @notice calls afterInitialize hook if permissioned and validates return value
function afterInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96, int24 tick)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(AFTER_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.afterInitialize, (msg.sender, key, sqrtPriceX96, tick)));
    }
}
```

### beforeModifyLiquidity

* If `liquidityDelta` is greater than 0 and the Hooks have `BEFORE_ADD_LIQUIDITY_FLAG` permission, execute the `beforeAddLiquidity` method through [callHook](#callhook);
* Otherwise, if `liquidityDelta` is less than or equal to 0 and the Hooks have `BEFORE_REMOVE_LIQUIDITY_FLAG` permission, execute the `beforeRemoveLiquidity` method through [callHook](#callhook).

```solidity
/// @notice calls beforeModifyLiquidity hook if permissioned and validates return value
function beforeModifyLiquidity(
    IHooks self,
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    bytes calldata hookData
) internal noSelfCall(self) {
    if (params.liquidityDelta > 0 && self.hasPermission(BEFORE_ADD_LIQUIDITY_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeAddLiquidity, (msg.sender, key, params, hookData)));
    } else if (params.liquidityDelta <= 0 && self.hasPermission(BEFORE_REMOVE_LIQUIDITY_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeRemoveLiquidity, (msg.sender, key, params, hookData)));
    }
}
```

### afterModifyLiquidity

Initialize `callerDelta` as `delta`, and perform the following operations:

* If `liquidityDelta` is greater than 0
  * If the Hooks have `AFTER_ADD_LIQUIDITY_FLAG` permission, execute the `afterAddLiquidity` method through [callHookWithReturnDelta](#callhookandreturndelta);
    * If the Hooks have `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG` permission, parse `hookDelta`.
    * Subtract `hookDelta` from `callerDelta`.
* If `liquidityDelta` is less than or equal to 0
  * If the Hooks have `AFTER_REMOVE_LIQUIDITY_FLAG` permission, execute the `afterRemoveLiquidity` method through [callHookWithReturnDelta](#callhookandreturndelta);
    * If the Hooks have `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG` permission, parse `hookDelta`.
    * Subtract `hookDelta` from `callerDelta`.

During this process, the Hooks contract can modify `callerDelta` by returning `hookDelta`, thereby affecting the change in liquidity.

```solidity
/// @notice calls afterModifyLiquidity hook if permissioned and validates return value
function afterModifyLiquidity(
    IHooks self,
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    BalanceDelta delta,
    BalanceDelta feesAccrued,
    bytes calldata hookData
) internal returns (BalanceDelta callerDelta, BalanceDelta hookDelta) {
    if (msg.sender == address(self)) return (delta, BalanceDeltaLibrary.ZERO_DELTA);

    callerDelta = delta;
    if (params.liquidityDelta > 0) {
        if (self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG)) {
            hookDelta = BalanceDelta.wrap(
                self.callHookWithReturnDelta(
                    abi.encodeCall(
                        IHooks.afterAddLiquidity, (msg.sender, key, params, delta, feesAccrued, hookData)
                    ),
                    self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG)
                )
            );
            callerDelta = callerDelta - hookDelta;
        }
    } else {
        if (self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)) {
            hookDelta = BalanceDelta.wrap(
                self.callHookWithReturnDelta(
                    abi.encodeCall(
                        IHooks.afterRemoveLiquidity, (msg.sender, key, params, delta, feesAccrued, hookData)
                    ),
                    self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
                )
            );
            callerDelta = callerDelta - hookDelta;
        }
    }
}
```

### beforeSwap

* If the Hooks have `BEFORE_SWAP_FLAG` permission, execute the `beforeSwap` method through [callHook](#callhook);
  * If the pool supports dynamic fees, the Hooks can override the current LP fee by returning `lpFeeOverride`;
  * If the Hooks have `BEFORE_SWAP_RETURNS_DELTA_FLAG` permission, parse `hookReturn`, and modify `amountToSwap` based on `hookReturn` (upper 128 bits).
    > Note: The upper 128 bits of `hookReturn` represent `hookDeltaSpecified`, and the lower 128 bits represent `hookDeltaUnspecified`.

    Depending on `params.amountSpecified` and `params.zeroForOne`, `hookDeltaSpecified` and `hookDeltaUnspecified` may represent `amount0` or `amount1`, with the following combinations:

    | `params.amountSpecified < 0` | `params.zeroForOne` | `hookDeltaSpecified` | `hookDeltaUnspecified` |
    | --- | --- | --- | --- |
    | `true` | `true` | `amount0` | `amount1` |
    | `true` | `false` | `amount1` | `amount0` |
    | `false` | `true` | `amount1` | `amount0` |
    | `false` | `false` | `amount0` | `amount1` |

  * The return value of `beforeSwap` affects (and modifies) the value of `amountToSwap`, but does not affect the swap type (exact input/output).

```solidity
/// @notice calls beforeSwap hook if permissioned and validates return value
function beforeSwap(IHooks self, PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    internal
    returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride)
{
    amountToSwap = params.amountSpecified;
    if (msg.sender == address(self)) return (amountToSwap, BeforeSwapDeltaLibrary.ZERO_DELTA, lpFeeOverride);

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        bytes memory result = callHook(self, abi.encodeCall(IHooks.beforeSwap, (msg.sender, key, params, hookData)));

        // A length of 96 bytes is required to return a bytes4, a 32 byte delta, and an LP fee
        if (result.length != 96) InvalidHookResponse.selector.revertWith();

        // dynamic fee pools that want to override the cache fee, return a valid fee with the override flag. If override flag
        // is set but an invalid fee is returned, the transaction will revert. Otherwise the current LP fee will be used
        if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();

        // skip this logic for the case where the hook return is 0
        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());

            // any return in unspecified is passed to the afterSwap hook for handling
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();

            // Update the swap amount according to the hook's return, and check that the swap type doesn't change (exact input/output)
            if (hookDeltaSpecified != 0) {
                bool exactInput = amountToSwap < 0;
                amountToSwap += hookDeltaSpecified;
                if (exactInput ? amountToSwap > 0 : amountToSwap < 0) {
                    HookDeltaExceedsSwapAmount.selector.revertWith();
                }
            }
        }
    }
}
```

### afterSwap

* If the Hooks have `AFTER_SWAP_FLAG` permission, execute the `afterSwap` method through [callHookWithReturnDelta](#callhookandreturndelta);
  * If the Hooks have `AFTER_SWAP_RETURNS_DELTA_FLAG` permission, parse `hookDelta`
  * Note: `afterSwap` only returns `hookDeltaUnspecified`; while `beforeSwap` returns both `hookDeltaSpecified` and `hookDeltaUnspecified`.
    * `beforeSwap` affects the input value `amountToSwap`, while `afterSwap` affects the output value `hookDeltaUnspecified`.
* Subtract `hookDelta` from `swapDelta`.

Depending on `params.amountSpecified` and `params.zeroForOne`, determine the order of `hookDeltaSpecified` and `hookDeltaUnspecified`, with the following combinations:

| `params.amountSpecified < 0` | `params.zeroForOne` | `hookDeltaSpecified` | `hookDeltaUnspecified` |
| --- | --- | --- | --- |
| `true` | `true` | `amount0` | `amount1` |
| `true` | `false` | `amount1` | `amount0` |
| `false` | `true` | `amount1` | `amount0` |
| `false` | `false` | `amount0` | `amount1` |

Therefore, when `params.amountSpecified < 0 == params.zeroForOne`, `hookDeltaSpecified` always represents `amount0`, and `hookDeltaUnspecified` always represents `amount1`.

```solidity
/// @notice calls afterSwap hook if permissioned and validates return value
function afterSwap(
    IHooks self,
    PoolKey memory key,
    IPoolManager.SwapParams memory params,
    BalanceDelta swapDelta,
    bytes calldata hookData,
    BeforeSwapDelta beforeSwapHookReturn
) internal returns (BalanceDelta, BalanceDelta) {
    if (msg.sender == address(self)) return (swapDelta, BalanceDeltaLibrary.ZERO_DELTA);

    int128 hookDeltaSpecified = beforeSwapHookReturn.getSpecifiedDelta();
    int128 hookDeltaUnspecified = beforeSwapHookReturn.getUnspecifiedDelta();

    if (self.hasPermission(AFTER_SWAP_FLAG)) {
        hookDeltaUnspecified += self.callHookWithReturnDelta(
            abi.encodeCall(IHooks.afterSwap, (msg.sender, key, params, swapDelta, hookData)),
            self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)
        ).toInt128();
    }

    BalanceDelta hookDelta;
    if (hookDeltaUnspecified != 0 || hookDeltaSpecified != 0) {
        hookDelta = (params.amountSpecified < 0 == params.zeroForOne)
            ? toBalanceDelta(hookDeltaSpecified, hookDeltaUnspecified)
            : toBalanceDelta(hookDeltaUnspecified, hookDeltaSpecified);

        // the caller has to pay for (or receive) the hook's delta
        swapDelta = swapDelta - hookDelta;
    }
    return (swapDelta, hookDelta);
}
```

### beforeDonate

If the Hooks have `BEFORE_DONATE_FLAG` permission, execute the `beforeDonate` method through [callHook](#callhook).

```solidity
/// @notice calls beforeDonate hook if permissioned and validates return value
function beforeDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(BEFORE_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}
```

### afterDonate

If the Hooks have `AFTER_DONATE_FLAG` permission, execute the `afterDonate` method through [callHook](#callhook).

```solidity
/// @notice calls afterDonate hook if permissioned and validates return value
function afterDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(AFTER_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.afterDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}
```

### hasPermission

Determine whether the Hooks address has specific permissions, that is, check whether the specified bit is 1.

```solidity
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;
}
```
