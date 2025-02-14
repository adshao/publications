# v4-periphery

v4-periphery [[Github](https://github.com/Uniswap/v4-periphery/)]：包含 Uniswap v4 外围合约，主要包括：

* [PositionManager.sol](./v4-periphery/zh/PositionManager.md)：PositionManager 合约，用于管理头寸的创建、销毁、修改流动性等操作，底层调用 [PoolManager](./PoolManager.md) 执行具体操作。
    * 外部合约通过 PositionManager 合约来操作头寸，而不是直接调用 v4-core 的 PoolManager 合约
    * 支持将多个操作组合成一个事务，保证事务的原子性，同时减少 gas 消耗

* [V4Router.sol](./v4-periphery/zh/V4Router.md)：V4Router 合约，用于执行交易操作，调用 [PoolManager](./PoolManager.md) 合约来执行具体的交易操作
    * 支持单跳和多跳交易
    * 支持指定输入或输出代币数量
