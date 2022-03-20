# 深入理解 Uniswap v3 合约代码

## Tick

### TickMath

TickMath主要包含两个方法：

* `getSqrtRatioAtTick`：根据tick计算开根号价格$\sqrt{P}$
* `getTickAtSqrtRatio`：根据开根号价格$\sqrt{P}$计算tick


#### getSqrtRatioAtTick

该方法对应白皮书中的公式6.2:

$$
\sqrt{p}(i) = \sqrt{1.0001}^i = 1.0001^{\frac{i}{2}}
$$

其中，$i$即为tick。

因为Uniswap v3支持的价格（token1/token0）区间为$[2^{-128}, 2^{128}]$，根据白皮书公式6.1:

$$
p(i) = 1.0001^i
$$

因此，对应的最大tick（MAX_TICK）为：

$$
i = \lfloor log_{1.0001}{p(i)} \rfloor = \lfloor log_{1.0001}{2^{128}} \rfloor = \lfloor 887272.7517970635 \rfloor = 887272
$$

最小tick（MIN_TICK）为：

$$
i = \lceil log_{1.0001}{2^{-128}} \rceil = \lceil -887272.7517970635 \rceil = -887272
$$


假设$i$ $\geq 0$，对于一个给定的tick $i$，它总可以表示为二进制，因此以下式子总是成立：

$$
\begin{cases} i = \sum_{n=0}^{19}{(x_n \cdot 2^n)} = x_0 \cdot 1 + x_1 \cdot 2 + x_2 \cdot 4 + ... + x_{19}\cdot 524288 \\ x_n \in \{0, 1\} \end{cases} \tag{1.1}
$$

其中，$x_n$为$i$的二进制位。如$i=6$，其对应的二进制为：`000000000000000000000110`，则$x_1 = 1, x_2 = 1$，其余$x_n$均为0。

同样可以推出$i < 0$也可以用类似的公式表示。

我们先看$i < 0$的情况：

如果 $i < 0$，则：

$$
\sqrt{p}(i) = 1.0001^{\frac{i}{2}} = 1.0001^{-\frac{|i|}{2}} = \frac{1}{1.0001^{\frac{|i|}{2}}} = \frac{1}{1.0001^{\frac{1}{2}(\sum_{n=0}^{19}{(x_n \cdot 2^n)})}} \\ = \frac{1}{1.0001^{\frac{1}{2} \cdot x_0}} \cdot \frac{1}{1.0001^{\frac{2}{2} \cdot x_1}} \cdot \frac{1}{1.0001^{\frac{4}{2} \cdot x_2}} \cdot ... \cdot \frac{1}{1.0001^{\frac{524288}{2} \cdot x_{19}}}
$$

根据二进制位$x_n$的值，可以总结如下：

$$
\frac{1}{1.0001^{\frac{x_n \cdot 2^n}{2}}} \begin{cases} = 1 & \text{$x_n = 0, n \geq 0, i < 0$}\\
< 1 & \text{$x_n = 1, n \geq 0, i < 0$}
\end{cases}
$$

为了最小化精度误差，在计算过程中，使用`Q128.128`（128位定点数）表示中间价格，对于每一个价格$p$，均需要左移128位。由于$i < 0, x_n = 1$时，$\frac{1}{1.0001^{\frac{x_n \cdot 2^n}{2}}} < 1$，因此在连续乘积过程中不会有溢出问题。

可以总结计算$\sqrt{p}(i)$的方法：

* 初始值为1，从第0位开始，从低位到高位（从右往左）循环遍历$i$的二进制比特位
* 如果该位不为0，则乘以对应的$\frac{2^{128}}{1.0001^{\frac{2^n}{2}}}$，其中$2^{128}$表示左移128位
* 如果该位为0，则乘以1，可以省略


```solidity
/// @notice Calculates sqrt(1.0001^tick) * 2^96
/// @dev Throws if |tick| > max tick
/// @param tick The input tick for the above formula
/// @return sqrtPriceX96 A Fixed point Q64.96 number representing the sqrt of the ratio of the two assets (token1/token0)
/// at the given tick
function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) {
    uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));
    require(absTick <= uint256(MAX_TICK), 'T');

    // 如果第0位非0，则ratio = 0xfffcb933bd6fad37aa2d162d1a594001 ，即：2^128 / 1.0001^0.5
    uint256 ratio = absTick & 0x1 != 0 ? 0xfffcb933bd6fad37aa2d162d1a594001 : 0x100000000000000000000000000000000;
    // 如果第1位非0，则乘以 0xfff97272373d413259a46990580e213a ，即：2^128 / 1.0001^1，因为两个乘数均为Q128.128，最终结果多乘了2^128，因此需要右移128
    if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
    // 如果第2位非0，则乘以 0xfff2e50f5f656932ef12357cf3c7fdcc ，即：2^128 / 1.0001^2，
    if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
    // 以此类推
    if (absTick & 0x8 != 0) ratio = (ratio * 0xffe5caca7e10e4e61c3624eaa0941cd0) >> 128;
    if (absTick & 0x10 != 0) ratio = (ratio * 0xffcb9843d60f6159c9db58835c926644) >> 128;
    if (absTick & 0x20 != 0) ratio = (ratio * 0xff973b41fa98c081472e6896dfb254c0) >> 128;
    if (absTick & 0x40 != 0) ratio = (ratio * 0xff2ea16466c96a3843ec78b326b52861) >> 128;
    if (absTick & 0x80 != 0) ratio = (ratio * 0xfe5dee046a99a2a811c461f1969c3053) >> 128;
    if (absTick & 0x100 != 0) ratio = (ratio * 0xfcbe86c7900a88aedcffc83b479aa3a4) >> 128;
    if (absTick & 0x200 != 0) ratio = (ratio * 0xf987a7253ac413176f2b074cf7815e54) >> 128;
    if (absTick & 0x400 != 0) ratio = (ratio * 0xf3392b0822b70005940c7a398e4b70f3) >> 128;
    if (absTick & 0x800 != 0) ratio = (ratio * 0xe7159475a2c29b7443b29c7fa6e889d9) >> 128;
    if (absTick & 0x1000 != 0) ratio = (ratio * 0xd097f3bdfd2022b8845ad8f792aa5825) >> 128;
    if (absTick & 0x2000 != 0) ratio = (ratio * 0xa9f746462d870fdf8a65dc1f90e061e5) >> 128;
    if (absTick & 0x4000 != 0) ratio = (ratio * 0x70d869a156d2a1b890bb3df62baf32f7) >> 128;
    if (absTick & 0x8000 != 0) ratio = (ratio * 0x31be135f97d08fd981231505542fcfa6) >> 128;
    if (absTick & 0x10000 != 0) ratio = (ratio * 0x9aa508b5b7a84e1c677de54f3e99bc9) >> 128;
    if (absTick & 0x20000 != 0) ratio = (ratio * 0x5d6af8dedb81196699c329225ee604) >> 128;
    if (absTick & 0x40000 != 0) ratio = (ratio * 0x2216e584f5fa1ea926041bedfe98) >> 128;
    // 如果第19位非0，因为（2^19 = 0x80000=524288），则乘以 0x2216e584f5fa1ea926041bedfe98，即：2^128 / 1.0001^(524288/2)
    // tick的最大值为887272，因此其二进制最多只需要20位表示，从0开始计数，最后一位为第19位。
    if (absTick & 0x80000 != 0) ratio = (ratio * 0x48a170391f7dc42444e8fa2) >> 128;

    if (tick > 0) ratio = type(uint256).max / ratio;

    // this divides by 1<<32 rounding up to go from a Q128.128 to a Q128.96.
    // we then downcast because we know the result always fits within 160 bits due to our tick input constraint
    // we round up in the division so getTickAtSqrtRatio of the output price is always consistent
    sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
}
```

假设 $i > 0$ 时：

$$
\sqrt{p_{Q128128}(i)} = 2^{128} \cdot \sqrt{p(i)} = 2^{128} \cdot 1.0001^{\frac{i}{2}} \\ = \frac{2^{128}}{1.0001^{-\frac{i}{2}}} = \frac{2^{256}}{2^{128} \cdot \sqrt{p(-i)}} = \frac{2^{256}}{\sqrt{p_{Q128128}(-i)}}
$$

因此，只需要算出 $i < 0$ 时的 ratio 值，使用$2^{256}$除以ratio即可得出 $i > 0$ 时，使用`Q128.128`表示的ratio值：
```solidity
if (tick > 0) ratio = type(uint256).max / ratio;
```

代码最后一行将ratio右移32位，转化为`Q128.96`格式的定点数：

```solidity
sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
```

这里算的是开根号价格$\sqrt{p}$，由于价格$p$最大为$2^{128}$，因此$\sqrt{p}$最大为$2^{64}$，也就是整数部分最大只需要64位表示，因此最终的sqrtPriceX96一定可以用160位（64+96，即`Q64.96`格式的定点数）表示。

#### getTickAtSqrtRatio

该方法对应白皮书中的公式6.8：

$$
i_c = \lfloor \log_{\sqrt{1.0001}} \sqrt{P} \rfloor
$$

本方法涉及在Solidity中计算对数，根据对数公式，可以推出：

$$
\log_{\sqrt{1.0001}} \sqrt{P} = \frac{log_2{\sqrt{P}}}{log_2{\sqrt{1.0001}}} = log_2{\sqrt{P}}  \cdot log_{\sqrt{1.0001}}{2}
$$

由于 $log_{\sqrt{1.0001}}{2}$ 是一个常数，因此我们只需要计算 $log_2{\sqrt{P}}$ 即可。

将输入的参数为$\sqrt{P}$看作$x$，问题转化为求 $log_2{x}$。

把结果分为整数部分$n$和小数部分$m$，则：

$$
n \leq log_2{x} = n + m < n + 1
$$

##### 整数部分

对于$n$，因为：

$$
2^n \leq x < 2^{n+1}
$$

可以通过二分查找找到$n$值：

* 对于256位的数，$0 \leq n < 256$，可以用8位比特表示$n$
* 从二进制表示的第8位（k=7）到第1位（k=0）（从最高位到最低位），依次比较$x$是否大于$2^{2^{k}} - 1$，如果大于则标记该位为1，并右移$2^k$位；否则标记0
* 最终标记后的8位二进制即为$n$值

使用python代码描述如下：

```python
def find_msb(x):
    msb = 0
    for k in reversed(range(8)): // k = 7, 6, 5. 4, 3, 2, 1, 0
        if x > 2 ** (2 ** k) - 1:
            msb += 2 ** k // 标记该位为1，即加上 2 ** k
            x /= 2 ** (2 ** k) // 右移 2 ** k 位
    return msb
```

Uniswap v3中的Solidity代码如下：

```solidity
/// @notice Calculates the greatest tick value such that getRatioAtTick(tick) <= ratio
/// @dev Throws in case sqrtPriceX96 < MIN_SQRT_RATIO, as MIN_SQRT_RATIO is the lowest value getRatioAtTick may
/// ever return.
/// @param sqrtPriceX96 The sqrt ratio for which to compute the tick as a Q64.96
/// @return tick The greatest tick for which the ratio is less than or equal to the input ratio
function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24 tick) {
    // second inequality must be < because the price can never reach the price at the max tick
    require(sqrtPriceX96 >= MIN_SQRT_RATIO && sqrtPriceX96 < MAX_SQRT_RATIO, 'R');
    uint256 ratio = uint256(sqrtPriceX96) << 32; // 右移32位，转化为Q128.128格式

    uint256 r = ratio;
    uint256 msb = 0;

    assembly {
        // 如果大于2 ** (2 ** 7) - 1，则保存临时变量：2 ** 7
        let f := shl(7, gt(r, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF))
        // msb += 2 ** 7
        msb := or(msb, f)
        // r /= (2 ** (2 ** 7))，即右移 2 ** 7
        r := shr(f, r)
    }
    assembly {
        let f := shl(6, gt(r, 0xFFFFFFFFFFFFFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(5, gt(r, 0xFFFFFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(4, gt(r, 0xFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(3, gt(r, 0xFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(2, gt(r, 0xF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(1, gt(r, 0x3))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := gt(r, 0x1)
        msb := or(msb, f)
    }
```

##### 小数部分

对于小数部分$m$：

$$
0 \leq m = log_2{x} - n = log_2{\frac{x}{2^n}} < 1 \tag{1.2}
$$

其中，$n$为上文算出的msb，即整数部分。

我们先将$\frac{x}{2^n}$看做一个整体$r$，则：

$$
0 \leq log_2{r} < 1
$$

$$
1 \leq r = \frac{x}{2^n} < 2
$$

这里我们希望求出$log_2{r}$，如果能够将$log_2{r}$表示成一个不断收敛的数列，当小数位足够多时，就可以近似求出$log_2{r}$的值。

根据对数公式，我们可以推导以下两个等式：

$$
log_2{r} = \frac{2 \cdot log_2{r}}{2} = \frac{log_2{r^2}}{2} \tag{1.3}
$$

$$
log_2{r} = log_2{2 \cdot \frac{r}{2}} = 1 + log_2{\frac{r}{2}} \tag{1.4}
$$

我们循环套用上述两个公式，可以整理以下方法：

1. 因为初始时 $log_2{r} < 1$，因此先应用公式1.3，将问题转化为求$log_2{r^2}$，注意此时基数为$\frac{1}{2}$；
    - 事实上，每一次进入步骤1，新的基数都是上一次基数的$\frac{1}{2}$，比如第二次进入步骤1的基数为$\frac{1}{4}$，以此类推。
2. 如果$r^2$ >= 2，则应用公式1.4，分离出1，并将问题转化为求$log_2{\frac{r^2}{2}}$；
    - 因为公式1.4是在公式1.3之后判断，因此这里的1需要乘以上一次步骤1的基数，如果是第一次则记录$\frac{1}{2}$，第二次则记录$\frac{1}{4}$，以此类推；
    - 因为$1 \leq r < 2$，且$2 \leq r^2 < 4$，因此$1 \leq \frac{r^2}{2} < 2$，将$\frac{r^2}{2}$看做一个整体$r$，又回到步骤1求解$log_2{r}$，并且$1 \leq r < 2$。
3. 如果$r^2 < 2$，则回到步骤1继续。

可以将上述步骤总结为以下公式：

$$
log_2{r} = m_1 \cdot \frac{1}{2} + m_2 \cdot \frac{1}{4} + ... + m_n \cdot \frac{1}{2^n} = \sum^{\infty}_{i=1}(m_i \cdot \frac{1}{2^i}) \tag{1.5}
$$

其中，$m_i \in {0, 1}$。

这其实就是小数的二进制表示法，小数的二进制第一位表示为$2^{-1}$，第二位为$2^{-2}$，以此类推。而在我们上述计算$log_2{r}$的步骤中，如果进入步骤2，则相当于标记该位为1；如果进入步骤3，则相当于标记该位为0。

重复以上步骤的过程，即为确认小数部分二进制位从高位到低位（从左到右）每一位的值，每一个循环确认一位。循环次数越多，计算得出的$log_2{r}$精度越高。

我们继续看Uniswap v3中计算小数部分的代码：

```solidity
        if (msb >= 128) r = ratio >> (msb - 127);
        else r = ratio << (127 - msb);
```

这里msb即为整数部分$n$。因为ratio是`Q128.128`，如果`msb >= 128`则表示`ratio >= 1`，因此需要右移整数位数得到小数部分`ratio >> msb`；`-127`表示左移127位，使用`Q129.127`表示小数部分；同样，如果`msb < 128`，则表示`ratio < 1`，其本身就只有小数部分，因此通过左移`127 - msb`位，将小数部分凑齐127位，也用`Q129.127`表示小数部分。

实际上，`ratio >> msb`即为公式1.2中的$\frac{x}{2^n}$，也就是步骤1中的$r$，在后续迭代算法（步骤1-3）中需要用到。

```solidity
    int256 log_2 = (int256(msb) - 128) << 64;
```
因为msb是基于`Q128.128`的ratio计算的，`int256(msb) - 128`表示$n$的真正值。`<< 64`使用`Q192.64`表示$n$。
这一行代码实际上是使用`Q192.64`保存整数部分的值。

下面代码循环计算二进制表示的小数部分的前14位小数：

```solidity
    assembly {
        // 根据步骤1，计算r^2，右移127位是因为两个r都是Q129.127
        r := shr(127, mul(r, r))
        // 因为1 <= r^2 < 4，仅需2位表示r^2的整数，
        // 因此从右往左数第129和128位表示r^2的整数部分，
        // 右移128位，仅剩129位，
        // 该值为1，则表示r >= 2；该值为0，则表示r < 2
        let f := shr(128, r)
        // 如果f == 1，则log_2 += Q192.64的1/2
        log_2 := or(log_2, shl(63, f))
        // 根据步骤2（即公式1.4），如果r >= 2（即f == 1），则r /= 2；否则不操作，即步骤3
        r := shr(f, r)
    }
    // 重复进行上述过程
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(62, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(61, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(60, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(59, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(58, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(57, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(56, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(55, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(54, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(53, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(52, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(51, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(50, f))
    }
```

上述计算的log_2即为`Q192.64`表示的$log_2{\sqrt{P}}$，精度为$2^{-14}$。

```solidity
    int256 log_sqrt10001 = log_2 * 255738958999603826347141; // 128.128 number
```
因为：

$$
\log_{\sqrt{1.0001}} \sqrt{P} = \frac{log_2{\sqrt{P}}}{log_2{\sqrt{1.0001}}} = log_2{\sqrt{P}}  \cdot log_{\sqrt{1.0001}}{2}
$$

这里`255738958999603826347141`即为$log_{\sqrt{1.0001}}{2} \cdot 2^{64}$，两个`Q192.64`乘以的结果为`Q128.64`（不会发生溢出）。

由于这里算出的$log_2{\sqrt{P}}$精度为$2^{-14}$，乘以`255738958999603826347141`后误差进一步放大，因此需要修正并确保结果是最接近给定价格的tick。

```solidity
    int24 tickLow = int24((log_sqrt10001 - 3402992956809132418596140100660247210) >> 128);
    int24 tickHi = int24((log_sqrt10001 + 291339464771989622907027621153398088495) >> 128);

    tick = tickLow == tickHi ? tickLow : getSqrtRatioAtTick(tickHi) <= sqrtPriceX96 ? tickHi : tickLow;
```

其中，`3402992956809132418596140100660247210`表示`0.01000049749154292 << 128`，`291339464771989622907027621153398088495`表示`0.8561697375276566 << 128`。

参考[abdk的这篇文章](https://hackmd.io/@abdk/SkVJeHK9v)，当精度为$2^{-14}$时，tick的最小误差为$−
0.85617$，最大误差为$0.0100005$。

我们的目的是寻找满足当前最大的tick，使得tick对应的$\sqrt{P}$小于等于传入的值。因此如果补偿后的tickHi满足要求，则优先使用tickHi；否则使用tickLow。

本节参考文章，有兴趣的朋友请扩展阅读：

* [Solidity中的对数计算](https://liaoph.com/logarithm-in-solidity/)
* [Math in Solidity](https://medium.com/coinmonks/math-in-solidity-part-5-exponent-and-logarithm-9aef8515136e)
* [Logarithm Approximation Precision](https://hackmd.io/@abdk/SkVJeHK9v)
