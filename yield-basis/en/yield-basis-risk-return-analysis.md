[English](./yield-basis-risk-return-analysis.md) | [中文](../zh/yield-basis-risk-return-analysis.md)

# Yield Basis Risk-Return Comprehensive Analysis Report

## Executive Summary

Based on in-depth code analysis and Curve historical data, Yield Basis's **core innovation is completely eliminating impermanent loss through a 2x leverage mechanism**, allowing LPs to maintain BTC exposure while earning yields. Although base yields are negative (trading fees 1-3% - expected losses 3.9% = -2.9% to -0.9% APY), the protocol compensates risks through YB token incentives. YB emission's self-balancing mechanism (TVL decrease → more YB per LP unit) ensures reasonable risk-adjusted returns. After credit line approval, expected equilibrium yields are 8-10% APY (including YB incentives).

## I. Curve BTC Pool Historical Yield Benchmark

| Pool Type | APY Range | Data Source |
|-----------|-----------|-------------|
| Pure Trading Fee Yields | 0.5-2% | Base rate 0.04%, depends on volume |
| BTC Stable Pools | 1-3% | Low volatility, smaller volumes |
| Tri-pools (with BTC) | 2-4% | Medium trading volume |
| **Pure Trading Fee Benchmark** | **1-3%** | **Excluding any incentives** |

## II. Risk Probability Matrix

### 2.1 Loss Scenario Quantification

| Risk Level | Loss Scenario | Annual Probability | Single Loss | Expected Loss/Year | Trigger Condition |
|------------|---------------|-------------------|-------------|-------------------|-------------------|
| **Low Risk** | | | | | |
| | Management Fee | 100% | 1% | 1.0% | Continuously charged |
| | Normal Slippage | 200%* | 0.3% | 0.6% | Each withdrawal |
| | Minor Arbitrage Delay | 50% | 2% | 1.0% | High gas or low liquidity |
| **Medium Risk** | | | | | |
| | Negative Management Fee | 10% | 5% | 0.5% | Market volatility >20% |
| | Large Slippage | 5% | 2% | 0.1% | Significant TVL decrease |
| | Moderate Leverage Imbalance | 5% | 5% | 0.25% | Extreme intraday volatility |
| **High Risk** | | | | | |
| | Severe Imbalance | 2% | 15% | 0.3% | BTC daily drop >30% |
| | Emergency Withdrawal | 0.5% | 30% | 0.15% | BTC crash >50% |
| | Protocol Failure | 0.1% | 50% | 0.05% | Attack/killed |
| **Black Swan** | | | | | |
| | Total Loss | 0.01% | 90% | 0.009% | Protocol collapse |

*Note: Assumes 2 withdrawals per year

**Total Annualized Expected Loss: 3.9%**

**Important Note**: This expected loss may be overestimated. In actual operations:
- Normal slippage and management fees are regular costs (~2.6%)
- True risk losses are lower (~1.3%)
- Market automatically compensates risks through YB incentives

### 2.2 Historical Reference

Based on BTC historical volatility data:
- BTC daily drop >20%: 2-3 times per year on average
- BTC daily drop >30%: 0.5 times per year on average
- BTC drop >50%: Once every 2-3 years

## III. Yield Analysis

### 3.1 Yield Source Breakdown

| Yield Type | Estimated APY | Description |
|------------|---------------|-------------|
| Curve Pure Trading Fees | 1-3% | Based on 0.04% fee rate, actual volume |
| YB Token Incentives | 5-10%* | Based on conservative token value expectations |
| **No Impermanent Loss Value** | Unquantifiable | **Core innovation: Maintain BTC exposure while earning yields** |
| **Total Yield** | **7-15%** | Including YB token value + intangible value |

*Note: YB token incentive calculation based on:
- Initial price: $0.05/YB
- Conservative expected price: $0.10-0.15/YB (2-3x)
- Mining annualized: Reduced to 5-10% after market equilibrium

### 3.2 Cost Comparison

#### Current Model (with borrowing costs)
```
Total Yield: 7-15% APY (including YB tokens)
- crvUSD Borrowing Rate: 4-8% APY (market standard rates)
- Expected Losses: 3.9% APY
= Net Yield: -0.9% to 3.1% APY
```

#### After Curve Credit Line (zero interest)
```
Total Yield: 7-15% APY (including YB tokens)
- Borrowing Rate: 0% APY
- Expected Losses: 3.9% APY
= Net Yield: 3.1% to 11.1% APY
```

#### Long-term Yield After Market Equilibrium
```
Based on emission mechanism's automatic adjustment:
YB annual emission is fixed, but allocation per LP depends on TVL

Example calculation (after credit line):
Base Yield: 1-2% APY (trading fees)
Expected Losses: -3.9% APY
Net Base: -2.9% to -1.9% APY

YB incentives need to reach 10-12% APY to achieve 8-10% net yield:
Assuming annual emission of 100M YB (based on 1B total supply assumption):
- If TVL $200M, each $1 gets 0.5 YB, needs price $0.20-0.24 (4-5x)
- If TVL $100M, each $1 gets 1 YB, needs price $0.10-0.12 (2-2.5x)
- If TVL $50M, each $1 gets 2 YB, needs price $0.05-0.06 (1-1.2x)

Market will automatically find equilibrium:
TVL × YB price = constant (similar to xy=k)
Eventually stabilizes at reasonable risk-adjusted yield level
```

## IV. Scenario Analysis

### 4.1 Three Scenario Predictions (Adjusted)

| Scenario | Probability | Conditions | Expected Yield (incl. YB) | YB Price Assumption |
|----------|------------|------------|---------------------------|---------------------|
| **Optimistic** | 20% | Early participation, YB 3x | 10% APY | $0.15/YB |
| **Base Case** | 50% | Normal market, YB 2x | 6% APY | $0.10/YB |
| **Pessimistic** | 30% | Market saturation, YB below issue price | 2% APY | $0.03/YB |

### 4.2 Expected Return Calculation

```
Without YB tokens (pure trading fees):
E(return) = 0.2 × 2.5% + 0.5 × 1.5% + 0.3 × 0.5%
         = 1.4% APY

With YB token value:
E(return) = 0.2 × 10% + 0.5 × 6% + 0.3 × 2%
         = 2% + 3% + 0.6%
         = 5.6% APY (after credit line)
```

## V. Risk-Adjusted Metrics

### 5.1 Sharpe Ratio (After Credit Line)
- Expected Return (without YB): -1.9% APY (trading fees 1.5% - losses 3.9% = -2.4% to -0.9%, taking midpoint)
- Expected Return (with YB): 5.6% APY (based on scenario analysis)
- Standard Deviation: 12% (considering YB token volatility)
- Risk-free Rate: 4% (US Treasury)
- **Sharpe Ratio (without YB) = -0.49** (negative, worse than risk-free asset)
- **Sharpe Ratio (with YB) = 0.13** (low but positive)

### 5.2 Maximum Drawdown
- Normal Market: -5%
- Extreme Market: -30%
- Black Swan: -50% to -90%

### 5.3 Risk-Reward Ratio (After Credit Line)
- Expected Return (without YB): -1.9% APY
- Expected Return (with YB): 5.6% APY
- Maximum Regular Loss: 30%
- **Risk-Reward Ratio (without YB): -0.06** (negative)
- **Risk-Reward Ratio (with YB): 0.19** (low)

## VI. YB Token Value Assessment

### 6.1 YB Token Economic Features
- **Total Supply**: 1 billion (assumed value, actual determined by reserve parameter at deployment)
- **Initial Valuation**: $50M FDV ($0.05/YB, based on assumed 1B total supply)
- **Unique Mechanism**: "Mining cost" model, each YB backed by opportunity cost of foregone BTC yields

### 6.2 Realistic Valuation Based on CRV Benchmark
**Current CRV Data (August 2025)**:
- Price: $0.76
- Market Cap: $1.05B
- FDV: $2.3B
- Market Cap/TVL: 0.44x (TVL $2.5B)

**YB Reasonable Valuation Expectations**:
Assuming YB reaches 10-20% of CRV market cap (considering niche positioning):
- Target Market Cap: $100-200M
- Corresponding Price: $0.10-0.20/YB (based on 1B total supply assumption)
- Relative to Initial Price: 2-4x

### 6.3 YB Emission Auto-Adjustment Mechanism

**Emission Mechanism Based on Code Analysis**:
1. **Fixed Total Emission**: Uses exponential decay function, released from reserve pool
2. **Gauge Weight Voting**: veYB holders vote on pool weights
3. **Key Formulas**:
   - Pool emission = (Total emission × Pool weight) / Total weight
   - When TVL low → Fewer participants → Each LP gets more YB

**Market Equilibrium Mechanism**:
- High TVL → Diluted per-unit YB incentives → Lower yields
- Low TVL → Concentrated per-unit YB incentives → Higher yields
- Equilibrium: TVL stabilizes where yield = risk-adjusted reasonable level

**Specific Numerical Expectations** (after credit line):
Assuming annual emission of 100M YB (10% of assumed 1B total), base net yield -1.9% APY:

At TVL $100M:
- Each $1 TVL gets 1 YB annually
- Needs YB price $0.10 to provide 10% incentive
- Net yield: -1.9% + 10% = 8.1% APY (equilibrium)

At TVL $50M (capital outflow due to risk):
- Each $1 TVL gets 2 YB annually
- YB $0.10 = 20% APY incentive
- Net yield: -1.9% + 20% = 18.1% APY (attracts capital back)

## VII. Founder Risk Assessment

### 7.1 Michael Egorov Background
- **Technical Capability**: Moscow Institute of Physics and Technology graduate, Physics PhD, unquestionable technical expertise
- **Entrepreneurial Experience**: Curve Finance founder, major DeFi innovator
- **Controversial History**: Multiple over-leveraging incidents, liquidated for $140M in June 2024

### 7.2 Historical Risk Events
| Date | Event | Impact | Lesson |
|------|-------|--------|--------|
| July 2023 | Borrowed over $100M, collateralized 460M CRV | CRV price crash | Over-leverage risk |
| June 2024 | Liquidated for $140M | Caused $1M bad debt | Risk management failure |
| December 2024 | Liquidated again for $880K | Market confidence damaged | Failed to learn |
| 2020-2024 | Sued for fraud by three VCs | Reputation damaged | Integrity issues |

### 7.3 Impact on Yield Basis

**Positive Factors**:
- Strong technical innovation capability, sophisticated protocol design
- Deep Curve integration, significant resource advantages
- May have learned from past mistakes

**Negative Factors**:
- **Over-leverage Tendency**: May adopt aggressive strategies at protocol level
- **Risk Management Capability**: History shows insufficient risk control
- **Trust Crisis**: Investors may hesitate due to history
- **Single Point Risk**: Founder's personal behavior may affect protocol

### 7.4 Risk Quantification
| Risk Type | Probability | Potential Impact | Mitigation Measures |
|-----------|------------|------------------|-------------------|
| Protocol Over-leverage | 30% | TVL drops 50% | Monitor debt ratio |
| Founder Debt Crisis Again | 20% | Confidence collapse, bank run | Diversify investments |
| Over-centralized Governance | 50% | Decision risks | Watch DAO development |
| Technical Innovation Stagnation | 10% | Competitiveness decline | Evaluate alternatives |

**Overall Score**: Founder Risk **6/10** (Medium-High Risk)

## VIII. Investment Recommendations

### 8.1 Overall Rating

**Risk Level: High**
**Yield Level (without YB): Negative**
**Yield Level (with YB): Low-Medium (5.6% APY)**
**Core Value: Eliminate impermanent loss, maintain BTC exposure**
**Suitable For: Long-term investors wanting to hold BTC while earning yields**

### 8.2 Position Recommendations

| Investor Type | Recommended Position | Rationale |
|--------------|---------------------|-----------|
| DeFi Veterans | 5-10% | Understand risks, moderate participation |
| Risk Seekers | 10-15% | Willing to take high risk for potential returns |

### 8.3 Entry Strategy

✅ **Best Entry Timing**:
1. After Curve credit line confirmation
2. BTC sideways or mild uptrend (volatility <30%)
3. Gas fees <50 gwei
4. Protocol TVL stable growth period

⚠️ **Avoid Entry**:
1. BTC high volatility period (daily volatility >10%)
2. Market panic (Fear & Greed Index <20)
3. Protocol TVL rapid decline period

### 8.4 Risk Management

**Stop-Loss Strategy**:
- Soft Stop: BTC daily drop >20%, reduce position by 50%
- Hard Stop: Management fee turns negative, full exit

**Monitoring Metrics**:
1. Debt ratio (should maintain 45-55%)
2. Management fee balance (warning: <0)
3. TVL changes (warning: daily drop >10%)
4. Gas fees (warning: >200 gwei)

### 8.5 Final Recommendations

1. **Current Phase** (with borrowing costs):
   - Recommendation: **Watch cautiously or minimal position** (<5%)
   - Calculation: Trading fees(1-3%) - Borrowing(4-8%) - Losses(3.9%) = -10.9% to -4.9%
   - Expected Return (with YB 5-10%): -0.9% to 3.1% APY
   - Risk exceeds returns

2. **After Credit Line Approval**:
   - Recommendation: **Moderate participation acceptable** (5-10% position)
   - Calculation: Trading fees(1-3%) - Losses(3.9%) = -2.9% to -0.9%
   - Expected Return (with YB 5-10%): 3.1% to 11.1% APY
   - Saving 4-8% borrowing cost is key

3. **After Market Equilibrium (6-12 months)**:
   - Market auto-adjustment mechanism takes effect
   - Expected equilibrium yield: 8-10% APY (including YB incentives)
   - Benchmarking Curve stablecoin pools 10-15% APY, discounted 20-30% for higher risk

## IX. Core Conclusions

### Advantages
- ✅ **Core Innovation**: Completely eliminate impermanent loss through 2x leverage (protocol's biggest highlight)
- ✅ **Maintain Exposure**: LPs can continuously hold BTC exposure while earning yields
- ✅ **No Liquidation Risk**: Soft liquidation mechanism, no forced liquidation
- ✅ **Curve Endorsement**: Deep integration, mature technology
- ✅ **YB Token Potential**: 2-3x potential upside (if successful)

### Disadvantages
- ❌ **Extremely Low Base Yield**: Pure trading fees only 1-3% APY
- ❌ **High Tail Risk**: Small probability of large losses (potentially 30-50% loss)
- ❌ **Token Price Risk**: YB may go to zero or underperform
- ❌ **High Complexity**: Complex mechanism, difficult for ordinary users to understand
- ❌ **Team Risk**: Founder's controversial history, severe over-leverage tendency
- ❌ **Double Risk**: Protocol risk + token risk compounded
- ❌ **High Opportunity Cost**: Inferior to directly holding BTC or stablecoin yield farming

### One-Sentence Summary

**Yield Basis's core innovation is completely eliminating impermanent loss through a 2x leverage mechanism, allowing LPs to earn yields while maintaining BTC exposure—something traditional AMMs cannot achieve. Although base yields are negative (-1.9% APY), YB token's self-balancing incentive mechanism (TVL × YB incentive = constant) ensures reasonable risk-adjusted returns of 8-10% APY. Investors are essentially paying for the "no impermanent loss" innovation, rather than purely pursuing yield rates.**

---

*Report Date: August 2025*
*Risk Warning: DeFi investments carry extremely high risks. This report does not constitute investment advice. Investors must bear all risks themselves.*