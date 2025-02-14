# v4-periphery

v4-periphery [[Github](https://github.com/Uniswap/v4-periphery/)]: Contains Uniswap v4 periphery contracts, mainly including:

* [PositionManager.sol](./PositionManager.md): PositionManager contract, used to manage operations such as creating, destroying, and modifying liquidity of positions, calling [PoolManager](../../v4-core/en/PoolManager.md) to execute specific operations.
    * External contracts operate positions through the PositionManager contract, rather than directly calling the v4-core PoolManager contract.
    * Supports combining multiple operations into one transaction to ensure atomicity and reduce gas consumption.

* [V4Router.sol](./V4Router.md): V4Router contract, used to execute trading operations, calling the [PoolManager](../../v4-core/en/PoolManager.md) contract to execute specific trading operations.
    * Supports single-hop and multi-hop trading.
    * Supports specifying input or output token amounts.
