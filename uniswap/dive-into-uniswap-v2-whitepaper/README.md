[English](./README.md) | [中文](./README_zh.md)

# Deep Dive into Uniswap v2 Whitepaper

## Preface

This article, as the first in the "Understanding Uniswap" series, begins with the Uniswap v2 whitepaper, explaining the design philosophy and the mathematical formula derivation process of the Uniswap v2 protocol.

There are already many articles explaining Uniswap on the internet. So, why write another one?

The initial reason was to record my personal summaries during the learning process of Uniswap. These summaries are not just translations; they are more about extending the knowledge points of the whitepaper, deriving mathematical formulas, and reflecting on the engineering implementation of contract codes, most of which are not elaborated in the original whitepaper.

Although Uniswap v3 has been out for a while, learning v2 is still fundamental to understanding v3; moreover, due to [the licensing restrictions of v3](https://uniswap.org/blog/uniswap-v3#license), many AMM projects on other EVM chains have mostly forked the v2 code, making it essential to thoroughly understand Uniswap v2.

Furthermore, as a foundational protocol in DeFi, Uniswap's industry status, theoretical foundation, and engineering implementation are classic examples in DeFi. For those looking to delve into DeFi or smart contract programming, Uniswap v2 serves as an excellent introductory material.

I hope this article helps you understand Uniswap v2 a bit better. Given my limited expertise, there may be errors in the text, and I welcome corrections.

The following text will explain according to the [Uniswap v2 Whitepaper](https://uniswap.org/whitepaper.pdf) chapter structure, with extended readings and mathematical formula derivation processes for key knowledge points explained in the form of comments.

# Uniswap v2 Core

## 1 Introduction

Uniswap v1 is a smart contract system on the Ethereum blockchain, implementing the AMM (Automated Market Maker) protocol based on the $x \cdot y = k$ formula. Each Uniswap v1 pair pool contains two tokens, ensuring the product of the two token balances does not decrease during liquidity provision. Traders pay a 0.3% fee for each transaction to the liquidity providers. The v1 contracts are non-upgradable.

Uniswap v2 is a new version based on the same formula, containing many anticipated features. One of the most important features is the support for any ERC20 token pairs, unlike v1, which only supports ERC20 and ETH pairs. Moreover, v2 offers a price oracle functionality, accumulating the relative price of the two tokens at the beginning of each block. This will allow other Ethereum contracts to obtain the time-weighted average price of any two tokens over any period; finally, v2 introduces "flash swaps," allowing users to borrow and use tokens freely on-chain, requiring only that these tokens be returned at the end of the transaction along with a fee.

Although the v2 contracts are also non-upgradable, they support modifying a variable in the factory contract to allow the Uniswap protocol to charge a 0.05% fee (i.e., $\frac{1}{6}$ of 0.3%) for each transaction. This fee is initially off but can be activated in the future, after which liquidity providers will only receive a 0.25% fee, not 0.3%.

> Note: Because 0.05% goes to the protocol.
> 
> Regarding the 0.05% protocol fee switch, it later led to a liquidity war between Sushiswap and Uniswap. Sushiswap forked Uniswap's code, [allocating the 0.05% protocol fee to SUSHI holders](https://docs.sushi.com/products/yield-farming/the-sushibar#what-is-xsushi), at one point threatening to take away Uniswap's liquidity; and ultimately forcing Uniswap to issue its token, UNI.

In section three, we will introduce how Uniswap v2 fixed some minor issues of Uniswap v1 while restructuring the contract implementation. By minimizing the logic of core contracts (that hold liquidity funds), the risk of Uniswap being attacked was reduced, making the system more upgradable.

This article discusses the structure of the core contracts and the factory contracts used to initialize pair contracts. In reality, using Uniswap v2 requires calling pair contracts through a router contract, which will help calculate the amount of tokens to transfer to pair contracts during trading and liquidity provision.

# 2 New Features

### 2.1 ERC-20 Pairs

Uniswap v1 used ETH as a bridge token, meaning every pair included ETH as one of its tokens. This simplified routing—for example, to trade between ABC and XYZ, one would use the ETH/ABC and ETH/XYZ pairs, reducing liquidity fragmentation.

> Note: As pairs always included ETH, compared to v2's any ERC-20 token combinations, v1 had significantly fewer pairs, concentrating liquidity on the ETH side.

However, this imposed significant costs on liquidity providers. All providers were exposed to ETH's price volatility, incurring impermanent loss when the relative price of their tokens shifted against ETH.

> Note: Impermanent loss, caused by the $x \cdot y = k$ formula, means market makers could end up with a lower total value of tokens after price movements. For an explanation of impermanent loss, see [Binance Academy's article](https://academy.binance.com/en/articles/impermanent-loss-explained).

For related tokens, like USD stablecoins, trading pairs like ABC/XYZ would incur less impermanent loss than ABC/ETH or XYZ/ETH.

> Note: Since ABC and XYZ prices move similarly against ETH, the ABC/XYZ price fluctuates less, compared to the larger fluctuations of ABC/ETH or XYZ/ETH.

Using ETH as a mandatory trading token also increased transaction costs. Directly using an ABC/XYZ pair would save on transaction fees and slippage compared to going through ETH.

> Note: In v1, trading from ABC to XYZ required trading ABC/ETH then ETH/XYZ, doubling fees and slippage.

Uniswap v2 allows liquidity providers to create contracts for any two ERC-20 tokens, increasing the number of potential trading pairs. While this makes finding the best trading path more complex, routing can be solved at a higher layer (e.g., via off-chain or on-chain routers or aggregators).

### 2.2 Price Oracle

The marginal price (excluding fees) provided by Uniswap at time $t$ can be determined by dividing the quantities of tokens a and b:

$$
p_t = \frac {r^a_t}{r^b_t} \quad \text{(1)}
$$

When Uniswap's price is incorrect, arbitrageurs can profit (paying enough in fees), meaning Uniswap's token prices follow market prices. This implies Uniswap prices can serve as a rough price oracle.

> Note: Arbitrage opportunities arise from price differences in the same token across markets, encouraging arbitrageurs to align Uniswap market prices with those of centralized exchanges or other DEXes.

However, Uniswap v1 couldn't provide a secure on-chain oracle as its price could be easily manipulated. If a contract used the current ETH-DAI price for derivative trading, an attacker could manipulate the price by buying ETH from the ETH-DAI pair, trigger derivative contract liquidations, then sell the ETH back, all in one atomic transaction or arranged by miners within a single block.

> Note: The instantaneity of the price means it can be easily manipulated by large buy/sell orders. Samczsun detailed such an attack [in a blog post](https://samczsun.com/taking-undercollateralized-loans-for-fun-and-for-profit/).

Uniswap v2 improves the oracle function by recording prices at the first transaction of each block (equivalent to after the last transaction of the previous block), making price manipulation harder. If an attacker manipulates the price at the end of a block, other arbitrageurs could correct it within the same block unless the attacker, filling the block with transactions, mines two consecutive blocks, they have no special advantage for arbitrage.

> Note: Since the oracle records prices once per block, manipulation would require control over all transactions in two consecutive blocks, showing v2's oracle is not robust. Improvements in v3 address this.

Uniswap v2 implements the oracle by recording cumulative prices before the first transaction of each block. Prices are time-weighted (based on the time since the last price update). This means, at any point, the cumulative price is the sum of every second's spot price in the contract's history.

$$
a_t = \sum_{i=1}^t p_i \quad \text{(2)}
$$

To estimate the time-weighted average price (TWAP) between $t_1$ and $t_2$, callers can record cumulative prices at $t_1$ and $t_2$, subtract $t_2$'s price from $t_1$'s, and divide by the time difference (note: the contract doesn't store historical cumulative prices, so callers must invoke the contract at the interval's start, read, and save the current price).

$$
p_{t_1,t_2} = \frac{\sum_{i=t_1}^{t_2} p_i}{t_2 - t_1} = \frac{\sum_{i=1}^{t_2} p_i - \sum_{i=1}^{t_1} p_i}{t_2 - t_1} = \frac {a_{t_2} - a_{t_1}}{t_2 - t_1} \quad \text{(3)}
$$

Oracle users can choose their interval. A longer interval means higher costs for attackers to manipulate the TWAP, but the average price will deviate more from the real-time price.

> Note: The arithmetic mean calculation in formula (3) is straightforward. However, note that since the contract only records the current cumulative price, applications must track and save historical prices themselves. The TWAP calculation in Uniswap v2 uses an (weighted) arithmetic mean, where understanding different types of means and their applications is essential.
> 
> In mathematics, the concept of Pythagorean means includes three classic types: arithmetic, geometric, and harmonic means.
> 
> * Arithmetic Mean
>
> $$ A(x_1,...,x_n) = \frac{1}{n}(x_1 + ... + x_n) $$
>
> * Geometric Mean
>
> $$ G(x_1,...,x_n) = \sqrt[n]{x_1 ... x_n} $$
>
> * Harmonic Mean
>
> $$ H(x_1,...,x_n) = \frac{n}{\frac{1}{x_1} + ... + \frac{1}{x_n}} $$
> 
> * For positive $x$, the relationship:
> 
> $$ A(x_1,...,x_n) \geq G(x_1,...,x_n) \geq H(x_1,...,x_n) $$
> 
> The arithmetic mean is the most common, easy to calculate but sensitive to outliers. The geometric mean is more suited to financial markets, which exhibit Brownian motion. The harmonic mean is more affected by small values and is generally used for average rates.
> 
> For Uniswap prices, the geometric mean would be more appropriate, but due to its complexity on Ethereum contracts, v2 opted for the arithmetic mean; however, v3 uses the geometric mean for its oracle.
> 
> In section 3.4, Uniswap v2 uses the geometric mean to calculate the initial liquidity token amount.

A dilemma arises: should we calculate the price of token A in terms of token B, or vice versa? While the spot price of B in terms of A (B/A) and A in terms of B (A/B) are always reciprocals in real-time, their arithmetic means over a period are not reciprocals. For instance, if the price at block 1 is 100 USD/ETH (B being USD, A being ETH) and at block 2 is 300 USD/ETH, the average price is 200 USD/ETH, but the average price of ETH/USD would be 1/150 ETH/USD. Since the contract cannot know which token users will use as the pricing unit, Uniswap v2 records prices for both tokens.

> Note: The calculation of average prices for the two token pricing scenarios is as follows:
> 
> * Priced in USD
>
> $$ A(x_1, x_2) = \frac{100 + 300}{2} = 200 \text{ USD/ETH} $$
>
> * Priced in ETH
>
> $$ A(\frac{1}{x_1}, \frac{1}{x_2}) = \frac{\frac{1}{100} + \frac{1}{300}}{2} = \frac{1}{150} \text{ ETH/USD} $$

Another issue is that users can directly send tokens to the pair contract (changing the token balances and affecting the price) without a transaction, bypassing the oracle update mechanism.

> Note: Since the oracle price needs updating before the first transaction of a block, if there's no trade, the oracle update is bypassed.

If the contract simply checks its balances and uses the current balance to update the oracle price, an attacker could manipulate the oracle price by sending tokens to the contract just before the first transaction of a block. If the last trade happened $x$ seconds ago in a previous block, the contract would incorrectly accumulate the manipulated new price multiplied by $x$, even though no trades occurred at that price.

> Example: If two tokens A and B had balances of 100 and 200 respectively after the last trade in a previous block, with the price of B in terms of A being $\frac{200}{100}=2$ , the price to be accumulated after $x$ seconds would be $2x$. However, if an attacker sent 100 B tokens to the contract just before the first trade of the next block, the price would become $\frac{200}{200}=1$ , and the contract would incorrectly accumulate at $1x$.

To prevent this, the core contract caches the token balances after each interaction, updating the oracle price using the cached (not real-time) balances. Besides preventing oracle price manipulation, this adjustment also led to a re-architecture of the contract structure, detailed in section 3.2.

#### 2.2.1 Precision

Due to Solidity's lack of support for non-integer data types, Uniswap v2 employs a simple binary fixed-point scheme for encoding and operating on prices. Specifically, prices at any moment are stored in a UQ112.112 format, meaning both sides of the decimal point have 112 bits of precision, unsigned. This format covers a range of $[0, 2^{112}-1]$, with a precision of $\frac{1}{2^{112}}$.

> Note: The theoretical maximum value for UQ112.112 is $2^{112}-\frac{1}{2^{112}}$, but since Uniswap v2 divides two uint112 values to arrive at a UQ112.112, its maximum value is $2^{112}-1$.

Choosing the UQ112.112 format was a practical consideration, as these numbers can be represented by a uint224 variable (occupying 224 bits, or 28 bytes), leaving 32 bits available in a 256-bit (bit) storage slot (EVM storage slots are 256 bits). For cached token balance variables, each token's balance can use a uint112 variable, also leaving 32 bits available in a 256-bit storage slot when declared. These spare spaces are used for cumulative operations. Specifically, the token balance is saved alongside the timestamp of the most recent trade block, modulo $2^{32}$, ensuring it can be represented with 32 bits. Additionally, while any moment's price (in UQ112.112 format) fits within 224 bits, the cumulative price over time does not. The spare 32 bits at the end of the storage slot are used to store overflow data from repeated price accumulations. This design means the price oracle only adds 3 SSTORE operations (currently costing 15,000 gas) per block's first transaction.

> This careful design demonstrates Uniswap's emphasis on minimizing user impact while implementing useful features. To avoid additional transaction costs from updating the oracle with every trade, Uniswap v2 updates only before the first transaction of each block.
> 
> Each token balance uses uint112 representation, and the timestamp uses 32 bits, totaling 256 bits, perfectly occupying one storage slot. Fewer storage slots mean less gas consumed during interactions, reducing user operation costs. The cumulative price uses 256 bits for representation.

The main drawback of this design is that 32 bits cannot ensure the timestamp will never overflow. In fact, the Unix timestamp will overflow the 32-bit maximum value on 02/07/2106. To ensure the system can operate normally after this date and after each 32-bit overflow cycle (approximately every 136 years), the oracle must be queried at least once per cycle. Since the core method for cumulative calculation is overflow-safe, it means even if transactions span a timestamp overflow, they can still be accurately accumulated, provided the oracle uses the correct overflow algorithm for checking the time interval.

> The focus here is on ensuring that the average price from the oracle can be correctly calculated across the timestamp overflow boundary. Assuming near the year 2106, with $t_1,t_2,t_3$ representing consecutive block times, where $t_1$ doesn't overflow (missing 1 second) and $t_2$, $t_3$ do, it can be shown that even after overflow, $t_2-t_1$ still calculates the correct time difference (3 seconds); similarly, even after cumulative price $a_{t_3}$ overflows, as long as the caller saved $a_{t_1}$, the correct difference can be calculated. $p_{t_1,t_3}$, the average price from $t_1$ to $t_3$, is calculated according to formula (3) as follows:
> 
> $uint32.max = 2^{32} - 1 = 4,294,967,295$
> 
> $t_1 = 4,294,967,294$
> 
> $t_2 = 4,294,967,297 \% 4,294,967,296 = 1$
> 
> $t_3 = 4,294,967,301 \% 4,294,967,296 = 5$
> 
> $\Delta{t_1,t_2} = 1 - 4,294,967,294 = -4,294,967,293 = 3$
> 
> $\Delta{t_2,t_3} = 5 - 1 = 4$
> 
> $\Delta{t_1,t_3} = 5 - 4,294,967,294 = -4,294,967,289 = 7$
> 
> $p_{t_1} = 100$
> 
> $p_{t_2} = 110$
> 
> $a_{t_1} = 1000$
> 
> $a_{t_2} = a_{t_1} + p_{t_1} * \Delta{t_1,t_2} = 1000 + 100 * 3 = 1300$
> 
> $a_{t_3} = a_{t_2} + p_{t_2} * \Delta{t_2,t_3} = 1300 + 110 * 4 = 1740$
> 
> $$ p_{t_1,t_3} = \frac {a_{t_3} - a_{t_1}}{t_3 - t_1} = \frac {\Delta{a_{t_1},a_{t_3}}} {\Delta{t_1,t_3}} = \frac {740}{7} = 105.71 $$

### 2.3 Flash Swaps

In Uniswap v1, users who wanted to buy ABC with XYZ had to first send XYZ to the contract to receive ABC. This was inconvenient for users who wished to use ABC to buy XYZ elsewhere, like during arbitrage opportunities with other contracts, or those looking to sell collateral to release their positions in Maker or Compound and repay Uniswap loans.

Uniswap v2 introduced a feature allowing users to receive and use tokens before paying for them, as long as they complete payment within the same transaction. The swap method calls an optional user-specified callback contract between transferring out tokens and checking the k value. Once the callback is complete, the Uniswap contract checks the current token balance and confirms it satisfies the k condition (after deducting fees). If the contract lacks sufficient balance, the entire transaction is rolled back.

Users can return the original tokens without performing a swap. This feature enables anyone to flash borrow any amount of tokens from Uniswap pools (flash loan fees are the same as transaction fees, both 0.30%).

> Note: Flash loans are highly useful in the DeFi space, allowing protocols with high TVL to earn fee income. Protocols like dYdX and Aave have introduced flash loan features. Uniswap v2's flash loan functionality actually uses the same swap method as its trading function.

### 2.4 Protocol Fee

Uniswap v2 includes a 0.05% protocol fee toggle. If activated, this fee is sent to a contract's feeTo address.

By default, no feeTo address is set, so no protocol fee is collected. A predefined feeToSetter address can call the Uniswap v2 factory contract's setFeeTo method to change the feeTo address. The feeToSetter can also call setFeeToSetter to change the contract's feeToSetter address.

If a feeTo address is set, the protocol begins collecting a 5 basis point (0.05%) fee, meaning 1/6 of the 30 basis point (0.30%) fee collected from liquidity providers is allocated to the protocol. This means traders continue to pay a 0.30% transaction fee, with 83.3% (5/6) of the fee (0.25% of the entire transaction) allocated to liquidity providers and the remaining 16.6% (1/6 of the fee, 0.05% of the transaction) allocated to the feeTo address.

Collecting a 0.05% fee on every transaction would incur additional gas costs. To avoid this, cumulative fees are only collected when providing or destroying liquidity. The contract calculates cumulative fees and mints new liquidity tokens for the fee beneficiary (feeTo address) when liquidity tokens are minted or destroyed.

The total cumulative fee can be calculated by the growth in $\sqrt{k}$ (i.e., $\sqrt{x \cdot y}$) priced since the last fee collection. The cumulative fee from $t_1$ to $t_2$ as a percentage of the liquidity at time $t_2$ is:

$$
f_{1,2} = 1 - \frac{\sqrt{k_1}}{\sqrt{k_2}} \quad \text{(4)}
$$

> This calculation might seem complex, but the rationale is as follows: If the number of liquidity tokens minted during the initial provision is calculated based on $\sqrt{x \cdot y}$, at different times $t_1$ and $t_2$, the liquidity token amount remains equal to $\sqrt{x_1 \cdot y_1}$ and $\sqrt{x_2 \cdot y_2}$, respectively. The growth part is hence the fee, thus deriving formula (4):
> 
> $l_1 = \sqrt{x_1 \cdot y_1}$
> 
> $l_2 = \sqrt{x_2 \cdot y_2}$
> 
> $fee = l_2 - l_1$
> 
> $$
> f_{1,2} = \frac{l_2 - l_1}{l_2} = 1 - \frac{\sqrt{x_1 \cdot y_1}}{\sqrt{x_2 \cdot y_2}} = 1 - \frac{\sqrt{k_1}}{\sqrt{k_2}}
> $$

If the protocol fee was activated before $t_1$, then between $t_1$ and $t_2$, the feeTo address should receive $\frac{1}{6}$ of the fees as the protocol fee. Thus, new liquidity tokens equal to $\frac{1}{6} \cdot f_{1,2}$ are minted for the feeTo address.

Assuming the liquidity tokens corresponding to the protocol fee are $s_m$, and $s_1$ is the liquidity token amount at time $t_1$, the following equation holds:

$$
\frac{s_m}{s_m + s_1} = \phi \cdot f_{1,2} \quad \text{(5)}
$$

Substituting $f_{1,2}$ from formula (4), $s_m$ is calculated as:

$$
s_m = \frac{\sqrt{k_2} - \sqrt{k_1}}{(\frac{1}{\phi} - 1) \cdot \sqrt{k_2} + \sqrt{k_1}} \cdot s_1 \quad \text{(6)}
$$

Substituting $\frac{1}{6}$ for the proportion, we get:

$$
s_m = \frac{\sqrt{k_2} - \sqrt{k_1}}{5 \cdot \sqrt{k_2} + \sqrt{k_1}} \cdot s_1 \quad \text{(7)}
$$

For instance, if the initial liquidity provider deposits 100 DAI and 1 ETH, receiving 10 liquidity tokens, and later (assuming no other liquidity providers), when the feeTo wishes to collect protocol fees, the token balances are 96 DAI and 1.5 ETH, substituting into formula (7) yields:

$$
s_m = \frac{\sqrt{1.5 \cdot 96} - \sqrt{1 \cdot 100}}{5 \cdot \sqrt{1.5 \cdot 96} + \sqrt{1 \cdot 100}} \cdot 10 \approx 0.0286 \quad \text{(8)}
$$

> When there's only swap activity (no mint/burn of liquidity), the pool's $k$ value increases due to fee sedimentation. Since the total amount of liquidity tokens (shares) remains constant, but the token balances in the pool keep increasing, as shown, $k_1 = 100 DAI \cdot 1 ETH = 100, k_2 = 96 DAI \cdot 1.5 ETH = 144, k_2 > k_1$.

### 2.5 Meta Transactions for Pool Shares

Uniswap v2 pool shares (i.e., liquidity tokens) natively support meta transactions. This means users can authorize third parties to transfer their liquidity tokens without initiating on-chain transactions themselves. Anyone can submit the user's signature via the permit method, paying gas costs, and perform other actions in the same transaction.

> This type of meta transaction, where third parties initiate on-chain transactions on behalf of users via offline signatures, is particularly useful in scenarios where the user's wallet lacks ETH for gas fees.
> 
> In Uniswap v2's core contracts, the signing feature is used for transferring liquidity tokens; this functionality is utilized in the peripheral router contract, as v2 separates contracts into core (essential swap/mint/burn functionalities) and periphery (external applications). Applications typically interact directly with the periphery contract, reducing user interaction with the core contract to a single offline-signed transaction for liquidity removal. The signing feature also facilitates integration with other contracts.
> 
> This involves EIP-712 and EIP-2612, which we'll detail in another article. Briefly, EIP-712 defines a way to sign structured data, addressing security concerns associated with signing a mere hash without understanding its content, such as inadvertently authorizing token transfers to malicious contracts. EIP-712 allows checking specific signature content, like transfer amounts and deadlines. EIP-2612, still under review, specifies the Solidity encoding standard for the permit method using EIP-712, introduced after Uniswap v2 and currently in the review stage.

## 3 Other Changes

### 3.1 Solidity

Uniswap v1 was implemented in Vyper, a Python-like language for smart contracts. Uniswap v2 is written in the more popular Solidity language, due to dependencies on capabilities not available in Vyper at development time, such as parsing non-standard ERC-20 token return values and accessing new opcodes through inline assembly, like chainid.

### 3.2 Contract Re-architecture

A design focus of Uniswap v2 was minimizing the external interface scope and complexity of the core pair contracts (where liquidity providers' token assets are stored). Any bug found in the core contracts could be catastrophic, potentially leading to millions of dollars in liquidity assets being stolen or frozen.

When assessing the security of core contracts, the primary concern is whether they can protect liquidity providers' assets from theft or freezing. Any functionalities that enhance or protect traders, rather than allowing basic asset exchanges in the pool, should be extracted to the router contract.

> The core contract retains only the most basic and important functions to ensure security, as all liquidity assets are stored within. The fewer the lines of code and the less change, the lower the probability of bugs. The core contract's core code is only about 200 lines.

In fact, some of the swap function's code could also be moved to the router contract. As mentioned, Uniswap v2 saves the last balance of each token (to prevent manipulation of the oracle mechanism). The new architecture further simplifies this compared to Uniswap v1.

In Uniswap v2, the seller sends tokens to the core contract before executing the swap method. The contract determines how many tokens were received by comparing the cached balance to the current balance. This means the core contract cannot know how the trader sent the tokens. In fact, they could use offline signed meta transactions or other future mechanisms for authorizing ERC-20 token transfers, not just the transferFrom method.

#### 3.2.1 Adjustment for Fee

Uniswap v1 implemented trading fees by reducing the amount of tokens deposited into the contract, subtracting 0.3% of the transaction fee before comparing the constant k function. The contract implicitly enforced the following constraint:

$$
(x_1 - 0.003 \cdot x_{in}) \cdot y_1 \geq x_0 \cdot y_0 \quad \text{(9)}
$$

> The two token balances, after deducting the trading fee, satisfy the constant k function.

With the introduction of flash loans in Uniswap v2, it's possible that both xin and yin are non-zero (when a user wants to return borrowed tokens instead of trading). To address fee calculation in this scenario, the contract enforces the following constraint:

$$
(x_1 - 0.003 \cdot x_{in}) \cdot (y_1 - 0.003 \cdot y_{in}) \geq x_0 \cdot y_0 \quad \text{(10)}
$$

> Uniswap's swap method can support both flash loans and trading. When borrowing both x and y tokens via a flash loan, a 0.3% fee is deducted from both, and then the remaining balances must satisfy the k constant.

To simplify on-chain computation, both sides of formula (10) are multiplied by 1,000,000, resulting in:

$$
(1000 \cdot x_1 - 3 \cdot x_{in}) \cdot (1000 \cdot y_1 - 3 \cdot y_{in}) \geq 1000000 \cdot x_0 \cdot y_0 \quad \text{(11)}
$$

> Since Solidity does not support floating-point numbers, scaling up simplifies the computation.

#### 3.2.2 sync() and skim()

To address potential modifications to pair contract balances by custom tokens and to elegantly solve issues with tokens exceeding $2^{112}$, Uniswap v2 provides two methods: sync() and skim().

sync() can be a recovery measure when a token undergoes asynchronous contraction. In such scenarios, trades would get suboptimal exchange rates, and if no liquidity providers correct this state, the pair would struggle to operate. sync() sets the contract's cached token balances to the current actual balances, helping the system recover.

skim() is a recovery measure for when sending a large amount of tokens causes a pair's token balance to overflow (exceeding the maximum value of uint112). It allows any user to withdraw the excess tokens (the difference between the actual token balance and $2^{112}-1$).

> In summary, due to external factors (not caused by Uniswap), the cached balances in the pair contract might not match the actual balances. sync() updates the cached balance to match the actual balance, and skim() updates the actual balance to match the cached balance, ensuring the system continues to operate. Anyone can execute these methods. Alpha Leak: If someone mistakenly transfers tokens into the contract, anyone can withdraw these tokens.

### 3.3 Handling Non-standard and Unusual Tokens

The ERC-20 standard requires transfer() and transferFrom() to return a boolean indicating success. However, some tokens, like USDT and BNB, do not return a value for these methods (or one of them). Uniswap v1 treated the absence of a return value as a failure, leading to transaction reverts and failures.

> Further reading:
> [EIP-20: ERC-20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
> [USDT Contract Address](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code)
> [BNB Contract Address](https://etherscan.io/address/0xB8c77482e45F1F44dE1745F52C74426C631bDD52#code)

Uniswap v2 addresses non-standard ERC-20 implementations differently. If a transfer() method does not return a value, Uniswap v2 assumes it indicates success, not failure. This change does not affect tokens adhering to the standard ERC-20 (as their transfer() methods return a value).

Also, Uniswap v1 assumed transfer() and transferFrom() could not trigger reentrant calls into the pair contract. This assumption conflicts with some ERC-20 tokens, including those supporting ERC-777 standard hooks. To fully support these tokens, Uniswap v2 introduces a "lock" mechanism to solve reentrancy issues for all publicly mutable state methods, also addressing custom callback reentrancy in flash loans.

> The lock is essentially a Solidity modifier, controlled by an unlock variable, to manage synchronous locking.

### 3.4 Initialization of Liquidity Token Supply

When a new liquidity provider deposits tokens into an existing Uniswap pair, the number of newly minted liquidity tokens is calculated based on the current token amounts:

$$
s_{minted} = \frac{x_{deposited}}{x_{starting}} \cdot s_{starting} \quad \text{(12)}
$$

> Since liquidity tokens are a type of ERC-20 token, holding a certain number of liquidity tokens represents a share of the pool's token balance. Therefore, for existing pairs with existing liquidity tokens, the ratio of the deposited token value to the total value should equal the ratio of the obtained liquidity token amount to the total amount:
> 
> $$
> \frac{s_{minted}}{s_{starting}} = \frac{x_{deposited}}{x_{starting}}
> $$

But what if they are the first liquidity provider? In this case, $x_{starting}$ is 0, making the above formula inapplicable.

Uniswap v1 set the initial liquidity token amount equal to the amount of ETH deposited (in wei). This made sense because if the initial liquidity was provided at the correct price, then 1 liquidity share (as ETH is a token with 18 decimals) would represent about 2ETH in value.

> Because providing liquidity in Uniswap v1/v2 requires depositing equal values of both tokens, if the share is equivalent to the ETH amount, then 1 share necessitates depositing 1ETH, and at the correct price, the other token's value would also be 1ETH, thus the total value of 1 liquidity share is 2ETH.

However, this meant the value of liquidity shares depended on the price ratio at which the initial liquidity was provided, which could be manipulated. Moreover, as Uniswap v2 supports any token pair, many will not include ETH.

Unlike v1, Uniswap v2 dictates that the number of liquidity tokens minted for the first provider equals the geometric mean of the deposited token amounts:

$$
s_{minted} = \sqrt{x_{deposited} \cdot {y_{deposited}}} \quad \text{(13)}
$$

This formula ensures that at any moment, the value of a liquidity share is independent of the price ratio of the deposited tokens. For instance, if the current price of 1 ABC is 100 XYZ, and the first deposit is 2 ABC and 200 XYZ (matching the 1:100 ratio), the provider receives $\sqrt{2 \cdot 200}=20$ liquidity tokens. These tokens represent the value of 2 ABC and 200 XYZ, along with any accumulated fees.

If the first deposit is 2 ABC and 800 XYZ (a 1:400 ratio), the provider receives $\sqrt{2 \cdot 800}=40$ liquidity tokens.

The formula ensures that the value of 1 liquidity share (token) will not be less than the geometric mean of the pool's token balances. Then, the value of a liquidity token could continuously grow, for instance, through accumulated trading fees or by others "donating" tokens to the pool.

Theoretically, it's possible for the value of the smallest liquidity token unit ($10^{-18}$, i.e., 1 wei) to become so high that it prevents other (smaller) liquidity providers from joining.

To address this, Uniswap v2 destroys the first minted $10^{-15}$ liquidity tokens (1,000 times the smallest unit). This loss is negligible for most pairs but significantly raises the cost of initial minting attacks. To raise the price of each liquidity token to $100, an attacker would need to donate $100,000 of tokens to the pool, which would then be permanently locked as liquidity.

> The initial minting attack involves an attacker being the first to add liquidity by depositing the minimum unit ($10^{-18}$, i.e., 1 wei) of liquidity, such as 1 wei ABC and 1 wei XYZ, thus minting 1 wei of liquidity tokens ($\sqrt{1}$); then, in the same transaction, the attacker continues to transfer (not mint) 1 million ABC and 1 million XYZ into the pool, followed by calling the sync() method to update the cached balances. Now, 1 wei of liquidity tokens is valued at $1 million + (10^{-18})$ ABC and $1 million + (10^{-18})$ XYZ, making it prohibitively expensive for other liquidity providers to participate.
> 
> By destroying the first minted 1,000 wei of liquidity tokens, an attacker aiming to raise each token's price to $100 must at least mint $1,000 + 1 = 1,001$ liquidity tokens, totaling $1,001 \cdot 100 = 100,100$, where $100,000 would be permanently destroyed, significantly increasing the attacker's cost.

### 3.5 Wrapping ETH - WETH

The interfaces for trading with Ethereum's native token, ETH, differ from those for ERC-20 tokens. As a result, many Ethereum protocols do not support ETH directly but use a wrapped version of ETH (WETH) that complies with the ERC-20 standard.

Uniswap v1 was an exception. Since every Uniswap v1 pair involved ETH as one of the trading tokens, it made sense to directly support ETH trades for gas efficiency.

With Uniswap v2's support for any ERC-20 token pairs, there's no need to support native ETH trades. Adding such support would double the core contract code size and split liquidity between ETH and WETH pairs. Native ETH must be wrapped into WETH before trading on Uniswap v2.

> Note: In fact, Uniswap v2's core contract does not support native ETH, but the periphery contract still facilitates native ETH trades by automatically converting ETH to WETH before interacting with the core contract. This reflects Uniswap's development principle of keeping the core contract simplified and addressing application logic and user experience through the periphery contract.

### 3.6 Deterministic Pair Addresses

Like Uniswap v1, all Uniswap v2 pair contracts are initialized by a unified factory contract. In Uniswap v1, these contracts were created using the CREATE opcode, meaning their addresses depended on the order of creation. Uniswap v2 uses Ethereum's new CREATE2 opcode to generate pair contracts with deterministic addresses. This means the addresses of the pair contracts can be computed off-chain without querying the on-chain state.

> Note: For more on CREATE vs. CREATE2 usage, see the [Solidity documentation](https://docs.soliditylang.org/en/develop/yul.html#yul-object).
> CREATE2 originates from [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014).
>
> If pair addresses were indeterminate, it would mean users wishing to trade ABC for XYZ would need to query the on-chain contract interface for the ABC/XYZ pair address before trading. The contract would also need to store mappings of token pairs to their trading addresses. Deterministic addresses allow frontend pages or apps to calculate the contract address using a specified algorithm, avoiding on-chain queries.

### 3.7 Maximum Token Balance

To more efficiently implement the oracle function, Uniswap v2 supports a maximum cached token balance of $2^{112}-1$. This figure is large enough to accommodate tokens with total supplies in the quadrillions with 18 decimal places.

Should the balance of any token exceed this maximum, calls to the swap method will fail (due to checks in the _update() method). To recover from such a situation, anyone can call the skim() method to remove excess tokens from the pool.

## References

* [1] Hayden Adams. 2018. url: https://hackmd.io/@477aQ9OrQTCbVR3fq1Qzxg/HJ9jLsfTz?type=view.
* [2] Guillermo Angeris et al. An analysis of Uniswap markets. 2019. arXiv: 1911.03380 [q-fin.TR].
* [3] samczsun. Taking undercollateralized loans for fun and for profit. Sept. 2019. url: https://samczsun.com/taking-undercollateralized-loans-for-fun-and-for-profit/.
* [4] Fabian Vogelsteller and Vitalik Buterin. Nov. 2015. url: https://eips.ethereum.org/EIPS/eip-20.
* [5] Jordi Baylina Jacques Dafflon and Thomas Shababi. EIP 777: ERC777 Token Standard. Nov. 2017. url: https://eips.ethereum.org/EIPS/eip-777.
* [6] Radar. WTF is WETH? url: https://weth.io/.
* [7] Uniswap.info. Wrapped Ether (WETH). url: https://uniswap.info/token/0xc02aaa39b223fe8d0a0e5c4f27e
* [8] Vitalik Buterin. EIP 1014: Skinny CREATE2. Apr. 2018. url: https://eips.ethereum.org/EIPS/eip-1014.
