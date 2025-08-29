[English](./yield-basis-risk-return-analysis.md) | [中文](../zh/yield-basis-risk-return-analysis.md)

# Yield Basis Risk-Return Comprehensive Analysis Report

## Executive Summary

Based on in-depth code analysis and Curve historical data, Yield Basis's **core innovation is completely eliminating impermanent loss through a 2x leverage mechanism**, allowing LPs to maintain BTC exposure while earning yields. The protocol offers a unique yield stacking mechanism: users can stake ybBTC to earn YB tokens, then lock YB as veYB to achieve dual yields (YB incentives + BTC admin fees), with voting rights further enhancing returns. Although base yields are negative (trading fees 0.5-2% - expected losses 3.9% = -3.4% to -1.9% APY), the combined strategy can achieve 2.1% to 11.1% APY after credit line approval. Market equilibrium is expected to stabilize at 8-12% APY through the flywheel effect of early veYB holders.

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

**Important Note: Users can achieve yield stacking through combined strategies**

#### Base Modes (Choose One)

##### Mode A: Hold ybBTC Unstaked (Earn BTC Trading Fees)
| Yield Type | Estimated APY | Description |
|------------|---------------|-------------|
| Curve Trading Fee Share | 0.5-2% | After deducting dynamic admin fee, distributed in BTC |
| **No Impermanent Loss Value** | Unquantifiable | **Core innovation: Maintain BTC exposure while earning yields** |
| **Total Yield** | **0.5-2%** | Pure BTC yield, no token risk |

##### Mode B: Stake ybBTC to Gauge (Earn YB Tokens)
| Yield Type | Estimated APY | Description |
|------------|---------------|-------------|
| YB Token Incentives | 5-10%* | Forgo BTC trading fees, receive YB emissions |
| **No Impermanent Loss Value** | Unquantifiable | **Core innovation: Maintain BTC exposure while earning yields** |
| **Total Yield** | **5-10%** | Pure YB token yield |

#### Advanced Strategy: Yield Stacking

##### Combined Strategy: Stake ybBTC + Lock YB as veYB
| Yield Source | Estimated APY | Description |
|------------|---------------|-------------|
| YB Token Incentives | 5-10% | From staking ybBTC |
| Dynamic Admin Fee Share | 1-5% | Lock earned YB as veYB, earn BTC |
| Governance Boost | - | veYB can vote to increase own pool's YB emission weight |
| **Combined Total Yield** | **6-15%** | YB tokens + BTC dual yield |

*Note:
- ybBTC holders must choose between unstaked (earn BTC fees) and staked (earn YB)
- **Yield Stacking Mechanism**: YB earned from staking ybBTC can be locked as veYB for dual yields
- veYB holders receive both protocol revenue and can vote to increase their pool's YB emissions
- Fee distribution: 50% for fee distribution, 50% for pool rebalancing
- Dynamic admin fee: Increases with staking rate (10%-100%), captured by veYB holders
- YB token price assumption: Initial $0.05, conservative expectation $0.10-0.15 (2-3x)

### 3.2 Cost Comparison

#### Current Model (with borrowing costs)

**Base Mode A: Unstaked ybBTC (Earn BTC fees)**
```
Total Yield: 0.5-2% APY (BTC trading fees)
- crvUSD Borrowing Rate: 4-8% APY
- Expected Losses: 3.9% APY
= Net Yield: -11.4% to -5.9% APY (negative returns)
```

**Base Mode B: Staked ybBTC (Earn YB tokens)**
```
Total Yield: 5-10% APY (YB token incentives)
- crvUSD Borrowing Rate: 4-8% APY
- Expected Losses: 3.9% APY
= Net Yield: -6.9% to 1.1% APY (near breakeven)
```

**Combined Strategy: Stake ybBTC + Lock veYB**
```
Total Yield: 6-15% APY (YB incentives + veYB admin fees)
- crvUSD Borrowing Rate: 4-8% APY
- Expected Losses: 3.9% APY
= Net Yield: -5.9% to 3.1% APY (possible profit)
```

#### After Curve Credit Line (zero interest)

**Base Mode A: Unstaked ybBTC (Earn BTC fees)**
```
Total Yield: 0.5-2% APY (BTC trading fees)
- Borrowing Rate: 0% APY
- Expected Losses: 3.9% APY
= Net Yield: -3.4% to -1.9% APY (still negative)
```

**Base Mode B: Staked ybBTC (Earn YB tokens)**
```
Total Yield: 5-10% APY (YB token incentives)
- Borrowing Rate: 0% APY
- Expected Losses: 3.9% APY
= Net Yield: 1.1% to 6.1% APY (positive returns)
```

**Combined Strategy: Stake ybBTC + Lock veYB**
```
Total Yield: 6-15% APY (YB incentives + veYB admin fees)
- Borrowing Rate: 0% APY
- Expected Losses: 3.9% APY
= Net Yield: 2.1% to 11.1% APY (good returns)
```

#### Long-term Yield After Market Equilibrium
```
Based on emission mechanism's automatic adjustment:
YB annual emission is fixed, but allocation per LP depends on TVL

Example calculation (after credit line):
Mode A (Unstaked):
- Base Yield: 1-2% APY (BTC trading fees)
- Expected Losses: -3.9% APY
- Net Yield: -2.9% to -1.9% APY (unsustainable)

Mode B (Staked for YB):
- Forgo BTC trading fees: 0% APY
- Expected Losses: -3.9% APY
- Need YB incentives > 3.9% APY to breakeven

Equilibrium Mechanism (Staking mode):
Assuming annual emission of 100M YB (10% of assumed 1B total):
- If TVL $200M, each $1 gets 0.5 YB, needs price $0.20 (4x) for 10% APY
- If TVL $100M, each $1 gets 1 YB, needs price $0.10 (2x) for 10% APY
- If TVL $50M, each $1 gets 2 YB, needs price $0.05 (1x) for 10% APY

Market will automatically find equilibrium:
TVL × YB price = constant (similar to xy=k)
Eventually stabilizes at reasonable risk-adjusted yield level (expected 6-8% APY)
```

## IV. Scenario Analysis

### 4.1 Three Scenario Predictions (Adjusted)

| Scenario | Probability | Conditions | Expected Yield (incl. YB) | YB Price Assumption |
|----------|------------|------------|---------------------------|---------------------|
| **Optimistic** | 20% | Early participation, YB 3x | 10% APY | $0.15/YB |
| **Base Case** | 50% | Normal market, YB 2x | 6% APY | $0.10/YB |
| **Pessimistic** | 30% | Market saturation, YB below issue price | 2% APY | $0.03/YB |

### 4.2 Expected Return Calculation (After Credit Line)

```
Base Mode A: Unstaked (pure BTC trading fees):
E(gross yield) = 0.2 × 1.75% + 0.5 × 1.25% + 0.3 × 0.75%
              = 1.2% APY
E(net yield) = 1.2% - 3.9% = -2.7% APY (negative returns)

Base Mode B: Staked (pure YB tokens):
E(YB yield) = 0.2 × 10% + 0.5 × 6% + 0.3 × 2%
           = 5.6% APY
E(net yield) = 5.6% - 3.9% = 1.7% APY (small profit)

Combined Strategy: Staked + veYB (yield stacking):
E(YB yield) = 5.6% APY
E(veYB yield) = 0.2 × 4% + 0.5 × 2.5% + 0.3 × 1%
             = 2.35% APY
E(total net yield) = 5.6% + 2.35% - 3.9% = 4.05% APY (reasonable returns)
```

## V. Risk-Adjusted Metrics

### 5.1 Sharpe Ratio (After Credit Line)

**Base Mode A: Unstaked (Earn BTC fees)**
- Expected Return: -2.7% APY (trading fees 1.2% - losses 3.9%)
- Standard Deviation: 8% (BTC price volatility)
- Risk-free Rate: 4% (US Treasury)
- **Sharpe Ratio = -0.84** (strongly negative)

**Base Mode B: Staked (Earn YB tokens)**
- Expected Return: 1.7% APY (YB 5.6% - losses 3.9%)
- Standard Deviation: 15% (YB token volatility higher)
- Risk-free Rate: 4% (US Treasury)
- **Sharpe Ratio = -0.15** (negative, worse than risk-free asset after risk adjustment)

**Combined Strategy: Staked + veYB**
- Expected Return: 4.05% APY (YB + veYB - losses)
- Standard Deviation: 12% (partial BTC yields reduce volatility)
- Risk-free Rate: 4% (US Treasury)
- **Sharpe Ratio = 0.004** (near zero, just covers risk-free rate)

### 5.2 Maximum Drawdown
- Normal Market: -5%
- Extreme Market: -30%
- Black Swan: -50% to -90%

### 5.3 Risk-Reward Ratio (After Credit Line)
- Expected Return (Base Mode A: BTC fees): -2.7% APY
- Expected Return (Base Mode B: YB tokens): 1.7% APY
- Expected Return (Combined Strategy): 4.05% APY
- Maximum Regular Loss: 30%
- **Risk-Reward Ratio (Mode A): -0.09** (negative)
- **Risk-Reward Ratio (Mode B): 0.06** (extremely low)
- **Risk-Reward Ratio (Combined): 0.14** (low but acceptable)

## VI. YB Token Value Assessment

### 6.1 YB Token Economic Features
- **Total Supply**: 1 billion (assumed value, actual determined by reserve parameter at deployment)
- **Initial Valuation**: $50M FDV ($0.05/YB, based on assumed 1B total supply)
- **Unique Mechanism**: "Mining cost" model, each YB backed by opportunity cost of foregone BTC yields
- **veYB Locking**: Lock YB up to 4 years for veYB, receive protocol revenue share and governance rights
- **Fee Capture**: veYB holders receive dynamic admin fees (10%-50% of total trading fees)

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

### 6.3 Fee Distribution and Incentive Balance Mechanism

**Dynamic Admin Fee Mechanism**:
- 0% staking rate: 10% admin fee (veYB receives minimum)
- 50% staking rate: ~36.4% admin fee (veYB receives moderate)
- 100% staking rate: 100% admin fee (veYB receives all)

**Yield Stacking Effect**:
1. **Single Strategy**:
   - Unstaked ybBTC: Only receive BTC trading fees
   - Staked ybBTC: Only receive YB tokens
2. **Combined Strategy** (Yield Stacking):
   - Stake ybBTC → Receive YB
   - Lock YB as veYB → Additionally receive admin fee income
   - veYB voting rights → Increase own pool's YB emissions
   - **Total Yield = YB incentives + veYB admin fees + voting boost**

**Flywheel Effect**:
- Early participants achieve higher yields through combined strategy
- veYB holders form aligned interest group
- Concentrated voting rights direct YB emissions to core pools
- Creates positive cycle: More staking → More veYB → Higher yields

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

- **Risk Level: High**
- **Yield Level (without YB): Negative**
- **Yield Level (with YB): Low-Medium (5.6% APY)**
- **Yield Level (Combined Strategy): Medium (4.05% APY expected, 8-12% APY at equilibrium)**
- **Core Value: Eliminate impermanent loss, maintain BTC exposure**
- **Suitable For: Long-term investors wanting to hold BTC while earning yields**

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
   - Base Mode A (Unstaked): -11.4% to -5.9% APY (severe losses)
   - Base Mode B (Staked): -6.9% to 1.1% APY (near breakeven)
   - Combined Strategy: -5.9% to 3.1% APY (possible small profit)
   - Conclusion: Even combined strategy struggles to cover costs

2. **After Credit Line Approval**:
   - Recommendation: **Moderate participation in combined strategy acceptable** (5-10% position)
   - Base Mode A (Unstaked): -3.4% to -1.9% APY (not recommended)
   - Base Mode B (Staked): 1.1% to 6.1% APY (acceptable)
   - **Combined Strategy: 2.1% to 11.1% APY (recommended)**
   - Optimal strategy: 70% stake ybBTC, lock all earned YB as veYB

3. **After Market Equilibrium (6-12 months)**:
   - Flywheel effect forms, early veYB holders gain advantage
   - Expected equilibrium yields:
     - Unstaked ybBTC: -1% to 0% APY (basically unprofitable)
     - Pure staked ybBTC: 4-6% APY (after YB price stabilizes)
     - **Combined Strategy: 8-12% APY (YB + veYB + voting rights)**
   - Recommendation: Establish veYB position early for long-term compounding

## IX. Core Conclusions

### Advantages
- ✅ **Core Innovation**: Completely eliminate impermanent loss through 2x leverage (protocol's biggest highlight)
- ✅ **Maintain Exposure**: LPs can continuously hold BTC exposure while earning yields
- ✅ **No Liquidation Risk**: Soft liquidation mechanism, no forced liquidation
- ✅ **Curve Endorsement**: Deep integration, mature technology
- ✅ **YB Token Potential**: 2-3x potential upside (if successful)
- ✅ **Diversified Yields**: Three participation methods suit different risk preferences
- ✅ **Self-Balancing Mechanism**: Dynamic fee distribution ensures long-term viability of all strategies

### Disadvantages
- ❌ **Extremely Low Base Yield**: All modes initially lose money after deducting losses
- ❌ **High Tail Risk**: Small probability of large losses (potentially 30-50% loss)
- ❌ **Token Price Risk**: YB may go to zero or underperform
- ❌ **High Complexity**: Three-way game theory mechanism, difficult for ordinary users
- ❌ **Team Risk**: Founder's controversial history, severe over-leverage tendency
- ❌ **Multiple Risk Stacking**: Protocol risk + token risk + strategy selection risk
- ❌ **High Opportunity Cost**: Inferior to directly holding BTC or stablecoin yield farming
- ❌ **Liquidity Risk**: veYB lock-up period up to 4 years

### One-Sentence Summary

**Yield Basis completely eliminates impermanent loss through a 2x leverage mechanism, allowing LPs to maintain BTC exposure while earning yields. The protocol's core value lies in its yield stacking mechanism: after staking ybBTC to earn YB tokens, users can lock YB as veYB to achieve dual yields of YB incentives + BTC admin fees, while further enhancing returns through voting rights. After Curve credit line approval, the combined strategy can achieve expected returns of 8-12% APY. Early participants establishing veYB positions can enjoy long-term compounding from the flywheel effect. Despite the 3.9% expected loss, the protocol provides a unique BTC yield opportunity for risk-tolerant investors through carefully designed incentive mechanisms and yield stacking.**

---

*Report Date: August 2025*
*Risk Warning: DeFi investments carry extremely high risks. This report does not constitute investment advice. Investors must bear all risks themselves.*