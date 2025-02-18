[English](./README.md) | [中文](./README_zh.md)

# Deep Dive into Uniswap v4 Smart Contracts

###### tags: `uniswap` `uniswap-v4` `smart contract` `solidity`

# Workflow

The following diagram shows the workflow of Uniswap v4 contracts:

![](./assets/uniswap-v4-workflow.png)

Similar to Uniswap v2/v3, Uniswap v4 contracts are also divided into two repositories:

* [v4-core](./v4-core/en/README.md) [[Github](https://github.com/Uniswap/v4-core)]: Contains the core contracts of Uniswap v4, mainly including:
    * [PoolManager.sol](./v4-core/en/PoolManager.md): Singleton contract that manages all Uniswap v4 pools, providing all external interfaces for pools, including creation, liquidity modification, swap, etc.
    * Library contracts:
        * [Pool.sol](./v4-core/en/PoolLibrary.md): Pool Library contract, used to execute specific operations for each pool, such as liquidity modification, swap, etc.
        * [Position.sol](./v4-core/en/PositionLibrary.md): Position Library contract, used to execute specific operations related to each position, such as modifying liquidity and fees.
        * [Hooks.sol](./v4-core/en/HooksLibrary.md): Hooks Library contract, used to execute the Hooks functions of Uniswap v4 contracts.
        * [CurrencyDelta.sol](./v4-core/en/CurrencyDeltaLibrary.md): CurrencyDelta Library contract, used to execute operations related to Flash Accounting.
        * [BalanceDelta.sol](./v4-core/en/BalanceDelta.md)：BalanceDelta defines the type of balance delta.

* [v4-periphery](./v4-periphery/en/README.md) [[Github](https://github.com/Uniswap/v4-periphery/)]: Contains the peripheral contracts of Uniswap v4, mainly including:
    * [PositionManager.sol](./v4-periphery/en/PositionManager.md): PositionManager contract, used to manage the minting, burning, and liquidity modification of positions, calling [PoolManager](./v4-core/en/PoolManager.md) to execute specific operations.
        * External contracts operate positions through the PositionManager contract, rather than directly calling the v4-core PoolManager contract.
        * Supports combining multiple operations into a single transaction, ensuring atomicity of transactions and reducing gas consumption.
    * [V4Router.sol](./v4-periphery/en/V4Router.md): V4Router contract, used to execute trading operations, calling [PoolManager](./v4-core/en/PoolManager.md) to execute specific trading operations.
        * Supports single-hop and multi-hop trading.
        * Supports specifying the exact input or output token amount.
