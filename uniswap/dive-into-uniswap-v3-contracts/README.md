# 深入理解 Uniswap v3 合约代码

> 如果无法正常显示文档中的数学公式，请安装Chrome浏览器插件：[MathJax Plugin for Github](https://chrome.google.com/webstore/detail/mathjax-plugin-for-github/ioemnmodlmafdkllaclgeombjnmnbima?hl=en)
>
> Install Chrome extension [MathJax Plugin for Github](https://chrome.google.com/webstore/detail/mathjax-plugin-for-github/ioemnmodlmafdkllaclgeombjnmnbima?hl=en) if the math formulas are not rendered correctly on your browser.


## Tick

### TickMath

TickMath主要包含两个方法：

* `getSqrtRatioAtTick`：根据tick计算开根号价格$\sqrt{P}$
* `getTickAtSqrtRatio`：根据根号价格$\sqrt{P}$计算tick


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


对于一个给定的tick $i$（$i$是自然数），它总可以表示为二进制，因此以下式子总是成立：

$$
\begin{cases} i = \sum_{n=0}^{19}{(x_n \cdot 2^n)} = x_0 \cdot 1 + x_1 \cdot 2 + x_2 \cdot 4 + ... + x_{19}\cdot 524288 \\ x_n \in \{0, 1\} \end{cases} \tag{1.1}
$$

其中，$x_n$为$i$的二进制位。如$i=6$，其对应的二进制为：`000000000000000000000110`，则$x_1 = 1, x_2 = 1$，其余$x_n$均为0。

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

可以总结计算$\sqrt{p}(i)$的办法，初始值为1，从右往左循环遍历$i$的二进制比特位，如果该位不为0，则乘以对应的$\frac{2^{128}}{1.0001^{\frac{2^n}{2}}}$，其中$2^{128}$表示左移128位。


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
    // 因为tick的最大值为887272，因此其二进制最多只需要20位表示，从0开始计数，最后一位为第19位。
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

因此，只需要算出 i < 0 时的 ratio 值，使用$2^{256}$处以 ratio 即可得出 i > 0 的ratio值：
```solidity
if (tick > 0) ratio = type(uint256).max / ratio;
```

代码最后一行将ratio右移32位，转化为`Q128.96`格式的定点数；由于这里实际上是开根号价格，由于价格$p$最大为$2^{128}$，因此$\sqrt{p}$最大为$2^{64}$，也就是整数部分最大只需要64位表示，因此最终的sqrtPriceX96一定可以用160位（64+96）表示，即`Q64.96`格式的定点数。
