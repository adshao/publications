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
