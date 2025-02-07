# v4-core

v4-core 包含 Uniswap v4 核心合约，主要包括： 

* [PoolManager.sol](./PoolManager.md)：单例合约，管理所有 Uniswap v4 池子，提供池子所有对外接口，包括创建、修改流动性、交易等操作
* Library 合约：
    * [Pool.sol](./PoolLibrary.md)：Pool Library 合约，用于具体执行每个池子的操作，比如修改流动性、交易等
    * [Position.sol](./PositionLibrary.md)：Position Library 合约，用于具体执行每个头寸的相关操作，如更新流动性和手续费等
    * [Hooks.sol](./HooksLibrary.md)：Hooks Library 合约，用于执行 Uniswap v4 合约的 Hooks 钩子函数
    * [CurrencyDelta.sol](./CurrencyDeltaLibrary.md)：CurrencyDelta Library 合约，用于执行闪电记账（Flash Accouting）相关操作
