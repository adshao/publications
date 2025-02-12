# Actions Library

Actions Library 定义了 Uniswap v4 periphery 合约支持的所有操作。

```solidity
// pool actions
// liquidity actions
uint256 internal constant INCREASE_LIQUIDITY = 0x00;
uint256 internal constant DECREASE_LIQUIDITY = 0x01;
uint256 internal constant MINT_POSITION = 0x02;
uint256 internal constant BURN_POSITION = 0x03;
uint256 internal constant INCREASE_LIQUIDITY_FROM_DELTAS = 0x04;
uint256 internal constant MINT_POSITION_FROM_DELTAS = 0x05;

// swapping
uint256 internal constant SWAP_EXACT_IN_SINGLE = 0x06;
uint256 internal constant SWAP_EXACT_IN = 0x07;
uint256 internal constant SWAP_EXACT_OUT_SINGLE = 0x08;
uint256 internal constant SWAP_EXACT_OUT = 0x09;

// donate
// note this is not supported in the position manager or router
uint256 internal constant DONATE = 0x0a;

// closing deltas on the pool manager
// settling
uint256 internal constant SETTLE = 0x0b;
uint256 internal constant SETTLE_ALL = 0x0c;
uint256 internal constant SETTLE_PAIR = 0x0d;
// taking
uint256 internal constant TAKE = 0x0e;
uint256 internal constant TAKE_ALL = 0x0f;
uint256 internal constant TAKE_PORTION = 0x10;
uint256 internal constant TAKE_PAIR = 0x11;

uint256 internal constant CLOSE_CURRENCY = 0x12;
uint256 internal constant CLEAR_OR_TAKE = 0x13;
uint256 internal constant SWEEP = 0x14;

uint256 internal constant WRAP = 0x15;
uint256 internal constant UNWRAP = 0x16;

// minting/burning 6909s to close deltas
// note this is not supported in the position manager or router
uint256 internal constant MINT_6909 = 0x17;
uint256 internal constant BURN_6909 = 0x18;
```

- 流动性相关操作：
    - INCREASE_LIQUIDITY：增加流动性
    - DECREASE_LIQUIDITY：减少流动性
    - MINT_POSITION：铸造头寸
    - BURN_POSITION：销毁头寸
    - INCREASE_LIQUIDITY_FROM_DELTAS：根据余额增加流动性
    - MINT_POSITION_FROM_DELTAS：根据余额铸造头寸

- 交换相关操作：
    - SWAP_EXACT_IN_SINGLE：精确输入单个资产
    - SWAP_EXACT_IN：精确输入
    - SWAP_EXACT_OUT_SINGLE：精确输出单个资产
    - SWAP_EXACT_OUT：精确输出

- 捐赠操作：
    - DONATE：捐赠

- 结算操作：
    - SETTLE：结算单个代币
    - SETTLE_ALL：结算所有代币
    - SETTLE_PAIR：结算交易对
    - TAKE：提取单个代币
    - TAKE_ALL：提取所有代币
    - TAKE_PORTION：提取部分代币
    - TAKE_PAIR：提取交易对
    - CLOSE_CURRENCY：结算或提取单个代币
    - CLEAR_OR_TAKE：放弃或提取单个代币
    - SWEEP：提取所有代币

- 包装操作：
    - WRAP：包装
    - UNWRAP：解包

- 铸造/销毁 ERC6909 token 操作：
    - MINT_6909：铸造 ERC6909
    - BURN_6909：销毁 ERC6909
