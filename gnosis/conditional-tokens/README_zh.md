[English](./README.md) | [中文](./README_zh.md)

# Conditional Tokens 深度解析

[Conditional Tokens](https://conditional-tokens.readthedocs.io/en/latest/index.html) (条件代币) 是 Gnosis 团队开发的一套基于 [ERC-1155](https://ethereum.org/developers/docs/standards/tokens/erc-1155/) 标准的智能合约框架。它的核心目的是为了支持复杂的“组合预测市场” (Combinatorial Prediction Markets)，并解决传统预测市场在处理嵌套条件时面临的流动性分割问题。

本文将详细说明这套系统的设计初衷和运作机制。

## Conditional Tokens 设计原理

### 组合预测市场的“路径依赖”问题

在预测市场中，我们不仅关心单一事件（如“谁赢得选举？”），往往还关心连锁事件（如“如果 A 赢得选举，股市是涨还是跌？”）。这被称为组合预测市场（Combinatorial Prediction Markets）。

假设有两个独立的预测问题：
* 问题 A（选举）：结果是 Alice 还是 Bob？
* 问题 B（经济）：结果是 High（涨） 还是 Low（跌）？

我们要预测的是：“Alice 当选 且 经济上涨”的情况。

**传统方法的局限性**

在旧的合约设计中，要实现这种组合，需要对代币进行层层嵌套（Nest）。但这会产生一个严重的问题：路径依赖（Path Dependence）。
* 路径 1：你先抵押资金生成代表“Alice”的代币，然后再将“Alice代币”作为抵押品，去生成“Alice & High”的代币。
![](./assets/path_1.png)

* 路径 2：你先抵押资金生成代表“High”的代币，然后再将“High代币”作为抵押品，去生成“High & Alice”的代币。
![](./assets/path_2.png)

虽然逻辑上 Alice & High 和 High & Alice 代表的是完全相同的现实结果，但在区块链上，它们是两个完全不同的代币。
![](./assets/path_3.png)

这导致了以下问题：
* 流动性割裂：看好同一个结果的人，因为生成的顺序不同，持有的代币无法互通，无法在同一个池子里交易。
* 用户体验差：用户如果不按照特定顺序操作，就无法合并手中的头寸。

### 设计目标：路径无关性（可互换性）

为了解决上述痛点，Conditional Tokens 的设计目标非常明确：实现路径无关性 (Path Independence)，即可互换性 (Fungibility)。
系统必须保证：无论用户是先选 A 再选 B，还是先选 B 再选 A，只要最终的条件集合是一样的，生成的代币 ID (Token ID) 必须是完全相同的。

用数学语言表达，我们需要满足交换律（Commutativity）：

$$Token(A + B) = Token(B + A)$$

### 为什么不使用简单的异或 / 加法？

要实现交换律，我们需要选择一种数学运算来生成代币 ID。

**位运算异或 (XOR) 或 普通加法**

最容易想到的方案是使用异或（XOR）或模加法。
假设 A 的特征值是 Hash(A)，B 的特征值是 Hash(B)。
计算代币 ID = Hash(A) ^ Hash(B)。
由于异或满足交换律，A ^ B 等于 B ^ A。这看起来很完美且计算速度极快。

但是由于简单算法（异或、加法）是线性的，而在密码学中，线性关系容易受到广义生日攻击 (Generalized Birthday Attack / Wagner's Algorithm) 的威胁。

**攻击逻辑如下：**

1. 攻击者想要伪造一个高价值的 ID（由条件 A 和 B 生成）。
2. 攻击者可以极低成本创建大量的“垃圾条件”（C, D, E, F...）。
3. 由于运算是线性的，攻击者可以通过算法高效地在垃圾条件中找到一组组合（例如 C+D+E），使得它们的异或结果恰好等于 A+B 的结果。即发生了哈希碰撞。攻击者用垃圾条件的代币，骗过了合约，提走了属于 A+B 条件的真实抵押品。

### 最终方案：椭圆曲线 (Elliptic Curve)

为了同时满足“交换律”和“抗碰撞安全”，Gnosis 选择了 椭圆曲线密码学 (ECC) 中的点加法运算。

**为什么选择椭圆曲线？**

**满足交换律：**

在椭圆曲线上，两个点 $P_1$ 和 $P_2$ 相加，几何定义决定了 $P_1 + P_2 = P_2 + P_1$。这意味着无论条件组合的顺序如何，计算出的最终点（以及对应的 ID）是唯一的。这解决了流动性割裂问题。因此，不论用户以何种顺序组合条件，生成的代币 ID 都是相同的。

**非线性与安全性：**

椭圆曲线的点加法涉及复杂的几何切线运算，这是一种高度非线性的操作。
它破坏了简单的线性关系，使得上述的“广义生日攻击”变得极度困难，几乎不可行。
黑客无法通过简单的计算凑出两个垃圾条件来碰撞出真实条件的 ID。这保证了它是抗碰撞安全的。

**ID 生成流程**

在代码层面，代币 ID 的生成逻辑如下：

1. 将每个基本条件映射为椭圆曲线上的一个点。
2. 当条件组合时，执行椭圆曲线点加法。
3. 将最终得到的点进行无损编码，生成 Collection ID。

## Conditional Tokens 核心概念

### Condition 条件事件

在 Conditional Tokens 框架中，条件事件（Condition）是预测市场的核心单元。条件事件是一个预设多个备选结果的问题（Question），通常由预言机（Oracle）在未来某个时刻揭晓结果（Outcome）。

### Outcome Collection 结果集合

结果集合（Outcome Collection）是指在特定条件事件下，所有可能结果的组合。不包括空集和全集，因为空集没有实际意义，全集则表示所有结果都发生，也没有预测价值。

比如，对于一个有三个选项的条件事件（A、B、C），其结果集合包括：

* {A}：表示结果为 A；
* {B}：表示结果为 B；
* {C}：表示结果为 C；
* {A, B}：表示结果为 A 或 B；
* {A, C}：表示结果为 A 或 C；
* {B, C}：表示结果为 B 或 C。

#### Index Set 索引集合

可以使用 `Index Set` 来表示结果集合，其中每个结果对应一个二进制位，1 表示包含该结果，0 表示不包含。从低位到高位依次对应结果 A、B、C。例如：
* {A} 对应的 Index Set 为 0b001 = 1
* {B} 对应的 Index Set 为 0b010 = 2
* {A, B} 对应的 Index Set 为 0b011 = 3

#### Collection ID 集合 ID

Collection ID 表示结果集合的唯一 ID，使用 32 字节表示。

将 Condition ID 和 Index Set 进行 Hash 后，将结果映射到椭圆曲线上的一个点，即为该条件事件选定的结果集的 Collection ID。

针对不同条件事件的结果集合，可以分别计算单个条件事件结果集合的 Collection ID，然后通过椭圆曲线点加法，计算出组合条件事件结果集合的 Collection ID：
1. 映射 (Map): 将 Condition ID + Index Set 进行 Hash，确定性地映射到椭圆曲线上的一个点 $P_{new}$。
1. 解码 (Decode): 如果存在 parentCollectionId，将其从 bytes32 解码回椭圆曲线上的点 $P_{parent}$。
1. 加法 (Add): 执行椭圆曲线加法：$P_{result} = P_{parent} + P_{new}$。因为椭圆曲线加法满足交换律，所以无论条件事件的组合顺序如何，最终的 $P_{result}$ 都是相同的。
1. 编码 (Encode): 将最终的点 $P_{result}$ 进行无损压缩编码（取 x 坐标 + y 坐标的奇偶校验位，压缩进 32 字节），直接作为新的 Collection ID 返回。

### Position 头寸

在 Conditional Tokens 框架中，用户通过持有 ERC-1155 标准的条件代币来表示其在特定头寸上的权益，其中，Position ID 即作为 ERC-1155 代币的 Token ID。

头寸（Position）包括以下两个要素：

* 抵押品代币（Collateral Token）：用户用于进入市场的 ERC-20 基础代币，如 USDC 等。
* 结果集合（Outcome Collection）：用户选择的预测结果。
  * 如果需要表示组合条件事件的结果集合，则先按照 [Collection ID](#collection-id-集合-id) 的方式计算出组合事件的 Collection ID。

头寸 ID（Position ID）可通过对抵押品代币地址和结果集合的 Collection ID 进行 Hash 运算得到。

## 智能合约的实现逻辑

Conditional Tokens 的智能合约主要由 `ConditionalTokens.sol` 实现。它通过以下几个核心操作来管理预测市场的生命周期：

* 准备条件事件 (Prepare Condition)
* 揭晓事件结果 (Resolve Condition)
* 分割头寸 (Split Position)
* 合并头寸 (Merge Position)
* 赎回头寸 (Redeem Position)

### 准备条件事件 Preparing a Condition

每个条件事件由以下参数定义：

* oracle 地址：负责决议该条件结果的预言机。
* questionId 问题 ID：唯一标识该问题的哈希值。由调用者设置，它可以是 IPFS 哈希或其他唯一标识符。
* outcomeSlotCount：该条件可能的结果数量（例如三选一为3），不超过 256。

```solidity
// @dev This function prepares a condition by initializing a payout vector associated with the condition.
/// @param oracle The account assigned to report the result for the prepared condition.
/// @param questionId An identifier for the question to be answered by the oracle.
/// @param outcomeSlotCount The number of outcome slots which should be used for this condition. Must not exceed 256.
function prepareCondition(address oracle, bytes32 questionId, uint outcomeSlotCount) external {
    // Limit of 256 because we use a partition array that is a number of 256 bits.
    require(outcomeSlotCount <= 256, "too many outcome slots");
    require(outcomeSlotCount > 1, "there should be more than one outcome slot");
    bytes32 conditionId = CTHelpers.getConditionId(oracle, questionId, outcomeSlotCount);
    require(payoutNumerators[conditionId].length == 0, "condition already prepared");
    payoutNumerators[conditionId] = new uint[](outcomeSlotCount);
    emit ConditionPreparation(conditionId, oracle, questionId, outcomeSlotCount);
}
```

`prepareCondition` 方法通过“预言机地址” + “问题 ID” + “结果数量”三者的哈希值，生成唯一的条件事件 ID，即 conditionId。

`payoutNumerators` 是一个映射，存储了每个 conditionId 对应的结果赔付比例数组，初始时为空数组。比如，对于一个有 3 个结果的 Condition，其 `payoutNumerators` 为 [1, 1, 0]，表示前两个结果各赔付 50%，第三个结果赔付 0。可通过 `payoutNumerators[conditionId].length` 来判断该 Condition 是否已经初始化。

### 揭晓事件结果 Resolving a Condition

在事件结束以后，预言机会调用 `reportPayouts` 方法，公布该条件的最终结果。

```solidity
/// @dev Called by the oracle for reporting results of conditions. Will set the payout vector for the condition with the ID ``keccak256(abi.encodePacked(oracle, questionId, outcomeSlotCount))``, where oracle is the message sender, questionId is one of the parameters of this function, and outcomeSlotCount is the length of the payouts parameter, which contains the payoutNumerators for each outcome slot of the condition.
/// @param questionId The question ID the oracle is answering for
/// @param payouts The oracle's answer
function reportPayouts(bytes32 questionId, uint[] calldata payouts) external {
    uint outcomeSlotCount = payouts.length;
    require(outcomeSlotCount > 1, "there should be more than one outcome slot");
    // IMPORTANT, the oracle is enforced to be the sender because it's part of the hash.
    bytes32 conditionId = CTHelpers.getConditionId(msg.sender, questionId, outcomeSlotCount);
    require(payoutNumerators[conditionId].length == outcomeSlotCount, "condition not prepared or found");
    require(payoutDenominator[conditionId] == 0, "payout denominator already set");

    uint den = 0;
    for (uint i = 0; i < outcomeSlotCount; i++) {
        uint num = payouts[i];
        den = den.add(num);

        require(payoutNumerators[conditionId][i] == 0, "payout numerator already set");
        payoutNumerators[conditionId][i] = num;
    }
    require(den > 0, "payout is all zeroes");
    payoutDenominator[conditionId] = den;
    emit ConditionResolution(conditionId, msg.sender, questionId, outcomeSlotCount, payoutNumerators[conditionId]);
}
```

`reportPayouts` 方法要求调用者必须是预言机地址（msg.sender），并且传入的 `payouts` 数组长度必须与之前准备的条件事件的结果数量一致。

在[准备条件事件](#准备条件事件-preparing-a-condition)中，我们知道一个条件事件由 `oracle 地址 + questionId + outcomeSlotCount` 唯一确定。因此，这里通过传入 `questionId`，结合 `msg.sender`（即 oracle 地址） 和 `payouts.length`（即 outcomeSlotCount），可以重新计算出 `conditionId`。

`payoutNumerators` 数组会被更新为预言机公布的结果赔付比例（分子），而 `payoutDenominator` 则被设置为所有赔付比例之和（分母），确保后续结算时可以正确计算每个结果的实际赔付金额。

### Split Position 分割头寸

这是进入市场的操作。

分为两种情形：
1. 用户锁定抵押品，一般是 USDC 等稳定币，铸造出一组互斥的结果代币。
2. 用户锁定一组结果代币，铸造出更细分的子结果代币。比如，将“Alice 或 Bob”代币拆分为“Alice”和“Bob”两个代币。

分割前后的头寸的价值应相等，根据概率守恒（所有可能结果概率之和为 1）：
1. 对于情形1，将 1 个单位的抵押品拆分为所有可能结果的代币。
2. 对于情形2，将 1 个单位的结果集合代币拆分为其子集合的代币，因为子集合的概率之和等于父集合的概率。

![](./assets/split_1.png)

```solidity
/// @dev This function splits a position. If splitting from the collateral, this contract will attempt to transfer `amount` collateral from the message sender to itself. Otherwise, this contract will burn `amount` stake held by the message sender in the position being split worth of EIP 1155 tokens. Regardless, if successful, `amount` stake will be minted in the split target positions. If any of the transfers, mints, or burns fail, the transaction will revert. The transaction will also revert if the given partition is trivial, invalid, or refers to more slots than the condition is prepared with.
/// @param collateralToken The address of the positions' backing collateral token.
/// @param parentCollectionId The ID of the outcome collections common to the position being split and the split target positions. May be null, in which only the collateral is shared.
/// @param conditionId The ID of the condition to split on.
/// @param partition An array of disjoint index sets representing a nontrivial partition of the outcome slots of the given condition. E.g. A|B and C but not A|B and B|C (is not disjoint). Each element's a number which, together with the condition, represents the outcome collection. E.g. 0b110 is A|B, 0b010 is B, etc.
/// @param amount The amount of collateral or stake to split.
function splitPosition(
    IERC20 collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint[] calldata partition,
    uint amount
) external {
    require(partition.length > 1, "got empty or singleton partition");
    uint outcomeSlotCount = payoutNumerators[conditionId].length;
    require(outcomeSlotCount > 0, "condition not prepared yet");

    // For a condition with 4 outcomes fullIndexSet's 0b1111; for 5 it's 0b11111...
    uint fullIndexSet = (1 << outcomeSlotCount) - 1;
    // freeIndexSet starts as the full collection
    uint freeIndexSet = fullIndexSet;
    // This loop checks that all condition sets are disjoint (the same outcome is not part of more than 1 set)
    uint[] memory positionIds = new uint[](partition.length);
    uint[] memory amounts = new uint[](partition.length);
    for (uint i = 0; i < partition.length; i++) {
        uint indexSet = partition[i];
        require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
        require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
        freeIndexSet ^= indexSet;
        positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
        amounts[i] = amount;
    }

    if (freeIndexSet == 0) {
        // Partitioning the full set of outcomes for the condition in this branch
        if (parentCollectionId == bytes32(0)) {
            require(collateralToken.transferFrom(msg.sender, address(this), amount), "could not receive collateral tokens");
        } else {
            _burn(
                msg.sender,
                CTHelpers.getPositionId(collateralToken, parentCollectionId),
                amount
            );
        }
    } else {
        // Partitioning a subset of outcomes for the condition in this branch.
        // For example, for a condition with three outcomes A, B, and C, this branch
        // allows the splitting of a position $:(A|C) to positions $:(A) and $:(C).
        _burn(
            msg.sender,
            CTHelpers.getPositionId(collateralToken,
                CTHelpers.getCollectionId(parentCollectionId, conditionId, fullIndexSet ^ freeIndexSet)),
            amount
        );
    }

    _batchMint(
        msg.sender,
        // position ID is the ERC 1155 token ID
        positionIds,
        amounts,
        ""
    );
    emit PositionSplit(msg.sender, collateralToken, parentCollectionId, conditionId, partition, amount);
}
```

方法 `splitPosition` 接受以下参数：
* `collateralToken`：抵押品代币地址。
* `parentCollectionId`：父结果集合 ID，如果是从抵押品拆分，则为 null。
* `conditionId`：要拆分的条件事件 ID。
* `partition`：一个数组，表示要拆分成的子结果集合的索引集合 `Index Set`。
* `amount`：要拆分的数量。

```solidity
// For a condition with 4 outcomes fullIndexSet's 0b1111; for 5 it's 0b11111...
uint fullIndexSet = (1 << outcomeSlotCount) - 1;
// freeIndexSet starts as the full collection
uint freeIndexSet = fullIndexSet;
```

上面这段代码计算了给定条件事件的所有可能结果的全集 `fullIndexSet`，对于 4 个结果的事件，`fullIndexSet = 0b1111`，并初始化了一个 `freeIndexSet`，用于跟踪哪些结果还未被分配到子集合中。

```solidity
for (uint i = 0; i < partition.length; i++) {
    uint indexSet = partition[i];
    require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
    require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
    freeIndexSet ^= indexSet;
    positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
    amounts[i] = amount;
}
```

这段代码遍历 `partition` 数组，验证每个子集合的合法性：
* 每个子集合必须非空且不等于全集，我们在[结果集合](#outcome-collection-结果集合)中提到过，空集和全集没有实际意义。
* 子集合之间必须互斥（disjoint），即没有重叠的结果。
  通过位运算 `indexSet & freeIndexSet` 来检查互斥性，并使用异或操作 `freeIndexSet ^= indexSet` 来更新剩余未分配的结果。
  同时，计算出每个子集合对应的头寸 ID（Position ID），并将拆分数量存入 `amounts` 数组。因为拆分前后的总价值必须相等，所以每个子集合的拆分数量都等于传入的 `amount`，因为每个子集合的概率之和等于父集合的概率。
* 如果 `freeIndexSet` 最终为 0，表示拆分的是全集：
  * 如果 `parentCollectionId` 为 null，表示是从抵押品拆分，则从用户账户扣除相应数量的抵押品代币。
  * 否则，从用户账户扣除相应数量的父结果集合代币。
* 如果 `freeIndexSet` 不为 0，表示拆分的是子集合，则从用户账户扣除相应数量的父结果集合代币。与全集拆分不同，这里需要计算出父集合的 Index Set，即 `fullIndexSet ^ freeIndexSet`，表示已被拆分的结果集合。

最后，调用 ERC-1155 合约的 `_batchMint` 方法，将拆分后的子结果集合代币铸造到用户账户，token ID 即为 Position ID。注意，每个子集合的数量均为 `amount`。

### Merge Positions 合并头寸

这是 Split 的逆操作。当用户集齐了所有可能结果的代币时，可以将它们合并回抵押品，或者合并成更大的结果集合代币。

![](./assets/merge_1.png)

```solidity
function mergePositions(
    IERC20 collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint[] calldata partition,
    uint amount
) external {
    require(partition.length > 1, "got empty or singleton partition");
    uint outcomeSlotCount = payoutNumerators[conditionId].length;
    require(outcomeSlotCount > 0, "condition not prepared yet");

    uint fullIndexSet = (1 << outcomeSlotCount) - 1;
    uint freeIndexSet = fullIndexSet;
    uint[] memory positionIds = new uint[](partition.length);
    uint[] memory amounts = new uint[](partition.length);
    for (uint i = 0; i < partition.length; i++) {
        uint indexSet = partition[i];
        require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
        require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
        freeIndexSet ^= indexSet;
        positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
        amounts[i] = amount;
    }
    _batchBurn(
        msg.sender,
        positionIds,
        amounts
    );

    if (freeIndexSet == 0) {
        if (parentCollectionId == bytes32(0)) {
            require(collateralToken.transfer(msg.sender, amount), "could not send collateral tokens");
        } else {
            _mint(
                msg.sender,
                CTHelpers.getPositionId(collateralToken, parentCollectionId),
                amount,
                ""
            );
        }
    } else {
        _mint(
            msg.sender,
            CTHelpers.getPositionId(collateralToken,
                CTHelpers.getCollectionId(parentCollectionId, conditionId, fullIndexSet ^ freeIndexSet)),
            amount,
            ""
        );
    }

    emit PositionsMerge(msg.sender, collateralToken, parentCollectionId, conditionId, partition, amount);
}
```

`mergePositions` 方法的参数和逻辑与 `splitPosition` 非常相似。主要区别在于：
* 先调用 `_batchBurn` 方法，从用户账户扣除所有子结果集合代币。
* 然后根据 `freeIndexSet` 的值，决定是将其合并回抵押品，还是合并成更大的结果集合代币。
  * 如果 `freeIndexSet` 为 0，表示合并的是全集：
    * 如果 `parentCollectionId` 为 null，表示合并回抵押品，则将相应数量的抵押品代币发送给用户。
    * 否则，铸造相应数量的父结果集合代币给用户。
  * 如果 `freeIndexSet` 不为 0，表示合并的是子集合，则铸造相应数量的父结果集合代币给用户。

### Redeem Positions 赎回头寸

这是在预言机（Oracle）公布结果后的结算操作。只有持有“正确结果”代币的用户才能兑换价值，错误结果的代币价值归零。

```solidity
function redeemPositions(IERC20 collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint[] calldata indexSets) external {
    uint den = payoutDenominator[conditionId];
    require(den > 0, "result for condition not received yet");
    uint outcomeSlotCount = payoutNumerators[conditionId].length;
    require(outcomeSlotCount > 0, "condition not prepared yet");

    uint totalPayout = 0;

    uint fullIndexSet = (1 << outcomeSlotCount) - 1;
    for (uint i = 0; i < indexSets.length; i++) {
        uint indexSet = indexSets[i];
        require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
        uint positionId = CTHelpers.getPositionId(collateralToken,
            CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));

        uint payoutNumerator = 0;
        for (uint j = 0; j < outcomeSlotCount; j++) {
            if (indexSet & (1 << j) != 0) {
                payoutNumerator = payoutNumerator.add(payoutNumerators[conditionId][j]);
            }
        }

        uint payoutStake = balanceOf(msg.sender, positionId);
        if (payoutStake > 0) {
            totalPayout = totalPayout.add(payoutStake.mul(payoutNumerator).div(den));
            _burn(msg.sender, positionId, payoutStake);
        }
    }

    if (totalPayout > 0) {
        if (parentCollectionId == bytes32(0)) {
            require(collateralToken.transfer(msg.sender, totalPayout), "could not transfer payout to message sender");
        } else {
            _mint(msg.sender, CTHelpers.getPositionId(collateralToken, parentCollectionId), totalPayout, "");
        }
    }
    emit PayoutRedemption(msg.sender, collateralToken, parentCollectionId, conditionId, indexSets, totalPayout);
}
```

`redeemPositions` 方法接受以下参数：
* `collateralToken`：抵押品代币地址。
* `parentCollectionId`：父结果集合 ID，如果是从抵押品赎回，则为 null。
* `conditionId`：要赎回的条件事件 ID。
* `indexSets`：一个数组，表示用户持有的结果集合的索引集合 `Index Set`。

方法逻辑如下：

首先，检查该条件事件是否已经揭晓结果（`payoutDenominator[conditionId] > 0`）。因为 `payoutDenominator` 只有在 [揭晓事件结果](#揭晓事件结果-resolving-a-condition) 时才会被设置。因此，如果为 0，表示结果尚未公布，无法赎回。

然后，遍历用户提供的 `indexSets` 数组，对于每个结果集合 `indexSet`：
* 验证其合法性（非空且不等于全集）。
* 计算该结果集合的赔付比例 `payoutNumerator`，即将包含在该集合中的所有结果的赔付比例相加。`indexSet & (1 << j) != 0` 表示该集合包含第 j 个结果，则将其赔付比例加入 `payoutNumerator`，如果该结果的赔付比例为 0，则不会影响总和。
  * 这里没有判断 indexSet 是否有效，因为在 [Split Position](#split-position-分割头寸) 和 [Merge Positions](#merge-positions-合并头寸) 中已经确保了用户只能持有合法的结果集合代币。
  * 如果用户尝试赎回一个不合法的 indexSet 或者其并不持有的 Position，那么在 `_burn` 时会失败，从而保证了安全性。
* 获取用户在该结果集合上的持仓数量 `payoutStake`。
  * 如果用户持有该结果集合的代币，则计算其应得的赔付金额，并将该结果集合的代币从用户账户中销毁。
  * 此处通过 `payoutStake.mul(payoutNumerator).div(den)` 计算用户在该结果集合上的实际赔付金额，即该结果集合的持仓数量乘以其赔付比例。

最后，如果用户有任何应得的赔付金额，则根据 `parentCollectionId` 的值，决定是将其赎回为抵押品代币，还是铸造为父结果集合代币。
* 如果 `parentCollectionId` 为 null，表示赎回为抵押品，则将相应数量的抵押品代币发送给用户。
* 否则，铸造相应数量的父结果集合代币给用户。

## 总结

Conditional Tokens 的核心价值在于它下沉到了数学底层来解决业务问题：

1. 为了支持复杂的组合预测，它必须支持条件的任意嵌套。
2. 为了避免嵌套带来的流动性分散，它必须保证路径无关性（使用满足交换律的算法）。
3. 为了防止算法被攻击者利用伪造代币，它放弃了简单的异或，选择了安全的椭圆曲线点加法。

这一整套设计，使得开发者可以在以太坊上构建深度流动性、可组合且安全的预测市场应用。

## 参考资料

* [Conditional Tokens Documentation](https://conditional-tokens.readthedocs.io/en/latest/)
* [Conditional Tokens Contracts](https://github.com/gnosis/conditional-tokens-contracts)
