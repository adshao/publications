# v4-core

v4-core [[Github](https://github.com/Uniswap/v4-core)]: Contains Uniswap v4 core contracts, mainly including:

* [PoolManager.sol](./PoolManager.md): Singleton contract that manages all Uniswap v4 pools, providing all external interfaces for pools, including creation, liquidity modification, trading, and other operations.
* Library contracts:
    * [Pool.sol](./PoolLibrary.md): Pool Library contract, used to specifically execute operations for each pool, such as modifying liquidity, trading, etc.
    * [Position.sol](./PositionLibrary.md): Position Library contract, used to specifically execute operations related to each position, such as updating liquidity and fees.
    * [Hooks.sol](./HooksLibrary.md): Hooks Library contract, used to execute Hooks functions of Uniswap v4 contracts.
    * [CurrencyDelta.sol](./CurrencyDeltaLibrary.md): CurrencyDelta Library contract, used to execute operations related to Flash Accounting.
    * [BalanceDelta.sol](./BalanceDelta.md): BalanceDelta defines the types of balance delta for accounting.
