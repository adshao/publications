[English](./README.md) | [中文](./README_zh.md)

# Deep Dive Into Uniswap v3 Smart Contracts (Part 2)

###### tags: `uniswap` `uniswap-v3` `smart contract` `solidity`

## Uniswap-v3-periphery

The [Uniswap-v3-core](../dive-into-uniswap-v3-contracts/README.md) contract defines the basic methods, while the Uniswap-v3-periphery contract is the one we directly interact with.

For example, it is well known that Uniswap v3 positions are NFTs. These NFTs are created and managed in the periphery contract, without any concept of NFTs in the core contract.

### NonfungiblePositionManager.sol

Position management contract, globally unique, responsible for managing positions of all trading pairs, mainly including the following methods:

* [createAndInitializePoolIfNecessary](#createAndInitializePoolIfNecessary): Create and initialize the contract
* [mint](#mint): Create a position
* [increaseLiquidity](#increaseLiquidity): Increase liquidity
* [decreaseLiquidity](#decreaseLiquidity): Decrease liquidity
* [burn](#burn): Burn the position
* [collect](#collect): Retrieve tokens

It is important to note that this contract inherits from `ERC721` and can mint NFTs. Since each Uniswap v3 position (determined by `owner`, `tickLower`, and `tickUpper`) is unique, it is highly suitable for representation as an NFT.

#### createAndInitializePoolIfNecessary

As mentioned in [Uniswap-v3-core](../dive-into-uniswap-v3-contracts/README.md), after a trading pair contract is created, it needs to be initialized before use.

This method combines a series of operations into one: create and initialize the trading pair.

```solidity
/// @inheritdoc IPoolInitializer
function createAndInitializePoolIfNecessary(
    address token0,
    address token1,
    uint24 fee,
    uint160 sqrtPriceX96
) external payable override returns (address pool) {
    require(token0 < token1);
    pool = IUniswapV3Factory(factory).getPool(token0, token1, fee);

    if (pool == address(0)) {
        pool = IUniswapV3Factory(factory).createPool(token0, token1, fee);
        IUniswapV3Pool(pool).initialize(sqrtPriceX96);
    } else {
        (uint160 sqrtPriceX96Existing, , , , , , ) = IUniswapV3Pool(pool).slot0();
        if (sqrtPriceX96Existing == 0) {
            IUniswapV3Pool(pool).initialize(sqrtPriceX96);
        }
    }
}
```

First, the `pool` object is obtained based on the trading pair tokens (`token0` and `token1`) and `fee`:

* If it does not exist, call the Uniswap-v3-core factory contract `createPool` to create and initialize the trading pair
* If it already exists, determine whether it has been initialized (price) based on the extra `slot0`; if not, call the Uniswap-v3-core `initialize` method to initialize it.

#### mint

Create a new position, with parameters as follows:

* `token0`: Token 0
* `token1`: Token 1
* `fee`: Fee level (must match the fee levels defined in the factory contract)
* `tickLower`: Lower price limit
* `tickUpper`: Upper price limit
* `amount0Desired`: Desired amount of token 0 to deposit
* `amount1Desired`: Desired amount of token 1 to deposit
* `amount0Min`: Minimum amount of `token0` to deposit (to prevent front-running)
* `amount1Min`: Minimum amount of `token1` to deposit (to prevent front-running)
* `recipient`: Position recipient
* `deadline`: Deadline (requests are invalid after this time) (to prevent replay attacks)

Returns:

* `tokenId`: Each position is assigned a unique `tokenId`, representing the NFT
* `liquidity`: The position's liquidity
* `amount0`: The amount of `token0`
* `amount1`: The amount of `token1`

```solidity
/// @inheritdoc INonfungiblePositionManager
function mint(MintParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: params.token0,
            token1: params.token1,
            fee: params.fee,
            recipient: address(this),
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
```

First, liquidity is added through the [addLiquidity](#addLiquidity) method to obtain the actual `liquidity`, consumed `amount0`, `amount1`, and trading pair `pool`.

```solidity
    _mint(params.recipient, (tokenId = _nextId++));
```

Through the `ERC721` contract's `_mint` method, mint an NFT to the recipient `recipient`, with `tokenId` incrementing from 1.

```solidity
    bytes32 positionKey = PositionKey.compute(address(this), params.tickLower, params.tickUpper);
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    // idempotent set
    uint80 poolId =
        cachePoolKey(
            address(pool),
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee})
        );

    _positions[tokenId] = Position({
        nonce: 0,
        operator: address(0),
        poolId: poolId,
        tickLower: params.tickLower,
        tickUpper: params.tickUpper,
        liquidity: liquidity,
        feeGrowthInside0LastX128: feeGrowthInside0LastX128,
        feeGrowthInside1LastX128: feeGrowthInside1LastX128,
        tokensOwed0: 0,
        tokensOwed1: 0
    });

    emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
}
```

Finally, save the position information to `_positions`.

#### increaseLiquidity

Add liquidity to a position. Note that you can change the amount of tokens in the position, but you cannot change the price range.

Parameters as follows:

* `tokenId`: The `tokenId` returned when creating the position, i.e., the NFT's `tokenId`
* `amount0Desired`: The desired amount of `token0` to add
* `amount1Desired`: The desired amount of `token1` to add
* `amount0Min`: The minimum amount of `token0` to add (to prevent front-running)
* `amount1Min`: The minimum amount of `token1` to add (to prevent front-running)
* `deadline`: Deadline (requests are invalid after this time) (to prevent replay attacks)

```solidity
/// @inheritdoc INonfungiblePositionManager
function increaseLiquidity(IncreaseLiquidityParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    Position storage position = _positions[params.tokenId];

    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: poolKey.token0,
            token1: poolKey.token1,
            fee: poolKey.fee,
            tickLower: position.tickLower,
            tickUpper: position.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min,
            recipient: address(this)
        })
    );
```

First, get the position information based on `tokenId`; like in the [mint](#mint) method, here, liquidity is added through the [addLiquidity](#addLiquidity), returning the added `liquidity`, consumed `amount0` and `amount1`, and the trading pair contract `pool`.

```solidity
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);

    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    position.tokensOwed0 += uint128(
        FullMath.mulDiv(
            feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );
    position.tokensOwed1 += uint128(
        FullMath.mulDiv(
            feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );

    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    position.liquidity += liquidity;

    emit IncreaseLiquidity(params.tokenId, liquidity, amount0, amount1);
}
```

Update the contract's position state based on the latest position information in the `pool` object, such as `tokensOwed0` and `tokensOwed1` for `token0` and `token1` to be retrieved, and the position's current liquidity, etc.

#### decreaseLiquidity

Remove liquidity, which can be partial or full. After removal, the tokens will be recorded as tokens to be retrieved, and the [collect](#collect) method must be called again to retrieve the tokens.

Parameters as follows:

* `tokenId`: The `tokenId` returned when creating the position, i.e., the NFT's `tokenId`
* `liquidity`: The amount of liquidity to remove
* `amount0Min`: The minimum amount of `token0` to remove (to prevent front-running)
* `amount1Min`: The minimum amount of `token1` to remove (to prevent front-running)
* `deadline`: Deadline (requests are invalid after this time) (to prevent replay attacks)

```solidity
/// @inheritdoc INonfungiblePositionManager
function decreaseLiquidity(DecreaseLiquidityParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    checkDeadline(params.deadline)
    returns (uint256 amount0, uint256 amount1)
{
```

Note, here the `isAuthorizedForToken` modifer is used:

```solidity
modifier isAuthorizedForToken(uint256 tokenId) {
    require(_isApprovedOrOwner(msg.sender, tokenId), 'Not approved');
    _;
}
```

To confirm that the current user has the right to operate the `tokenId`, otherwise, removal is prohibited.

```solidity
    require(params.liquidity > 0);
    Position storage position = _positions[params.tokenId];

    uint128 positionLiquidity = position.liquidity;
    require(positionLiquidity >= params.liquidity);
```

To confirm that the position liquidity is greater than or equal to the liquidity to be removed.

```solidity
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
    (amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity);

    require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

Call the Uniswap-v3-core's [burn](../dive-into-uniswap-v3-contracts/README.md#burn) method to destroy liquidity, returning the corresponding `token0` and `token1` token amounts `amount0` and `amount1`, and confirm that they meet the `amount0Min` and `amount1Min` restrictions.

```solidity
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);
    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    position.tokensOwed0 +=
        uint128(amount0) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        );
    position.tokensOwed1 +=
        uint128(amount1) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        );

    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    // subtraction is safe because we checked positionLiquidity is gte params.liquidity
    position.liquidity = positionLiquidity - params.liquidity;

    emit DecreaseLiquidity(params.tokenId, params.liquidity, amount0, amount1);
}
```

Similar to [increaseLiquidity](#increaseLiquidity), here it calculates the position's tokens to be retrieved, etc.

#### burn

Destroy the position NFT. Only when the position's liquidity is 0, and the amount of tokens to be retrieved is 0, can the NFT be destroyed.

Likewise, calling this method requires verifying that the current user owns the `tokenId`.

```solidity
/// @inheritdoc INonfungiblePositionManager
function burn(uint256 tokenId) external payable override isAuthorizedForToken(tokenId) {
    Position storage position = _positions[tokenId];
    require(position.liquidity == 0 && position.tokensOwed0 == 0 && position.tokensOwed1 == 0, 'Not cleared');
    delete _positions[tokenId];
    _burn(tokenId);
}
```

#### collect

Retrieve tokens to be collected.

Parameters as follows:

* `tokenId`: The `tokenId` returned when creating the position, i.e., the NFT's `tokenId`
* `recipient`: Token recipient
* `amount0Max`: The maximum amount of `token0` tokens to collect
* `amount1Max`: The maximum amount of `token1` tokens to collect

```solidity
/// @inheritdoc INonfungiblePositionManager
function collect(CollectParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    require(params.amount0Max > 0 || params.amount1Max > 0);
    // allow collecting to the nft position manager address with address 0
    address recipient = params.recipient == address(0) ? address(this) : params.recipient;

    Position storage position = _positions[params.tokenId];

    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));

    (uint128 tokensOwed0, uint128 tokensOwed1) = (position.tokensOwed0, position.tokensOwed1);
```

Obtain the number of tokens to be retrieved.

```solidity
    // trigger an update of the position fees owed and fee growth snapshots if it has any liquidity
    if (position.liquidity > 0) {
        pool.burn(position.tickLower, position.tickUpper, 0);
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) =
            pool.positions(PositionKey.compute(address(this), position.tickLower, position.tickUpper));

        tokensOwed0 += uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );
        tokensOwed1 += uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );

        position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    }
```

If the position contains liquidity, trigger an update of the position state. Here, 0 liquidity is used to trigger. This is because Uniswap-v3-core only updates the position state during `mint` and `burn`, and the `collect` method may be called after `swap`, which may result in the position state not being the latest.

```solidity
    // compute the arguments to give to the pool#collect method
    (uint128 amount0Collect, uint128 amount1Collect) =
        (
            params.amount0Max > tokensOwed0 ? tokensOwed0 : params.amount0Max,
            params.amount1Max > tokensOwed1 ? tokensOwed1 : params.amount1Max
        );

    // the actual amounts collected are returned
    (amount0, amount1) = pool.collect(
        recipient,
        position.tickLower,
        position.tickUpper,
        amount0Collect,
        amount1Collect
    );

    // sometimes there will be a few less wei than expected due to rounding down in core, but we just subtract the full amount expected
    // instead of the actual amount so we can burn the token
    (position.tokensOwed0, position.tokensOwed1) = (tokensOwed0 - amount0Collect, tokensOwed1 - amount1Collect);

    emit Collect(params.tokenId, recipient, amount0Collect, amount1Collect);
}
```

Call Uniswap-v3-core's `collect` method to retrieve tokens and update the position's tokens to be retrieved.

### SwapRouter.sol

Swap tokens, including the following methods:

* [exactInputSingle](#exactInputSingle): Single-step swap, specifying the amount of input tokens to get as many output tokens as possible
* [exactInput](#exactInput): Multi-step swap, specifying the amount of input tokens to get as many output tokens as possible
* [exactOutputSingle](#exactOutputSingle): Single-step swap, specifying the amount of output tokens to provide as few input tokens as possible
* [exactOutput](#exactOutput): Multi-step swap, specifying the amount of output tokens to provide as few input tokens as possible

Additionally, the contract also implements:

* [uniswapV3SwapCallback](#uniswapV3SwapCallback): Swap callback method
* [exactInputInternal](#exactInputInternal): Single-step swap, internal method, specifying the amount of input tokens to get as many output tokens as possible
* [exactOutputInternal](#exactOutputInternal): Single-step swap, internal method, specifying the amount of output tokens to provide as few input tokens as possible

#### exactInputSingle

Single-step swap, specifying the amount of input tokens to get as many output tokens as possible.

Parameters as follows:

* `tokenIn`: Input token address
* `tokenOut`: Output token address
* `fee`: Fee level
* `recipient`: Output token recipient
* `deadline`: Deadline, requests are invalid after this time
* `amountIn`: The amount of input tokens
* `amountOutMinimum`: The minimum amount of output tokens to receive
* `sqrtPriceLimitX96`: (Highest or lowest) limit price

Returns:

* `amountOut`: The amount of output tokens

```solidity
/// @inheritdoc ISwapRouter
function exactInputSingle(ExactInputSingleParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    amountOut = exactInputInternal(
        params.amountIn,
        params.recipient,
        params.sqrtPriceLimitX96,
        SwapCallbackData({path: abi.encodePacked(params.tokenIn, params.fee, params.tokenOut), payer: msg.sender})
    );
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

This method actually calls [exactInputInternal](#exactInputInternal), finally ensuring the output token amount `amountOut` meets the minimum output token requirement `amountOutMinimum`.

Note, `SwapCallbackData` in the `path` is encoded according to the format defined in [Path.sol](#Pathsol).

#### exactInput

Multi-step swap, specifying the amount of input tokens to get as many output tokens as possible.

Parameters as follows:

* `path`: Swap path, format see: [Path.sol](#Pathsol)
* `recipient`: Output token recipient
* `deadline`: Transaction deadline
* `amountIn`: The amount of input tokens
* `amountOutMinimum`: The minimum amount of output tokens

Returns:

* `amountOut`: The amount of output tokens

```solidity
/// @inheritdoc ISwapRouter
function exactInput(ExactInputParams memory params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    address payer = msg.sender; // msg.sender pays for the first hop

    while (true) {
        bool hasMultiplePools = params.path.hasMultiplePools();

        // the outputs of prior swaps become the inputs to subsequent ones
        params.amountIn = exactInputInternal(
            params.amountIn,
            hasMultiplePools ? address(this) : params.recipient, // for intermediate swaps, this contract custodies
            0,
            SwapCallbackData({
                path: params.path.getFirstPool(), // only the first pool in the path is necessary
                payer: payer
            })
        );

        // decide whether to continue or terminate
        if (hasMultiplePools) {
            payer = address(this); // at this point, the caller has paid
            params.path = params.path.skipToken();
        } else {
            amountOut = params.amountIn;
            break;
        }
    }

    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

In a multi-step swap, it needs to be split into multiple single-step swaps according to the swap path and proceed in a loop until the path ends.

If it is the first step of the swap, then `payer` is the contract caller; otherwise, `payer` is the current `SwapRouter` contract.

In the loop, first determine whether there are 2 or more pools left in the `path` according to [hasMultiplePools](#hasMultiplePools). If yes, then the intermediate swap steps' recipient address is set to the current `SwapRouter` contract; otherwise, it is set to the entrance parameter `recipient`.

After each step of the swap, remove the first 20+3 bytes from the current `path`, i.e., pop the front token+fee information, enter the next swap, and use the output of each step as the input for the next swap.

Each step of the swap calls [exactInputInternal](#exactInputInternal) for execution.

After multiple steps of swapping, confirm that the final `amountOut` meets the minimum output token requirement `amountOutMinimum`.

#### exactOutputSingle

Single-step swap, specifying the amount of output tokens to provide as few input tokens as possible.

Parameters as follows:

* `tokenIn`: Input token address
* `tokenOut`: Output token address
* `fee`: Fee level
* `recipient`: Output token recipient
* `deadline`: Request deadline
* `amountOut`: The amount of output tokens
* `amountInMaximum`: The maximum amount of input tokens
* `sqrtPriceLimitX96`: The maximum or minimum token price

Returns:

* `amountIn`: The actual amount of input tokens

```solidity
/// @inheritdoc ISwapRouter
function exactOutputSingle(ExactOutputSingleParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // avoid an SLOAD by using the swap return data
    amountIn = exactOutputInternal(
        params.amountOut,
        params.recipient,
        params.sqrtPriceLimitX96,
        SwapCallbackData({path: abi.encodePacked(params.tokenOut, params.fee, params.tokenIn), payer: msg.sender})
    );

    require(amountIn <= params.amountInMaximum, 'Too much requested');
    // has to be reset even though we don't use it in the single hop case
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}
```

Call [exactOutputInternal](#exactOutputInternal) to complete the single-step swap, and ensure the actual input token amount `amountIn` is less than or equal to the maximum input token amount `amountInMaximum`.

#### exactOutput

Multi-step swap, specifying the amount of output tokens to provide as few input tokens as possible.

Parameters as follows:

* `path`: Swap path, format see: [Path.sol](#Path)
* `recipient`: Output token recipient
* `deadline`: Request deadline
* `amountOut`: The specified amount of output tokens
* `amountInMaximum`: The maximum amount of input tokens

```solidity
/// @inheritdoc ISwapRouter
function exactOutput(ExactOutputParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // it's okay that the payer is fixed to msg.sender here, as they're only paying for the "final" exact output
    // swap, which happens first, and subsequent swaps are paid for within nested callback frames
    exactOutputInternal(
        params.amountOut,
        params.recipient,
        0,
        SwapCallbackData({path: params.path, payer: msg.sender})
    );

    amountIn = amountInCached;
    require(amountIn <= params.amountInMaximum, 'Too much requested');
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}
```

Call [exactOutputInternal](#exactOutputInternal) to complete the swap. Note, this method will continue the next step of the swap in the callback method, so it does not need a loop trade like [exactInput](#exactInput).

Finally, ensure the actual input token amount `amountIn` is less than or equal to the maximum input token amount `amountInMaximum`.

#### exactInputInternal

Single-step swap, internal method, specifying the amount of input tokens to get as many output tokens as possible.

```solidity
/// @dev Performs a single exact input swap
function exactInputInternal(
    uint256 amountIn,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) private returns (uint256 amountOut) {
    // allow swapping to the router address with address 0
    if (recipient == address(0)) recipient = address(this);
```

If `recipient` is not specified, then the default is the current `SwapRouter` contract address. This is because, in multi-step swaps, intermediate tokens need to be saved in the current `SwapRouter` contract.

```solidity
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
```

Decode the first pool information in `path` according to [decodeFirstPool](#decodeFirstPool).

```solidity
    bool zeroForOne = tokenIn < tokenOut;
```

Since Uniswap v3 pools have `token0` address less than `token1`, determine whether the current swap is from `token0` to `token1` based on the two token addresses. Note, `tokenIn` can be either `token0` or `token1`.

```solidity
    (int256 amount0, int256 amount1) =
        getPool(tokenIn, tokenOut, fee).swap(
            recipient,
            zeroForOne,
            amountIn.toInt256(),
            sqrtPriceLimitX96 == 0
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
            abi.encode(data)
        );
```

Call the [swap](#swap) method, getting the required `amount0` and `amount1` to complete this swap. If swapping from `token0` to `token1`, then `amount1` is negative; otherwise, `amount0` is negative.

If `sqrtPriceLimitX96` is not specified, then default to the lowest or highest price, because in multi-step swaps, it's not possible to specify the price for each step.

```solidity
    return uint256(-(zeroForOne ? amount1 : amount0));
```

Returns `amountOut`.

#### exactOutputInternal

Single-step swap, private method, specifying the amount of output tokens to provide as few input tokens as possible.

```solidity
/// @dev Performs a single exact output swap
function exactOutputInternal(
    uint256 amountOut,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) private returns (uint256 amountIn) {
    // allow swapping to the router address with address 0
    if (recipient == address(0)) recipient = address(this);

    (address tokenOut, address tokenIn, uint24 fee) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;
```

This part of the code is similar to [exactInputInternal](#exactInputInternal).

```solidity
    (int256 amount0Delta, int256 amount1Delta) =
        getPool(tokenIn, tokenOut, fee).swap(
            recipient,
            zeroForOne,
            -amountOut.toInt256(),
            sqrtPriceLimitX96 == 0
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
            abi.encode(data)
        );
```

Call the Uniswap-v3-core's `swap` method to complete a single-step swap. Note, since the amount of output tokens is specified, here, `-amountOut.toInt256()` is used.
The returned `amount0Delta` and `amount1Delta` are the required `token0` amount and the actual output `token1` amount.

```solidity
    uint256 amountOutReceived;
    (amountIn, amountOutReceived) = zeroForOne
        ? (uint256(amount0Delta), uint256(-amount1Delta))
        : (uint256(amount1Delta), uint256(-amount0Delta));
    // it's technically possible to not receive the full output amount,
    // so if no price limit has been specified, require this possibility away
    if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);
}
```

#### uniswapV3SwapCallback

Swap's callback method, implementing the `IUniswapV3SwapCallback.uniswapV3SwapCallback` interface.

Parameters as follows:

* `amount0Delta`: The `amount0` generated by this swap (corresponding to the `token0`); for the contract, if greater than 0, it indicates that the token should be input; if less than 0, it indicates that the token should be received
* `amount1Delta`: The `amount1` generated by this swap (corresponding to the `token1`); for the contract, if greater than 0, it indicates that the token should be input; if less than 0, it indicates that the token should be received
* `_data`: Callback parameters, here of type `SwapCallbackData`

```solidity
/// @inheritdoc IUniswapV3SwapCallback
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata _data
) external override {
    require(amount0Delta > 0 || amount1Delta > 0); // swaps entirely within 0-liquidity regions are not supported
    SwapCallbackData memory data = abi.decode(_data, (SwapCallbackData));
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
    CallbackValidation.verifyCallback(factory, tokenIn, tokenOut, fee);
```

Decode the callback parameter `_data` and get the information of the first trading pair on the trading path.

```solidity
    (bool isExactInput, uint256 amountToPay) =
        amount0Delta > 0
            ? (tokenIn < tokenOut, uint256(amount0Delta))
            : (tokenOut < tokenIn, uint256(amount1Delta));
```

Based on different inputs, there are several trading combinations:

|Scenario|Explanation|amount0Delta > 0|amount1Delta > 0|tokenIn < tokenOut|isExactInput|amountToPay|
|---|---|---|---|---|---|---|
|1|Specify `token0` amount input, output as much `token1` as possible|true|false|true|true|amount0Delta|
|2|Provide as few `token0` as possible, output specified `token1` amount|true|false|true|false|amount0Delta|
|3|Specify `token1` amount input, output as much `token0` as possible|false|true|false|true|amount1Delta|
|4|Provide as few `token1` as possible, output specified `token0` amount|false|true|false|false|amount1Delta|

```solidity
    if (isExactInput) {
        pay(tokenIn, data.payer, msg.sender, amountToPay);
    } else {
        // either initiate the next swap or pay
        if (data.path.hasMultiplePools()) {
            data.path = data.path.skipToken();
            exactOutputInternal(amountToPay, msg.sender, 0, data);
        } else {
            amountInCached = amountToPay;
            tokenIn = tokenOut; // swap in/out because exact output swaps are reversed
            pay(tokenIn, data.payer, msg.sender, amountToPay);
        }
    }
```

* If `isExactInput`, i.e., specify input token scenario, scenarios 1 and 3 in the table above, then directly transfer `amount0Delta` (scenario 1) or `amount1Delta` (scenario 3) (both are positive numbers) to the `SwapRouter` contract.
* If it is a specify output token scenario
    - If it is a multi-step swap, then remove the first 23 characters (pop the front token+fee), use the required input as the output for the next step, and enter the next step of the swap
    - If it is a single-step swap (or the last step), then swap `tokenIn` and `tokenOut`, and transfer to the `SwapRouter` contract

### LiquidityManagement.sol

#### uniswapV3MintCallback

Callback method for adding liquidity.

Parameters as follows:

* `amount0Owed`: The amount of `token0` to be transferred
* `amount1Owed`: The amount of `token1` to be transferred
* `data`: Callback parameters passed in the `mint` method

```solidity
/// @inheritdoc IUniswapV3MintCallback
function uniswapV3MintCallback(
    uint256 amount0Owed,
    uint256 amount1Owed,
    bytes calldata data
) external override {
    MintCallbackData memory decoded = abi.decode(data, (MintCallbackData));
    CallbackValidation.verifyCallback(factory, decoded.poolKey);

    if (amount0Owed > 0) pay(decoded.poolKey.token0, decoded.payer, msg.sender, amount0Owed);
    if (amount1Owed > 0) pay(decoded.poolKey.token1, decoded.payer, msg.sender, amount1Owed);
}
```

First, decode the callback parameter `MintCallbackData` in reverse and confirm that this method is called by the specified trading pair contract, because this method is an `external` method and can be called externally, thus needing to confirm the caller.

Finally, transfer the specified token amounts to the caller.

#### addLiquidity

Add liquidity to an initialized trading pair (pool).

```solidity
/// @notice Add liquidity to an initialized pool
function addLiquidity(AddLiquidityParams memory params)
    internal
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1,
        IUniswapV3Pool pool
    )
{
    PoolAddress.PoolKey memory poolKey =
        PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee});

    pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
```

Get the trading pair `pool` based on `factory`, `token0`, `token1`, and `fee`.

```solidity
    // compute the liquidity amount
    {
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        uint160 sqrtRatioAX96 = TickMath.getSqrtRatioAtTick(params.tickLower);
        uint160 sqrtRatioBX96 = TickMath.getSqrtRatioAtTick(params.tickUpper);

        liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96,
            sqrtRatioAX96,
            sqrtRatioBX96,
            params.amount0Desired,
            params.amount1Desired
        );
    }
```

From `slot0`, get the current price `sqrtPriceX96`, calculate the lowest price `sqrtRatioAX96` and highest price `sqrtRatioBX96` based on `tickLower` and `tickUpper`.

Calculate the maximum liquidity that can be obtained according to [getLiquidityForAmounts](#getLiquidityForAmounts).

```solidity
    (amount0, amount1) = pool.mint(
        params.recipient,
        params.tickLower,
        params.tickUpper,
        liquidity,
        abi.encode(MintCallbackData({poolKey: poolKey, payer: msg.sender}))
    );
```

Use the Uniswap-v3-core's `mint` method to add liquidity and return the actual consumed `amount0` and `amount1`.

In the Uniswap-v3-core's `mint` method mentioned, the caller needs to implement [uniswapV3MintCallback](#LiquidityManagement.uniswapV3MintCallback) interface. Here, `MintCallbackData` is passed as a callback parameter, which can be decoded in reverse in the `uniswapV3MintCallback` method to obtain the trading pair and user information.

```solidity
require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

Finally, ensure the actual consumed `amount0` and `amount1` meet the minimum requirements of `amount0Min` and `amount1Min`.

### LiquidityAmounts.sol

#### getLiquidityForAmount0

Calculate liquidity based on `amount0` and the price range.

According to the formula in Uniswap-v3-core's `getAmount0Delta`:

$$
amount0 = x_b - x_a = L \cdot (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}}) = L \cdot (\frac{\sqrt{P_a} - \sqrt{P_b}}{\sqrt{P_a} \cdot \sqrt{P_b}})
$$

We get:

$$
L = amount0 \cdot (\frac{\sqrt{P_a} \cdot \sqrt{P_b}}{\sqrt{P_a} - \sqrt{P_b}})
$$

```solidity
/// @notice Computes the amount of liquidity received for a given amount of token0 and price range
/// @dev Calculates amount0 * (sqrt(upper) * sqrt(lower)) / (sqrt(upper) - sqrt(lower))
/// @param sqrtRatioAX96 A sqrt price representing the first tick boundary
/// @param sqrtRatioBX96 A sqrt price representing the second tick boundary
/// @param amount0 The amount0 being sent in
/// @return liquidity The amount of returned liquidity
function getLiquidityForAmount0(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount0
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);
    uint256 intermediate = FullMath.mulDiv(sqrtRatioAX96, sqrtRatioBX96, FixedPoint96.Q96);
    return toUint128(FullMath.mulDiv(amount0, intermediate, sqrtRatioBX96 - sqrtRatioAX96));
}
```

#### getLiquidityForAmount1

Calculate liquidity based on `amount1` and the price range.

According to the formula in Uniswap-v3-core's `getAmount1Delta`:

$$
amount1 = y_b - y_a = L \cdot \Delta{\sqrt{P}} = L \cdot (\sqrt{P_b} - \sqrt{P_a})
$$

We get:

$$
L = \frac{amount1}{\sqrt{P_b} - \sqrt{P_a}}
$$

```solidity
/// @notice Computes the amount of liquidity received for a given amount of token1 and price range
/// @dev Calculates amount1 / (sqrt(upper) - sqrt(lower)).
/// @param sqrtRatioAX96 A sqrt price representing the first tick boundary
/// @param sqrtRatioBX96 A sqrt price representing the second tick boundary
/// @param amount1 The amount1 being sent in
/// @return liquidity The amount of returned liquidity
function getLiquidityForAmount1(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);
    return toUint128(FullMath.mulDiv(amount1, FixedPoint96.Q96, sqrtRatioBX96 - sqrtRatioAX96));
}
```

#### getLiquidityForAmounts

Calculates the maximum liquidity that can be returned based on the current price.

* Because when $\sqrt{P}$ increases, $x$ is consumed, so if the current price is below the lower point of the price range, liquidity must be calculated completely based on $x$ or `amount0`
* Conversely, if the current price is above the upper point of the price range, liquidity needs to be calculated based on $y$ or `amount1`

As shown in the figure below:

$$
p,...,\overbrace{p_a,...,p_b}^{amount0}
$$

$$
\overbrace{p_a,...}^{amount1},p,\overbrace{...,p_b}^{amount0}
$$

$$
\overbrace{p_a,...,p_b}^{amount1},...,p
$$

Where $p$ represents the current price, $p_a$ represents the lower point of the range, and $p_b$ represents the upper point of the range.

```solidity
/// @notice Computes the maximum amount of liquidity received for a given amount of token0, token1, the current
/// pool prices and the prices at the tick boundaries
/// @param sqrtRatioX96 A sqrt price representing the current pool prices
/// @param sqrtRatioAX96 A sqrt price representing the first tick boundary
/// @param sqrtRatioBX96 A sqrt price representing the second tick boundary
/// @param amount0 The amount of token0 being sent in
/// @param amount1 The amount of token1 being sent in
/// @return liquidity The maximum amount of liquidity received
function getLiquidityForAmounts(
    uint160 sqrtRatioX96,
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    if (sqrtRatioX96 <= sqrtRatioAX96) {
        liquidity = getLiquidityForAmount0(sqrtRatioAX96, sqrtRatioBX96, amount0);
    } else if (sqrtRatioX96 < sqrtRatioBX96) {
        uint128 liquidity0 = getLiquidityForAmount0(sqrtRatioX96, sqrtRatioBX96, amount0);
        uint128 liquidity1 = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioX96, amount1);

        liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
    } else {
        liquidity = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioBX96, amount1);
    }
}
```

### Path.sol

In Uniswap v3 [SwapRouter](#SwapRoutersol), the trading path is encoded as a `bytes` type string, formatted as:

$$
\overbrace{token_0}^{20}\overbrace{fee_0}^{3}\overbrace{token_1}^{20}\overbrace{fee_1}^{3}\overbrace{token_2}^{20}...
$$

Where the length of $token_n$ is 20 bytes, and the length of $fee_n$ is 3 bytes. The above path indicates swapping from `token0` to `token1` using a pool (`token0`, `token1`, `fee0`) with fee level `fee0`, then continuing to swap to `token2` using a pool (`token1`, `token2`, `fee1`) with fee level `fee1`.

Example of a trading path `path`:

$$
0x\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{token0,20bytes}\overbrace{0001f4}^{fee,3bytes}\overbrace{68b3465833fb72a70ecdf485e0e4c7bd8665fc45}^{token1,20bytes}
$$

#### hasMultiplePools

Determines if the trading path goes through multiple pools (2 or more).

```solidity
/// @notice Returns true iff the path contains two or more pools
/// @param path The encoded swap path
/// @return True if path contains two or more pools, otherwise false
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

From the encoding of the path above, if passing through 2 pools, it must include at least 3 tokens, so the path length must be at least $20+3+20+3+20=66$ bytes. In the code, `MULTIPLE_POOLS_MIN_LENGTH` is equal to 66.

#### numPools

Calculates the number of pools in the path.

The algorithm is:

$$
num = \frac{length - 20}{20 + 3}
$$

```solidity
/// @notice Returns the number of pools in the path
/// @param path The encoded swap path
/// @return The number of pools in the path
function numPools(bytes memory path) internal pure returns (uint256) {
    // Ignore the first token address. From then on, every fee and token offset indicates a pool.
    return ((path.length - ADDR_SIZE) / NEXT_OFFSET);
}
```

#### decodeFirstPool

Decodes the information of the first path, including `token0`, `token1`, and `fee`.

It returns the substring 0-19 (`token0`, converted to `address` type), substring 20-22 (`fee`, converted to `uint24` type), and substring 23-42 (`token1`, converted to `address` type). Please refer to the `BytesLib.sol` method [toAddress](#toAddress) and [toUint24](#toUint24).

```solidity
/// @notice Decodes the first pool in path
/// @param path The bytes encoded swap path
/// @return tokenA The first token of the given pool
/// @return tokenB The second token of the given pool
/// @return fee The fee level of the pool
function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenA,
        address tokenB,
        uint24 fee
    )
{
    tokenA = path.toAddress(0);
    fee = path.toUint24(ADDR_SIZE);
    tokenB = path.toAddress(NEXT_OFFSET);
}
```

#### getFirstPool

Returns the path of the first pool, i.e., the substring composed of the first 43 (20+3+20) characters.

```solidity
/// @notice Gets the segment corresponding to the first pool in the path
/// @param path The bytes encoded swap path
/// @return The segment containing all data necessary to target the first pool in the path
function getFirstPool(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(0, POP_OFFSET);
}
```

#### skipToken

Skips the first `token+fee` on the current path, i.e., skips the first 20+3 characters.

```solidity
/// @notice Skips a token + fee element from the buffer and returns the remainder
/// @param path The swap path
/// @return The remaining token + fee elements in the path
function skipToken(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
}
```

### BytesLib.sol

#### toAddress

Reads an address (20 characters) from a specified index in the string:

```solidity
function toAddress(bytes memory _bytes, uint256 _start) internal pure returns (address) {
    require(_start + 20 >= _start, 'toAddress_overflow');
    require(_bytes.length >= _start + 20, 'toAddress_outOfBounds');
    address tempAddress;

    assembly {
        tempAddress := div(mload(add(add(_bytes, 0x20), _start)), 0x1000000000000000000000000)
    }

    return tempAddress;
}
```

Since the type of variable `_bytes` is `bytes`, according to the [ABI definition](https://docs.soliditylang.org/en/develop/abi-spec.html), the first 32 bytes of `bytes` store the length of the string. Therefore, we need to skip the first 32 bytes, i.e., `add(_bytes, 0x20); add(add(_bytes, 0x20), _start)` indicates positioning to the specified index `_start` of the string; `mload` reads 32 bytes from this index. Since the `address` type is only 20 bytes, it requires `div 0x1000000000000000000000000`, which means shifting right by 12 bytes.

Assuming `_strat = 0`, the distribution of `_bytes` is as shown below:

$$
0x\overbrace{0000000...2b}^{length,32bytes}\underbrace{\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{address, 20 bytes}\overbrace{0001f468b3465833fb72a70e}^{div,12 bytes}}_{mload, 32bytes}cdf485e0e4c7bd8665fc45
$$

#### toUint24

Reads a `uint24` (24 bits, i.e., 3 characters) from a specified index in the string:

```solidity
function toUint24(bytes memory _bytes, uint256 _start) internal pure returns (uint24) {
    require(_start + 3 >= _start, 'toUint24_overflow');
    require(_bytes.length >= _start + 3, 'toUint24_outOfBounds');
    uint24 tempUint;

    assembly {
        tempUint := mload(add(add(_bytes, 0x3), _start))
    }

    return tempUint;
}
```

Since the first 32 characters of `_bytes` represent the length of the string, `mload` reads 32 bytes and ensures that the 3 bytes starting from `_start` are in the least significant position of the 32 bytes read. Assigning this value to a variable of type `uint24` will only retain the least significant 3 bytes.

Assuming `_start = 0`, the distribution of `_bytes` is as shown below:

$$
0x\overbrace{000000}^{0x3+\_start}\underbrace{0...2b\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{address1,20bytes}\overbrace{0001f4}^{fee,3bytes}}_{mload,32bytes}\overbrace{68b3465833fb72a70ecdf485e0e4c7bd8665fc45}^{address2,20bytes}
$$

### OracleLibrary.sol

According to formulas 5.3-5.5 in the white paper, the geometric mean price from $t_1$ to $t_2$ is calculated as follows:

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{\sum_{i=t_1}^{t_2} \log_{1.0001}(P_i)}{t_2 - t_1} \quad \text{(5.3)}
$$

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1} \quad \text{(5.4)}
$$

$$
P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2 - t_1}} \quad \text{(5.5)}
$$

This contract provides oracle-related methods, including:

* [consult](#consult): Queries the geometric mean price from a period of time ago to now (in `tick` form)
* [getQuoteAtTick](#getQuoteAtTick): Calculates the token price based on `tick`

#### consult

Queries the geometric mean price from a period of time ago to now (in `tick` form).

Parameters are as follows:

* `pool`: The pool address of the trading pair
* `period`: The interval in seconds

Returns:

* `timeWeightedAverageTick`: The time-weighted average price

```solidity
/// @notice Fetches time-weighted average tick using Uniswap V3 oracle
/// @param pool Address of Uniswap V3 pool that we want to observe
/// @param period Number of seconds in the past to start calculating time-weighted average
/// @return timeWeightedAverageTick The time-weighted average tick from (block.timestamp - period) to block.timestamp
function consult(address pool, uint32 period) internal view returns (int24 timeWeightedAverageTick) {
    require(period != 0, 'BP');

    uint32[] memory secondAgos = new uint32[](2);
    secondAgos[0] = period;
    secondAgos[1] = 0;
```

Construct two observation points, the first is `period` time ago, and the second is now.

```solidity
    (int56[] memory tickCumulatives, ) = IUniswapV3Pool(pool).observe(secondAgos);
    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
```

According to the `IUniswapV3Pool.observe` method, obtain the cumulative `tick`, i.e., $a_{t_2}$ and $a_{t_1}$ in formula 5.4.

$$
tickCumulativesDelta = a_{t_2} - a_{t_1}
$$

```solidity
    timeWeightedAverageTick = int24(tickCumulativesDelta / period);
```

$$
    timeWeightedAverageTick = \frac{tickCumulativesDelta}{t_2 - t_1}
$$

```solidity
    // Always round to negative infinity
    if (tickCumulativesDelta < 0 && (tickCumulativesDelta % period != 0)) timeWeightedAverageTick--;
```

If `tickCumulativesDelta` is negative and cannot be divided by `period` without a remainder, then subtract 1 from the average price.

#### getQuoteAtTick

```solidity
/// @notice Given a tick and a token amount, calculates the amount of token received in exchange
/// @param tick Tick value used to calculate the quote
/// @param baseAmount Amount of token to be converted
/// @param baseToken Address of an ERC20 token contract used as the baseAmount denomination
/// @param quoteToken Address of an ERC20 token contract used as the quoteAmount denomination
/// @return quoteAmount Amount of quoteToken received for baseAmount of baseToken
function getQuoteAtTick(
    int24 tick,
    uint128 baseAmount,
    address baseToken,
    address quoteToken
) internal pure returns (uint256 quoteAmount) {
    uint160 sqrtRatioX96 = TickMath.getSqrtRatioAtTick(tick);

    // Calculate quoteAmount with better precision if it doesn't overflow when multiplied by itself
    if (sqrtRatioX96 <= type(uint128).max) {
        uint256 ratioX192 = uint256(sqrtRatioX96) * sqrtRatioX96;
        quoteAmount = baseToken < quoteToken
            ? FullMath.mulDiv(ratioX192, baseAmount, 1 << 192)
            : FullMath.mulDiv(1 << 192, baseAmount, ratioX192);
    } else {
        uint256 ratioX128 = FullMath.mulDiv(sqrtRatioX96, sqrtRatioX96, 1 << 64);
        quoteAmount = baseToken < quoteToken
            ? FullMath.mulDiv(ratioX128, baseAmount, 1 << 128)
            : FullMath.mulDiv(1 << 128, baseAmount, ratioX128);
    }
}
```

Based on the Uniswap-v3-core `getSqrtRatioAtTick` method, calculate $\sqrt{P}$ corresponding to `tick`, i.e., $\sqrt{\frac{token1}{token0}}$.

If `baseToken < quoteToken`, then `baseToken` is `token0`, `quoteToken` is `token1`:

$$
quoteAmount = baseAmount \cdot (\sqrt{P})^2
$$

Otherwise, `baseToken` is `token1`, `quoteToken` is `token0`:

$$
quoteAmount = \frac{baseAmount}{(\sqrt{P})^2}
$$
