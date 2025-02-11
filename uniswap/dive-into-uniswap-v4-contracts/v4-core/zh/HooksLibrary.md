# Hooks Library

## Hooks 地址设计

### Hooks 权限

Uniswap v4 的 Hooks 地址的低 14 位分别表示以下权限标志：

- `BEFORE_INITIALIZE_FLAG = 1 << 13`：在初始化池子前执行
- `AFTER_INITIALIZE_FLAG = 1 << 12`：在初始化池子后执行

- `BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11`：在增加流动性前执行
- `AFTER_ADD_LIQUIDITY_FLAG = 1 << 10`：在增加流动性后执行

- `BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9`：在移除流动性前执行
- `AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 8`：在移除流动性后执行

- `BEFORE_SWAP_FLAG = 1 << 7`：在交易前执行
- `AFTER_SWAP_FLAG = 1 << 6`：在交易后执行

- `BEFORE_DONATE_FLAG = 1 << 5`：在捐赠前执行
- `AFTER_DONATE_FLAG = 1 << 4`：在捐赠后执行

- `BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3`：在交易返回 delta 前执行
  - 必须开启 `BEFORE_SWAP_FLAG`
- `AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2`：在交易返回 delta 后执行
  - 必须开启 `AFTER_SWAP_FLAG`
- `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1`：在增加流动性返回 delta 后执行
  - 必须开启 `AFTER_ADD_LIQUIDITY_FLAG`
- `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0`：在移除流动性返回 delta 后执行
  - 必须开启 `AFTER_REMOVE_LIQUIDITY_FLAG`

### Hooks 地址

Hooks 的权限被设计成包含在 Hooks 地址中。通过检查 Hooks 地址特定的 bit 位，来分别判断该 Hooks 是否有特定权限。

比如 Hooks 地址为 `0x0000000000000000000000000000000000002400`，其低 14 位为 `10 0100 0000 0000`，因此该 Hooks 具有 `before initialize` 和 `after add liquidity` 权限。

### 如何生成 Hooks 地址

由于 Hooks 地址是在部署后才生成的，要确保 Hooks 地址的权限正确，一般会通过不断修改 `CREATE2` 的 `salt` 来生成 Hooks 地址，直到满足权限要求。

以下方法是计算 Hooks 地址的示例：

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

## 函数定义

### isValidHookAddress

检查 Hooks 地址的合法性。

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

检查 Hooks 地址权限的合法性：
* 如果开启了 `BEFORE_SWAP_RETURNS_DELTA_FLAG`，则必须开启 `BEFORE_SWAP_FLAG`
* 如果开启了 `AFTER_SWAP_RETURNS_DELTA_FLAG`，则必须开启 `AFTER_SWAP_FLAG`
* 如果开启了 `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG`，则必须开启 `AFTER_ADD_LIQUIDITY_FLAG`
* 如果开启了 `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG`，则必须开启 `AFTER_REMOVE_LIQUIDITY_FLAG`

检查 Hooks 地址的逻辑：
* 如果 Hooks 为 `address(0)`，即没有设置 Hooks 地址，那么池子手续费不能为动态费率，即 `fee` 不能等于 `0x800000`。
* 如果 Hooks 地址不为 `address(0)`，必须满足以下两个条件之一
  * Hooks 地址的低 14 位必须至少有一个权限标志；
  * 如果没有权限标志，那么该地址仅作为实现动态费率的合约，因此 `fee` 必须为动态费率（即 `0x800000`）。

### callHook

执行 Hooks 合约方法的调用，并统一错误处理。

其中，`self` 为 Hooks 地址，`data` 为调用数据，如 `abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96))`。

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

通过 [callHook](#callhook) 执行 Hooks 合约方法的调用，如果 `parseReturn ==  true`，则返回 delta。

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

如果 Hooks 具有 `BEFORE_INITIALIZE_FLAG` 权限，则通过 [callHook](#callhook) 执行 `beforeInitialize` 方法。

```solidity
/// @notice calls beforeInitialize hook if permissioned and validates return value
function beforeInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96) internal noSelfCall(self) {
    if (self.hasPermission(BEFORE_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96)));
    }
}
```

### afterInitialize

如果 Hooks 具有 `AFTER_INITIALIZE_FLAG` 权限，则通过 [callHook](#callhook) 执行 `afterInitialize` 方法。

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

* 如果 `liquidityDelta` 大于 0 且 Hooks 具有 `BEFORE_ADD_LIQUIDITY_FLAG` 权限，则通过 [callHook](#callhook) 执行 `beforeAddLiquidity` 方法；
* 否则，如果 `liquidityDelta` 小于等于 0 且 Hooks 具有 `BEFORE_REMOVE_LIQUIDITY_FLAG` 权限，则通过 [callHook](#callhook) 执行 `beforeRemoveLiquidity` 方法。

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

初始化 `callerDelta` 为 `delta`，并执行以下操作：

* 如果 `liquidityDelta` 大于 0
  * 如果 Hooks 具有 `AFTER_ADD_LIQUIDITY_FLAG` 权限，则通过 [callHookWithReturnDelta](#callhookandreturndelta) 执行 `afterAddLiquidity` 方法；
    * 如果 Hooks 具有 `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG` 权限，则解析 `hookDelta`。
    * 将 callerDelta 扣除 hookDelta。
* 如果 `liquidityDelta` 小于等于 0
  * 如果 Hooks 具有 `AFTER_REMOVE_LIQUIDITY_FLAG` 权限，则通过 [callHookWithReturnDelta](#callhookandreturndelta) 执行 `afterRemoveLiquidity` 方法；
    * 如果 Hooks 具有 `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG` 权限，则解析 `hookDelta`。
    * 将 callerDelta 扣除 hookDelta。

在这个过程中，Hooks 合约可以通过返回 `hookDelta` 来修改 `callerDelta`，从而影响流动性的变化。

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

* 如果 Hooks 具有 `BEFORE_SWAP_FLAG` 权限，则通过 [callHook](#callhook) 执行 `beforeSwap` 方法；
  * 如果池子支持动态费率，Hooks 可以通过返回 `lpFeeOverride` 来覆盖当前的 LP 费率；
  * 如果 Hooks 具有 `BEFORE_SWAP_RETURNS_DELTA_FLAG` 权限，则解析 `hookReturn`，并根据 `hookReturn`（高 128 位）来修改 `amountToSwap`。
  * 注意：`hookReturn` 的高 128 位表示 `hookDeltaSpecified`，低 128 位表示 `hookDeltaUnspecified`。

    根据 `params.amountSpecified` 和 `params.zeroForOne`，`hookDeltaSpecified` 和 `hookDeltaUnspecified` 可能代表 `amount0` 或 `amount1`，有如下组合：

    | `params.amountSpecified < 0` | `params.zeroForOne` | `hookDeltaSpecified` | `hookDeltaUnspecified` |
    | --- | --- | --- | --- |
    | `true` | `true` | `amount0` | `amount1` |
    | `true` | `false` | `amount1` | `amount0` |
    | `false` | `true` | `amount1` | `amount0` |
    | `false` | `false` | `amount0` | `amount1` |

  * `beforeSwap` 的返回值会影响（并修改） `amountToSwap` 的值，但不会影响交易类型（exact input/output）。

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

* 如果 Hooks 具有 `AFTER_SWAP_FLAG` 权限，则通过 [callHookWithReturnDelta](#callhookandreturndelta) 执行 `afterSwap` 方法；
  * 如果 Hooks 具有 `AFTER_SWAP_RETURNS_DELTA_FLAG` 权限，则解析 `hookDelta`
  * 注意：`afterSwap` 仅返回 `hookDeltaUnspecified`；而 `beforeSwap` 返回的是 `hookDeltaSpecified` 和 `hookDeltaUnspecified`。
    * `beofreSwap` 影响输入值 `amountToSwap`，而 `afterSwap` 影响输出值 `hookDeltaUnspecified`。
* 将 `hookDelta` 从 `swapDelta` 中扣除。

根据 `params.amountSpecified` 和 `params.zeroForOne`，判断 `hookDeltaSpecified` 和 `hookDeltaUnspecified` 的顺序，有如下组合：

| `params.amountSpecified < 0` | `params.zeroForOne` | `hookDeltaSpecified` | `hookDeltaUnspecified` |
| --- | --- | --- | --- |
| `true` | `true` | `amount0` | `amount1` |
| `true` | `false` | `amount1` | `amount0` |
| `false` | `true` | `amount1` | `amount0` |
| `false` | `false` | `amount0` | `amount1` |

因此，当 `params.amountSpecified < 0 == params.zeroForOne` 时，`hookDeltaSpecified` 总表示 `amount0`，`hookDeltaUnspecified` 总表示 `amount1`。

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

如果 Hooks 具有 `BEFORE_DONATE_FLAG` 权限，则通过 [callHook](#callhook) 执行 `beforeDonate` 方法。

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

如果 Hooks 具有 `AFTER_DONATE_FLAG` 权限，则通过 [callHook](#callhook) 执行 `afterDonate` 方法。

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

判断 Hooks 地址是否具有特定权限，即检查指定的 bit 是否为 1。

```solidity
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;
}
```
