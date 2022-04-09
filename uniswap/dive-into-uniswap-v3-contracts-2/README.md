# 深入理解 Uniswap v3 合约代码 （二）

###### tags: `uniswap` `solidity` `logarithm` `uniswap-v3` `tick` `periphery` `contract`

## Uniswap-v3-periphery

[Uniswap-v3-core](https://hackmd.io/TDPPCAIgRRqVDPwsSm6Kfw)合约定义的是基础方法，而Uniswap-v3-periphery合约才是我们平常直接交互的合约。

比如，众所周知Uniswap v3头寸是一个NFT，这个NFT就是在periphery合约中创建和管理的，在core合约中并没有任何NFT的概念。

### NonfungiblePositionManager.sol

头寸管理合约，全局仅有一个，负责管理所有交易对的头寸，主要包括以下几个方法：

* [createAndInitializePoolIfNecessary](#createAndInitializePoolIfNecessary)：创建并初始化合约
* [mint](#mint)：创建头寸
* [increaseLiquidity](#increaseLiquidity)：添加流动性
* [decreaseLiquidity](#decreaseLiquidity)：减少流动性
* [burn](#burn)：销毁头寸
* [collect](#collect)：取回代币

需要特别注意，该合约继承了`ERC721`，可以mint NFT。因为每个Uniswap v3的头寸（由`owner`、`tickLower`和`tickUpper`确定）是唯一的，因此非常适合用NFT表示。

#### createAndInitializePoolIfNecessary

我们在[Uniswap-v3-core](https://hackmd.io/TDPPCAIgRRqVDPwsSm6Kfw)中提到，一个交易对合约被创建后，需要初始化才能使用。

本方法就把这一系列操作合并成一个方法：创建并初始化交易对。

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

首先根据交易对代币（`token`和`token1`）和手续费`fee`获取`pool`对象：

* 如果不存在，则调用Uniswap-v3-core工厂合约`createPool`创建该交易对并初始化
* 如果已存在，则根据额`slot0`判断是否已经初始化（价格），如果没有则调用Uniswap-v3-core的`initialize`方法进行初始化。

#### mint

创建新头寸，方法接受的参数如下：

* `token0`：代币0
* `token1`：代币1
* `fee`：手续费等级（需符合工厂合约中定义的手续费等级）
* `tickLower`：价格区间低点
* `tickUpper`：价格区间高点
* `amount0Desired`：希望存入的代币0数量
* `amount1Desired`：希望存入的代币1数量
* `amount0Min`：最少存入的`token0`数量（防止被frontrun）
* `amount1Min`：最少存入的`token1`数量（防止被frontrun）
* `recipient`：头寸接收者
* `deadline`：截止时间（超过该时间后请求无效）（防止重放攻击）

返回：

* `tokenId`：每个头寸会分配一个唯一的`tokenId`，代表NFT
* `liquidity`：头寸的流动性
* `amount0`：`token0`的数量
* `amount1`：`token1`的数量

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

首先通过[addLiquidity](#addLiquidity)方法完成流动性添加，获得实际得到的流动性`liquidity`，消耗的`amount0`、`amount1`，以及交易对`pool`。

```solidity
    _mint(params.recipient, (tokenId = _nextId++));
```

通过`ERC721`合约的`_mint`方法，向接收者`recipient`铸造NFT，`tokenId`从1开始递增。

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

最后，保存头寸信息到`_positions`中。

#### increaseLiquidity

为一个头寸添加流动性。需注意，可以修改头寸的代币数量，但是不能修改价格区间。

参数如下：

* `tokenId`：创建头寸时返回的`tokenId`，即NFT的`tokenId`
* `amount0Desired`：希望添加的`token0`数量
* `amount1Desired`：希望添加的`token1`数量
* `amount0Min`：最少添加的`token0`数量（防止被frontrun）
* `amount1Min`：最少添加的`token1`数量（防止被frontrun）
* `deadline`：截止时间（超过该时间后请求无效）（防止重放攻击）

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

首先根据`tokenId`获取头寸信息；与[mint](#mint)方法一样，这里调用[addLiquidity](#addLiquidity)添加流动性，返回添加成功的流动性`liquidity`，所消耗的`amount0`和`amount1`，以及交易对合约`pool`。

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

根据`pool`对象里的最新头寸信息，更新本合约的头寸状态，比如`token0`和`token1`的可取回代币数`tokensOwed0`和`tokensOwed1`，以及头寸当前流动性等。

> `tokensOwed0`, `tokensOwed1`, `feeGrowthInside0LastX128`，`feeGrowthInside1LastX128`和`liquidity`这几个数据在`pool.positions`方法都能够获取到，不知道这里为什么要再算一次。

#### decreaseLiquidity

移除流动性，可以移除部分或者所有流动性，移除后的代币将以待取回代币形式记录，需要再次调用[collect](#collect)方法取回代币。

参数如下：

* `tokenId`：创建头寸时返回的`tokenId`，即NFT的`tokenId`
* `liquidity`：希望移除的流动性数量
* `amount0Min`：最少移除的`token0`数量（防止被frontrun）
* `amount1Min`：最少移除的`token1`数量（防止被frontrun）
* `deadline`：截止时间（超过该时间请求无效）（防止重放攻击）

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

注意，这里使用`isAuthorizedForToken` modifer：

```solidity
modifier isAuthorizedForToken(uint256 tokenId) {
    require(_isApprovedOrOwner(msg.sender, tokenId), 'Not approved');
    _;
}
```

确认当前用户具备操作该`tokenId`的权限，否则禁止移除。

```solidity
    require(params.liquidity > 0);
    Position storage position = _positions[params.tokenId];

    uint128 positionLiquidity = position.liquidity;
    require(positionLiquidity >= params.liquidity);
```

确认头寸流动性大于等于待移除流动性。

```solidity
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
    (amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity);

    require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

调用[Uniswap-v3-core](https://hackmd.io/TDPPCAIgRRqVDPwsSm6Kfw)的[burn](https://hackmd.io/TDPPCAIgRRqVDPwsSm6Kfw#burn)方法销毁流动性，返回该流动性对应的`token0`和`token1`的代币数量`amount0`和`amount1`，确认其符合`amount0Min`和`amount1Min`的限制。

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

与[increaseLiquidity](#increaseLiquidity)相同，此处计算头寸的待取回代币等信息。

#### burn

销毁头寸NFT。仅当该头寸的流动性为0，并且待取回代币数量都是0时，才能销毁NFT。

同样，调用该方法需要验证当前用户拥有`tokenId`的权限。

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

取回待领取代币。

参数如下：

* `tokenId`：创建头寸时返回的`tokenId`，即NFT的`tokenId`
* `recipient`：代币接收者
* `amount0Max`：最多领取的`token0`代币数量
* `amount1Max`：最多领取的`token1`代币数量

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

获取待取回代币数量。

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

如果该头寸含有流动性，则触发一次头寸状态的更新，这里使用`burn` 0流动性来触发。这是因为Uniswap-v3-core只在`mint`和`burn`时才更新头寸状态，而`collect`方法可能在`swap`之后被调用，可能会导致头寸状态不是最新的。

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

调用Uniswap-v3-core的`collect`方法取回代币，并更新头寸的待取回代币数量。

### SwapRouter.sol

交换代币，包括以下几个方法：

* [exactInputSingle](#exactInputSingle)：单步交换，指定输入代币数量，尽可能多地获得输出代币
* [exactInput](#exactInput)：多步交换，指定输入代币数量，尽可能多地获得输出代币
* [exactOutputSingle](#exactOutputSingle)：单步交换，指定输出代币数量，尽可能少地提供输入代币
* [exactOutput](#exactOutput)：多步交换，指定输出代币数量，尽可能少地提供输入代币

另外，该合约也实现了：

* [uniswapV3SwapCallback](#uniswapV3SwapCallback)：交换回调方法
* [exactInputInternal](#exactInputInternal)：单步交换，内部方法，指定输入代币数量，尽可能多地获得输出代币
* [exactOutputInternal](#exactOutputInternal)：单步交换，内部方法，指定输出代币数量，尽可能少地提供输入代币

#### exactInputSingle

单步交换，指定输入代币数量，尽可能多地获得输出代币。

参数如下：

* `tokenIn`：输入代币地址
* `tokenOut`：输出代币地址
* `fee`：手续费等级
* `recipient`：输出代币接收者
* `deadline`：截止时间，超过该时间请求无效
* `amountIn`：输入的代币数量
* `amountOutMinimum`：最少收到的输出代币数量
* `sqrtPriceLimitX96`：（最高或最低）限制价格

返回：

* `amountOut`：输出代币数量

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

该方法实际上调用[exactInputInternal](#exactInputInternal)，最后确认输出代币数量`amountOut`符合最小输出代币要求`amountOutMinimum`。

注意，`SwapCallbackData`中的`path`按照[Path.sol](#Pathsol)中定义的格式编码。

#### exactInput

多步交换，指定输入代币数量，尽可能多地获得输出代币。

参数如下：

* `path`：交换路径，格式请参考：[Path.sol](#Pathsol)
* `recipient`：输出代币收款人
* `deadline`：交易截止时间
* `amountIn`：输入代币数量
* `amountOutMinimum`：最少输出代币数量

返回：

* `amountOut`：输出代币

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

在多步交换中，需要按照交换路径，拆成多个单步交换，循环进行，直到路径结束。

如果是第一步交换，则`payer`为合约调用方，否则，`payer`为当前`SwapRouter`合约。

在循环中首先根据[hasMultiplePools](#hasMultiplePools)判断路径`path`中是否剩余2个及以上的池子。如果有，则中间交换步骤的收款地址设置为当前`SwapRouter`合约，否则设置为入口参数`recipient`。

每一步交换后，将当前交换路径`path`的前20+3个字节删除，即弹出（pop）最前面的token+fee信息，进入下一次交换，并将每一步交换的输出作为下一次交换的输入。

每一步交换调用[exactInputInternal](#exactInputInternal)进行。

多步交换后，确认最后的`amountOut`满足最小输出代币要求`amountOutMinimum`。

#### exactOutputSingle

单步交换，指定输出代币数量，尽可能少地提供输入代币。

参数如下：

* `tokenIn`：输入代币地址
* `tokenOut`：输出代币地址
* `fee`：手续费等级
* `recipient`：输出代币收款人
* `deadline`：请求截止时间
* `amountOut`：输出代币数量
* `amountInMaximum`：最大输入代币数量
* `sqrtPriceLimitX96`：最大或最小代币价格

返回：

* `amountIn`：实际输入代币数量

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

调用[exactOutputInternal](#exactOutputInternal)完成单步交换，并确认实际输入代币数量`amountIn`小于等于最大输入代币数量`amountInMaximum`。

#### exactOutput

多步交换，指定输出代币数量，尽可能少地提供输入代币。

参数如下：

* `path`：交换路径，格式请参考：[Path.sol](#Path)
* `recipient`：输出代币收款人
* `deadline`：请求截止时间
* `amountOut`：指定输出代币数量
* `amountInMaximum`：最大输入代币数量

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

调用[exactOutputInternal](#exactOutputInternal)完成交换，注意，该方法会在回调方法中继续完成下一步交换，因此不需要像[exactInput](#exactInput)使用循环交易。

最后确认实际输入代币数量`amountIn`小于等于最大输入代币数量`amountInMaximum`。

#### exactInputInternal

单步交换，内部方法，指定输入代币数量，尽可能多地获得输出代币。

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

如果没有指定`recipient`，则默认为当前`SwapRouter`合约地址。这是因为在多步交换时，需要将中间代币保存在当前`SwapRouter`合约。

```solidity
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
```

根据[decodeFirstPool](#decodeFirstPool)解析`path`中第一个池子的信息。

```solidity
    bool zeroForOne = tokenIn < tokenOut;
```

因为Uniswap v3池子`token0`地址小于`token1`，根据两个代币地址判断当前是否由`token0`交换到`token1`。注意，`tokenIn`可以是`token0`或`token1`。

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

调用[swap](#swap)方法，获得完成本次交换所需的`amount0`和`amount1`。如果是从`token0`交换`token1`，则`amount1`是负数；反之，`amount0`是负数。

如果没有指定`sqrtPriceLimitX96`，则默认为最低或最高价格，因为在多步交换中，无法指定每一步的价格。

```solidity
    return uint256(-(zeroForOne ? amount1 : amount0));
```

返回`amountOut`。

#### exactOutputInternal

单步交换，内部方法，指定输出代币数量，尽可能少地提供输入代币。

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

这部分代码与[exactInputInternal](#exactInputInternal)类似。

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

调用[Uniswap-v3-core](https://hackmd.io/TDPPCAIgRRqVDPwsSm6Kfw)的`swap`方法完成单步交换，注意，因为是指定输出代币数量，此处需要使用`-amountOut.toInt256()`。
返回的`amount0Delta`和`amount1Delta`为完成本次交换所需的`token0`数量和实际输出的`token1`数量。

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

swap的回调方法，实现`IUniswapV3SwapCallback.uniswapV3SwapCallback`接口。

参数如下：

* `amount0Delta`：本次交换产生的`amount0`（对应代币为`token0`）；对于合约而言，如果大于0，则表示应输入代币；如果小于0，则表示应收到代币
* `amount1Delta`：本次交换产生的`amount1`（对应代币为`token1`）；对于合约而言，如果大于0，则表示应输入代币；如果小于0，则表示应收到代币
* `_data`：回调参数，这里为`SwapCallbackData`类型

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

解析回调参数`_data`，根据[decodeFirstPool](#decodeFirstPool)获得交易路径上的第一个交易对信息。

```solidity
    (bool isExactInput, uint256 amountToPay) =
        amount0Delta > 0
            ? (tokenIn < tokenOut, uint256(amount0Delta))
            : (tokenOut < tokenIn, uint256(amount1Delta));
```

根据不同输入，有以下几种交易组合：

|场景|说明|amount0Delta > 0|amount1Delta > 0|tokenIn < tokenOut|isExactInput|amountToPay|
|---|---|---|---|---|---|---|
|1|输入指定数量`token0`，输出尽可能多`token1`|true|false|true|true|amount0Delta|
|2|输入尽可能少`token0`，输出指定数量`token1`|true|false|true|false|amount0Delta|
|3|输入指定数量`token1`，输出尽可能多`token0`|false|true|false|true|amount1Delta|
|4|输入尽可能少`token1`，输出指定数量`token0`|false|true|false|false|amount1Delta|

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

* 如果`isExactInput`，即指定输入代币的场景，上表中的场景1和场景3，则直接向`SwapRouter`合约转账`amount0Delta`（场景1）或`amount1Delta`（场景3）（都是正数）。
* 如果是指定输出代币的场景
    - 如果是多步交换，则移除前23的字符（pop最前面的token+fee），将需要的输入作为下一步的输出，进入下一步交换
    - 如果是单步交换（或最后一步），则`tokenIn`与`tokenOut`交换，并向`SwapRouter`合约转账

### LiquidityManagement.sol

#### uniswapV3MintCallback

添加流动性的回调方法。

参数如下：

* `amount0Owed`：应转账的`token0`数量
* `amount1Owed`：应转账的`token1`数量
* `data`：在`mint`方法中传入的回调参数

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

首先反向解析回调参数`MintCallbackData`，并确认该方法是被指定的交易对合约调用，因为该方法是一个`external`方法，可以被外部调用，因此需要确认调用方。

最后，向调用方转入指定的代币数量。

#### addLiquidity

给已初始化的交易对（池子）添加流动性。

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

根据`factory`、`token0`、`token1`和`fee`获取交易对`pool`。

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

从`slot0`获取当前价格`sqrtPriceX96`，根据`tickLower`和`tickUpper`计算区间的最低价格`sqrtRatioAX96`和最高价格`sqrtRatioBX96`。

根据[getLiquidityForAmounts](#getLiquidityForAmounts)计算能够获得的最大流动性。

```solidity
    (amount0, amount1) = pool.mint(
        params.recipient,
        params.tickLower,
        params.tickUpper,
        liquidity,
        abi.encode(MintCallbackData({poolKey: poolKey, payer: msg.sender}))
    );
```

使用[Uniswap-v3-core](https://hackmd.io/TDPPCAIgRRqVDPwsSm6Kfw)的`mint`方法添加流动性，并返回实际消耗的`amount0`和`amount1`。

我们在Uniswap-v3-core的`mint`方法中提到，调用方需实现[uniswapV3MintCallback](#LiquidityManagement.uniswapV3MintCallback)接口。这里传入`MintCallbackData`作为回调参数，在`uniswapV3MintCallback`方法中可以反向解析出来，以便获取交易对和用户信息。

```solidity
require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

最后，确认实际消耗的`amount0`和`amount1`满足`amount0Min`和`amount1Min`的最低要求。

### LiquidityAmounts.sol

#### getLiquidityForAmount0

根据`amount0`和价格区间计算流动性。

根据Uniswap-v3-core的`getAmount0Delta`中的公式：

$$
amount0 = x_b - x_a = L \cdot (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}}) = L \cdot (\frac{\sqrt{P_a} - \sqrt{P_b}}{\sqrt{P_a} \cdot \sqrt{P_b}})
$$

可得：

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

根据`amount1`和价格区间计算流动性。

根据Uniswap-v3-core的`getAmount1Delta`公式：

$$
amount1 = y_b - y_a = L \cdot \Delta{\sqrt{P}} = L \cdot (\sqrt{P_b} - \sqrt{P_a})
$$

可得：

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

根据当前价格，计算能够返回的最大流动性。

* 因为当$\sqrt{P}$增大时，需要消耗$x$，因此如果当前价格低于价格区间低点时，需要完全根据$x$即`amount0`计算流动性
* 反之，如果当前价格高于价格区间高点，需要根据$y$即`amount1`计算流动性

如下图所示：

$$
p,...,\overbrace{p_a,...,p_b}^{amount0}
$$

$$
\overbrace{p_a,...}^{amount1},p,\overbrace{...,p_b}^{amount0}
$$

$$
\overbrace{p_a,...,p_b}^{amount1},...,p
$$

其中，$p$表示当前价格，$p_a$表示区间低点，$p_b$表示区间高点。

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

在Uniswap v3 [SwapRouter](#SwapRoutersol)中，交易路径被编码为一个`bytes`类型字符串，其格式为：

$$
\overbrace{token_0}^{20}\overbrace{fee_0}^{3}\overbrace{token_1}^{20}\overbrace{fee_1}^{3}\overbrace{token_2}^{20}...
$$

其中，$token_n$的长度为20个字节（bytes），$fee_n$的长度为3个字节，上述路径表示：从`token0`交换到`token1`，使用手续费等级为`fee0`的池子（`token0`、`token1`、`fee0`），继续交换到`token2`，使用手续费等级为`fee1`的池子（`token1`、`token2`、`fee1`）。

交易路径`path`示例如下：

$$
0x\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{token0,20bytes}\overbrace{0001f4}^{fee,3bytes}\overbrace{68b3465833fb72a70ecdf485e0e4c7bd8665fc45}^{token1,20bytes}
$$

#### hasMultiplePools

判断交易路径是否经过多个池子（2个及以上）。

```solidity
/// @notice Returns true iff the path contains two or more pools
/// @param path The encoded swap path
/// @return True if path contains two or more pools, otherwise false
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

我们从上述路径编码可知，如果经过2个池子，至少包含3个代币，则路径长度至少需要$20+3+20+3+20=66$个字节。代码中`MULTIPLE_POOLS_MIN_LENGTH`即等于66。

#### numPools

计算路径中的池子数量。

算法为：

$$
num = \frac{length - 20}{20 + 3}
$$

```solidity
/// @notice Returns the number of pools in the path
/// @param path The encoded swap path
/// @return The number of pools in the path
function numPools(bytes memory path) internal pure returns (uint256) {
    // Ignore the first token address. From then on every fee and token offset indicates a pool.
    return ((path.length - ADDR_SIZE) / NEXT_OFFSET);
}
```

#### decodeFirstPool

解析第一个path的信息，包括`token0`，`token1`和`fee`。

分别返回字符串中0-19子串（`token0`，转`address`类型），20-22子串（`fee`，转`uint24`类型），和23-42子串（`token1`，转`address`类型）。请参考`BytesLib.sol`的[toAddress](#toAddress)和[toUint24](#toUint24)方法。

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

返回第一个池子的路径，即返回前43（即20+3+20）个字符组成的子字符串。

```solidity
/// @notice Gets the segment corresponding to the first pool in the path
/// @param path The bytes encoded swap path
/// @return The segment containing all data necessary to target the first pool in the path
function getFirstPool(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(0, POP_OFFSET);
}
```

#### skipToken

跳过当前路径上的第一个`token+fee`，即跳过前20+3个字符。

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

从字符串的指定序号起，读取一个地址（20个字符）：

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

因为变量`_bytes`类型为`bytes`，根据[ABI定义](https://docs.soliditylang.org/en/develop/abi-spec.html)，`bytes`的第一个32字节存储字符串的长度（length），因此需要先跳过前面32字节，即`add(_bytes, 0x20)`；`add(add(_bytes, 0x20), _start)`表示定位到字符串指定序号`_start`；`mload`读取从该序号起的32个字节，因为`address`类型只有20字节，因此需要`div 0x1000000000000000000000000`，即右移12字节。

假设`_strat = 0`，`_bytes`的分布如下图所示：

$$
0x\overbrace{0000000...2b}^{length,32bytes}\underbrace{\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{address, 20 bytes}\overbrace{0001f468b3465833fb72a70e}^{div,12 bytes}}_{mload, 32bytes}cdf485e0e4c7bd8665fc45
$$

#### toUint24

从字符串的指定序号起，读取一个`uint24`（24位，即3个字符）：

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

因为`_bytes`前32个字符表示字符串长度；`mload`读取32字节，可以确保从`_start`开始的3个字节在读取出来的32字节的最低位，赋值给类型为`uint24`的变量将只保留最低位的3个字节。

假设`_strat = 0`，`_bytes`的分布如下图所示：

$$
0x\overbrace{000000}^{0x3+\_start}\underbrace{0...2b\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{address1,20bytes}\overbrace{0001f4}^{fee,3bytes}}_{mload,32bytes}\overbrace{68b3465833fb72a70ecdf485e0e4c7bd8665fc45}^{address2,20bytes}
$$

### OracleLibrary.sol

根据白皮书公式5.3-5.5，计算$t_1$至$t_2$时间内的几何平均价格如下：

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{\sum^{t_2}_{i=t_1} \log_{1.0001}(P_i)}{t_2 - t_1} \tag{5.3}
$$

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1} \tag{5.4}
$$

$$
P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2 - t_1}} \tag{5.5}
$$

本合约提供价格预言机相关方法，包括如下方法：

* [consult](#consult)：查询从一段时间前到现在的几何平均价格（以`tick`形式）
* [getQuoteAtTick](#getQuoteAtTick)：根据`tick`计算代币价格

#### consult

查询从一段时间前到现在的几何平均价格（以`tick`形式）。

参数如下：

* `pool`: 交易对池子地址
* `period`：以秒计数的区间

返回：

* `timeWeightedAverageTick`：时间加权平均价格

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

构造两个监测点，第一个为`period`时间之前，第二个为现在。

```solidity
    (int56[] memory tickCumulatives, ) = IUniswapV3Pool(pool).observe(secondAgos);
    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
```

根据`IUniswapV3Pool.observe`方法获取累积`tick`，即公式5.4中的$a_{t_2}$和$a_{t_1}$。

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

如果`tickCumulativesDelta`为负数，并且无法被`period`整除，则将平均价格-1。

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

根据Uniswap-v3-core的`getSqrtRatioAtTick`方法计算`tick`对应的$\sqrt{P}$，即$\sqrt{\frac{token1}{token0}}$。

如果`baseToken < quoteToken`，则`baseToken`为`token0`，`quoteToken`为`token1`：

$$
quoteAmount = baseAmount \cdot (\sqrt{P})^2
$$

反之，`baseToken`为`token1`，`quoteToken`为`token0`：

$$
quoteAmount = \frac{baseAmount}{(\sqrt{P})^2}
$$
